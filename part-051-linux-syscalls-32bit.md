# Part 051: Linux System Calls (32-bit x86)

## บทนำ: การสื่อสารระหว่าง User Space และ Kernel

ใน Linux โปรแกรมในระดับ user space ไม่สามารถเข้าถึงฮาร์ดแวร์หรือทรัพยากรระบบโดยตรงได้
ต้องผ่านกระบวนการที่เรียกว่า **System Call** (syscall) ซึ่งเป็นช่องทางอย่างเป็นทางการ
ในการขอบริการจาก kernel

```
User Space Program
       |
       | (int 0x80)
       v
  Linux Kernel
       |
       v
  Hardware / Resources
```

---

## 1. กลไก int 0x80

### 1.1 ความหมายของ int 0x80

`int 0x80` คือคำสั่ง **software interrupt** ที่ใช้กระตุ้นให้ CPU เปลี่ยนโหมดจาก
user mode ไปเป็น kernel mode เพื่อประมวลผล system call

- `int` = interrupt instruction
- `0x80` = interrupt vector number 128 (decimal)
- Linux kernel ลงทะเบียน handler ไว้ที่ vector นี้โดยเฉพาะ

### 1.2 ขั้นตอนการทำงาน

```
1. โปรแกรมวาง syscall number ลงใน EAX
2. วาง arguments ลงใน EBX, ECX, EDX, ESI, EDI, EBP
3. เรียก int 0x80
4. CPU สวิตช์จาก ring 3 (user) ไป ring 0 (kernel)
5. Kernel อ่าน EAX เพื่อรู้ว่า syscall ใด
6. Kernel ดำเนินการ
7. Kernel วาง return value ลงใน EAX
8. CPU กลับมา ring 3
9. โปรแกรมอ่าน return value จาก EAX
```

### 1.3 การเปรียบเทียบกับ C

```c
// C code
ssize_t n = write(1, "Hello\n", 6);

// Assembly equivalent
mov eax, 4      ; syscall number ของ write
mov ebx, 1      ; fd = 1 (stdout)
mov ecx, msg    ; buffer address
mov edx, 6      ; length
int 0x80        ; เรียก kernel
; ผลลัพธ์อยู่ใน EAX
```

---

## 2. Registers สำหรับ System Calls

### 2.1 EAX — Syscall Number

EAX ต้องมี **syscall number** ก่อนเรียก `int 0x80`

```nasm
mov eax, 1    ; sys_exit
mov eax, 4    ; sys_write
mov eax, 3    ; sys_read
```

### 2.2 EBX — Argument 1 (arg1)

```nasm
; sys_write(fd, buf, count)
mov ebx, 1    ; fd = stdout
```

### 2.3 ECX — Argument 2 (arg2)

```nasm
mov ecx, buffer    ; ที่อยู่ของบัฟเฟอร์
```

### 2.4 EDX — Argument 3 (arg3)

```nasm
mov edx, 13    ; จำนวน bytes
```

### 2.5 ESI — Argument 4 (arg4)

```nasm
mov esi, flags    ; สำหรับ mmap หรือ syscall ที่มี 4 arguments
```

### 2.6 EDI — Argument 5 (arg5)

```nasm
mov edi, fd    ; สำหรับ mmap
```

### 2.7 EBP — Argument 6 (arg6)

```nasm
mov ebp, offset    ; สำหรับ mmap
```

### 2.8 ตารางสรุป Registers

| Register | บทบาท          | ตัวอย่าง              |
|----------|----------------|----------------------|
| EAX      | syscall number | `mov eax, 4`         |
| EBX      | argument 1     | `mov ebx, 1` (fd)    |
| ECX      | argument 2     | `mov ecx, buf`       |
| EDX      | argument 3     | `mov edx, len`       |
| ESI      | argument 4     | `mov esi, flags`     |
| EDI      | argument 5     | `mov edi, fd`        |
| EBP      | argument 6     | `mov ebp, offset`    |
| EAX      | return value   | อ่านหลัง int 0x80    |

---

## 3. Return Value และ errno

### 3.1 Return Value ใน EAX

หลัง `int 0x80` ค่า EAX จะเก็บผลลัพธ์:
- ถ้า `>= 0` = สำเร็จ (ค่าที่ return)
- ถ้า `< 0` = เกิดข้อผิดพลาด (negative errno)

### 3.2 ค่า errno ที่พบบ่อย

| ค่า  | ชื่อ         | ความหมาย                       |
|------|-------------|-------------------------------|
| -1   | EPERM       | Operation not permitted        |
| -2   | ENOENT      | No such file or directory      |
| -3   | ESRCH       | No such process                |
| -4   | EINTR       | Interrupted system call        |
| -5   | EIO         | I/O error                      |
| -9   | EBADF       | Bad file number                |
| -11  | EAGAIN      | Try again                      |
| -12  | ENOMEM      | Out of memory                  |
| -13  | EACCES      | Permission denied              |
| -17  | EEXIST      | File exists                    |
| -20  | ENOTDIR     | Not a directory                |
| -22  | EINVAL      | Invalid argument               |

### 3.3 การตรวจสอบ Error

```nasm
; หลัง syscall ตรวจสอบ EAX
int 0x80

; วิธีที่ 1: ตรวจสอบ sign bit
test eax, eax
js  .error          ; jump ถ้า EAX < 0 (negative = error)
jns .success        ; jump ถ้า EAX >= 0

; วิธีที่ 2: เปรียบเทียบกับ -1
cmp eax, -1
je  .error

; วิธีที่ 3: ตรวจสอบช่วง error (Linux: -4095 ถึง -1)
cmp eax, -4095
jbe .success        ; ถ้า EAX > -4095 แสดงว่าสำเร็จ
neg eax             ; แปลงเป็น positive error code
```

---

## 4. Syscall Table: /usr/include/asm/unistd_32.h

ไฟล์นี้เป็นตำแหน่งที่เก็บ syscall numbers สำหรับ 32-bit x86

```bash
# ดู syscall numbers
cat /usr/include/asm/unistd_32.h | head -50
# หรือ
grep "__NR_write" /usr/include/asm/unistd_32.h
```

### 4.1 Syscall Numbers ที่สำคัญ

```nasm
; ===== File Operations =====
%define SYS_read        3
%define SYS_write       4
%define SYS_open        5
%define SYS_close       6
%define SYS_stat        106    ; stat (ใหม่)
%define SYS_fstat       108
%define SYS_lstat       107
%define SYS_lseek       19
%define SYS_ioctl       54
%define SYS_fcntl       55
%define SYS_unlink      10
%define SYS_rename      38
%define SYS_mkdir       39
%define SYS_rmdir       40
%define SYS_creat       8
%define SYS_link        9
%define SYS_symlink     83
%define SYS_chmod       15
%define SYS_getdents    141

; ===== Process =====
%define SYS_exit        1
%define SYS_fork        2
%define SYS_execve      11
%define SYS_waitpid     7
%define SYS_getpid      20
%define SYS_getppid     64
%define SYS_kill        37
%define SYS_signal      48

; ===== Memory =====
%define SYS_brk         45
%define SYS_mmap        90
%define SYS_munmap      91
%define SYS_mprotect    125

; ===== Networking =====
%define SYS_socketcall  102    ; บน x86 32-bit ใช้ socketcall
; หรือใช้ syscall numbers โดยตรง (kernel ใหม่)
%define SYS_socket      359
%define SYS_bind        361
%define SYS_connect     362
%define SYS_listen      363
%define SYS_accept      364

; ===== Time =====
%define SYS_time        13
%define SYS_gettimeofday 78

; ===== I/O =====
%define SYS_dup         41
%define SYS_dup2        63
%define SYS_pipe        42
%define SYS_select      82
```

---

## 5. รายละเอียด Syscalls สำคัญ

### 5.1 exit(1) — จบโปรแกรม

```
Prototype: void _exit(int status)
EAX = 1
EBX = exit status (0 = success, non-zero = error)
Return: ไม่มี (โปรแกรมจบ)
```

```nasm
; ออกจากโปรแกรมด้วย status 0
mov eax, 1      ; SYS_exit
mov ebx, 0      ; exit status = 0 (success)
int 0x80
```

### 5.2 fork(2) — สร้าง child process

```
Prototype: pid_t fork(void)
EAX = 2
Return: ใน parent = PID ของ child, ใน child = 0, error = -1
```

```nasm
mov eax, 2      ; SYS_fork
int 0x80
; ตอนนี้ทั้ง parent และ child รันโค้ดต่อจากนี้
; parent ได้ child PID ใน EAX
; child ได้ 0 ใน EAX
test eax, eax
jz  .child_code     ; ถ้า EAX = 0 คือ child
; parent code...
```

### 5.3 read(3) — อ่านข้อมูล

```
Prototype: ssize_t read(int fd, void *buf, size_t count)
EAX = 3
EBX = file descriptor
ECX = buffer address
EDX = max bytes to read
Return: จำนวน bytes ที่อ่านได้, 0 = EOF, -1 = error
```

