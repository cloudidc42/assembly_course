# Part 040: ARM System Calls บน Linux

## บทนำ (Introduction)

System Call คือกลไกที่โปรแกรม user-space ใช้ขอบริการจาก kernel ใน ARM Linux เราใช้ instruction พิเศษเพื่อเข้าสู่ kernel mode และระบุหมายเลข syscall ผ่าน register

```
ARM32:  R7 = syscall number, SWI #0
AArch64: X8 = syscall number, SVC #0
```

---

## 1. ARM32 System Calls

### 1.1 โครงสร้างพื้นฐาน ARM32 Syscall

```asm
@ ARM32 syscall convention
@ R7  = syscall number (ตัวเลขบอก kernel ว่าต้องการทำอะไร)
@ R0  = argument 1 / return value
@ R1  = argument 2
@ R2  = argument 3
@ R3  = argument 4
@ R4  = argument 5
@ R5  = argument 6
@ SWI #0 (หรือ SVC #0) = trigger software interrupt เข้า kernel

@ ตัวอย่าง: sys_write(1, "Hello\n", 6)
@ syscall number ของ write = 4 (ARM32 Linux)
mov r7, #4          @ syscall: write
mov r0, #1          @ fd = stdout
ldr r1, =msg        @ pointer ไปยัง string
mov r2, #6          @ จำนวน bytes
swi #0              @ เรียก kernel!
```

### 1.2 ARM32 Syscall Numbers ที่ใช้บ่อย

| Syscall | Number (ARM32) | Description |
|---------|---------------|-------------|
| exit    | 1             | ออกจากโปรแกรม |
| fork    | 2             | สร้าง child process |
| read    | 3             | อ่านจาก file descriptor |
| write   | 4             | เขียนไปยัง file descriptor |
| open    | 5             | เปิด file |
| close   | 6             | ปิด file |
| waitpid | 7             | รอ child process |
| execve  | 11            | รัน program ใหม่ |
| getpid  | 20            | ดึง process ID |
| mmap2   | 192           | map memory |
| socket  | 281           | สร้าง socket |
| bind    | 282           | bind socket |
| connect | 283           | connect socket |
| listen  | 284           | listen socket |
| accept  | 285           | accept connection |

### 1.3 Hello World ARM32

```asm
@ arm32_hello.s - Hello World แบบ ARM32
@ Compile: arm-linux-gnueabi-as arm32_hello.s -o arm32_hello.o
@           arm-linux-gnueabi-ld arm32_hello.o -o arm32_hello
@ Run:     qemu-arm ./arm32_hello

.section .data
    msg:    .ascii "Hello, ARM32 World!\n"   @ string ที่จะแสดง
    len:    .word  20                         @ ความยาว string

.section .text
.global _start

_start:
    @ === sys_write(stdout, msg, len) ===
    mov r7, #4              @ syscall number: write (4)
    mov r0, #1              @ file descriptor: stdout (1)
    ldr r1, =msg            @ pointer ไปยัง message
    ldr r2, =len            @ โหลด address ของ len
    ldr r2, [r2]            @ โหลด value ของ len
    swi #0                  @ เรียก kernel

    @ === sys_exit(0) ===
    mov r7, #1              @ syscall number: exit (1)
    mov r0, #0              @ exit code = 0 (success)
    swi #0                  @ ออกจากโปรแกรม
```

### 1.4 ARM32 Syscall ด้วย inline asm ใน C

```c
/* arm32_syscall_c.c - ทดสอบ syscall จาก C */
#include <stdio.h>

/* เรียก write syscall โดยตรงด้วย inline assembly */
static inline long arm32_write(int fd, const void *buf, size_t count) {
    long result;
    __asm__ volatile (
        "mov r7, #4\n"      /* syscall: write */
        "mov r0, %1\n"      /* fd */
        "mov r1, %2\n"      /* buf */
        "mov r2, %3\n"      /* count */
        "swi #0\n"          /* เรียก kernel */
        "mov %0, r0\n"      /* เก็บ return value */
        : "=r"(result)
        : "r"(fd), "r"(buf), "r"(count)
        : "r0", "r1", "r2", "r7"
    );
    return result;
}

int main() {
    const char *msg = "Direct syscall from C on ARM32!\n";
    arm32_write(1, msg, 32);
    return 0;
}
```

---

## 2. AArch64 System Calls

### 2.1 โครงสร้างพื้นฐาน AArch64 Syscall

```asm
// AArch64 syscall convention
// X8  = syscall number
// X0  = argument 1 / return value
// X1  = argument 2
// X2  = argument 3
// X3  = argument 4
// X4  = argument 5
// X5  = argument 6
// SVC #0 = trigger supervisor call เข้า kernel

// ตัวอย่าง: sys_write(1, "Hello\n", 6)
// syscall number ของ write = 64 (AArch64 Linux)
mov x8, #64         // syscall: write
mov x0, #1          // fd = stdout
adr x1, msg         // pointer ไปยัง string
mov x2, #6          // จำนวน bytes
svc #0              // เรียก kernel!
```

### 2.2 AArch64 Syscall Numbers ที่ใช้บ่อย

| Syscall  | Number (AArch64) | Description |
|----------|-----------------|-------------|
| io_setup | 0               | async I/O setup |
| read     | 63              | อ่านจาก fd |
| write    | 64              | เขียนไปยัง fd |
| openat   | 56              | เปิด file (relative) |
| close    | 57              | ปิด file |
| lseek    | 62              | เลื่อน file position |
| mmap     | 222             | map memory |
| mprotect | 226             | ป้องกัน memory |
| munmap   | 215             | unmap memory |
| exit     | 93              | ออกจาก thread |
| exit_group | 94            | ออกจาก process |
| waitid   | 95              | รอ child |
| execve   | 221             | รัน program |
| getpid   | 172             | ดึง PID |
| socket   | 198             | สร้าง socket |
| bind     | 200             | bind socket |
| connect  | 203             | connect |
| listen   | 201             | listen |
| accept   | 202             | accept |
| sendto   | 206             | ส่ง data |
| recvfrom | 207             | รับ data |

### 2.3 Hello World AArch64

```asm
// aarch64_hello.s - Hello World แบบ AArch64
// Compile: aarch64-linux-gnu-as aarch64_hello.s -o aarch64_hello.o
//          aarch64-linux-gnu-ld aarch64_hello.o -o aarch64_hello
// Run:     qemu-aarch64 ./aarch64_hello

.section .data
    msg:    .ascii "Hello, AArch64 World!\n"  // string ที่จะแสดง
    msg_len = . - msg                          // คำนวณ length อัตโนมัติ

.section .text
.global _start

_start:
    // === sys_write(stdout, msg, msg_len) ===
    mov x8, #64             // syscall number: write (64)
    mov x0, #1              // file descriptor: stdout (1)
    adr x1, msg             // address ของ message
    mov x2, #msg_len        // ความยาว message
    svc #0                  // เรียก kernel

    // === sys_exit_group(0) ===
    mov x8, #94             // syscall number: exit_group (94)
    mov x0, #0              // exit code = 0 (success)
    svc #0                  // ออกจากโปรแกรม
```

---

## 3. File I/O บน ARM32

### 3.1 Open/Read/Write/Close ARM32

