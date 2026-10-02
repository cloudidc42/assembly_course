# Part 053: File I/O Programming ใน Assembly

## บทนำ (Introduction)

การเขียน File I/O ใน Assembly เป็นทักษะสำคัญที่ช่วยให้เราเข้าใจว่า Operating System จัดการไฟล์อย่างไร
ทุก high-level language ที่เราใช้ (Python, C, Java) ล้วนเรียก syscall เหล่านี้ภายใต้ hood

ในบทนี้เราจะเรียนรู้:
- การเปิด/ปิดไฟล์ด้วย `open` และ `close` syscall
- การอ่าน/เขียนด้วย `read` และ `write` syscall
- การเลื่อน file pointer ด้วย `lseek`
- การ duplicate file descriptor ด้วย `dup`/`dup2`
- การสร้าง pipe สำหรับ inter-process communication
- การดู metadata ด้วย `stat`/`fstat`
- การทำงานกับ directory ด้วย `getdents64`
- การ memory-map ไฟล์ด้วย `mmap`
- โปรแกรมตัวอย่าง: cat, head, tail, wc, cp, grep

---

## 1. Open Syscall และ Flags

### 1.1 ความหมายของ File Descriptor

**File Descriptor (fd)** คือตัวเลขจำนวนเต็มที่ kernel ใช้แทนไฟล์ที่เปิดอยู่

```
fd = 0  →  stdin  (standard input)
fd = 1  →  stdout (standard output)
fd = 2  →  stderr (standard error)
fd = 3+ →  ไฟล์ที่เราเปิดเอง
```

เมื่อเรียก `open()` kernel จะ:
1. หา fd ที่ว่างที่เล็กที่สุด
2. สร้าง file description ใน kernel (position, flags, inode)
3. คืน fd กลับมาให้เรา

### 1.2 Open Flags

Flags เหล่านี้นิยามใน `<fcntl.h>` และใช้ใน rdi ของ open syscall:

```nasm
; ค่า flags สำหรับ open syscall (Linux x86-64)
O_RDONLY    equ 0        ; เปิดเพื่ออ่านอย่างเดียว
O_WRONLY    equ 1        ; เปิดเพื่อเขียนอย่างเดียว
O_RDWR      equ 2        ; เปิดเพื่ออ่านและเขียน

; Flags เพิ่มเติม (OR กับ flag หลัก)
O_CREAT     equ 0x40     ; สร้างไฟล์ถ้ายังไม่มี (ต้องระบุ mode)
O_TRUNC     equ 0x200    ; ตัดไฟล์ให้ขนาด 0 ถ้ามีอยู่แล้ว
O_APPEND    equ 0x400    ; เขียนต่อท้ายไฟล์เสมอ
O_NONBLOCK  equ 0x800    ; Non-blocking mode
O_EXCL      equ 0x80     ; Error ถ้าไฟล์มีอยู่แล้ว (ใช้กับ O_CREAT)
O_CLOEXEC   equ 0x80000  ; ปิด fd เมื่อ exec()
O_DSYNC     equ 0x1000   ; Synchronized I/O data integrity
O_SYNC      equ 0x101000 ; Synchronized I/O file integrity
```

### 1.3 Mode Permissions (Octal)

เมื่อใช้ `O_CREAT` ต้องระบุ mode (permission bits):

```
Permission bits (octal):
  S_IRUSR = 0400   → User can read
  S_IWUSR = 0200   → User can write
  S_IXUSR = 0100   → User can execute
  S_IRGRP = 0040   → Group can read
  S_IWGRP = 0020   → Group can write
  S_IXGRP = 0010   → Group can execute
  S_IROTH = 0004   → Others can read
  S_IWOTH = 0002   → Others can write
  S_IXOTH = 0001   → Others can execute

ตัวอย่าง mode ที่ใช้บ่อย:
  0644 = rw-r--r--  (ไฟล์ text ทั่วไป)
  0755 = rwxr-xr-x  (executable)
  0600 = rw-------  (ไฟล์ส่วนตัว)
  0777 = rwxrwxrwx  (ทุกคนเข้าถึงได้ - ไม่ปลอดภัย)
```

**หมายเหตุ:** mode จริงที่ได้ = mode & ~umask (ค่า umask ปกติ = 0022)

### 1.4 โปรแกรมตัวอย่าง: เปิดและปิดไฟล์

```nasm
; open_close.asm - ตัวอย่างการเปิดและปิดไฟล์
; Compile: nasm -f elf64 open_close.asm -o open_close.o
; Link:    ld open_close.o -o open_close

section .data
    filename    db "test.txt", 0        ; ชื่อไฟล์ (null-terminated)
    msg_ok      db "เปิดไฟล์สำเร็จ fd=", 0
    msg_err     db "เปิดไฟล์ไม่สำเร็จ", 10, 0
    msg_close   db "ปิดไฟล์สำเร็จ", 10, 0
    newline     db 10, 0

section .bss
    fd_buf      resb 4                  ; buffer สำหรับ fd number

section .text
    global _start

_start:
    ; เรียก open syscall
    ; syscall number: rax = 2 (open)
    ; rdi = pointer to filename
    ; rsi = flags (O_RDONLY = 0)
    ; rdx = mode (0 สำหรับ O_RDONLY)
    mov rax, 2              ; syscall: open
    mov rdi, filename       ; path
    mov rsi, 0              ; flags: O_RDONLY
    mov rdx, 0              ; mode: ไม่ใช้สำหรับ O_RDONLY
    syscall

    ; ตรวจสอบ error: ถ้า rax < 0 แสดงว่า error
    cmp rax, 0
    jl .error

    ; บันทึก fd
    mov r12, rax            ; เก็บ fd ไว้ใน r12

    ; แสดง fd
    mov rax, 1              ; syscall: write
    mov rdi, 1              ; fd: stdout
    mov rsi, msg_ok
    mov rdx, 18             ; ความยาว
    syscall

    ; แปลง fd เป็น ASCII แล้วแสดง
    mov rax, r12            ; fd value
    add rax, '0'            ; แปลงเป็น ASCII (สมมติ fd < 10)
    mov [fd_buf], al
    mov rax, 1
    mov rdi, 1
    mov rsi, fd_buf
    mov rdx, 1
    syscall

    ; ขึ้นบรรทัดใหม่
    mov rax, 1
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall

    ; ปิดไฟล์
    ; syscall number: rax = 3 (close)
    ; rdi = fd
    mov rax, 3              ; syscall: close
    mov rdi, r12            ; fd ที่ต้องการปิด
    syscall

    ; แสดง success message
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_close
    mov rdx, 14
    syscall

    jmp .exit

.error:
    ; แสดง error message
    mov rax, 1
    mov rdi, 2              ; stderr
    mov rsi, msg_err
    mov rdx, 19
    syscall

.exit:
    ; exit syscall
    mov rax, 60             ; syscall: exit
    mov rdi, 0              ; exit code
    syscall
```

### 1.5 creat Syscall

`creat()` เป็น shorthand สำหรับ `open(path, O_WRONLY|O_CREAT|O_TRUNC, mode)`:

```nasm
; syscall number สำหรับ creat = 85
; rdi = path
; rsi = mode (permission bits)

mov rax, 85             ; syscall: creat
mov rdi, filename       ; path
mov rsi, 0644o          ; mode: rw-r--r--
syscall
; rax = fd หรือ -errno ถ้า error
```

---

## 2. Read และ Write Syscall

### 2.1 read syscall

```nasm
; syscall: read
; rax = 0
; rdi = fd
; rsi = buffer address
; rdx = count (จำนวน bytes ที่ต้องการอ่าน)
; return: rax = bytes ที่อ่านจริง, 0 = EOF, -1 = error
```

**สิ่งสำคัญที่ต้องรู้เกี่ยวกับ read:**
- `read()` อาจคืนค่าน้อยกว่า `count` ที่ขอ (partial read)
- เกิดได้เมื่อ: EOF ใกล้แล้ว, signal มาขัด, network/pipe
- ต้อง loop จนกว่าจะอ่านครบหรือ EOF

### 2.2 Complete Read Loop