```nasm
; อ่านจาก stdin (fd=0)
mov eax, 3          ; SYS_read
mov ebx, 0          ; stdin
mov ecx, buffer     ; ที่เก็บข้อมูล
mov edx, 1024       ; จำนวน bytes สูงสุด
int 0x80
; EAX = จำนวน bytes ที่อ่านได้จริง
```

### 5.4 write(4) — เขียนข้อมูล

```
Prototype: ssize_t write(int fd, const void *buf, size_t count)
EAX = 4
EBX = file descriptor
ECX = buffer address
EDX = bytes to write
Return: จำนวน bytes ที่เขียน, -1 = error
```

```nasm
; เขียนไปยัง stdout (fd=1)
mov eax, 4          ; SYS_write
mov ebx, 1          ; stdout
mov ecx, msg        ; ข้อความ
mov edx, msg_len    ; ความยาว
int 0x80
```

### 5.5 open(5) — เปิดไฟล์

```
Prototype: int open(const char *pathname, int flags, mode_t mode)
EAX = 5
EBX = pathname (null-terminated string)
ECX = flags
EDX = mode (สำหรับ O_CREAT)
Return: file descriptor (>= 0), -1 = error
```

**Flags สำคัญ:**

```nasm
%define O_RDONLY    0       ; อ่านอย่างเดียว
%define O_WRONLY    1       ; เขียนอย่างเดียว
%define O_RDWR      2       ; อ่านและเขียน
%define O_CREAT     64      ; สร้างไฟล์ถ้าไม่มี (0x40)
%define O_TRUNC     512     ; ตัดไฟล์ให้ขนาด 0 (0x200)
%define O_APPEND    1024    ; เพิ่มต่อท้าย (0x400)
%define O_NONBLOCK  2048    ; non-blocking mode
```

```nasm
; เปิดไฟล์เพื่ออ่าน
mov eax, 5              ; SYS_open
mov ebx, filename       ; ชื่อไฟล์
mov ecx, 0              ; O_RDONLY
mov edx, 0              ; mode (ไม่ใช้เพราะไม่ได้ O_CREAT)
int 0x80
; EAX = file descriptor หรือ -1
```

### 5.6 close(6) — ปิดไฟล์

```
Prototype: int close(int fd)
EAX = 6
EBX = file descriptor
Return: 0 = success, -1 = error
```

```nasm
mov eax, 6      ; SYS_close
mov ebx, [fd]   ; file descriptor
int 0x80
```

### 5.7 execve(11) — รันโปรแกรมอื่น

```
Prototype: int execve(const char *filename, char *const argv[], char *const envp[])
EAX = 11
EBX = ที่อยู่ของ string ชื่อโปรแกรม
ECX = ที่อยู่ของ array of argument strings (null-terminated)
EDX = ที่อยู่ของ array of environment strings (null-terminated)
Return: ไม่ return ถ้าสำเร็จ, -1 = error
```

```nasm
; รัน /bin/sh
mov eax, 11             ; SYS_execve
mov ebx, prog_name      ; "/bin/sh"
mov ecx, argv_array     ; ["/bin/sh", NULL]
mov edx, envp_array     ; [NULL]
int 0x80
```

### 5.8 waitpid(7) — รอ child process

```
Prototype: pid_t waitpid(pid_t pid, int *status, int options)
EAX = 7
EBX = pid (-1 = รอ child ใดก็ได้)
ECX = ที่อยู่เก็บ status
EDX = options (0 = block จนกว่า child จะจบ)
Return: PID ของ child ที่จบ, -1 = error
```

```nasm
mov eax, 7          ; SYS_waitpid
mov ebx, -1         ; รอ child ใดก็ได้
mov ecx, child_status  ; เก็บ exit status
mov edx, 0          ; options = 0 (blocking)
int 0x80
```

### 5.9 getpid(20) — ได้ PID ของตัวเอง

```
Prototype: pid_t getpid(void)
EAX = 20
Return: PID ของ process ปัจจุบัน
```

```nasm
mov eax, 20     ; SYS_getpid
int 0x80
; EAX = PID
```

### 5.10 kill(37) — ส่ง signal

```
Prototype: int kill(pid_t pid, int sig)
EAX = 37
EBX = target PID
ECX = signal number
Return: 0 = success, -1 = error
```

**Signals สำคัญ:**

```nasm
%define SIGHUP   1      ; Hangup
%define SIGINT   2      ; Interrupt (Ctrl+C)
%define SIGQUIT  3      ; Quit
%define SIGKILL  9      ; Kill (ไม่สามารถ catch ได้)
%define SIGTERM  15     ; Terminate
%define SIGSTOP  19     ; Stop
%define SIGCONT  18     ; Continue
```

```nasm
; ส่ง SIGTERM ไปยัง PID 1234
mov eax, 37     ; SYS_kill
mov ebx, 1234   ; target PID
mov ecx, 15     ; SIGTERM
int 0x80
```

### 5.11 brk(45) — จัดการ heap

```
Prototype: int brk(void *addr)
EAX = 45
EBX = new end of data segment (0 = query current)
Return: new end of data segment
```

```nasm
; query current brk
mov eax, 45     ; SYS_brk
mov ebx, 0      ; 0 = query
int 0x80
; EAX = current end of heap
mov [heap_end], eax

; ขยาย heap ขึ้น 4096 bytes
add eax, 4096
mov ebx, eax
mov eax, 45
int 0x80
; EAX = new end (ถ้าเท่ากับที่ขอ = สำเร็จ)
```

### 5.12 mmap(90) — map memory

```
Prototype: void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)
EAX = 90
EBX = ที่อยู่ของ struct mmap_arg_struct
```

**โครงสร้าง mmap_arg_struct (32-bit):**

```nasm
; struct mmap_arg_struct {
;     unsigned long addr;    ; ที่อยู่ที่ต้องการ (0 = kernel เลือก)
;     unsigned long len;     ; ขนาด
;     unsigned long prot;    ; PROT_READ|PROT_WRITE
;     unsigned long flags;   ; MAP_PRIVATE|MAP_ANONYMOUS
;     unsigned long fd;      ; file descriptor (-1 สำหรับ anonymous)
;     unsigned long offset;  ; offset ใน file
; }

%define PROT_READ     1
%define PROT_WRITE    2
%define PROT_EXEC     4
%define MAP_SHARED    1
%define MAP_PRIVATE   2
%define MAP_ANONYMOUS 0x20

section .data
mmap_args:
    dd 0            ; addr = 0 (kernel เลือก)
    dd 4096         ; len = 4096
    dd 3            ; prot = PROT_READ|PROT_WRITE
    dd 0x22         ; flags = MAP_PRIVATE|MAP_ANONYMOUS
    dd -1           ; fd = -1
    dd 0            ; offset = 0

section .text
    mov eax, 90             ; SYS_mmap
    mov ebx, mmap_args      ; pointer to struct
    int 0x80
    ; EAX = address ของ mapped memory หรือ -1
```

### 5.13 munmap(91) — unmap memory

```
Prototype: int munmap(void *addr, size_t length)
EAX = 91
EBX = address
ECX = length
Return: 0 = success, -1 = error
```

```nasm
mov eax, 91         ; SYS_munmap
mov ebx, [mapped_addr]  ; address ที่ได้จาก mmap
mov ecx, 4096       ; ขนาดที่จะ unmap
int 0x80
```

### 5.14 stat(106) — ข้อมูลไฟล์

```
Prototype: int stat(const char *pathname, struct stat *statbuf)
EAX = 106
EBX = pathname
ECX = pointer to stat struct
Return: 0 = success, -1 = error
```

**โครงสร้าง stat (32-bit):**

```nasm
; struct stat {
;     dev_t  st_dev;         ; +0  (8 bytes)
;     ino_t  st_ino;         ; +8  (4 bytes)
;     mode_t st_mode;        ; +12 (4 bytes)
;     nlink_t st_nlink;      ; +16 (4 bytes)
;     uid_t  st_uid;         ; +20 (4 bytes)
;     gid_t  st_gid;         ; +24 (4 bytes)
;     dev_t  st_rdev;        ; +28 (8 bytes)
;     off_t  st_size;        ; +36 (4 bytes ใน 32-bit)
;     ...
; }

section .bss
stat_buf: resb 144      ; ขนาดของ stat struct

section .text
    mov eax, 106            ; SYS_stat
    mov ebx, filename       ; ชื่อไฟล์
    mov ecx, stat_buf       ; buffer
    int 0x80
    ; อ่านขนาดไฟล์
    mov eax, [stat_buf + 36]  ; st_size
```

### 5.15 getdents(141) — อ่านรายการใน directory

```
Prototype: int getdents(unsigned int fd, struct linux_dirent *dirp, unsigned int count)
EAX = 141
EBX = file descriptor ของ directory
ECX = buffer
EDX = buffer size
Return: จำนวน bytes ที่อ่าน, 0 = end, -1 = error
```

### 5.16 socket(359), connect(362), bind(361), listen(363), accept(364)

บน Linux x86 32-bit มีสองวิธีใช้ network syscalls:

