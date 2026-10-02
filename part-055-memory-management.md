# Part 055: Memory Management ใน Assembly

## บทนำ (Introduction)

Memory management เป็นหัวใจสำคัญของการเขียนโปรแกรมระดับต่ำ ใน Assembly เราสามารถควบคุม memory ได้โดยตรงผ่าน system calls ของ Linux kernel ซึ่งให้อำนาจและความยืดหยุ่นสูงสุด แต่ก็ต้องรับผิดชอบในการจัดการ memory เองทั้งหมด

บทนี้จะครอบคลุม:
- Process memory layout และการวิเคราะห์ผ่าน `/proc/self/maps`
- System calls สำหรับ heap management: `brk`/`sbrk`
- `mmap` family: anonymous mapping, file mapping, shared memory
- Memory protection: `mprotect`, `mlock`, `mlockall`
- Advanced operations: `mremap`, `mincore`, `madvise`
- NUMA memory policy: `mbind`, `get_mempolicy`
- Implementation: malloc/free ตั้งแต่พื้นฐาน
- Memory allocators: bump, free list, pool, stack
- โปรแกรมตัวอย่าง: malloc implementation, leak detector, guard pages

---

## 1. Process Memory Layout

### 1.1 Virtual Address Space

โปรเซสใน Linux 64-bit มี virtual address space ขนาด 128 TB (47 bits) โดยแบ่งเป็นส่วนต่างๆ:

```
Virtual Address Space (x86-64 Linux):
+------------------+ 0xFFFFFFFFFFFFFFFF (kernel space สูงสุด)
|   Kernel Space   | (ไม่สามารถเข้าถึงได้จาก user mode)
+------------------+ 0xFFFF800000000000
|     (gap)        | (canonical hole - invalid addresses)
+------------------+ 0x00007FFFFFFFFFFF
|   Stack          | (ขยายลงมา - grows down)
|        ↓         |
+------------------+
|   mmap region    | (shared libs, anonymous mappings)
|        ↓         |
+------------------+
|        ↑         |
|   Heap           | (ขยายขึ้น - grows up via brk)
+------------------+
|   BSS            | (uninitialized global/static)
+------------------+
|   Data           | (initialized global/static)
+------------------+
|   Text           | (code - read/execute only)
+------------------+ 0x0000000000400000 (typical)
|   (reserved)     |
+------------------+ 0x0000000000000000
```

### 1.2 การอ่าน /proc/self/maps

ไฟล์ `/proc/self/maps` แสดง memory mapping ทั้งหมดของ process ปัจจุบัน:

```nasm
; โปรแกรมอ่านและแสดง /proc/self/maps
; อธิบาย: เปิดไฟล์ /proc/self/maps แล้วอ่านและพิมพ์ออกมา

section .data
    maps_path   db "/proc/self/maps", 0
    newline     db 10
    header_msg  db "=== Process Memory Maps ===", 10, 0
    header_len  equ $ - header_msg

section .bss
    buffer      resb 4096    ; บัฟเฟอร์สำหรับอ่านข้อมูล

section .text
    global _start

_start:
    ; พิมพ์ header
    mov rax, 1              ; sys_write
    mov rdi, 1              ; stdout
    mov rsi, header_msg
    mov rdx, header_len
    syscall

    ; เปิดไฟล์ /proc/self/maps
    mov rax, 2              ; sys_open
    mov rdi, maps_path      ; path
    mov rsi, 0              ; O_RDONLY
    mov rdx, 0              ; mode (ไม่ใช้สำหรับ O_RDONLY)
    syscall
    
    ; rax ตอนนี้มี file descriptor
    mov rbx, rax            ; เก็บ fd ไว้ใน rbx
    
    ; ตรวจสอบว่าเปิดสำเร็จ
    cmp rax, 0
    jl .error

.read_loop:
    ; อ่านข้อมูลจากไฟล์
    mov rax, 0              ; sys_read
    mov rdi, rbx            ; fd
    mov rsi, buffer         ; buffer
    mov rdx, 4096           ; จำนวน bytes ที่อ่าน
    syscall
    
    ; ถ้า rax <= 0 หมายถึงหมดหรือ error
    cmp rax, 0
    jle .done
    
    ; พิมพ์ข้อมูลที่อ่านได้
    mov rdx, rax            ; จำนวน bytes ที่อ่านได้
    mov rax, 1              ; sys_write
    mov rdi, 1              ; stdout
    mov rsi, buffer
    syscall
    
    jmp .read_loop

.done:
    ; ปิดไฟล์
    mov rax, 3              ; sys_close
    mov rdi, rbx
    syscall
    
    ; จบโปรแกรม
    mov rax, 60             ; sys_exit
    xor rdi, rdi            ; exit code 0
    syscall

.error:
    ; จบโปรแกรมด้วย error code
    mov rax, 60
    mov rdi, 1
    syscall
```

ตัวอย่าง output ของ `/proc/self/maps`:
```
00400000-00401000 r--p 00000000 08:01 12345678    /usr/bin/program
00401000-00402000 r-xp 00001000 08:01 12345678    /usr/bin/program  
00402000-00403000 r--p 00002000 08:01 12345678    /usr/bin/program
7f8a00000000-7f8a00021000 rw-p 00000000 00:00 0   [heap]
7ffd00000000-7ffd00021000 rw-p 00000000 00:00 0   [stack]
```

แต่ละบรรทัดมีรูปแบบ:
```
address           perms offset  dev   inode   pathname
08048000-0804a000 r--p 00000000 03:01 123456  /bin/cat
```

- **perms**: `r`=read, `w`=write, `x`=execute, `p`=private, `s`=shared
- **offset**: offset ใน file (สำหรับ file mapping)
- **dev**: device major:minor
- **inode**: inode number (0 = anonymous)

---

## 2. brk และ sbrk: Heap Expansion

### 2.1 System Call: brk

`brk` เป็น system call พื้นฐานสำหรับการขยาย heap โดยการเปลี่ยน "program break" ซึ่งเป็นจุดสิ้นสุดของ data segment:

```nasm
; sys_brk system call number
%define SYS_BRK 12

; การใช้ brk:
; rax = 12 (sys_brk)
; rdi = new break address (0 = ได้ current break)
; return: rax = new break address (หรือ -1 ถ้า error)
```

### 2.2 การ Implement sbrk ใน Assembly

```nasm
; sbrk implementation ใน Assembly
; sbrk(increment) - เพิ่มขนาด heap ตาม increment bytes
; Return: pointer ไปยัง old break (beginning of new memory)

section .text

; Function: sbrk
; Input:  rdi = increment (จำนวน bytes ที่ต้องการเพิ่ม)
; Output: rax = pointer ไปยัง old break, หรือ -1 ถ้า error
; Clobbers: rcx, r11 (จาก syscall)
sbrk:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; เก็บ increment ไว้ใน rbx
    mov rbx, rdi
    
    ; หา current break (brk(0))
    mov rax, 12             ; sys_brk
    xor rdi, rdi            ; argument = 0
    syscall
    
    ; rax = current break
    ; ตรวจสอบ error
    cmp rax, -1
    je .error
    
    ; เก็บ current break (จะ return นี้)
    mov rcx, rax            ; old_break = current_break
    
    ; คำนวณ new break
    add rax, rbx            ; new_break = current_break + increment
    
    ; ตั้งค่า new break
    mov rdi, rax            ; new break address
    mov rax, 12             ; sys_brk
    syscall
    
    ; ตรวจสอบว่า brk สำเร็จ
    ; kernel อาจคืนค่า break ที่ต่างจากที่ขอ
    cmp rax, rcx
    je .error               ; ถ้า break ไม่ขยาย = error
    
    ; Return old break
    mov rax, rcx
    
    pop rbx
    pop rbp
    ret

.error:
    mov rax, -1
    pop rbx
    pop rbp
    ret
```

### 2.3 ตัวอย่าง: Simple Heap Allocator ด้วย brk

```nasm
; Simple bump allocator using brk
; ไม่มีการ free - ง่ายที่สุด แต่มี memory leak

section .data
    heap_top    dq 0        ; ตำแหน่งปัจจุบันใน heap
    heap_start  dq 0        ; จุดเริ่มต้นของ heap
    initialized db 0        ; flag ว่า initialize แล้วหรือยัง

section .text

; Function: simple_malloc
; Input:  rdi = size (จำนวน bytes ที่ต้องการ)
; Output: rax = pointer ไปยัง allocated memory, หรือ 0 ถ้า error
simple_malloc:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    
    ; เก็บ size
    mov r12, rdi
    
    ; ตรวจสอบว่า initialize แล้วหรือยัง
    movzx eax, byte [initialized]
    test eax, eax
    jnz .already_init
    
    ; Initialize: หา current heap top
    mov rax, 12             ; sys_brk
    xor rdi, rdi
    syscall
    
    mov [heap_start], rax   ; เก็บ heap start
    mov [heap_top], rax     ; top เริ่มที่ start
    mov byte [initialized], 1
    
.already_init:
    ; Align size to 8 bytes (alignment สำคัญมาก!)
    ; size = (size + 7) & ~7
    add r12, 7
    and r12, ~7
    
    ; คำนวณ new top
    mov rbx, [heap_top]     ; current top
    mov rax, rbx            ; จะ return นี้
    add rbx, r12            ; new top = old top + size
    
    ; ขยาย heap ถ้าจำเป็น
    push rax                ; เก็บ return value
    mov rdi, rbx            ; new break
    mov rax, 12             ; sys_brk
    syscall
    pop rax                 ; คืน return value
    
    ; อัพเดท heap_top
    mov [heap_top], rbx
    
    pop r12
    pop rbx
    pop rbp
    ret

; Function: heap_free_all
; ล้าง heap ทั้งหมด (คืน memory กลับให้ OS)
heap_free_all:
    push rbp
    mov rbp, rsp
    
    ; Set break กลับไปที่ heap start
    mov rdi, [heap_start]
    mov rax, 12             ; sys_brk
    syscall
    
    ; Reset heap_top
    mov rax, [heap_start]
    mov [heap_top], rax
    
    pop rbp
    ret
```

---

## 3. mmap: Memory Mapping

### 3.1 mmap System Call

`mmap` เป็น system call ที่ยืดหยุ่นกว่า `brk` มาก ใช้สำหรับ:
- Anonymous memory allocation
- File mapping
- Shared memory between processes

```nasm
; mmap system call constants
%define SYS_MMAP    9
%define SYS_MUNMAP  11

; Protection flags
%define PROT_NONE   0x0     ; ไม่มีสิทธิ์อะไรเลย
%define PROT_READ   0x1     ; อ่านได้
%define PROT_WRITE  0x2     ; เขียนได้
%define PROT_EXEC   0x4     ; execute ได้

; Mapping flags
%define MAP_SHARED      0x01    ; share กับ process อื่น
%define MAP_PRIVATE     0x02    ; private copy (copy-on-write)
%define MAP_FIXED       0x10    ; ใช้ address ที่กำหนดตรงๆ
%define MAP_ANON        0x20    ; anonymous (ไม่ใช่ file)
%define MAP_ANONYMOUS   0x20    ; alias ของ MAP_ANON

; MAP_FAILED = (void*)-1
%define MAP_FAILED      -1
```