```nasm
; complete_read.asm - การอ่านไฟล์แบบ complete (handle partial reads)

section .bss
    buffer      resb 4096           ; buffer ขนาด 4KB
    total_buf   resb 65536          ; buffer สำหรับเก็บข้อมูลทั้งหมด

section .text

; ฟังก์ชัน read_all: อ่านจนกว่าจะ EOF หรือ error
; Input:  rdi = fd, rsi = buffer, rdx = buffer_size
; Output: rax = total bytes read, หรือ -1 ถ้า error
read_all:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14

    mov r12, rdi            ; fd
    mov r13, rsi            ; buffer start
    mov r14, rdx            ; max size
    xor rbx, rbx            ; total bytes read = 0

.read_loop:
    ; คำนวณ remaining space
    mov rax, r14
    sub rax, rbx
    jle .done               ; ถ้าไม่มี space แล้ว หยุด

    ; เรียก read
    mov rax, 0              ; syscall: read
    mov rdi, r12            ; fd
    lea rsi, [r13 + rbx]    ; buffer + offset
    mov rdx, rax            ; remaining space
    syscall

    ; ตรวจสอบผลลัพธ์
    cmp rax, 0
    jl .error               ; error
    je .done                ; EOF (read คืน 0)

    ; เพิ่ม bytes ที่อ่านได้
    add rbx, rax

    jmp .read_loop

.done:
    mov rax, rbx            ; return total bytes read
    jmp .cleanup

.error:
    mov rax, -1             ; return -1 ถ้า error

.cleanup:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

### 2.3 write syscall

```nasm
; syscall: write
; rax = 1
; rdi = fd
; rsi = buffer address
; rdx = count
; return: rax = bytes ที่เขียนจริง, -1 = error
```

**Complete Write Loop** (handle partial writes):

```nasm
; write_all: เขียนข้อมูลให้ครบ
; Input:  rdi = fd, rsi = buffer, rdx = count
; Output: rax = 0 ถ้าสำเร็จ, -1 ถ้า error
write_all:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14

    mov r12, rdi            ; fd
    mov r13, rsi            ; buffer
    mov r14, rdx            ; total bytes to write
    xor rbx, rbx            ; bytes written so far = 0

.write_loop:
    mov rax, r14
    sub rax, rbx
    jle .done               ; เขียนครบแล้ว

    mov rax, 1              ; syscall: write
    mov rdi, r12            ; fd
    lea rsi, [r13 + rbx]    ; buffer + offset
    mov rdx, rax            ; remaining bytes
    syscall

    cmp rax, 0
    jle .error              ; error หรือ 0 bytes (ผิดปกติ)

    add rbx, rax
    jmp .write_loop

.done:
    xor rax, rax            ; return 0 = success
    jmp .cleanup

.error:
    mov rax, -1

.cleanup:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 3. lseek Syscall

### 3.1 ความหมายของ lseek

`lseek` ใช้เลื่อน file offset (position) ของ file descriptor

```nasm
; syscall: lseek
; rax = 8
; rdi = fd
; rsi = offset (signed 64-bit)
; rdx = whence (จุดอ้างอิง)
; return: rax = new position จาก beginning of file, หรือ -1 ถ้า error
```

### 3.2 Whence Values

```nasm
SEEK_SET    equ 0       ; จากต้นไฟล์ (absolute position)
SEEK_CUR    equ 1       ; จาก current position (relative)
SEEK_END    equ 2       ; จากท้ายไฟล์ (ใช้ offset ลบเพื่อย้อนกลับ)
```

### 3.3 ตัวอย่างการใช้ lseek

```nasm
; lseek_demo.asm - ตัวอย่างการใช้ lseek

section .data
    filename    db "data.bin", 0
    write_data  db "Hello World!", 0
    read_buf    times 13 db 0

section .text
    global _start

_start:
    ; เปิดไฟล์เพื่ออ่านเขียน (สร้างถ้าไม่มี)
    mov rax, 2
    mov rdi, filename
    mov rsi, 0x42           ; O_RDWR | O_CREAT = 2 | 64
    mov rdx, 0644o
    syscall
    mov r12, rax            ; เก็บ fd

    ; เขียนข้อมูล
    mov rax, 1
    mov rdi, r12
    mov rsi, write_data
    mov rdx, 12
    syscall

    ; ไปที่ต้นไฟล์ (SEEK_SET, offset=0)
    mov rax, 8              ; syscall: lseek
    mov rdi, r12
    mov rsi, 0              ; offset = 0
    mov rdx, 0              ; SEEK_SET
    syscall
    ; rax = 0 (new position)

    ; อ่านไฟล์ตั้งแต่ต้น
    mov rax, 0
    mov rdi, r12
    mov rsi, read_buf
    mov rdx, 12
    syscall

    ; หา file size โดยใช้ SEEK_END
    mov rax, 8
    mov rdi, r12
    mov rsi, 0              ; offset = 0
    mov rdx, 2              ; SEEK_END
    syscall
    ; rax = file size (เพราะ offset 0 จากท้าย = ขนาดไฟล์)
    mov r13, rax            ; เก็บ file size

    ; กลับไปที่ offset 6 จากต้นไฟล์
    mov rax, 8
    mov rdi, r12
    mov rsi, 6              ; offset = 6
    mov rdx, 0              ; SEEK_SET
    syscall

    ; ปิดไฟล์
    mov rax, 3
    mov rdi, r12
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### 3.4 Tell Position (หา current position)

```nasm
; tell: หา current file position
; Input:  rdi = fd
; Output: rax = current position
tell:
    mov rax, 8          ; lseek
    ; rdi ยังคงเป็น fd
    mov rsi, 0          ; offset = 0
    mov rdx, 1          ; SEEK_CUR
    syscall
    ret
    ; rax = current position
```

---

## 4. dup และ dup2 Syscall

### 4.1 dup Syscall

`dup` สร้าง duplicate ของ file descriptor:

```nasm
; syscall: dup
; rax = 32
; rdi = oldfd
; return: rax = new fd (ตัวเล็กสุดที่ว่าง)

mov rax, 32
mov rdi, oldfd
syscall
; rax = new fd ที่ชี้ไปที่เดียวกับ oldfd
```

### 4.2 dup2 Syscall

`dup2` duplicate fd ไปยัง fd ที่ระบุ:

```nasm
; syscall: dup2
; rax = 33
; rdi = oldfd
; rsi = newfd (ถ้าเปิดอยู่จะปิดก่อน)
; return: rax = newfd

mov rax, 33
mov rdi, oldfd
mov rsi, newfd
syscall
```

### 4.3 Redirect stdout ไปยังไฟล์

```nasm
; redirect_stdout.asm - เปลี่ยน stdout ไปเขียนลงไฟล์

section .data
    outfile     db "output.txt", 0
    message     db "ข้อความนี้จะไปในไฟล์ output.txt", 10, 0
    msg_len     equ $ - message

section .text
    global _start

_start:
    ; เปิดไฟล์สำหรับเขียน (สร้างใหม่/ตัดเนื้อหาเดิม)
    ; flags: O_WRONLY|O_CREAT|O_TRUNC = 1|64|512 = 577 = 0x241
    mov rax, 2
    mov rdi, outfile
    mov rsi, 0x241          ; O_WRONLY|O_CREAT|O_TRUNC
    mov rdx, 0644o          ; permission
    syscall
    mov r12, rax            ; บันทึก file fd

    ; บันทึก stdout fd ก่อนเปลี่ยน
    mov rax, 32             ; dup
    mov rdi, 1              ; stdout
    syscall
    mov r13, rax            ; บันทึก stdout backup

    ; เปลี่ยน stdout (fd=1) ให้ชี้ไปที่ไฟล์
    mov rax, 33             ; dup2
    mov rdi, r12            ; oldfd = file
    mov rsi, 1              ; newfd = stdout
    syscall

    ; ปิด file fd เดิม (ไม่จำเป็นอีกแล้วเพราะ dup2 แล้ว)
    mov rax, 3
    mov rdi, r12
    syscall

    ; ตอนนี้ write ไปที่ stdout = write ลงไฟล์
    mov rax, 1              ; write
    mov rdi, 1              ; stdout (= ไฟล์)
    mov rsi, message
    mov rdx, msg_len
    syscall

    ; คืน stdout กลับมา
    mov rax, 33             ; dup2
    mov rdi, r13            ; backup stdout
    mov rsi, 1              ; restore stdout
    syscall

    ; ปิด backup
    mov rax, 3
    mov rdi, r13
    syscall

    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 5. Pipe Syscall

### 5.1 ความหมายของ Pipe

Pipe เป็น unidirectional channel สำหรับส่งข้อมูลระหว่าง processes:
- ข้อมูลเขียนจาก write end
- ข้อมูลอ่านจาก read end
- มี kernel buffer (ปกติ 64KB บน Linux)

