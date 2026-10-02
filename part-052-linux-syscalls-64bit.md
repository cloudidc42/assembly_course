# Part 052: Linux System Calls (64-bit x86-64)

## บทนำ (Introduction)

System calls คือ interface หลักระหว่าง user space programs กับ Linux kernel การเรียก system call ใน 64-bit x86-64 Linux ใช้ instruction `syscall` ซึ่งแตกต่างจาก 32-bit ที่ใช้ `int 0x80`

---

## 1. syscall Instruction vs int 0x80

### 1.1 ความแตกต่างหลัก

| Feature | 64-bit (`syscall`) | 32-bit (`int 0x80`) |
|---------|-------------------|---------------------|
| Instruction | `syscall` | `int 0x80` |
| Syscall number | RAX | EAX |
| First arg | RDI | EBX |
| Second arg | RSI | ECX |
| Third arg | RDX | EDX |
| Fourth arg | R10 | ESI |
| Fifth arg | R8 | EDI |
| Sixth arg | R9 | EBP |
| Return value | RAX | EAX |

### 1.2 ทำไมต้องใช้ syscall แทน int 0x80

```nasm
; อย่าใช้แบบนี้ใน 64-bit mode (ผิด!)
; mov eax, 1        ; write syscall number ของ 32-bit
; int 0x80          ; 32-bit interface

; ใช้แบบนี้ถูกต้องสำหรับ 64-bit
mov rax, 1          ; write syscall number ของ 64-bit
syscall             ; 64-bit interface
```

`syscall` instruction เร็วกว่า `int 0x80` เพราะไม่ต้องผ่าน interrupt descriptor table (IDT)

---

## 2. Calling Convention สำหรับ System Calls

### 2.1 Register Usage

```
RAX = syscall number (หมายเลข system call)
RDI = argument 1 (พารามิเตอร์ที่ 1)
RSI = argument 2 (พารามิเตอร์ที่ 2)
RDX = argument 3 (พารามิเตอร์ที่ 3)
R10 = argument 4 (พารามิเตอร์ที่ 4) [ไม่ใช่ RCX!]
R8  = argument 5 (พารามิเตอร์ที่ 5)
R9  = argument 6 (พารามิเตอร์ที่ 6)

RAX = return value (ค่าที่ส่งกลับ - อาจเป็นค่าลบถ้า error)
```

### 2.2 Registers ที่ถูก Clobber โดย syscall

**สำคัญมาก:** `syscall` instruction จะทำลายค่าใน:
- **RCX** - kernel เก็บ return address ไว้ที่นี่
- **R11** - kernel เก็บ RFLAGS ไว้ที่นี่

```nasm
; ตัวอย่างการ save/restore registers ก่อน syscall
push rcx        ; save RCX ก่อนถ้าต้องการใช้หลัง syscall
push r11        ; save R11 ก่อนถ้าต้องการใช้หลัง syscall

mov rax, 1      ; syscall number
mov rdi, 1      ; argument 1
syscall         ; RCX และ R11 ถูกทำลายตรงนี้!

pop r11         ; restore R11
pop rcx         ; restore RCX
```

### 2.3 Error Handling

```nasm
; หลัง syscall ตรวจสอบ error
syscall
test rax, rax   ; ตรวจว่า RAX เป็น 0 หรือไม่
js .error       ; ถ้า negative = error (sign bit set)
; หรือ
cmp rax, -4095  ; Linux error range: -1 ถึง -4095
jae .error      ; ถ้า >= (unsigned) = error
```

---

## 3. Syscall Number Table (/usr/include/asm/unistd_64.h)

```c
// ตำแหน่งไฟล์: /usr/include/asm/unistd_64.h
// หรือ: /usr/include/x86_64-linux-gnu/asm/unistd_64.h

#define __NR_read                0
#define __NR_write               1
#define __NR_open                2
#define __NR_close               3
#define __NR_stat                4
#define __NR_fstat               5
#define __NR_lstat               6
#define __NR_poll                7
#define __NR_lseek               8
#define __NR_mmap                9
#define __NR_mprotect           10
#define __NR_munmap             11
#define __NR_brk                12
#define __NR_rt_sigaction       13
#define __NR_rt_sigprocmask     14
#define __NR_ioctl              16
#define __NR_pread64            17
#define __NR_pwrite64           18
#define __NR_readv              19
#define __NR_writev             20
#define __NR_pipe               22
#define __NR_dup                32
#define __NR_dup2               33
#define __NR_nanosleep          35
#define __NR_getpid             39
#define __NR_socket             41
#define __NR_connect            42
#define __NR_accept             43
#define __NR_sendto             44
#define __NR_recvfrom           45
#define __NR_bind               49
#define __NR_listen             50
#define __NR_clone              56
#define __NR_fork               57
#define __NR_vfork              58
#define __NR_execve             59
#define __NR_exit               60
#define __NR_wait4              61
#define __NR_kill               62
#define __NR_futex             202
#define __NR_epoll_wait        232
#define __NR_epoll_ctl         233
#define __NR_epoll_create1     291
```

---

## 4. System Calls อย่างละเอียด

### 4.1 read (syscall 0)

```c
// C prototype:
ssize_t read(int fd, void *buf, size_t count);
```

```nasm
; Assembly:
; RDI = file descriptor
; RSI = buffer pointer
; RDX = number of bytes to read
; RAX = bytes read (หรือ -errno ถ้า error)

mov rax, 0          ; SYS_read
mov rdi, 0          ; fd = 0 (stdin)
mov rsi, buffer     ; pointer ไปยัง buffer
mov rdx, 1024       ; อ่านสูงสุด 1024 bytes
syscall
```

### 4.2 write (syscall 1)

```c
// C prototype:
ssize_t write(int fd, const void *buf, size_t count);
```

```nasm
; Assembly:
; RDI = file descriptor
; RSI = buffer pointer
; RDX = number of bytes to write
; RAX = bytes written (หรือ -errno ถ้า error)

mov rax, 1          ; SYS_write
mov rdi, 1          ; fd = 1 (stdout)
mov rsi, message    ; pointer ไปยัง string
mov rdx, msg_len    ; ความยาว string
syscall
```

### 4.3 open (syscall 2)

```c
// C prototype:
int open(const char *pathname, int flags, mode_t mode);
```

```nasm
; Assembly:
; RDI = pathname string pointer
; RSI = flags (O_RDONLY=0, O_WRONLY=1, O_RDWR=2, O_CREAT=64, ...)
; RDX = mode (permissions เช่น 0644)
; RAX = file descriptor (หรือ -errno ถ้า error)

; Flags สำคัญ:
; O_RDONLY  = 0
; O_WRONLY  = 1
; O_RDWR    = 2
; O_CREAT   = 64  (0x40)
; O_TRUNC   = 512 (0x200)
; O_APPEND  = 1024 (0x400)
; O_NONBLOCK = 2048 (0x800)

mov rax, 2          ; SYS_open
mov rdi, filename   ; pointer ไปยัง filename string
mov rsi, 0          ; O_RDONLY
mov rdx, 0          ; mode (ไม่จำเป็นสำหรับ O_RDONLY)
syscall
```

### 4.4 close (syscall 3)

```c
// C prototype:
int close(int fd);
```

```nasm
; RDI = file descriptor
; RAX = 0 (success) หรือ -errno

mov rax, 3          ; SYS_close
mov rdi, r12        ; fd ที่ต้องการ close (สมมติเก็บไว้ใน r12)
syscall
```

### 4.5 stat และ fstat (syscall 4, 5)

```c
// C prototype:
int stat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);
```

```nasm
; struct stat layout (ย่อ):
; offset 0:  st_dev   (8 bytes)
; offset 8:  st_ino   (8 bytes)
; offset 16: st_nlink (8 bytes)
; offset 24: st_mode  (4 bytes)
; offset 28: st_uid   (4 bytes)
; offset 32: st_gid   (4 bytes)
; offset 40: st_rdev  (8 bytes)
; offset 48: st_size  (8 bytes)
; offset 56: st_blksize (8 bytes)
; offset 64: st_blocks  (8 bytes)

; fstat example:
mov rax, 5          ; SYS_fstat
mov rdi, 1          ; fd = stdout
lea rsi, [statbuf]  ; pointer ไปยัง stat buffer
syscall
```

### 4.6 poll (syscall 7)

```c
// C prototype:
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

```nasm
; struct pollfd:
; offset 0: fd      (4 bytes)
; offset 4: events  (2 bytes) - POLLIN=1, POLLOUT=4
; offset 6: revents (2 bytes) - filled by kernel

; RDI = pointer ไปยัง pollfd array
; RSI = number of fds
; RDX = timeout in milliseconds (-1 = infinite)

section .bss
pollfd_arr: resb 8      ; 1 pollfd struct

section .text
    ; setup pollfd
    mov dword [pollfd_arr], 0   ; fd = stdin
    mov word [pollfd_arr+4], 1  ; events = POLLIN

    mov rax, 7          ; SYS_poll
    lea rdi, [pollfd_arr]
    mov rsi, 1          ; 1 fd
    mov rdx, 5000       ; 5 second timeout
    syscall
```

### 4.7 lseek (syscall 8)

```c
// C prototype:
off_t lseek(int fd, off_t offset, int whence);
```

```nasm
; whence values:
; SEEK_SET = 0  (จากต้นไฟล์)
; SEEK_CUR = 1  (จากตำแหน่งปัจจุบัน)
; SEEK_END = 2  (จากท้ายไฟล์)

; หา file size:
mov rax, 8          ; SYS_lseek
mov rdi, r12        ; fd
mov rsi, 0          ; offset = 0
mov rdx, 2          ; SEEK_END
syscall             ; RAX = file size

; กลับไปต้นไฟล์:
mov rax, 8          ; SYS_lseek
mov rdi, r12        ; fd
mov rsi, 0          ; offset = 0
mov rdx, 0          ; SEEK_SET
syscall
```

### 4.8 mmap (syscall 9)

```c
// C prototype:
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

```nasm
; prot flags:
; PROT_READ  = 1
; PROT_WRITE = 2
; PROT_EXEC  = 4
; PROT_NONE  = 0

; map flags:
; MAP_SHARED  = 1
; MAP_PRIVATE = 2
; MAP_ANONYMOUS = 32 (0x20)  - ไม่ bind กับ file

; RDI = addr (0 = kernel เลือก)
; RSI = length
; RDX = prot
; R10 = flags
; R8  = fd (-1 สำหรับ anonymous)
; R9  = offset

; allocate 4096 bytes anonymous memory:
mov rax, 9          ; SYS_mmap
mov rdi, 0          ; addr = NULL (kernel เลือก)
mov rsi, 4096       ; length = 1 page
mov rdx, 3          ; PROT_READ | PROT_WRITE
mov r10, 34         ; MAP_PRIVATE | MAP_ANONYMOUS (2|32=34)
mov r8, -1          ; fd = -1 (anonymous)
mov r9, 0           ; offset = 0
syscall             ; RAX = pointer ไปยัง memory
```