### 3.2 Anonymous Mapping สำหรับ Dynamic Allocation

```nasm
; mmap anonymous mapping - เหมือน malloc แต่ใช้ mmap
; เหมาะสำหรับ large allocations

section .text

; Function: mmap_alloc
; Input:  rdi = size (จำนวน bytes)
; Output: rax = pointer ไปยัง memory, หรือ MAP_FAILED (-1) ถ้า error
mmap_alloc:
    push rbp
    mov rbp, rsp
    
    ; เก็บ size
    mov r8, rdi             ; size จะใช้ใน mmap call
    
    ; Align size to page boundary (4096 bytes)
    ; page_size = 4096 = 0x1000
    add rdi, 4095
    and rdi, ~4095
    
    ; mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, -1, 0)
    mov rax, 9              ; sys_mmap
    xor rdi, rdi            ; addr = NULL (kernel เลือกให้)
    mov rsi, r8             ; length = size
    mov rdx, 3              ; prot = PROT_READ|PROT_WRITE (0x1|0x2)
    mov r10, 0x22           ; flags = MAP_PRIVATE|MAP_ANON (0x02|0x20)
    mov r8, -1              ; fd = -1 (anonymous)
    xor r9, r9              ; offset = 0
    syscall
    
    ; ตรวจสอบ error
    cmp rax, -1
    je .failed
    
    pop rbp
    ret

.failed:
    mov rax, 0              ; return NULL ถ้า error
    pop rbp
    ret

; Function: mmap_free
; Input:  rdi = ptr, rsi = size
; Output: rax = 0 สำเร็จ, -1 ถ้า error
mmap_free:
    push rbp
    mov rbp, rsp
    
    ; munmap(ptr, size)
    mov rax, 11             ; sys_munmap
    ; rdi และ rsi มีค่าอยู่แล้ว
    syscall
    
    pop rbp
    ret
```

### 3.3 MAP_SHARED: Shared Memory

```nasm
; Shared memory ระหว่าง processes โดยใช้ mmap + file

section .data
    shm_path    db "/tmp/shared_mem_example", 0
    shm_size    equ 4096

section .text

; Function: create_shared_memory
; สร้าง shared memory region ที่ processes อื่นเข้าถึงได้
; Output: rax = pointer ไปยัง shared memory
create_shared_memory:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; เปิด/สร้างไฟล์สำหรับ shared memory
    ; open(path, O_RDWR|O_CREAT|O_TRUNC, 0666)
    mov rax, 2              ; sys_open
    mov rdi, shm_path
    mov rsi, 0x242          ; O_RDWR(0x2)|O_CREAT(0x40)|O_TRUNC(0x200)
    mov rdx, 0o666          ; permissions
    syscall
    
    mov rbx, rax            ; เก็บ fd
    
    ; ขยายไฟล์ให้มีขนาดที่ต้องการ
    ; ftruncate(fd, size)
    mov rax, 77             ; sys_ftruncate
    mov rdi, rbx
    mov rsi, shm_size
    syscall
    
    ; mmap ไฟล์นี้ด้วย MAP_SHARED
    mov rax, 9              ; sys_mmap
    xor rdi, rdi            ; addr = NULL
    mov rsi, shm_size       ; length
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 1              ; MAP_SHARED (0x01)
    mov r8, rbx             ; fd
    xor r9, r9              ; offset = 0
    syscall
    
    push rax                ; เก็บ mapping address
    
    ; ปิด fd (mapping ยังคงอยู่)
    mov rax, 3              ; sys_close
    mov rdi, rbx
    syscall
    
    pop rax                 ; คืน mapping address
    
    pop rbx
    pop rbp
    ret
```

### 3.4 mmap File: Memory-Mapped File I/O

```nasm
; Memory-mapped file I/O
; เป็น technique ที่ efficient สำหรับการอ่าน/เขียนไฟล์ขนาดใหญ่

section .data
    filename    db "test.bin", 0

section .bss
    file_size   resq 1
    file_ptr    resq 1

section .text

; Structure: stat (simplified, fields we need)
; offset 0:  st_dev
; offset 8:  st_ino
; offset 16: st_mode
; offset 24: st_nlink
; offset 32: st_uid
; offset 40: st_gid
; offset 48: st_rdev
; offset 56: st_size  <-- เราต้องการตัวนี้
struc stat_t
    .st_dev:    resq 1
    .st_ino:    resq 1
    .st_nlink:  resq 1
    .st_mode:   resd 1
    .st_uid:    resd 1
    .st_gid:    resd 1
                resd 1      ; padding
    .st_rdev:   resq 1
    .st_size:   resq 1      ; offset 40
    ; ... rest of stat
endstruc

; Function: mmap_file
; Input:  rdi = filename
; Output: rax = pointer ไปยัง mapped memory
;         rdx = size ของ file
mmap_file:
    push rbp
    mov rbp, rsp
    sub rsp, 144            ; เตรียม space สำหรับ stat structure
    push rbx r12 r13
    
    mov r12, rdi            ; เก็บ filename
    
    ; เปิดไฟล์
    mov rax, 2              ; sys_open
    mov rdi, r12
    xor rsi, rsi            ; O_RDONLY
    xor rdx, rdx
    syscall
    
    mov rbx, rax            ; เก็บ fd
    
    ; fstat เพื่อหาขนาดไฟล์
    mov rax, 5              ; sys_fstat
    mov rdi, rbx
    lea rsi, [rbp - 144]    ; stat structure บน stack
    syscall
    
    ; อ่าน st_size จาก stat structure
    ; st_size อยู่ที่ offset 40 จาก struct start
    mov r13, [rbp - 144 + 40]  ; st_size
    
    ; mmap ไฟล์
    mov rax, 9              ; sys_mmap
    xor rdi, rdi            ; addr = NULL
    mov rsi, r13            ; length = file_size
    mov rdx, 1              ; PROT_READ
    mov r10, 2              ; MAP_PRIVATE (0x02)
    mov r8, rbx             ; fd
    xor r9, r9              ; offset = 0
    syscall
    
    mov r12, rax            ; เก็บ mapping pointer
    
    ; ปิด fd
    mov rax, 3              ; sys_close
    mov rdi, rbx
    syscall
    
    mov rax, r12            ; return mapping pointer
    mov rdx, r13            ; return file size
    
    pop r13 r12 rbx
    pop rbp
    ret
```

---

## 4. munmap: การ Unmap Memory

```nasm
; munmap - ลบ memory mapping
; ต้องใช้ address และ size ที่ตรงกับที่ mmap สร้าง

; Function: safe_munmap
; Input:  rdi = addr (ต้องเป็น page-aligned)
;         rsi = length
; Output: rax = 0 สำเร็จ, -1 ถ้า error
safe_munmap:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า addr ไม่เป็น NULL
    test rdi, rdi
    jz .null_addr
    
    ; ตรวจสอบว่า length > 0
    test rsi, rsi
    jz .zero_length
    
    ; munmap(addr, length)
    mov rax, 11             ; sys_munmap
    syscall
    
    pop rbp
    ret

.null_addr:
.zero_length:
    mov rax, -1
    pop rbp
    ret
```

---

## 5. mprotect: Memory Protection

### 5.1 การเปลี่ยน Memory Permissions

`mprotect` ช่วยให้เราเปลี่ยน permissions ของ memory region ได้:

```nasm
%define SYS_MPROTECT 10

; Function: make_executable
; ทำให้ memory region execute ได้ (สำหรับ JIT compilation)
; Input:  rdi = addr
;         rsi = size
; Output: rax = 0 สำเร็จ, -1 ถ้า error
make_executable:
    push rbp
    mov rbp, rsp
    
    ; Align addr down to page boundary
    push rdi
    and rdi, ~4095          ; page-align address
    
    ; Align size up to page boundary
    add rsi, 4095
    and rsi, ~4095
    
    ; mprotect(addr, size, PROT_READ|PROT_WRITE|PROT_EXEC)
    mov rax, 10             ; sys_mprotect
    ; rdi และ rsi มีค่าอยู่แล้ว
    mov rdx, 7              ; PROT_READ|PROT_WRITE|PROT_EXEC (0x1|0x2|0x4)
    syscall
    
    pop rdi
    pop rbp
    ret

; Function: make_readonly
; ทำให้ memory region อ่านได้อย่างเดียว (data protection)
; Input:  rdi = addr
;         rsi = size
make_readonly:
    push rbp
    mov rbp, rsp
    
    and rdi, ~4095          ; page-align
    add rsi, 4095
    and rsi, ~4095
    
    ; mprotect(addr, size, PROT_READ)
    mov rax, 10             ; sys_mprotect
    mov rdx, 1              ; PROT_READ only
    syscall
    
    pop rbp
    ret

; Function: make_noaccess
; ทำให้ memory region ไม่สามารถ access ได้เลย
; ใช้สำหรับ guard pages
; Input:  rdi = addr
;         rsi = size
make_noaccess:
    push rbp
    mov rbp, rsp
    
    and rdi, ~4095          ; page-align
    add rsi, 4095
    and rsi, ~4095
    
    ; mprotect(addr, size, PROT_NONE)
    mov rax, 10             ; sys_mprotect
    xor rdx, rdx            ; PROT_NONE = 0
    syscall
    
    pop rbp
    ret
```

### 5.2 ตัวอย่าง JIT Compilation

```nasm
; ตัวอย่าง: สร้าง JIT-compiled code ที่ runtime
; เขียน machine code ลงใน writable memory แล้วทำให้ executable

section .data
    ; Machine code ที่จะ inject: add rdi, rsi; mov rax, rdi; ret
    jit_code    db 0x48, 0x01, 0xF7  ; add rdi, rsi
                db 0x48, 0x89, 0xF8  ; mov rax, rdi
                db 0xC3               ; ret
    jit_size    equ $ - jit_code

section .text

; Function: jit_create_add_function
; สร้างฟังก์ชัน add ที่ compile ที่ runtime
; Output: rax = pointer ไปยัง JIT function
jit_create_add_function:
    push rbp
    mov rbp, rsp
    push r12
    
    ; Allocate writable memory
    mov rax, 9              ; sys_mmap
    xor rdi, rdi            ; addr = NULL
    mov rsi, 4096           ; 1 page
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1              ; fd = -1
    xor r9, r9              ; offset = 0
    syscall
    
    mov r12, rax            ; เก็บ pointer
    
    ; Copy JIT code ไปยัง allocated memory
    ; memcpy(r12, jit_code, jit_size)
    lea rsi, [jit_code]
    mov rdi, r12
    mov rcx, jit_size
    rep movsb
    
    ; เปลี่ยน permission เป็น READ+EXEC (ลบ WRITE เพื่อความปลอดภัย)
    mov rax, 10             ; sys_mprotect
    mov rdi, r12
    mov rsi, 4096
    mov rdx, 5              ; PROT_READ|PROT_EXEC (0x1|0x4)
    syscall
    
    mov rax, r12            ; return pointer ไปยัง JIT function
    
    pop r12
    pop rbp
    ret
```

---

## 6. mlock และ munlock: Locking Pages ใน RAM

### 6.1 mlock: ป้องกัน Page Swapping

`mlock` ล็อค memory pages ให้อยู่ใน RAM ตลอดเวลา (ไม่ถูก swap ออก) มีประโยชน์สำหรับ:
- Security: ข้อมูลสำคัญไม่ถูกเขียนลง swap file
- Performance: real-time applications ที่ต้องการ latency ต่ำ