**วิธีที่ 1: ใช้ socketcall (102) — วิธีเดิม**

```nasm
; SYS_socketcall ใช้ sub-call number ใน EBX
%define SYS_SOCKET   1
%define SYS_BIND     2
%define SYS_CONNECT  3
%define SYS_LISTEN   4
%define SYS_ACCEPT   5

; สร้าง socket
section .data
socket_args:
    dd 2    ; AF_INET
    dd 1    ; SOCK_STREAM
    dd 0    ; protocol

section .text
    mov eax, 102        ; SYS_socketcall
    mov ebx, 1          ; SYS_SOCKET
    mov ecx, socket_args
    int 0x80
    ; EAX = socket fd
```

**วิธีที่ 2: ใช้ syscall numbers โดยตรง (kernel >= 4.3)**

```nasm
%define AF_INET      2
%define SOCK_STREAM  1

; สร้าง TCP socket
mov eax, 359        ; SYS_socket
mov ebx, AF_INET    ; domain
mov ecx, SOCK_STREAM ; type
mov edx, 0          ; protocol (TCP)
int 0x80
mov [sockfd], eax   ; เก็บ socket fd
```

---

## 6. โปรแกรมตัวอย่างสมบูรณ์

### 6.1 Hello World

```nasm
; hello.asm - Hello World แบบ 32-bit Linux
; คอมไพล์: nasm -f elf32 hello.asm -o hello.o
; ลิงก์:   ld -m elf_i386 hello.o -o hello
; รัน:     ./hello

section .data
    ; ข้อความที่จะแสดง
    msg db "Hello, World!", 0x0A    ; 0x0A = newline
    msg_len equ $ - msg             ; คำนวณความยาวอัตโนมัติ

section .text
    global _start

_start:
    ; ===== เขียนข้อความออก stdout =====
    mov eax, 4          ; syscall: sys_write
    mov ebx, 1          ; fd: stdout (file descriptor 1)
    mov ecx, msg        ; buffer: ที่อยู่ของข้อความ
    mov edx, msg_len    ; count: จำนวน bytes
    int 0x80            ; เรียก kernel

    ; ===== จบโปรแกรมด้วย exit status 0 =====
    mov eax, 1          ; syscall: sys_exit
    mov ebx, 0          ; status: 0 (สำเร็จ)
    int 0x80            ; เรียก kernel
```

### 6.2 Cat File (อ่านและแสดงเนื้อหาไฟล์)

```nasm
; cat.asm - อ่านเนื้อหาไฟล์และแสดงผล
; ใช้: ./cat filename
; คอมไพล์: nasm -f elf32 cat.asm -o cat.o && ld -m elf_i386 cat.o -o mycat

%define SYS_READ    3
%define SYS_WRITE   4
%define SYS_OPEN    5
%define SYS_CLOSE   6
%define SYS_EXIT    1
%define O_RDONLY    0
%define STDIN       0
%define STDOUT      1
%define STDERR      2
%define BUF_SIZE    4096

section .bss
    buffer  resb BUF_SIZE   ; buffer สำหรับอ่านไฟล์
    fd      resd 1          ; เก็บ file descriptor

section .text
    global _start

_start:
    ; ===== ตรวจสอบ arguments =====
    ; [esp] = argc, [esp+4] = argv[0], [esp+8] = argv[1]
    mov eax, [esp]          ; อ่าน argc
    cmp eax, 2              ; ต้องมี 2 arguments (ชื่อโปรแกรม + ชื่อไฟล์)
    jl  .no_args            ; ถ้าน้อยกว่า 2 ไม่มี argument

    ; ===== เปิดไฟล์ =====
    mov eax, SYS_OPEN       ; syscall: sys_open
    mov ebx, [esp+8]        ; argv[1] = ชื่อไฟล์
    mov ecx, O_RDONLY       ; flags: อ่านอย่างเดียว
    mov edx, 0              ; mode: ไม่ใช้
    int 0x80

    ; ตรวจสอบว่าเปิดได้
    test eax, eax
    js   .open_error        ; ถ้า EAX < 0 เกิดข้อผิดพลาด
    mov  [fd], eax          ; เก็บ file descriptor

.read_loop:
    ; ===== อ่านข้อมูลจากไฟล์ =====
    mov eax, SYS_READ       ; syscall: sys_read
    mov ebx, [fd]           ; fd: file descriptor ที่เปิดไว้
    mov ecx, buffer         ; buf: ที่เก็บข้อมูล
    mov edx, BUF_SIZE       ; count: จำนวน bytes สูงสุด
    int 0x80

    ; ตรวจสอบ EOF หรือ error
    test eax, eax
    jz   .done              ; ถ้า EAX = 0 ถึง EOF แล้ว
    js   .read_error        ; ถ้า EAX < 0 เกิดข้อผิดพลาด

    ; ===== เขียนไปยัง stdout =====
    mov edx, eax            ; count = จำนวน bytes ที่อ่านได้
    mov eax, SYS_WRITE      ; syscall: sys_write
    mov ebx, STDOUT         ; fd: stdout
    mov ecx, buffer         ; buf: ข้อมูลที่อ่านมา
    int 0x80

    jmp .read_loop          ; วนลูปอ่านต่อ

.done:
    ; ===== ปิดไฟล์ =====
    mov eax, SYS_CLOSE      ; syscall: sys_close
    mov ebx, [fd]           ; fd: ปิด file descriptor
    int 0x80

    ; จบโปรแกรม สำเร็จ
    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.no_args:
    ; แสดงข้อความ usage
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, usage_msg
    mov edx, usage_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

.open_error:
    ; แสดง error เปิดไฟล์ไม่ได้
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_open
    mov edx, err_open_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

.read_error:
    ; ปิดไฟล์แล้วออก
    mov eax, SYS_CLOSE
    mov ebx, [fd]
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

section .data
    usage_msg   db "Usage: mycat <filename>", 0x0A
    usage_len   equ $ - usage_msg
    err_open    db "Error: Cannot open file", 0x0A
    err_open_len equ $ - err_open
```

### 6.3 CP (Copy File)

```nasm
; cp.asm - คัดลอกไฟล์
; ใช้: ./mycp source destination
; คอมไพล์: nasm -f elf32 cp.asm -o cp.o && ld -m elf_i386 cp.o -o mycp

%define SYS_READ    3
%define SYS_WRITE   4
%define SYS_OPEN    5
%define SYS_CLOSE   6
%define SYS_EXIT    1
%define O_RDONLY    0
%define O_WRONLY    1
%define O_CREAT     64      ; 0x40
%define O_TRUNC     512     ; 0x200
%define MODE_644    420     ; 0644 octal = 420 decimal
%define BUF_SIZE    8192
%define STDOUT      1
%define STDERR      2

section .bss
    buffer  resb BUF_SIZE
    src_fd  resd 1          ; file descriptor ไฟล์ต้นฉบับ
    dst_fd  resd 1          ; file descriptor ไฟล์ปลายทาง

section .text
    global _start

_start:
    ; ตรวจสอบว่ามี 2 arguments
    mov eax, [esp]
    cmp eax, 3              ; ต้องมี 3 (prog, src, dst)
    jne .usage_error

    ; ===== เปิดไฟล์ต้นฉบับ =====
    mov eax, SYS_OPEN
    mov ebx, [esp+8]        ; argv[1] = source
    mov ecx, O_RDONLY
    mov edx, 0
    int 0x80
    test eax, eax
    js   .src_error
    mov  [src_fd], eax

    ; ===== เปิด/สร้างไฟล์ปลายทาง =====
    mov eax, SYS_OPEN
    mov ebx, [esp+12]       ; argv[2] = destination
    mov ecx, O_WRONLY | O_CREAT | O_TRUNC  ; 577
    mov edx, MODE_644       ; permissions: rw-r--r--
    int 0x80
    test eax, eax
    js   .dst_error
    mov  [dst_fd], eax

.copy_loop:
    ; ===== อ่านจากต้นฉบับ =====
    mov eax, SYS_READ
    mov ebx, [src_fd]
    mov ecx, buffer
    mov edx, BUF_SIZE
    int 0x80

    test eax, eax
    jz   .copy_done         ; EOF
    js   .read_err

    ; ===== เขียนไปปลายทาง =====
    ; ต้องเขียนทั้งหมดที่อ่านมา (loop สำหรับ partial write)
    push eax                ; เก็บจำนวน bytes ที่อ่าน
    mov  esi, buffer        ; pointer เริ่มต้น
    mov  edi, eax           ; จำนวน bytes ที่ต้องเขียน

.write_loop:
    mov  eax, SYS_WRITE
    mov  ebx, [dst_fd]
    mov  ecx, esi           ; current position
    mov  edx, edi           ; bytes remaining
    int  0x80

    test eax, eax
    js   .write_err
    jz   .write_err

    sub  edi, eax           ; ลดจำนวนที่ยังต้องเขียน
    add  esi, eax           ; เลื่อน pointer
    jnz  .write_loop        ; ถ้ายังเหลือให้เขียนต่อ

    pop  eax                ; restore
    jmp  .copy_loop

.copy_done:
    ; ===== ปิดทั้งสองไฟล์ =====
    mov eax, SYS_CLOSE
    mov ebx, [src_fd]
    int 0x80

    mov eax, SYS_CLOSE
    mov ebx, [dst_fd]
    int 0x80

    ; สำเร็จ
    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.usage_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, usage_msg
    mov edx, usage_len
    int 0x80
    jmp .exit_fail

.src_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_src
    mov edx, err_src_len
    int 0x80
    jmp .exit_fail

.dst_error:
    mov eax, SYS_CLOSE
    mov ebx, [src_fd]
    int 0x80
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_dst
    mov edx, err_dst_len
    int 0x80
    jmp .exit_fail

.read_err:
.write_err:
    mov eax, SYS_CLOSE
    mov ebx, [src_fd]
    int 0x80
    mov eax, SYS_CLOSE
    mov ebx, [dst_fd]
    int 0x80

.exit_fail:
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

section .data
    usage_msg   db "Usage: mycp <source> <destination>", 0x0A
    usage_len   equ $ - usage_msg
    err_src     db "Error: Cannot open source file", 0x0A
    err_src_len equ $ - err_src
    err_dst     db "Error: Cannot open destination file", 0x0A
    err_dst_len equ $ - err_dst
```