```nasm
; syscall: pipe
; rax = 22
; rdi = pointer to int[2] array
;        pipefd[0] = read end fd
;        pipefd[1] = write end fd
; return: rax = 0 สำเร็จ, -1 error

section .bss
    pipefd  resq 1          ; int[2] = 8 bytes

mov rax, 22
mov rdi, pipefd
syscall
; pipefd[0] = read end
; pipefd[1] = write end
```

### 5.2 ตัวอย่าง Pipe ระหว่าง Parent และ Child

```nasm
; pipe_demo.asm - ส่งข้อมูลผ่าน pipe ระหว่าง parent/child

section .data
    message     db "Hello from parent!", 0
    msg_len     equ 18
    read_buf    times 64 db 0

section .bss
    pipefd      resd 2      ; int[2]: [0]=read, [1]=write

section .text
    global _start

_start:
    ; สร้าง pipe
    mov rax, 22             ; syscall: pipe
    mov rdi, pipefd
    syscall
    cmp rax, 0
    jne .error

    ; fork
    mov rax, 57             ; syscall: fork
    syscall

    cmp rax, 0
    je .child_code          ; child process: rax = 0
    jl .error               ; fork error

    ; === PARENT PROCESS ===
    ; ปิด read end (parent จะเขียน)
    mov rax, 3
    mov rdi, dword [pipefd]  ; pipefd[0] = read end
    syscall

    ; เขียนข้อมูลไปที่ pipe
    mov rax, 1              ; write
    mov rdi, dword [pipefd+4] ; pipefd[1] = write end
    mov rsi, message
    mov rdx, msg_len
    syscall

    ; ปิด write end
    mov rax, 3
    mov rdi, dword [pipefd+4]
    syscall

    ; รอ child
    mov rax, 61             ; syscall: wait4
    mov rdi, -1
    xor rsi, rsi
    xor rdx, rdx
    xor r10, r10
    syscall

    jmp .parent_exit

.child_code:
    ; === CHILD PROCESS ===
    ; ปิด write end (child จะอ่าน)
    mov rax, 3
    mov rdi, dword [pipefd+4]
    syscall

    ; อ่านจาก pipe
    mov rax, 0              ; read
    mov rdi, dword [pipefd]  ; read end
    mov rsi, read_buf
    mov rdx, 64
    syscall

    ; เขียน stdout
    mov rdx, rax            ; bytes read
    mov rax, 1
    mov rdi, 1
    mov rsi, read_buf
    syscall

    ; ปิด read end
    mov rax, 3
    mov rdi, dword [pipefd]
    syscall

    ; child exit
    mov rax, 60
    xor rdi, rdi
    syscall

.parent_exit:
    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 6. stat และ fstat Syscall

### 6.1 struct stat Layout (x86-64 Linux)

```c
// struct stat ใน x86-64 Linux (128 bytes)
struct stat {
    uint64_t  st_dev;       // offset 0:  Device ID
    uint64_t  st_ino;       // offset 8:  Inode number
    uint64_t  st_nlink;     // offset 16: Number of hard links
    uint32_t  st_mode;      // offset 24: File type and mode
    uint32_t  st_uid;       // offset 28: User ID of owner
    uint32_t  st_gid;       // offset 32: Group ID of owner
    uint32_t  __pad0;       // offset 36: Padding
    uint64_t  st_rdev;      // offset 40: Device ID (if special file)
    int64_t   st_size;      // offset 48: Total size in bytes
    int64_t   st_blksize;   // offset 56: Block size for I/O
    int64_t   st_blocks;    // offset 64: Number of 512B blocks
    // timestamps (each is timespec: seconds + nanoseconds)
    int64_t   st_atime;     // offset 72: Access time (seconds)
    int64_t   st_atimensec; // offset 80: Access time (nanoseconds)
    int64_t   st_mtime;     // offset 88: Modification time
    int64_t   st_mtimensec; // offset 96: Modification time (nanoseconds)
    int64_t   st_ctime;     // offset 104: Status change time
    int64_t   st_ctimensec; // offset 112: Status change time (nanoseconds)
    int64_t   __unused[3];  // offset 120: Reserved
};
```

### 6.2 stat Syscall

```nasm
; stat_demo.asm - ดู file metadata

section .data
    filename    db "test.txt", 0
    size_msg    db "File size: ", 0
    newline     db 10

section .bss
    statbuf     resb 144    ; struct stat (144 bytes เผื่อไว้)

section .text
    global _start

_start:
    ; เรียก stat syscall
    ; rax = 4 (stat)
    ; rdi = path
    ; rsi = pointer to struct stat
    mov rax, 4              ; syscall: stat
    mov rdi, filename
    mov rsi, statbuf
    syscall

    cmp rax, 0
    jne .error

    ; อ่าน file size จาก offset 48
    mov rax, qword [statbuf + 48]   ; st_size
    ; rax = file size ในหน่วย bytes

    ; แสดง "File size: "
    mov rax, 1
    mov rdi, 1
    mov rsi, size_msg
    mov rdx, 11
    syscall

    ; แปลง size เป็น string และแสดง
    mov rdi, qword [statbuf + 48]
    call print_uint64

    ; newline
    mov rax, 1
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall

    ; อ่าน permissions จาก st_mode (offset 24)
    mov eax, dword [statbuf + 24]   ; st_mode
    and eax, 0xFFF                  ; เอาแค่ permission bits (12 bits)
    ; eax = permission bits เช่น 0644

    ; ตรวจสอบว่าเป็น regular file หรือเปล่า
    mov eax, dword [statbuf + 24]
    and eax, 0xF000                 ; เอา file type bits
    cmp eax, 0x8000                 ; S_IFREG = 0x8000
    ; je ถ้าเป็น regular file

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    mov rax, 60
    mov rdi, 1
    syscall

; print_uint64: แสดงตัวเลข 64-bit
; Input: rdi = number
print_uint64:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov rax, rdi
    mov rcx, 10
    lea rsi, [rsp + 20]
    mov byte [rsi], 0
    dec rsi

.digit_loop:
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rsi], dl
    dec rsi
    test rax, rax
    jnz .digit_loop

    inc rsi
    ; คำนวณ length
    lea rdx, [rsp + 20]
    sub rdx, rsi

    mov rax, 1
    mov rdi, 1
    ; rsi ยังชี้ที่ string
    syscall

    add rsp, 32
    pop rbp
    ret
```

### 6.3 fstat Syscall

```nasm
; fstat ใช้ fd แทน path
; rax = 5
; rdi = fd
; rsi = pointer to struct stat

mov rax, 5              ; syscall: fstat
mov rdi, r12            ; fd
mov rsi, statbuf
syscall
```

### 6.4 File Type Bits

```nasm
; File type bits ใน st_mode
S_IFMT      equ 0xF000  ; Bitmask for file type
S_IFSOCK    equ 0xC000  ; Socket
S_IFLNK     equ 0xA000  ; Symbolic link
S_IFREG     equ 0x8000  ; Regular file
S_IFBLK     equ 0x6000  ; Block device
S_IFDIR     equ 0x4000  ; Directory
S_IFCHR     equ 0x2000  ; Character device
S_IFIFO     equ 0x1000  ; FIFO/pipe

; ตรวจสอบว่าเป็น directory
mov eax, dword [statbuf + 24]
and eax, S_IFMT
cmp eax, S_IFDIR
je is_directory
```

---

## 7. openat, statat, mkdirat (AT Variants)

### 7.1 ทำไมต้องมี AT Variants

`openat` แก้ปัญหา TOCTOU (Time-of-check to time-of-use) race condition:

```nasm
; openat syscall
; rax = 257
; rdi = dirfd (AT_FDCWD = -100 สำหรับ current directory)
; rsi = path (relative to dirfd)
; rdx = flags
; r10 = mode

AT_FDCWD    equ -100    ; ใช้ current working directory

; เปิดไฟล์ relative ถึง current directory
mov rax, 257
mov rdi, -100           ; AT_FDCWD
mov rsi, filename
mov rdx, 0              ; O_RDONLY
mov r10, 0
syscall
```

### 7.2 mkdirat Syscall

```nasm
; mkdirat: สร้าง directory
; rax = 258
; rdi = dirfd
; rsi = path
; rdx = mode

mov rax, 258
mov rdi, -100           ; AT_FDCWD
mov rsi, dirname        ; ชื่อ directory
mov rdx, 0755o          ; permissions
syscall
```

### 7.3 fstatat Syscall (newfstatat)

```nasm
; newfstatat / fstatat
; rax = 262
; rdi = dirfd
; rsi = path
; rdx = statbuf
; r10 = flags (0 หรือ AT_SYMLINK_NOFOLLOW = 0x100)