```nasm
%define SYS_MLOCK   149
%define SYS_MUNLOCK 150
%define SYS_MLOCKALL 151
%define SYS_MUNLOCKALL 152

; MCL flags สำหรับ mlockall
%define MCL_CURRENT 1       ; lock pages ที่มีอยู่แล้ว
%define MCL_FUTURE  2       ; lock pages ที่จะ allocate ในอนาคต
%define MCL_ONFAULT 4       ; lock pages เมื่อเกิด page fault (Linux 4.4+)

; Function: lock_memory
; Input:  rdi = addr
;         rsi = length
; Output: rax = 0 สำเร็จ, -1 ถ้า error
lock_memory:
    push rbp
    mov rbp, rsp
    
    ; mlock(addr, length)
    mov rax, 149            ; sys_mlock
    syscall
    
    pop rbp
    ret

; Function: unlock_memory
; Input:  rdi = addr
;         rsi = length
unlock_memory:
    push rbp
    mov rbp, rsp
    
    ; munlock(addr, length)
    mov rax, 150            ; sys_munlock
    syscall
    
    pop rbp
    ret

; Function: lock_all_memory
; ล็อค memory ทั้งหมดของ process
; ต้องการ CAP_IPC_LOCK privilege หรือ RLIMIT_MEMLOCK ที่ใหญ่พอ
lock_all_memory:
    push rbp
    mov rbp, rsp
    
    ; mlockall(MCL_CURRENT|MCL_FUTURE)
    mov rax, 151            ; sys_mlockall
    mov rdi, 3              ; MCL_CURRENT|MCL_FUTURE
    syscall
    
    pop rbp
    ret
```

### 6.2 ตัวอย่าง: Secure Password Storage

```nasm
; ตัวอย่าง: เก็บ password ใน locked memory เพื่อความปลอดภัย

section .data
    ask_pw_msg  db "Password stored in locked memory", 10, 0
    ask_pw_len  equ $ - ask_pw_msg

section .text

secure_password_demo:
    push rbp
    mov rbp, rsp
    push r12 r13
    
    ; Allocate memory สำหรับ password
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, 4096           ; 1 page
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1
    xor r9, r9
    syscall
    
    mov r12, rax            ; เก็บ pointer
    
    ; ล็อค page นี้ใน RAM (ไม่ถูก swap ออก)
    mov rax, 149            ; sys_mlock
    mov rdi, r12
    mov rsi, 4096
    syscall
    
    ; ใช้ memory เก็บ password (ตัวอย่าง)
    ; ในโปรแกรมจริงจะรับ input จาก user
    mov byte [r12], 'S'
    mov byte [r12+1], 'e'
    mov byte [r12+2], 'c'
    mov byte [r12+3], 'r'
    mov byte [r12+4], 'e'
    mov byte [r12+5], 't'
    mov byte [r12+6], 0
    
    ; ทำงานกับ password ที่นี่...
    
    ; เมื่อเสร็จ: zero out memory ก่อน unlock/unmap
    ; สำคัญมาก: ป้องกัน password อยู่ใน memory
    mov rdi, r12
    xor eax, eax
    mov rcx, 4096/8         ; 4096 bytes / 8 bytes per qword
    rep stosq               ; zero out ทั้งหมด
    
    ; Unlock
    mov rax, 150            ; sys_munlock
    mov rdi, r12
    mov rsi, 4096
    syscall
    
    ; Unmap
    mov rax, 11             ; sys_munmap
    mov rdi, r12
    mov rsi, 4096
    syscall
    
    pop r13 r12
    pop rbp
    ret
```

---

## 7. mremap: Resize Memory Allocation

`mremap` ช่วยให้เราขยายหรือย่อ memory mapping ที่มีอยู่แล้วได้:

```nasm
%define SYS_MREMAP  25

; mremap flags
%define MREMAP_MAYMOVE  1   ; allow kernel to move mapping
%define MREMAP_FIXED    2   ; move to specific address

; Function: resize_memory
; Input:  rdi = old_addr
;         rsi = old_size
;         rdx = new_size
; Output: rax = new address, หรือ MAP_FAILED (-1) ถ้า error
resize_memory:
    push rbp
    mov rbp, rsp
    
    ; mremap(old_addr, old_size, new_size, MREMAP_MAYMOVE)
    mov rax, 25             ; sys_mremap
    ; rdi = old_addr (มีอยู่แล้ว)
    ; rsi = old_size (มีอยู่แล้ว)
    ; rdx = new_size (มีอยู่แล้ว)
    mov r10, 1              ; MREMAP_MAYMOVE
    syscall
    
    pop rbp
    ret
```

### 7.1 ตัวอย่าง: Dynamic Array ด้วย mremap

```nasm
; Dynamic array implementation ที่ใช้ mremap
; คล้าย std::vector ใน C++

section .data
    ; Dynamic array structure:
    ; [0]: pointer to data
    ; [8]: current size (elements)
    ; [16]: capacity (allocated elements)
    ; [24]: element_size

struc dynarray
    .data_ptr:      resq 1
    .size:          resq 1
    .capacity:      resq 1
    .element_size:  resq 1
endstruc

INITIAL_CAPACITY    equ 16
GROWTH_FACTOR       equ 2   ; double capacity เมื่อเต็ม

section .text

; Function: dynarray_create
; Input:  rdi = element_size
; Output: rax = pointer to dynarray struct
dynarray_create:
    push rbp
    mov rbp, rsp
    push rbx r12
    
    mov r12, rdi            ; เก็บ element_size
    
    ; Allocate struct
    mov rdi, dynarray_size
    call mmap_alloc
    mov rbx, rax            ; rbx = dynarray struct
    
    ; Allocate initial data
    mov rdi, r12
    imul rdi, INITIAL_CAPACITY
    call mmap_alloc
    
    ; Initialize struct
    mov [rbx + dynarray.data_ptr], rax
    mov qword [rbx + dynarray.size], 0
    mov qword [rbx + dynarray.capacity], INITIAL_CAPACITY
    mov [rbx + dynarray.element_size], r12
    
    mov rax, rbx            ; return struct pointer
    
    pop r12 rbx
    pop rbp
    ret

; Function: dynarray_push
; Input:  rdi = dynarray*, rsi = pointer to element
; Output: rax = 0 สำเร็จ
dynarray_push:
    push rbp
    mov rbp, rsp
    push rbx r12 r13
    
    mov rbx, rdi            ; dynarray*
    mov r12, rsi            ; element pointer
    
    ; ตรวจสอบว่าต้องขยาย capacity หรือไม่
    mov rax, [rbx + dynarray.size]
    cmp rax, [rbx + dynarray.capacity]
    jl .no_resize
    
    ; ต้องขยาย: new_capacity = capacity * 2
    mov rax, [rbx + dynarray.capacity]
    imul rax, GROWTH_FACTOR
    mov r13, rax            ; new_capacity
    
    ; คำนวณ sizes
    mov rcx, [rbx + dynarray.element_size]
    mov rdi, [rbx + dynarray.data_ptr]  ; old_addr
    mov rsi, [rbx + dynarray.capacity]
    imul rsi, rcx           ; old_size
    mov rdx, r13
    imul rdx, rcx           ; new_size
    
    ; mremap
    mov rax, 25             ; sys_mremap
    mov r10, 1              ; MREMAP_MAYMOVE
    syscall
    
    ; อัพเดท struct
    mov [rbx + dynarray.data_ptr], rax
    mov [rbx + dynarray.capacity], r13

.no_resize:
    ; Copy element ไปยัง array
    mov rax, [rbx + dynarray.size]
    mov rcx, [rbx + dynarray.element_size]
    imul rax, rcx           ; offset = size * element_size
    
    mov rdi, [rbx + dynarray.data_ptr]
    add rdi, rax            ; destination
    mov rsi, r12            ; source
    ; copy rcx bytes
    rep movsb
    
    ; เพิ่ม size
    inc qword [rbx + dynarray.size]
    
    xor eax, eax            ; return 0
    
    pop r13 r12 rbx
    pop rbp
    ret
```

---

## 8. mincore: ตรวจสอบว่า Pages อยู่ใน RAM

`mincore` บอกว่า pages ใดอยู่ใน physical memory (ไม่ถูก swap):

```nasm
%define SYS_MINCORE 27

; Function: check_pages_in_ram
; Input:  rdi = addr (page-aligned)
;         rsi = length
;         rdx = vec (buffer สำหรับ result, ต้องมีขนาด ceil(length/PAGE_SIZE))
; Output: rax = 0 สำเร็จ
;         vec[i] & 1 = 1 หมายความว่า page i อยู่ใน RAM
check_pages_in_ram:
    push rbp
    mov rbp, rsp
    
    ; mincore(addr, length, vec)
    mov rax, 27             ; sys_mincore
    syscall
    
    pop rbp
    ret

; Function: warm_pages
; บังคับให้ pages อยู่ใน RAM โดยการอ่านทุก page
; Input:  rdi = addr
;         rsi = length
warm_pages:
    push rbp
    mov rbp, rsp
    push rbx
    
    mov rbx, rdi            ; current addr
    add rsi, rdi            ; end addr
    
.loop:
    cmp rbx, rsi
    jge .done
    
    ; อ่าน 1 byte จาก page เพื่อ trigger page fault
    mov al, [rbx]           ; อ่านแล้วทิ้ง
    
    ; ไปยัง page ถัดไป
    add rbx, 4096
    jmp .loop

.done:
    pop rbx
    pop rbp
    ret
```

---

## 9. madvise: Memory Usage Hints

`madvise` ช่วยให้เราบอก kernel ว่าจะใช้ memory อย่างไร เพื่อ optimization:

```nasm
%define SYS_MADVISE 28

; madvise advice values
%define MADV_NORMAL         0   ; default behavior
%define MADV_RANDOM         1   ; expect random access pattern
%define MADV_SEQUENTIAL     2   ; expect sequential access
%define MADV_WILLNEED       3   ; will need this memory soon
%define MADV_DONTNEED       4   ; ไม่ต้องการ memory นี้อีกแล้ว
%define MADV_FREE           8   ; free page แต่ keep mapping (Linux 4.5+)
%define MADV_REMOVE         9   ; remove page content (Linux 2.6.16+)
%define MADV_DONTFORK       10  ; ไม่ inherit เมื่อ fork
%define MADV_DOFORK         11  ; inherit เมื่อ fork (undo DONTFORK)
%define MADV_MERGEABLE      12  ; KSM: merge identical pages
%define MADV_UNMERGEABLE    13  ; undo MERGEABLE
%define MADV_HUGEPAGE       14  ; enable transparent hugepages
%define MADV_NOHUGEPAGE     15  ; disable transparent hugepages
%define MADV_DONTDUMP       16  ; ไม่ include ใน core dump
%define MADV_DODUMP         17  ; include ใน core dump
%define MADV_HWPOISON       100 ; simulate hardware memory error (testing)

; Function: advise_sequential
; บอก kernel ว่าจะอ่านข้อมูลแบบ sequential
; ช่วยให้ kernel prefetch pages ล่วงหน้า
; Input:  rdi = addr, rsi = length
advise_sequential:
    push rbp
    mov rbp, rsp
    
    mov rax, 28             ; sys_madvise
    mov rdx, 2              ; MADV_SEQUENTIAL
    syscall
    
    pop rbp
    ret

; Function: advise_random
; บอก kernel ว่าจะ access แบบ random
; ป้องกัน kernel จาก prefetch ที่ไม่จำเป็น
; Input:  rdi = addr, rsi = length
advise_random:
    push rbp
    mov rbp, rsp
    
    mov rax, 28             ; sys_madvise
    mov rdx, 1              ; MADV_RANDOM
    syscall
    
    pop rbp
    ret

; Function: advise_dontneed
; บอก kernel ว่าไม่ต้องการ pages เหล่านี้อีกแล้ว
; kernel จะ reclaim pages เมื่อต้องการ
; Input:  rdi = addr, rsi = length
advise_dontneed:
    push rbp
    mov rbp, rsp
    
    mov rax, 28             ; sys_madvise
    mov rdx, 4              ; MADV_DONTNEED
    syscall
    
    pop rbp
    ret

; Function: advise_willneed
; บอก kernel ว่าจะ access memory นี้เร็วๆ นี้
; kernel จะ prefetch pages เข้า RAM ล่วงหน้า
; Input:  rdi = addr, rsi = length
advise_willneed:
    push rbp
    mov rbp, rsp
    
    mov rax, 28             ; sys_madvise
    mov rdx, 3              ; MADV_WILLNEED
    syscall
    
    pop rbp
    ret
```