### 6.4 WC -l (นับจำนวนบรรทัด)

```nasm
; wc_l.asm - นับจำนวนบรรทัดในไฟล์ (เหมือน wc -l)
; ใช้: ./mywc filename
; คอมไพล์: nasm -f elf32 wc_l.asm -o wc_l.o && ld -m elf_i386 wc_l.o -o mywc

%define SYS_READ    3
%define SYS_WRITE   4
%define SYS_OPEN    5
%define SYS_CLOSE   6
%define SYS_EXIT    1
%define O_RDONLY    0
%define STDOUT      1
%define STDERR      2
%define BUF_SIZE    4096
%define NEWLINE     0x0A

section .bss
    buffer      resb BUF_SIZE
    fd          resd 1
    line_count  resd 1      ; เก็บจำนวนบรรทัด
    num_buf     resb 20     ; buffer สำหรับแปลงตัวเลข

section .text
    global _start

_start:
    ; ตรวจสอบ arguments
    mov eax, [esp]
    cmp eax, 2
    jl  .usage_error

    ; เริ่มนับที่ 0
    mov dword [line_count], 0

    ; เปิดไฟล์
    mov eax, SYS_OPEN
    mov ebx, [esp+8]
    mov ecx, O_RDONLY
    mov edx, 0
    int 0x80
    test eax, eax
    js   .open_error
    mov  [fd], eax

.read_loop:
    ; อ่านข้อมูล
    mov eax, SYS_READ
    mov ebx, [fd]
    mov ecx, buffer
    mov edx, BUF_SIZE
    int 0x80

    test eax, eax
    jz   .count_done    ; EOF
    js   .read_error

    ; นับ newline characters
    mov  esi, buffer    ; pointer เริ่มต้น
    mov  ecx, eax       ; จำนวน bytes ที่อ่านได้

.scan_loop:
    mov  al, [esi]      ; อ่าน 1 byte
    cmp  al, NEWLINE    ; เปรียบเทียบกับ newline
    jne  .not_newline
    inc  dword [line_count]  ; นับ

.not_newline:
    inc  esi            ; เลื่อนไปข้างหน้า
    loop .scan_loop     ; วนซ้ำ ECX ครั้ง

    jmp  .read_loop

.count_done:
    ; ปิดไฟล์
    mov eax, SYS_CLOSE
    mov ebx, [fd]
    int 0x80

    ; แปลงจำนวนเป็น string และแสดงผล
    mov eax, [line_count]
    call itoa           ; แปลง EAX เป็น string ใน num_buf

    ; แสดงจำนวน
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, num_buf
    mov edx, [num_buf_len]
    int 0x80

    ; แสดง newline
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, newline
    mov edx, 1
    int 0x80

    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

; ===== Function: itoa =====
; Input:  EAX = number
; Output: ผล string ใน num_buf, ความยาวใน [num_buf_len]
itoa:
    push ebx
    push ecx
    push edx
    push edi

    mov  edi, num_buf + 19  ; เริ่มจากท้าย buffer
    mov  byte [edi], 0      ; null terminator
    dec  edi

    ; กรณีพิเศษ: เลข 0
    test eax, eax
    jnz  .convert
    mov  byte [edi], '0'
    mov  dword [num_buf_len], 1
    ; คัดลอกไปต้น buffer
    jmp  .done_itoa

.convert:
    xor  ecx, ecx           ; นับจำนวน digits

.digit_loop:
    test eax, eax
    jz   .build_string

    xor  edx, edx           ; เคลียร์ EDX ก่อน div
    mov  ebx, 10
    div  ebx                ; EAX = quotient, EDX = remainder
    add  dl, '0'            ; แปลงเป็น ASCII
    dec  edi
    mov  [edi], dl          ; วางที่ท้าย
    inc  ecx
    jmp  .digit_loop

.build_string:
    ; คัดลอก string ไปต้น num_buf
    push esi
    mov  esi, edi
    mov  edi, num_buf
    mov  [num_buf_len], ecx
.copy_loop_itoa:
    mov  al, [esi]
    mov  [edi], al
    inc  esi
    inc  edi
    loop .copy_loop_itoa
    pop  esi

.done_itoa:
    pop  edi
    pop  edx
    pop  ecx
    pop  ebx
    ret

.usage_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, usage_msg
    mov edx, usage_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

.open_error:
.read_error:
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

section .data
    newline         db 0x0A
    usage_msg       db "Usage: mywc <filename>", 0x0A
    usage_len       equ $ - usage_msg

section .bss
    num_buf_len     resd 1
```

### 6.5 Simple LS (แสดงรายการไฟล์)

```nasm
; ls.asm - แสดงรายชื่อไฟล์ใน directory
; ใช้: ./myls [directory]
; คอมไพล์: nasm -f elf32 ls.asm -o ls.o && ld -m elf_i386 ls.o -o myls

%define SYS_READ        3
%define SYS_WRITE       4
%define SYS_OPEN        5
%define SYS_CLOSE       6
%define SYS_EXIT        1
%define SYS_GETDENTS    141
%define O_RDONLY        0
%define O_DIRECTORY     65536   ; 0x10000
%define STDOUT          1
%define STDERR          2
%define BUF_SIZE        4096

; โครงสร้าง linux_dirent
; struct linux_dirent {
;   ino_t   d_ino;      +0  (4 bytes บน 32-bit)
;   off_t   d_off;      +4  (4 bytes)
;   ushort  d_reclen;   +8  (2 bytes)
;   char    d_name[];   +10 (ชื่อไฟล์)
; }

section .bss
    dir_buf resb BUF_SIZE   ; buffer สำหรับ directory entries
    fd      resd 1

section .text
    global _start

_start:
    ; ตรวจสอบ argument
    mov eax, [esp]
    cmp eax, 2
    jge .open_arg           ; ถ้ามี argument ใช้มัน

    ; ไม่มี argument ใช้ "."
    mov ebx, dot_dir
    jmp .do_open

.open_arg:
    mov ebx, [esp+8]        ; argv[1]

.do_open:
    ; เปิด directory
    mov eax, SYS_OPEN
    ; ebx ยังคงเป็น path
    mov ecx, O_RDONLY | O_DIRECTORY  ; 65536
    mov edx, 0
    int 0x80
    test eax, eax
    js   .open_error
    mov  [fd], eax

.read_dir:
    ; อ่าน directory entries
    mov eax, SYS_GETDENTS
    mov ebx, [fd]
    mov ecx, dir_buf
    mov edx, BUF_SIZE
    int 0x80

    test eax, eax
    jz   .done              ; ไม่มีรายการเพิ่ม
    js   .read_error

    ; วนผ่าน entries
    mov  esi, dir_buf       ; pointer เริ่มต้น
    mov  edi, eax           ; จำนวน bytes ทั้งหมด

.process_entry:
    ; แสดงชื่อไฟล์ (อยู่ที่ offset +10)
    mov  eax, esi
    add  eax, 10            ; d_name offset
    push eax

    ; คำนวณความยาวชื่อ (strlen)
    mov  ebx, eax
.strlen_loop:
    mov  cl, [ebx]
    test cl, cl
    jz   .strlen_done
    inc  ebx
    jmp  .strlen_loop
.strlen_done:
    sub  ebx, eax           ; ความยาว = pointer ปัจจุบัน - ต้นชื่อ
    pop  eax

    ; เขียนชื่อไฟล์
    push ebx
    mov  edx, ebx           ; length
    mov  ecx, eax           ; buffer = d_name
    mov  ebx, STDOUT
    mov  eax, SYS_WRITE
    int  0x80
    pop  edx                ; restore (ไม่ใช้แล้ว)

    ; เขียน newline
    mov  eax, SYS_WRITE
    mov  ebx, STDOUT
    mov  ecx, newline
    mov  edx, 1
    int  0x80

    ; เลื่อนไปยัง entry ถัดไป
    movzx eax, word [esi+8]  ; d_reclen (2 bytes)
    add   esi, eax
    sub   edi, eax
    jg    .process_entry     ; ถ้ายังมีรายการ

    jmp   .read_dir

.done:
    ; ปิด directory
    mov eax, SYS_CLOSE
    mov ebx, [fd]
    int 0x80

    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.open_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_msg
    mov edx, err_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

.read_error:
    mov eax, SYS_CLOSE
    mov ebx, [fd]
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

section .data
    dot_dir  db ".", 0
    newline  db 0x0A
    err_msg  db "Error: Cannot open directory", 0x0A
    err_len  equ $ - err_msg
```