AT_SYMLINK_NOFOLLOW equ 0x100   ; ไม่ follow symlink

mov rax, 262
mov rdi, -100           ; AT_FDCWD
mov rsi, filename
mov rdx, statbuf
mov r10, 0              ; flags
syscall
```

---

## 8. Directory: getdents64

### 8.1 linux_dirent64 Structure

```c
// struct linux_dirent64
struct linux_dirent64 {
    uint64_t  d_ino;        // offset 0:  Inode number
    int64_t   d_off;        // offset 8:  Offset to next entry
    uint16_t  d_reclen;     // offset 16: Length of this record
    uint8_t   d_type;       // offset 18: File type
    char      d_name[];     // offset 19: Filename (null-terminated)
};

// d_type values:
// DT_UNKNOWN = 0
// DT_FIFO    = 1  (named pipe)
// DT_CHR     = 2  (character device)
// DT_DIR     = 4  (directory)
// DT_BLK     = 6  (block device)
// DT_REG     = 8  (regular file)
// DT_LNK     = 10 (symbolic link)
// DT_SOCK    = 12 (socket)
```

### 8.2 getdents64 Syscall

```nasm
; getdents64: อ่าน directory entries
; rax = 217
; rdi = fd (directory fd)
; rsi = buffer
; rdx = buffer_size
; return: rax = bytes used in buffer, 0 = done, -1 = error
```

### 8.3 โปรแกรม List Directory

```nasm
; listdir.asm - แสดงรายชื่อไฟล์ใน directory

section .data
    dot_dir     db ".", 0           ; current directory
    tab         db "  ", 0
    newline     db 10

section .bss
    dirbuf      resb 8192           ; buffer สำหรับ directory entries

section .text
    global _start

_start:
    ; เปิด directory
    ; O_RDONLY = 0, O_DIRECTORY = 0x10000
    mov rax, 2              ; open
    mov rdi, dot_dir
    mov rsi, 0x10000        ; O_DIRECTORY
    xor rdx, rdx
    syscall
    cmp rax, 0
    jle .error
    mov r12, rax            ; บันทึก dir fd

.getdents_loop:
    ; เรียก getdents64
    mov rax, 217            ; getdents64
    mov rdi, r12            ; dir fd
    mov rsi, dirbuf
    mov rdx, 8192           ; buffer size
    syscall

    cmp rax, 0
    jl .error               ; error
    je .done                ; ไม่มี entry เพิ่มเติม

    ; iterate ผ่าน entries
    xor rbx, rbx            ; offset ใน buffer = 0
    mov r14, rax            ; total bytes returned

.entry_loop:
    cmp rbx, r14
    jge .getdents_loop      ; อ่าน batch ถัดไป

    ; pointer ไปยัง current entry
    lea r13, [dirbuf + rbx]

    ; อ่าน d_reclen (offset 16)
    movzx rcx, word [r13 + 16]  ; d_reclen

    ; อ่าน d_type (offset 18)
    movzx rax, byte [r13 + 18]  ; d_type

    ; ชื่อไฟล์อยู่ที่ offset 19
    lea rsi, [r13 + 19]

    ; ข้าม . และ ..
    cmp byte [rsi], '.'
    je .skip_entry

    ; แสดงชื่อไฟล์
    ; คำนวณ string length ของ d_name
    push rsi
    xor rdx, rdx
.strlen_loop:
    cmp byte [rsi + rdx], 0
    je .strlen_done
    inc rdx
    jmp .strlen_loop
.strlen_done:
    pop rsi

    mov rax, 1
    mov rdi, 1
    ; rsi = d_name, rdx = length
    syscall

    ; แสดง newline
    mov rax, 1
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall

.skip_entry:
    ; เลื่อนไปยัง entry ถัดไป
    add rbx, rcx
    jmp .entry_loop

.done:
    ; ปิด directory
    mov rax, 3
    mov rdi, r12
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 9. mmap สำหรับ File

### 9.1 ทำไมต้องใช้ mmap

`mmap` map ไฟล์เข้า address space โดยตรง:
- ไม่ต้องมี read/write loop (kernel จัดการ pagefault)
- เหมาะกับการอ่านไฟล์ขนาดใหญ่ที่ random access
- ประหยัด memory (multiple processes share same pages)

### 9.2 mmap Syscall

```nasm
; syscall: mmap
; rax = 9
; rdi = addr (0 = kernel เลือกให้)
; rsi = length
; rdx = prot (protection flags)
; r10 = flags
; r8  = fd (-1 ถ้า anonymous)
; r9  = offset (ต้องเป็น page-aligned)
; return: rax = virtual address (หรือ MAP_FAILED = -1)

; Protection flags:
PROT_NONE   equ 0       ; ไม่สามารถ access ได้
PROT_READ   equ 1       ; สามารถอ่านได้
PROT_WRITE  equ 2       ; สามารถเขียนได้
PROT_EXEC   equ 4       ; สามารถ execute ได้

; Map flags:
MAP_SHARED      equ 1   ; การแก้ไขจะเห็นได้โดย process อื่น
MAP_PRIVATE     equ 2   ; Copy-on-write (ไม่กระทบไฟล์จริง)
MAP_ANONYMOUS   equ 32  ; ไม่ map ไฟล์ (fd = -1)
MAP_FIXED       equ 16  ; บังคับ map ที่ addr ที่ระบุ
```

### 9.3 ตัวอย่าง mmap ไฟล์

```nasm
; mmap_file.asm - อ่านไฟล์ด้วย mmap

section .data
    filename    db "large_file.txt", 0

section .bss
    statbuf     resb 144

section .text
    global _start

_start:
    ; เปิดไฟล์
    mov rax, 2
    mov rdi, filename
    xor rsi, rsi            ; O_RDONLY
    xor rdx, rdx
    syscall
    cmp rax, 0
    jle .error
    mov r12, rax            ; fd

    ; หา file size ด้วย fstat
    mov rax, 5              ; fstat
    mov rdi, r12
    mov rsi, statbuf
    syscall
    mov r13, qword [statbuf + 48]   ; st_size = file size

    ; mmap ไฟล์
    mov rax, 9              ; mmap
    xor rdi, rdi            ; addr = 0 (kernel เลือก)
    mov rsi, r13            ; length = file size
    mov rdx, 1              ; PROT_READ
    mov r10, 2              ; MAP_PRIVATE
    mov r8, r12             ; fd
    xor r9, r9              ; offset = 0
    syscall

    ; ตรวจสอบ MAP_FAILED
    cmp rax, -1
    je .error
    mov r14, rax            ; บันทึก mapped address

    ; ตอนนี้ r14 ชี้ที่ข้อมูลในไฟล์
    ; เราสามารถอ่านได้โดยตรงด้วย [r14 + offset]

    ; ตัวอย่าง: เขียน content ไปที่ stdout
    mov rax, 1              ; write
    mov rdi, 1              ; stdout
    mov rsi, r14            ; mapped data
    mov rdx, r13            ; file size
    syscall

    ; munmap (คืน mapping)
    ; rax = 11 (munmap)
    ; rdi = addr
    ; rsi = length
    mov rax, 11
    mov rdi, r14
    mov rsi, r13
    syscall

    ; ปิดไฟล์
    mov rax, 3
    mov rdi, r12
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 10. โปรแกรม cat

```nasm
; cat.asm - implement cat command
; Usage: ./cat [file1] [file2] ...
; Compile: nasm -f elf64 cat.asm -o cat.o && ld cat.o -o cat

section .bss
    buffer      resb 65536          ; 64KB buffer

section .text
    global _start

_start:
    ; rdi = argc, rsi = argv[]
    ; kernel puts argc ที่ [rsp], argv ที่ [rsp+8]
    pop rdi                 ; argc
    mov r15, rsp            ; argv pointer

    ; ถ้าไม่มี argument ให้อ่านจาก stdin
    cmp rdi, 1
    je .read_stdin

    ; loop ผ่าน arguments
    mov r14, 1              ; index เริ่มจาก 1 (ข้าม argv[0])

.arg_loop:
    cmp r14, rdi
    jge .done               ; หมด arguments แล้ว

    ; เปิดไฟล์ argv[r14]
    mov rax, 2              ; open
    mov rdi, [r15 + r14*8]  ; argv[i]
    xor rsi, rsi            ; O_RDONLY
    xor rdx, rdx
    syscall

    cmp rax, 0
    jl .next_arg            ; ข้ามถ้าเปิดไม่ได้

    mov r12, rax            ; fd

    ; copy ไฟล์ไปยัง stdout
    call cat_fd

    ; ปิดไฟล์
    mov rax, 3
    mov rdi, r12
    syscall