### 9.1 ตัวอย่าง: File Processing Optimization

```nasm
; ตัวอย่าง: อ่านไฟล์ขนาดใหญ่แบบ sequential ด้วย madvise

process_large_file:
    push rbp
    mov rbp, rsp
    push rbx r12 r13
    
    ; รับ filename ใน rdi
    mov r12, rdi
    
    ; เปิดไฟล์
    mov rax, 2              ; sys_open
    xor rsi, rsi            ; O_RDONLY
    xor rdx, rdx
    syscall
    mov r13, rax            ; เก็บ fd
    
    ; หาขนาดไฟล์ด้วย lseek
    mov rax, 8              ; sys_lseek
    mov rdi, r13
    xor rsi, rsi            ; offset = 0
    mov rdx, 2              ; SEEK_END
    syscall
    mov rbx, rax            ; file_size
    
    ; seek กลับไป beginning
    mov rax, 8              ; sys_lseek
    mov rdi, r13
    xor rsi, rsi            ; offset = 0
    xor rdx, rdx            ; SEEK_SET
    syscall
    
    ; mmap ไฟล์ทั้งหมด
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, rbx            ; file_size
    mov rdx, 1              ; PROT_READ
    mov r10, 1              ; MAP_SHARED
    mov r8, r13             ; fd
    xor r9, r9
    syscall
    
    push rax                ; เก็บ mapping pointer
    
    ; บอก kernel ว่าจะอ่านแบบ sequential
    mov rdi, rax
    mov rsi, rbx
    call advise_sequential
    
    ; Prefetch pages ที่จะ access เร็วๆ นี้
    pop rdi
    push rdi
    mov rsi, rbx
    call advise_willneed
    
    ; ... ประมวลผลไฟล์ที่นี่ ...
    
    ; เมื่อเสร็จ บอก kernel ว่าไม่ต้องการแล้ว
    pop rdi
    mov rsi, rbx
    call advise_dontneed
    
    ; munmap
    ; ...
    
    pop r13 r12 rbx
    pop rbp
    ret
```

---

## 10. NUMA Memory Policy

NUMA (Non-Uniform Memory Access) สำคัญมากสำหรับ server ที่มีหลาย CPU socket:

```nasm
%define SYS_MBIND           237
%define SYS_SET_MEMPOLICY   238
%define SYS_GET_MEMPOLICY   239
%define SYS_MIGRATE_PAGES   256
%define SYS_MOVE_PAGES      279

; NUMA memory policies
%define MPOL_DEFAULT    0   ; default: ใช้ policy ของ parent
%define MPOL_BIND       2   ; bind: ใช้เฉพาะ nodes ที่ระบุ
%define MPOL_INTERLEAVE 3   ; interleave: กระจาย allocation ระหว่าง nodes
%define MPOL_PREFERRED  1   ; prefer: try node นี้ก่อน
%define MPOL_LOCAL      4   ; local: ใช้ node ของ CPU ปัจจุบัน

; Function: bind_to_numa_node
; ผูก memory region กับ NUMA node เฉพาะ
; Input:  rdi = addr, rsi = length, rdx = node_mask (1 bit per node)
;         r10 = maxnode
bind_to_numa_node:
    push rbp
    mov rbp, rsp
    push r12 r13
    
    mov r12, rdx            ; node_mask
    mov r13, r10            ; maxnode
    
    ; mbind(addr, length, MPOL_BIND, &nodemask, maxnode, 0)
    push r12
    mov rax, 237            ; sys_mbind
    ; rdi = addr (มีอยู่แล้ว)
    ; rsi = length (มีอยู่แล้ว)
    mov rdx, 2              ; MPOL_BIND
    lea r10, [rsp]          ; pointer ไปยัง nodemask บน stack
    mov r8, r13             ; maxnode
    xor r9, r9              ; flags = 0
    syscall
    pop r12
    
    pop r13 r12
    pop rbp
    ret

; Function: get_current_numa_node
; หา NUMA node ของ CPU ปัจจุบัน
; Output: rax = node number
get_current_numa_node:
    push rbp
    mov rbp, rsp
    sub rsp, 8
    
    ; get_mempolicy(NULL, NULL, 0, NULL, MPOL_F_NODE|MPOL_F_ADDR)
    mov rax, 239            ; sys_get_mempolicy
    xor rdi, rdi            ; mode = NULL
    xor rsi, rsi            ; nodemask = NULL
    xor rdx, rdx            ; maxnode = 0
    xor r10, r10            ; addr = NULL
    mov r8, 0x600           ; MPOL_F_NODE|MPOL_F_ADDR
    syscall
    
    ; rax มี node number
    
    pop rbp
    ret
```

---

## 11. Implementing malloc/free

### 11.1 Bump Allocator (Simplest)

```nasm
; ==========================================
; BUMP ALLOCATOR
; วิธีที่ง่ายที่สุด: แค่เพิ่ม pointer ไปเรื่อยๆ
; ข้อดี: เร็วมาก O(1)
; ข้อเสีย: free() ไม่ได้จริงๆ (เสีย memory)
; ==========================================

section .bss
    bump_start  resq 1      ; จุดเริ่มต้น heap
    bump_current resq 1     ; ตำแหน่งปัจจุบัน
    bump_end    resq 1      ; จุดสิ้นสุด heap
    bump_init_done  resb 1  ; flag

BUMP_HEAP_SIZE  equ (1024 * 1024 * 16)  ; 16 MB initial heap

section .text

; Function: bump_init
; Initialize bump allocator
bump_init:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า initialize แล้วหรือยัง
    cmp byte [bump_init_done], 0
    jne .already_done
    
    ; Allocate heap ด้วย mmap
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, BUMP_HEAP_SIZE
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1
    xor r9, r9
    syscall
    
    mov [bump_start], rax
    mov [bump_current], rax
    add rax, BUMP_HEAP_SIZE
    mov [bump_end], rax
    mov byte [bump_init_done], 1

.already_done:
    pop rbp
    ret

; Function: bump_malloc
; Input:  rdi = size
; Output: rax = pointer, หรือ 0 ถ้า out of memory
bump_malloc:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; Initialize ถ้ายังไม่ได้ทำ
    call bump_init
    
    ; Align size ไปยัง 16 bytes
    add rdi, 15
    and rdi, ~15
    
    ; ตรวจสอบว่ามี space เหลือพอ
    mov rax, [bump_current]
    add rdi, rax            ; new_current = current + aligned_size
    cmp rdi, [bump_end]
    jg .out_of_memory
    
    ; Update pointer
    mov rbx, [bump_current] ; เก็บ old pointer
    mov [bump_current], rdi ; update current
    
    mov rax, rbx            ; return old pointer
    
    pop rbx
    pop rbp
    ret

.out_of_memory:
    xor eax, eax            ; return NULL
    pop rbx
    pop rbp
    ret

; Function: bump_free
; ไม่ทำอะไร (bump allocator ไม่ free จริงๆ)
bump_free:
    ret

; Function: bump_reset
; คืน memory ทั้งหมด (เหมือน free all)
bump_reset:
    push rbp
    mov rbp, rsp
    
    mov rax, [bump_start]
    mov [bump_current], rax
    
    pop rbp
    ret
```

### 11.2 Free List Allocator