### 6.6 chmod (เปลี่ยน permissions)

```nasm
; chmod.asm - เปลี่ยน file permissions
; ใช้: ./mychmod <mode> <file>
; ตัวอย่าง: ./mychmod 755 myfile.txt
; คอมไพล์: nasm -f elf32 chmod.asm -o chmod.o && ld -m elf_i386 chmod.o -o mychmod

%define SYS_CHMOD   15
%define SYS_WRITE   4
%define SYS_EXIT    1
%define STDOUT      1
%define STDERR      2

section .bss
    mode_val    resd 1      ; เก็บค่า mode ที่แปลงแล้ว

section .text
    global _start

_start:
    ; ตรวจสอบ arguments: ต้องมี mode และ filename
    mov eax, [esp]
    cmp eax, 3
    jne .usage_error

    ; ===== แปลง mode string เป็น octal number =====
    ; argv[1] = mode string (เช่น "755")
    mov  esi, [esp+8]       ; argv[1] = mode string
    xor  eax, eax           ; ผลลัพธ์
    xor  ecx, ecx           ; temporary

.parse_octal:
    mov  cl, [esi]
    test cl, cl
    jz   .parse_done
    cmp  cl, '0'
    jl   .bad_mode
    cmp  cl, '7'
    jg   .bad_mode
    sub  cl, '0'            ; แปลงเป็น digit
    shl  eax, 3             ; คูณด้วย 8 (shift left 3)
    add  eax, ecx
    inc  esi
    jmp  .parse_octal

.parse_done:
    mov  [mode_val], eax

    ; ===== เรียก sys_chmod =====
    mov eax, SYS_CHMOD
    mov ebx, [esp+12]       ; argv[2] = filename
    mov ecx, [mode_val]     ; mode
    int 0x80

    ; ตรวจสอบผลลัพธ์
    test eax, eax
    js   .chmod_error

    ; สำเร็จ
    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.bad_mode:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_mode
    mov edx, err_mode_len
    int 0x80
    jmp .exit_fail

.usage_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, usage_msg
    mov edx, usage_len
    int 0x80
    jmp .exit_fail

.chmod_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_chmod
    mov edx, err_chmod_len
    int 0x80
    jmp .exit_fail

.exit_fail:
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

section .data
    usage_msg       db "Usage: mychmod <mode> <file>", 0x0A
    usage_len       equ $ - usage_msg
    err_mode        db "Error: Invalid mode (use octal like 755)", 0x0A
    err_mode_len    equ $ - err_mode
    err_chmod       db "Error: chmod failed", 0x0A
    err_chmod_len   equ $ - err_chmod
```

### 6.7 mkdir (สร้าง directory)

```nasm
; mkdir.asm - สร้าง directory
; ใช้: ./mymkdir <dirname>
; คอมไพล์: nasm -f elf32 mkdir.asm -o mkdir.o && ld -m elf_i386 mkdir.o -o mymkdir

%define SYS_MKDIR   39
%define SYS_WRITE   4
%define SYS_EXIT    1
%define STDERR      2
%define MODE_755    493     ; 0755 octal = 493 decimal

section .text
    global _start

_start:
    ; ตรวจสอบ arguments
    mov eax, [esp]
    cmp eax, 2
    jne .usage_error

    ; ===== เรียก sys_mkdir =====
    mov eax, SYS_MKDIR
    mov ebx, [esp+8]        ; argv[1] = directory name
    mov ecx, MODE_755       ; permissions: rwxr-xr-x
    int 0x80

    ; ตรวจสอบผลลัพธ์
    test eax, eax
    js   .mkdir_error

    ; สำเร็จ แสดงข้อความยืนยัน
    mov eax, 4              ; SYS_WRITE
    mov ebx, 1              ; stdout
    mov ecx, success_msg
    mov edx, success_len
    int 0x80

    mov eax, 1              ; SYS_EXIT
    mov ebx, 0
    int 0x80

.usage_error:
    mov eax, 4
    mov ebx, STDERR
    mov ecx, usage_msg
    mov edx, usage_len
    int 0x80
    mov eax, 1
    mov ebx, 1
    int 0x80

.mkdir_error:
    ; ตรวจสอบ error code
    neg  eax                ; แปลงเป็น positive
    cmp  eax, 17            ; EEXIST = 17
    je   .already_exists

    mov eax, 4
    mov ebx, STDERR
    mov ecx, err_mkdir
    mov edx, err_mkdir_len
    int 0x80
    mov eax, 1
    mov ebx, 1
    int 0x80

.already_exists:
    mov eax, 4
    mov ebx, STDERR
    mov ecx, err_exists
    mov edx, err_exists_len
    int 0x80
    mov eax, 1
    mov ebx, 1
    int 0x80

section .data
    usage_msg       db "Usage: mymkdir <dirname>", 0x0A
    usage_len       equ $ - usage_msg
    success_msg     db "Directory created successfully", 0x0A
    success_len     equ $ - success_msg
    err_mkdir       db "Error: Cannot create directory", 0x0A
    err_mkdir_len   equ $ - err_mkdir
    err_exists      db "Error: Directory already exists", 0x0A
    err_exists_len  equ $ - err_exists
```

### 6.8 rm (ลบไฟล์)

```nasm
; rm.asm - ลบไฟล์
; ใช้: ./myrm <filename>
; คอมไพล์: nasm -f elf32 rm.asm -o rm.o && ld -m elf_i386 rm.o -o myrm

%define SYS_UNLINK  10
%define SYS_WRITE   4
%define SYS_EXIT    1
%define STDOUT      1
%define STDERR      2

section .text
    global _start

_start:
    ; ตรวจสอบ arguments
    mov eax, [esp]
    cmp eax, 2
    jne .usage_error

    ; ===== เรียก sys_unlink (ลบไฟล์) =====
    mov eax, SYS_UNLINK
    mov ebx, [esp+8]        ; argv[1] = ชื่อไฟล์ที่จะลบ
    int 0x80

    ; ตรวจสอบผลลัพธ์
    test eax, eax
    js   .unlink_error

    ; สำเร็จ
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, success_msg
    mov edx, success_len
    int 0x80

    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.usage_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, usage_msg
    mov edx, usage_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

.unlink_error:
    ; จัดการ error code
    neg  eax

    cmp  eax, 2             ; ENOENT
    je   .no_file
    cmp  eax, 13            ; EACCES
    je   .no_perm
    cmp  eax, 1             ; EPERM (ลบ directory ด้วย unlink ไม่ได้)
    je   .is_dir

    ; general error
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_generic
    mov edx, err_generic_len
    int 0x80
    jmp .exit_fail

.no_file:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_nofile
    mov edx, err_nofile_len
    int 0x80
    jmp .exit_fail

.no_perm:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_perm
    mov edx, err_perm_len
    int 0x80
    jmp .exit_fail

.is_dir:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_isdir
    mov edx, err_isdir_len
    int 0x80

.exit_fail:
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

section .data
    usage_msg       db "Usage: myrm <filename>", 0x0A
    usage_len       equ $ - usage_msg
    success_msg     db "File removed successfully", 0x0A
    success_len     equ $ - success_msg
    err_generic     db "Error: Cannot remove file", 0x0A
    err_generic_len equ $ - err_generic
    err_nofile      db "Error: File not found", 0x0A
    err_nofile_len  equ $ - err_nofile
    err_perm        db "Error: Permission denied", 0x0A
    err_perm_len    equ $ - err_perm
    err_isdir       db "Error: Is a directory (use rmdir)", 0x0A
    err_isdir_len   equ $ - err_isdir
```

### 6.9 Write String to File (เขียน string ลงไฟล์)