.next_arg:
    inc r14
    jmp .arg_loop

.done:
    mov rax, 60
    xor rdi, rdi
    syscall

.read_stdin:
    xor r12, r12            ; fd = 0 (stdin)
    call cat_fd
    jmp .done

; cat_fd: copy fd r12 ไปยัง stdout
cat_fd:
    push rbp
    mov rbp, rsp

.read_loop:
    ; อ่านจาก fd
    mov rax, 0              ; read
    mov rdi, r12            ; fd
    mov rsi, buffer
    mov rdx, 65536
    syscall

    cmp rax, 0
    jle .cat_done           ; EOF หรือ error

    ; เขียนไปยัง stdout
    mov rdx, rax            ; bytes ที่อ่านได้
    mov rax, 1              ; write
    mov rdi, 1              ; stdout
    mov rsi, buffer
    syscall

    jmp .read_loop

.cat_done:
    pop rbp
    ret
```

---

## 11. โปรแกรม head -n

```nasm
; head.asm - implement head -n N command
; ค้นหา N บรรทัดแรก
; Usage: ./head -n 10 file.txt

section .data
    newline     db 10

section .bss
    buffer      resb 4096           ; อ่านทีละ 4KB

section .text
    global _start

_start:
    pop r15                 ; argc
    mov r14, rsp            ; argv

    ; Default: แสดง 10 บรรทัด
    mov r13, 10             ; default N

    ; parse arguments: ถ้ามี -n X
    mov r11, 1              ; arg index

.parse_loop:
    cmp r11, r15
    jge .parse_done

    mov rdi, [r14 + r11*8]  ; argv[i]
    ; ตรวจสอบว่าเริ่มด้วย '-n'
    cmp byte [rdi], '-'
    jne .not_flag
    cmp byte [rdi+1], 'n'
    jne .not_flag

    ; ถ้ามี '-n N' เป็น separate argument
    cmp byte [rdi+2], 0
    je .n_separate
    ; '-nN' รูปแบบ combined
    lea rdi, [rdi+2]
    call atoi
    mov r13, rax
    jmp .next_arg

.n_separate:
    inc r11
    cmp r11, r15
    jge .parse_done
    mov rdi, [r14 + r11*8]
    call atoi
    mov r13, rax
    jmp .next_arg

.not_flag:
    ; เปิดไฟล์นี้
    push r11
    mov r12, r11            ; บันทึก index
    pop r11
    jmp .process_file_at

.next_arg:
    inc r11
    jmp .parse_loop

.parse_done:
    ; ถ้าไม่มีไฟล์ อ่าน stdin
    mov r12, 0              ; stdin fd
    jmp .process_fd

.process_file_at:
    mov rax, 2
    mov rdi, [r14 + r12*8]
    xor rsi, rsi
    xor rdx, rdx
    syscall
    cmp rax, 0
    jl .exit
    mov r12, rax

.process_fd:
    ; อ่านและแสดง N บรรทัดแรก
    xor rbx, rbx            ; line count = 0
    xor r9, r9              ; buffer offset

.head_loop:
    ; อ่าน 1 byte ทีละ byte (simple but slow)
    mov rax, 0
    mov rdi, r12
    mov rsi, buffer
    mov rdx, 1
    syscall

    cmp rax, 0
    jle .head_done

    ; เขียน byte นี้
    mov rax, 1
    mov rdi, 1
    mov rsi, buffer
    mov rdx, 1
    syscall

    ; ตรวจสอบว่าเป็น newline
    cmp byte [buffer], 10
    jne .head_loop

    inc rbx
    cmp rbx, r13
    jl .head_loop

.head_done:
    ; ปิดไฟล์ (ถ้าไม่ใช่ stdin)
    cmp r12, 0
    je .exit
    mov rax, 3
    mov rdi, r12
    syscall

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

; atoi: แปลง ASCII string เป็นตัวเลข
; Input: rdi = string pointer
; Output: rax = number
atoi:
    xor rax, rax
    xor rcx, rcx
.atoi_loop:
    movzx rcx, byte [rdi]
    test rcx, rcx
    jz .atoi_done
    sub rcx, '0'
    cmp rcx, 9
    ja .atoi_done
    imul rax, rax, 10
    add rax, rcx
    inc rdi
    jmp .atoi_loop
.atoi_done:
    ret
```

---

## 12. โปรแกรม tail -n (Tricky!)

```nasm
; tail.asm - implement tail -n N
; วิธีที่ใช้: อ่านไฟล์ทั้งหมด หา position ของ newline ย้อนหลัง N ครั้ง
; สำหรับไฟล์ขนาดใหญ่ ใช้ circular buffer

section .bss
    ; Circular buffer strategy สำหรับ tail
    ; เก็บ positions ของ newlines
    nl_positions    resq 65536      ; เก็บ positions ของ newlines (max 65536 lines)
    buffer          resb 65536      ; 64KB read buffer

section .data
    default_n   dq 10               ; default: 10 บรรทัดสุดท้าย

section .text
    global _start

_start:
    pop r15                 ; argc
    mov r14, rsp            ; argv

    mov r13, 10             ; default N = 10 บรรทัด

    ; parse -n argument (simple: assume argv[1]=-n, argv[2]=N, argv[3]=file)
    ; ตรวจสอบ arguments...
    cmp r15, 1
    je .tail_stdin

    ; หา N และ filename
    ; (simplified: ถ้า argc >= 3 และ argv[1]="-n")
    mov r12, 0              ; stdin fd default

    cmp r15, 4
    jge .parse_n

    ; ถ้า argc = 2 ให้ทำ tail file
    cmp r15, 2
    je .open_file_arg1

    jmp .tail_stdin

.parse_n:
    ; argv[1] = "-n", argv[2] = N, argv[3] = file
    mov rdi, [r14 + 16]     ; argv[2] = N string
    call atoi
    mov r13, rax

    mov rax, 2              ; open argv[3]
    mov rdi, [r14 + 24]
    xor rsi, rsi
    xor rdx, rdx
    syscall
    cmp rax, 0
    jl .exit
    mov r12, rax
    jmp .do_tail

.open_file_arg1:
    mov rax, 2
    mov rdi, [r14 + 8]      ; argv[1]
    xor rsi, rsi
    xor rdx, rdx
    syscall
    cmp rax, 0
    jl .exit
    mov r12, rax
    jmp .do_tail

.tail_stdin:
    xor r12, r12            ; stdin

.do_tail:
    ; Strategy: บันทึก positions ของ newlines ทั้งหมด
    ; แล้วหา position ของ newline ที่ N ตัวจากท้าย
    ; แล้ว lseek ไปที่นั้นและ output ส่วนที่เหลือ

    xor rbx, rbx            ; total newlines found
    xor r9, r9              ; file position

    ; Pass 1: scan หา newlines ทั้งหมด
.scan_loop:
    mov rax, 0              ; read
    mov rdi, r12
    mov rsi, buffer
    mov rdx, 65536
    syscall

    cmp rax, 0
    jle .scan_done

    mov r10, rax            ; bytes read
    xor r8, r8              ; index ใน buffer

.scan_buf:
    cmp r8, r10
    jge .scan_next_chunk

    cmp byte [buffer + r8], 10  ; newline?
    jne .not_nl

    ; บันทึก position ของ newline
    mov rax, r9
    add rax, r8
    ; เก็บใน circular buffer: nl_positions[rbx % max_nl]
    mov rcx, rbx
    and rcx, 65535           ; modulo 65536
    mov [nl_positions + rcx*8], rax

    inc rbx

.not_nl:
    inc r8
    jmp .scan_buf

.scan_next_chunk:
    add r9, r10
    jmp .scan_loop

.scan_done:
    ; ตอนนี้ rbx = total newlines
    ; หา position ที่ N บรรทัดจากท้าย

    ; ถ้า N >= total lines แสดงทั้งหมด
    cmp r13, rbx
    jge .output_all

    ; หา newline index ของ (total - N) ตัวจากต้น
    mov rax, rbx
    sub rax, r13            ; index = total - N
    ; position ใน circular buffer
    and rax, 65535
    mov r8, [nl_positions + rax*8]  ; position ของ newline นั้น
    inc r8                   ; เริ่มหลัง newline

    ; lseek ไปที่ position นั้น
    mov rax, 8              ; lseek
    mov rdi, r12
    mov rsi, r8
    mov rdx, 0              ; SEEK_SET
    syscall