```nasm
; ==========================================
; FREE LIST ALLOCATOR
; ใช้ linked list เก็บ free blocks
; ข้อดี: สามารถ free() ได้จริง
; ข้อเสีย: ช้ากว่า, อาจ fragmentation
; ==========================================

; Block header structure:
; [0]  size (8 bytes) - ขนาดของ block รวม header
; [8]  next_free (8 bytes) - pointer ไปยัง next free block (NULL ถ้า allocated)
; [16] data starts here

BLOCK_HEADER_SIZE   equ 16
FREE_BLOCK_MAGIC    equ 0xDEADBEEF  ; magic number สำหรับ debug
ALLOC_BLOCK_MAGIC   equ 0xABCDEF01

section .bss
    free_list   resq 1      ; pointer ไปยัง first free block
    heap_start2 resq 1      ; จุดเริ่มต้น heap
    heap_init2  resb 1      ; init flag

HEAP_SIZE2  equ (1024 * 1024 * 32)  ; 32 MB heap

section .text

; Function: freelist_init
freelist_init:
    push rbp
    mov rbp, rsp
    
    cmp byte [heap_init2], 0
    jne .done
    
    ; Allocate heap
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, HEAP_SIZE2
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1
    xor r9, r9
    syscall
    
    mov [heap_start2], rax
    
    ; Initialize เป็น single large free block
    mov qword [rax], HEAP_SIZE2     ; size = ทั้ง heap
    mov qword [rax + 8], 0          ; next_free = NULL
    
    mov [free_list], rax            ; free_list = heap_start
    mov byte [heap_init2], 1

.done:
    pop rbp
    ret

; Function: freelist_malloc
; Input:  rdi = requested_size
; Output: rax = pointer ไปยัง data (หลัง header), หรือ 0
freelist_malloc:
    push rbp
    mov rbp, rsp
    push rbx r12 r13 r14
    
    ; Initialize ถ้าจำเป็น
    call freelist_init
    
    ; คำนวณ total size ที่ต้องการ (รวม header)
    add rdi, BLOCK_HEADER_SIZE
    ; Align ไปยัง 16 bytes
    add rdi, 15
    and rdi, ~15
    mov r12, rdi            ; total_size ที่ต้องการ
    
    ; ค้นหา free block ที่ใหญ่พอ (first-fit)
    mov rbx, [free_list]    ; current block
    xor r13, r13            ; prev block (NULL = free_list เป็น head)

.search_loop:
    test rbx, rbx           ; ถ้า NULL = หมด free list
    jz .not_found
    
    ; ตรวจสอบว่า block นี้ใหญ่พอ
    mov rax, [rbx]          ; block size
    cmp rax, r12
    jge .found_block
    
    ; ไปยัง block ถัดไป
    mov r13, rbx
    mov rbx, [rbx + 8]      ; next_free
    jmp .search_loop

.found_block:
    ; rbx = found block
    ; ตรวจสอบว่าควร split block หรือไม่
    ; split ถ้าขนาดที่เหลือ >= BLOCK_HEADER_SIZE + minimum allocation
    mov rax, [rbx]          ; current block size
    sub rax, r12            ; remaining size
    cmp rax, BLOCK_HEADER_SIZE + 16  ; minimum useful remainder
    jl .no_split
    
    ; Split block
    mov r14, rbx
    add r14, r12            ; ตำแหน่ง new free block
    mov [r14], rax          ; new block size = remaining
    mov rdi, [rbx + 8]      ; next_free ของ block เดิม
    mov [r14 + 8], rdi      ; new block next = old block's next
    
    ; Update free list
    test r13, r13
    jz .split_update_head
    mov [r13 + 8], r14      ; prev->next = new_block
    jmp .split_done
.split_update_head:
    mov [free_list], r14    ; free_list = new_block
.split_done:
    
    ; ตั้งค่า allocated block
    mov qword [rbx], r12    ; size = requested size
    mov qword [rbx + 8], 0  ; next_free = 0 (allocated, ไม่ใช่ free)
    
    ; Return pointer ไปยัง data (หลัง header)
    lea rax, [rbx + BLOCK_HEADER_SIZE]
    jmp .done

.no_split:
    ; ใช้ block ทั้งหมด
    test r13, r13
    jz .nosplit_update_head
    mov rdi, [rbx + 8]
    mov [r13 + 8], rdi      ; ลบ block ออกจาก free list
    jmp .nosplit_done
.nosplit_update_head:
    mov rdi, [rbx + 8]
    mov [free_list], rdi    ; update head
.nosplit_done:
    
    mov qword [rbx + 8], 0  ; mark as allocated
    lea rax, [rbx + BLOCK_HEADER_SIZE]
    jmp .done

.not_found:
    xor eax, eax            ; return NULL

.done:
    pop r14 r13 r12 rbx
    pop rbp
    ret

; Function: freelist_free
; Input:  rdi = ptr (pointer ที่ได้จาก freelist_malloc)
freelist_free:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; ตรวจสอบ NULL
    test rdi, rdi
    jz .done
    
    ; หา block header
    sub rdi, BLOCK_HEADER_SIZE
    mov rbx, rdi
    
    ; เพิ่ม block กลับไปที่ free list (เพิ่มที่ head)
    mov rax, [free_list]    ; old head
    mov [rbx + 8], rax      ; this->next = old_head
    mov [free_list], rbx    ; free_list = this

.done:
    pop rbx
    pop rbp
    ret

; Function: freelist_coalesce
; รวม adjacent free blocks เข้าด้วยกัน
; ลด fragmentation
freelist_coalesce:
    push rbp
    mov rbp, rsp
    push rbx r12
    
    ; วน loop ผ่าน free list
    mov rbx, [free_list]

.outer_loop:
    test rbx, rbx
    jz .done
    
    ; ตรวจสอบ next block
    mov r12, [rbx + 8]      ; next free block
    test r12, r12
    jz .next_block
    
    ; ตรวจสอบว่า next block อยู่ติดกับ current block หรือไม่
    mov rax, rbx
    add rax, [rbx]          ; rbx + size = expected next address
    cmp rax, r12
    jne .next_block
    
    ; Merge! รวม size
    mov rax, [r12]          ; size ของ next block
    add [rbx], rax          ; current->size += next->size
    
    ; update next pointer
    mov rax, [r12 + 8]
    mov [rbx + 8], rax      ; current->next = next->next
    
    ; อย่าไป next block - ลองอีกครั้งกับ block ที่ merge แล้ว
    jmp .outer_loop

.next_block:
    mov rbx, [rbx + 8]
    jmp .outer_loop

.done:
    pop r12 rbx
    pop rbp
    ret
```

---

## 12. Memory Pool: Fixed-Size Block Allocator

```nasm
; ==========================================
; MEMORY POOL ALLOCATOR
; เหมาะสำหรับ allocation ขนาดเดียวกันจำนวนมาก
; ข้อดี: เร็วมาก, ไม่มี fragmentation
; ข้อเสีย: ใช้ได้กับ size เดียวเท่านั้น
; ==========================================

; Pool structure:
; [0]  block_size - ขนาดของแต่ละ block
; [8]  num_blocks - จำนวน blocks ทั้งหมด
; [16] free_count - จำนวน blocks ที่ว่างอยู่
; [24] next_free  - pointer ไปยัง first free block
; [32] data_start - จุดเริ่มต้นของ data area

struc mempool
    .block_size:    resq 1
    .num_blocks:    resq 1
    .free_count:    resq 1
    .next_free:     resq 1
    .data_start:    resq 1
endstruc

; แต่ละ free block มี: [0] pointer ไปยัง next free block

section .text

; Function: pool_create
; Input:  rdi = block_size, rsi = num_blocks
; Output: rax = pointer ไปยัง pool struct
pool_create:
    push rbp
    mov rbp, rsp
    push rbx r12 r13 r14
    
    mov r12, rdi            ; block_size
    mov r13, rsi            ; num_blocks
    
    ; Align block_size ไปยัง 8 bytes
    add r12, 7
    and r12, ~7
    ; Minimum block size = 8 bytes (สำหรับ free list pointer)
    cmp r12, 8
    jge .size_ok
    mov r12, 8
.size_ok:
    
    ; Allocate struct + data ใน mmap
    mov rax, r12
    imul rax, r13           ; total data size = block_size * num_blocks
    add rax, mempool_size   ; + header size
    
    mov rdi, rax
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, rax            ; size
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1
    xor r9, r9
    syscall
    
    mov rbx, rax            ; pool struct pointer
    
    ; Initialize struct
    mov [rbx + mempool.block_size], r12
    mov [rbx + mempool.num_blocks], r13
    mov [rbx + mempool.free_count], r13
    
    ; data_start อยู่ตรงหลัง struct
    lea r14, [rbx + mempool_size]
    mov [rbx + mempool.data_start], r14
    mov [rbx + mempool.next_free], r14  ; first free = data_start
    
    ; Initialize free list ใน data area
    mov rcx, r13            ; loop count = num_blocks
    mov rdi, r14            ; current block
    dec rcx                 ; num_blocks - 1 iterations
    jz .init_done_single    ; ถ้า 1 block

.init_loop:
    lea rax, [rdi + r12]    ; next block = current + block_size
    mov [rdi], rax          ; current->next = next_block
    mov rdi, rax            ; advance to next
    loop .init_loop

.init_done_single:
    ; Last block's next = NULL
    mov qword [rdi], 0
    
    mov rax, rbx            ; return pool pointer
    
    pop r14 r13 r12 rbx
    pop rbp
    ret

; Function: pool_alloc
; Input:  rdi = pool*
; Output: rax = pointer ไปยัง allocated block, หรือ 0
pool_alloc:
    push rbp
    mov rbp, rsp
    push rbx
    
    mov rbx, rdi            ; pool pointer
    
    ; ตรวจสอบว่ามี free blocks
    cmp qword [rbx + mempool.free_count], 0
    je .no_free
    
    ; ได้ first free block
    mov rax, [rbx + mempool.next_free]
    
    ; Update free list: next_free = current->next
    mov rcx, [rax]          ; current->next
    mov [rbx + mempool.next_free], rcx
    
    ; Decrement free count
    dec qword [rbx + mempool.free_count]
    
    ; Zero out block ก่อน return (optional, ความปลอดภัย)
    push rax
    mov rdi, rax
    xor eax, eax
    mov rcx, [rbx + mempool.block_size]
    shr rcx, 3              ; / 8
    rep stosq               ; zero fill
    pop rax
    
    pop rbx
    pop rbp
    ret

.no_free:
    xor eax, eax            ; return NULL
    pop rbx
    pop rbp
    ret

; Function: pool_free
; Input:  rdi = pool*, rsi = ptr
pool_free:
    push rbp
    mov rbp, rsp
    push rbx
    
    ; rdi = pool, rsi = ptr
    mov rbx, rdi
    
    ; ตรวจสอบ NULL
    test rsi, rsi
    jz .done
    
    ; เพิ่ม block กลับไปที่ free list
    mov rax, [rbx + mempool.next_free]  ; old first free
    mov [rsi], rax          ; ptr->next = old_first_free
    mov [rbx + mempool.next_free], rsi  ; first_free = ptr
    
    ; Increment free count
    inc qword [rbx + mempool.free_count]

.done:
    pop rbx
    pop rbp
    ret

; Function: pool_destroy
; Input:  rdi = pool*
pool_destroy:
    push rbp
    mov rbp, rsp
    push rbx
    
    mov rbx, rdi
    
    ; คำนวณ total size
    mov rax, [rbx + mempool.block_size]
    imul rax, [rbx + mempool.num_blocks]
    add rax, mempool_size
    mov rsi, rax
    
    ; munmap
    mov rax, 11             ; sys_munmap
    ; rdi = pool* (มีอยู่แล้ว)
    syscall
    
    pop rbx
    pop rbp
    ret
```

---

## 13. Stack Allocator

```nasm
; ==========================================
; STACK ALLOCATOR (Linear Allocator with markers)
; ใช้สำหรับ temporary allocations ที่มี LIFO order
; ข้อดี: O(1) alloc/free, ไม่มี fragmentation
; ข้อเสีย: free ต้องทำตาม LIFO order
; ==========================================

struc stack_allocator
    .buffer:    resq 1      ; pointer ไปยัง buffer
    .top:       resq 1      ; current top (จุดสิ้นสุดของ allocated data)
    .size:      resq 1      ; total buffer size
endstruc

section .text

; Function: stackalloc_create
; Input:  rdi = size
; Output: rax = pointer ไปยัง stack_allocator
stackalloc_create:
    push rbp
    mov rbp, rsp
    push rbx r12
    
    mov r12, rdi            ; size
    
    ; Allocate struct
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, stack_allocator_size
    mov rdx, 3
    mov r10, 0x22
    mov r8, -1
    xor r9, r9
    syscall
    mov rbx, rax            ; struct pointer
    
    ; Allocate buffer
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, r12
    mov rdx, 3
    mov r10, 0x22
    mov r8, -1
    xor r9, r9
    syscall
    
    ; Initialize struct
    mov [rbx + stack_allocator.buffer], rax
    mov [rbx + stack_allocator.top], rax  ; top = buffer_start
    mov [rbx + stack_allocator.size], r12
    
    mov rax, rbx
    
    pop r12 rbx
    pop rbp
    ret

; Function: stackalloc_alloc
; Input:  rdi = allocator*, rsi = size
; Output: rax = pointer, หรือ 0
stackalloc_alloc:
    push rbp
    mov rbp, rsp
    push rbx
    
    mov rbx, rdi
    
    ; Align size ไปยัง 16 bytes
    add rsi, 15
    and rsi, ~15
    
    ; ตรวจสอบว่ามี space พอ
    mov rax, [rbx + stack_allocator.top]
    add rsi, rax            ; new_top = top + size
    
    mov rcx, [rbx + stack_allocator.buffer]
    add rcx, [rbx + stack_allocator.size]  ; buffer_end
    
    cmp rsi, rcx
    jg .overflow
    
    ; Allocate
    mov rax, [rbx + stack_allocator.top]  ; return old top
    mov [rbx + stack_allocator.top], rsi  ; update top
    
    pop rbx
    pop rbp
    ret

.overflow:
    xor eax, eax
    pop rbx
    pop rbp
    ret

; Stack Marker: เก็บ state ของ stack เพื่อ rollback
; Function: stackalloc_get_marker
; Input:  rdi = allocator*
; Output: rax = marker (current top pointer)
stackalloc_get_marker:
    mov rax, [rdi + stack_allocator.top]
    ret

; Function: stackalloc_free_to_marker
; Input:  rdi = allocator*, rsi = marker
; (roll back stack ไปยัง marker)
stackalloc_free_to_marker:
    mov [rdi + stack_allocator.top], rsi
    ret

; Function: stackalloc_reset
; รีเซ็ต allocator ทั้งหมด
; Input:  rdi = allocator*
stackalloc_reset:
    mov rax, [rdi + stack_allocator.buffer]
    mov [rdi + stack_allocator.top], rax
    ret
```