### 4.9 mprotect (syscall 10)

```c
// C prototype:
int mprotect(void *addr, size_t len, int prot);
```

```nasm
; เปลี่ยน memory protection
mov rax, 10         ; SYS_mprotect
mov rdi, mem_addr   ; address (ต้อง page-aligned)
mov rsi, 4096       ; length
mov rdx, 5          ; PROT_READ | PROT_EXEC (1|4=5)
syscall
```

### 4.10 munmap (syscall 11)

```c
// C prototype:
int munmap(void *addr, size_t length);
```

```nasm
mov rax, 11         ; SYS_munmap
mov rdi, mem_addr   ; address ที่ต้องการ unmap
mov rsi, 4096       ; length
syscall
```

### 4.11 brk (syscall 12)

```c
// C prototype:
int brk(void *addr);
```

```nasm
; หา current break point:
mov rax, 12         ; SYS_brk
mov rdi, 0          ; addr = 0 = return current brk
syscall             ; RAX = current program break

; ขยาย heap:
mov rax, 12         ; SYS_brk
lea rdi, [rax + 4096] ; เพิ่ม 4096 bytes
syscall
```

### 4.12 rt_sigaction (syscall 13)

```c
// C prototype:
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

```nasm
; struct sigaction layout (simplified):
; offset 0:  sa_handler/sa_sigaction (8 bytes)
; offset 8:  sa_flags (8 bytes)
; offset 16: sa_restorer (8 bytes)
; offset 24: sa_mask (8 bytes per word, 128 bytes total)

; signals:
; SIGHUP  = 1
; SIGINT  = 2
; SIGQUIT = 3
; SIGKILL = 9
; SIGTERM = 15
; SIGCHLD = 17
; SIGCONT = 18
; SIGSTOP = 19

; ติดตั้ง SIGINT handler:
; RDI = signum
; RSI = new sigaction struct
; RDX = old sigaction struct (NULL = ไม่สนใจ)
; R10 = size of sigset_t (ใช้ 8)

mov rax, 13         ; SYS_rt_sigaction
mov rdi, 2          ; SIGINT
lea rsi, [sigact]   ; new action
mov rdx, 0          ; old action = NULL
mov r10, 8          ; sigset_t size
syscall
```

### 4.13 rt_sigprocmask (syscall 14)

```c
// C prototype:
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

```nasm
; how values:
; SIG_BLOCK   = 0  (เพิ่ม signals ใน mask)
; SIG_UNBLOCK = 1  (ลบ signals จาก mask)
; SIG_SETMASK = 2  (set mask ใหม่)

mov rax, 14         ; SYS_rt_sigprocmask
mov rdi, 2          ; SIG_SETMASK
lea rsi, [new_mask] ; new signal mask
lea rdx, [old_mask] ; save old mask
mov r10, 8          ; sigset_t size
syscall
```

### 4.14 ioctl (syscall 16)

```c
// C prototype:
int ioctl(int fd, unsigned long request, ...);
```

```nasm
; TIOCGWINSZ = 0x5413 (get terminal size)
; struct winsize:
; offset 0: ws_row (2 bytes)
; offset 2: ws_col (2 bytes)
; offset 4: ws_xpixel (2 bytes)
; offset 6: ws_ypixel (2 bytes)

mov rax, 16         ; SYS_ioctl
mov rdi, 1          ; stdout fd
mov rsi, 0x5413     ; TIOCGWINSZ
lea rdx, [winsize]  ; pointer ไปยัง winsize struct
syscall
```

### 4.15 pread64 และ pwrite64 (syscall 17, 18)

```nasm
; pread64: อ่านจาก offset ที่กำหนดโดยไม่เปลี่ยน file position
; pwrite64: เขียนไปยัง offset ที่กำหนดโดยไม่เปลี่ยน file position

; RDI = fd
; RSI = buffer
; RDX = count
; R10 = offset

mov rax, 17         ; SYS_pread64
mov rdi, r12        ; fd
lea rsi, [buffer]   ; buffer
mov rdx, 1024       ; count
mov r10, 0          ; offset = 0 (อ่านจากต้น)
syscall
```

### 4.16 readv และ writev (syscall 19, 20)

```c
// C prototype:
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

```nasm
; struct iovec:
; offset 0: iov_base (8 bytes) - pointer ไปยัง buffer
; offset 8: iov_len  (8 bytes) - size ของ buffer

; เขียนหลาย buffers ในครั้งเดียว (scatter/gather I/O)

section .data
    str1    db "Hello, ", 7
    str2    db "World!", 6
    newline db 10, 1

section .bss
    iovec_arr: resb 48  ; 3 iovec structs (3 * 16 bytes)

section .text
    ; setup iovec array
    lea rax, [str1]
    mov [iovec_arr], rax
    mov qword [iovec_arr+8], 7

    lea rax, [str2]
    mov [iovec_arr+16], rax
    mov qword [iovec_arr+24], 6

    lea rax, [newline]
    mov [iovec_arr+32], rax
    mov qword [iovec_arr+40], 1

    ; writev
    mov rax, 20         ; SYS_writev
    mov rdi, 1          ; stdout
    lea rsi, [iovec_arr]
    mov rdx, 3          ; 3 iovec structs
    syscall
```

### 4.17 pipe (syscall 22)

```c
// C prototype:
int pipe(int pipefd[2]);
```

```nasm
; pipefd[0] = read end
; pipefd[1] = write end

section .bss
    pipefd: resd 2      ; 2 integers

section .text
    mov rax, 22         ; SYS_pipe
    lea rdi, [pipefd]
    syscall

    ; ตอนนี้:
    ; pipefd[0] = fd สำหรับอ่าน
    ; pipefd[1] = fd สำหรับเขียน
```

### 4.18 dup และ dup2 (syscall 32, 33)

```nasm
; dup: duplicate file descriptor
; dup2: duplicate to specific fd number

; dup
mov rax, 32         ; SYS_dup
mov rdi, 1          ; fd ที่ต้องการ duplicate
syscall             ; RAX = new fd (lowest available)

; dup2 - redirect stdout ไปยัง file:
mov rax, 33         ; SYS_dup2
mov rdi, file_fd    ; old fd (file)
mov rsi, 1          ; new fd (stdout)
syscall             ; ทำให้ fd 1 ชี้ไปยัง file_fd
```

### 4.19 nanosleep (syscall 35)

```c
// C prototype:
int nanosleep(const struct timespec *req, struct timespec *rem);
```

```nasm
; struct timespec:
; offset 0: tv_sec  (8 bytes) - seconds
; offset 8: tv_nsec (8 bytes) - nanoseconds (0-999999999)

section .bss
    timespec: resq 2    ; 2 qwords

section .text
    ; sleep 1.5 วินาที:
    mov qword [timespec], 1         ; 1 second
    mov qword [timespec+8], 500000000  ; 500 ms = 5*10^8 ns

    mov rax, 35         ; SYS_nanosleep
    lea rdi, [timespec]
    mov rsi, 0          ; rem = NULL
    syscall
```

### 4.20 getpid (syscall 39)

```nasm
; ไม่มี arguments
mov rax, 39         ; SYS_getpid
syscall             ; RAX = current process ID
```

### 4.21 fork (syscall 57)

```nasm
; ไม่มี arguments
; RAX = 0 ใน child process
; RAX = child PID ใน parent process
; RAX = -errno ถ้า error

mov rax, 57         ; SYS_fork
syscall
test rax, rax
jz .child_code      ; เป็น 0 แสดงว่าเป็น child
js .fork_error      ; negative แสดงว่า error
; ถ้าถึงตรงนี้แสดงว่าเป็น parent, RAX = child PID
```

### 4.22 vfork (syscall 58)

```nasm
; คล้าย fork แต่ parent จะ block จนกว่า child จะ exit หรือ exec
; ใช้ vstack เดียวกันกับ parent

mov rax, 58         ; SYS_vfork
syscall
test rax, rax
jz .child_code      ; ใน child
```

### 4.23 execve (syscall 59)

```c
// C prototype:
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

```nasm
; RDI = path to executable
; RSI = argv array (NULL-terminated array of string pointers)
; RDX = envp array (NULL-terminated array of string pointers)

section .data
    prog_path   db "/bin/echo", 0
    arg0        db "/bin/echo", 0
    arg1        db "Hello from execve!", 0

section .bss
    argv_arr:   resq 3  ; [arg0_ptr, arg1_ptr, NULL]

section .text
    lea rax, [arg0]
    mov [argv_arr], rax
    lea rax, [arg1]
    mov [argv_arr+8], rax
    mov qword [argv_arr+16], 0  ; NULL terminator

    mov rax, 59         ; SYS_execve
    lea rdi, [prog_path]
    lea rsi, [argv_arr]
    mov rdx, 0          ; envp = NULL (inherit)
    syscall
    ; ถ้ากลับมาแสดงว่า error
```

### 4.24 exit (syscall 60)

```nasm
; RDI = exit status code

mov rax, 60         ; SYS_exit
mov rdi, 0          ; exit code = 0 (success)
syscall             ; ไม่กลับมา
```

### 4.25 wait4 (syscall 61)

```c
// C prototype:
pid_t wait4(pid_t pid, int *wstatus, int options, struct rusage *rusage);
```

```nasm
; รอ child process

section .bss
    child_status: resd 1

section .text
    mov rax, 61         ; SYS_wait4
    mov rdi, -1         ; pid = -1 (รอ child ใดก็ได้)
    lea rsi, [child_status]
    mov rdx, 0          ; options = 0 (blocking)
    mov r10, 0          ; rusage = NULL
    syscall             ; RAX = child PID
```

### 4.26 kill (syscall 62)

```c
// C prototype:
int kill(pid_t pid, int sig);
```

```nasm
; ส่ง signal ไปยัง process

mov rax, 62         ; SYS_kill
mov rdi, r12        ; pid
mov rsi, 15         ; SIGTERM
syscall
```

### 4.27 socket, bind, listen, accept, connect (syscall 41-50)