```nasm
; write_file.asm - เขียน string ลงไฟล์
; ใช้: ./write_file
; คอมไพล์: nasm -f elf32 write_file.asm -o write_file.o && ld -m elf_i386 write_file.o -o write_file

%define SYS_OPEN    5
%define SYS_WRITE   4
%define SYS_CLOSE   6
%define SYS_EXIT    1
%define O_WRONLY    1
%define O_CREAT     64
%define O_TRUNC     512
%define MODE_644    420     ; rw-r--r--
%define STDOUT      1
%define STDERR      2

section .data
    ; ชื่อไฟล์ที่จะสร้าง
    filename    db "/tmp/output.txt", 0

    ; เนื้อหาที่จะเขียน (หลายบรรทัด)
    content     db "สวัสดีครับ, นี่คือการทดสอบ Assembly System Call", 0x0A
                db "Line 2: Hello from assembly language", 0x0A
                db "Line 3: Writing to file with int 0x80", 0x0A
    content_len equ $ - content

    success_msg db "File written successfully: /tmp/output.txt", 0x0A
    success_len equ $ - success_msg
    err_msg     db "Error: Cannot write to file", 0x0A
    err_len     equ $ - err_msg

section .bss
    fd resd 1

section .text
    global _start

_start:
    ; ===== เปิดไฟล์เพื่อเขียน (สร้างใหม่หรือ overwrite) =====
    mov eax, SYS_OPEN
    mov ebx, filename
    mov ecx, O_WRONLY | O_CREAT | O_TRUNC   ; 577
    mov edx, MODE_644
    int 0x80

    test eax, eax
    js   .open_error
    mov  [fd], eax

    ; ===== เขียนเนื้อหาลงไฟล์ =====
    mov eax, SYS_WRITE
    mov ebx, [fd]
    mov ecx, content
    mov edx, content_len
    int 0x80

    test eax, eax
    js   .write_error

    ; ===== ปิดไฟล์ =====
    mov eax, SYS_CLOSE
    mov ebx, [fd]
    int 0x80

    ; แสดงข้อความสำเร็จ
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, success_msg
    mov edx, success_len
    int 0x80

    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.open_error:
.write_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, err_msg
    mov edx, err_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80
```

### 6.10 Read All Input (อ่านจาก stdin จนหมด)

```nasm
; read_all.asm - อ่านจาก stdin ทั้งหมดแล้วแสดงผล (เหมือน cat)
; ใช้: echo "hello" | ./read_all
; หรือ: ./read_all < input.txt
; คอมไพล์: nasm -f elf32 read_all.asm -o read_all.o && ld -m elf_i386 read_all.o -o read_all

%define SYS_READ    3
%define SYS_WRITE   4
%define SYS_EXIT    1
%define STDIN       0
%define STDOUT      1
%define BUF_SIZE    4096

section .bss
    buffer resb BUF_SIZE        ; buffer สำหรับอ่าน
    total_bytes resd 1          ; นับ bytes ที่อ่านได้ทั้งหมด

section .data
    ; ข้อความสรุป
    summary_prefix  db 0x0A, "--- Total bytes read: "
    summary_pre_len equ $ - summary_prefix
    num_str         times 20 db 0   ; สำหรับ itoa

section .text
    global _start

_start:
    mov dword [total_bytes], 0  ; เริ่มนับที่ 0

.read_loop:
    ; ===== อ่านจาก stdin =====
    mov eax, SYS_READ
    mov ebx, STDIN              ; fd = 0 (stdin)
    mov ecx, buffer             ; buffer
    mov edx, BUF_SIZE           ; max bytes
    int 0x80

    ; ตรวจสอบผลลัพธ์
    test eax, eax
    jz   .eof                   ; EOF: ไม่มีข้อมูลเพิ่ม
    js   .read_error            ; Error

    ; เพิ่มจำนวน bytes ทั้งหมด
    add  [total_bytes], eax

    ; ===== เขียนออก stdout ทันที =====
    mov  edx, eax               ; จำนวน bytes
    mov  eax, SYS_WRITE
    mov  ebx, STDOUT
    mov  ecx, buffer
    int  0x80

    jmp .read_loop

.eof:
    ; แสดงสรุป
    mov  eax, SYS_WRITE
    mov  ebx, STDOUT
    mov  ecx, summary_prefix
    mov  edx, summary_pre_len
    int  0x80

    ; แปลงจำนวนเป็น string
    mov  eax, [total_bytes]
    mov  edi, num_str + 19
    mov  byte [edi], 0x0A       ; newline ท้าย
    dec  edi

    test eax, eax
    jnz  .do_convert
    mov  byte [edi], '0'
    dec  edi
    jmp  .print_num

.do_convert:
.cvt_loop:
    test eax, eax
    jz   .print_num
    xor  edx, edx
    mov  ecx, 10
    div  ecx
    add  dl, '0'
    mov  [edi], dl
    dec  edi
    jmp  .cvt_loop

.print_num:
    inc  edi                    ; เลื่อนกลับไปยัง first digit
    ; คำนวณความยาว
    mov  esi, num_str + 20      ; ท้าย string (รวม newline)
    sub  esi, edi
    mov  edx, esi               ; length

    mov  eax, SYS_WRITE
    mov  ebx, STDOUT
    mov  ecx, edi               ; start of number string
    int  0x80

    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.read_error:
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80
```

### 6.11 Process Fork + Exec

```nasm
; fork_exec.asm - สร้าง child process และรันโปรแกรมอื่น
; โปรแกรมนี้ fork แล้ว child รัน /bin/ls
; parent รอ child จบแล้วแสดง exit status
; คอมไพล์: nasm -f elf32 fork_exec.asm -o fork_exec.o && ld -m elf_i386 fork_exec.o -o fork_exec

%define SYS_FORK    2
%define SYS_EXEC    11
%define SYS_WAITPID 7
%define SYS_WRITE   4
%define SYS_EXIT    1
%define STDOUT      1
%define STDERR      2

section .data
    ; โปรแกรมที่จะรัน
    prog_name   db "/bin/ls", 0

    ; Arguments array สำหรับ execve
    ; argv = ["/bin/ls", "-la", NULL]
    argv_arr:
        dd prog_name        ; argv[0] = ชื่อโปรแกรม
        dd arg_la           ; argv[1] = "-la"
        dd 0                ; argv[2] = NULL (terminate)

    arg_la      db "-la", 0

    ; Environment array (empty)
    envp_arr:
        dd 0                ; NULL = ไม่มี env

    ; Messages
    parent_msg  db "Parent: Waiting for child...", 0x0A
    parent_len  equ $ - parent_msg
    fork_err    db "Error: fork() failed", 0x0A
    fork_err_len equ $ - fork_err
    child_done  db "Parent: Child finished with status: "
    child_done_len equ $ - child_done
    exec_err    db "Child: exec() failed", 0x0A
    exec_err_len equ $ - exec_err

section .bss
    child_pid   resd 1      ; เก็บ PID ของ child
    child_status resd 1     ; เก็บ exit status ของ child
    status_buf  resb 12     ; buffer สำหรับแปลง status

section .text
    global _start

_start:
    ; ===== FORK =====
    mov eax, SYS_FORK
    int 0x80

    ; ตรวจสอบ fork result
    test eax, eax
    js   .fork_error        ; < 0: error
    jz   .child_code        ; = 0: นี่คือ child process

    ; ===== PARENT CODE =====
    mov [child_pid], eax    ; เก็บ child PID

    ; แสดงข้อความ
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, parent_msg
    mov edx, parent_len
    int 0x80

    ; ===== รอ child (WAITPID) =====
    mov eax, SYS_WAITPID
    mov ebx, [child_pid]    ; รอ child PID นี้
    mov ecx, child_status   ; เก็บ status ที่นี่
    mov edx, 0              ; options = 0
    int 0x80

    ; แสดง exit status
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, child_done
    mov edx, child_done_len
    int 0x80

    ; แสดงตัวเลข exit status
    ; child_status เก็บ wait status (ต้องแกะด้วย WEXITSTATUS)
    mov  eax, [child_status]
    shr  eax, 8             ; WEXITSTATUS = (status >> 8) & 0xFF
    and  eax, 0xFF

    ; แปลงเป็น string และแสดง
    call .print_number

    ; Exit parent
    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

    ; ===== CHILD CODE =====
.child_code:
    ; ===== EXECVE /bin/ls -la =====
    mov eax, SYS_EXEC
    mov ebx, prog_name      ; ชื่อโปรแกรม
    mov ecx, argv_arr       ; arguments
    mov edx, envp_arr       ; environment
    int 0x80

    ; ถ้า execve กลับมา แสดงว่า error
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, exec_err
    mov edx, exec_err_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

.fork_error:
    mov eax, SYS_WRITE
    mov ebx, STDERR
    mov ecx, fork_err
    mov edx, fork_err_len
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

; Function: พิมพ์ตัวเลขใน EAX + newline
.print_number:
    push eax
    push ebx
    push ecx
    push edx
    push edi

    mov  edi, status_buf + 11
    mov  byte [edi], 0x0A
    dec  edi
    mov  byte [edi], 0

    ; ตรวจสอบ 0
    pop  edi
    push edi
    mov  eax, [esp+4]       ; original eax

    test eax, eax
    jnz  .num_loop
    ; เลข 0
    mov  edi, status_buf
    mov  byte [edi], '0'
    mov  byte [edi+1], 0x0A
    mov  edx, 2
    jmp  .num_print

.num_loop:
    mov  edi, status_buf + 10
    mov  byte [edi+1], 0x0A
    xor  ecx, ecx

.digit_extract:
    test eax, eax
    jz   .num_done
    xor  edx, edx
    mov  ebx, 10
    div  ebx
    add  dl, '0'
    mov  [edi], dl
    dec  edi
    inc  ecx
    jmp  .digit_extract

.num_done:
    inc  edi
    add  ecx, 1             ; +1 สำหรับ newline
    mov  edx, ecx

.num_print:
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, edi
    int 0x80

    pop  edi
    pop  edx
    pop  ecx
    pop  ebx
    pop  eax
    ret
```