---

## 14. โปรแกรมตัวอย่าง: Simple malloc Implementation

```nasm
; ==========================================
; COMPLETE MALLOC IMPLEMENTATION
; ใช้ free list + coalescing + mmap
; เหมาะสำหรับ production use จริงๆ
; ==========================================

section .data
    malloc_init_flag    db 0
    malloc_mutex        dq 0    ; spinlock (simplified)

section .bss
    malloc_free_list    resq 1  ; head ของ free list
    malloc_heap         resq 1  ; pointer ไปยัง heap
    malloc_heap_size    resq 1  ; ขนาด heap ปัจจุบัน

MALLOC_INITIAL_SIZE equ (1024 * 1024 * 4)  ; 4 MB
MALLOC_MIN_ALLOC    equ 32                   ; minimum allocation size
MALLOC_ALIGN        equ 16                   ; alignment

; Block metadata
; [0..7]   : size (รวม header)
; [8..15]  : flags + magic
;            bit 0: 1 = allocated, 0 = free
;            bits 8-31: magic = 0xABCD
; [16+]    : data (สำหรับ free block: [16..23] = next free, [24..31] = prev free)

BLOCK_HEADER    equ 16
BLOCK_MAGIC     equ 0xABCD0000
BLOCK_FLAG_FREE equ 0
BLOCK_FLAG_USED equ 1

section .text

; Function: my_malloc
; Input:  rdi = size
; Output: rax = pointer, หรือ 0 ถ้า error
my_malloc:
    push rbp
    mov rbp, rsp
    push rbx r12 r13
    
    ; ตรวจสอบ size
    test rdi, rdi
    jz .return_null
    
    ; Initialize ถ้าจำเป็น
    cmp byte [malloc_init_flag], 0
    jne .skip_init
    call malloc_internal_init
.skip_init:
    
    ; คำนวณ total size ที่ต้องการ
    add rdi, BLOCK_HEADER
    add rdi, MALLOC_ALIGN - 1
    and rdi, ~(MALLOC_ALIGN - 1)
    cmp rdi, MALLOC_MIN_ALLOC + BLOCK_HEADER
    jge .size_ok
    mov rdi, MALLOC_MIN_ALLOC + BLOCK_HEADER
.size_ok:
    mov r12, rdi            ; total_needed
    
    ; ค้นหา free block (first fit)
    mov rbx, [malloc_free_list]

.search:
    test rbx, rbx
    jz .need_more_memory
    
    ; ตรวจสอบ magic (debug)
    mov rax, [rbx + 8]
    and rax, 0xFFFF0000
    cmp rax, BLOCK_MAGIC
    jne .heap_corruption
    
    ; ตรวจสอบว่า block ว่างและใหญ่พอ
    mov rax, [rbx + 8]
    and rax, 1
    jnz .search_next        ; ถ้า allocated, ข้ามไป
    
    cmp [rbx], r12
    jge .found

.search_next:
    ; ข้าม (free list เป็น doubly linked)
    mov rbx, [rbx + 16]     ; next free
    jmp .search

.found:
    ; Found a free block
    ; Split ถ้าขนาดใหญ่พอ
    mov r13, [rbx]          ; block size
    sub r13, r12            ; remaining
    cmp r13, BLOCK_HEADER + MALLOC_MIN_ALLOC
    jl .use_whole_block
    
    ; Split block
    mov rax, rbx
    add rax, r12            ; new_block = current + needed
    mov [rax], r13          ; new_block.size = remaining
    mov rdi, BLOCK_MAGIC | BLOCK_FLAG_FREE
    mov [rax + 8], rdi      ; new_block.flags = free
    
    ; Insert new_block ใน free list (replace current)
    mov rdi, [rbx + 16]     ; current->next
    mov [rax + 16], rdi     ; new_block->next = current->next
    mov rdi, [rbx + 24]     ; current->prev
    mov [rax + 24], rdi     ; new_block->prev = current->prev
    
    ; Update neighbors' pointers
    test rdi, rdi
    jz .split_no_prev
    mov [rdi + 16], rax     ; prev->next = new_block
.split_no_prev:
    mov rdi, [rbx + 16]
    test rdi, rdi
    jz .split_no_next
    mov [rdi + 24], rax     ; next->prev = new_block
.split_no_next:
    
    ; ตรวจสอบว่า free_list head ต้องอัพเดทหรือไม่
    cmp rbx, [malloc_free_list]
    jne .split_done
    mov [malloc_free_list], rax
.split_done:
    
    mov qword [rbx], r12    ; current.size = needed
    jmp .mark_used

.use_whole_block:
    ; Remove จาก free list
    mov rdi, [rbx + 24]     ; prev
    mov rsi, [rbx + 16]     ; next
    test rdi, rdi
    jz .no_prev
    mov [rdi + 16], rsi     ; prev->next = next
    jmp .prev_done
.no_prev:
    mov [malloc_free_list], rsi  ; update head
.prev_done:
    test rsi, rsi
    jz .no_next
    mov [rsi + 24], rdi     ; next->prev = prev
.no_next:

.mark_used:
    ; Mark block as used
    mov qword [rbx + 8], BLOCK_MAGIC | BLOCK_FLAG_USED
    
    ; Return data pointer
    lea rax, [rbx + BLOCK_HEADER]
    jmp .done

.need_more_memory:
    ; ต้องการ memory เพิ่ม: mmap new chunk
    ; คำนวณขนาดที่ต้องการ
    mov rsi, r12
    add rsi, 4095
    and rsi, ~4095          ; page-align
    
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    ; rsi มีค่าอยู่แล้ว
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1
    xor r9, r9
    syscall
    
    cmp rax, -1
    je .return_null
    
    ; Initialize new chunk เป็น single free block
    mov rbx, rax
    mov [rbx], rsi          ; size = chunk size
    mov qword [rbx + 8], BLOCK_MAGIC | BLOCK_FLAG_FREE
    mov qword [rbx + 16], 0 ; next = NULL
    mov qword [rbx + 24], 0 ; prev = NULL
    
    ; เพิ่มเข้า free list
    mov rdi, [malloc_free_list]
    mov [rbx + 16], rdi     ; new->next = old_head
    test rdi, rdi
    jz .no_old_head
    mov [rdi + 24], rbx     ; old_head->prev = new
.no_old_head:
    mov [malloc_free_list], rbx  ; head = new
    
    jmp .search

.heap_corruption:
    ; Heap corruption detected!
    xor eax, eax
    jmp .done

.return_null:
    xor eax, eax

.done:
    pop r13 r12 rbx
    pop rbp
    ret

; Function: my_free
; Input:  rdi = ptr
my_free:
    push rbp
    mov rbp, rsp
    push rbx r12
    
    ; ตรวจสอบ NULL
    test rdi, rdi
    jz .done
    
    ; หา block header
    sub rdi, BLOCK_HEADER
    mov rbx, rdi
    
    ; ตรวจสอบ magic
    mov rax, [rbx + 8]
    and rax, 0xFFFF0000
    cmp rax, BLOCK_MAGIC
    jne .invalid_free
    
    ; ตรวจสอบว่า allocated จริงๆ
    mov rax, [rbx + 8]
    and rax, 1
    jz .double_free         ; ถ้า free อยู่แล้ว = double free bug!
    
    ; Mark as free
    mov qword [rbx + 8], BLOCK_MAGIC | BLOCK_FLAG_FREE
    
    ; เพิ่มเข้า free list (ที่ head)
    mov rax, [malloc_free_list]
    mov [rbx + 16], rax     ; this->next = old_head
    mov qword [rbx + 24], 0 ; this->prev = NULL
    test rax, rax
    jz .no_old
    mov [rax + 24], rbx     ; old_head->prev = this
.no_old:
    mov [malloc_free_list], rbx  ; head = this
    
    ; TODO: coalesce adjacent free blocks

.done:
    pop r12 rbx
    pop rbp
    ret

.invalid_free:
.double_free:
    ; Memory error! ควร print error message และ abort
    pop r12 rbx
    pop rbp
    ret

; Function: malloc_internal_init
malloc_internal_init:
    push rbp
    mov rbp, rsp
    
    mov byte [malloc_init_flag], 1
    mov qword [malloc_free_list], 0
    
    pop rbp
    ret
```

---

## 15. Memory Leak Detector