```nasm
; socket families:
; AF_INET  = 2 (IPv4)
; AF_INET6 = 10 (IPv6)
; AF_UNIX  = 1 (Unix domain)

; socket types:
; SOCK_STREAM = 1 (TCP)
; SOCK_DGRAM  = 2 (UDP)

; สร้าง TCP socket:
mov rax, 41         ; SYS_socket
mov rdi, 2          ; AF_INET
mov rsi, 1          ; SOCK_STREAM
mov rdx, 0          ; protocol = 0 (auto)
syscall             ; RAX = socket fd
```

### 4.28 clone (syscall 56)

```nasm
; clone ใช้สำหรับสร้าง threads (pthreads ใช้ clone)
; flags ที่สำคัญ:
; CLONE_VM        = 0x100  (share memory space)
; CLONE_FS        = 0x200  (share filesystem)
; CLONE_FILES     = 0x400  (share file descriptors)
; CLONE_SIGHAND   = 0x800  (share signal handlers)
; CLONE_THREAD    = 0x10000 (ทำให้เป็น thread)

; RDI = clone flags
; RSI = stack pointer สำหรับ new thread
; RDX = parent_tidptr
; R10 = child_tidptr
; R8  = TLS pointer

mov rax, 56         ; SYS_clone
```

### 4.29 futex (syscall 202)

```c
// C prototype (simplified):
int futex(int *uaddr, int futex_op, int val, ...);
```

```nasm
; FUTEX operations:
; FUTEX_WAIT    = 0
; FUTEX_WAKE    = 1
; FUTEX_WAIT_PRIVATE = 128
; FUTEX_WAKE_PRIVATE = 129

; รอจนกว่า *uaddr != expected_val:
mov rax, 202        ; SYS_futex
lea rdi, [futex_var]
mov rsi, 128        ; FUTEX_WAIT_PRIVATE
mov rdx, 0          ; expected value
mov r10, 0          ; timeout = NULL (infinite)
syscall
```

### 4.30 epoll_create1, epoll_ctl, epoll_wait (syscall 291, 233, 232)

```nasm
; epoll: event notification สำหรับ I/O multiplexing

; สร้าง epoll instance:
mov rax, 291        ; SYS_epoll_create1
mov rdi, 0          ; flags = 0
syscall             ; RAX = epoll fd

; struct epoll_event:
; offset 0: events (4 bytes) - EPOLLIN=1, EPOLLOUT=4, EPOLLET=0x80000000
; offset 4: data (8 bytes union - ใช้ .fd หรือ .ptr)

; EPOLLIN  = 1    (data available to read)
; EPOLLOUT = 4    (ready to write)
; EPOLLERR = 8    (error)
; EPOLLHUP = 16   (hangup)
; EPOLLET  = 0x80000000 (edge-triggered)

; เพิ่ม fd ไปยัง epoll:
; EPOLL_CTL_ADD = 1
; EPOLL_CTL_MOD = 2
; EPOLL_CTL_DEL = 3

mov rax, 233        ; SYS_epoll_ctl
mov rdi, epoll_fd   ; epoll fd
mov rsi, 1          ; EPOLL_CTL_ADD
mov rdx, client_fd  ; fd ที่ต้องการ monitor
lea r10, [ev_struct]
syscall

; รอ events:
; RDI = epoll fd
; RSI = pointer ไปยัง epoll_event array
; RDX = maxevents
; R10 = timeout ms (-1 = infinite)

mov rax, 232        ; SYS_epoll_wait
mov rdi, epoll_fd
lea rsi, [events_arr]
mov rdx, 64         ; maxevents = 64
mov r10, -1         ; timeout = infinite
syscall             ; RAX = number of ready fds
```

---

## 5. โปรแกรมตัวอย่างที่สมบูรณ์

### 5.1 echo - พิมพ์ arguments ออก stdout

```nasm
; echo.asm - clone ของ /bin/echo
; ใช้: nasm -f elf64 echo.asm -o echo.o && ld echo.o -o echo
; รัน: ./echo Hello World

global _start

section .text

_start:
    ; [rsp]    = argc
    ; [rsp+8]  = argv[0] (program name)
    ; [rsp+16] = argv[1] (first argument)
    ; ...

    mov rbx, [rsp]      ; rbx = argc
    lea r12, [rsp+8]    ; r12 = pointer ไปยัง argv array

    ; ข้าม argv[0] (program name)
    inc r12             ; เลื่อน pointer ไปยัง argv[1] (เพิ่ม index)
    dec rbx             ; ลด argc (ไม่นับ argv[0])
    ; หมายเหตุ: r12 เป็น pointer ของ pointer ต้องเพิ่ม 8 bytes
    sub r12, 8          ; reset
    add r12, 8          ; argv[1]
    dec rbx             ; argc - 1

    test rbx, rbx
    jz .done            ; ถ้าไม่มี argument ข้ามไป done

    mov r13, 0          ; index = 0

.print_args:
    cmp r13, rbx
    jge .print_newline  ; พิมพ์ newline แล้วจบ

    ; เพิ่ม space ระหว่าง arguments
    test r13, r13
    jz .no_space

    ; พิมพ์ space
    push rax            ; save registers
    push rcx
    push r11
    mov rax, 1          ; SYS_write
    mov rdi, 1          ; stdout
    lea rsi, [space]    ; space character
    mov rdx, 1
    syscall
    pop r11
    pop rcx
    pop rax

.no_space:
    ; หา pointer ไปยัง argument string
    mov r14, [r12 + r13*8]  ; argv[index]

    ; หาความยาว string
    mov r15, r14        ; r15 = current position
.strlen:
    cmp byte [r15], 0
    je .found_len
    inc r15
    jmp .strlen
.found_len:
    sub r15, r14        ; r15 = length

    ; พิมพ์ argument
    push rax
    push rcx
    push r11
    mov rax, 1          ; SYS_write
    mov rdi, 1          ; stdout
    mov rsi, r14        ; string pointer
    mov rdx, r15        ; length
    syscall
    pop r11
    pop rcx
    pop rax

    inc r13
    jmp .print_args

.print_newline:
    mov rax, 1          ; SYS_write
    mov rdi, 1          ; stdout
    lea rsi, [newline]
    mov rdx, 1
    syscall

.done:
    mov rax, 60         ; SYS_exit
    mov rdi, 0
    syscall

section .data
    space   db ' '
    newline db 10
```

---

### 5.2 cat - อ่านไฟล์และแสดงออก stdout

```nasm
; cat.asm - อ่านไฟล์และแสดงเนื้อหา
; ใช้: nasm -f elf64 cat.asm -o cat.o && ld cat.o -o cat
; รัน: ./cat /etc/hostname

global _start

section .bss
    buffer: resb 4096   ; 4KB read buffer

section .text

_start:
    mov rbx, [rsp]      ; argc
    lea r12, [rsp+8]    ; argv

    ; ตรวจสอบ arguments
    cmp rbx, 2
    jl .read_stdin      ; ถ้าไม่มี argument อ่านจาก stdin

    ; เปิดไฟล์ที่ระบุ
    mov r13, 1          ; index = 1 (ข้าม argv[0])

.next_file:
    cmp r13, rbx
    jge .done

    ; เปิดไฟล์
    mov r14, [r12 + r13*8]  ; argv[r13]

    mov rax, 2          ; SYS_open
    mov rdi, r14        ; filename
    mov rsi, 0          ; O_RDONLY
    mov rdx, 0
    syscall

    test rax, rax
    js .open_error      ; error ถ้า negative

    mov r15, rax        ; r15 = file fd

    ; อ่านและพิมพ์ loop
.read_loop:
    mov rax, 0          ; SYS_read
    mov rdi, r15        ; fd
    lea rsi, [buffer]
    mov rdx, 4096
    syscall

    test rax, rax
    jz .file_done       ; EOF ถ้า 0 bytes
    js .read_error      ; error ถ้า negative

    ; พิมพ์ bytes ที่อ่านได้
    mov rdx, rax        ; bytes to write = bytes read
    mov rax, 1          ; SYS_write
    mov rdi, 1          ; stdout
    lea rsi, [buffer]
    syscall

    jmp .read_loop

.file_done:
    ; ปิดไฟล์
    mov rax, 3          ; SYS_close
    mov rdi, r15
    syscall

    inc r13
    jmp .next_file

.read_stdin:
    ; อ่านจาก stdin
    mov r15, 0          ; fd = stdin
    jmp .read_loop

.done:
    mov rax, 60         ; SYS_exit
    xor rdi, rdi
    syscall

.open_error:
    ; พิมพ์ error message
    mov rax, 1
    mov rdi, 2          ; stderr
    lea rsi, [err_open]
    mov rdx, err_open_len
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

.read_error:
    mov rax, 60
    mov rdi, 1
    syscall

section .data
    err_open        db "cat: ไม่สามารถเปิดไฟล์ได้", 10
    err_open_len    equ $ - err_open
```

---

### 5.3 wc - นับ lines, words, characters