.output_rest:
    ; อ่านและ output ส่วนที่เหลือ
    mov rax, 0
    mov rdi, r12
    mov rsi, buffer
    mov rdx, 65536
    syscall

    cmp rax, 0
    jle .tail_done

    mov rdx, rax
    mov rax, 1
    mov rdi, 1
    mov rsi, buffer
    syscall

    jmp .output_rest

.output_all:
    ; lseek กลับต้น
    mov rax, 8
    mov rdi, r12
    xor rsi, rsi
    xor rdx, rdx
    syscall
    jmp .output_rest

.tail_done:
    cmp r12, 0
    je .exit
    mov rax, 3
    mov rdi, r12
    syscall

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

; atoi function
atoi:
    xor rax, rax
    xor rcx, rcx
.atoi_loop:
    movzx rcx, byte [rdi]
    test rcx, rcx
    jz .atoi_done
    sub rcx, '0'
    cmp rcx, 9
    ja .atoi_done
    imul rax, rax, 10
    add rax, rcx
    inc rdi
    jmp .atoi_loop
.atoi_done:
    ret
```

---

## 13. โปรแกรม wc (Word Count)

```nasm
; wc.asm - implement wc command
; แสดง: lines, words, characters

section .bss
    buffer      resb 65536

section .data
    space       db " ", 0
    newline     db 10

section .text
    global _start

_start:
    pop r15                 ; argc
    mov r14, rsp            ; argv

    ; เริ่มนับจาก stdin หรือ file
    cmp r15, 2
    jge .open_file

    ; stdin
    xor r12, r12
    jmp .do_wc

.open_file:
    mov rax, 2
    mov rdi, [r14 + 8]      ; argv[1]
    xor rsi, rsi
    xor rdx, rdx
    syscall
    cmp rax, 0
    jl .exit
    mov r12, rax

.do_wc:
    xor rbx, rbx            ; line count
    xor r9, r9              ; word count
    xor r10, r10            ; char count
    xor r11, r11            ; in_word flag (0 = not in word)

.read_loop:
    mov rax, 0              ; read
    mov rdi, r12
    mov rsi, buffer
    mov rdx, 65536
    syscall

    cmp rax, 0
    jle .print_results

    mov r13, rax            ; bytes read
    xor r8, r8              ; buffer index

.process_bytes:
    cmp r8, r13
    jge .read_loop

    movzx rax, byte [buffer + r8]
    inc r8
    inc r10                 ; char count++

    ; ตรวจสอบ newline
    cmp al, 10
    je .is_newline

    ; ตรวจสอบ whitespace (space, tab, newline)
    cmp al, ' '
    je .is_whitespace
    cmp al, 9               ; tab
    je .is_whitespace
    cmp al, 13              ; carriage return
    je .is_whitespace

    ; ไม่ใช่ whitespace = อยู่ใน word
    cmp r11, 0
    jne .process_bytes      ; ยังอยู่ใน word อยู่แล้ว

    ; เริ่ม word ใหม่
    inc r9                  ; word count++
    mov r11, 1              ; in_word = true
    jmp .process_bytes

.is_newline:
    inc rbx                 ; line count++
    ; fall through to whitespace handling

.is_whitespace:
    mov r11, 0              ; in_word = false
    jmp .process_bytes

.print_results:
    ; แสดง lines
    mov rdi, rbx
    call print_uint64

    mov rax, 1
    mov rdi, 1
    mov rsi, space
    mov rdx, 1
    syscall

    ; แสดง words
    mov rdi, r9
    call print_uint64

    mov rax, 1
    mov rdi, 1
    mov rsi, space
    mov rdx, 1
    syscall

    ; แสดง chars
    mov rdi, r10
    call print_uint64

    mov rax, 1
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall

    ; ปิดไฟล์
    cmp r12, 0
    je .exit
    mov rax, 3
    mov rdi, r12
    syscall

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

; print_uint64: พิมพ์ตัวเลข 64-bit
; Input: rdi = number
print_uint64:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov rax, rdi
    mov r8, 10
    lea rsi, [rsp + 20]
    mov byte [rsi], 0
    dec rsi
    xor rcx, rcx

.digit_loop:
    xor rdx, rdx
    div r8
    add dl, '0'
    mov [rsi], dl
    dec rsi
    inc rcx
    test rax, rax
    jnz .digit_loop

    inc rsi
    mov rdx, rcx
    mov rax, 1
    mov rdi, 1
    syscall

    add rsp, 32
    pop rbp
    ret
```

---

## 14. โปรแกรม cp ด้วย mmap

```nasm
; cp_mmap.asm - implement cp โดยใช้ mmap
; Usage: ./cp src dest
; ใช้ mmap เพื่อ copy ไฟล์อย่างมีประสิทธิภาพ

section .bss
    statbuf     resb 144

section .text
    global _start

_start:
    pop r15                 ; argc
    mov r14, rsp            ; argv

    ; ต้องมี 2 arguments: src และ dest
    cmp r15, 3
    jne .usage_error

    ; เปิด source file
    mov rax, 2              ; open
    mov rdi, [r14 + 8]      ; argv[1] = src
    xor rsi, rsi            ; O_RDONLY
    xor rdx, rdx
    syscall
    cmp rax, 0
    jl .error
    mov r12, rax            ; src fd

    ; หา file size
    mov rax, 5              ; fstat
    mov rdi, r12
    mov rsi, statbuf
    syscall
    cmp rax, 0
    jl .error

    mov r13, qword [statbuf + 48]   ; file size

    ; ถ้าไฟล์ว่าง
    test r13, r13
    jz .empty_file

    ; mmap source
    mov rax, 9              ; mmap
    xor rdi, rdi            ; addr = 0
    mov rsi, r13            ; length = file size
    mov rdx, 1              ; PROT_READ
    mov r10, 2              ; MAP_PRIVATE
    mov r8, r12             ; src fd
    xor r9, r9              ; offset = 0
    syscall
    cmp rax, -1
    je .error
    mov r11, rax            ; src_map

    ; เปิด destination file
    ; flags: O_WRONLY|O_CREAT|O_TRUNC = 1|64|512 = 577
    mov rax, 2              ; open
    mov rdi, [r14 + 16]     ; argv[2] = dest
    mov rsi, 577            ; O_WRONLY|O_CREAT|O_TRUNC
    mov rdx, 0644o          ; permissions
    syscall
    cmp rax, 0
    jl .unmap_error
    mov rbx, rax            ; dest fd

    ; เขียน mapped data ไปยัง dest
    mov rax, 1              ; write
    mov rdi, rbx            ; dest fd
    mov rsi, r11            ; src_map
    mov rdx, r13            ; file size
    syscall

    ; ปิด dest
    mov rax, 3
    mov rdi, rbx
    syscall

    ; munmap src
    mov rax, 11
    mov rdi, r11
    mov rsi, r13
    syscall

.empty_file:
    ; ปิด src
    mov rax, 3
    mov rdi, r12
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall

.unmap_error:
    mov rax, 11
    mov rdi, r11
    mov rsi, r13
    syscall
    jmp .error

.error:
.usage_error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 15. โปรแกรม Simple Grep