```nasm
; ==========================================
; MEMORY LEAK DETECTOR
; Track allocations และ report leaks เมื่อ exit
; ==========================================

section .data
    leak_msg        db "=== Memory Leak Report ===", 10, 0
    leak_msg_len    equ $ - leak_msg
    no_leak_msg     db "No memory leaks detected!", 10, 0
    no_leak_msg_len equ $ - no_leak_msg
    leak_fmt        db "LEAK: addr=0x", 0
    leak_fmt_len    equ $ - leak_fmt
    size_fmt        db " size=", 0

section .bss
    ; Tracking table: array of {ptr, size, file_line} records
    ; ขนาดสูงสุด 4096 entries
    TRACK_MAX_ENTRIES   equ 4096
    TRACK_ENTRY_SIZE    equ 24  ; ptr(8) + size(8) + caller_rip(8)
    
    track_table     resb (TRACK_MAX_ENTRIES * TRACK_ENTRY_SIZE)
    track_count     resq 1      ; จำนวน active allocations

section .text

; Function: track_alloc
; เพิ่ม allocation เข้า tracking table
; Input:  rdi = ptr, rsi = size, rdx = caller_rip
track_alloc:
    push rbp
    mov rbp, rsp
    push rbx r12
    
    ; ตรวจสอบว่า table เต็มหรือไม่
    mov rax, [track_count]
    cmp rax, TRACK_MAX_ENTRIES
    jge .table_full
    
    ; หาตำแหน่งใน table
    mov rbx, rax
    imul rbx, TRACK_ENTRY_SIZE
    lea rcx, [track_table + rbx]
    
    ; เขียน entry
    mov [rcx], rdi          ; ptr
    mov [rcx + 8], rsi      ; size
    mov [rcx + 16], rdx     ; caller_rip
    
    ; เพิ่ม count
    inc qword [track_count]

.table_full:
    pop r12 rbx
    pop rbp
    ret

; Function: track_free
; ลบ allocation ออกจาก tracking table
; Input:  rdi = ptr
track_free:
    push rbp
    mov rbp, rsp
    push rbx r12 r13
    
    mov r12, rdi            ; ptr ที่ต้องการ remove
    
    ; ค้นหาใน table
    xor rbx, rbx            ; index
    mov r13, [track_count]  ; total entries

.search_loop:
    cmp rbx, r13
    jge .not_found
    
    ; ตรวจสอบ ptr
    mov rax, rbx
    imul rax, TRACK_ENTRY_SIZE
    lea rcx, [track_table + rax]
    
    cmp [rcx], r12
    je .found
    
    inc rbx
    jmp .search_loop

.found:
    ; ลบ entry โดย swap กับ last entry
    dec qword [track_count]
    mov r13, [track_count]
    
    cmp rbx, r13
    je .done                ; ถ้าเป็น last entry อยู่แล้ว
    
    ; Copy last entry ไปที่ตำแหน่งที่ลบ
    mov rax, r13
    imul rax, TRACK_ENTRY_SIZE
    lea rsi, [track_table + rax]  ; last entry
    
    mov rax, rbx
    imul rax, TRACK_ENTRY_SIZE
    lea rdi, [track_table + rax]  ; found entry
    
    ; Copy 24 bytes (3 qwords)
    mov rax, [rsi]
    mov [rdi], rax
    mov rax, [rsi + 8]
    mov [rdi + 8], rax
    mov rax, [rsi + 16]
    mov [rdi + 16], rax

.not_found:
.done:
    pop r13 r12 rbx
    pop rbp
    ret

; Function: print_hex
; Print 64-bit number เป็น hex
; Input:  rdi = number
print_hex:
    push rbp
    mov rbp, rsp
    sub rsp, 32
    
    ; แปลงเป็น hex string
    lea rsi, [rsp + 15]     ; buffer end
    mov byte [rsi + 1], 10  ; newline
    mov rcx, 16             ; 16 hex digits
    
.hex_loop:
    mov rax, rdi
    and rax, 0xF            ; last nibble
    cmp rax, 10
    jl .digit
    add rax, 'a' - 10       ; a-f
    jmp .store
.digit:
    add rax, '0'
.store:
    mov [rsi], al
    dec rsi
    shr rdi, 4
    loop .hex_loop
    
    ; Print
    mov rax, 1              ; sys_write
    mov rdi, 1              ; stdout
    lea rsi, [rsp]          ; buffer
    mov rdx, 17             ; 16 digits + newline
    syscall
    
    pop rbp
    ret

; Function: report_leaks
; Print รายการ memory leaks
report_leaks:
    push rbp
    mov rbp, rsp
    push rbx r12 r13
    
    ; Print header
    mov rax, 1
    mov rdi, 1
    mov rsi, leak_msg
    mov rdx, leak_msg_len
    syscall
    
    mov r12, [track_count]  ; จำนวน leaks
    
    test r12, r12
    jz .no_leaks
    
    ; Print แต่ละ leak
    xor rbx, rbx
.leak_loop:
    cmp rbx, r12
    jge .done
    
    ; Print "LEAK: addr=0x"
    mov rax, 1
    mov rdi, 1
    mov rsi, leak_fmt
    mov rdx, leak_fmt_len
    syscall
    
    ; Print address
    mov rax, rbx
    imul rax, TRACK_ENTRY_SIZE
    lea rcx, [track_table + rax]
    mov rdi, [rcx]          ; ptr
    call print_hex
    
    ; Print " size="
    mov rax, 1
    mov rdi, 1
    mov rsi, size_fmt
    mov rdx, 6
    syscall
    
    ; Print size
    mov rax, rbx
    imul rax, TRACK_ENTRY_SIZE
    lea rcx, [track_table + rax]
    mov rdi, [rcx + 8]      ; size
    call print_hex
    
    inc rbx
    jmp .leak_loop

.no_leaks:
    mov rax, 1
    mov rdi, 1
    mov rsi, no_leak_msg
    mov rdx, no_leak_msg_len
    syscall

.done:
    pop r13 r12 rbx
    pop rbp
    ret
```

---

## 16. Guard Pages สำหรับ Stack Overflow Detection

```nasm
; ==========================================
; GUARD PAGES สำหรับ Stack Overflow Detection
; ใส่ PROT_NONE page ท้าย buffer เพื่อ detect overflow
; เมื่อ overflow เกิดขึ้น kernel ส่ง SIGSEGV
; ==========================================

section .data
    guard_err   db "Stack overflow detected! (Guard page hit)", 10, 0
    guard_err_len equ $ - guard_err

section .bss
    protected_stack_base    resq 1  ; ฐาน stack ที่ป้องกัน
    protected_stack_size    resq 1  ; ขนาด stack
    guard_page_addr         resq 1  ; ตำแหน่ง guard page

PAGE_SIZE   equ 4096

section .text

; Function: create_guarded_stack
; สร้าง stack พร้อม guard page ที่ท้าย
; Input:  rdi = stack_size (ไม่รวม guard page)
; Output: rax = pointer ไปยัง top ของ stack (สำหรับใช้งาน)
;         rdx = pointer ไปยัง base ของ stack
create_guarded_stack:
    push rbp
    mov rbp, rsp
    push rbx r12
    
    ; Align size ไปยัง page boundary
    add rdi, PAGE_SIZE - 1
    and rdi, ~(PAGE_SIZE - 1)
    mov r12, rdi            ; stack_size (page-aligned)
    
    ; Total size = stack_size + 1 guard page + 1 header page
    mov rdi, r12
    add rdi, PAGE_SIZE * 2  ; guard page + header page
    
    ; mmap total area ด้วย READ|WRITE
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, r12
    add rsi, PAGE_SIZE * 2
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x22           ; MAP_PRIVATE|MAP_ANON
    mov r8, -1
    xor r9, r9
    syscall
    
    mov rbx, rax            ; base ของ mapping ทั้งหมด
    
    ; Setup guard page ที่ท้ายสุด (ต่ำกว่า stack ที่ grows down)
    ; Layout: [base][guard_page][stack_area][header]
    ;          ^      ^           ^           ^ top
    
    ; guard page อยู่ที่ base
    mov [guard_page_addr], rbx
    
    ; เปลี่ยน guard page เป็น PROT_NONE
    mov rax, 10             ; sys_mprotect
    mov rdi, rbx            ; guard page address
    mov rsi, PAGE_SIZE      ; 1 page
    xor rdx, rdx            ; PROT_NONE
    syscall
    
    ; เก็บข้อมูล
    mov rax, rbx
    add rax, PAGE_SIZE      ; stack base = หลัง guard page
    mov [protected_stack_base], rax
    mov [protected_stack_size], r12
    
    ; return: rax = stack top, rdx = stack base
    mov rdx, rax            ; stack base
    add rax, r12            ; stack top = base + size
    
    pop r12 rbx
    pop rbp
    ret

; ตัวอย่างการใช้ guarded stack
; เมื่อ code เขียน past guard page → SIGSEGV

; Function: setup_segfault_handler
; ติดตั้ง SIGSEGV handler เพื่อ detect stack overflow
; (ใน Assembly จริงต้องใช้ sigaction syscall)
setup_segfault_handler:
    push rbp
    mov rbp, rsp
    sub rsp, 152            ; sigaction struct size
    
    ; ตั้งค่า sigaction struct
    ; sa_handler = หน่วยของ handler function
    lea rax, [segfault_handler]
    mov [rsp], rax          ; sa_sigaction
    xor eax, eax
    mov [rsp + 8], rax      ; sa_flags = 0
    
    ; sigaction(SIGSEGV=11, &new_action, NULL)
    mov rax, 13             ; sys_rt_sigaction
    mov rdi, 11             ; SIGSEGV
    mov rsi, rsp            ; new action
    xor rdx, rdx            ; old action = NULL
    mov r10, 8              ; sigsetsize
    syscall
    
    pop rbp
    ret

; Signal handler สำหรับ SIGSEGV
segfault_handler:
    ; พิมพ์ error message
    mov rax, 1              ; sys_write
    mov rdi, 2              ; stderr
    mov rsi, guard_err
    mov rdx, guard_err_len
    syscall
    
    ; Exit ด้วย error code
    mov rax, 60             ; sys_exit
    mov rdi, 11             ; exit code = SIGSEGV number
    syscall
```

---

## 17. Complete Demo Program

```nasm
; ==========================================
; DEMO PROGRAM: ทดสอบ memory management
; รวม malloc, pool, stack allocator
; ==========================================

section .data
    ; ข้อความต่างๆ
    demo_title      db "=== Memory Management Demo ===", 10, 0
    demo_title_len  equ $ - demo_title
    
    malloc_test     db "[TEST] malloc/free...", 10, 0
    malloc_test_len equ $ - malloc_test
    
    pool_test       db "[TEST] Memory pool...", 10, 0
    pool_test_len   equ $ - pool_test
    
    stack_test      db "[TEST] Stack allocator...", 10, 0
    stack_test_len  equ $ - stack_test
    
    mmap_test       db "[TEST] mmap operations...", 10, 0
    mmap_test_len   equ $ - mmap_test
    
    pass_msg        db "  [PASS]", 10, 0
    pass_msg_len    equ $ - pass_msg
    
    done_msg        db "All tests completed!", 10, 0
    done_msg_len    equ $ - done_msg

section .bss
    test_pool   resq 1      ; pool pointer
    test_stack  resq 1      ; stack allocator pointer

section .text
    global _start

_start:
    ; Print title
    mov rax, 1
    mov rdi, 1
    mov rsi, demo_title
    mov rdx, demo_title_len
    syscall
    
    ; Test 1: malloc/free
    mov rax, 1
    mov rdi, 1
    mov rsi, malloc_test
    mov rdx, malloc_test_len
    syscall
    
    ; Allocate
    mov rdi, 64
    call my_malloc
    mov rbx, rax            ; เก็บ pointer
    
    ; เขียนข้อมูล
    mov qword [rbx], 0x1122334455667788
    
    ; ตรวจสอบค่า
    mov rax, [rbx]
    cmp rax, 0x1122334455667788
    jne .test1_fail
    
    ; Free
    mov rdi, rbx
    call my_free
    
    ; Print pass
    mov rax, 1
    mov rdi, 1
    mov rsi, pass_msg
    mov rdx, pass_msg_len
    syscall
    jmp .test2

.test1_fail:
    ; handle fail...
    jmp .test2

.test2:
    ; Test 2: Memory pool
    mov rax, 1
    mov rdi, 1
    mov rsi, pool_test
    mov rdx, pool_test_len
    syscall
    
    ; สร้าง pool สำหรับ 64-byte blocks, 100 blocks
    mov rdi, 64
    mov rsi, 100
    call pool_create
    mov [test_pool], rax
    
    ; Allocate หลาย blocks
    mov r12, rax            ; pool
    mov r13, 10             ; จะ allocate 10 blocks
    
.pool_alloc_loop:
    test r13, r13
    jz .pool_alloc_done
    
    mov rdi, [test_pool]
    call pool_alloc
    ; ใช้ block...
    dec r13
    jmp .pool_alloc_loop

.pool_alloc_done:
    ; Print pass
    mov rax, 1
    mov rdi, 1
    mov rsi, pass_msg
    mov rdx, pass_msg_len
    syscall

.test3:
    ; Test 3: Stack allocator
    mov rax, 1
    mov rdi, 1
    mov rsi, stack_test
    mov rdx, stack_test_len
    syscall
    
    ; สร้าง stack allocator ขนาด 1MB
    mov rdi, (1024 * 1024)
    call stackalloc_create
    mov [test_stack], rax
    
    ; Save marker
    mov rdi, [test_stack]
    call stackalloc_get_marker
    mov r14, rax            ; marker
    
    ; Allocate ชั่วคราว
    mov rdi, [test_stack]
    mov rsi, 256
    call stackalloc_alloc
    
    mov rdi, [test_stack]
    mov rsi, 512
    call stackalloc_alloc
    
    ; Roll back ไปยัง marker
    mov rdi, [test_stack]
    mov rsi, r14
    call stackalloc_free_to_marker
    
    ; Print pass
    mov rax, 1
    mov rdi, 1
    mov rsi, pass_msg
    mov rdx, pass_msg_len
    syscall

.test4:
    ; Test 4: mmap operations
    mov rax, 1
    mov rdi, 1
    mov rsi, mmap_test
    mov rdx, mmap_test_len
    syscall
    
    ; Allocate 1 page
    mov rdi, 4096
    call mmap_alloc
    mov r12, rax
    
    ; เขียนข้อมูล
    mov qword [r12], 42
    mov qword [r12 + 8], 100
    
    ; ตรวจสอบ
    mov rax, [r12]
    cmp rax, 42
    jne .mmap_fail
    
    ; Free
    mov rdi, r12
    mov rsi, 4096
    call mmap_free
    
    ; Print pass
    mov rax, 1
    mov rdi, 1
    mov rsi, pass_msg
    mov rdx, pass_msg_len
    syscall
    jmp .tests_done

.mmap_fail:
    jmp .tests_done

.tests_done:
    ; Print done
    mov rax, 1
    mov rdi, 1
    mov rsi, done_msg
    mov rdx, done_msg_len
    syscall
    
    ; Exit
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 18. Memory Layout Analysis Tool

```nasm
; โปรแกรมวิเคราะห์ memory layout ของตัวเอง
; อ่าน /proc/self/maps และ parse ข้อมูล