```nasm
; wc.asm - นับ lines, words, characters
; ใช้: nasm -f elf64 wc.asm -o wc.o && ld wc.o -o wc
; รัน: ./wc file.txt

global _start

section .bss
    buffer:     resb 65536  ; 64KB buffer
    line_str:   resb 32
    word_str:   resb 32
    char_str:   resb 32

section .data
    space   db ' '
    newline db 10

section .text

; ฟังก์ชัน: แปลง integer เป็น string
; input: RAX = number, RDI = buffer pointer
; output: RCX = length
int_to_str:
    push rbx
    push rdx
    mov rbx, rdi        ; save buffer pointer
    add rdi, 20         ; ชี้ไปท้าย buffer
    mov byte [rdi], 0   ; null terminator
    mov rcx, 0          ; count = 0

    test rax, rax
    jnz .convert_loop
    ; กรณีพิเศษ: number = 0
    dec rdi
    mov byte [rdi], '0'
    inc rcx
    jmp .done_convert

.convert_loop:
    test rax, rax
    jz .done_convert
    xor rdx, rdx
    mov rbx, 10
    div rbx             ; rax = quotient, rdx = remainder
    dec rdi
    add dl, '0'
    mov [rdi], dl
    inc rcx
    jmp .convert_loop

.done_convert:
    ; copy ไปยัง buffer
    push rsi
    mov rsi, rdi
    mov rdi, rbx        ; original buffer pointer
    ; rcx มี length แล้ว
    push rcx
    cld
    rep movsb
    pop rcx
    pop rsi
    pop rdx
    pop rbx
    ret

_start:
    mov rbx, [rsp]      ; argc
    lea r12, [rsp+8]    ; argv

    ; ตรวจสอบ arguments
    cmp rbx, 2
    jl .usage_error

    ; เปิดไฟล์
    mov r13, [r12+8]    ; argv[1]
    mov rax, 2          ; SYS_open
    mov rdi, r13
    xor rsi, rsi        ; O_RDONLY
    xor rdx, rdx
    syscall

    test rax, rax
    js .open_error

    mov r14, rax        ; file fd

    ; counters
    xor r15, r15        ; lines = 0
    xor rbp, rbp        ; words = 0
    xor rbx, rbx        ; chars = 0
    mov r9, 0           ; in_word = false

.read_loop:
    mov rax, 0          ; SYS_read
    mov rdi, r14
    lea rsi, [buffer]
    mov rdx, 65536
    syscall

    test rax, rax
    jz .print_results
    js .done

    mov r8, rax         ; bytes read
    xor rcx, rcx        ; byte index = 0

.process_byte:
    cmp rcx, r8
    jge .read_loop

    movzx eax, byte [buffer + rcx]
    inc rcx
    inc rbx             ; chars++

    ; ตรวจสอบ newline
    cmp al, 10          ; '\n'
    jne .check_word
    inc r15             ; lines++

.check_word:
    ; ตรวจสอบ whitespace (space=32, tab=9, newline=10, CR=13)
    cmp al, 32
    je .is_space
    cmp al, 9
    je .is_space
    cmp al, 10
    je .is_space
    cmp al, 13
    je .is_space

    ; ไม่ใช่ space - ตรวจสอบว่าเพิ่ง start word
    test r9, r9
    jnz .process_byte   ; ถ้าอยู่ใน word แล้ว continue
    inc rbp             ; words++
    mov r9, 1           ; in_word = true
    jmp .process_byte

.is_space:
    mov r9, 0           ; in_word = false
    jmp .process_byte

.print_results:
    ; ปิดไฟล์
    mov rax, 3
    mov rdi, r14
    syscall

    ; พิมพ์ lines
    mov rax, r15
    lea rdi, [line_str]
    call int_to_str
    ; แสดงผล
    mov rax, 1
    mov rdi, 1
    lea rsi, [line_str]
    mov rdx, rcx
    syscall

    ; space
    mov rax, 1
    mov rdi, 1
    lea rsi, [space]
    mov rdx, 1
    syscall

    ; พิมพ์ words
    mov rax, rbp
    lea rdi, [word_str]
    call int_to_str
    mov rax, 1
    mov rdi, 1
    lea rsi, [word_str]
    mov rdx, rcx
    syscall

    ; space
    mov rax, 1
    mov rdi, 1
    lea rsi, [space]
    mov rdx, 1
    syscall

    ; พิมพ์ chars
    mov rax, rbx
    lea rdi, [char_str]
    call int_to_str
    mov rax, 1
    mov rdi, 1
    lea rsi, [char_str]
    mov rdx, rcx
    syscall

    ; space + filename + newline
    mov rax, 1
    mov rdi, 1
    lea rsi, [space]
    mov rdx, 1
    syscall

    ; filename (r13)
    mov r8, r13
.fn_len:
    cmp byte [r8], 0
    je .fn_done
    inc r8
    jmp .fn_len
.fn_done:
    sub r8, r13
    mov rax, 1
    mov rdi, 1
    mov rsi, r13
    mov rdx, r8
    syscall

    mov rax, 1
    mov rdi, 1
    lea rsi, [newline]
    mov rdx, 1
    syscall

.done:
    mov rax, 60
    xor rdi, rdi
    syscall

.open_error:
    mov rax, 60
    mov rdi, 1
    syscall

.usage_error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

### 5.4 cp - คัดลอกไฟล์

```nasm
; cp.asm - คัดลอกไฟล์
; ใช้: nasm -f elf64 cp.asm -o cp.o && ld cp.o -o cp
; รัน: ./cp source.txt dest.txt

global _start

section .bss
    buffer: resb 65536  ; 64KB buffer

section .data
    err_usage   db "usage: cp <source> <dest>", 10
    err_usage_l equ $ - err_usage
    err_open    db "cp: ไม่สามารถเปิดไฟล์ต้นฉบับ", 10
    err_open_l  equ $ - err_open
    err_create  db "cp: ไม่สามารถสร้างไฟล์ปลายทาง", 10
    err_create_l equ $ - err_create

section .text

_start:
    mov rbx, [rsp]      ; argc
    lea r12, [rsp+8]    ; argv

    ; ต้องการ 2 arguments
    cmp rbx, 3
    jne .usage_error

    ; เปิด source file
    mov rdi, [r12+8]    ; argv[1] = source
    mov rax, 2          ; SYS_open
    xor rsi, rsi        ; O_RDONLY = 0
    xor rdx, rdx
    syscall

    test rax, rax
    js .open_error

    mov r14, rax        ; source fd

    ; สร้าง/เปิด destination file
    ; O_WRONLY|O_CREAT|O_TRUNC = 1|64|512 = 577
    mov rdi, [r12+16]   ; argv[2] = dest
    mov rax, 2          ; SYS_open
    mov rsi, 577        ; O_WRONLY|O_CREAT|O_TRUNC
    mov rdx, 0644q      ; permissions = 0644 (octal)
    syscall

    test rax, rax
    js .create_error

    mov r15, rax        ; dest fd

.copy_loop:
    ; อ่านจาก source
    mov rax, 0          ; SYS_read
    mov rdi, r14
    lea rsi, [buffer]
    mov rdx, 65536
    syscall

    test rax, rax
    jz .copy_done       ; EOF
    js .copy_error

    ; เขียนไปยัง dest
    mov r13, rax        ; bytes to write

.write_loop:
    ; อาจต้องเขียนหลายครั้งถ้า write เขียนได้ไม่ครบ
    lea rsi, [buffer]
    mov rdx, r13

    mov rax, 1          ; SYS_write
    mov rdi, r15
    syscall

    test rax, rax
    js .write_error

    sub r13, rax        ; ลด bytes ที่ยังต้องเขียน
    jnz .write_partial

    jmp .copy_loop

.write_partial:
    ; write ยังไม่เสร็จ ต้องเขียนต่อ
    ; (ปรับ pointer แต่ในตัวอย่างนี้เราใช้ buffer เดิม)
    jmp .write_loop

.copy_done:
    ; ปิดทั้งสอง fd
    mov rax, 3
    mov rdi, r14
    syscall

    mov rax, 3
    mov rdi, r15
    syscall

    mov rax, 60         ; SYS_exit
    xor rdi, rdi
    syscall

.usage_error:
    mov rax, 1
    mov rdi, 2
    lea rsi, [err_usage]
    mov rdx, err_usage_l
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

.open_error:
    mov rax, 1
    mov rdi, 2
    lea rsi, [err_open]
    mov rdx, err_open_l
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

.create_error:
    mov rax, 3
    mov rdi, r14
    syscall
    mov rax, 1
    mov rdi, 2
    lea rsi, [err_create]
    mov rdx, err_create_l
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

.copy_error:
.write_error:
    mov rax, 3
    mov rdi, r14
    syscall
    mov rax, 3
    mov rdi, r15
    syscall
    mov rax, 60
    mov rdi, 1
    syscall
```

---

### 5.5 Simple HTTP Server

```nasm
; http_server.asm - Simple HTTP server on port 8080
; ใช้: nasm -f elf64 http_server.asm -o http_server.o && ld http_server.o -o http_server
; รัน: ./http_server  แล้วเปิด http://localhost:8080

global _start

; ค่าคงที่
%define AF_INET         2
%define SOCK_STREAM     1
%define SOL_SOCKET      1
%define SO_REUSEADDR    2
%define BACKLOG         10
%define PORT            8080
%define BUFFER_SIZE     4096

section .bss
    sock_fd:    resd 1          ; server socket fd
    client_fd:  resd 1          ; client socket fd
    client_addr: resb 16        ; struct sockaddr_in (client)
    addr_len:   resd 1          ; length of client addr
    recv_buf:   resb BUFFER_SIZE ; receive buffer
    send_buf:   resb BUFFER_SIZE ; send buffer

section .data
    ; struct sockaddr_in สำหรับ bind
    ; sa_family = AF_INET (2, little-endian 2 bytes)
    ; sin_port  = 8080 in network byte order = 0x901F
    ; sin_addr  = 0.0.0.0 = 0
    server_addr:
        dw  2           ; AF_INET
        dw  0x901F      ; port 8080 in big-endian (htons(8080))
        dd  0           ; INADDR_ANY = 0.0.0.0
        dq  0           ; padding
    server_addr_len equ $ - server_addr

    ; HTTP response
    http_200:
        db "HTTP/1.1 200 OK", 13, 10
        db "Content-Type: text/html; charset=utf-8", 13, 10
        db "Connection: close", 13, 10
        db 13, 10
    http_200_len equ $ - http_200

    html_body:
        db "<!DOCTYPE html>", 10
        db "<html><head><title>Assembly HTTP Server</title></head>", 10
        db "<body>", 10
        db "<h1>Hello from x86-64 Assembly!</h1>", 10
        db "<p>This HTTP server was written in pure assembly language.</p>", 10
        db "<p>Syscalls used: socket, bind, listen, accept, recv, send, close</p>", 10
        db "</body></html>", 10
    html_body_len equ $ - html_body

    ; int setsockopt option value = 1
    opt_one: dd 1

    ; startup message
    start_msg   db "HTTP Server listening on port 8080...", 10
    start_msg_l equ $ - start_msg

    ; connection message
    conn_msg    db "Client connected!", 10
    conn_msg_l  equ $ - conn_msg

section .text

_start:
    ; พิมพ์ startup message
    mov rax, 1
    mov rdi, 1
    lea rsi, [start_msg]
    mov rdx, start_msg_l
    syscall

    ;--- สร้าง socket ---
    ; socket(AF_INET, SOCK_STREAM, 0)
    mov rax, 41         ; SYS_socket
    mov rdi, AF_INET    ; IPv4
    mov rsi, SOCK_STREAM ; TCP
    xor rdx, rdx        ; protocol = 0
    syscall

    test rax, rax
    js .error_exit
    mov [sock_fd], eax  ; เก็บ socket fd

    ;--- setsockopt SO_REUSEADDR ---
    ; setsockopt(sock_fd, SOL_SOCKET, SO_REUSEADDR, &1, 4)
    mov rax, 54         ; SYS_setsockopt
    mov edi, [sock_fd]
    mov rsi, SOL_SOCKET
    mov rdx, SO_REUSEADDR
    lea r10, [opt_one]
    mov r8, 4
    syscall

    ;--- bind ---
    ; bind(sock_fd, &server_addr, sizeof(server_addr))
    mov rax, 49         ; SYS_bind
    mov edi, [sock_fd]
    lea rsi, [server_addr]
    mov rdx, 16         ; sizeof(struct sockaddr_in)
    syscall

    test rax, rax
    js .error_exit

    ;--- listen ---
    ; listen(sock_fd, BACKLOG)
    mov rax, 50         ; SYS_listen
    mov edi, [sock_fd]
    mov rsi, BACKLOG
    syscall

    test rax, rax
    js .error_exit