```asm
@ arm32_fileio.s - File I/O แบบสมบูรณ์ด้วย ARM32
@ Compile: arm-linux-gnueabi-as arm32_fileio.s -o arm32_fileio.o
@           arm-linux-gnueabi-ld arm32_fileio.o -o arm32_fileio

@ === Constants ===
.equ SYS_EXIT,   1
.equ SYS_READ,   3
.equ SYS_WRITE,  4
.equ SYS_OPEN,   5
.equ SYS_CLOSE,  6

@ O_RDONLY, O_WRONLY, O_RDWR, O_CREAT flags
.equ O_RDONLY,   0
.equ O_WRONLY,   1
.equ O_RDWR,     2
.equ O_CREAT,    64      @ 0100 octal
.equ O_TRUNC,    512     @ 01000 octal

@ File permissions
.equ S_IRWXU,    448     @ 0700 octal
.equ S_IRUSR,    256     @ 0400 octal
.equ S_IWUSR,    128     @ 0200 octal

.section .data
    input_file:  .asciz "input.txt"       @ ชื่อ input file
    output_file: .asciz "output.txt"      @ ชื่อ output file
    err_msg:     .ascii "Error opening file!\n"
    err_len:     .word  20

.section .bss
    buffer:      .space 4096              @ buffer สำหรับอ่าน/เขียน
    fd_in:       .space 4                 @ file descriptor input
    fd_out:      .space 4                 @ file descriptor output

.section .text
.global _start

_start:
    @ ====================================
    @ เปิด input file (O_RDONLY)
    @ sys_open(pathname, flags, mode)
    @ ====================================
    mov r7, #SYS_OPEN
    ldr r0, =input_file     @ pathname
    mov r1, #O_RDONLY       @ flags: read only
    mov r2, #0              @ mode: ไม่สำคัญสำหรับ O_RDONLY
    swi #0

    @ ตรวจสอบ error (ถ้า r0 < 0 = error)
    cmp r0, #0
    blt open_error          @ ถ้า error ให้ไป error handler

    @ บันทึก fd ไว้
    ldr r3, =fd_in
    str r0, [r3]            @ fd_in = r0

    @ ====================================
    @ เปิด/สร้าง output file
    @ sys_open(pathname, O_WRONLY|O_CREAT|O_TRUNC, 0644)
    @ ====================================
    mov r7, #SYS_OPEN
    ldr r0, =output_file        @ pathname
    mov r1, #(O_WRONLY | O_CREAT | O_TRUNC)  @ flags
    mov r2, #0644               @ permissions: rw-r--r--
    swi #0

    cmp r0, #0
    blt open_error

    ldr r3, =fd_out
    str r0, [r3]

    @ ====================================
    @ วนลูปอ่าน + เขียน
    @ ====================================
read_write_loop:
    @ อ่านจาก input file
    mov r7, #SYS_READ
    ldr r3, =fd_in
    ldr r0, [r3]            @ fd_in
    ldr r1, =buffer         @ buffer address
    mov r2, #4096           @ อ่านสูงสุด 4096 bytes
    swi #0

    @ ถ้า r0 == 0: EOF
    cmp r0, #0
    beq copy_done

    @ ถ้า r0 < 0: error
    blt read_error

    @ บันทึกจำนวน bytes ที่อ่านได้
    mov r4, r0              @ r4 = bytes_read

    @ เขียนไปยัง output file
    mov r7, #SYS_WRITE
    ldr r3, =fd_out
    ldr r0, [r3]            @ fd_out
    ldr r1, =buffer         @ buffer address
    mov r2, r4              @ bytes to write
    swi #0

    @ วนลูปต่อ
    b read_write_loop

copy_done:
    @ ====================================
    @ ปิด files
    @ ====================================
    mov r7, #SYS_CLOSE
    ldr r3, =fd_in
    ldr r0, [r3]
    swi #0

    mov r7, #SYS_CLOSE
    ldr r3, =fd_out
    ldr r0, [r3]
    swi #0

    @ ====================================
    @ จบโปรแกรม
    @ ====================================
    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

open_error:
read_error:
    @ แสดง error message
    mov r7, #SYS_WRITE
    mov r0, #2              @ stderr
    ldr r1, =err_msg
    ldr r2, =err_len
    ldr r2, [r2]
    swi #0

    mov r7, #SYS_EXIT
    mov r0, #1
    swi #0
```

---

## 4. File I/O บน AArch64

### 4.1 openat/read/write/close AArch64

```asm
// aarch64_fileio.s - File I/O แบบสมบูรณ์ด้วย AArch64
// Note: AArch64 ใช้ openat แทน open (AT_FDCWD = -100)

// === Syscall Numbers ===
.equ SYS_OPENAT,     56
.equ SYS_CLOSE,      57
.equ SYS_READ,       63
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

// === Constants ===
.equ AT_FDCWD,       -100       // current working directory
.equ O_RDONLY,       0
.equ O_WRONLY,       1
.equ O_CREAT,        64
.equ O_TRUNC,        512
.equ STDOUT,         1
.equ STDERR,         2

.section .data
    input_file:  .asciz "input.txt"
    output_file: .asciz "output.txt"
    err_open:    .ascii "ไม่สามารถเปิด file ได้!\n"
    err_open_len = . - err_open

.section .bss
    .lcomm buffer, 8192         // buffer ขนาด 8KB

.section .text
.global _start

_start:
    // ====================================
    // เปิด input file ด้วย openat
    // openat(AT_FDCWD, path, O_RDONLY, 0)
    // ====================================
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD           // dirfd = AT_FDCWD
    adr x1, input_file          // pathname
    mov x2, #O_RDONLY           // flags
    mov x3, #0                  // mode
    svc #0

    // ตรวจสอบ error
    cmn x0, #4096               // ถ้า x0 > -4096: error
    b.hi open_error
    mov x19, x0                 // x19 = fd_in (callee-saved)

    // ====================================
    // เปิด/สร้าง output file
    // ====================================
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD
    adr x1, output_file
    mov x2, #(O_WRONLY | O_CREAT | O_TRUNC)
    mov x3, #0644               // permissions
    svc #0

    cmn x0, #4096
    b.hi open_error
    mov x20, x0                 // x20 = fd_out

    // ====================================
    // วนลูป copy
    // ====================================
.copy_loop:
    // อ่านจาก input
    mov x8, #SYS_READ
    mov x0, x19                 // fd_in
    adr x1, buffer
    mov x2, #8192
    svc #0

    cbz x0, .copy_done          // EOF: x0 == 0
    tbnz x0, #63, .read_error   // Error: bit 63 set (negative)

    mov x21, x0                 // x21 = bytes_read

    // เขียนไปยัง output
    mov x8, #SYS_WRITE
    mov x0, x20                 // fd_out
    adr x1, buffer
    mov x2, x21                 // bytes to write
    svc #0

    b .copy_loop

.copy_done:
    // ปิด files
    mov x8, #SYS_CLOSE
    mov x0, x19
    svc #0

    mov x8, #SYS_CLOSE
    mov x0, x20
    svc #0

    // จบโปรแกรม
    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0

.open_error:
.read_error:
    mov x8, #SYS_WRITE
    mov x0, #STDERR
    adr x1, err_open
    mov x2, #err_open_len
    svc #0

    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 5. Process Management

### 5.1 Fork และ Exec ใน ARM32

```asm
@ arm32_fork.s - Fork และ Child/Parent process
@ แสดงการทำงานของ fork syscall

.equ SYS_FORK,    2
.equ SYS_WRITE,   4
.equ SYS_GETPID,  20
.equ SYS_EXIT,    1
.equ SYS_WAITPID, 7

.section .data
    parent_msg: .ascii "Parent process running\n"
    parent_len: .word  23
    child_msg:  .ascii "Child process running\n"
    child_len:  .word  22
    newline:    .ascii "\n"

.section .text
.global _start

_start:
    @ เรียก fork() เพื่อสร้าง child process
    mov r7, #SYS_FORK
    swi #0

    @ fork() return:
    @   parent: r0 = child PID (> 0)
    @   child:  r0 = 0
    @   error:  r0 = -1

    cmp r0, #0
    beq child_code      @ r0 == 0 แสดงว่าเราเป็น child
    blt fork_error      @ r0 < 0 แสดงว่า error

parent_code:
    @ === Parent process ===
    mov r4, r0          @ บันทึก child PID ไว้ใน r4

    @ แสดง parent message
    mov r7, #SYS_WRITE
    mov r0, #1
    ldr r1, =parent_msg
    ldr r2, =parent_len
    ldr r2, [r2]
    swi #0

    @ รอ child process (waitpid)
    mov r7, #SYS_WAITPID
    mov r0, r4          @ child PID
    mov r1, #0          @ status pointer (NULL)
    mov r2, #0          @ options = 0
    swi #0

    @ parent จบ
    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

child_code:
    @ === Child process ===
    mov r7, #SYS_WRITE
    mov r0, #1
    ldr r1, =child_msg
    ldr r2, =child_len
    ldr r2, [r2]
    swi #0

    @ child จบ
    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

fork_error:
    mov r7, #SYS_EXIT
    mov r0, #1
    swi #0
```

### 5.2 Exec ใน AArch64

```asm
// aarch64_exec.s - execve ตัวอย่าง
// รัน /bin/ls ด้วย execve syscall

.equ SYS_EXECVE,     221
.equ SYS_EXIT_GROUP, 94

.section .data
    prog:     .asciz "/bin/ls"       // program ที่จะรัน
    arg0:     .asciz "/bin/ls"       // argv[0]
    arg1:     .asciz "-la"           // argv[1]
    arg_end:  .quad 0                // NULL terminator

    // สร้าง argv array
    argv:     .quad arg0             // argv[0]
              .quad arg1             // argv[1]
              .quad 0                // argv[2] = NULL

    // environment ว่าง
    envp:     .quad 0                // envp[0] = NULL

.section .text
.global _start

_start:
    // execve(prog, argv, envp)
    mov x8, #SYS_EXECVE
    adr x0, prog            // pathname
    adr x1, argv            // argv
    adr x2, envp            // envp
    svc #0

    // ถ้า execve สำเร็จ จะไม่กลับมาถึงที่นี่
    // ถ้ามาถึงที่นี่ แสดงว่า error
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 6. mmap ใน ARM

### 6.1 mmap บน ARM32