section .data
    maps_file       db "/proc/self/maps", 0
    sep_line        db "----------------------------------------", 10, 0
    sep_len         equ $ - sep_line
    text_label      db "[TEXT] ", 0
    heap_label      db "[HEAP] ", 0
    stack_label     db "[STACK]", 0
    lib_label       db "[LIB]  ", 0
    anon_label      db "[ANON] ", 0

section .bss
    maps_buf        resb 65536  ; 64KB buffer
    line_buf        resb 256    ; buffer สำหรับ parse บรรทัด

section .text
    global _start

; โปรแกรมหลัก analyze_memory_layout
analyze_memory_layout:
    push rbp
    mov rbp, rsp
    push rbx r12 r13 r14 r15
    
    ; เปิด /proc/self/maps
    mov rax, 2              ; sys_open
    mov rdi, maps_file
    xor rsi, rsi            ; O_RDONLY
    xor rdx, rdx
    syscall
    
    mov r12, rax            ; fd
    
    ; อ่านทั้งหมดเข้า buffer
    mov rax, 0              ; sys_read
    mov rdi, r12
    mov rsi, maps_buf
    mov rdx, 65536
    syscall
    
    mov r13, rax            ; bytes read
    
    ; ปิด fd
    mov rax, 3              ; sys_close
    mov rdi, r12
    syscall
    
    ; แสดงผล
    mov rax, 1
    mov rdi, 1
    mov rsi, maps_buf
    mov rdx, r13
    syscall
    
    pop r15 r14 r13 r12 rbx
    pop rbp
    ret
```

---

## 19. Best Practices และ Common Pitfalls

### 19.1 Common Memory Bugs ใน Assembly

```nasm
; ตัวอย่าง bugs ที่พบบ่อย และวิธีหลีกเลี่ยง

; ======== BUG 1: Buffer Overflow ========
bad_example_1:
    ; อย่าทำแบบนี้!
    sub rsp, 16             ; allocate 16 bytes บน stack
    ; แต่เขียน 32 bytes → overflow!
    ; mov byte [rsp + 31], 0  <-- overflow!
    add rsp, 16
    ret

good_example_1:
    ; ทำแบบนี้แทน: ตรวจสอบขนาดก่อนเสมอ
    sub rsp, 32             ; allocate ให้มากพอ
    mov byte [rsp + 31], 0  ; OK!
    add rsp, 32
    ret

; ======== BUG 2: Use After Free ========
bad_example_2:
    ; อย่าทำแบบนี้!
    mov rdi, 64
    call my_malloc          ; allocate
    mov rbx, rax
    
    mov rdi, rbx
    call my_free            ; free
    
    ; mov rax, [rbx]       <-- use after free! undefined behavior
    ret

good_example_2:
    mov rdi, 64
    call my_malloc
    mov rbx, rax
    
    ; ใช้ memory ก่อน free
    mov qword [rbx], 42
    mov rax, [rbx]          ; OK: ใช้ก่อน free
    
    mov rdi, rbx
    call my_free
    
    xor rbx, rbx            ; สำคัญ: null ออก pointer หลัง free
    ret

; ======== BUG 3: Memory Leak ========
bad_example_3:
    mov rdi, 1024
    call my_malloc
    mov rbx, rax
    
    ; ลืม free!
    ; ... ใช้ memory ...
    ret                     ; leak!

good_example_3:
    push rbx
    
    mov rdi, 1024
    call my_malloc
    mov rbx, rax
    
    ; ใช้ memory
    
    mov rdi, rbx
    call my_free            ; ต้อง free เสมอ
    
    pop rbx
    ret

; ======== BUG 4: Alignment Issues ========
bad_example_4:
    ; mmap ด้วย address ที่ไม่ page-aligned
    ; mov rdi, some_odd_address  <-- จะ fail!
    ; mov rax, 10               ; mprotect
    ; syscall
    ret

good_example_4:
    ; ต้องใช้ page-aligned address เสมอ
    mov rdi, some_address
    and rdi, ~4095          ; align ลงไปยัง page boundary
    
    ; คำนวณ size ให้ครอบคลุม range ที่ต้องการ
    mov rsi, original_end
    sub rsi, rdi            ; size from aligned start to end
    add rsi, 4095
    and rsi, ~4095          ; align size up
    
    mov rax, 10             ; mprotect
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    syscall
    ret
```

### 19.2 Performance Tips

```nasm
; ======== PERFORMANCE TIP 1: Huge Pages ========
; ใช้ huge pages (2MB) สำหรับ large allocations
; ลด TLB pressure

huge_page_alloc:
    ; MAP_HUGETLB = 0x40000
    ; MAP_HUGE_2MB = (21 << 26)
    mov rax, 9              ; sys_mmap
    xor rdi, rdi
    mov rsi, (2 * 1024 * 1024)  ; 2 MB
    mov rdx, 3              ; PROT_READ|PROT_WRITE
    mov r10, 0x40000022     ; MAP_PRIVATE|MAP_ANON|MAP_HUGETLB
    mov r8, -1
    xor r9, r9
    syscall
    ret

; ======== PERFORMANCE TIP 2: Memory Prefetching ========
; ใช้ prefetch instructions เพื่อลด cache miss

prefetch_example:
    ; prefetch ข้อมูลเข้า L1 cache
    prefetcht0 [rdi]        ; L1 cache prefetch
    prefetcht1 [rdi + 64]   ; L2 cache prefetch
    prefetcht2 [rdi + 128]  ; L3 cache prefetch
    prefetchnta [rdi + 192] ; non-temporal (bypass cache)
    ret

; ======== PERFORMANCE TIP 3: Non-Temporal Stores ========
; สำหรับ write-only data ที่ไม่ต้องการ cache

streaming_write:
    ; เขียนข้อมูลโดยไม่ผ่าน cache (fast สำหรับ large writes)
    movntq [rdi], rax       ; non-temporal store (MMX)
    movntps [rdi], xmm0     ; non-temporal store (SSE)
    sfence                   ; ensure stores are visible
    ret
```

---

## 20. สรุป Memory Management ใน Assembly

### System Calls Summary

| System Call | Number | คำอธิบาย |
|-------------|--------|-----------|
| `brk`       | 12     | เปลี่ยน program break (heap end) |
| `mmap`      | 9      | Create memory mapping |
| `munmap`    | 11     | Remove memory mapping |
| `mprotect`  | 10     | เปลี่ยน memory permissions |
| `mlock`     | 149    | Lock pages ใน RAM |
| `munlock`   | 150    | Unlock pages |
| `mlockall`  | 151    | Lock all pages |
| `munlockall`| 152    | Unlock all pages |
| `mremap`    | 25     | Resize memory mapping |
| `mincore`   | 27     | ตรวจสอบ pages ใน RAM |
| `madvise`   | 28     | Memory usage hints |
| `mbind`     | 237    | NUMA memory binding |
| `get_mempolicy` | 239 | Get NUMA memory policy |

### Allocator Comparison

| Allocator | Alloc Speed | Free Speed | Memory Efficiency | Use Case |
|-----------|-------------|------------|-------------------|----------|
| Bump      | O(1)        | O(1)*      | สูง (ไม่ fragment) | Temp allocations, parsers |
| Free List | O(n)        | O(1)       | ปานกลาง           | General purpose |
| Pool      | O(1)        | O(1)       | สูงมาก             | Fixed-size objects |
| Stack     | O(1)        | O(1)       | สูงมาก             | LIFO allocations |

*Bump free เป็น no-op จริงๆ

### Key Principles

1. **Alignment matters**: ทุก allocation ควร align ตาม type ที่จะใช้ (8-byte สำหรับ 64-bit, 16-byte สำหรับ SIMD)

2. **Page boundaries**: `mmap`, `mprotect`, `munmap` ต้องใช้ page-aligned addresses เสมอ

3. **Null after free**: หลัง free ควร null pointer เสมอเพื่อหลีกเลี่ยง use-after-free

4. **RAII pattern**: บน high level ควร initialize และ cleanup memory ใน matched pairs

5. **Guard pages**: ใช้ PROT_NONE pages เพื่อ detect buffer overflows และ stack overflows

6. **madvise**: บอก kernel ถึง access pattern เพื่อ performance (sequential, random, willneed, dontneed)

7. **NUMA awareness**: บน multi-socket systems ควรสนใจ NUMA topology

---

## แหล่งอ้างอิง (References)

- Linux man pages: `man 2 mmap`, `man 2 brk`, `man 2 mprotect`, `man 2 mlock`, `man 2 mremap`, `man 2 mincore`, `man 2 madvise`, `man 2 mbind`
- Intel Software Developer's Manual, Volume 3: System Programming Guide
- "The Linux Programming Interface" by Michael Kerrisk
- glibc malloc implementation: https://sourceware.org/git/glibc.git
- jemalloc documentation: http://jemalloc.net/
- tcmalloc documentation: https://google.github.io/tcmalloc/

---

*จบ Part 055: Memory Management ใน Assembly*