.accept_loop:
    ;--- accept ---
    ; accept(sock_fd, &client_addr, &addr_len)
    mov dword [addr_len], 16    ; ตั้งค่าเริ่มต้น addr_len

    mov rax, 43         ; SYS_accept
    mov edi, [sock_fd]
    lea rsi, [client_addr]
    lea rdx, [addr_len]
    syscall

    test rax, rax
    js .accept_loop     ; ถ้า error ลอง accept อีกครั้ง
    mov [client_fd], eax ; เก็บ client fd

    ; แจ้ง connection
    mov rax, 1
    mov rdi, 1
    lea rsi, [conn_msg]
    mov rdx, conn_msg_l
    syscall

    ;--- recv request ---
    ; recvfrom(client_fd, recv_buf, BUFFER_SIZE, 0, NULL, NULL)
    mov rax, 45         ; SYS_recvfrom
    mov edi, [client_fd]
    lea rsi, [recv_buf]
    mov rdx, BUFFER_SIZE
    xor r10, r10        ; flags = 0
    xor r8, r8          ; src_addr = NULL
    xor r9, r9          ; addrlen = NULL
    syscall

    ; ไม่ตรวจสอบ request ใด ๆ ส่ง response ทันที

    ;--- ส่ง HTTP header ---
    mov rax, 44         ; SYS_sendto
    mov edi, [client_fd]
    lea rsi, [http_200]
    mov rdx, http_200_len
    xor r10, r10        ; flags = 0
    xor r8, r8
    xor r9, r9
    syscall

    ;--- ส่ง HTML body ---
    mov rax, 44         ; SYS_sendto
    mov edi, [client_fd]
    lea rsi, [html_body]
    mov rdx, html_body_len
    xor r10, r10
    xor r8, r8
    xor r9, r9
    syscall

    ;--- close client connection ---
    mov rax, 3          ; SYS_close
    mov edi, [client_fd]
    syscall

    ; วนกลับรับ connection ใหม่
    jmp .accept_loop

.error_exit:
    ; ปิด socket ถ้าเปิดไว้แล้ว
    mov eax, [sock_fd]
    test eax, eax
    jz .final_exit
    mov rax, 3
    mov edi, [sock_fd]
    syscall

.final_exit:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 6. Advanced Topics

### 6.1 Error Numbers (errno)

```nasm
; เมื่อ syscall ล้มเหลว RAX มีค่า -errno
; errno ที่พบบ่อย:
; EPERM    = 1   (Operation not permitted)
; ENOENT   = 2   (No such file or directory)
; ESRCH    = 3   (No such process)
; EINTR    = 4   (Interrupted system call)
; EIO      = 5   (I/O error)
; ENXIO    = 6   (No such device or address)
; EBADF    = 9   (Bad file descriptor)
; ECHILD   = 10  (No child processes)
; EAGAIN   = 11  (Try again / Resource temporarily unavailable)
; ENOMEM   = 12  (Out of memory)
; EACCES   = 13  (Permission denied)
; EFAULT   = 14  (Bad address)
; EBUSY    = 16  (Device or resource busy)
; EEXIST   = 17  (File exists)
; EXDEV    = 18  (Cross-device link)
; ENODEV   = 19  (No such device)
; EISDIR   = 21  (Is a directory)
; EINVAL   = 22  (Invalid argument)
; EMFILE   = 24  (Too many open files)
; ENOSPC   = 28  (No space left on device)
; ERANGE   = 34  (Math result not representable)
; EWOULDBLOCK = 11 (= EAGAIN)
; EINPROGRESS = 115 (Operation now in progress)
; ECONNREFUSED = 111 (Connection refused)

; การแปลง errno เป็น positive:
; mov rbx, rax    ; save return value
; neg rbx         ; rbx = errno (positive)
; cmp rbx, 4096
; jb .handle_error
```

### 6.2 Signal Handling Example

```nasm
; signal_handler.asm - จัดการ SIGINT (Ctrl+C)

global _start

section .bss
    sigact:     resb 152    ; struct sigaction (152 bytes)
    running:    resb 1      ; flag: ยังทำงานอยู่หรือไม่

section .data
    msg_running db "กำลังทำงาน... กด Ctrl+C เพื่อหยุด", 10
    msg_running_l equ $ - msg_running
    msg_caught  db "รับ SIGINT แล้ว! กำลังหยุด...", 10
    msg_caught_l equ $ - msg_caught
    msg_done    db "หยุดแล้ว", 10
    msg_done_l  equ $ - msg_done

section .text

; SIGINT handler function
sigint_handler:
    ; บันทึก flag ว่าได้รับ signal แล้ว
    mov byte [running], 0

    ; พิมพ์ข้อความ
    ; หมายเหตุ: ใน signal handler ควรใช้แค่ async-signal-safe functions
    mov rax, 1
    mov rdi, 1
    lea rsi, [msg_caught]
    mov rdx, msg_caught_l
    syscall
    ret

_start:
    ; ติดตั้ง SIGINT handler
    ; เคลียร์ struct sigaction ก่อน
    xor rax, rax
    mov rcx, 19             ; 152 bytes / 8 = 19 qwords
    lea rdi, [sigact]
    rep stosq

    ; ตั้ง sa_handler
    lea rax, [sigint_handler]
    mov [sigact], rax

    ; ตั้ง sa_flags = SA_RESTORER (4) | SA_RESTART (0x10000000)
    ; จริง ๆ ต้องตั้ง SA_RESTORER และให้ restorer function ด้วย
    ; แต่ใน kernel ใหม่ ๆ มักไม่จำเป็น

    ; ติดตั้ง handler
    mov rax, 13             ; SYS_rt_sigaction
    mov rdi, 2              ; SIGINT
    lea rsi, [sigact]
    xor rdx, rdx            ; oldact = NULL
    mov r10, 8              ; sigset_t size
    syscall

    ; ตั้งค่า running = 1
    mov byte [running], 1

    ; แสดงข้อความ
    mov rax, 1
    mov rdi, 1
    lea rsi, [msg_running]
    mov rdx, msg_running_l
    syscall

.wait_loop:
    ; ตรวจสอบ flag
    cmp byte [running], 0
    je .exit_program

    ; pause: รอ signal (ใช้ nanosleep แทน pause เพื่อไม่ block ตลอด)
    ; sleep 100ms
    sub rsp, 16
    mov qword [rsp], 0          ; tv_sec = 0
    mov qword [rsp+8], 100000000 ; tv_nsec = 100ms

    mov rax, 35             ; SYS_nanosleep
    mov rdi, rsp
    xor rsi, rsi
    syscall

    add rsp, 16

    jmp .wait_loop

.exit_program:
    mov rax, 1
    mov rdi, 1
    lea rsi, [msg_done]
    mov rdx, msg_done_l
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### 6.3 Memory Mapping ไฟล์ (mmap file)

```nasm
; mmap_file.asm - อ่านไฟล์โดยใช้ mmap

global _start

section .bss
    stat_buf:   resb 144    ; struct stat

section .data
    filename    db "test.txt", 0
    newline     db 10

section .text

_start:
    ; เปิดไฟล์
    mov rax, 2          ; SYS_open
    lea rdi, [filename]
    xor rsi, rsi        ; O_RDONLY
    xor rdx, rdx
    syscall

    test rax, rax
    js .error

    mov r12, rax        ; save fd

    ; หา file size ด้วย fstat
    mov rax, 5          ; SYS_fstat
    mov rdi, r12
    lea rsi, [stat_buf]
    syscall

    ; st_size อยู่ที่ offset 48
    mov r13, [stat_buf + 48]    ; file size

    test r13, r13
    jz .close_and_exit

    ; mmap ไฟล์
    ; mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0)
    mov rax, 9          ; SYS_mmap
    xor rdi, rdi        ; addr = NULL
    mov rsi, r13        ; length = file size
    mov rdx, 1          ; PROT_READ
    mov r10, 2          ; MAP_PRIVATE
    mov r8, r12         ; fd
    xor r9, r9          ; offset = 0
    syscall

    test rax, rax
    js .close_and_exit

    mov r14, rax        ; save mmap pointer

    ; เขียนเนื้อหาออก stdout
    mov rax, 1          ; SYS_write
    mov rdi, 1
    mov rsi, r14
    mov rdx, r13
    syscall

    ; munmap
    mov rax, 11         ; SYS_munmap
    mov rdi, r14
    mov rsi, r13
    syscall

.close_and_exit:
    ; ปิดไฟล์
    mov rax, 3          ; SYS_close
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

### 6.4 Fork และ Exec ตัวอย่างสมบูรณ์

```nasm
; fork_exec.asm - สาธิตการใช้ fork + execve + wait4

global _start

section .bss
    child_status:   resd 1

section .data
    prog        db "/bin/ls", 0
    arg0        db "/bin/ls", 0
    arg1        db "-la", 0
    arg2        db "/tmp", 0

    parent_msg  db "Parent: กำลัง fork...", 10
    parent_ml   equ $ - parent_msg
    child_msg   db "Child: กำลัง exec ls...", 10
    child_ml    equ $ - child_msg
    done_msg    db "Parent: child เสร็จแล้ว", 10
    done_ml     equ $ - done_msg

section .bss
    argv_arr:   resq 4  ; [arg0, arg1, arg2, NULL]

section .text

_start:
    ; เตรียม argv array
    lea rax, [arg0]
    mov [argv_arr], rax
    lea rax, [arg1]
    mov [argv_arr+8], rax
    lea rax, [arg2]
    mov [argv_arr+16], rax
    mov qword [argv_arr+24], 0  ; NULL

    ; พิมพ์ parent message
    mov rax, 1
    mov rdi, 1
    lea rsi, [parent_msg]
    mov rdx, parent_ml
    syscall

    ; fork
    mov rax, 57         ; SYS_fork
    syscall

    test rax, rax
    jz .child           ; child process
    js .fork_error      ; error
    mov r12, rax        ; parent: save child PID

    ; === Parent Process ===
    ; รอ child
    mov rax, 61         ; SYS_wait4
    mov rdi, r12        ; wait for specific child
    lea rsi, [child_status]
    xor rdx, rdx        ; options = 0
    xor r10, r10        ; rusage = NULL
    syscall

    ; พิมพ์ done message
    mov rax, 1
    mov rdi, 1
    lea rsi, [done_msg]
    mov rdx, done_ml
    syscall

    ; exit
    mov rax, 60
    xor rdi, rdi
    syscall

.child:
    ; === Child Process ===
    ; พิมพ์ child message
    mov rax, 1
    mov rdi, 1
    lea rsi, [child_msg]
    mov rdx, child_ml
    syscall

    ; execve ls
    mov rax, 59         ; SYS_execve
    lea rdi, [prog]
    lea rsi, [argv_arr]
    xor rdx, rdx        ; envp = NULL
    syscall

    ; ถ้ากลับมาแสดงว่า execve ล้มเหลว
    mov rax, 60
    mov rdi, 1
    syscall

.fork_error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 7. Pipe และ I/O Redirection

### 7.1 Pipe Example

```nasm
; pipe_example.asm - สาธิตการใช้ pipe