```nasm
; grep_simple.asm - simple grep (scan + print matching lines)
; Usage: ./grep pattern file
; ค้นหา pattern ในแต่ละบรรทัด แล้วพิมพ์บรรทัดที่พบ

section .bss
    filebuf     resb 1048576    ; 1MB buffer สำหรับไฟล์
    linebuf     resb 4096       ; buffer สำหรับแต่ละบรรทัด

section .data
    newline     db 10

section .text
    global _start

_start:
    pop r15                 ; argc
    mov r14, rsp            ; argv

    cmp r15, 3
    jne .exit

    ; argv[1] = pattern, argv[2] = filename
    mov r13, [r14 + 8]      ; pattern string
    mov r12, [r14 + 16]     ; filename

    ; เปิดไฟล์
    mov rax, 2
    mov rdi, r12
    xor rsi, rsi
    xor rdx, rdx
    syscall
    cmp rax, 0
    jl .exit
    mov r12, rax            ; fd

    ; คำนวณ pattern length
    mov rdi, r13
    call strlen
    mov rbp, rax            ; pattern length

    ; อ่านไฟล์ทั้งหมด
    mov rax, 0
    mov rdi, r12
    mov rsi, filebuf
    mov rdx, 1048576
    syscall
    cmp rax, 0
    jle .close_exit
    mov r11, rax            ; bytes read

    ; scan แต่ละบรรทัด
    xor rbx, rbx            ; current position
    xor r9, r9              ; line start

.scan_lines:
    cmp rbx, r11
    jge .done_scan

    movzx rax, byte [filebuf + rbx]

    cmp al, 10              ; newline?
    jne .not_nl

    ; พบ end of line: ตรวจสอบว่า pattern อยู่ในบรรทัดนี้
    mov rdi, filebuf
    add rdi, r9             ; line start
    mov rsi, rbx
    sub rsi, r9             ; line length
    mov rdx, r13            ; pattern
    mov rcx, rbp            ; pattern length
    call memmem

    cmp rax, -1
    je .no_match

    ; พบ match: พิมพ์บรรทัดนี้
    mov rax, 1
    mov rdi, 1
    lea rsi, [filebuf + r9] ; line start
    mov rdx, rbx
    sub rdx, r9             ; line length
    syscall

    ; พิมพ์ newline
    mov rax, 1
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall

.no_match:
    ; เลื่อน line start
    lea r9, [rbx + 1]

.not_nl:
    inc rbx
    jmp .scan_lines

.done_scan:
.close_exit:
    mov rax, 3
    mov rdi, r12
    syscall

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

; strlen: คำนวณความยาว null-terminated string
; Input:  rdi = string
; Output: rax = length
strlen:
    xor rax, rax
.strlen_loop:
    cmp byte [rdi + rax], 0
    je .strlen_done
    inc rax
    jmp .strlen_loop
.strlen_done:
    ret

; memmem: ค้นหา pattern ใน buffer
; Input:  rdi = haystack, rsi = haystack_len, rdx = needle, rcx = needle_len
; Output: rax = offset ถ้าพบ, หรือ -1 ถ้าไม่พบ
memmem:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    push r15

    mov r12, rdi            ; haystack
    mov r13, rsi            ; haystack_len
    mov r14, rdx            ; needle
    mov r15, rcx            ; needle_len

    ; ถ้า needle ยาวกว่า haystack ไม่มีทางพบ
    cmp r15, r13
    ja .not_found

    ; ถ้า needle ว่าง พบที่ 0 เสมอ
    test r15, r15
    jz .found_zero

    ; คำนวณ max position ที่จะ scan
    mov rbx, r13
    sub rbx, r15            ; rbx = haystack_len - needle_len

    xor r8, r8              ; current position

.outer_loop:
    cmp r8, rbx
    jg .not_found

    ; เปรียบเทียบ needle ที่ position r8
    xor r9, r9              ; inner index

.inner_loop:
    cmp r9, r15
    jge .match_found

    movzx rax, byte [r12 + r8 + r9]
    movzx rcx, byte [r14 + r9]
    cmp al, cl
    jne .no_match_here

    inc r9
    jmp .inner_loop

.match_found:
    mov rax, r8             ; return position
    jmp .memmem_done

.no_match_here:
    inc r8
    jmp .outer_loop

.not_found:
    mov rax, -1
    jmp .memmem_done

.found_zero:
    xor rax, rax

.memmem_done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 16. Error Handling และ errno

### 16.1 syscall Error Return

```nasm
; Linux syscall คืน -errno ถ้า error
; errno values ที่เกี่ยวข้องกับ File I/O:
ENOENT      equ 2       ; ไม่มีไฟล์หรือ directory
EACCES      equ 13      ; Permission denied
EEXIST      equ 17      ; ไฟล์มีอยู่แล้ว
ENOTDIR     equ 20      ; ไม่ใช่ directory
EISDIR      equ 21      ; เป็น directory
EINVAL      equ 22      ; Invalid argument
EMFILE      equ 24      ; Too many open files (process limit)
ENFILE      equ 23      ; Too many open files (system limit)
EFBIG       equ 27      ; File too large
ENOSPC      equ 28      ; No space left on device
EROFS       equ 30      ; Read-only file system
EPIPE       equ 32      ; Broken pipe
EBADF       equ 9       ; Bad file descriptor
```

### 16.2 ตัวอย่าง Error Handling

```nasm
; error_handling.asm - การจัดการ error อย่างถูกต้อง

section .data
    errmsg_noent    db "Error: file not found", 10, 0
    errmsg_acces    db "Error: permission denied", 10, 0
    errmsg_generic  db "Error: unknown error", 10, 0

section .text
    global _start

check_open_error:
    ; Input: rax = return value จาก open syscall
    ; ถ้า rax >= 0 ไม่มี error
    cmp rax, 0
    jge .no_error

    ; รับ errno (= -rax)
    neg rax
    mov r10, rax

    cmp r10, 2              ; ENOENT
    je .err_noent

    cmp r10, 13             ; EACCES
    je .err_acces

    ; error อื่นๆ
    mov rax, 1
    mov rdi, 2              ; stderr
    mov rsi, errmsg_generic
    mov rdx, 21
    syscall
    jmp .error_exit

.err_noent:
    mov rax, 1
    mov rdi, 2
    mov rsi, errmsg_noent
    mov rdx, 22
    syscall
    jmp .error_exit

.err_acces:
    mov rax, 1
    mov rdi, 2
    mov rsi, errmsg_acces
    mov rdx, 25
    syscall

.error_exit:
    mov rax, 60
    mov rdi, 1
    syscall

.no_error:
    ret
```

---

## 17. Buffered I/O ใน Assembly

### 17.1 ทำไมต้องทำ Buffering

การเรียก `read`/`write` ทีละ 1 byte ช้ามาก เพราะต้อง context switch ทุกครั้ง
การทำ buffering ช่วยลด syscall calls ลงมาก

```nasm
; buffered_io.asm - ตัวอย่าง buffered I/O

section .bss
    ; Input buffer
    in_buf          resb 4096
    in_buf_pos      resq 1      ; current read position
    in_buf_end      resq 1      ; end of valid data

    ; Output buffer
    out_buf         resb 4096
    out_buf_pos     resq 1      ; current write position

section .text

; buffered_read_byte: อ่าน 1 byte จาก fd r12 โดยใช้ buffer
; Output: al = byte, หรือ -1 ถ้า EOF
buffered_read_byte:
    ; ตรวจสอบว่า buffer ยังมีข้อมูล
    mov rax, [in_buf_pos]
    cmp rax, [in_buf_end]
    jl .has_data

    ; refill buffer
    mov rax, 0              ; read
    mov rdi, r12
    mov rsi, in_buf
    mov rdx, 4096
    syscall

    cmp rax, 0
    jle .eof

    ; อัพเดท positions
    xor rcx, rcx
    mov [in_buf_pos], rcx   ; pos = 0
    mov [in_buf_end], rax   ; end = bytes read

.has_data:
    mov rcx, [in_buf_pos]
    movzx rax, byte [in_buf + rcx]
    inc rcx
    mov [in_buf_pos], rcx
    ret

.eof:
    mov rax, -1
    ret

; buffered_write_byte: เขียน 1 byte ลง buffer (flush เมื่อเต็ม)
; Input: al = byte, r13 = output fd
buffered_write_byte:
    mov rcx, [out_buf_pos]
    mov [out_buf + rcx], al
    inc rcx
    mov [out_buf_pos], rcx

    cmp rcx, 4096
    jl .done

    ; flush buffer
    call flush_output_buf

.done:
    ret

; flush_output_buf: flush output buffer ไปยัง fd r13
flush_output_buf:
    mov rcx, [out_buf_pos]
    test rcx, rcx
    jz .already_empty

    mov rax, 1              ; write
    mov rdi, r13
    mov rsi, out_buf
    mov rdx, rcx
    syscall

    xor rcx, rcx
    mov [out_buf_pos], rcx

.already_empty:
    ret
```

---

## 18. Advanced Topics

### 18.1 O_DIRECT: Bypass Page Cache

```nasm
; O_DIRECT = 0x4000
; ข้าม page cache, เหมาะสำหรับ database หรือ backup software
; ต้องการ buffer ที่ aligned ตาม sector size (ปกติ 512 bytes หรือ 4096 bytes)
; rdx ต้อง aligned ด้วย (transfer size)

O_DIRECT    equ 0x4000

; สร้าง aligned buffer
; ใช้ mmap แทน stack/bss เพื่อ alignment
mov rax, 9              ; mmap
xor rdi, rdi
mov rsi, 4096           ; 1 page
mov rdx, 3              ; PROT_READ|PROT_WRITE
mov r10, 34             ; MAP_PRIVATE|MAP_ANONYMOUS
mov r8, -1              ; no fd
xor r9, r9
syscall
; rax = aligned buffer (page-aligned)
```

### 18.2 sendfile Syscall

```nasm
; sendfile: copy ระหว่าง fd โดยไม่ต้องผ่าน userspace
; rax = 40
; rdi = out_fd
; rsi = in_fd
; rdx = offset pointer (หรือ NULL)
; r10 = count
; return: bytes transferred