---

## 7. Networking Syscalls

### 7.1 TCP Server (รับ connection)

```nasm
; tcp_server.asm - Simple TCP server บน port 8080
; คอมไพล์: nasm -f elf32 tcp_server.asm -o tcp_server.o
; ลิงก์:   ld -m elf_i386 tcp_server.o -o tcp_server
; ทดสอบ:  ./tcp_server & curl http://localhost:8080/

%define SYS_SOCKET  359
%define SYS_BIND    361
%define SYS_LISTEN  363
%define SYS_ACCEPT  364
%define SYS_WRITE   4
%define SYS_CLOSE   6
%define SYS_EXIT    1
%define AF_INET     2
%define SOCK_STREAM 1
%define INADDR_ANY  0
%define PORT        8080
%define STDOUT      1

; sockaddr_in structure
; struct sockaddr_in {
;   short  sin_family;    +0 (2 bytes) = AF_INET
;   ushort sin_port;      +2 (2 bytes) = port (big-endian)
;   uint   sin_addr;      +4 (4 bytes) = IP address
;   char   sin_zero[8];   +8 (8 bytes) = zeros
; }

section .data
    ; sockaddr_in สำหรับ bind
    server_addr:
        dw AF_INET          ; sin_family = 2
        dw 0x901F           ; sin_port = 8080 big-endian (0x1F90 -> 0x901F)
        dd INADDR_ANY       ; sin_addr = 0.0.0.0
        times 8 db 0        ; sin_zero

    http_response:
        db "HTTP/1.1 200 OK", 0x0D, 0x0A
        db "Content-Type: text/plain", 0x0D, 0x0A
        db "Content-Length: 13", 0x0D, 0x0A
        db 0x0D, 0x0A
        db "Hello, World!"
    http_len equ $ - http_response

    listen_msg db "Server listening on port 8080...", 0x0A
    listen_len equ $ - listen_msg

section .bss
    sockfd      resd 1      ; server socket
    clientfd    resd 1      ; client socket
    client_addr resb 16     ; ที่อยู่ client

section .text
    global _start

_start:
    ; ===== สร้าง socket =====
    mov eax, SYS_SOCKET
    mov ebx, AF_INET        ; domain
    mov ecx, SOCK_STREAM    ; type
    mov edx, 0              ; protocol
    int 0x80
    test eax, eax
    js   .error
    mov  [sockfd], eax

    ; ===== bind socket กับ address:port =====
    mov eax, SYS_BIND
    mov ebx, [sockfd]
    mov ecx, server_addr    ; struct sockaddr_in
    mov edx, 16             ; sizeof(sockaddr_in)
    int 0x80
    test eax, eax
    jnz  .error

    ; ===== listen สำหรับ connections =====
    mov eax, SYS_LISTEN
    mov ebx, [sockfd]
    mov ecx, 5              ; backlog = 5
    int 0x80
    test eax, eax
    jnz  .error

    ; แสดงข้อความว่ากำลัง listen
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, listen_msg
    mov edx, listen_len
    int 0x80

.accept_loop:
    ; ===== รับ connection =====
    mov eax, SYS_ACCEPT
    mov ebx, [sockfd]
    mov ecx, client_addr    ; ที่อยู่ client
    push dword 16
    mov  edx, esp           ; pointer to address length
    int  0x80
    add  esp, 4

    test eax, eax
    js   .accept_loop       ; ถ้า error ลองใหม่

    mov [clientfd], eax

    ; ===== ส่ง HTTP response =====
    mov eax, SYS_WRITE
    mov ebx, [clientfd]
    mov ecx, http_response
    mov edx, http_len
    int 0x80

    ; ===== ปิด client socket =====
    mov eax, SYS_CLOSE
    mov ebx, [clientfd]
    int 0x80

    jmp .accept_loop        ; รอ connection ถัดไป

.error:
    mov eax, SYS_CLOSE
    mov ebx, [sockfd]
    int 0x80
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80
```

---

## 8. Memory Management ด้วย mmap

### 8.1 Allocate Anonymous Memory

```nasm
; mmap_demo.asm - จัดการ memory ด้วย mmap
; คอมไพล์: nasm -f elf32 mmap_demo.asm -o mmap_demo.o && ld -m elf_i386 mmap_demo.o -o mmap_demo

%define SYS_MMAP    90
%define SYS_MUNMAP  91
%define SYS_WRITE   4
%define SYS_EXIT    1
%define STDOUT      1
%define PROT_RDWR   3       ; PROT_READ | PROT_WRITE
%define MAP_PRIV_ANON 0x22  ; MAP_PRIVATE | MAP_ANONYMOUS

section .data
    ; struct mmap_arg_struct
    mmap_args:
        dd 0            ; addr = 0 (kernel เลือก)
        dd 4096         ; len = 4096 bytes (1 page)
        dd PROT_RDWR    ; prot = อ่านและเขียน
        dd MAP_PRIV_ANON ; flags
        dd -1           ; fd = -1 (anonymous)
        dd 0            ; offset = 0

    ; Messages
    ok_msg      db "Memory allocated at: 0x"
    ok_len      equ $ - ok_msg
    newline     db 0x0A, 0
    write_msg   db "Wrote to memory: Hello!"
    write_len   equ $ - write_msg
    free_msg    db 0x0A, "Memory freed", 0x0A
    free_len    equ $ - free_msg

section .bss
    mapped_addr resd 1
    hex_buf     resb 9      ; "XXXXXXXX" + newline

section .text
    global _start

_start:
    ; ===== Allocate memory ด้วย mmap =====
    mov eax, SYS_MMAP
    mov ebx, mmap_args
    int 0x80

    ; ตรวจสอบว่าสำเร็จ
    cmp eax, -1
    je  .mmap_error
    mov [mapped_addr], eax

    ; แสดง address ที่ได้
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, ok_msg
    mov edx, ok_len
    int 0x80

    ; แปลง address เป็น hex และแสดง
    mov eax, [mapped_addr]
    call print_hex

    ; ===== เขียนข้อมูลลงใน mapped memory =====
    mov esi, [mapped_addr]              ; ที่อยู่ที่ allocate
    mov dword [esi], 'Hell'             ; เขียน 4 bytes
    mov dword [esi+4], 'o! ('          ; เขียน 4 bytes ถัดไป
    ; (ตัวอย่างง่ายๆ เท่านั้น)

    ; แสดงว่าเขียนแล้ว
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, write_msg
    mov edx, write_len
    int 0x80

    ; ===== Free memory ด้วย munmap =====
    mov eax, SYS_MUNMAP
    mov ebx, [mapped_addr]
    mov ecx, 4096
    int 0x80

    ; แสดง freed
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, free_msg
    mov edx, free_len
    int 0x80

    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80

.mmap_error:
    mov eax, SYS_EXIT
    mov ebx, 1
    int 0x80

; Function: พิมพ์ EAX เป็น hex + newline
print_hex:
    push eax
    push ebx
    push ecx
    push edx

    mov  edi, hex_buf
    mov  ecx, 8             ; 8 hex digits
    mov  edx, eax

.hex_loop:
    rol  edx, 4             ; rotate left 4 bits
    mov  al, dl
    and  al, 0x0F
    cmp  al, 9
    jle  .is_digit
    add  al, 'A' - 10
    jmp  .store
.is_digit:
    add  al, '0'
.store:
    mov  [edi], al
    inc  edi
    loop .hex_loop

    mov  byte [edi], 0x0A
    mov  byte [edi+1], 0

    mov  eax, SYS_WRITE
    mov  ebx, STDOUT
    mov  ecx, hex_buf
    mov  edx, 9
    int  0x80

    pop  edx
    pop  ecx
    pop  ebx
    pop  eax
    ret
```

---

## 9. เทคนิคขั้นสูง

### 9.1 Saving และ Restoring Registers

```nasm
; Convention: syscall ทำลาย EAX, ECX, EDX
; ส่วน EBX, ESI, EDI, EBP ถูกรักษาไว้
; แต่ syscall บางตัวอาจมีผลต่อ registers อื่น

; วิธีที่ดีที่สุดคือ save ก่อน syscall ถ้าต้องการค่าเดิม
save_regs_example:
    push eax        ; save EAX
    push ecx        ; save ECX
    push edx        ; save EDX

    ; ทำ syscall
    mov eax, 4
    mov ebx, 1
    mov ecx, msg
    mov edx, 10
    int 0x80

    pop  edx        ; restore EDX
    pop  ecx        ; restore ECX
    pop  eax        ; restore EAX
    ret
```

### 9.2 การใช้ Stack สำหรับ Local Buffer