```asm
@ arm32_mmap.s - จัดการ memory ด้วย mmap

@ mmap2 syscall (ARM32 ใช้ mmap2 = 192)
.equ SYS_MMAP2,   192
.equ SYS_MUNMAP,  91
.equ SYS_WRITE,   4
.equ SYS_EXIT,    1

@ mmap flags
.equ PROT_READ,   1
.equ PROT_WRITE,  2
.equ MAP_PRIVATE, 2
.equ MAP_ANON,    32         @ MAP_ANONYMOUS

.equ PAGE_SIZE,   4096

.section .data
    msg:     .ascii "mmap สำเร็จ!\n"
    msg_len: .word  18

.section .text
.global _start

_start:
    @ ====================================
    @ จอง memory 4096 bytes ด้วย mmap
    @ mmap2(NULL, 4096, PROT_READ|PROT_WRITE,
    @        MAP_PRIVATE|MAP_ANON, -1, 0)
    @ ====================================
    mov r7, #SYS_MMAP2
    mov r0, #0              @ addr = NULL (kernel เลือกให้)
    mov r1, #PAGE_SIZE      @ length = 4096 bytes
    mov r2, #(PROT_READ | PROT_WRITE)  @ protection
    mov r3, #(MAP_PRIVATE | MAP_ANON)  @ flags
    mov r4, #-1             @ fd = -1 (anonymous)
    mov r5, #0              @ offset = 0
    swi #0

    @ ตรวจสอบ error: ถ้า r0 == MAP_FAILED (-1)
    cmn r0, #1              @ เปรียบเทียบกับ -1
    beq mmap_error

    @ บันทึก pointer ไว้
    mov r6, r0              @ r6 = mmap_ptr

    @ เขียนข้อมูลลงใน mapped memory
    ldr r3, =msg
    ldr r2, [r3]
    str r2, [r6]            @ เก็บ 4 bytes แรก

    @ แสดงว่า mmap สำเร็จ
    mov r7, #SYS_WRITE
    mov r0, #1
    ldr r1, =msg
    ldr r2, =msg_len
    ldr r2, [r2]
    swi #0

    @ ====================================
    @ คืน memory ด้วย munmap
    @ munmap(addr, length)
    @ ====================================
    mov r7, #SYS_MUNMAP
    mov r0, r6              @ address ที่ได้จาก mmap
    mov r1, #PAGE_SIZE      @ length
    swi #0

    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

mmap_error:
    mov r7, #SYS_EXIT
    mov r0, #1
    swi #0
```

### 6.2 mmap บน AArch64

```asm
// aarch64_mmap.s - mmap ใน AArch64

.equ SYS_MMAP,       222
.equ SYS_MUNMAP,     215
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

.equ PROT_READ,      1
.equ PROT_WRITE,     2
.equ MAP_PRIVATE,    2
.equ MAP_ANONYMOUS,  32

.section .data
    ok_msg:  .ascii "mmap AArch64 สำเร็จ!\n"
    ok_len = . - ok_msg

.section .text
.global _start

_start:
    // mmap(NULL, 65536, PROT_R|PROT_W, MAP_PRIVATE|MAP_ANON, -1, 0)
    mov x8, #SYS_MMAP
    mov x0, #0                          // addr = NULL
    mov x1, #65536                      // size = 64KB
    mov x2, #(PROT_READ | PROT_WRITE)   // protection
    mov x3, #(MAP_PRIVATE | MAP_ANONYMOUS) // flags
    mov x4, #-1                         // fd = -1
    mov x5, #0                          // offset = 0
    svc #0

    // ตรวจ error
    cmn x0, #4096
    b.hi mmap_error

    mov x19, x0             // x19 = mapped address (callee-saved)

    // ใช้งาน mapped memory
    mov x2, #0x48656c6c     // "Hell"
    str x2, [x19]           // เขียนไปยัง mapped area

    // แสดงผลสำเร็จ
    mov x8, #SYS_WRITE
    mov x0, #1
    adr x1, ok_msg
    mov x2, #ok_len
    svc #0

    // munmap
    mov x8, #SYS_MUNMAP
    mov x0, x19             // address
    mov x1, #65536          // size
    svc #0

    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0

mmap_error:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 7. Signal Handling ใน ARM

### 7.1 Signal Handling ARM32

```asm
@ arm32_signal.s - จัดการ signals ด้วย sigaction

.equ SYS_SIGACTION, 67       @ rt_sigaction คือ 174
.equ SYS_KILL,      37
.equ SYS_GETPID,    20
.equ SYS_WRITE,     4
.equ SYS_EXIT,      1
.equ SYS_PAUSE,     29       @ รอ signal

.equ SIGINT,        2        @ Ctrl+C
.equ SIGUSR1,       10       @ user-defined signal 1

@ struct sigaction (simplified)
@ sa_handler: word
@ sa_flags:   word
@ sa_restorer: word
@ sa_mask:    8 bytes (64-bit mask)

.section .data
    caught_msg: .ascii "SIGUSR1 ถูก catch แล้ว!\n"
    caught_len: .word  30
    int_msg:    .ascii "SIGINT ถูก catch! กำลังออก...\n"
    int_len:    .word  31

.section .bss
    .align 4
    sa_struct:  .space 32    @ struct sigaction

.section .text
.global _start
.global sigusr1_handler
.global sigint_handler

sigusr1_handler:
    @ Handler สำหรับ SIGUSR1
    mov r7, #SYS_WRITE
    mov r0, #1
    ldr r1, =caught_msg
    ldr r2, =caught_len
    ldr r2, [r2]
    swi #0
    bx lr                    @ กลับจาก handler

sigint_handler:
    @ Handler สำหรับ SIGINT
    mov r7, #SYS_WRITE
    mov r0, #1
    ldr r1, =int_msg
    ldr r2, =int_len
    ldr r2, [r2]
    swi #0

    @ ออกจากโปรแกรม
    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