global _start

section .bss
    pipefd:     resd 2      ; [read_fd, write_fd]
    buf:        resb 256

section .data
    msg         db "ข้อความผ่าน pipe!", 10
    msg_len     equ $ - msg
    prefix      db "Child รับได้: "
    prefix_len  equ $ - prefix

section .text

_start:
    ; สร้าง pipe
    mov rax, 22         ; SYS_pipe
    lea rdi, [pipefd]
    syscall

    test rax, rax
    js .error

    ; fork
    mov rax, 57         ; SYS_fork
    syscall

    test rax, rax
    jz .child
    js .error

    ; === Parent: เขียนเข้า pipe ===
    ; ปิด read end
    mov rax, 3
    mov edi, [pipefd]   ; close read fd
    syscall

    ; เขียน message
    mov rax, 1          ; SYS_write
    mov edi, [pipefd+4] ; write fd
    lea rsi, [msg]
    mov rdx, msg_len
    syscall

    ; ปิด write end (ส่ง EOF ไปยัง child)
    mov rax, 3
    mov edi, [pipefd+4]
    syscall

    ; รอ child
    mov rax, 61         ; SYS_wait4
    mov rdi, -1
    xor rsi, rsi
    xor rdx, rdx
    xor r10, r10
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall

.child:
    ; === Child: อ่านจาก pipe ===
    ; ปิด write end
    mov rax, 3
    mov edi, [pipefd+4]
    syscall

    ; พิมพ์ prefix
    mov rax, 1
    mov rdi, 1
    lea rsi, [prefix]
    mov rdx, prefix_len
    syscall

    ; อ่านจาก pipe และพิมพ์
.read_pipe:
    mov rax, 0          ; SYS_read
    mov edi, [pipefd]   ; read fd
    lea rsi, [buf]
    mov rdx, 256
    syscall

    test rax, rax
    jz .child_done
    js .error

    mov rdx, rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [buf]
    syscall

    jmp .read_pipe

.child_done:
    ; ปิด read end
    mov rax, 3
    mov edi, [pipefd]
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

## 8. Networking ตัวอย่างสมบูรณ์

### 8.1 TCP Client

```nasm
; tcp_client.asm - เชื่อมต่อไปยัง server
; รัน: ./tcp_client (จะ connect ไปยัง localhost:8080)

global _start

%define AF_INET     2
%define SOCK_STREAM 1

section .bss
    sock_fd:    resd 1
    recv_buf:   resb 4096

section .data
    ; struct sockaddr_in สำหรับ connect
    server_addr:
        dw  2           ; AF_INET
        dw  0x901F      ; port 8080 big-endian
        db  127, 0, 0, 1 ; 127.0.0.1 (localhost)
        dq  0           ; padding

    http_request:
        db "GET / HTTP/1.0", 13, 10
        db "Host: localhost:8080", 13, 10
        db 13, 10
    http_request_len equ $ - http_request

    connected_msg   db "เชื่อมต่อแล้ว! ส่ง HTTP request...", 10
    connected_ml    equ $ - connected_msg

section .text

_start:
    ; สร้าง socket
    mov rax, 41         ; SYS_socket
    mov rdi, AF_INET
    mov rsi, SOCK_STREAM
    xor rdx, rdx
    syscall

    test rax, rax
    js .error
    mov [sock_fd], eax

    ; connect
    mov rax, 42         ; SYS_connect
    mov edi, [sock_fd]
    lea rsi, [server_addr]
    mov rdx, 16
    syscall

    test rax, rax
    js .error

    ; แจ้ง connection สำเร็จ
    mov rax, 1
    mov rdi, 1
    lea rsi, [connected_msg]
    mov rdx, connected_ml
    syscall

    ; ส่ง HTTP request
    mov rax, 44         ; SYS_sendto
    mov edi, [sock_fd]
    lea rsi, [http_request]
    mov rdx, http_request_len
    xor r10, r10
    xor r8, r8
    xor r9, r9
    syscall

    ; รับ response
.recv_loop:
    mov rax, 45         ; SYS_recvfrom
    mov edi, [sock_fd]
    lea rsi, [recv_buf]
    mov rdx, 4096
    xor r10, r10
    xor r8, r8
    xor r9, r9
    syscall

    test rax, rax
    jle .done

    ; พิมพ์ response
    mov rdx, rax
    mov rax, 1
    mov rdi, 1
    lea rsi, [recv_buf]
    syscall

    jmp .recv_loop

.done:
    ; ปิด socket
    mov rax, 3
    mov edi, [sock_fd]
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

## 9. epoll Server ที่สมบูรณ์

```nasm
; epoll_server.asm - Non-blocking server ใช้ epoll
; รองรับ multiple connections พร้อมกัน

global _start

%define AF_INET         2
%define SOCK_STREAM     1
%define SOCK_NONBLOCK   2048    ; O_NONBLOCK สำหรับ socket
%define SOL_SOCKET      1
%define SO_REUSEADDR    2
%define EPOLL_CTL_ADD   1
%define EPOLL_CTL_DEL   3
%define EPOLLIN         1
%define EPOLLET         0x80000000
%define MAX_EVENTS      64
%define PORT            8888
%define BUF_SIZE        1024

section .bss
    server_fd:      resd 1
    epoll_fd:       resd 1
    events:         resb MAX_EVENTS * 12    ; epoll_event array (12 bytes each)
    recv_buf:       resb BUF_SIZE

section .data
    server_addr:
        dw  2               ; AF_INET
        dw  0xB822          ; port 8888 big-endian
        dd  0               ; INADDR_ANY
        dq  0
    opt_one:    dd 1
    ready_msg   db "epoll server พร้อมรับ connection บน port 8888", 10
    ready_ml    equ $ - ready_msg
    http_resp   db "HTTP/1.1 200 OK", 13, 10, "Content-Length: 5", 13, 10, 13, 10, "Hello"
    http_resp_l equ $ - http_resp

section .text

_start:
    ; พิมพ์ ready message
    mov rax, 1
    mov rdi, 1
    lea rsi, [ready_msg]
    mov rdx, ready_ml
    syscall

    ; สร้าง non-blocking socket
    mov rax, 41             ; SYS_socket
    mov rdi, AF_INET
    mov rsi, SOCK_STREAM | SOCK_NONBLOCK    ; non-blocking
    xor rdx, rdx
    syscall
    test rax, rax
    js .fatal
    mov [server_fd], eax

    ; setsockopt SO_REUSEADDR
    mov rax, 54             ; SYS_setsockopt
    mov edi, [server_fd]
    mov rsi, SOL_SOCKET
    mov rdx, SO_REUSEADDR
    lea r10, [opt_one]
    mov r8, 4
    syscall

    ; bind
    mov rax, 49
    mov edi, [server_fd]
    lea rsi, [server_addr]
    mov rdx, 16
    syscall
    test rax, rax
    js .fatal

    ; listen
    mov rax, 50
    mov edi, [server_fd]
    mov rsi, 128
    syscall
    test rax, rax
    js .fatal

    ; สร้าง epoll instance
    mov rax, 291            ; SYS_epoll_create1
    xor rdi, rdi
    syscall
    test rax, rax
    js .fatal
    mov [epoll_fd], eax

    ; เพิ่ม server_fd ไปยัง epoll
    ; struct epoll_event: events(4) + data.fd(4) = 8 bytes (บน stack)
    sub rsp, 12
    mov dword [rsp], EPOLLIN    ; events
    mov eax, [server_fd]
    mov dword [rsp+4], eax      ; data.fd

    mov rax, 233            ; SYS_epoll_ctl
    mov edi, [epoll_fd]
    mov rsi, EPOLL_CTL_ADD
    mov edx, [server_fd]
    mov r10, rsp
    syscall
    add rsp, 12

.event_loop:
    ; epoll_wait
    mov rax, 232            ; SYS_epoll_wait
    mov edi, [epoll_fd]
    lea rsi, [events]
    mov rdx, MAX_EVENTS
    mov r10, -1             ; timeout = infinite
    syscall

    test rax, rax
    jle .event_loop         ; 0 หรือ error -> loop ใหม่

    mov r12, rax            ; r12 = number of events
    xor r13, r13            ; event index = 0

.process_events:
    cmp r13, r12
    jge .event_loop

    ; อ่าน event: struct epoll_event อยู่ที่ events + r13*12
    ; offset 0: events (4 bytes)
    ; offset 4: data.fd (4 bytes)
    imul r14, r13, 12
    lea r15, [events]
    add r15, r14

    mov eax, [r15+4]        ; data.fd
    mov ecx, [server_fd]

    cmp eax, ecx
    je .new_connection      ; ถ้า server_fd มี event = connection ใหม่

    ; ถ้า client fd มี event
    mov r11d, eax           ; save client fd
    jmp .handle_client

.new_connection:
    ; accept non-blocking connection
    mov rax, 43             ; SYS_accept
    mov edi, [server_fd]
    xor rsi, rsi
    xor rdx, rdx
    syscall

    test rax, rax
    js .next_event

    ; เพิ่ม client fd ไปยัง epoll
    mov r11d, eax           ; client fd

    sub rsp, 12
    mov dword [rsp], EPOLLIN | EPOLLET  ; edge-triggered
    mov [rsp+4], r11d
    mov rax, 233            ; SYS_epoll_ctl
    mov edi, [epoll_fd]
    mov rsi, EPOLL_CTL_ADD
    mov edx, r11d
    mov r10, rsp
    syscall
    add rsp, 12

    jmp .next_event

