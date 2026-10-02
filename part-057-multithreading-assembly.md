# Part 057: Multithreading ใน Assembly (x86-64 Linux)

## บทนำ (Introduction)

Multithreading ใน Assembly เป็นหัวข้อที่ลึกและซับซ้อนที่สุดหัวข้อหนึ่ง เพราะเราต้องจัดการโดยตรงกับ:
- Linux kernel syscalls สำหรับสร้าง thread (`clone`)
- การจัดการ stack สำหรับแต่ละ thread
- Thread Local Storage (TLS) ผ่าน FS/GS registers
- Atomic operations และ memory ordering
- Synchronization primitives (mutex, spinlock, futex)
- Lock-free data structures

ในภาษา C เราใช้ `pthread` library ซึ่งเป็น wrapper บน syscalls เหล่านี้
ในภาษา Assembly เราจะเรียก syscalls โดยตรง เพื่อเข้าใจกลไกภายในอย่างแท้จริง

---

## สารบัญ (Table of Contents)

1. [clone syscall และ Thread Creation](#clone-syscall)
2. [Thread Stack Allocation ด้วย mmap](#thread-stack)
3. [Thread Local Storage (TLS)](#tls)
4. [futex Syscall](#futex)
5. [Mutex Implementation](#mutex)
6. [Spinlock](#spinlock)
7. [Atomic Operations](#atomic-ops)
8. [Memory Ordering](#memory-ordering)
9. [Lock-Free SPSC Queue](#spsc-queue)
10. [โปรแกรม: Parallel Sum Array](#parallel-sum)
11. [โปรแกรม: Thread Pool](#thread-pool)
12. [โปรแกรม: Mutex-Protected Counter](#mutex-counter)
13. [โปรแกรม: Lock-Free Stack (Treiber Stack)](#treiber-stack)

---

## 1. clone syscall และ Thread Creation {#clone-syscall}

### ความแตกต่างระหว่าง fork() และ clone()

```
fork()  → สร้าง process ใหม่ (copy address space ทั้งหมด)
clone() → สร้าง thread/process ที่ share resources กับ parent ได้
```

### Flags สำหรับ Thread Creation

| Flag              | ค่า Hex     | ความหมาย                                      |
|-------------------|-------------|-----------------------------------------------|
| CLONE_VM          | 0x00000100  | Share virtual memory กับ parent               |
| CLONE_FS          | 0x00000200  | Share filesystem info (cwd, umask)            |
| CLONE_FILES       | 0x00000400  | Share file descriptor table                   |
| CLONE_SIGHAND     | 0x00000800  | Share signal handlers                         |
| CLONE_THREAD      | 0x00010000  | อยู่ใน thread group เดียวกัน (same PID)       |
| CLONE_SETTLS      | 0x00080000  | กำหนด TLS descriptor สำหรับ thread ใหม่      |
| CLONE_PARENT_SETTID| 0x00100000 | เขียน TID ลงใน parent's address              |
| CLONE_CHILD_CLEARTID| 0x00200000| Clear TID เมื่อ thread exit (futex wake)      |
| CLONE_CHILD_SETTID| 0x01000000  | เขียน TID ลงใน child's address              |

### CLONE_FLAGS รวมกัน (Typical Thread Creation)

```nasm
; CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD | CLONE_SETTLS
; = 0x100 | 0x200 | 0x400 | 0x800 | 0x10000 | 0x80000
; = 0x90F00

THREAD_FLAGS equ 0x3D0F00
; รวม CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID | CLONE_CHILD_SETTID ด้วย
```

### Prototype ของ clone syscall

```c
// C prototype
long clone(unsigned long flags,
           void *stack,
           int *parent_tid,
           int *child_tid,
           unsigned long tls);
```

### clone syscall number

```nasm
; syscall numbers ใน x86-64 Linux
SYS_clone  equ 56
SYS_exit   equ 60
SYS_mmap   equ 9
SYS_munmap equ 11
SYS_futex  equ 202
SYS_arch_prctl equ 158
SYS_gettid equ 186
SYS_write  equ 1
SYS_read   equ 0
```

### ตัวอย่างการสร้าง Thread แรก

```nasm
; create_thread.asm - โปรแกรมสาธิตการสร้าง thread ด้วย clone syscall
; nasm -f elf64 create_thread.asm -o create_thread.o
; ld create_thread.o -o create_thread

section .data
    ; ข้อความสำหรับแต่ละ thread
    main_msg    db "Main thread กำลังทำงาน", 10, 0
    main_len    equ $ - main_msg - 1
    child_msg   db "Child thread กำลังทำงาน", 10, 0
    child_len   equ $ - child_msg - 1
    done_msg    db "Thread เสร็จสิ้น", 10, 0
    done_len    equ $ - done_msg - 1

section .bss
    child_tid   resd 1      ; เก็บ TID ของ child thread
    parent_tid  resd 1      ; เก็บ TID ของ parent

section .text
global _start

; ============================================================
; thread_func - ฟังก์ชันที่ child thread จะรัน
; ============================================================
thread_func:
    ; เขียนข้อความว่า child thread กำลังทำงาน
    mov     rax, 1          ; sys_write
    mov     rdi, 1          ; stdout
    lea     rsi, [rel child_msg]
    mov     rdx, child_len
    syscall

    ; Exit thread (ไม่ใช่ exit process!)
    ; sys_exit ใน thread จะ exit เฉพาะ thread นั้น
    mov     rax, 60         ; sys_exit
    xor     rdi, rdi        ; exit code = 0
    syscall

_start:
    ; เขียนข้อความของ main thread
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel main_msg]
    mov     rdx, main_len
    syscall

    ; === จัดสรร stack สำหรับ thread ใหม่ ===
    ; mmap(NULL, STACK_SIZE, PROT_READ|PROT_WRITE,
    ;       MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0)
    mov     rax, 9          ; sys_mmap
    xor     rdi, rdi        ; addr = NULL (kernel เลือกให้)
    mov     rsi, 0x10000    ; size = 64KB stack
    mov     rdx, 3          ; PROT_READ | PROT_WRITE
    mov     r10, 0x20022    ; MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK
    mov     r8, -1          ; fd = -1 (anonymous)
    xor     r9, r9          ; offset = 0
    syscall

    ; rax = ตำแหน่งเริ่มต้นของ stack (ต่ำสุด)
    ; stack บน x86-64 โตจากบนลงล่าง ดังนั้นต้อง + size
    mov     r12, rax        ; เก็บ base address ไว้ (เพื่อ munmap ภายหลัง)
    add     rax, 0x10000    ; ชี้ไปที่ top of stack

    ; จัด align stack ให้ 16 bytes
    and     rax, -16

    ; === เรียก clone syscall ===
    ; rdi = flags
    ; rsi = child stack pointer (top of stack)
    ; rdx = parent_tid pointer
    ; r10 = child_tid pointer
    ; r8  = TLS (0 = ไม่ใช้ TLS สำหรับตัวอย่างนี้)
    mov     rax, 56                 ; sys_clone
    mov     rdi, 0x00010F00         ; CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD
    ; rsi ยังคือ top of child stack (ตั้งค่าไว้แล้ว)
    lea     rdx, [rel parent_tid]   ; parent_tid ptr
    lea     r10, [rel child_tid]    ; child_tid ptr
    xor     r8, r8                  ; tls = 0
    syscall

    ; ถ้า rax = 0 แสดงว่าเราคือ child thread
    ; ถ้า rax > 0 แสดงว่าเราคือ parent thread (rax = child TID)
    test    rax, rax
    jz      .is_child

    ; === Parent Thread ===
    ; รอให้ child เสร็จ (ใช้ futex อย่างง่าย)
    ; ในตัวอย่างนี้ใช้การ sleep แบบง่าย
    mov     r13, rax        ; เก็บ child TID

    ; เขียนข้อความ done
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel done_msg]
    mov     rdx, done_len
    syscall

    ; Exit program
    mov     rax, 60
    xor     rdi, rdi
    syscall

.is_child:
    ; child thread จะทำงานที่นี่
    jmp     thread_func
```

---

## 2. Thread Stack Allocation ด้วย mmap {#thread-stack}

### ทำไมต้องใช้ mmap?

Thread แต่ละตัวต้องการ stack เป็นของตัวเอง เพราะ:
- Stack เก็บ return addresses, local variables, saved registers
- ถ้า stack ทับกัน โปรแกรมจะ crash หรือทำงานผิด

### การคำนวณ Stack Size

```nasm
; ขนาด stack ทั่วไป
STACK_SIZE_4K   equ 0x1000      ; 4 KB  - น้อยมาก
STACK_SIZE_64K  equ 0x10000     ; 64 KB - เหมาะสำหรับ thread ธรรมดา
STACK_SIZE_1M   equ 0x100000    ; 1 MB  - เหมาะสำหรับ recursive
STACK_SIZE_8M   equ 0x800000    ; 8 MB  - default ของ Linux

; mmap flags
PROT_READ     equ 1
PROT_WRITE    equ 2
MAP_PRIVATE   equ 2
MAP_ANONYMOUS equ 0x20
MAP_STACK     equ 0x20000      ; บอก kernel ว่าใช้เป็น stack
MAP_GROWSDOWN equ 0x100        ; stack โตลงล่าง
```

### ฟังก์ชัน alloc_stack

```nasm
; ============================================================
; alloc_stack - จัดสรร stack สำหรับ thread ใหม่
; Input:  rdi = ขนาด stack ที่ต้องการ (bytes)
; Output: rax = pointer ไปที่ TOP of stack (ใช้เป็น sp ของ thread ใหม่)
;         rbx = pointer ไปที่ BASE of stack (เพื่อ munmap ภายหลัง)
; Clobbers: rcx, rdx, r8, r9, r10
; ============================================================
alloc_stack:
    push    rbp
    mov     rbp, rsp
    push    rbx

    ; บันทึก size ไว้
    mov     rbx, rdi

    ; mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0)
    mov     rax, 9          ; sys_mmap
    xor     rdi, rdi        ; addr = NULL
    mov     rsi, rbx        ; size
    mov     rdx, 3          ; PROT_READ | PROT_WRITE
    mov     r10, 0x20022    ; MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK
    mov     r8, -1          ; fd = -1
    xor     r9, r9          ; offset = 0
    syscall

    ; ตรวจสอบ error (mmap คืนค่าลบถ้า error)
    cmp     rax, -4096
    jae     .mmap_error

    ; คำนวณ top of stack
    mov     rbx, rax        ; เก็บ base address
    add     rax, [rbp-8]    ; top = base + size (ดึง size จาก stack)

    ; Align to 16 bytes (ABI requirement)
    and     rax, -16

    pop     rbx
    pop     rbp
    ret

.mmap_error:
    ; คืน NULL เมื่อเกิด error
    xor     rax, rax
    pop     rbx
    pop     rbp
    ret

; ============================================================
; free_stack - คืน stack ที่จัดสรรไว้
; Input:  rdi = base address ของ stack
;         rsi = size ของ stack
; ============================================================
free_stack:
    mov     rax, 11         ; sys_munmap
    syscall                 ; rdi = addr, rsi = length ตั้งค่าแล้ว
    ret
```

### Guard Page - ป้องกัน Stack Overflow

```nasm
; ============================================================
; alloc_stack_with_guard - จัดสรร stack พร้อม guard page
; Guard page คือหน้า memory ที่ไม่มี permission ใดๆ
; ถ้า stack overflow ลงมาถึง guard page จะเกิด SIGSEGV
; ============================================================
alloc_stack_with_guard:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12

    mov     r12, rdi        ; เก็บ requested size

    ; คำนวณ total size = stack_size + guard_page_size
    add     rdi, 0x1000     ; เพิ่ม guard page (4KB)

    ; mmap ทั้งหมดก่อน
    mov     rax, 9
    xor     rdi, rdi
    mov     rsi, r12
    add     rsi, 0x1000     ; total size
    mov     rdx, 3          ; PROT_READ | PROT_WRITE
    mov     r10, 0x20022    ; MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK
    mov     r8, -1
    xor     r9, r9
    syscall

    mov     rbx, rax        ; เก็บ base

    ; mprotect guard page ให้ PROT_NONE (ไม่มี permission)
    ; guard page อยู่ที่ต่ำสุด (stack โตจากบนลงล่าง)
    mov     rax, 10         ; sys_mprotect
    mov     rdi, rbx        ; addr = base (bottom of allocation)
    mov     rsi, 0x1000     ; size = 4KB (guard page)
    xor     rdx, rdx        ; PROT_NONE = 0
    syscall

    ; top of usable stack = base + total_size
    mov     rax, rbx
    add     rax, r12
    add     rax, 0x1000
    and     rax, -16        ; align

    pop     r12
    pop     rbx
    pop     rbp
    ret
```

---

## 3. Thread Local Storage (TLS) {#tls}

### TLS คืออะไร?

Thread Local Storage (TLS) คือ memory ที่แต่ละ thread มีสำเนาเป็นของตัวเอง
ตัวแปรที่ประกาศเป็น `__thread` ใน C จะถูกเก็บใน TLS

### FS Register และ arch_prctl

ใน x86-64 Linux:
- **FS register** ชี้ไปยัง Thread Control Block (TCB) ของ thread ปัจจุบัน
- ใช้ `arch_prctl(ARCH_SET_FS, addr)` เพื่อกำหนด base address ของ FS

```nasm
; arch_prctl constants
ARCH_SET_GS     equ 0x1001
ARCH_SET_FS     equ 0x1002
ARCH_GET_FS     equ 0x1003
ARCH_GET_GS     equ 0x1004
```

### TLS Block Structure

```
TLS Block Layout (ใช้ variant 2 - glibc):

High address
+------------------+
| pthread struct   |  ← FS points here (offset 0)
+------------------+
| TLS vars block 1 |
+------------------+
| TLS vars block 2 |
+------------------+
| ...              |
Low address
```

### การตั้งค่า TLS อย่างง่าย

```nasm
; tls_example.asm - ตัวอย่างการใช้ TLS ใน Assembly
section .data
    tls_setup_msg   db "กำลังตั้งค่า TLS...", 10, 0
    tls_read_msg    db "อ่านค่า TLS สำเร็จ", 10, 0

section .bss
    ; Thread Local Block structure อย่างง่าย
    ; ในโปรแกรมจริงนี้คือ struct สำหรับแต่ละ thread
    tls_block_size  equ 256

section .text

; ============================================================
; setup_tls - กำหนด FS register ให้ชี้ไปยัง TLS block
; Input:  rdi = pointer ไปยัง TLS block
; ============================================================
setup_tls:
    push    rbp
    mov     rbp, rsp

    mov     rax, 158        ; sys_arch_prctl
    mov     rdi, 0x1002     ; ARCH_SET_FS
    ; rsi = address ของ TLS block (ตั้งค่าโดย caller)
    syscall

    ; ตรวจสอบ error
    test    rax, rax
    js      .error

    pop     rbp
    ret

.error:
    ; error handling
    pop     rbp
    ret

; ============================================================
; get_tls_base - อ่านค่า FS base address ปัจจุบัน
; Output: rax = FS base address
; ============================================================
get_tls_base:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 8

    mov     rax, 158        ; sys_arch_prctl
    mov     rdi, 0x1003     ; ARCH_GET_FS
    lea     rsi, [rsp]      ; pointer รับค่า
    syscall

    mov     rax, [rsp]      ; อ่านค่าที่เก็บไว้

    leave
    ret

; ============================================================
; get_thread_id - อ่าน Thread ID ของ thread ปัจจุบัน
; Output: rax = TID
; ============================================================
get_thread_id:
    mov     rax, 186        ; sys_gettid
    syscall
    ret
```

### TLS Structure สำหรับ Thread Pool

```nasm
; ============================================================
; Thread Control Block (TCB) - โครงสร้างสำหรับจัดการ thread
; ============================================================

struc TCB
    .self       resq 1      ; pointer ไปหาตัวเอง (offset 0, FS:0x00)
    .tid        resd 1      ; Thread ID
    .state      resd 1      ; Thread state (RUNNING, SLEEPING, etc.)
    .stack_base resq 1      ; base ของ stack ที่จัดสรร
    .stack_size resq 1      ; ขนาด stack
    .errno_val  resd 1      ; per-thread errno
    .padding    resd 1      ; padding สำหรับ alignment
    .user_data  resq 1      ; ข้อมูลเพิ่มเติมของ user
    .size       equ $ - TCB
endstruc

; Thread states
THREAD_IDLE     equ 0
THREAD_RUNNING  equ 1
THREAD_SLEEPING equ 2
THREAD_DEAD     equ 3

; ============================================================
; create_tcb - สร้าง Thread Control Block ใหม่
; Input:  rdi = stack_base, rsi = stack_size
; Output: rax = pointer ไปยัง TCB ที่สร้าง
; ============================================================
create_tcb:
    push    rbp
    mov     rbp, rsp
    push    r12
    push    r13

    mov     r12, rdi        ; เก็บ stack_base
    mov     r13, rsi        ; เก็บ stack_size

    ; จัดสรรหน่วยความจำสำหรับ TCB
    mov     rax, 9          ; sys_mmap
    xor     rdi, rdi
    mov     rsi, TCB.size
    mov     rdx, 3          ; PROT_READ|PROT_WRITE
    mov     r10, 0x22       ; MAP_PRIVATE|MAP_ANONYMOUS
    mov     r8, -1
    xor     r9, r9
    syscall

    ; กรอกข้อมูล TCB
    mov     [rax + TCB.self], rax       ; self pointer
    mov     [rax + TCB.stack_base], r12
    mov     [rax + TCB.stack_size], r13
    mov     dword [rax + TCB.state], THREAD_IDLE

    pop     r13
    pop     r12
    pop     rbp
    ret

; เข้าถึง TLS ผ่าน FS register:
; mov rax, fs:[0x00]    ; อ่าน self pointer
; mov rax, fs:[0x08]    ; อ่าน TID area
; mov dword [fs:0x10], eax  ; เขียน state
```

---

## 4. futex Syscall {#futex}

### futex คืออะไร?

**futex** (Fast Userspace muTEX) คือ synchronization primitive ที่รวด็วที่สุดใน Linux
- ถ้า lock ว่าง: ทำงานใน userspace เลย ไม่ต้อง syscall
- ถ้า lock ถูกครอง: เรียก kernel ให้ thread รอ (sleep)

### futex Operations

```nasm
; futex operations constants
FUTEX_WAIT          equ 0   ; รอจนกว่าค่าจะเปลี่ยน
FUTEX_WAKE          equ 1   ; ปลุก thread ที่รออยู่
FUTEX_FD            equ 2   ; ได้ file descriptor สำหรับ poll (deprecated)
FUTEX_REQUEUE       equ 3   ; ย้าย waiter ไปยัง futex อื่น
FUTEX_CMP_REQUEUE   equ 4   ; REQUEUE พร้อม compare
FUTEX_WAKE_OP       equ 5   ; WAKE พร้อม operation
FUTEX_LOCK_PI       equ 6   ; Priority inheritance lock
FUTEX_UNLOCK_PI     equ 7   ; Priority inheritance unlock
FUTEX_TRYLOCK_PI    equ 8
FUTEX_WAIT_BITSET   equ 9
FUTEX_WAKE_BITSET   equ 10
FUTEX_PRIVATE_FLAG  equ 128 ; เพิ่ม flag นี้สำหรับ process-private futex (เร็วกว่า)

; Private versions (ใช้บ่อยที่สุด)
FUTEX_WAIT_PRIVATE  equ (FUTEX_WAIT | FUTEX_PRIVATE_FLAG)
FUTEX_WAKE_PRIVATE  equ (FUTEX_WAKE | FUTEX_PRIVATE_FLAG)
```

### futex syscall signature

```c
// C prototype
int futex(uint32_t *uaddr,      // pointer ไปยัง futex variable
          int futex_op,          // operation (FUTEX_WAIT, FUTEX_WAKE, etc.)
          uint32_t val,          // ค่าที่ใช้เปรียบเทียบ (WAIT) หรือจำนวน wakeup (WAKE)
          const struct timespec *timeout, // timeout (NULL = ไม่ timeout)
          uint32_t *uaddr2,      // สำหรับ REQUEUE operations
          uint32_t val3);        // สำหรับ CMP_REQUEUE
```

### ฟังก์ชัน futex_wait และ futex_wake

```nasm
; ============================================================
; futex_wait - รอจนกว่า futex value จะไม่ใช่ expected_val
; Input:  rdi = pointer ไปยัง futex variable (uint32_t*)
;         esi = expected value (ถ้าค่าปัจจุบันไม่ตรง จะ return ทันที)
; Output: rax = 0 ถ้าถูก wake, -EAGAIN ถ้าค่าไม่ตรง
; ============================================================
futex_wait:
    push    rbp
    mov     rbp, rsp

    ; sys_futex(uaddr, FUTEX_WAIT_PRIVATE, val, NULL, NULL, 0)
    mov     rax, 202                ; sys_futex
    ; rdi = uaddr (ตั้งค่าแล้ว)
    mov     rsi, FUTEX_WAIT_PRIVATE ; op
    ; rdx = val (esi ที่ pass เข้ามา - ต้อง extend เป็น rdx)
    movzx   rdx, esi               ; zero extend val to rdx
    xor     r10, r10               ; timeout = NULL (รอตลอดไป)
    xor     r8, r8                 ; uaddr2 = NULL
    xor     r9, r9                 ; val3 = 0
    syscall

    pop     rbp
    ret

; ============================================================
; futex_wake - ปลุก thread ที่กำลังรออยู่บน futex
; Input:  rdi = pointer ไปยัง futex variable
;         esi = จำนวน thread ที่จะปลุก (INT_MAX = ปลุกทั้งหมด)
; Output: rax = จำนวน thread ที่ถูกปลุก
; ============================================================
futex_wake:
    push    rbp
    mov     rbp, rsp

    ; sys_futex(uaddr, FUTEX_WAKE_PRIVATE, val, NULL, NULL, 0)
    mov     rax, 202                ; sys_futex
    ; rdi = uaddr
    mov     rsi, FUTEX_WAKE_PRIVATE ; op
    movzx   rdx, esi               ; val = จำนวนที่จะปลุก
    xor     r10, r10               ; timeout = NULL
    xor     r8, r8
    xor     r9, r9
    syscall

    pop     rbp
    ret

; ============================================================
; futex_requeue - ย้าย waiter จาก futex1 ไปยัง futex2
; ใช้สำหรับ condition variable implementation
; Input:  rdi = futex1 address
;         rsi = futex2 address
;         edx = จำนวน thread ที่จะ wake จาก futex1
;         ecx = จำนวน thread ที่จะ requeue ไปยัง futex2
; ============================================================
futex_requeue:
    push    rbp
    mov     rbp, rsp

    mov     r10, rsi        ; uaddr2 = futex2
    mov     rsi, FUTEX_REQUEUE | FUTEX_PRIVATE_FLAG
    ; rdx = wake count, r10 = uaddr2, r9 = requeue count (ecx)
    movzx   r9, ecx
    mov     rax, 202
    syscall

    pop     rbp
    ret
```

---

## 5. Mutex Implementation {#mutex}

### Mutex States

```
State = 0 : UNLOCKED (ว่าง)
State = 1 : LOCKED, no waiters (ถูกครอง, ไม่มีคนรอ)
State = 2 : LOCKED, with waiters (ถูกครอง, มีคนรอ)
```

### Mutex Structure

```nasm
struc MUTEX
    .state  resd 1      ; 0=unlocked, 1=locked, 2=locked+waiters
    .owner  resd 1      ; TID ของ thread ที่ครอง mutex
    .size   equ $ - MUTEX
endstruc

MUTEX_UNLOCKED  equ 0
MUTEX_LOCKED    equ 1
MUTEX_CONTESTED equ 2
```

### Mutex Lock

```nasm
; ============================================================
; mutex_lock - ล็อค mutex (blocking)
; Input:  rdi = pointer ไปยัง mutex
; Clobbers: rax, rcx, rdx
;
; Algorithm:
;   1. ลอง CAS: 0 → 1 (fast path, ไม่มี syscall)
;   2. ถ้าไม่สำเร็จ: set state = 2 (มี waiters)
;   3. เรียก futex_wait จนกว่าจะ lock ได้
; ============================================================
mutex_lock:
    push    rbp
    mov     rbp, rsp
    push    rbx

    mov     rbx, rdi        ; เก็บ mutex pointer

    ; === Fast path: ลอง lock โดยไม่ต้องเรียก kernel ===
    ; cmpxchg [mutex], 1  ถ้า mutex == 0 → set = 1
    xor     eax, eax        ; expected = 0 (UNLOCKED)
    mov     ecx, 1          ; new value = 1 (LOCKED)
    lock cmpxchg dword [rbx + MUTEX.state], ecx
    jz      .locked         ; ZF=1 หมายถึง swap สำเร็จ (lock ได้)

    ; === Slow path: ต้องรอ kernel ===
.wait_loop:
    ; Set state = 2 (CONTESTED) ก่อน sleep
    ; ใช้ xchg เพื่อ atomically set และอ่านค่าเดิม
    mov     eax, 2          ; CONTESTED
    xchg    dword [rbx + MUTEX.state], eax

    ; ถ้าค่าเดิมเป็น 0 (unlocked) แสดงว่าเราได้ lock แล้ว!
    test    eax, eax
    jz      .locked

    ; เรียก futex_wait: รอจนกว่าค่าจะไม่ใช่ 2
    mov     rax, 202        ; sys_futex
    mov     rdi, rbx        ; uaddr = mutex
    mov     rsi, FUTEX_WAIT_PRIVATE
    mov     rdx, 2          ; val = 2 (รอเฉพาะเมื่อ state == 2)
    xor     r10, r10        ; timeout = NULL
    xor     r8, r8
    xor     r9, r9
    syscall
    ; syscall อาจ return ด้วย EAGAIN (state เปลี่ยนก่อน sleep)
    ; หรือ return 0 (ถูก wake)
    ; ในทั้งสองกรณี ลอง lock ใหม่

    ; ลอง lock ใหม่: set state = 2 (ยังคง contested)
    mov     rdi, rbx        ; restore mutex pointer
    jmp     .wait_loop

.locked:
    ; บันทึก owner TID
    mov     rax, 186        ; sys_gettid
    syscall
    mov     [rbx + MUTEX.owner], eax

    pop     rbx
    pop     rbp
    ret

; ============================================================
; mutex_trylock - ลอง lock mutex โดยไม่ blocking
; Input:  rdi = pointer ไปยัง mutex
; Output: rax = 0 ถ้า lock สำเร็จ, 1 ถ้า mutex ถูกครอง
; ============================================================
mutex_trylock:
    xor     eax, eax        ; expected = 0
    mov     ecx, 1          ; new = 1
    lock cmpxchg dword [rdi + MUTEX.state], ecx
    jz      .success

    mov     eax, 1          ; คืน 1 = ล้มเหลว
    ret

.success:
    ; บันทึก owner
    push    rdi
    mov     rax, 186        ; sys_gettid
    syscall
    pop     rdi
    mov     [rdi + MUTEX.owner], eax
    xor     eax, eax        ; คืน 0 = สำเร็จ
    ret

; ============================================================
; mutex_unlock - ปล่อย mutex
; Input:  rdi = pointer ไปยัง mutex
;
; Algorithm:
;   1. Set state = 0 atomically (ลบ owner ด้วย)
;   2. ถ้าค่าเดิม == 2 (มี waiters): เรียก futex_wake(1)
; ============================================================
mutex_unlock:
    push    rbp
    mov     rbp, rsp
    push    rbx

    mov     rbx, rdi

    ; Clear owner
    mov     dword [rbx + MUTEX.owner], 0

    ; Atomically set state = 0 และดูว่าค่าเดิมเป็นอะไร
    ; ถ้าเดิม = 1 (locked, no waiters) → แค่ unlock ก็พอ
    ; ถ้าเดิม = 2 (contested) → ต้อง wake waiter
    mov     eax, 1          ; คาดว่า = 1 (LOCKED)
    xor     ecx, ecx        ; new = 0 (UNLOCKED)
    lock cmpxchg dword [rbx + MUTEX.state], ecx
    jz      .done           ; ถ้า state เป็น 1 → unlock สำเร็จ ไม่มี waiter

    ; state เป็น 2 (contested) → ต้อง wake
    ; ก่อนอื่น set state = 0
    mov     dword [rbx + MUTEX.state], 0

    ; ปลุก 1 thread ที่รออยู่
    mov     rax, 202        ; sys_futex
    mov     rdi, rbx        ; uaddr = mutex
    mov     rsi, FUTEX_WAKE_PRIVATE
    mov     rdx, 1          ; wake 1 thread
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

.done:
    pop     rbx
    pop     rbp
    ret
```

---

## 6. Spinlock {#spinlock}

### Spinlock vs Mutex

| ลักษณะ        | Spinlock                    | Mutex                      |
|---------------|----------------------------|---------------------------|
| การรอ         | วน loop (ใช้ CPU)           | Sleep (ไม่ใช้ CPU)         |
| เหมาะสำหรับ  | Critical section สั้นมาก   | Critical section ยาว       |
| Context switch| ไม่มี                       | มี (kernel involvement)    |
| Latency       | ต่ำมาก                      | สูงกว่า                    |
| CPU usage     | สูง (busy-wait)             | ต่ำ                        |

### Spinlock ด้วย XCHG (Test-and-Set)

```nasm
struc SPINLOCK
    .lock   resd 1      ; 0=unlocked, 1=locked
    .pad    resd 3      ; padding เพื่อหลีกเลี่ยง false sharing
    .size   equ $ - SPINLOCK
endstruc

; ============================================================
; spinlock_lock - ล็อค spinlock
; Input:  rdi = pointer ไปยัง spinlock
;
; LOCK XCHG: atomic exchange ระหว่าง register กับ memory
; LOCK prefix ไม่จำเป็นสำหรับ XCHG (implicit lock) แต่ใส่ไว้เพื่อชัดเจน
; ============================================================
spinlock_lock:
    mov     eax, 1          ; ค่าที่จะ set (LOCKED)
.retry:
    ; xchg: atomically swap eax กับ [rdi]
    ; ถ้า [rdi] เดิมเป็น 0 (unlocked): eax = 0, [rdi] = 1 → lock สำเร็จ
    ; ถ้า [rdi] เดิมเป็น 1 (locked):   eax = 1, [rdi] = 1 → ยังล็อคอยู่
    xchg    dword [rdi], eax
    test    eax, eax
    jz      .locked         ; ถ้าค่าเดิมเป็น 0 → เราได้ lock

    ; === Backoff: ลดการแย่ง cache line ===
    ; PAUSE บอก CPU ว่า "นี่คือ spin loop"
    ; CPU จะลด power และให้ SMT thread อื่นทำงานได้
.spin:
    ; อ่านค่าแบบ non-atomic ก่อน (เพื่อหลีกเลี่ยง lock bus livelocks)
    pause                   ; PAUSE instruction - สำคัญมากใน spin loop!
    mov     eax, [rdi]      ; อ่านค่าปัจจุบัน
    test    eax, eax
    jnz     .spin           ; ถ้ายัง locked → ยังคง spin

    ; ดูเหมือน unlock แล้ว → ลอง XCHG ใหม่
    mov     eax, 1
    jmp     .retry

.locked:
    ret

; ============================================================
; spinlock_unlock - ปลดล็อค spinlock
; Input:  rdi = pointer ไปยัง spinlock
; ============================================================
spinlock_unlock:
    ; Memory fence ก่อน unlock เพื่อให้แน่ใจว่า
    ; การเขียน critical section สิ้นสุดก่อน
    ; ใน x86 store-to-store ordering ถูกรับประกัน แต่ explicit fence ชัดเจนกว่า
    mov     dword [rdi], 0  ; Set to 0 (UNLOCKED) - store ใน x86 เป็น release อยู่แล้ว
    ret

; ============================================================
; spinlock_trylock - ลอง lock โดยไม่ blocking
; Input:  rdi = pointer ไปยัง spinlock
; Output: rax = 0 สำเร็จ, 1 ล้มเหลว
; ============================================================
spinlock_trylock:
    mov     eax, 1
    xchg    dword [rdi], eax
    ; eax = ค่าเดิม: 0 = ได้ lock, 1 = ไม่ได้ lock
    ret
```

### Spinlock ด้วย CMPXCHG (Compare-and-Swap)

```nasm
; ============================================================
; spinlock_lock_cas - ใช้ CMPXCHG แทน XCHG
; CMPXCHG ดีกว่า XCHG เพราะ:
; - ไม่ต้อง xchg เมื่อ lock ถูกครอง (ลด cache line invalidation)
; ============================================================
spinlock_lock_cas:
    xor     eax, eax        ; expected = 0 (UNLOCKED)
    mov     ecx, 1          ; new = 1 (LOCKED)
.retry:
    lock cmpxchg dword [rdi], ecx
    jz      .locked         ; ZF=1: swap สำเร็จ

    ; Spin with backoff
.spin:
    pause
    cmp     dword [rdi], 0  ; รอให้ unlock ก่อน
    jnz     .spin

    xor     eax, eax        ; reset expected
    jmp     .retry

.locked:
    ret
```

### Ticket Spinlock (Fair Lock)

```nasm
; ============================================================
; Ticket Spinlock - ป้องกัน starvation
; ทำงานเหมือน queue ตั๋ว: แต่ละ thread ได้หมายเลขตั๋ว
; เมื่อถึงหมายเลขของตัวเอง จึงจะได้ lock
; ============================================================
struc TICKET_LOCK
    .next_ticket    resw 1  ; ticket ถัดไปที่จะออก
    .serving        resw 1  ; ticket ที่กำลัง serve อยู่
    .pad            resd 1
    .size           equ $ - TICKET_LOCK
endstruc

; ============================================================
; ticket_lock_acquire - รับ ticket และรอคิว
; Input:  rdi = pointer ไปยัง ticket lock
; ============================================================
ticket_lock_acquire:
    ; Atomically increment next_ticket และได้ค่าเดิม
    ; LOCK XADD: atomic add + return old value
    mov     ax, 1
    lock xadd word [rdi + TICKET_LOCK.next_ticket], ax
    ; ax = ticket ที่เราได้รับ

.wait:
    ; รอจนกว่า serving == ticket ของเรา
    cmp     ax, word [rdi + TICKET_LOCK.serving]
    je      .served
    pause
    jmp     .wait

.served:
    ret

; ============================================================
; ticket_lock_release - ปล่อย lock (increment serving)
; Input:  rdi = pointer ไปยัง ticket lock
; ============================================================
ticket_lock_release:
    ; เพิ่ม serving counter → thread ถัดไปในคิวจะได้ lock
    lock inc word [rdi + TICKET_LOCK.serving]
    ret
```

---

## 7. Atomic Operations {#atomic-ops}

### PAUSE Instruction

```nasm
; PAUSE - ใช้ใน spin loop เพื่อ:
; 1. บอก CPU ว่านี่คือ spin-wait loop
; 2. ลด power consumption
; 3. ให้ Hyper-Threading thread อื่นใช้ execution resources
; 4. ป้องกัน memory order violation penalty

spin_wait_example:
    ; ตัวอย่าง spin loop ที่ถูกต้อง
.loop:
    pause               ; สำคัญ! ขาด pause = ประสิทธิภาพแย่มาก
    mov     eax, [rdi]  ; อ่านค่าที่รอ
    test    eax, eax
    jz      .loop
    ret
```

### CMPXCHG - Compare and Swap (CAS)

```nasm
; ============================================================
; LOCK CMPXCHG - Compare and Exchange
;
; Pseudocode:
;   if (*dest == expected) {
;       *dest = new_val;
;       ZF = 1;
;   } else {
;       expected_register = *dest;  // อ่านค่าจริงออกมา
;       ZF = 0;
;   }
;
; สำคัญ: CMPXCHG ใช้ AL/AX/EAX/RAX เป็น expected value
; ============================================================

; ตัวอย่าง: atomic increment
; Input:  rdi = pointer ไปยัง counter (int64)
; Output: rax = ค่าหลังจาก increment
atomic_increment:
    mov     rax, [rdi]      ; อ่านค่าปัจจุบัน
.retry:
    lea     rcx, [rax+1]    ; new = old + 1
    lock cmpxchg [rdi], rcx ; ถ้า [rdi] == rax → set [rdi] = rcx
    jnz     .retry          ; ZF=0 หมายถึงมีคนอื่นเปลี่ยนค่า → ลองใหม่
    ; rax = ค่าเดิม (ก่อน increment)
    inc     rax             ; คืนค่าหลัง increment
    ret

; CMPXCHG16B - 128-bit Compare and Swap (ใช้สำหรับ lock-free structures)
; ต้องการ: rdx:rax = expected, rcx:rbx = new value
; ถ้า [rdi] == rdx:rax → [rdi] = rcx:rbx
; ===============================================
; CMPXCHG16B ต้องการ LOCK prefix เสมอเมื่อใช้ multi-thread

atomic_cas128:
    ; rdi = address (16-byte aligned!)
    ; rcx:rbx = new value (high:low)
    ; rdx:rax = expected (set โดย caller)
    lock cmpxchg16b [rdi]
    ret
```

### XADD - Exchange and Add

```nasm
; ============================================================
; LOCK XADD - Atomic Add and Return Old Value
;
; Pseudocode:
;   old = *dest;
;   *dest += src;
;   src = old;  (src register ได้ค่าเดิม)
;
; มีประโยชน์สำหรับ: reference counting, ticket locks,
;                     atomic fetch-and-add
; ============================================================

; ตัวอย่าง: atomic fetch-and-add
; Input:  rdi = pointer ไปยัง counter
;         rsi = ค่าที่ต้องการบวก
; Output: rax = ค่าเดิม (ก่อนบวก)
atomic_fetch_add:
    mov     rax, rsi
    lock xadd [rdi], rax    ; [rdi] += rax, rax = ค่าเดิม
    ret

; ตัวอย่าง: reference counting
; Input:  rdi = pointer ไปยัง ref count
; Output: rax = ref count ใหม่
ref_count_inc:
    mov     eax, 1
    lock xadd dword [rdi], eax
    inc     eax             ; คืนค่าหลัง increment
    ret

ref_count_dec:
    mov     eax, -1
    lock xadd dword [rdi], eax
    dec     eax             ; คืนค่าหลัง decrement
    ret
```

### LOCK Prefix กับคำสั่งต่างๆ

```nasm
; ============================================================
; LOCK prefix - บังคับให้คำสั่งเป็น atomic
; ใช้ได้กับ: ADD, SUB, INC, DEC, AND, OR, XOR, NOT, NEG,
;             XCHG, CMPXCHG, CMPXCHG8B/16B, XADD, BTC/BTS/BTR
;
; XCHG ไม่ต้องใส่ LOCK เพราะ implicit lock อยู่แล้ว
; แต่ใส่ก็ไม่ผิด เพื่อความชัดเจน
; ============================================================

; Atomic operations ตัวอย่าง:
atomic_examples:
    ; ADD
    lock add  dword [rdi], 5        ; atomic: *ptr += 5
    lock add  qword [rdi], rax      ; atomic: *ptr += rax

    ; SUB
    lock sub  dword [rdi], 1        ; atomic: *ptr -= 1

    ; INC / DEC (เท่ากับ ADD/SUB 1)
    lock inc  dword [rdi]           ; atomic: (*ptr)++
    lock dec  qword [rdi]           ; atomic: (*ptr)--

    ; AND / OR / XOR
    lock and  dword [rdi], 0xFF     ; atomic: *ptr &= 0xFF
    lock or   dword [rdi], 0x01     ; atomic: *ptr |= 0x01
    lock xor  dword [rdi], 0xFF     ; atomic: *ptr ^= 0xFF

    ; BTS - Bit Test and Set (atomic)
    lock bts  dword [rdi], 5        ; atomically set bit 5
    ; CF = ค่าเดิมของ bit

    ; BTR - Bit Test and Reset (atomic)
    lock btr  dword [rdi], 3        ; atomically clear bit 3
    ; CF = ค่าเดิมของ bit

    ret
```

---

## 8. Memory Ordering {#memory-ordering}

### ทำไม Memory Ordering สำคัญ?

CPU และ compiler สามารถ reorder instructions เพื่อเพิ่มประสิทธิภาพ
ซึ่งอาจทำให้ multi-threaded code ทำงานผิดพลาดได้

### Memory Barrier Types

```
Store Buffer:  CPU เก็บ store operations ไว้ใน buffer ก่อน flush ไปยัง cache
Invalidation Queue: CPU เก็บ cache invalidation requests ไว้ก่อนประมวลผล
```

### MFENCE, SFENCE, LFENCE

```nasm
; ============================================================
; MFENCE - Memory Fence (Full Barrier)
; รับประกันว่า memory operations ก่อน MFENCE สิ้นสุดก่อน
; memory operations หลัง MFENCE เริ่มต้น
; ทั้ง loads และ stores
; ============================================================
mfence_example:
    ; === ตัวอย่าง: กำหนดค่าตัวแปรแล้ว set flag ===
    mov     qword [data_ptr], 42    ; เขียน data
    mfence                          ; รับประกันว่า data ถูกเขียนก่อน
    mov     dword [ready_flag], 1   ; set flag

    ; Thread อื่น:
    ; ถ้าเห็น ready_flag = 1 จะเห็น data = 42 ด้วยเสมอ
    ret

; ============================================================
; SFENCE - Store Fence
; รับประกันว่า store operations ก่อน SFENCE สิ้นสุดก่อน
; store operations หลัง SFENCE
; ใช้กับ SSE non-temporal stores (MOVNTQ, MOVNTPS, etc.)
; ============================================================
sfence_example:
    ; Non-temporal stores ไม่ผ่าน cache
    ; ต้องใช้ SFENCE เพื่อรับประกัน ordering
    movntq  [rdi], mm0      ; Non-temporal store
    movntq  [rdi+8], mm1
    sfence                  ; รับประกัน stores สิ้นสุดก่อน
    ret

; ============================================================
; LFENCE - Load Fence
; รับประกันว่า load operations ก่อน LFENCE สิ้นสุดก่อน
; load operations หลัง LFENCE
; ใช้บ่อยในการป้องกัน Spectre attacks
; ============================================================
lfence_example:
    ; อ่านข้อมูลที่อาจเป็น speculative
    mov     rax, [rdi]
    lfence                  ; หยุด speculative execution
    ; ใช้ rax อย่างปลอดภัย
    ret

; ============================================================
; ใน x86/x86-64 memory model:
; - Loads ไม่ถูก reorder กัน
; - Stores ไม่ถูก reorder กัน (store-store order)
; - Load ไม่ถูก reorder กับ earlier store (load-acquire behavior)
; - Store CAN be reorder หลัง later load (store-load weak order)
;
; ดังนั้น MFENCE ใช้เพื่อป้องกัน store-load reordering เป็นหลัก
; ============================================================
```

### Memory Ordering ใน Practice

```nasm
; ============================================================
; ตัวอย่าง: Producer-Consumer ที่ถูกต้อง
; ============================================================

; Thread ฝั่ง Producer (เขียนข้อมูล แล้ว set flag)
producer_store:
    ; เขียนข้อมูลลงใน buffer
    mov     rax, [data_value]
    mov     [buffer + 0], rax   ; เขียน element 1

    mov     rax, [data_value + 8]
    mov     [buffer + 8], rax   ; เขียน element 2

    ; === MFENCE รับประกันว่า writes ข้างบนสิ้นสุดก่อน flag ===
    mfence

    ; Set ready flag
    mov     dword [ready], 1

    ret

; Thread ฝั่ง Consumer (รอ flag แล้วอ่านข้อมูล)
consumer_load:
.wait:
    mov     eax, [ready]
    test    eax, eax
    jz      .wait               ; รอ flag

    ; === LFENCE รับประกัน ordering ของ loads ===
    lfence

    ; อ่านข้อมูล (มั่นใจว่า producer เขียนเสร็จแล้ว)
    mov     rax, [buffer + 0]
    mov     rbx, [buffer + 8]

    ret
```

---

## 9. Lock-Free SPSC Queue {#spsc-queue}

### SPSC Queue คืออะไร?

**Single Producer Single Consumer (SPSC) Queue** คือ ring buffer ที่:
- มี 1 thread สำหรับเขียน (producer)
- มี 1 thread สำหรับอ่าน (consumer)
- ไม่ต้องใช้ lock! (lock-free)

### โครงสร้าง SPSC Queue

```nasm
; ============================================================
; SPSC Ring Buffer - Lock-Free Queue
; ============================================================

; ขนาด queue ต้องเป็น power of 2 เพื่อใช้ AND masking
SPSC_SIZE   equ 1024                ; ต้องเป็น power of 2
SPSC_MASK   equ (SPSC_SIZE - 1)    ; 0x3FF

struc SPSC_QUEUE
    ; === Cache line 0: Producer data ===
    .head       resq 1  ; write index (producer ใช้)
    .pad_head   resb 56 ; padding ให้ครบ 64 bytes (cache line)

    ; === Cache line 1: Consumer data ===
    .tail       resq 1  ; read index (consumer ใช้)
    .pad_tail   resb 56

    ; === Cache line 2+: Shared read-only config ===
    .capacity   resq 1  ; ขนาด buffer
    .mask       resq 1  ; mask = capacity - 1
    .pad_cfg    resb 48

    ; === Data buffer (separate allocation) ===
    .buffer     resq 1  ; pointer ไปยัง data array
    .size       equ $ - SPSC_QUEUE
endstruc

; หมายเหตุ: การแยก head และ tail ไว้คนละ cache line
; ป้องกัน False Sharing (สองอ่าน/เขียน cache line เดียวกัน)
```

### SPSC Enqueue และ Dequeue

```nasm
; ============================================================
; spsc_init - เริ่มต้น SPSC queue
; Input:  rdi = pointer ไปยัง SPSC_QUEUE struct
;         rsi = pointer ไปยัง buffer (ขนาด = SPSC_SIZE * 8 bytes)
; ============================================================
spsc_init:
    mov     qword [rdi + SPSC_QUEUE.head], 0
    mov     qword [rdi + SPSC_QUEUE.tail], 0
    mov     qword [rdi + SPSC_QUEUE.capacity], SPSC_SIZE
    mov     qword [rdi + SPSC_QUEUE.mask], SPSC_MASK
    mov     [rdi + SPSC_QUEUE.buffer], rsi
    ret

; ============================================================
; spsc_enqueue - เพิ่มข้อมูลลงใน queue (เรียกโดย Producer)
; Input:  rdi = pointer ไปยัง queue
;         rsi = ค่าที่จะใส่ (64-bit)
; Output: rax = 0 สำเร็จ, 1 queue เต็ม
;
; Memory ordering:
;   - อ่าน tail ด้วย acquire (เพื่อเห็นการ dequeue ล่าสุด)
;   - เขียน data ก่อน update head
;   - update head ด้วย release (store-release = SFENCE ก่อน)
; ============================================================
spsc_enqueue:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12
    push    r13

    mov     rbx, rdi        ; queue
    mov     r12, rsi        ; value

    ; อ่าน head (producer's own index)
    mov     r13, [rbx + SPSC_QUEUE.head]    ; head

    ; คำนวณ next head
    lea     rax, [r13 + 1]
    and     rax, [rbx + SPSC_QUEUE.mask]    ; next = (head + 1) & mask

    ; ตรวจสอบว่า queue เต็มหรือเปล่า
    ; queue เต็มเมื่อ next == tail
    mov     rcx, [rbx + SPSC_QUEUE.tail]    ; อ่าน tail (acquire)
    ; ใน x86: load เป็น acquire โดย default
    cmp     rax, rcx
    je      .full

    ; เขียนข้อมูลลงใน buffer[head]
    mov     rcx, [rbx + SPSC_QUEUE.buffer]  ; buffer pointer
    mov     [rcx + r13*8], r12              ; buffer[head] = value

    ; Store-Release: รับประกัน data ถูกเขียนก่อน head update
    ; ใน x86: store เป็น release โดย default
    ; แต่ใส่ SFENCE เพื่อความชัดเจนในโค้ดที่ต้องการ portability
    sfence

    ; Update head
    lea     rax, [r13 + 1]
    and     rax, [rbx + SPSC_QUEUE.mask]
    mov     [rbx + SPSC_QUEUE.head], rax

    xor     eax, eax        ; return 0 (สำเร็จ)
    jmp     .done

.full:
    mov     eax, 1          ; return 1 (เต็ม)

.done:
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret

; ============================================================
; spsc_dequeue - ดึงข้อมูลออกจาก queue (เรียกโดย Consumer)
; Input:  rdi = pointer ไปยัง queue
;         rsi = pointer ที่จะเก็บค่าที่อ่านได้
; Output: rax = 0 สำเร็จ, 1 queue ว่าง
; ============================================================
spsc_dequeue:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12

    mov     rbx, rdi        ; queue
    mov     r12, rsi        ; output pointer

    ; อ่าน tail (consumer's own index)
    mov     rax, [rbx + SPSC_QUEUE.tail]    ; tail

    ; ตรวจสอบว่า queue ว่างหรือเปล่า
    ; queue ว่างเมื่อ tail == head
    mov     rcx, [rbx + SPSC_QUEUE.head]    ; อ่าน head (acquire)
    cmp     rax, rcx
    je      .empty

    ; อ่านข้อมูลจาก buffer[tail]
    mov     rcx, [rbx + SPSC_QUEUE.buffer]
    mov     rdx, [rcx + rax*8]              ; value = buffer[tail]

    ; Load-Acquire: ใน x86 loads เป็น acquire โดย default
    ; แต่ LFENCE ชัดเจนกว่า
    lfence

    ; Update tail
    lea     rax, [rax + 1]
    and     rax, [rbx + SPSC_QUEUE.mask]
    mov     [rbx + SPSC_QUEUE.tail], rax

    ; เก็บค่าลงใน output
    mov     [r12], rdx

    xor     eax, eax        ; return 0 (สำเร็จ)
    jmp     .done

.empty:
    mov     eax, 1          ; return 1 (ว่าง)

.done:
    pop     r12
    pop     rbx
    pop     rbp
    ret
```

---

## 10. โปรแกรม: Parallel Sum Array {#parallel-sum}

### โปรแกรม parallel_sum.asm

```nasm
; ============================================================
; parallel_sum.asm - คำนวณผลรวม array ด้วย multiple threads
;
; Algorithm:
;   1. แบ่ง array ออกเป็น N chunks (N = จำนวน threads)
;   2. แต่ละ thread คำนวณผลรวมของ chunk ตัวเอง
;   3. Main thread รอทุก thread เสร็จ (ใช้ futex)
;   4. Main thread รวมผลรวมย่อยทั้งหมด
;
; nasm -f elf64 parallel_sum.asm -o parallel_sum.o
; ld parallel_sum.o -o parallel_sum
; ============================================================

section .data
    ; ข้อมูลทดสอบ: array ขนาด 1000 ตัว
    align 64
    test_array:
        %assign i 0
        %rep 1000
            dq i+1          ; ค่า 1, 2, 3, ..., 1000
            %assign i i+1
        %endrep
    ARRAY_SIZE  equ 1000

    ; จำนวน threads
    NUM_THREADS equ 4

    ; ข้อความ
    result_msg  db "ผลรวม: ", 0
    newline     db 10, 0

section .bss
    ; ผลรวมย่อยของแต่ละ thread
    partial_sums    resq NUM_THREADS

    ; Thread stacks
    thread_stacks   resq NUM_THREADS    ; เก็บ base address ของ stack

    ; Thread arguments
    ; แต่ละ thread ต้องการ: start_idx, count, result_ptr
    struc THREAD_ARG
        .start      resq 1  ; index เริ่มต้น
        .count      resq 1  ; จำนวน elements
        .array      resq 1  ; pointer ไปยัง array
        .result     resq 1  ; pointer ไปยัง output
        .done_flag  resd 1  ; 0=กำลังทำ, 1=เสร็จ
        .padding    resd 1
        .size       equ $ - THREAD_ARG
    endstruc
    thread_args     resb (THREAD_ARG.size * NUM_THREADS)

    ; Completion counter (เพื่อรอ thread ทั้งหมด)
    threads_done    resd 1

section .text
global _start

; ============================================================
; sum_thread - ฟังก์ชันที่แต่ละ thread รัน
; Thread argument อยู่ใน stack ของ thread (pass via stack magic)
; แต่ในที่นี้ใช้ global thread_args array แทน
; rdi = pointer ไปยัง THREAD_ARG struct
; ============================================================
sum_thread:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12
    push    r13
    push    r14

    ; rdi ยังไม่ได้รับค่า เพราะ clone ไม่ส่ง argument
    ; ใช้ r15 ที่ save ไว้ก่อน clone (ดูตอน setup)
    ; ในโปรแกรมนี้ใช้ thread index จาก TLS แทน

    ; ดึง thread argument จาก r15 (เซ็ตก่อน clone)
    mov     rbx, r15        ; r15 = pointer ไปยัง THREAD_ARG

    ; โหลด arguments
    mov     r12, [rbx + THREAD_ARG.start]   ; start index
    mov     r13, [rbx + THREAD_ARG.count]   ; count
    mov     r14, [rbx + THREAD_ARG.array]   ; array pointer

    ; คำนวณผลรวม
    xor     rax, rax        ; sum = 0
    lea     r14, [r14 + r12*8]  ; ptr = array + start * 8

.loop:
    test    r13, r13
    jz      .done

    add     rax, [r14]      ; sum += *ptr
    add     r14, 8          ; ptr++
    dec     r13             ; count--
    jmp     .loop

.done:
    ; เก็บผลลัพธ์
    mov     r14, [rbx + THREAD_ARG.result]
    mov     [r14], rax

    ; Set done flag
    mov     dword [rbx + THREAD_ARG.done_flag], 1

    ; Wake main thread ที่รออยู่
    mov     rax, 202                ; sys_futex
    lea     rdi, [rbx + THREAD_ARG.done_flag]
    mov     rsi, 1                  ; FUTEX_WAKE
    mov     rdx, 1                  ; wake 1
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    ; Exit thread
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    mov     rax, 60
    xor     rdi, rdi
    syscall

; ============================================================
; print_number - พิมพ์ตัวเลข 64-bit
; Input: rdi = ตัวเลข
; ============================================================
print_number:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32

    ; แปลงเลขเป็น string
    lea     rsi, [rsp + 30]
    mov     byte [rsi+1], 10    ; newline
    mov     rcx, 10

    mov     rax, rdi
    test    rax, rax
    jnz     .convert

    ; กรณี 0
    mov     byte [rsi], '0'
    dec     rsi
    jmp     .print

.convert:
    xor     rdx, rdx
    div     rcx                 ; rax = quotient, rdx = remainder
    add     dl, '0'
    mov     [rsi], dl
    dec     rsi
    test    rax, rax
    jnz     .convert

.print:
    inc     rsi
    lea     rdx, [rsp + 32]
    sub     rdx, rsi
    inc     rdx                 ; รวม newline

    mov     rax, 1
    mov     rdi, 1
    syscall

    leave
    ret

; ============================================================
; _start - Main program
; ============================================================
_start:
    ; คำนวณขนาด chunk สำหรับแต่ละ thread
    ; elements_per_thread = ARRAY_SIZE / NUM_THREADS
    mov     rax, ARRAY_SIZE
    xor     rdx, rdx
    mov     rcx, NUM_THREADS
    div     rcx                 ; rax = elements per thread (250)
    mov     r11, rax            ; r11 = chunk size

    ; สร้าง thread arguments และ threads
    xor     r12, r12            ; thread index = 0

.create_threads:
    cmp     r12, NUM_THREADS
    jge     .wait_threads

    ; คำนวณ offset ของ THREAD_ARG
    mov     rax, THREAD_ARG.size
    imul    rax, r12
    lea     rbx, [rel thread_args]
    add     rbx, rax            ; rbx = &thread_args[i]

    ; กำหนด start index
    mov     rax, r11
    imul    rax, r12
    mov     [rbx + THREAD_ARG.start], rax

    ; กำหนด count (thread สุดท้ายรับส่วนที่เหลือ)
    mov     rax, r12
    inc     rax
    cmp     rax, NUM_THREADS
    jne     .normal_count

    ; Thread สุดท้าย: count = ARRAY_SIZE - start
    mov     rax, ARRAY_SIZE
    sub     rax, [rbx + THREAD_ARG.start]
    mov     [rbx + THREAD_ARG.count], rax
    jmp     .set_array

.normal_count:
    mov     [rbx + THREAD_ARG.count], r11

.set_array:
    lea     rax, [rel test_array]
    mov     [rbx + THREAD_ARG.array], rax

    ; กำหนด result pointer
    lea     rax, [rel partial_sums]
    mov     rcx, r12
    lea     rax, [rax + rcx*8]
    mov     [rbx + THREAD_ARG.result], rax

    ; Clear done flag
    mov     dword [rbx + THREAD_ARG.done_flag], 0

    ; จัดสรร stack สำหรับ thread
    mov     rax, 9          ; sys_mmap
    xor     rdi, rdi
    mov     rsi, 0x10000    ; 64KB stack
    mov     rdx, 3
    mov     r10, 0x20022
    mov     r8, -1
    xor     r9, r9
    syscall

    ; เก็บ stack base
    lea     rcx, [rel thread_stacks]
    mov     [rcx + r12*8], rax

    ; คำนวณ top of stack
    add     rax, 0x10000
    and     rax, -16

    ; Save r15 = argument pointer (จะถูก inherit ใน child)
    mov     r15, rbx

    ; Create thread
    mov     rax, 56                 ; sys_clone
    mov     rdi, 0x00010F00        ; CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD
    mov     rsi, rax                ; stack top (ตั้งค่าข้างบน แต่ rax ถูก override)
    ; ต้องตั้งค่า rsi ให้เป็น stack top ก่อนเรียก syscall

    ; === แก้ไข: ตั้งค่า rsi ให้ถูกต้อง ===
    ; เอา stack top จากที่คำนวณ
    mov     rsi, [rcx + r12*8]
    add     rsi, 0x10000
    and     rsi, -16

    xor     rdx, rdx               ; parent_tid = NULL
    xor     r10, r10               ; child_tid = NULL
    xor     r8, r8                 ; tls = 0
    syscall

    test    rax, rax
    jz      sum_thread             ; child thread ไป sum_thread

    ; Parent: ไปสร้าง thread ถัดไป
    inc     r12
    jmp     .create_threads

.wait_threads:
    ; รอ thread ทั้งหมด (ตรวจ done flags)
    xor     r12, r12

.check_loop:
    cmp     r12, NUM_THREADS
    jge     .all_done

    ; ดึง done flag ของ thread i
    mov     rax, THREAD_ARG.size
    imul    rax, r12
    lea     rbx, [rel thread_args]
    add     rbx, rax

.wait_one:
    mov     eax, [rbx + THREAD_ARG.done_flag]
    test    eax, eax
    jnz     .thread_done

    ; ยังไม่เสร็จ: futex wait
    mov     rax, 202
    lea     rdi, [rbx + THREAD_ARG.done_flag]
    mov     rsi, 128        ; FUTEX_WAIT_PRIVATE (0 | 128)
    xor     rdx, rdx        ; val = 0 (รอเมื่อ flag == 0)
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall
    jmp     .wait_one

.thread_done:
    inc     r12
    jmp     .check_loop

.all_done:
    ; รวมผลรวมย่อยทั้งหมด
    xor     rbx, rbx        ; total = 0
    xor     r12, r12

.sum_loop:
    cmp     r12, NUM_THREADS
    jge     .print_result

    lea     rax, [rel partial_sums]
    add     rbx, [rax + r12*8]
    inc     r12
    jmp     .sum_loop

.print_result:
    ; พิมพ์ "ผลรวม: "
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel result_msg]
    mov     rdx, 8
    syscall

    ; พิมพ์ผลรวม
    mov     rdi, rbx
    call    print_number

    ; Exit
    mov     rax, 60
    xor     rdi, rdi
    syscall
```

---

## 11. โปรแกรม: Thread Pool {#thread-pool}

```nasm
; ============================================================
; thread_pool.asm - Thread Pool Implementation ใน Assembly
;
; Architecture:
;   - Fixed number of worker threads
;   - Task queue (SPSC per worker หรือ shared MPMC queue)
;   - Worker threads รอ task แล้ว execute
;   - Main thread submit tasks และรอผล
;
; โปรแกรมนี้ใช้ simplified version: shared task queue + mutex
; ============================================================

section .data
    ; Pool configuration
    POOL_SIZE           equ 4       ; จำนวน worker threads
    TASK_QUEUE_SIZE     equ 64      ; ขนาด task queue

    pool_created_msg    db "Thread pool สร้างสำเร็จ", 10, 0
    pool_created_len    equ $ - pool_created_msg - 1
    task_done_msg       db "Task เสร็จสิ้น", 10, 0
    task_done_len       equ $ - task_done_msg - 1
    pool_shutdown_msg   db "Pool กำลัง shutdown...", 10, 0
    pool_shutdown_len   equ $ - pool_shutdown_msg - 1

section .bss
    ; Task structure
    struc TASK
        .func       resq 1  ; function pointer (ฟังก์ชันที่จะรัน)
        .arg        resq 1  ; argument
        .result     resq 1  ; ผลลัพธ์
        .done       resd 1  ; flag บอกว่าเสร็จ
        .pad        resd 1
        .size       equ $ - TASK
    endstruc

    ; Task queue (ring buffer)
    task_queue      resb (TASK.size * TASK_QUEUE_SIZE)
    queue_head      resq 1  ; write index
    queue_tail      resq 1  ; read index
    queue_mutex     resd 1  ; mutex สำหรับ queue
    queue_size      resd 1  ; จำนวน tasks ใน queue
    queue_not_empty resd 1  ; futex สำหรับ signal worker
    queue_not_full  resd 1  ; futex สำหรับ signal submitter

    ; Worker states
    worker_running  resd 1  ; 1 = ทำงาน, 0 = shutdown
    worker_tids     resd POOL_SIZE
    worker_stacks   resq POOL_SIZE

section .text
global _start

; ============================================================
; mutex_lock_simple - Simple mutex lock ด้วย futex
; Input: rdi = mutex address (uint32_t*)
; ============================================================
mutex_lock_simple:
    push    rbx
    mov     rbx, rdi

.try_lock:
    xor     eax, eax            ; expected = 0 (UNLOCKED)
    mov     ecx, 1              ; new = 1 (LOCKED)
    lock cmpxchg dword [rbx], ecx
    jz      .got_lock

    ; ถ้า lock ครอง: spin รอสักครู่แล้วลองใหม่
    pause
    jmp     .try_lock

.got_lock:
    pop     rbx
    ret

; ============================================================
; mutex_unlock_simple
; Input: rdi = mutex address
; ============================================================
mutex_unlock_simple:
    mov     dword [rdi], 0
    ret

; ============================================================
; task_queue_push - เพิ่ม task ลงใน queue
; Input:  rdi = function pointer
;         rsi = argument
; Output: rax = pointer ไปยัง TASK (เพื่อตรวจ done flag)
; ============================================================
task_queue_push:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12
    push    r13

    mov     r12, rdi    ; function
    mov     r13, rsi    ; argument

.wait_not_full:
    ; Lock queue mutex
    lea     rdi, [rel queue_mutex]
    call    mutex_lock_simple

    ; ตรวจสอบว่า queue เต็มไหม
    mov     eax, [rel queue_size]
    cmp     eax, TASK_QUEUE_SIZE
    jl      .can_push

    ; Queue เต็ม: unlock แล้วรอ
    lea     rdi, [rel queue_mutex]
    call    mutex_unlock_simple

    ; Spin wait (หรือใช้ futex_wait จะดีกว่า)
    pause
    jmp     .wait_not_full

.can_push:
    ; คำนวณตำแหน่งใน queue
    mov     rax, [rel queue_head]
    mov     rcx, TASK_QUEUE_SIZE
    xor     rdx, rdx
    div     rcx
    ; rdx = head % TASK_QUEUE_SIZE

    ; คำนวณ pointer ไปยัง task slot
    mov     rax, TASK.size
    imul    rax, rdx
    lea     rbx, [rel task_queue]
    add     rbx, rax            ; rbx = &task_queue[head % size]

    ; กรอกข้อมูล task
    mov     [rbx + TASK.func], r12
    mov     [rbx + TASK.arg], r13
    mov     qword [rbx + TASK.result], 0
    mov     dword [rbx + TASK.done], 0

    ; Update head
    inc     qword [rel queue_head]
    inc     dword [rel queue_size]

    ; Unlock
    lea     rdi, [rel queue_mutex]
    call    mutex_unlock_simple

    ; Signal worker thread ว่ามี task แล้ว
    ; wake futex
    mov     rax, 202
    lea     rdi, [rel queue_not_empty]
    mov     rsi, 1              ; FUTEX_WAKE_PRIVATE
    mov     rdx, 1              ; wake 1 thread
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    mov     rax, rbx            ; คืน task pointer

    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret

; ============================================================
; task_queue_pop - ดึง task ออกจาก queue (เรียกโดย worker)
; Output: rax = pointer ไปยัง TASK (หรือ NULL ถ้า queue ว่าง)
; ============================================================
task_queue_pop:
    push    rbp
    mov     rbp, rsp
    push    rbx

    ; Lock queue mutex
    lea     rdi, [rel queue_mutex]
    call    mutex_lock_simple

    ; ตรวจสอบว่า queue ว่างไหม
    mov     eax, [rel queue_size]
    test    eax, eax
    jz      .empty

    ; คำนวณตำแหน่ง tail
    mov     rax, [rel queue_tail]
    mov     rcx, TASK_QUEUE_SIZE
    xor     rdx, rdx
    div     rcx
    ; rdx = tail % TASK_QUEUE_SIZE

    mov     rax, TASK.size
    imul    rax, rdx
    lea     rbx, [rel task_queue]
    add     rbx, rax

    ; Update tail
    inc     qword [rel queue_tail]
    dec     dword [rel queue_size]

    ; Unlock
    lea     rdi, [rel queue_mutex]
    call    mutex_unlock_simple

    mov     rax, rbx            ; คืน task pointer
    jmp     .done

.empty:
    ; Unlock
    lea     rdi, [rel queue_mutex]
    call    mutex_unlock_simple

    xor     rax, rax            ; คืน NULL

.done:
    pop     rbx
    pop     rbp
    ret

; ============================================================
; worker_thread - ฟังก์ชันของ worker thread
; วน loop รับ tasks มา execute จนกว่า worker_running = 0
; ============================================================
worker_thread:
    push    rbp
    mov     rbp, rsp
    push    rbx

.main_loop:
    ; ตรวจสอบว่ายังควรทำงานอยู่ไหม
    mov     eax, [rel worker_running]
    test    eax, eax
    jz      .shutdown

    ; พยายาม pop task
    call    task_queue_pop
    test    rax, rax
    jz      .wait_for_task

    ; มี task! Execute มัน
    mov     rbx, rax            ; เก็บ task pointer

    ; เรียก task function
    mov     rdi, [rbx + TASK.arg]
    call    qword [rbx + TASK.func]

    ; เก็บผลลัพธ์
    mov     [rbx + TASK.result], rax

    ; Mark task as done
    mov     dword [rbx + TASK.done], 1

    ; Wake ใครก็ตามที่รอ task นี้
    mov     rax, 202
    lea     rdi, [rbx + TASK.done]
    mov     rsi, 1
    mov     rdx, 0x7FFFFFFF     ; wake all
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    jmp     .main_loop

.wait_for_task:
    ; Queue ว่าง: รอ signal จาก producer
    ; futex wait on queue_not_empty
    mov     rax, 202
    lea     rdi, [rel queue_not_empty]
    mov     rsi, 128            ; FUTEX_WAIT_PRIVATE
    xor     rdx, rdx            ; val = 0 (รอเมื่อ == 0)
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    jmp     .main_loop

.shutdown:
    pop     rbx
    pop     rbp
    mov     rax, 60
    xor     rdi, rdi
    syscall

; ============================================================
; example_task - ตัวอย่าง task function
; Input:  rdi = value
; Output: rax = value * value (square)
; ============================================================
example_task:
    mov     rax, rdi
    imul    rax, rax
    ret

; ============================================================
; _start - Main program
; ============================================================
_start:
    ; Initialize
    mov     dword [rel worker_running], 1
    mov     qword [rel queue_head], 0
    mov     qword [rel queue_tail], 0
    mov     dword [rel queue_mutex], 0
    mov     dword [rel queue_size], 0

    ; เขียน message
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel pool_created_msg]
    mov     rdx, pool_created_len
    syscall

    ; สร้าง worker threads
    xor     r12, r12

.create_workers:
    cmp     r12, POOL_SIZE
    jge     .submit_tasks

    ; จัดสรร stack
    mov     rax, 9
    xor     rdi, rdi
    mov     rsi, 0x10000
    mov     rdx, 3
    mov     r10, 0x20022
    mov     r8, -1
    xor     r9, r9
    syscall

    lea     rcx, [rel worker_stacks]
    mov     [rcx + r12*8], rax
    add     rax, 0x10000
    and     rax, -16

    ; สร้าง thread
    mov     rsi, rax
    mov     rax, 56
    mov     rdi, 0x00010F00
    xor     rdx, rdx
    xor     r10, r10
    xor     r8, r8
    syscall

    test    rax, rax
    jz      worker_thread       ; child → worker

    inc     r12
    jmp     .create_workers

.submit_tasks:
    ; ส่ง tasks ให้ worker threads
    xor     r12, r12

.submit_loop:
    cmp     r12, 8              ; ส่ง 8 tasks
    jge     .wait_results

    ; Submit task: คำนวณ square ของตัวเลข
    lea     rdi, [rel example_task]
    lea     rsi, [r12 + 1]
    call    task_queue_push

    ; rax = task pointer, เก็บไว้เพื่อรอผล
    inc     r12
    jmp     .submit_loop

.wait_results:
    ; Shutdown pool
    mov     dword [rel worker_running], 0

    ; Wake all workers
    mov     rax, 202
    lea     rdi, [rel queue_not_empty]
    mov     rsi, 1
    mov     rdx, POOL_SIZE
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    ; เขียน shutdown message
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel pool_shutdown_msg]
    mov     rdx, pool_shutdown_len
    syscall

    ; Exit
    mov     rax, 60
    xor     rdi, rdi
    syscall
```

---

## 12. โปรแกรม: Mutex-Protected Counter {#mutex-counter}

```nasm
; ============================================================
; mutex_counter.asm - Counter ที่ protected ด้วย mutex
;
; โปรแกรมสร้าง N threads แต่ละอัน increment counter M ครั้ง
; ผลลัพธ์ควรเป็น N * M เสมอ (correctness test)
;
; Build:
; nasm -f elf64 mutex_counter.asm -o mutex_counter.o
; ld mutex_counter.o -o mutex_counter
; ============================================================

section .data
    NUM_THREADS     equ 8
    INCREMENTS_EACH equ 10000
    EXPECTED_TOTAL  equ (NUM_THREADS * INCREMENTS_EACH)

    start_msg   db "เริ่ม increment counter ด้วย ", 0
    start_msg2  db " threads", 10, 0
    result_msg  db "Counter = ", 0
    expect_msg  db "Expected = ", 0
    pass_msg    db "TEST PASSED!", 10, 0
    fail_msg    db "TEST FAILED!", 10, 0

section .bss
    ; Shared counter (ที่ต้องการ protect)
    counter         resq 1

    ; Mutex สำหรับ protect counter
    counter_mutex   resd 1

    ; Thread management
    thread_stacks   resq NUM_THREADS
    threads_done    resd 1          ; futex: จำนวน threads ที่เสร็จแล้ว
    threads_created resd 1          ; จำนวน threads ที่สร้าง

section .text
global _start

; ============================================================
; mutex_lock_full - Full mutex implementation
; ============================================================
mutex_lock_full:
    push    rbx
    mov     rbx, rdi

.fast_path:
    xor     eax, eax
    mov     ecx, 1
    lock cmpxchg dword [rbx], ecx
    jz      .locked

.slow_path:
    mov     eax, 1
    xchg    dword [rbx], eax
    test    eax, eax
    jz      .locked

    mov     rax, 202
    mov     rdi, rbx
    mov     rsi, 128            ; FUTEX_WAIT_PRIVATE
    mov     rdx, 1              ; val = 1
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    mov     rdi, rbx
    jmp     .slow_path

.locked:
    pop     rbx
    ret

; ============================================================
; mutex_unlock_full
; ============================================================
mutex_unlock_full:
    lock xchg dword [rdi], eax
    ; ถ้าเดิมเป็น 0: ไม่มีใครรอ ไม่ต้องทำอะไร
    test    eax, eax
    jz      .done

    ; มีคนรอ: wake
    mov     rax, 202
    mov     rsi, 129            ; FUTEX_WAKE_PRIVATE
    mov     rdx, 1
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

.done:
    ret

; ============================================================
; increment_thread - Thread ที่ increment counter
; r15 = thread index (set ก่อน clone)
; ============================================================
increment_thread:
    push    rbp
    mov     rbp, rsp

    ; วน increment INCREMENTS_EACH ครั้ง
    mov     ecx, INCREMENTS_EACH

.loop:
    ; Lock mutex
    lea     rdi, [rel counter_mutex]
    call    mutex_lock_full

    ; Increment counter (ต้องทำใน critical section)
    inc     qword [rel counter]

    ; Unlock mutex
    lea     rdi, [rel counter_mutex]
    call    mutex_unlock_full

    dec     ecx
    jnz     .loop

    ; Atomically increment threads_done
    lock inc dword [rel threads_done]

    ; Wake main thread
    mov     rax, 202
    lea     rdi, [rel threads_done]
    mov     rsi, 129
    mov     rdx, 1
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    ; Exit thread
    pop     rbp
    mov     rax, 60
    xor     rdi, rdi
    syscall

; ============================================================
; print_decimal - พิมพ์ตัวเลข decimal
; Input: rdi = number
; ============================================================
print_decimal:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 64

    lea     rsi, [rsp + 62]
    mov     byte [rsi+1], 10    ; newline
    mov     r8, 10

    mov     rax, rdi
.cvt:
    xor     rdx, rdx
    div     r8
    add     dl, '0'
    mov     [rsi], dl
    dec     rsi
    test    rax, rax
    jnz     .cvt

    inc     rsi
    lea     rdx, [rsp + 64]
    sub     rdx, rsi

    mov     rax, 1
    mov     rdi, 1
    syscall

    leave
    ret

; ============================================================
; print_str - พิมพ์ null-terminated string
; Input: rdi = string pointer
; ============================================================
print_str:
    push    rdi
    xor     rcx, rcx

.len:
    cmp     byte [rdi + rcx], 0
    je      .done_len
    inc     rcx
    jmp     .len

.done_len:
    mov     rdx, rcx
    mov     rsi, rdi
    mov     rax, 1
    mov     rdi, 1
    pop     rdi
    push    rdi
    syscall
    pop     rdi
    ret

; ============================================================
; _start
; ============================================================
_start:
    ; Initialize
    mov     qword [rel counter], 0
    mov     dword [rel counter_mutex], 0
    mov     dword [rel threads_done], 0

    ; Print start message
    lea     rdi, [rel start_msg]
    call    print_str

    mov     rdi, NUM_THREADS
    call    print_decimal

    lea     rdi, [rel start_msg2]
    call    print_str

    ; สร้าง threads
    xor     r12, r12

.create:
    cmp     r12, NUM_THREADS
    jge     .wait_all

    ; จัดสรร stack
    mov     rax, 9
    xor     rdi, rdi
    mov     rsi, 0x10000
    mov     rdx, 3
    mov     r10, 0x20022
    mov     r8, -1
    xor     r9, r9
    syscall

    lea     rcx, [rel thread_stacks]
    mov     [rcx + r12*8], rax
    add     rax, 0x10000
    and     rax, -16
    mov     r15, r12            ; thread index

    mov     rsi, rax
    mov     rax, 56
    mov     rdi, 0x00010F00
    xor     rdx, rdx
    xor     r10, r10
    xor     r8, r8
    syscall

    test    rax, rax
    jz      increment_thread

    inc     r12
    jmp     .create

.wait_all:
    ; รอ threads ทั้งหมดเสร็จ
.wait_loop:
    mov     eax, [rel threads_done]
    cmp     eax, NUM_THREADS
    je      .all_done

    mov     rax, 202
    lea     rdi, [rel threads_done]
    mov     rsi, 128
    mov     rdx, dword [rel threads_done]
    xor     r10, r10
    xor     r8, r8
    xor     r9, r9
    syscall

    jmp     .wait_loop

.all_done:
    ; แสดงผล
    lea     rdi, [rel result_msg]
    call    print_str

    mov     rdi, [rel counter]
    call    print_decimal

    lea     rdi, [rel expect_msg]
    call    print_str

    mov     rdi, EXPECTED_TOTAL
    call    print_decimal

    ; ตรวจสอบ
    mov     rax, [rel counter]
    cmp     rax, EXPECTED_TOTAL
    jne     .fail

    lea     rdi, [rel pass_msg]
    call    print_str
    jmp     .exit

.fail:
    lea     rdi, [rel fail_msg]
    call    print_str

.exit:
    mov     rax, 60
    xor     rdi, rdi
    syscall
```

---

## 13. โปรแกรม: Lock-Free Stack (Treiber Stack) {#treiber-stack}

### Treiber Stack คืออะไร?

**Treiber Stack** เป็น lock-free concurrent stack ที่ใช้ CAS เพื่อ:
- Push: สร้าง node ใหม่ → CAS head จาก old_head เป็น new_node
- Pop: อ่าน head → CAS head จาก head เป็น head->next

### ABA Problem

ปัญหาสำคัญของ CAS-based structures:
```
Thread 1 อ่าน head = A
Thread 2 pop A, push B, push A กลับ (head = A อีกครั้ง)
Thread 1 CAS ประสบความสำเร็จ แต่ state เปลี่ยนไปแล้ว!
```

แก้ด้วย: Version counter (tagged pointer) ใช้ upper bits ของ pointer

```nasm
; ============================================================
; treiber_stack.asm - Lock-Free Stack ด้วย CMPXCHG16B
;
; Treiber Stack พร้อม ABA protection ด้วย version counter
;
; Node structure:
;   +----------+----------+
;   |   next   |  value   |
;   +----------+----------+
;   (16 bytes, 16-byte aligned)
;
; Head structure (16 bytes, atomic update ด้วย CMPXCHG16B):
;   +------------------+------------------+
;   |  ptr (8 bytes)   | version (8 bytes)|
;   +------------------+------------------+
;
; nasm -f elf64 treiber_stack.asm -o treiber_stack.o
; ld treiber_stack.o -o treiber_stack
; ============================================================

section .data
    push_msg    db "Push สำเร็จ: ", 0
    pop_msg     db "Pop ได้ค่า: ", 0
    empty_msg   db "Stack ว่าง!", 10, 0
    test_msg    db "ทดสอบ Treiber Stack:", 10, 0

section .bss
    ; Stack head (16 bytes, 16-byte aligned)
    align 16
    stack_head_ptr      resq 1  ; pointer ไปยัง top node
    stack_head_ver      resq 1  ; version counter (ป้องกัน ABA)

    ; Node pool (pre-allocated เพื่อหลีกเลี่ยง malloc)
    NODE_POOL_SIZE  equ 1024
    node_pool_next  resq 1  ; next free node index

    align 16
    node_pool:
    ; แต่ละ node: next(8) + value(8) = 16 bytes
    ; next = pointer ไปยัง node ถัดไป
    ; value = ข้อมูล
    resb (16 * NODE_POOL_SIZE)

section .text
global _start

; ============================================================
; alloc_node - จัดสรร node จาก pool
; Output: rax = pointer ไปยัง node (16-byte aligned)
;         หรือ NULL ถ้า pool เต็ม
; ============================================================
alloc_node:
    ; Atomically increment pool index
    mov     rax, 1
    lock xadd qword [rel node_pool_next], rax
    ; rax = old index (index ที่เราได้รับ)

    cmp     rax, NODE_POOL_SIZE
    jge     .full

    ; คำนวณ pointer
    imul    rax, 16
    lea     rcx, [rel node_pool]
    add     rax, rcx

    ; Clear node
    mov     qword [rax], 0      ; next = NULL
    mov     qword [rax+8], 0    ; value = 0
    ret

.full:
    xor     rax, rax    ; คืน NULL
    ret

; ============================================================
; treiber_push - Push ค่าลงใน stack
; Input:  rdi = ค่าที่จะ push
; Output: rax = 0 สำเร็จ, -1 ล้มเหลว (pool เต็ม)
; ============================================================
treiber_push:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12

    mov     r12, rdi        ; เก็บ value

    ; จัดสรร node ใหม่
    call    alloc_node
    test    rax, rax
    jz      .fail

    mov     rbx, rax        ; rbx = new node

    ; กำหนดค่า
    mov     [rbx + 8], r12  ; node->value = value

.retry:
    ; อ่าน current head (ทั้ง ptr และ version)
    ; ต้องอ่านแบบ atomic! ใช้ CMPXCHG16B เพื่อ read
    ; หรืออ่านสองครั้ง (ไม่ atomic แต่ใช้ได้ใน simple case)
    mov     rax, [rel stack_head_ptr]   ; old ptr
    mov     rdx, [rel stack_head_ver]   ; old version

    ; new node->next = old head ptr
    mov     [rbx], rax      ; node->next = old_ptr

    ; CAS: ถ้า head == (old_ptr, old_ver) → head = (new_node, old_ver+1)
    ; rdx:rax = expected (version:ptr)
    ; rcx:rbx = new value (new_ver:new_node)
    mov     rcx, rdx
    inc     rcx             ; new_version = old_version + 1

    ; CMPXCHG16B: compare rdx:rax with [rdi], swap with rcx:rbx if equal
    ; ต้องการ 16-byte alignment!
    lea     rdi, [rel stack_head_ptr]
    lock cmpxchg16b [rdi]
    jnz     .retry          ; ZF=0: ล้มเหลว (มีคนอื่นเปลี่ยน) → ลองใหม่

    xor     eax, eax        ; return 0 (สำเร็จ)
    jmp     .done

.fail:
    mov     eax, -1

.done:
    pop     r12
    pop     rbx
    pop     rbp
    ret

; ============================================================
; treiber_pop - Pop ค่าออกจาก stack
; Output: rax = ค่าที่ pop ได้
;         CF = 1 ถ้า stack ว่าง
; ============================================================
treiber_pop:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12

.retry:
    ; อ่าน current head
    mov     rax, [rel stack_head_ptr]   ; old_ptr
    mov     rdx, [rel stack_head_ver]   ; old_version

    ; ถ้า stack ว่าง
    test    rax, rax
    jz      .empty

    ; อ่าน next node (rax = head node)
    mov     rbx, [rax]      ; old_next = head->next
    mov     r12, [rax + 8]  ; value = head->value

    ; CAS: ถ้า head == (old_ptr, old_ver) → head = (old_next, old_ver+1)
    mov     rcx, rdx
    inc     rcx             ; new_version

    lea     rdi, [rel stack_head_ptr]
    lock cmpxchg16b [rdi]
    jnz     .retry          ; ล้มเหลว → ลองใหม่

    ; สำเร็จ! คืนค่า
    mov     rax, r12
    clc                     ; CF = 0 (success)
    jmp     .done

.empty:
    stc                     ; CF = 1 (empty)
    xor     rax, rax

.done:
    pop     r12
    pop     rbx
    pop     rbp
    ret

; ============================================================
; print_number_nl - พิมพ์ตัวเลขตามด้วย newline
; Input: rdi = number
; ============================================================
print_number_nl:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 40

    mov     r8, 10
    lea     rsi, [rsp + 38]
    mov     byte [rsi+1], 10

    mov     rax, rdi
.cvt:
    xor     rdx, rdx
    div     r8
    add     dl, '0'
    mov     [rsi], dl
    dec     rsi
    test    rax, rax
    jnz     .cvt

    inc     rsi
    lea     rdx, [rsp + 40]
    sub     rdx, rsi

    mov     rax, 1
    mov     rdi, 1
    syscall

    leave
    ret

; ============================================================
; print_cstr - พิมพ์ null-terminated string
; Input: rdi = string
; ============================================================
print_cstr:
    push    rdi
    xor     rcx, rcx
.scan:
    cmp     byte [rdi+rcx], 0
    je      .pr
    inc     rcx
    jmp     .scan
.pr:
    mov     rdx, rcx
    mov     rsi, [rsp]
    mov     rax, 1
    mov     rdi, 1
    syscall
    pop     rdi
    ret

; ============================================================
; _start - ทดสอบ Treiber Stack
; ============================================================
_start:
    ; Initialize
    mov     qword [rel stack_head_ptr], 0
    mov     qword [rel stack_head_ver], 0
    mov     qword [rel node_pool_next], 0

    ; Print test header
    lea     rdi, [rel test_msg]
    call    print_cstr

    ; Push 5 values
    mov     rdi, 10
    call    treiber_push
    lea     rdi, [rel push_msg]
    call    print_cstr
    mov     rdi, 10
    call    print_number_nl

    mov     rdi, 20
    call    treiber_push
    lea     rdi, [rel push_msg]
    call    print_cstr
    mov     rdi, 20
    call    print_number_nl

    mov     rdi, 30
    call    treiber_push
    lea     rdi, [rel push_msg]
    call    print_cstr
    mov     rdi, 30
    call    print_number_nl

    mov     rdi, 42
    call    treiber_push
    lea     rdi, [rel push_msg]
    call    print_cstr
    mov     rdi, 42
    call    print_number_nl

    mov     rdi, 99
    call    treiber_push
    lea     rdi, [rel push_msg]
    call    print_cstr
    mov     rdi, 99
    call    print_number_nl

    ; Pop ทั้งหมด (ควรได้ 99, 42, 30, 20, 10 ตามลำดับ LIFO)
    .pop_loop:
    call    treiber_pop
    jc      .pop_done       ; CF = 1 = empty

    push    rax
    lea     rdi, [rel pop_msg]
    call    print_cstr
    pop     rdi
    call    print_number_nl
    jmp     .pop_loop

.pop_done:
    ; ลอง pop อีกครั้ง (ควรได้ empty)
    call    treiber_pop
    jnc     .unexpected
    lea     rdi, [rel empty_msg]
    call    print_cstr
    jmp     .exit

.unexpected:
    ; ไม่ควรเกิดขึ้น
    jmp     .exit

.exit:
    mov     rax, 60
    xor     rdi, rdi
    syscall
```

---

## สรุปและแนวทางปฏิบัติ (Summary & Best Practices)

### Do's ✓
1. **ใช้ PAUSE** ในทุก spin loop เสมอ
2. **ตั้งค่า stack guard page** เพื่อตรวจจับ stack overflow
3. **Separate cache lines** ระหว่าง producer และ consumer data (False Sharing)
4. **ใช้ version counter** กับ CAS-based structures เพื่อป้องกัน ABA
5. **MFENCE ก่อน setting ready flag** ใน producer-consumer pattern
6. **ใช้ FUTEX_PRIVATE_FLAG** เมื่อ futex ใช้เฉพาะภายใน process

### Don'ts ✗
1. **อย่า busy-wait โดยไม่มี PAUSE** - ทำให้ Hyper-Threading ทำงานแย่ลง
2. **อย่าใช้ volatile แทน atomic** - ใน Assembly ต้องใช้ LOCK prefix
3. **อย่าลืม MFENCE** เมื่อใช้ non-temporal stores (MOVNT*)
4. **อย่า share cache line** ระหว่าง read-mostly และ write-heavy data
5. **อย่าสร้าง thread โดยไม่ขนาด stack** - ต้องจัดสรรและ free ด้วยตัวเอง

### Atomic Operation Cheatsheet

```
ต้องการ          → ใช้
─────────────────────────────────────────────
Atomic set       → lock xchg [mem], reg
Test-and-set     → xchg [mem], reg  (xchg implicit lock)
Compare-and-swap → lock cmpxchg [mem], reg  (rax=expected)
CAS 128-bit      → lock cmpxchg16b [mem]  (rdx:rax=expected, rcx:rbx=new)
Fetch-and-add    → lock xadd [mem], reg  (reg=old value after)
Atomic inc/dec   → lock inc/dec [mem]
Atomic add/sub   → lock add/sub [mem], imm/reg
Atomic and/or/xor→ lock and/or/xor [mem], imm/reg
Bit test+set     → lock bts [mem], bit
Bit test+reset   → lock btr [mem], bit
Memory fence     → mfence (full) / sfence (store) / lfence (load)
Spin hint        → pause
```

### Syscall Numbers สำหรับ Threading (x86-64 Linux)

```
sys_clone       = 56
sys_futex       = 202
sys_gettid      = 186
sys_arch_prctl  = 158
sys_mmap        = 9
sys_munmap      = 11
sys_mprotect    = 10
sys_exit        = 60
sys_exit_group  = 231
sys_nanosleep   = 35
sys_sched_yield = 24
```

---

## แบบฝึกหัด (Exercises)

1. **Easy**: เขียนฟังก์ชัน `spinlock_lock` ที่ใช้ exponential backoff (ล่าช้านานขึ้นเมื่อ retry)
2. **Medium**: ปรับปรุง SPSC queue ให้ print stats (จำนวน enqueue/dequeue, collision counts)
3. **Medium**: เพิ่ม timeout ให้กับ `mutex_lock` โดยใช้ `futex_wait` พร้อม `timespec`
4. **Hard**: เขียน MPMC (Multi-Producer Multi-Consumer) lock-free queue ด้วย CMPXCHG
5. **Hard**: Implement Read-Write Lock (อนุญาต concurrent reads แต่ exclusive write)
6. **Expert**: เพิ่ม priority inheritance ให้กับ mutex เพื่อป้องกัน priority inversion

---

## References

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Volume 3A
- Linux kernel source: `kernel/futex/futex.c`
- Herlihy, M. & Shavit, N.: "The Art of Multiprocessor Programming"
- Paul E. McKenney: "Is Parallel Programming Hard, And If So, What Can You Do About It?"
- "Memory Barriers: a Hardware View for Software Hackers" - Paul E. McKenney
- Treiber, R.K.: "Systems Programming: Coping with Parallelism" (1986)
- glibc source: `nptl/` directory (pthread implementation)

---

*Part 057 - Multithreading ใน Assembly | Assembly Language Course*
*เนื้อหาครอบคลุม: clone syscall, TLS, futex, mutex, spinlock, atomic ops, memory ordering, lock-free structures*