_start:
    @ ตั้งค่า signal handler สำหรับ SIGUSR1
    @ rt_sigaction(signum, &new_action, &old_action, sigsetsize)
    ldr r0, =sa_struct
    ldr r1, =sigusr1_handler
    str r1, [r0]             @ sa_handler = sigusr1_handler
    mov r1, #0
    str r1, [r0, #4]         @ sa_flags = 0

    mov r7, #174             @ rt_sigaction
    mov r0, #SIGUSR1         @ signal number
    ldr r1, =sa_struct       @ new action
    mov r2, #0               @ old action = NULL
    mov r3, #8               @ sigsetsize = 8
    swi #0

    @ ดึง PID ของตัวเอง
    mov r7, #SYS_GETPID
    swi #0
    mov r4, r0               @ r4 = our PID

    @ ส่ง SIGUSR1 ให้ตัวเอง
    mov r7, #SYS_KILL
    mov r0, r4               @ pid = our PID
    mov r1, #SIGUSR1         @ signal
    swi #0

    @ รอ signal ถัดไป
    mov r7, #SYS_PAUSE
    swi #0

    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0
```

---

## 8. Socket Programming ใน ARM

### 8.1 TCP Server บน AArch64

```asm
// aarch64_tcpserver.s - TCP Server อย่างง่าย บน AArch64
// รับการ connect และ echo ข้อความกลับ

.equ SYS_SOCKET,     198
.equ SYS_BIND,       200
.equ SYS_LISTEN,     201
.equ SYS_ACCEPT,     202
.equ SYS_READ,       63
.equ SYS_WRITE,      64
.equ SYS_CLOSE,      57
.equ SYS_EXIT_GROUP, 94

.equ AF_INET,        2       // IPv4
.equ SOCK_STREAM,    1       // TCP
.equ IPPROTO_TCP,    6

.equ SOL_SOCKET,     1
.equ SO_REUSEADDR,   2

.section .data
    // struct sockaddr_in
    .align 4
    server_addr:
        .2byte AF_INET          // sin_family
        .2byte 0x5000           // sin_port = 80 (big-endian: 0x0050 -> 0x5000)
        .4byte 0                // sin_addr = INADDR_ANY (0.0.0.0)
        .8byte 0                // padding

    banner:   .ascii "AArch64 TCP Echo Server\r\n"
    ban_len = . - banner

    hello:    .ascii "Hello from ARM64 Assembly!\r\n"
    hello_len = . - hello

.section .bss
    .lcomm recv_buf, 1024

.section .text
.global _start

_start:
    // === สร้าง socket ===
    mov x8, #SYS_SOCKET
    mov x0, #AF_INET            // domain = IPv4
    mov x1, #SOCK_STREAM        // type = TCP
    mov x2, #0                  // protocol = 0
    svc #0

    cmn x0, #4096
    b.hi socket_error
    mov x19, x0                 // x19 = server_fd

    // === bind ===
    mov x8, #SYS_BIND
    mov x0, x19                 // sockfd
    adr x1, server_addr         // struct sockaddr *
    mov x2, #16                 // addrlen
    svc #0

    cmn x0, #4096
    b.hi socket_error

    // === listen ===
    mov x8, #SYS_LISTEN
    mov x0, x19                 // sockfd
    mov x1, #5                  // backlog
    svc #0

    // === รับ connection loop ===
accept_loop:
    mov x8, #SYS_ACCEPT
    mov x0, x19                 // server fd
    mov x1, #0                  // client addr = NULL
    mov x2, #0                  // addrlen = NULL
    svc #0

    cmn x0, #4096
    b.hi accept_loop            // ถ้า error ลอง accept ใหม่
    mov x20, x0                 // x20 = client_fd

    // ส่ง hello message
    mov x8, #SYS_WRITE
    mov x0, x20
    adr x1, hello
    mov x2, #hello_len
    svc #0

    // echo loop
.echo_loop:
    mov x8, #SYS_READ
    mov x0, x20                 // client_fd
    adr x1, recv_buf
    mov x2, #1024
    svc #0

    cbz x0, .close_client       // EOF
    tbnz x0, #63, .close_client // error

    // echo back
    mov x21, x0
    mov x8, #SYS_WRITE
    mov x0, x20
    adr x1, recv_buf
    mov x2, x21
    svc #0

    b .echo_loop

.close_client:
    mov x8, #SYS_CLOSE
    mov x0, x20
    svc #0

    b accept_loop               // รับ connection ต่อไป

socket_error:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 9. โปรแกรมสมบูรณ์: cat

### 9.1 cat ใน ARM32

```asm
@ arm32_cat.s - cat command ด้วย ARM32
@ ใช้: ./cat [file1 file2 ...] หรือ stdin ถ้าไม่มี args

.equ SYS_READ,   3
.equ SYS_WRITE,  4
.equ SYS_OPEN,   5
.equ SYS_CLOSE,  6
.equ SYS_EXIT,   1

.equ STDIN,      0
.equ STDOUT,     1
.equ STDERR,     2
.equ O_RDONLY,   0

.section .bss
    .lcomm buf, 4096

.section .text
.global _start

_start:
    @ โหลด argc และ argv จาก stack
    @ ตอน start: sp -> argc, sp+4 -> argv[0], sp+8 -> argv[1], ...
    pop {r4}            @ r4 = argc
    mov r5, sp          @ r5 = argv pointer

    @ ถ้า argc == 1 ไม่มี arguments อ่านจาก stdin
    cmp r4, #1
    beq read_stdin

    @ ข้าม argv[0] (ชื่อโปรแกรม)
    add r5, r5, #4      @ r5 = &argv[1]
    sub r4, r4, #1      @ argc--

process_args:
    cmp r4, #0
    beq exit_success

    @ โหลด filename
    ldr r6, [r5]        @ r6 = argv[i]

    @ เปิด file
    mov r7, #SYS_OPEN
    mov r0, r6          @ pathname
    mov r1, #O_RDONLY
    mov r2, #0
    swi #0

    cmp r0, #0
    blt file_error

    mov r7, r0          @ r7 = fd (ชั่วคราว)
    @ แต่ r7 ต้องใช้สำหรับ syscall ด้วย ต้องเซฟก่อน
    push {r4, r5, r6, r7}  @ เซฟ state
    mov r8, r0          @ r8 = fd

cat_loop:
    @ อ่านจาก file
    mov r7, #SYS_READ
    mov r0, r8          @ fd
    ldr r1, =buf
    mov r2, #4096
    swi #0

    cmp r0, #0
    beq cat_done        @ EOF
    blt cat_done        @ error

    @ เขียนไปยัง stdout
    mov r2, r0          @ bytes to write
    mov r7, #SYS_WRITE
    mov r0, #STDOUT
    ldr r1, =buf
    swi #0

    b cat_loop

cat_done:
    @ ปิด file
    mov r7, #SYS_CLOSE
    mov r0, r8
    swi #0

    pop {r4, r5, r6, r7}   @ คืน state

    @ ไปยัง argument ถัดไป
    add r5, r5, #4
    sub r4, r4, #1
    b process_args

read_stdin:
    mov r8, #STDIN
    b cat_loop

file_error:
    @ TODO: แสดง error message พร้อมชื่อ file

exit_success:
    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0
```

### 9.2 cat ใน AArch64 (สมบูรณ์)

```asm
// aarch64_cat.s - cat command แบบ AArch64 สมบูรณ์

.equ SYS_OPENAT,     56
.equ SYS_CLOSE,      57
.equ SYS_READ,       63
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

.equ AT_FDCWD,       -100
.equ O_RDONLY,       0
.equ STDIN,          0
.equ STDOUT,         1
.equ STDERR,         2
.equ BUF_SIZE,       65536   // 64KB buffer

.section .data
    err_msg:  .ascii "cat: ไม่สามารถเปิด file: "
    err_len = . - err_msg
    newline:  .ascii "\n"

.section .bss
    .lcomm buffer, BUF_SIZE

.section .text
.global _start

_start:
    // อ่าน argc จาก stack
    ldr x19, [sp]               // x19 = argc
    add x20, sp, #8             // x20 = argv pointer

    // ถ้า argc == 1 อ่านจาก stdin
    cmp x19, #1
    b.eq .do_stdin

    // ข้าม argv[0]
    add x20, x20, #8            // x20 = &argv[1]
    sub x19, x19, #1            // argc--

.next_file:
    cbz x19, .exit_ok

    // โหลด filename
    ldr x21, [x20]              // x21 = filename

    // เปิด file
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD
    mov x1, x21
    mov x2, #O_RDONLY
    mov x3, #0
    svc #0

    cmn x0, #4096
    b.hi .file_err

    mov x22, x0                 // x22 = fd

.cat_fd:
    // อ่าน
    mov x8, #SYS_READ
    mov x0, x22
    adr x1, buffer
    mov x2, #BUF_SIZE
    svc #0

    cbz x0, .close_fd           // EOF
    tbnz x0, #63, .close_fd    // error

    // เขียน
    mov x23, x0
    mov x8, #SYS_WRITE
    mov x0, #STDOUT
    adr x1, buffer
    mov x2, x23
    svc #0

    b .cat_fd

.close_fd:
    mov x8, #SYS_CLOSE
    mov x0, x22
    svc #0

    add x20, x20, #8            // ไป arg ถัดไป
    sub x19, x19, #1
    b .next_file

.do_stdin:
    mov x22, #STDIN
    b .cat_fd

.file_err:
    // แสดง error message
    mov x8, #SYS_WRITE
    mov x0, #STDERR
    adr x1, err_msg
    mov x2, #err_len
    svc #0

    mov x8, #SYS_WRITE
    mov x0, #STDERR
    mov x1, x21                 // filename
    // คำนวณ length ของ filename
    mov x24, x21
.strlen_loop:
    ldrb w25, [x24], #1
    cbnz w25, .strlen_loop
    sub x2, x24, x21
    sub x2, x2, #1              // ไม่นับ null terminator
    svc #0

    mov x8, #SYS_WRITE
    mov x0, #STDERR
    adr x1, newline
    mov x2, #1
    svc #0

    add x20, x20, #8
    sub x19, x19, #1
    b .next_file

.exit_ok:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0
```

---

## 10. โปรแกรมสมบูรณ์: head

### 10.1 head ใน AArch64

```asm
// aarch64_head.s - head command (แสดง 10 บรรทัดแรก)
// ใช้: ./head [file]

.equ SYS_OPENAT,     56
.equ SYS_CLOSE,      57
.equ SYS_READ,       63
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

.equ AT_FDCWD,       -100
.equ O_RDONLY,       0
.equ STDOUT,         1
.equ STDIN,          0
.equ MAX_LINES,      10

.section .bss
    .lcomm buf, 4096

.section .text
.global _start

_start:
    ldr x19, [sp]           // argc
    add x20, sp, #8         // argv

    // เลือก fd
    cmp x19, #1
    b.eq .use_stdin

    // เปิด file argument
    ldr x21, [x20, #8]      // argv[1]
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD
    mov x1, x21
    mov x2, #O_RDONLY
    mov x3, #0
    svc #0

    cmn x0, #4096
    b.hi .err
    mov x22, x0
    b .head_start

.use_stdin:
    mov x22, #STDIN

.head_start:
    mov x23, #0             // x23 = line_count

.read_loop:
    // อ่านทีละ 1 byte
    mov x8, #SYS_READ
    mov x0, x22
    adr x1, buf
    mov x2, #1
    svc #0

    cbz x0, .done_head
    tbnz x0, #63, .done_head

    // เขียน byte
    mov x8, #SYS_WRITE
    mov x0, #STDOUT
    adr x1, buf
    mov x2, #1
    svc #0

    // ตรวจว่าเป็น newline หรือเปล่า
    ldrb w24, buf
    cmp w24, #10            // '\n'
    b.ne .read_loop

    // เพิ่ม line count
    add x23, x23, #1
    cmp x23, #MAX_LINES
    b.lt .read_loop

.done_head:
    // ปิด file ถ้าไม่ใช่ stdin
    cmp x22, #STDIN
    b.eq .exit_ok
    mov x8, #SYS_CLOSE
    mov x0, #22
    svc #0

.exit_ok:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0

.err:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 11. โปรแกรมสมบูรณ์: wc (word count)

### 11.1 wc ใน AArch64

```asm
// aarch64_wc.s - wc command (นับ lines, words, bytes)
// Output: lines words bytes filename

.equ SYS_OPENAT,     56
.equ SYS_CLOSE,      57
.equ SYS_READ,       63
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

.equ AT_FDCWD,       -100
.equ O_RDONLY,       0
.equ STDIN,          0
.equ STDOUT,         1

.section .data
    space:   .ascii " "
    newline: .ascii "\n"

.section .bss
    .lcomm buf, 4096
    .lcomm num_buf, 32      // สำหรับแปลง number -> string

.section .text
.global _start

// ฟังก์ชัน: แปลง uint64 -> string decimal
// Input:  x0 = number
// Output: x0 = pointer to string, x1 = length
// ใช้ num_buf เป็น buffer
itoa_decimal:
    adr x2, num_buf
    add x2, x2, #31         // ชี้ที่ end ของ buffer
    strb wzr, [x2]          // null terminate
    mov x3, #0              // length counter

    // กรณีพิเศษ: number == 0
    cbnz x0, .itoa_loop
    sub x2, x2, #1
    mov w4, #'0'
    strb w4, [x2]
    mov x3, #1
    b .itoa_done

.itoa_loop:
    cbz x0, .itoa_done
    mov x4, #10
    udiv x5, x0, x4         // x5 = x0 / 10
    msub x6, x5, x4, x0     // x6 = x0 % 10 (remainder)
    add x6, x6, #'0'
    sub x2, x2, #1
    strb w6, [x2]
    add x3, x3, #1
    mov x0, x5
    b .itoa_loop

.itoa_done:
    mov x0, x2
    mov x1, x3
    ret

_start:
    ldr x19, [sp]           // argc
    add x20, sp, #8         // argv

    // ตรวจ args
    cmp x19, #1
    b.eq .use_stdin

    ldr x21, [x20, #8]      // argv[1] = filename
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD
    mov x1, x21
    mov x2, #O_RDONLY
    mov x3, #0
    svc #0

    cmn x0, #4096
    b.hi .exit_err
    mov x22, x0             // x22 = fd
    b .count_start

.use_stdin:
    mov x22, #STDIN
    mov x21, #0             // ไม่มี filename

.count_start:
    mov x23, #0             // lines
    mov x24, #0             // words
    mov x25, #0             // bytes
    mov x26, #0             // in_word flag

.count_loop:
    // อ่านทีละ block
    mov x8, #SYS_READ
    mov x0, x22
    adr x1, buf
    mov x2, #4096
    svc #0

    cbz x0, .count_done
    tbnz x0, #63, .count_done

    mov x27, x0             // bytes_read
    add x25, x25, x27       // total_bytes += bytes_read

    // วิเคราะห์แต่ละ byte
    adr x28, buf
.byte_loop:
    cbz x27, .count_loop
    sub x27, x27, #1

    ldrb w0, [x28], #1

    // ตรวจ newline
    cmp w0, #10             // '\n'
    b.ne .check_space
    add x23, x23, #1        // lines++
    mov x26, #0             // not in word
    b .byte_loop

.check_space:
    // ตรวจ whitespace (space, tab, newline, carriage return)
    cmp w0, #32             // ' '
    b.eq .is_space
    cmp w0, #9              // '\t'
    b.eq .is_space
    cmp w0, #13             // '\r'
    b.eq .is_space

    // ตัวอักขระปกติ
    cbnz x26, .byte_loop    // ถ้าอยู่ใน word แล้ว ข้ามไป
    add x24, x24, #1        // words++
    mov x26, #1             // in_word = 1
    b .byte_loop

.is_space:
    mov x26, #0             // in_word = 0
    b .byte_loop

.count_done:
    // ปิด file
    cmp x22, #STDIN
    b.eq .print_result
    mov x8, #SYS_CLOSE
    mov x0, x22
    svc #0

.print_result:
    // แสดง: lines words bytes [filename]

    // แสดง lines
    mov x0, x23
    bl itoa_decimal
    mov x8, #SYS_WRITE
    mov x2, x1
    mov x1, x0
    mov x0, #STDOUT
    svc #0
    // space
    mov x8, #SYS_WRITE
    mov x0, #STDOUT
    adr x1, space
    mov x2, #1
    svc #0

    // แสดง words
    mov x0, x24
    bl itoa_decimal
    mov x8, #SYS_WRITE
    mov x2, x1
    mov x1, x0
    mov x0, #STDOUT
    svc #0
    // space
    mov x8, #SYS_WRITE
    mov x0, #STDOUT
    adr x1, space
    mov x2, #1
    svc #0

    // แสดง bytes
    mov x0, x25
    bl itoa_decimal
    mov x8, #SYS_WRITE
    mov x2, x1
    mov x1, x0
    mov x0, #STDOUT
    svc #0

    // แสดง filename ถ้ามี
    cbz x21, .print_newline
    // space
    mov x8, #SYS_WRITE
    mov x0, #STDOUT
    adr x1, space
    mov x2, #1
    svc #0
    // filename (คำนวณ length)
    mov x0, x21
    bl itoa_decimal         // ใช้ strlen แทน (simplified ใช้ write แบบ loop)
    // TODO: แสดง filename จริงๆ

.print_newline:
    mov x8, #SYS_WRITE
    mov x0, #STDOUT
    adr x1, newline
    mov x2, #1
    svc #0

.exit_ok2:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0

.exit_err:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 12. Error Handling กับ errno

### 12.1 errno ใน ARM Assembly

```asm
// aarch64_errno.s - ตัวอย่าง error handling กับ errno

// syscall ส่งคืนค่า negative ถ้า error
// ค่าที่ส่งคืน: -errno (เช่น -2 คือ ENOENT)
// errno values ที่ใช้บ่อย:

// EPERM   = 1    Operation not permitted
// ENOENT  = 2    No such file or directory
// ESRCH   = 3    No such process
// EINTR   = 4    Interrupted system call
// EIO     = 5    I/O error
// ENOEXEC = 8    Exec format error
// EBADF   = 9    Bad file descriptor
// ECHILD  = 10   No child processes
// EAGAIN  = 11   Resource temporarily unavailable
// ENOMEM  = 12   Out of memory
// EACCES  = 13   Permission denied
// EFAULT  = 14   Bad address
// EBUSY   = 16   Device busy
// EEXIST  = 17   File exists
// ENODEV  = 19   No such device
// ENOTDIR = 20   Not a directory
// EISDIR  = 21   Is a directory
// EINVAL  = 22   Invalid argument
// EMFILE  = 24   Too many open files
// ENOSPC  = 28   No space left on device
// ESPIPE  = 29   Illegal seek
// EROFS   = 30   Read-only file system
// EPIPE   = 32   Broken pipe
// ENOTSUP = 95   Not supported

.equ SYS_OPENAT,     56
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

.equ AT_FDCWD,       -100
.equ O_RDONLY,       0
.equ STDERR,         2

.section .data
    nofile:     .asciz "nonexistent.txt"    // file ที่ไม่มีอยู่

    // Error messages
    err_generic:  .ascii "System call error\n"
    err_gen_len = . - err_generic
    err_enoent:   .ascii "ENOENT: ไม่พบ file\n"
    err_enoent_len = . - err_enoent
    err_eacces:   .ascii "EACCES: ไม่มีสิทธิ์\n"
    err_eacces_len = . - err_eacces

.section .text
.global _start

// check_error: ตรวจสอบ return value จาก syscall
// Input:  x0 = syscall return value
// Output: x0 = ค่าเดิม (ถ้าไม่ error), ไม่ return ถ้า fatal error
check_syscall_error:
    // ตรวจว่าเป็น error หรือเปล่า
    cmn x0, #4096           // ถ้า x0 > -4096: ถือว่า error
    b.ls .no_error

    // คำนวณ errno (neg ของ return value)
    neg x1, x0              // x1 = errno

    // ENOENT = 2
    cmp x1, #2
    b.ne .check_eacces
    mov x8, #SYS_WRITE
    mov x0, #STDERR
    adr x1, err_enoent
    mov x2, #err_enoent_len
    svc #0
    b .print_done

.check_eacces:
    // EACCES = 13
    cmp x1, #13
    b.ne .generic_error
    mov x8, #SYS_WRITE
    mov x0, #STDERR
    adr x1, err_eacces
    mov x2, #err_eacces_len
    svc #0
    b .print_done

.generic_error:
    mov x8, #SYS_WRITE
    mov x0, #STDERR
    adr x1, err_generic
    mov x2, #err_gen_len
    svc #0

.print_done:
    // จบด้วย exit code 1
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0

.no_error:
    ret

_start:
    // ลองเปิด file ที่ไม่มีอยู่
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD
    adr x1, nofile
    mov x2, #O_RDONLY
    mov x3, #0
    svc #0

    // ตรวจ error
    bl check_syscall_error

    // ถ้าไม่ error (ไม่น่าจะเกิด)
    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0
```

---

## 13. Syscall Table Reference

### 13.1 ARM32 Syscall Table (คัดสรร)

```
ARM32 Linux Syscall Table (เลือกที่สำคัญ)
==========================================
No.  Name          Description
------------------------------------------------------------
0    restart_syscall  Restart syscall
1    exit             Exit thread
2    fork             Fork process
3    read             Read from fd
4    write            Write to fd
5    open             Open file
6    close            Close fd
7    waitpid          Wait for process
8    creat            Create file
9    link             Create hard link
10   unlink           Delete file
11   execve           Execute program
12   chdir            Change directory
13   time             Get time
14   mknod            Create special file
15   chmod            Change permissions
16   lchown           Change owner (no follow)
20   getpid           Get process ID
21   mount            Mount filesystem
22   umount           Unmount filesystem
23   setuid           Set user ID
24   getuid           Get user ID
25   stime            Set time
26   ptrace           Process tracing
27   alarm            Set alarm
28   oldfstat         (obsolete)
29   pause            Pause until signal
30   utime            Set file access/mod times
33   access           Check file access
36   sync             Sync filesystem
37   kill             Send signal
38   rename           Rename file
39   mkdir            Create directory
40   rmdir            Remove directory
41   dup              Duplicate fd
42   pipe             Create pipe
43   times            Get process times
45   brk              Change data segment
46   setgid           Set group ID
47   getgid           Get group ID
48   signal           Signal handling (old)
49   geteuid          Get effective user ID
50   getegid          Get effective group ID
54   ioctl            I/O control
55   fcntl            File control
57   setpgid          Set process group
60   umask            Set file creation mask
61   chroot           Change root directory
63   dup2             Duplicate fd to fd2
64   getppid          Get parent PID
65   getpgrp          Get process group
66   setsid           Create new session
78   gettimeofday     Get time + timezone
79   settimeofday     Set time + timezone
85   readlink         Read symbolic link
91   munmap           Unmap memory
92   truncate         Truncate file
93   ftruncate        Truncate open file
94   fchmod           Change file mode
102  socketcall       Socket operations (old)
140  llseek           Seek large file
174  rt_sigaction     Set signal action
175  rt_sigprocmask   Mask signals
192  mmap2            Map memory (2nd version)
195  stat64           Get file status
197  fstat64          Get open file status
204  waitid           Wait for child
220  getdents64       Get directory entries
221  fcntl64          File control (64-bit)
252  exit_group       Exit all threads
281  socket           Create socket
282  bind             Bind socket
283  connect          Connect socket
284  listen           Listen on socket
285  accept           Accept connection
286  getsockname      Get socket name
287  getpeername      Get peer name
288  socketpair       Create socket pair
289  send             Send message
290  recv             Receive message
291  sendto           Send to address
292  recvfrom         Receive from address
293  shutdown         Shutdown connection
294  setsockopt       Set socket option
295  getsockopt       Get socket option
296  sendmsg          Send message (iov)
297  recvmsg          Receive message (iov)
```

### 13.2 AArch64 Syscall Table (คัดสรร)

```
AArch64 Linux Syscall Table (เลือกที่สำคัญ)
=============================================
No.  Name          Description
------------------------------------------------------------
0    io_setup         Async I/O setup
3    io_cancel        Cancel async I/O
4    io_getevents     Get async I/O events
5    setxattr         Set extended attribute
6    lsetxattr        Set xattr (no follow)
7    fsetxattr        Set xattr on fd
8    getxattr         Get extended attribute
9    lgetxattr        Get xattr (no follow)
10   fgetxattr        Get xattr on fd
11   listxattr        List xattrs
12   llistxattr       List xattrs (no follow)
13   flistxattr       List xattrs on fd
14   removexattr      Remove xattr
15   lremovexattr     Remove xattr (no follow)
16   fremovexattr     Remove xattr on fd
17   getcwd           Get current directory
18   lookup_dcookie   Dcache lookup
19   eventfd2         Create eventfd
20   epoll_create1    Create epoll fd
21   epoll_ctl        Control epoll
22   epoll_pwait      Epoll wait with signals
23   dup              Duplicate fd
24   dup3             Duplicate fd to fd3
25   fcntl            File control
26   inotify_init1    Inotify init
27   inotify_add_watch Add watch
28   inotify_rm_watch  Remove watch
29   ioctl            I/O control
30   ioprio_set       Set I/O priority
31   ioprio_get       Get I/O priority
32   flock            Apply lock
33   mknodat          Create file node
34   mkdirat          Create directory
35   unlinkat         Unlink file
36   symlinkat        Create symlink
37   linkat           Create hard link
38   renameat         Rename file
39   umount2          Unmount filesystem
40   mount            Mount filesystem
41   pivot_root       Change root filesystem
42   nfsservctl       NFS server control
43   statfs           Get filesystem stats
44   fstatfs          Get fs stats (fd)
45   truncate         Truncate file
46   ftruncate        Truncate open file
47   fallocate        Preallocate file space
48   faccessat        Check access
49   chdir            Change directory
50   fchdir           Change directory (fd)
51   chroot           Change root
52   fchmod           Change file mode
53   fchmodat         Change mode at
54   fchownat         Change owner at
55   fchown           Change owner
56   openat           Open file
57   close            Close fd
58   vhangup          Virtual hangup
59   pipe2            Create pipe
60   quotactl         Quota control
61   getdents64       Get directory entries
62   lseek            Seek file
63   read             Read from fd
64   write            Write to fd
65   readv            Read vectors
66   writev           Write vectors
67   pread64          Read at offset
68   pwrite64         Write at offset
69   preadv           Read vectors at offset
70   pwritev          Write vectors at offset
71   sendfile         Transfer data
72   pselect6         Sync I/O multiplex
73   ppoll            Poll descriptors
74   signalfd4        Signal fd
75   vmsplice         Splice user pages
76   splice           Splice data
77   tee              Duplicate pipe data
78   readlinkat       Read symlink
79   fstatat          Get file status
80   fstat            Get file status (fd)
81   sync             Sync buffers
82   fsync            Sync file data
83   fdatasync        Sync data only
84   sync_file_range  Sync file range
85   timerfd_create   Create timer fd
86   timerfd_settime  Set timer
87   timerfd_gettime  Get timer status
88   utimensat        Update timestamps
89   acct             Process accounting
90   capget           Get capabilities
91   capset           Set capabilities
92   personality      Set process personality
93   exit             Exit thread
94   exit_group       Exit all threads
95   waitid           Wait for child
96   set_tid_address  Set tid address
97   unshare          Unshare namespaces
98   futex            Fast mutex
99   set_robust_list  Set robust futex list
100  get_robust_list  Get robust futex list
101  nanosleep        Nanosecond sleep
102  getitimer        Get interval timer
103  setitimer        Set interval timer
104  kexec_load       Load kernel for exec
105  init_module      Load module
106  delete_module    Unload module
107  timer_create     Create POSIX timer
108  timer_gettime    Get timer
109  timer_getoverrun Get overruns
110  timer_settime    Set timer
111  timer_delete     Delete timer
112  clock_settime    Set clock
113  clock_gettime    Get clock
114  clock_getres     Get clock resolution
115  clock_nanosleep  High-res sleep
116  syslog           Kernel log
117  ptrace           Process tracing
118  sched_setparam   Set scheduling params
119  sched_setscheduler Set scheduler
120  sched_getscheduler Get scheduler
121  sched_getparam   Get scheduling params
122  sched_setaffinity Set CPU affinity
123  sched_getaffinity Get CPU affinity
124  sched_yield      Yield processor
125  sched_get_priority_max Get max priority
126  sched_get_priority_min Get min priority
127  sched_rr_get_interval Get RR interval
128  restart_syscall  Restart syscall
129  kill             Send signal
130  tkill            Send signal to thread
131  tgkill           Send signal to group
132  sigaltstack      Set alt signal stack
133  rt_sigsuspend    Wait for signal
134  rt_sigaction     Set signal action
135  rt_sigprocmask   Block signals
136  rt_sigpending    Get pending signals
137  rt_sigtimedwait  Wait for signal (timed)
138  rt_sigqueueinfo  Queue signal + data
139  rt_sigreturn     Return from handler
140  setpriority      Set process priority
141  getpriority      Get process priority
142  reboot           Reboot system
143  setregid         Set real+effective GID
144  setgid           Set group ID
145  setreuid         Set real+effective UID
146  setuid           Set user ID
147  setresuid        Set real/effective/saved UID
148  getresuid        Get real/effective/saved UID
149  setresgid        Set real/effective/saved GID
150  getresgid        Get real/effective/saved GID
151  setfsuid         Set filesystem UID
152  setfsgid         Set filesystem GID
153  times            Get process times
154  setpgid          Set process group ID
155  getpgid          Get process group ID
156  getsid           Get session ID
157  setsid           Set session ID
158  getgroups        Get group list
159  setgroups        Set group list
160  uname            Get system name
161  sethostname      Set hostname
162  setdomainname    Set domain name
163  getrlimit        Get resource limits
164  setrlimit        Set resource limits
165  getrusage        Get resource usage
166  umask            Set file creation mask
167  prctl            Process control
168  getcpu           Get CPU + NUMA node
169  gettimeofday     Get time + timezone
170  settimeofday     Set time + timezone
171  adjtimex         Tune kernel clock
172  getpid           Get process ID
173  getppid          Get parent PID
174  getuid           Get user ID
175  geteuid          Get effective UID
176  getgid           Get group ID
177  getegid          Get effective GID
178  gettid           Get thread ID
179  sysinfo          Get system info
180  mq_open          Open message queue
...
198  socket           Create socket
199  socketpair       Create socket pair
200  bind             Bind socket
201  listen           Listen on socket
202  accept           Accept connection
203  connect          Connect socket
204  getsockname      Get socket name
205  getpeername      Get peer name
206  sendto           Send message
207  recvfrom         Receive message
208  setsockopt       Set socket option
209  getsockopt       Get socket option
210  shutdown         Shutdown socket
211  sendmsg          Send message vector
212  recvmsg          Receive message vector
213  readahead        Initiate file readahead
214  brk              Change data segment
215  munmap           Unmap memory
216  mremap           Remap memory
217  add_key          Add key
218  request_key      Request key
219  keyctl           Key management
220  clone            Clone process
221  execve           Execute program
222  mmap             Map memory
223  fadvise64        File advice
224  swapon           Enable swap
225  swapoff          Disable swap
226  mprotect         Set memory protection
227  msync            Sync mapped memory
228  mlock            Lock memory
229  munlock          Unlock memory
230  mlockall         Lock all memory
231  munlockall       Unlock all memory
232  mincore          Query page residency
233  madvise          Memory advice
234  remap_file_pages Remap pages
235  mbind            Set NUMA policy
236  get_mempolicy    Get NUMA policy
237  set_mempolicy    Set NUMA policy
238  migrate_pages    Migrate pages
239  move_pages       Move pages
```

---

## 14. Compilation และ QEMU

### 14.1 Setup สำหรับ ARM Development

```bash
# ติดตั้ง cross-compiler และ QEMU บน Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    gcc-arm-linux-gnueabi \       # ARM32 compiler
    binutils-arm-linux-gnueabi \  # ARM32 assembler + linker
    gcc-aarch64-linux-gnu \       # AArch64 compiler
    binutils-aarch64-linux-gnu \  # AArch64 assembler + linker
    qemu-user \                   # QEMU user-mode emulator
    qemu-user-static              # QEMU static binary

# ตรวจสอบการติดตั้ง
arm-linux-gnueabi-as --version
aarch64-linux-gnu-as --version
qemu-arm --version
qemu-aarch64 --version
```

### 14.2 Makefile สำหรับโปรเจค ARM

```makefile
# Makefile สำหรับ ARM Assembly โปรเจค

# Cross compilers
ARM32_AS  = arm-linux-gnueabi-as
ARM32_LD  = arm-linux-gnueabi-ld
ARM64_AS  = aarch64-linux-gnu-as
ARM64_LD  = aarch64-linux-gnu-ld

# QEMU
QEMU_ARM  = qemu-arm
QEMU_A64  = qemu-aarch64

# Assembler flags
ARM32_FLAGS = -mfloat-abi=soft
ARM64_FLAGS =

# ============ ARM32 targets ============
%.arm32.o: %.s
	$(ARM32_AS) $(ARM32_FLAGS) $< -o $@

%.arm32: %.arm32.o
	$(ARM32_LD) $< -o $@

# ============ AArch64 targets ============
%.a64.o: %.s
	$(ARM64_AS) $(ARM64_FLAGS) $< -o $@

%.a64: %.a64.o
	$(ARM64_LD) $< -o $@

# ============ Convenience rules ============
run-arm32-%: %.arm32
	$(QEMU_ARM) ./$<

run-a64-%: %.a64
	$(QEMU_A64) ./$<

# ============ Example targets ============
all: hello.arm32 hello.a64 cat.arm32 cat.a64 wc.a64

clean:
	rm -f *.o *.arm32 *.a64

.PHONY: all clean
```

### 14.3 วิธี Compile และ Run

```bash
# === ARM32 ===

# Assemble
arm-linux-gnueabi-as -o hello.o arm32_hello.s

# Link
arm-linux-gnueabi-ld -o hello hello.o

# Run บน QEMU
qemu-arm ./hello

# Debug กับ GDB
qemu-arm -g 1234 ./hello &
arm-linux-gnueabi-gdb hello
(gdb) target remote :1234
(gdb) break _start
(gdb) continue

# === AArch64 ===

# Assemble
aarch64-linux-gnu-as -o hello64.o aarch64_hello.s

# Link
aarch64-linux-gnu-ld -o hello64 hello64.o

# Run บน QEMU
qemu-aarch64 ./hello64

# Debug
qemu-aarch64 -g 1234 ./hello64 &
aarch64-linux-gnu-gdb hello64
(gdb) target remote :1234
(gdb) break _start
(gdb) continue

# === ดู Syscall ที่ถูกเรียก ===
qemu-arm -strace ./hello 2>&1 | head -20

# === รัน บน Raspberry Pi จริง (ถ้ามี) ===
scp hello pi@raspberrypi:~/
ssh pi@raspberrypi ./hello
```

### 14.4 Objdump และ Analysis

```bash
# ดู disassembly ของ ARM32
arm-linux-gnueabi-objdump -d hello.o

# ดู sections
arm-linux-gnueabi-readelf -a hello

# ดู symbols
arm-linux-gnueabi-nm hello

# ดู string ใน binary
strings hello

# ตรวจ architecture
file hello
# Output: hello: ELF 32-bit LSB executable, ARM, EABI5 version 1...
```

---

## 15. แบบฝึกหัด (Exercises)

### Exercise 1: ARM32 Basic Syscalls

```asm
@ exercise1_arm32.s
@ จงเขียนโปรแกรมที่:
@ 1. รับ string จาก stdin (sys_read)
@ 2. นับจำนวนตัวอักษร
@ 3. แสดง "Length: N" ออก stdout
@ Template:

.section .data
    prompt:  .ascii "Enter text: "
    plen:    .word  12
    result:  .ascii "Length: "
    rlen:    .word  8

.section .bss
    input:   .space 256

.section .text
.global _start

_start:
    @ TODO: แสดง prompt
    @ TODO: อ่าน input จาก stdin
    @ TODO: นับตัวอักษร (ไม่นับ newline)
    @ TODO: แปลง count เป็น string
    @ TODO: แสดง "Length: N\n"
    @ TODO: exit
```

### Exercise 2: AArch64 File Processing

```asm
// exercise2_aarch64.s
// จงเขียน grep แบบง่าย:
// ค้นหา "ERROR" ใน file และแสดงบรรทัดที่พบ
//
// Usage: ./mygrep filename
//
// Hint:
// - อ่านทีละ byte หรือ block
// - เก็บบรรทัดปัจจุบันไว้ใน buffer
// - เมื่อพบ newline ตรวจสอบว่ามี "ERROR" หรือเปล่า
// - ถ้ามี แสดงบรรทัดนั้น

.section .bss
    .lcomm line_buf, 1024
    .lcomm file_buf, 4096

.section .text
.global _start

_start:
    // TODO: เปิด file จาก argv[1]
    // TODO: วนอ่าน
    // TODO: สะสมบรรทัดใน line_buf
    // TODO: เมื่อพบ newline -> ตรวจหา "ERROR"
    // TODO: ถ้าพบ -> print line
    // TODO: reset line_buf
    // TODO: loop จนจบ file
```

### Exercise 3: Socket Client ARM64

```asm
// exercise3_socket.s
// สร้าง TCP client เชื่อมต่อไปยัง localhost:8080
// ส่ง HTTP GET request และรับ response
//
// HTTP GET request:
// "GET / HTTP/1.0\r\nHost: localhost\r\n\r\n"

.section .data
    host_addr:      // TODO: struct sockaddr_in สำหรับ 127.0.0.1:8080
    http_req: .ascii "GET / HTTP/1.0\r\nHost: localhost\r\n\r\n"
    req_len = . - http_req

.section .bss
    .lcomm resp_buf, 4096

.section .text
.global _start

_start:
    // TODO: socket(AF_INET, SOCK_STREAM, 0)
    // TODO: connect(fd, &addr, sizeof(addr))
    // TODO: write(fd, http_req, req_len)
    // TODO: อ่าน response loop
    // TODO: write response ไปยัง stdout
    // TODO: close(fd)
    // TODO: exit
```

### Exercise 4: mmap File Mapping

```asm
// exercise4_mmap.s
// จงเขียนโปรแกรมที่:
// 1. รับ filename จาก args
// 2. เปิด file
// 3. ดึง file size (fstat)
// 4. mmap ทั้ง file เข้า memory
// 5. ค้นหาตัวอักษร 'A' ในทั้ง file
// 6. แสดงจำนวนครั้งที่พบ
// 7. munmap และ close

// Hint: fstat syscall = 80 (AArch64)
// struct stat: offset 48 = st_size (8 bytes)
// mmap flags: PROT_READ, MAP_PRIVATE

.section .bss
    .lcomm stat_buf, 144    // struct stat ขนาด 144 bytes

.section .text
.global _start

_start:
    // TODO: implement
```

### Exercise 5: Fork และ Pipe

```asm
@ exercise5_pipe.s (ARM32)
@ จงสร้าง parent-child pipeline:
@ parent: รัน "ls -la" แล้วเขียนผ่าน pipe
@ child:  รับจาก pipe แล้วนับบรรทัด (wc -l equivalent)
@
@ Syscalls ที่ใช้:
@ pipe(fd[2])  = syscall 42
@ fork()       = syscall 2
@ dup2(old,new)= syscall 63
@ execve()     = syscall 11

.section .bss
    pipe_fds: .space 8    @ int pipe_fds[2]

.section .text
.global _start

_start:
    @ TODO: เรียก pipe() เพื่อสร้าง pipe
    @ TODO: fork()
    @ parent: ปิด read end, dup2 write end ไปยัง stdout
    @         exec "ls -la"
    @ child:  ปิด write end, dup2 read end ไปยัง stdin
    @         นับบรรทัดและแสดง
```

---

## 16. Tips และ Best Practices

### 16.1 Syscall Safety

```asm
// aarch64_safe_write.s
// ตัวอย่าง safe write ที่รองรับ partial writes

// safe_write: เขียนข้อมูลให้ครบ
// Input:  x0 = fd, x1 = buf, x2 = count
// Output: x0 = 0 (success), -1 (error)
// Clobbers: x3, x4, x5, x8

safe_write:
    mov x3, x1              // current position in buf
    mov x4, x2              // remaining bytes

.sw_loop:
    cbz x4, .sw_done        // ถ้าเหลือ 0 bytes -> จบ

    mov x8, #64             // SYS_WRITE
    mov x0, x0              // fd (unchanged)
    mov x1, x3              // current buf ptr
    mov x2, x4              // remaining count
    svc #0

    // ตรวจ error
    cmn x0, #4096
    b.hi .sw_error

    // อัพเดต position
    add x3, x3, x0          // buf ptr += written
    sub x4, x4, x0          // remaining -= written
    b .sw_loop

.sw_done:
    mov x0, #0
    ret

.sw_error:
    mov x0, #-1
    ret
```

### 16.2 String Utilities ใน ARM

```asm
// aarch64_strutil.s - utility functions

// strlen: คำนวณ string length
// Input:  x0 = string pointer
// Output: x0 = length (ไม่นับ null terminator)
strlen:
    mov x1, x0
.sl_loop:
    ldrb w2, [x1], #1
    cbnz w2, .sl_loop
    sub x0, x1, x0
    sub x0, x0, #1          // ไม่นับ null terminator
    ret

// strcpy: copy string
// Input:  x0 = dest, x1 = src
// Output: x0 = dest
strcpy:
    mov x2, x0
.sc_loop:
    ldrb w3, [x1], #1
    strb w3, [x2], #1
    cbnz w3, .sc_loop
    ret

// strcmp: compare strings
// Input:  x0 = str1, x1 = str2
// Output: x0 = 0 (equal), <0 (str1 < str2), >0 (str1 > str2)
strcmp:
.cmp_loop:
    ldrb w2, [x0], #1
    ldrb w3, [x1], #1
    sub w4, w2, w3
    cbnz w4, .cmp_done      // ต่างกัน -> return difference
    cbnz w2, .cmp_loop      // ยังไม่ถึง null -> วนต่อ
    // ถึง null ทั้งคู่พร้อมกัน = equal
    mov x0, #0
    ret
.cmp_done:
    sxtb x0, w4             // sign-extend result
    ret

// itoa: int to ASCII decimal
// Input:  x0 = number (unsigned), x1 = output buffer
// Output: x0 = length
itoa:
    mov x2, x1              // save start of buffer
    add x3, x1, #20         // end ptr (max 20 digits)
    mov x4, #0              // length

    // กรณี 0
    cbnz x0, .ia_loop
    mov w5, #'0'
    strb w5, [x1]
    mov x0, #1
    ret

.ia_loop:
    cbz x0, .ia_reverse
    mov x5, #10
    udiv x6, x0, x5
    msub x7, x6, x5, x0    // remainder
    add x7, x7, #'0'
    strb w7, [x3, x4]
    add x4, x4, #1
    mov x0, x6
    b .ia_loop

.ia_reverse:
    // reverse the digits into buffer
    mov x5, #0
.ia_rev_loop:
    cmp x5, x4
    b.ge .ia_done
    sub x6, x4, x5
    sub x6, x6, #1
    ldrb w7, [x3, x6]
    strb w7, [x1, x5]
    add x5, x5, #1
    b .ia_rev_loop
.ia_done:
    mov x0, x4
    ret
```

---

## 17. โปรแกรม ls แบบง่าย

### 17.1 ls ใน AArch64

```asm
// aarch64_ls.s - ls command อย่างง่าย
// แสดง entries ใน directory ปัจจุบัน

.equ SYS_OPENAT,     56
.equ SYS_CLOSE,      57
.equ SYS_GETDENTS64, 61
.equ SYS_WRITE,      64
.equ SYS_EXIT_GROUP, 94

.equ AT_FDCWD,       -100
.equ O_RDONLY,       0
.equ O_DIRECTORY,    65536   // O_DIRECTORY = 0200000 octal

// struct linux_dirent64 layout:
// d_ino:    8 bytes (inode)
// d_off:    8 bytes (offset)
// d_reclen: 2 bytes (record length)
// d_type:   1 byte  (file type)
// d_name:   variable (null-terminated)
// Header size: 19 bytes, then name starts at offset 19

.equ DIRENT_RECLEN_OFFSET,  16  // offset ของ d_reclen
.equ DIRENT_TYPE_OFFSET,    18  // offset ของ d_type
.equ DIRENT_NAME_OFFSET,    19  // offset ของ d_name

.equ DT_REG,    8           // regular file
.equ DT_DIR,    4           // directory
.equ DT_LNK,    10          // symbolic link

.section .data
    dot:    .asciz "."          // directory ปัจจุบัน
    newline: .ascii "\n"
    slash:   .ascii "/"         // สำหรับ directory entries

.section .bss
    .lcomm dir_buf, 32768       // buffer สำหรับ getdents64

.section .text
.global _start

_start:
    // เปิด directory "."
    mov x8, #SYS_OPENAT
    mov x0, #AT_FDCWD
    adr x1, dot
    mov x2, #(O_RDONLY | O_DIRECTORY)
    mov x3, #0
    svc #0

    cmn x0, #4096
    b.hi .ls_error
    mov x19, x0                 // x19 = dir_fd

.getdents_loop:
    // getdents64(fd, buf, bufsize)
    mov x8, #SYS_GETDENTS64
    mov x0, x19
    adr x1, dir_buf
    mov x2, #32768
    svc #0

    cbz x0, .ls_done            // ไม่มี entries เหลือ
    cmn x0, #4096
    b.hi .ls_error

    mov x20, #0                 // offset ใน buffer
    mov x21, x0                 // total bytes ใน buffer

.process_entry:
    cmp x20, x21
    b.ge .getdents_loop         // อ่าน batch ถัดไป

    // pointer ไปยัง entry ปัจจุบัน
    adr x22, dir_buf
    add x22, x22, x20           // x22 = current entry

    // ดึง reclen
    ldrh w23, [x22, #DIRENT_RECLEN_OFFSET]  // w23 = reclen

    // ดึง d_type
    ldrb w24, [x22, #DIRENT_TYPE_OFFSET]

    // ดึง d_name (offset 19 จาก entry start)
    add x25, x22, #DIRENT_NAME_OFFSET       // x25 = name ptr

    // ข้าม "." และ ".."
    ldrb w26, [x25]
    cmp w26, #'.'
    b.ne .print_entry

    ldrb w26, [x25, #1]
    cbz w26, .skip_entry        // เป็น "." เดี่ยว -> skip
    cmp w26, #'.'
    b.ne .print_entry
    ldrb w26, [x25, #2]
    cbz w26, .skip_entry        // เป็น ".." -> skip

.print_entry:
    // คำนวณ length ของ name
    mov x26, x25
.name_len_loop:
    ldrb w27, [x26], #1
    cbnz w27, .name_len_loop
    sub x26, x26, x25
    sub x26, x26, #1            // x26 = name_len

    // เขียนชื่อ
    mov x8, #SYS_WRITE
    mov x0, #1                  // stdout
    mov x1, x25                 // name
    mov x2, x26                 // length
    svc #0

    // ถ้าเป็น directory ใส่ "/" ต่อท้าย
    cmp w24, #DT_DIR
    b.ne .print_newline
    mov x8, #SYS_WRITE
    mov x0, #1
    adr x1, slash
    mov x2, #1
    svc #0

.print_newline:
    mov x8, #SYS_WRITE
    mov x0, #1
    adr x1, newline
    mov x2, #1
    svc #0

.skip_entry:
    // ไปยัง entry ถัดไป
    add x20, x20, x23           // offset += reclen
    b .process_entry

.ls_done:
    mov x8, #SYS_CLOSE
    mov x0, x19
    svc #0

    mov x8, #SYS_EXIT_GROUP
    mov x0, #0
    svc #0

.ls_error:
    mov x8, #SYS_EXIT_GROUP
    mov x0, #1
    svc #0
```

---

## 18. สรุป

ในบทนี้เราได้เรียนรู้:

1. **ARM32 Syscall Convention**: R7=syscall number, R0-R6=args, SWI #0
2. **AArch64 Syscall Convention**: X8=syscall number, X0-X5=args, SVC #0
3. **File I/O**: open/openat, read, write, close บนทั้งสอง architectures
4. **Process Management**: fork, exec, waitpid
5. **Memory Management**: mmap, munmap
6. **Signal Handling**: rt_sigaction, kill, pause
7. **Socket Programming**: socket, bind, listen, accept, connect
8. **Complete Programs**: cat, head, wc, ls
9. **Error Handling**: ตรวจสอบ errno จาก negative return values
10. **Syscall Tables**: รายการ syscall สำหรับ ARM32 และ AArch64

### Key Differences: ARM32 vs AArch64

| Feature        | ARM32                | AArch64             |
|----------------|---------------------|---------------------|
| Syscall reg    | R7                  | X8                  |
| Instruction    | SWI #0 หรือ SVC #0  | SVC #0              |
| Arg registers  | R0-R6               | X0-X5               |
| Return value   | R0                  | X0                  |
| open syscall   | sys_open (5)        | sys_openat (56)     |
| exit syscall   | sys_exit (1)        | sys_exit_group (94) |
| mmap syscall   | sys_mmap2 (192)     | sys_mmap (222)      |
| write syscall  | 4                   | 64                  |
| read syscall   | 3                   | 63                  |

---

**ต่อไป: Part 041 - ARM NEON SIMD และ Floating Point**