.handle_client:
    ; อ่าน request
    mov rax, 0              ; SYS_read
    mov edi, r11d
    lea rsi, [recv_buf]
    mov rdx, BUF_SIZE
    syscall

    test rax, rax
    jle .close_client

    ; ส่ง response
    mov rax, 1              ; SYS_write
    mov edi, r11d
    lea rsi, [http_resp]
    mov rdx, http_resp_l
    syscall

.close_client:
    ; ลบ fd จาก epoll และ close
    mov rax, 233
    mov edi, [epoll_fd]
    mov rsi, EPOLL_CTL_DEL
    mov edx, r11d
    xor r10, r10
    syscall

    mov rax, 3
    mov edi, r11d
    syscall

.next_event:
    inc r13
    jmp .process_events

.fatal:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 10. Macro Helpers สำหรับ Syscalls

```nasm
; syscall_macros.inc - Macros ที่สะดวกสำหรับ syscalls

; Macro สำหรับ write string
%macro write_str 2
    ; %1 = string label, %2 = length
    mov rax, 1
    mov rdi, 1
    lea rsi, [%1]
    mov rdx, %2
    syscall
%endmacro

; Macro สำหรับ write string ไปยัง stderr
%macro write_err 2
    mov rax, 1
    mov rdi, 2
    lea rsi, [%1]
    mov rdx, %2
    syscall
%endmacro

; Macro สำหรับ exit
%macro exit_code 1
    mov rax, 60
    mov rdi, %1
    syscall
%endmacro

; Macro สำหรับ open file
%macro open_file 3
    ; %1 = filename, %2 = flags, %3 = mode
    mov rax, 2
    lea rdi, [%1]
    mov rsi, %2
    mov rdx, %3
    syscall
%endmacro

; Macro สำหรับ close
%macro close_fd 1
    mov rax, 3
    mov rdi, %1
    syscall
%endmacro

; ตัวอย่างการใช้ macros:
section .data
    hello   db "Hello, Macros!", 10
    hello_l equ $ - hello

section .text
global _start
_start:
    write_str hello, hello_l
    exit_code 0
```

---

## 11. สรุปตาราง Syscall Numbers ที่สำคัญ

| Syscall Number | Name | ใช้งาน |
|---------------|------|--------|
| 0 | read | อ่านข้อมูล |
| 1 | write | เขียนข้อมูล |
| 2 | open | เปิดไฟล์ |
| 3 | close | ปิด file descriptor |
| 4 | stat | ข้อมูล file (by path) |
| 5 | fstat | ข้อมูล file (by fd) |
| 7 | poll | รอ I/O events |
| 8 | lseek | เลื่อน file position |
| 9 | mmap | map memory |
| 10 | mprotect | เปลี่ยน memory protection |
| 11 | munmap | unmap memory |
| 12 | brk | เปลี่ยน program break |
| 13 | rt_sigaction | ติดตั้ง signal handler |
| 14 | rt_sigprocmask | เปลี่ยน signal mask |
| 16 | ioctl | device control |
| 17 | pread64 | อ่านที่ offset |
| 18 | pwrite64 | เขียนที่ offset |
| 19 | readv | scatter read |
| 20 | writev | gather write |
| 22 | pipe | สร้าง pipe |
| 32 | dup | duplicate fd |
| 33 | dup2 | duplicate fd to specific number |
| 35 | nanosleep | sleep ที่ความละเอียด nanosecond |
| 39 | getpid | get process ID |
| 41 | socket | สร้าง socket |
| 42 | connect | เชื่อมต่อ socket |
| 43 | accept | รับ connection |
| 44 | sendto | ส่งข้อมูลผ่าน socket |
| 45 | recvfrom | รับข้อมูลจาก socket |
| 49 | bind | bind socket ไปยัง address |
| 50 | listen | รอรับ connections |
| 56 | clone | สร้าง thread/process |
| 57 | fork | สร้าง child process |
| 58 | vfork | สร้าง child (share stack) |
| 59 | execve | เรียกใช้โปรแกรม |
| 60 | exit | จบโปรแกรม |
| 61 | wait4 | รอ child process |
| 62 | kill | ส่ง signal |
| 202 | futex | fast userspace mutex |
| 232 | epoll_wait | รอ epoll events |
| 233 | epoll_ctl | จัดการ epoll fd |
| 291 | epoll_create1 | สร้าง epoll instance |

---

## 12. Tips และ Best Practices

### 12.1 ตรวจสอบ Error เสมอ

```nasm
; อย่าลืมตรวจสอบ return value
syscall
cmp rax, 0
jl .handle_error    ; RAX < 0 หมายถึง error

; หรือใช้ test สำหรับตรวจ 0/-
syscall
test rax, rax
js .error           ; jump ถ้า sign bit set (negative)
```

### 12.2 Stack Alignment

```nasm
; ก่อนเรียก library functions (ไม่ใช่ syscall) ต้องทำ 16-byte align
; syscall ไม่ต้องการ alignment
; แต่ถ้าใช้ SSE/AVX อาจต้องการ 16/32 byte alignment

; ตรวจสอบ alignment:
test rsp, 0xF       ; ตรวจว่า lower 4 bits = 0
jnz .misaligned
```

### 12.3 Preserve Callee-saved Registers

```nasm
; Registers ที่ต้องเก็บ (callee-saved ตาม System V ABI):
; RBX, RBP, R12, R13, R14, R15

; Syscall ทำลาย: RCX, R11
; Syscall arguments: RAX, RDI, RSI, RDX, R10, R8, R9

; ดังนั้นถ้าต้องการค่าที่เก็บไว้หลัง syscall ใช้:
; RBX, RBP, R12-R15 (แต่ต้อง save/restore ถ้าใช้ใน function)
```

### 12.4 การ Debug Syscalls

```bash
# ใช้ strace เพื่อดู syscalls ที่โปรแกรมเรียก
strace ./my_program

# ดูเฉพาะ syscalls บางตัว
strace -e trace=read,write,open,close ./my_program

# ดู timing
strace -T ./my_program

# บันทึกลงไฟล์
strace -o strace_output.txt ./my_program
```

### 12.5 Compile และ Link

```bash
# สำหรับไฟล์ .asm ที่ใช้ syscall โดยตรง:
nasm -f elf64 program.asm -o program.o
ld program.o -o program

# ถ้าต้องการ debug symbols:
nasm -f elf64 -g -F dwarf program.asm -o program.o
ld program.o -o program

# ตรวจสอบ binary:
file program
readelf -h program
objdump -d program
```

---

## 13. ตัวอย่างโปรแกรมขนาดเล็กสมบูรณ์

### 13.1 Hello World (การ minimal)

```nasm
; hello_minimal.asm
; nasm -f elf64 hello_minimal.asm -o hello_minimal.o && ld hello_minimal.o -o hello_minimal

global _start

section .data
    msg db "สวัสดีชาวโลก! (Hello World in Thai)", 10
    msg_len equ $ - msg

section .text
_start:
    ; write(1, msg, msg_len)
    mov rax, 1          ; SYS_write = 1
    mov rdi, 1          ; fd = 1 (stdout)
    mov rsi, msg        ; buffer
    mov rdx, msg_len    ; count
    syscall

    ; exit(0)
    mov rax, 60         ; SYS_exit = 60
    xor rdi, rdi        ; status = 0
    syscall
```

### 13.2 อ่าน Input จาก User

```nasm
; read_input.asm - รับ input จาก user แล้วแสดงผล

global _start

section .bss
    input_buf:  resb 256

section .data
    prompt      db "กรุณาป้อนชื่อ: "
    prompt_l    equ $ - prompt
    greeting    db "สวัสดี, "
    greeting_l  equ $ - greeting
    newline     db 10

section .text

_start:
    ; แสดง prompt
    mov rax, 1
    mov rdi, 1
    lea rsi, [prompt]
    mov rdx, prompt_l
    syscall

    ; รับ input
    mov rax, 0          ; SYS_read
    mov rdi, 0          ; stdin
    lea rsi, [input_buf]
    mov rdx, 255
    syscall

    ; RAX = bytes read (รวม newline)
    mov r12, rax        ; save length

    ; ลบ newline ท้าย
    test r12, r12
    jz .done
    lea rdi, [input_buf]
    add rdi, r12
    dec rdi
    cmp byte [rdi], 10
    jne .no_newline
    dec r12             ; ลดความยาว

.no_newline:
    ; แสดง "สวัสดี, "
    mov rax, 1
    mov rdi, 1
    lea rsi, [greeting]
    mov rdx, greeting_l
    syscall

    ; แสดงชื่อ
    mov rax, 1
    mov rdi, 1
    lea rsi, [input_buf]
    mov rdx, r12
    syscall

    ; แสดง newline
    mov rax, 1
    mov rdi, 1
    lea rsi, [newline]
    mov rdx, 1
    syscall

.done:
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 14. การเรียก Syscall ผ่าน C (สำหรับเปรียบเทียบ)

```c
// syscall() function ใน C
#include <sys/syscall.h>
#include <unistd.h>

// เรียก write ผ่าน syscall()
long result = syscall(SYS_write, 1, "Hello\n", 6);

// เทียบเท่ากับ assembly:
// mov rax, 1  (SYS_write)
// mov rdi, 1  (stdout)
// mov rsi, "Hello\n"
// mov rdx, 6
// syscall
// ; RAX = result
```

---

## 15. Vfork ความแตกต่างจาก Fork

```nasm
; vfork_example.asm
; vfork: parent ถูก suspend จนกว่า child จะ _exit() หรือ execve()
; child ใช้ stack และ address space เดียวกับ parent
; ต้องระวัง: child ไม่ควรแก้ไข variables ของ parent

global _start

section .data
    parent_msg  db "Parent รอ...", 10
    parent_ml   equ $ - parent_msg
    child_msg   db "Child กำลัง exec...", 10
    child_ml    equ $ - child_msg

section .bss
    argv_arr:   resq 2

section .data
    prog        db "/bin/true", 0
    arg0        db "/bin/true", 0

section .text

_start:
    ; เตรียม argv
    lea rax, [arg0]
    mov [argv_arr], rax
    mov qword [argv_arr+8], 0

    ; vfork
    mov rax, 58         ; SYS_vfork
    syscall

    test rax, rax
    jz .child           ; child ถ้า 0
    js .error

    ; Parent รอโดยอัตโนมัติ (kernel suspend parent จนกว่า child exec/exit)
    mov rax, 1
    mov rdi, 1
    lea rsi, [parent_msg]
    mov rdx, parent_ml
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall

.child:
    mov rax, 1
    mov rdi, 1
    lea rsi, [child_msg]
    mov rdx, child_ml
    syscall

    ; exec ทันที
    mov rax, 59         ; SYS_execve
    lea rdi, [prog]
    lea rsi, [argv_arr]
    xor rdx, rdx
    syscall

    ; ถ้า execve ล้มเหลว ต้องใช้ _exit (ไม่ใช่ exit!) เพื่อไม่ flush buffers
    mov rax, 60
    mov rdi, 1
    syscall

.error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 16. Clone สำหรับ Thread Creation

```nasm
; clone_thread.asm - สร้าง thread ด้วย clone syscall

global _start

%define CLONE_VM        0x100
%define CLONE_FS        0x200
%define CLONE_FILES     0x400
%define CLONE_SIGHAND   0x800
%define CLONE_THREAD    0x10000
%define CLONE_SETTLS    0x80000
%define CLONE_CHILD_CLEARTID 0x200000
%define CLONE_PARENT_SETTID  0x100000

%define THREAD_FLAGS (CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD)

section .bss
    thread_stack:   resb 65536      ; 64KB stack สำหรับ thread
    thread_tid:     resd 1          ; TID ของ thread

section .data
    thread_msg  db "Thread กำลังทำงาน!", 10
    thread_ml   equ $ - thread_msg
    main_msg    db "Main thread รอ...", 10
    main_ml     equ $ - main_msg

section .text

; Thread function
thread_func:
    ; stack ใหม่เริ่มต้นที่นี่
    mov rax, 1
    mov rdi, 1
    lea rsi, [thread_msg]
    mov rdx, thread_ml
    syscall

    ; ออกจาก thread (ใช้ exit ปกติ)
    mov rax, 60
    xor rdi, rdi
    syscall

_start:
    ; แสดง main message
    mov rax, 1
    mov rdi, 1
    lea rsi, [main_msg]
    mov rdx, main_ml
    syscall

    ; คำนวณ top of thread stack (stack grows downward)
    lea rsi, [thread_stack + 65536]

    ; clone syscall
    ; RDI = flags
    ; RSI = child stack (top)
    ; RDX = parent_tidptr
    ; R10 = child_tidptr
    ; R8  = TLS

    mov rax, 56         ; SYS_clone
    mov rdi, THREAD_FLAGS
    ; RSI ยังชี้ไปยัง top of stack
    xor rdx, rdx        ; parent_tidptr = NULL
    xor r10, r10        ; child_tidptr = NULL
    xor r8, r8          ; TLS = NULL
    syscall

    test rax, rax
    js .error           ; error

    ; Note: ใน clone thread model, child เริ่มทำงานต่อจาก syscall เหมือนกัน
    ; ต่างจาก fork ที่ child เริ่มหลัง syscall เช่นกัน
    ; แต่ใน thread context RAX=0 ใน child
    jnz .parent_thread

    ; Thread ใหม่ (child) - เรียก thread function
    call thread_func

.parent_thread:
    ; รอเล็กน้อยให้ thread ทำงาน
    sub rsp, 16
    mov qword [rsp], 0
    mov qword [rsp+8], 500000000    ; 500ms

    mov rax, 35         ; SYS_nanosleep
    mov rdi, rsp
    xor rsi, rsi
    syscall
    add rsp, 16

    mov rax, 60
    xor rdi, rdi
    syscall

.error:
    mov rax, 60
    mov rdi, 1
    syscall
```

---

## 17. Futex สำหรับ Synchronization

```nasm
; futex_example.asm - ใช้ futex สำหรับ mutual exclusion

global _start

%define FUTEX_WAIT_PRIVATE  128
%define FUTEX_WAKE_PRIVATE  129

section .bss
    mutex_var:  resd 1      ; 0 = unlocked, 1 = locked
    counter:    resd 1      ; shared counter

section .data
    lock_msg    db "Lock acquired", 10
    lock_ml     equ $ - lock_msg
    unlock_msg  db "Lock released", 10
    unlock_ml   equ $ - unlock_msg

section .text

; mutex_lock: lock futex mutex
; input: RDI = pointer ไปยัง mutex variable
mutex_lock:
.try_lock:
    xor eax, eax            ; expected = 0 (unlocked)
    mov ecx, 1              ; new value = 1 (locked)
    lock cmpxchg [rdi], ecx ; atomic compare-and-swap
    jz .locked              ; ถ้า ZF set = swap สำเร็จ (เราได้ lock)

    ; ไม่ได้ lock รอด้วย futex
    push rdi                ; save mutex pointer
    push rcx
    push r11

    ; futex(mutex, FUTEX_WAIT_PRIVATE, 1, NULL)
    ; รอจนกว่า *mutex != 1
    mov rax, 202            ; SYS_futex
    ; rdi ยังชี้ไปยัง mutex
    mov rsi, FUTEX_WAIT_PRIVATE
    mov rdx, 1              ; expected value
    xor r10, r10            ; timeout = NULL
    syscall

    pop r11
    pop rcx
    pop rdi

    jmp .try_lock           ; ลอง lock อีกครั้ง

.locked:
    ret

; mutex_unlock: unlock futex mutex
; input: RDI = pointer ไปยัง mutex variable
mutex_unlock:
    mov dword [rdi], 0      ; set mutex = 0

    ; ปลุก thread ที่รออยู่
    mov rax, 202            ; SYS_futex
    mov rsi, FUTEX_WAKE_PRIVATE
    mov rdx, 1              ; wake 1 thread
    syscall
    ret

_start:
    mov dword [mutex_var], 0    ; initialize mutex

    ; ล็อค
    lea rdi, [mutex_var]
    call mutex_lock

    ; แสดง lock acquired
    mov rax, 1
    mov rdi, 1
    lea rsi, [lock_msg]
    mov rdx, lock_ml
    syscall

    ; Critical section: เพิ่ม counter
    inc dword [counter]

    ; ปลด lock
    lea rdi, [mutex_var]
    call mutex_unlock

    ; แสดง unlock
    mov rax, 1
    mov rdi, 1
    lea rsi, [unlock_msg]
    mov rdx, unlock_ml
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 18. การตรวจสอบ Syscall Return Value แบบ EINTR

```nasm
; eintr_handler.asm - จัดการกับ EINTR (interrupted system call)

; EINTR = -4 (error number 4)
; เกิดขึ้นเมื่อ syscall ถูก interrupt โดย signal handler

section .bss
    buf: resb 1024

section .text

; safe_read: read ที่ handle EINTR อัตโนมัติ
; input: RDI=fd, RSI=buf, RDX=count
; output: RAX=bytes_read หรือ error
safe_read:
.retry:
    push rdi
    push rsi
    push rdx
    mov rax, 0          ; SYS_read
    syscall
    pop rdx
    pop rsi
    pop rdi

    ; ตรวจสอบ EINTR (-4)
    cmp rax, -4         ; -EINTR
    je .retry           ; ถ้า interrupted ลองใหม่

    ret

; safe_write: write ที่ handle EINTR และ partial writes
; input: RDI=fd, RSI=buf, RDX=count
; output: RAX=0 (success) หรือ error
safe_write:
    push r12
    push r13
    push r14
    mov r12, rdi        ; save fd
    mov r13, rsi        ; save buf
    mov r14, rdx        ; save total count

.write_loop:
    test r14, r14
    jz .done

    mov rax, 1          ; SYS_write
    mov rdi, r12
    mov rsi, r13
    mov rdx, r14
    syscall

    ; ตรวจสอบ EINTR
    cmp rax, -4
    je .write_loop

    ; ตรวจสอบ error อื่น
    test rax, rax
    js .error_ret

    ; ปรับ buffer pointer และ count
    add r13, rax
    sub r14, rax
    jmp .write_loop

.done:
    xor rax, rax
    pop r14
    pop r13
    pop r12
    ret

.error_ret:
    pop r14
    pop r13
    pop r12
    ret  ; RAX มี error code อยู่แล้ว
```

---

## 19. การ Build และ Test

```bash
# Script สำหรับ build ทุกโปรแกรม
#!/bin/bash

# echo
nasm -f elf64 echo.asm -o echo.o && ld echo.o -o echo_asm
./echo_asm Hello World Assembly

# cat
nasm -f elf64 cat.asm -o cat.o && ld cat.o -o cat_asm
echo "Test content" > /tmp/test.txt
./cat_asm /tmp/test.txt

# wc
nasm -f elf64 wc.asm -o wc.o && ld wc.o -o wc_asm
./wc_asm /tmp/test.txt

# cp
nasm -f elf64 cp.asm -o cp.o && ld cp.o -o cp_asm
./cp_asm /tmp/test.txt /tmp/test_copy.txt
cat /tmp/test_copy.txt

# http server (ทดสอบใน background)
nasm -f elf64 http_server.asm -o http_server.o && ld http_server.o -o http_server
./http_server &
sleep 1
curl -v http://localhost:8080
kill %1

echo "Tests complete!"
```

---

## 20. ตัวอย่างการใช้งาน strace

```bash
# ดู syscalls ของ ls
strace ls /tmp

# ดูเฉพาะ file-related syscalls
strace -e trace=file ls /tmp

# นับ syscalls
strace -c ls /tmp

# ดู syscalls ของ process ที่กำลังทำงาน (root required)
strace -p <PID>

# ตัวอย่าง output:
# execve("/bin/ls", ["ls", "/tmp"], 0x7fff... /* 17 vars */) = 0
# brk(NULL)                               = 0x55a...
# openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
# fstat(3, {st_mode=S_IFREG|0644, st_size=...}) = 0
# mmap(NULL, ..., PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f...
# write(1, "file1.txt  file2.txt\n", 21)   = 21
# exit_group(0)                           = ?
```

---

## บทสรุป

ในบทนี้เราได้เรียนรู้:

1. **syscall instruction** และความแตกต่างจาก int 0x80
2. **Register convention**: RAX (number), RDI/RSI/RDX/R10/R8/R9 (args), RAX (return)
3. **RCX และ R11** ถูกทำลายโดย syscall instruction
4. **Syscall numbers** จาก /usr/include/asm/unistd_64.h
5. **syscalls ที่สำคัญ**: read, write, open, close, stat, fstat, poll, lseek, mmap, mprotect, munmap, brk, rt_sigaction, rt_sigprocmask, ioctl, pread64, pwrite64, readv, writev, pipe, dup, dup2, nanosleep, getpid, socket, connect, accept, sendto, recvfrom, bind, listen, clone, fork, vfork, execve, exit, wait4, kill, futex, epoll_create1, epoll_ctl, epoll_wait
6. **โปรแกรมตัวอย่าง**: echo, cat, wc, cp, HTTP server

การเข้าใจ system calls เป็นพื้นฐานสำคัญของการเขียน assembly ระดับ OS และการพัฒนา low-level systems programming