```nasm
; allocate local buffer บน stack
stack_buffer_example:
    push ebp
    mov  ebp, esp

    ; สร้าง buffer 256 bytes บน stack
    sub  esp, 256
    ; ตอนนี้ ESP ชี้ไปยัง buffer
    mov  eax, esp           ; ที่อยู่ของ buffer

    ; ใช้งาน buffer
    mov eax, 3              ; SYS_READ
    mov ebx, 0              ; stdin
    mov ecx, esp            ; buffer address
    mov edx, 255            ; max 255 bytes
    int 0x80

    ; cleanup
    mov  esp, ebp
    pop  ebp
    ret
```

### 9.3 Non-blocking I/O

```nasm
; ตั้งค่า non-blocking สำหรับ socket
%define SYS_FCNTL   55
%define F_GETFL     3
%define F_SETFL     4
%define O_NONBLOCK  2048

set_nonblocking:
    ; ดึง flags ปัจจุบัน
    mov eax, SYS_FCNTL
    mov ebx, [sockfd]
    mov ecx, F_GETFL
    mov edx, 0
    int 0x80

    ; เพิ่ม O_NONBLOCK flag
    or  eax, O_NONBLOCK
    mov edx, eax

    ; set flags ใหม่
    mov eax, SYS_FCNTL
    mov ebx, [sockfd]
    mov ecx, F_SETFL
    ; edx = new flags
    int 0x80
    ret
```

---

## 10. Makefile สำหรับ Compile

```makefile
# Makefile สำหรับ 32-bit assembly programs
CC = nasm
LDFLAGS = -m elf_i386
CFLAGS = -f elf32

PROGRAMS = hello mycat mycp mywc myls mychmod mymkdir myrm \
           write_file read_all fork_exec

all: $(PROGRAMS)

%: %.asm
	$(CC) $(CFLAGS) $< -o $@.o
	ld $(LDFLAGS) $@.o -o $@

clean:
	rm -f *.o $(PROGRAMS)

# ทดสอบแต่ละโปรแกรม
test: all
	@echo "=== Testing hello ==="
	./hello
	@echo "=== Testing write_file ==="
	./write_file
	@echo "=== Testing read_all ==="
	echo "test input" | ./read_all
	@echo "=== Testing mymkdir ==="
	./mymkdir /tmp/test_asm_dir
	@echo "=== Testing mychmod ==="
	./mychmod 755 /tmp/test_asm_dir
	@echo "=== Testing myls ==="
	./myls /tmp
	@echo "=== Done ==="
```

---

## 11. ข้อควรระวัง

### 11.1 Alignment ของ Stack

```nasm
; บน 32-bit x86 stack ควร align ที่ 4 bytes
; บางระบบต้องการ 16-byte alignment ก่อน function call
; แต่สำหรับ syscall ไม่จำเป็น

; ตรวจสอบ alignment
test esp, 15        ; ตรวจสอบว่าหารด้วย 16 ได้
jnz  .misaligned   ; ถ้าไม่ align
```

### 11.2 Null-Terminated Strings

```nasm
; string ที่ส่งให้ syscall เช่น open, execve ต้องจบด้วย 0 (null byte)

filename db "/tmp/test.txt", 0    ; ถูกต้อง: มี null terminator
filename_bad db "/tmp/test.txt"   ; ผิด: ไม่มี null terminator
```

### 11.3 Atomic Operations

```nasm
; สำหรับ multi-process programs
; write() syscall เป็น atomic ถ้าขนาดน้อยกว่า PIPE_BUF (4096 bytes)
; แต่สำหรับ file operations ต้องใช้ lock

%define SYS_FLOCK   143         ; flock syscall
%define LOCK_EX     2           ; exclusive lock
%define LOCK_UN     8           ; unlock

; lock file
mov eax, SYS_FLOCK
mov ebx, [fd]
mov ecx, LOCK_EX
int 0x80

; ... ทำงาน ...

; unlock file
mov eax, SYS_FLOCK
mov ebx, [fd]
mov ecx, LOCK_UN
int 0x80
```

### 11.4 Signal Handling

```nasm
; การ handle SIGCHLD เพื่อป้องกัน zombie processes
%define SYS_SIGNAL  48
%define SIGCHLD     17
%define SIG_IGN     1           ; ignore signal

; ignore SIGCHLD เพื่อป้องกัน zombie
mov eax, SYS_SIGNAL
mov ebx, SIGCHLD
mov ecx, SIG_IGN
int 0x80
```

---

## 12. Debugging Syscalls

### 12.1 ใช้ strace

```bash
# trace syscalls ของโปรแกรม
strace ./hello

# ดูเฉพาะ syscalls ที่สนใจ
strace -e trace=open,read,write ./mycat /etc/hostname

# ดู syscall ของ process ที่รันอยู่
strace -p PID
```

### 12.2 ใช้ gdb

```bash
# debug ใน gdb
gdb ./hello
(gdb) break _start
(gdb) run
(gdb) info registers    ; ดู registers ทั้งหมด
(gdb) stepi             ; step แต่ละ instruction
(gdb) x/10x $esp        ; ดู stack
```

### 12.3 ตรวจสอบ Syscall Number

```bash
# ดู syscall numbers
grep -E "^#define __NR_" /usr/include/asm/unistd_32.h | head -30

# หรือใช้ ausyscall (ถ้ามี)
ausyscall 4             ; บอกว่า syscall 4 คือ write
ausyscall write         ; บอก number ของ write
```

---

## 13. สรุปและ Quick Reference

### 13.1 Template โปรแกรม Assembly 32-bit

```nasm
; template.asm - Template สำหรับโปรแกรม 32-bit Linux
; คอมไพล์: nasm -f elf32 template.asm -o template.o && ld -m elf_i386 template.o -o template

; ===== Syscall Numbers =====
%define SYS_EXIT    1
%define SYS_FORK    2
%define SYS_READ    3
%define SYS_WRITE   4
%define SYS_OPEN    5
%define SYS_CLOSE   6

; ===== File Descriptors =====
%define STDIN       0
%define STDOUT      1
%define STDERR      2

; ===== Open Flags =====
%define O_RDONLY    0
%define O_WRONLY    1
%define O_RDWR      2
%define O_CREAT     64
%define O_TRUNC     512
%define O_APPEND    1024

; ===== Permissions =====
%define MODE_644    420
%define MODE_755    493

section .data
    ; ข้อมูลแบบ static
    msg     db "Hello!", 0x0A
    msg_len equ $ - msg

section .bss
    ; ตัวแปรแบบ uninitialized
    buffer  resb 4096
    fd_var  resd 1

section .text
    global _start

_start:
    ; ===== เริ่มโค้ดที่นี่ =====

    ; เขียนข้อความ
    mov eax, SYS_WRITE
    mov ebx, STDOUT
    mov ecx, msg
    mov edx, msg_len
    int 0x80

    ; จบโปรแกรม
    mov eax, SYS_EXIT
    mov ebx, 0
    int 0x80
```

### 13.2 Cheat Sheet: Syscalls บ่อยใช้

```
sys_exit(1):   EAX=1,  EBX=status
sys_fork(2):   EAX=2   → EAX=child_pid (parent), 0 (child)
sys_read(3):   EAX=3,  EBX=fd, ECX=buf, EDX=count → EAX=bytes
sys_write(4):  EAX=4,  EBX=fd, ECX=buf, EDX=count → EAX=bytes
sys_open(5):   EAX=5,  EBX=path, ECX=flags, EDX=mode → EAX=fd
sys_close(6):  EAX=6,  EBX=fd
sys_waitpid(7): EAX=7, EBX=pid, ECX=&status, EDX=options
sys_execve(11): EAX=11, EBX=prog, ECX=argv, EDX=envp
sys_chmod(15): EAX=15, EBX=path, ECX=mode
sys_getpid(20): EAX=20 → EAX=pid
sys_mkdir(39): EAX=39, EBX=path, ECX=mode
sys_unlink(10): EAX=10, EBX=path
sys_kill(37):  EAX=37, EBX=pid, ECX=signal
sys_brk(45):   EAX=45, EBX=addr → EAX=new_brk
sys_mmap(90):  EAX=90, EBX=&args → EAX=addr
sys_munmap(91): EAX=91, EBX=addr, ECX=len
sys_stat(106): EAX=106, EBX=path, ECX=&stat
sys_getdents(141): EAX=141, EBX=fd, ECX=buf, EDX=size
```

---

## แหล่งอ้างอิง

1. **Linux Syscall Reference**: `/usr/include/asm/unistd_32.h`
2. **man pages**: `man 2 <syscall_name>` เช่น `man 2 write`
3. **strace manual**: `man strace`
4. **NASM documentation**: https://nasm.us/doc/
5. **Linux Kernel Source**: https://github.com/torvalds/linux
6. **x86 Assembly Language**: Intel IA-32 Architectures Software Developer's Manual

---

*Part 051: Linux System Calls (32-bit x86) — เนื้อหาครอบคลุมการใช้ int 0x80, syscall table, และโปรแกรมตัวอย่างสมบูรณ์*