mov rax, 40
mov rdi, dest_fd
mov rsi, src_fd
xor rdx, rdx            ; offset = NULL (ใช้ current position)
mov r10, file_size
syscall
```

### 18.3 fallocate Syscall

```nasm
; fallocate: จองพื้นที่ disk ล่วงหน้า
; rax = 285
; rdi = fd
; rsi = mode (0 = normal, FALLOC_FL_KEEP_SIZE = 1)
; rdx = offset
; r10 = len

mov rax, 285
mov rdi, fd
xor rsi, rsi            ; mode = 0
xor rdx, rdx            ; offset = 0
mov r10, desired_size   ; ขนาดที่ต้องการ
syscall
```

---

## 19. Syscall Reference Table

```
Syscall Number  Name        Description
---------------------------------------------------
0               read        อ่านจาก fd
1               write       เขียนไปยัง fd
2               open        เปิดไฟล์
3               close       ปิด fd
4               stat        ดู file metadata โดย path
5               fstat       ดู file metadata โดย fd
6               lstat       stat แต่ไม่ follow symlink
8               lseek       เลื่อน file offset
9               mmap        map memory/file
11              munmap      unmap memory
22              pipe        สร้าง pipe
32              dup         duplicate fd
33              dup2        duplicate fd ไปยัง fd ที่ระบุ
40              sendfile    copy ระหว่าง fd
57              fork        สร้าง process ลูก
60              exit        จบโปรแกรม
85              creat       สร้างไฟล์ใหม่
217             getdents64  อ่าน directory entries
257             openat      open relative to directory fd
258             mkdirat     สร้าง directory relative to fd
262             newfstatat  stat relative to directory fd
285             fallocate   จองพื้นที่ disk
```

---

## 20. การ Compile และ Test

### 20.1 Makefile สำหรับ Part 053

```makefile
# Makefile for File I/O Assembly programs

CC      = nasm
LD      = ld
FLAGS   = -f elf64

all: cat_prog head_prog tail_prog wc_prog cp_prog grep_prog

cat_prog: cat.asm
	$(CC) $(FLAGS) cat.asm -o cat.o
	$(LD) cat.o -o cat_prog

head_prog: head.asm
	$(CC) $(FLAGS) head.asm -o head.o
	$(LD) head.o -o head_prog

tail_prog: tail.asm
	$(CC) $(FLAGS) tail.asm -o tail.o
	$(LD) tail.o -o tail_prog

wc_prog: wc.asm
	$(CC) $(FLAGS) wc.asm -o wc.o
	$(LD) wc.o -o wc_prog

cp_prog: cp_mmap.asm
	$(CC) $(FLAGS) cp_mmap.asm -o cp_mmap.o
	$(LD) cp_mmap.o -o cp_prog

grep_prog: grep_simple.asm
	$(CC) $(FLAGS) grep_simple.asm -o grep_simple.o
	$(LD) grep_simple.o -o grep_prog

test: all
	# สร้างไฟล์ทดสอบ
	echo "line 1\nline 2\nline 3\nline 4\nline 5" > test.txt
	# ทดสอบ cat
	./cat_prog test.txt
	# ทดสอบ head
	./head_prog -n 3 test.txt
	# ทดสอบ wc
	./wc_prog test.txt
	# ทดสอบ cp
	./cp_prog test.txt test_copy.txt
	# ทดสอบ grep
	./grep_prog "line 3" test.txt

clean:
	rm -f *.o cat_prog head_prog tail_prog wc_prog cp_prog grep_prog
	rm -f test.txt test_copy.txt
```

### 20.2 Tips สำหรับการ Debug

```bash
# ดู syscalls ที่โปรแกรมเรียกด้วย strace
strace ./cat_prog file.txt

# ดู file descriptors ที่เปิดอยู่
ls -la /proc/$(pidof program)/fd

# ตรวจสอบว่าไฟล์ถูกเขียนถูกต้อง
xxd output.bin | head -20

# ดู memory maps
cat /proc/self/maps
```

---

## 21. สรุปและ Best Practices

### 21.1 หลักการสำคัญ

1. **ตรวจสอบ return value เสมอ**: syscall ทุกตัวอาจ fail
2. **ปิด fd ทุกตัวที่เปิด**: resource leak ทำให้ program ช้าลง
3. **Handle partial read/write**: อย่า assume ว่าได้ครบ
4. **ใช้ O_CLOEXEC**: ป้องกัน fd leak ผ่าน exec
5. **Buffer I/O**: ลด syscall overhead
6. **ใช้ mmap สำหรับ large files**: ประหยัด memory copy

### 21.2 Common Pitfalls

```nasm
; ผิด: assume read คืนครบ
mov rax, 0
mov rdi, fd
mov rsi, buffer
mov rdx, 1024
syscall
; อาจได้น้อยกว่า 1024 bytes!

; ถูก: loop จนกว่าจะครบ
call read_all           ; ใช้ read_all function

; ผิด: ลืม null-terminate string
; ถูก: เสมอ null-terminate ก่อนใช้เป็น filename

; ผิด: ใช้ path ที่ไม่ได้ null-terminate
mov rsi, some_string    ; ต้องมี byte 0 ต่อท้าย

; ผิด: mode ใน open ด้วย hex แต่ตั้งใจ octal
mov rdx, 0644           ; นี่คือ decimal 644! ไม่ใช่ octal
; ถูก:
mov rdx, 0644o          ; NASM syntax สำหรับ octal
; หรือ:
mov rdx, 0x1A4          ; 0644 octal = 420 decimal = 0x1A4 hex
```

### 21.3 Performance Tips

```nasm
; 1. ใช้ buffer ขนาด power-of-2 และ aligned
;    เหมาะสุด: 4096 bytes (1 page) หรือ 65536 bytes (16 pages)

; 2. อ่านไฟล์ขนาดใหญ่ด้วย mmap แทน read loop
;    เหมาะสำหรับ: random access, ไฟล์ > 1MB

; 3. ใช้ sendfile สำหรับ copy ไฟล์
;    เร็วกว่า read+write loop มาก

; 4. ใช้ O_RDWR เมื่อต้องการอ่านและเขียน
;    แทนการเปิด 2 fd แยกกัน

; 5. Batch directory operations ด้วย getdents64 buffer ใหญ่
;    buffer 8192+ bytes ลด syscall calls
```

---

## 22. แบบฝึกหัด (Exercises)

1. **Exercise 1**: แก้ไข cat.asm ให้รองรับ option `-n` (แสดงเลขบรรทัด)

2. **Exercise 2**: เขียน `tee` ที่รับ stdin แล้ว copy ไปทั้ง stdout และไฟล์

3. **Exercise 3**: เขียน `rev` ที่แสดงแต่ละบรรทัดในลำดับกลับ

4. **Exercise 4**: เขียน `uniq` ที่ลบบรรทัดซ้ำติดกัน

5. **Exercise 5**: ปรับปรุง grep ให้รองรับ case-insensitive search (`-i` option)

6. **Exercise 6**: เขียน directory lister ที่แสดง file size ด้วย (คล้าย `ls -la`)

7. **Exercise 7**: ใช้ mmap เพื่อ search pattern ใน binary file โดยไม่ต้องอ่านทั้งหมดเข้า memory

8. **Exercise 8**: เขียน simple `diff` ที่เปรียบเทียบ 2 ไฟล์ บรรทัดต่อบรรทัด

---

## 23. บทสรุป

File I/O ใน Assembly ช่วยให้เราเข้าใจ:

- **Kernel abstraction**: ทุก I/O operation เป็น syscall ที่ kernel จัดการ
- **File descriptor**: เป็น handle ที่เรียบง่ายแต่ทรงพลัง
- **Buffering**: การลด syscall overhead เป็นสิ่งสำคัญสำหรับ performance
- **mmap**: วิธีที่มีประสิทธิภาพสูงสุดสำหรับ large file processing
- **Error handling**: ทุก syscall อาจ fail ต้องตรวจสอบเสมอ

ความรู้เหล่านี้เป็นพื้นฐานที่ทุก high-level language runtime ใช้ เมื่อเข้าใจแล้ว
การ debug performance issues ของโปรแกรมใดๆ จะง่ายขึ้นมาก

**Next:** Part 054 - Network Programming: Socket syscalls, TCP/UDP server/client in Assembly
