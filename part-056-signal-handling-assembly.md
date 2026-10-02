# Part 056: Signal Handling ใน Assembly

## บทนำ (Introduction)

Signal เป็นกลไกการสื่อสารแบบ asynchronous ระหว่าง processes ใน Unix/Linux
เมื่อ signal ถูกส่งไปยัง process, kernel จะขัดจังหวะการทำงานปกติ
และเรียก signal handler ที่กำหนดไว้ การเขียน signal handling ใน Assembly
ต้องเข้าใจโครงสร้างข้อมูลระดับ kernel และ calling conventions อย่างลึกซึ้ง

---

## 1. Signal Concepts — แนวคิดพื้นฐาน

### 1.1 Standard Signals ที่สำคัญ

```
Signal Name   Number   Default Action    ความหมาย
-----------   ------   --------------    ---------
SIGHUP           1     Terminate         Terminal hangup
SIGINT           2     Terminate         Interrupt from keyboard (Ctrl+C)
SIGQUIT          3     Core dump         Quit from keyboard (Ctrl+\)
SIGILL           4     Core dump         Illegal instruction
SIGTRAP          5     Core dump         Trace/breakpoint trap
SIGABRT          6     Core dump         Abort signal (from abort())
SIGBUS           7     Core dump         Bus error (alignment error)
SIGFPE           8     Core dump         Floating-point exception
SIGKILL          9     Terminate         Kill signal (cannot be caught!)
SIGUSR1         10     Terminate         User-defined signal 1
SIGSEGV         11     Core dump         Segmentation violation
SIGUSR2         12     Terminate         User-defined signal 2
SIGPIPE         13     Terminate         Broken pipe
SIGALRM         14     Terminate         Timer signal from alarm()
SIGTERM         15     Terminate         Termination signal
SIGCHLD         17     Ignore            Child stopped or terminated
SIGCONT         18     Continue          Continue if stopped
SIGSTOP         19     Stop              Stop process (cannot be caught!)
SIGTSTP         20     Stop              Stop typed at terminal (Ctrl+Z)
SIGTTIN         21     Stop              Terminal input for bg process
SIGTTOU         22     Stop              Terminal output for bg process
SIGURG          23     Ignore            Urgent data on socket
SIGXCPU         24     Core dump         CPU time limit exceeded
SIGXFSZ         25     Core dump         File size limit exceeded
SIGVTALRM       26     Terminate         Virtual alarm clock
SIGPROF         27     Terminate         Profiling timer expired
SIGWINCH        28     Ignore            Window resize signal
SIGIO           29     Terminate         I/O now possible
SIGPWR          30     Terminate         Power failure
SIGSYS          31     Core dump         Bad system call
```

### 1.2 Signal ที่ไม่สามารถ Block หรือ Catch ได้

```
SIGKILL (9)  — ฆ่า process ทันที ไม่มีทางหลบได้
SIGSTOP (19) — หยุด process ชั่วคราว ไม่มีทางหลบได้
```

---

## 2. Signal Delivery Mechanism — กลไกการส่ง Signal

### 2.1 วงจรชีวิตของ Signal

```
ขั้นตอน 1: Signal Generation (การสร้าง signal)
  - keyboard (Ctrl+C → SIGINT)
  - kill() syscall
  - hardware exception (page fault → SIGSEGV)
  - alarm() / timer

ขั้นตอน 2: Signal Pending (signal รอดำเนินการ)
  - kernel บันทึก signal ใน pending set ของ process
  - ถ้า signal ถูก block จะรออยู่ใน pending set

ขั้นตอน 3: Signal Delivery (การส่ง signal)
  - เกิดขึ้นเมื่อ process กลับจาก kernel mode → user mode
  - kernel ตรวจสอบ pending signals ที่ไม่ถูก block
  - เรียก signal handler หรือทำ default action
```

### 2.2 Process Signal State

```
แต่ละ process มี:
  - pending set    : signals ที่รอ delivery
  - blocked set    : signals ที่ถูก block (signal mask)
  - disposition    : การจัดการสำหรับแต่ละ signal
                     (SIG_DFL, SIG_IGN, หรือ handler function)
```

---

## 3. sigaction Structure ใน Assembly

### 3.1 โครงสร้าง sigaction บน Linux x86-64

```nasm
; struct sigaction {
;     void (*sa_handler)(int);       ; offset 0  — handler หรือ SIG_DFL/SIG_IGN
;     unsigned long sa_flags;        ; offset 8  — flags
;     void (*sa_restorer)(void);     ; offset 16 — restorer (deprecated แต่ยังต้องใช้)
;     sigset_t sa_mask;              ; offset 24 — signals to block during handler
; };
; รวม sizeof(sigaction) = 152 bytes บน x86-64
; sizeof(sigset_t) = 128 bytes (1024 bits สำหรับ real-time signals)

struc sigaction_t
    .sa_handler    resq 1      ; 8 bytes
    .sa_flags      resq 1      ; 8 bytes
    .sa_restorer   resq 1      ; 8 bytes
    .sa_mask       resb 128    ; 128 bytes = sigset_t
endstruc
; SIGACTION_SIZE = 8+8+8+128 = 152
```

### 3.2 sa_flags ที่สำคัญ

```nasm
SA_NOCLDSTOP  equ 0x00000001  ; ไม่รับ SIGCHLD เมื่อ child หยุด
SA_NOCLDWAIT  equ 0x00000002  ; ไม่สร้าง zombie เมื่อ child ตาย
SA_SIGINFO    equ 0x00000004  ; handler รับ siginfo_t และ ucontext_t
SA_RESTORER   equ 0x04000000  ; ใช้ sa_restorer (ต้องกำหนดเมื่อใช้)
SA_ONSTACK    equ 0x08000000  ; ใช้ alternate signal stack
SA_RESTART    equ 0x10000000  ; restart syscalls อัตโนมัติ
SA_NODEFER    equ 0x40000000  ; ไม่ block signal เดิมขณะรัน handler
SA_RESETHAND  equ 0x80000000  ; reset handler เป็น SIG_DFL หลัง delivery
```

### 3.3 sigset_t Operations

```nasm
; sigset_t คือ bitmask ขนาด 128 bytes (1024 bits)
; signal N อยู่ใน bit (N-1) ของ sigset_t
; เช่น SIGINT(2) อยู่ใน bit 1 ของ word แรก

; sigemptyset — ล้าง sigset ทั้งหมด
; ทำโดย memset(sigset, 0, 128)

; sigfillset — ตั้งทุก bit
; ทำโดย memset(sigset, 0xFF, 128)

; sigaddset(sigset_t *set, int signum)
; word = (signum - 1) / 64   → index ใน array of uint64_t
; bit  = (signum - 1) % 64   → bit position ใน word
; set->words[word] |= (1 << bit)

; sigdelset(sigset_t *set, int signum)
; set->words[word] &= ~(1 << bit)

; sigismember(sigset_t *set, int signum)
; return (set->words[word] >> bit) & 1
```

---

## 4. rt_sigaction Syscall

### 4.1 Syscall Interface

```nasm
; rt_sigaction syscall
; syscall number: 13 (x86-64)
; rdi = signum
; rsi = pointer to new sigaction (หรือ NULL)
; rdx = pointer to old sigaction (หรือ NULL)
; r10 = sizeof(sigset_t) = 8 (standard) หรือ 128 (rt)
; return: 0 on success, -errno on error

%define SYS_rt_sigaction  13
%define SIGSETSIZE        8      ; kernel ใช้ 8 bytes สำหรับ basic sigset
```

### 4.2 ตัวอย่าง: ติดตั้ง Signal Handler

```nasm
; install_handler(signum, handler_func)
; rdi = signal number
; rsi = handler function pointer
install_handler:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 160            ; พื้นที่สำหรับ sigaction struct (152 bytes + padding)
    push    rdi                 ; เก็บ signum
    push    rsi                 ; เก็บ handler

    ; ล้าง sigaction struct ทั้งหมด
    lea     rdi, [rbp - 160]
    xor     esi, esi
    mov     edx, 152
    call    memset_simple

    pop     rsi                 ; คืน handler
    pop     rdi                 ; คืน signum

    ; ตั้งค่า sa_handler
    lea     rax, [rbp - 160]
    mov     [rax + 0], rsi      ; sa_handler = handler

    ; ตั้งค่า sa_flags = SA_RESTORER
    mov     qword [rax + 8], SA_RESTORER
    
    ; ตั้งค่า sa_restorer = __restore_rt
    lea     rcx, [rel __restore_rt]
    mov     [rax + 16], rcx     ; sa_restorer

    ; เรียก rt_sigaction syscall
    ; rdi ยังเป็น signum
    mov     rsi, rax            ; new sigaction
    xor     edx, edx            ; old sigaction = NULL
    mov     r10, 8              ; sizeof(sigset_t) สำหรับ kernel
    mov     eax, SYS_rt_sigaction
    syscall

    leave
    ret

; __restore_rt — trampoline สำหรับ return จาก signal handler
__restore_rt:
    mov     eax, SYS_rt_sigreturn   ; 15
    syscall
```

---

## 5. SA_SIGINFO — siginfo_t และ ucontext_t

### 5.1 โครงสร้าง siginfo_t

```nasm
; struct siginfo_t (simplified layout บน x86-64)
; typedef struct {
;     int     si_signo;      ; offset 0  — signal number
;     int     si_errno;      ; offset 4  — errno value
;     int     si_code;       ; offset 8  — signal code
;     union {                ; offset 16 — signal-specific data
;         ...
;         struct { pid_t pid; uid_t uid; } _kill;
;         struct { int timerid; int overrun; } _timer;
;         struct { void *addr; } _sigfault;
;     };
; } siginfo_t;

; si_code values ที่สำคัญ:
; SI_USER    0  — sent by kill(), pthread_kill()
; SI_KERNEL  128 — sent by kernel
; SI_QUEUE   -1 — sent by sigqueue()
; SEGV_MAPERR 1 — address not mapped (SIGSEGV)
; SEGV_ACCERR 2 — invalid permissions (SIGSEGV)
; FPE_INTDIV  1 — integer divide by zero (SIGFPE)
; BUS_ADRALN  1 — invalid address alignment (SIGBUS)

SIGINFO_SIGNO   equ 0
SIGINFO_ERRNO   equ 4
SIGINFO_CODE    equ 8
SIGINFO_PID     equ 16          ; สำหรับ kill signal
SIGINFO_UID     equ 20
SIGINFO_ADDR    equ 16          ; สำหรับ fault signal (SIGSEGV, SIGBUS)
```

### 5.2 โครงสร้าง ucontext_t (simplified)

```nasm
; struct ucontext_t บน x86-64
; typedef struct ucontext {
;     unsigned long uc_flags;
;     struct ucontext *uc_link;
;     stack_t uc_stack;
;     mcontext_t uc_mcontext;    ; ← register state ตอน signal เกิด
;     sigset_t uc_sigmask;
; } ucontext_t;

; mcontext_t offsets (gregs array):
; REG_R8   = 0   (index 0)
; REG_R9   = 1
; REG_R10  = 2
; REG_R11  = 3
; REG_R12  = 4
; REG_R13  = 5
; REG_R14  = 6
; REG_R15  = 7
; REG_RDI  = 8
; REG_RSI  = 9
; REG_RBP  = 10
; REG_RBX  = 11
; REG_RDX  = 12
; REG_RAX  = 13
; REG_RCX  = 14
; REG_RSP  = 15
; REG_RIP  = 16   ← instruction pointer ที่ทำให้เกิด fault
; REG_EFL  = 17   ← RFLAGS
; REG_CSGSFS = 18
; REG_ERR  = 19
; REG_TRAPNO = 20
; REG_OLDMASK = 21
; REG_CR2  = 22   ← fault address สำหรับ SIGSEGV

; ucontext_t offsets:
UC_FLAGS        equ 0
UC_LINK         equ 8
UC_STACK_SP     equ 16
UC_STACK_FLAGS  equ 24
UC_STACK_SIZE   equ 32
UC_MCONTEXT     equ 40          ; mcontext เริ่มที่ offset 40
MCONTEXT_GREGS  equ UC_MCONTEXT ; gregs array อยู่ต้น mcontext
GREG_SIZE       equ 8           ; แต่ละ register = 8 bytes
REG_RIP_IDX     equ 16
REG_RSP_IDX     equ 15
REG_RIP_OFFSET  equ (UC_MCONTEXT + REG_RIP_IDX * GREG_SIZE)  ; = 40 + 128 = 168
```

### 5.3 SA_SIGINFO Handler Signature

```nasm
; void handler(int signo, siginfo_t *info, void *ucontext)
; rdi = signal number
; rsi = pointer to siginfo_t
; rdx = pointer to ucontext_t
```

---

## 6. sigprocmask — การจัดการ Signal Mask

### 6.1 Syscall Interface

```nasm
; rt_sigprocmask syscall
; syscall number: 14 (x86-64)
; rdi = how (SIG_BLOCK=0, SIG_UNBLOCK=1, SIG_SETMASK=2)
; rsi = pointer to new set (หรือ NULL)
; rdx = pointer to old set (หรือ NULL)
; r10 = sizeof(sigset_t) = 8
; return: 0 on success

%define SYS_rt_sigprocmask  14
%define SIG_BLOCK           0
%define SIG_UNBLOCK         1
%define SIG_SETMASK         2
```

### 6.2 ตัวอย่าง Block/Unblock Signals

```nasm
; block_signal(signum) — block signal หนึ่งตัว
; rdi = signal number
block_signal:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16             ; พื้นที่สำหรับ sigset_t (8 bytes เพียงพอ)

    ; สร้าง sigset ที่มี signum
    push    rdi
    lea     rdi, [rbp - 16]
    xor     esi, esi
    mov     edx, 8
    call    memset_simple       ; sigemptyset
    pop     rdi

    ; sigaddset: word = (signum-1)/64, bit = (signum-1)%64
    dec     rdi
    mov     ecx, edi
    shr     ecx, 6              ; word index
    and     edi, 63             ; bit index
    lea     rax, [rbp - 16]
    bts     qword [rax + rcx*8], rdi   ; set bit

    ; rt_sigprocmask(SIG_BLOCK, &newset, NULL, 8)
    mov     rdi, SIG_BLOCK
    lea     rsi, [rbp - 16]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigprocmask
    syscall

    leave
    ret

; unblock_all_signals() — ปลด block ทุก signal
unblock_all_signals:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16

    ; sigfillset
    lea     rdi, [rbp - 16]
    mov     esi, 0xFF
    mov     edx, 8
    call    memset_simple

    ; rt_sigprocmask(SIG_UNBLOCK, &fullset, NULL, 8)
    mov     rdi, SIG_UNBLOCK
    lea     rsi, [rbp - 16]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigprocmask
    syscall

    leave
    ret
```

---

## 7. sigpending และ sigsuspend

### 7.1 sigpending — ตรวจ signals ที่รอ

```nasm
; rt_sigpending syscall
; syscall number: 127 (x86-64)
; rdi = pointer to sigset_t (output)
; rsi = sizeof(sigset_t) = 8
; return: 0 on success

%define SYS_rt_sigpending  127

; check_pending_signals(sigset_t *pending_set)
; rdi = output buffer
check_pending_signals:
    push    rbp
    mov     rbp, rsp
    ; rdi ยังคงเป็น pointer to sigset_t
    mov     rsi, 8
    mov     eax, SYS_rt_sigpending
    syscall
    leave
    ret
```

### 7.2 sigsuspend — รอ signal โดย suspend process

```nasm
; rt_sigsuspend syscall
; syscall number: 130 (x86-64)
; rdi = pointer to sigset_t (mask ระหว่าง suspend)
; rsi = sizeof(sigset_t) = 8
; return: -EINTR เสมอ (เมื่อ signal handler ทำงานแล้ว)

%define SYS_rt_sigsuspend  130

; wait_for_signal() — บล็อกทุก signal แล้วรอ SIGUSR1
; ใช้สำหรับ synchronization
wait_for_signal:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16

    ; สร้าง mask ที่บล็อกทุกอย่างยกเว้น SIGUSR1
    ; sigfillset
    lea     rdi, [rbp - 16]
    mov     esi, 0xFF
    mov     edx, 8
    call    memset_simple

    ; sigdelset SIGUSR1 (10)
    lea     rax, [rbp - 16]
    btr     qword [rax], 9      ; bit 9 = signal 10 (SIGUSR1)

    ; sigsuspend
    lea     rdi, [rbp - 16]
    mov     rsi, 8
    mov     eax, SYS_rt_sigsuspend
    syscall
    ; จะ return -EINTR หลัง SIGUSR1 handler ทำงาน

    leave
    ret
```

---

## 8. kill Syscall — ส่ง Signal

### 8.1 kill, tkill, tgkill

```nasm
; kill syscall
; syscall number: 62 (x86-64)
; rdi = pid (>0: process, 0: process group, -1: all, <-1: abs(pid) group)
; rsi = signal number
; return: 0 on success, -errno on error

%define SYS_kill    62
%define SYS_tkill   200         ; ส่ง signal ไปยัง thread เฉพาะ
%define SYS_tgkill  234         ; thread group kill (ปลอดภัยกว่า tkill)

; send_signal(pid, signum)
send_signal:
    ; rdi = pid, rsi = signum
    mov     eax, SYS_kill
    syscall
    ret

; self_signal(signum)  — ส่ง signal ให้ตัวเอง
self_signal:
    mov     rsi, rdi            ; signum
    mov     edi, 0              ; pid=0 = ส่งให้ทั้ง process group
    mov     eax, SYS_kill
    syscall
    ret

; getpid แล้วส่ง signal ให้ตัวเอง
raise_signal:
    push    rbx
    mov     rbx, rdi            ; เก็บ signum

    ; getpid
    mov     eax, 39             ; SYS_getpid
    syscall
    mov     edi, eax            ; pid

    mov     rsi, rbx            ; signum
    mov     eax, SYS_kill
    syscall

    pop     rbx
    ret
```

---

## 9. alarm และ setitimer

### 9.1 alarm syscall

```nasm
; alarm syscall
; syscall number: 37 (x86-64)
; rdi = seconds (0 = cancel alarm)
; return: seconds remaining from previous alarm (หรือ 0)

%define SYS_alarm  37

; set_alarm(seconds)
set_alarm:
    ; rdi = seconds
    mov     eax, SYS_alarm
    syscall
    ret
```

### 9.2 setitimer — Timer ที่ละเอียดกว่า

```nasm
; struct itimerval {
;     struct timeval it_interval;  ; offset 0  — interval สำหรับ periodic timer
;     struct timeval it_value;     ; offset 16 — เวลาที่เหลือจนถึง next expiration
; };
; struct timeval {
;     long tv_sec;   ; วินาที
;     long tv_usec;  ; microseconds
; };

; setitimer syscall
; syscall number: 38 (x86-64)
; rdi = which (ITIMER_REAL=0, ITIMER_VIRTUAL=1, ITIMER_PROF=2)
; rsi = pointer to new itimerval
; rdx = pointer to old itimerval (หรือ NULL)
; return: 0 on success

%define SYS_setitimer   38
%define ITIMER_REAL     0       ; นับเวลาจริง → SIGALRM
%define ITIMER_VIRTUAL  1       ; นับเวลา user-mode → SIGVTALRM
%define ITIMER_PROF     2       ; นับ user+kernel time → SIGPROF

; itimerval field offsets
ITIMER_INTERVAL_SEC   equ 0
ITIMER_INTERVAL_USEC  equ 8
ITIMER_VALUE_SEC      equ 16
ITIMER_VALUE_USEC     equ 24
ITIMERVAL_SIZE        equ 32
```

---

## 10. Async-Signal-Safe Functions

### 10.1 ฟังก์ชันที่ปลอดภัยใน Signal Handler

Signal handler ทำงานแบบ asynchronous จึงต้องใช้เฉพาะฟังก์ชันที่ async-signal-safe:

```
Async-signal-safe (ใช้ได้ใน handler):
  _exit, abort, accept, access, alarm, bind, cfgetispeed,
  cfgetospeed, cfsetispeed, cfsetospeed, chdir, chmod,
  chown, clock_gettime, close, connect, creat, dup, dup2,
  execl, execle, execv, execve, _exit, faccessat, fchdir,
  fchmod, fchmodat, fchown, fchownat, fcntl, fdatasync,
  fork, fstat, fstatat, fsync, ftruncate, futimens,
  getegid, geteuid, getgid, getgroups, getpeername,
  getpgrp, getpid, getppid, getsockname, getsockopt,
  getuid, kill, link, linkat, listen, lseek, lstat,
  mkdir, mkdirat, mkfifo, mkfifoat, mknod, mknodat,
  open, openat, pause, pipe, poll, posix_trace_event,
  pselect, pthread_kill, pthread_self, pthread_sigmask,
  raise, read, readlink, readlinkat, recv, recvfrom,
  recvmsg, rename, renameat, rmdir, select, sem_post,
  send, sendmsg, sendto, setgid, setpgid, setsid,
  setsockopt, setuid, shutdown, sigaction, sigaddset,
  sigdelset, sigemptyset, sigfillset, sigismember,
  signal, sigpause, sigpending, sigprocmask, sigqueue,
  sigset, sigsuspend, sleep, sockatmark, socket,
  socketpair, stat, symlink, symlinkat, tcdrain,
  tcflow, tcflush, tcgetattr, tcgetpgrp, tcsendbreak,
  tcsetattr, tcsetpgrp, time, timer_getoverrun,
  timer_gettime, timer_settime, times, umask, uname,
  unlink, unlinkat, utime, utimensat, utimes, wait,
  waitpid, write

ไม่ปลอดภัย (ห้ามใช้ใน handler!):
  malloc, free, printf, fprintf, sprintf, fopen,
  fclose, fread, fwrite, exit (ใช้ _exit แทน),
  setjmp, longjmp, pthread functions ส่วนใหญ่,
  ฟังก์ชันที่ใช้ global state ใดๆ
```

### 10.2 การเขียน Output ใน Signal Handler อย่างปลอดภัย

```nasm
; ใน signal handler ต้องใช้ write() syscall โดยตรง
; ไม่ใช้ printf หรือ puts

safe_write_msg:
    ; rdi = fd, rsi = buf, rdx = len
    mov     eax, 1              ; SYS_write
    syscall
    ret

; ตัวอย่างการแปลง int เป็น string ใน signal handler
; (ไม่ใช้ sprintf เพราะไม่ปลอดภัย)
int_to_str:
    ; rdi = number, rsi = buffer (min 20 bytes), rdx = pointer to length
    push    rbx
    push    r12
    push    r13

    mov     rbx, rdi            ; number
    mov     r12, rsi            ; buffer
    mov     r13, rdx            ; length pointer

    ; เขียนจากท้ายไปหน้า
    lea     rdi, [r12 + 19]
    mov     byte [rdi], 0       ; null terminator
    mov     ecx, 0              ; digit count

    test    rbx, rbx
    jnz     .convert_loop
    ; กรณี number = 0
    dec     rdi
    mov     byte [rdi], '0'
    inc     ecx
    jmp     .done

.convert_loop:
    test    rbx, rbx
    jz      .done
    xor     edx, edx
    mov     rax, rbx
    mov     r8d, 10
    div     r8                  ; rax = quotient, rdx = remainder
    mov     rbx, rax
    add     dl, '0'
    dec     rdi
    mov     [rdi], dl
    inc     ecx
    jmp     .convert_loop

.done:
    mov     [r13], ecx          ; เก็บความยาว
    mov     rax, rdi            ; return pointer to start

    pop     r13
    pop     r12
    pop     rbx
    ret
```

---

## 11. โปรแกรมสมบูรณ์: SIGINT Handler (Clean Shutdown)

```nasm
; =============================================================
; ไฟล์: sigint_handler.asm
; คำอธิบาย: จัดการ SIGINT เพื่อ cleanup ก่อน shutdown
; สร้าง: nasm -f elf64 sigint_handler.asm -o sigint_handler.o
;         ld sigint_handler.o -o sigint_handler
; ทดสอบ: ./sigint_handler แล้วกด Ctrl+C
; =============================================================

section .data
    ; ข้อความต่างๆ
    msg_start       db  "โปรแกรมเริ่มทำงาน... กด Ctrl+C เพื่อหยุด", 10
    msg_start_len   equ $ - msg_start
    msg_loop        db  "กำลังทำงาน (loop iteration: ", 0
    msg_loop_end    db  ")", 10, 0
    msg_cleanup     db  10, "กำลัง cleanup ก่อน shutdown...", 10
    msg_cleanup_len equ $ - msg_cleanup
    msg_done        db  "Cleanup เสร็จสิ้น ออกจากโปรแกรม", 10
    msg_done_len    equ $ - msg_done
    msg_received    db  "ได้รับ SIGINT!", 10
    msg_received_len equ $ - msg_received

section .bss
    ; ตัวแปร global สำหรับ signal handler
    g_running       resb 1      ; flag: 1=กำลังทำงาน, 0=หยุด
    num_buf         resb 20     ; buffer สำหรับแปลงตัวเลข
    sa_new          resb 152    ; sigaction struct ใหม่
    sa_old          resb 152    ; sigaction struct เดิม

section .text
global _start

; ค่าคงที่ syscall
%define SYS_read            0
%define SYS_write           1
%define SYS_nanosleep       35
%define SYS_alarm           37
%define SYS_exit            60
%define SYS_rt_sigaction    13
%define SYS_rt_sigreturn    15

; ค่าคงที่ signal
%define SIGINT              2
%define SA_RESTORER         0x04000000

; โครงสร้าง sigaction offsets
%define SA_HANDLER_OFF      0
%define SA_FLAGS_OFF        8
%define SA_RESTORER_OFF     16
%define SA_MASK_OFF         24

; timespec สำหรับ nanosleep
section .data
    sleep_ts:
        .tv_sec     dq  0           ; 0 วินาที
        .tv_nsec    dq  100000000   ; 100 ms = 100,000,000 ns

section .text

; =============================================================
; __restore_rt — trampoline สำหรับ return จาก signal handler
; kernel ต้องการฟังก์ชันนี้เมื่อใช้ SA_RESTORER
; =============================================================
__restore_rt:
    mov     eax, SYS_rt_sigreturn
    syscall

; =============================================================
; sigint_handler — จัดการ SIGINT
; rdi = signal number (ถ้าใช้ simple handler)
; =============================================================
sigint_handler:
    ; บันทึกว่าได้รับ signal
    ; (ใช้เฉพาะ async-signal-safe operations)
    mov     byte [rel g_running], 0     ; ตั้ง flag หยุดทำงาน

    ; แสดงข้อความ (ใช้ write syscall โดยตรง)
    mov     eax, SYS_write
    mov     edi, 1                      ; stdout
    lea     rsi, [rel msg_received]
    mov     edx, msg_received_len
    syscall

    ret

; =============================================================
; install_sigint_handler — ติดตั้ง handler สำหรับ SIGINT
; =============================================================
install_sigint_handler:
    push    rbp
    mov     rbp, rsp

    ; ล้าง sa_new struct
    lea     rdi, [rel sa_new]
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    ; sa_handler = sigint_handler
    lea     rax, [rel sigint_handler]
    mov     [rel sa_new + SA_HANDLER_OFF], rax

    ; sa_flags = SA_RESTORER
    mov     qword [rel sa_new + SA_FLAGS_OFF], SA_RESTORER

    ; sa_restorer = __restore_rt
    lea     rax, [rel __restore_rt]
    mov     [rel sa_new + SA_RESTORER_OFF], rax

    ; rt_sigaction(SIGINT, &sa_new, &sa_old, 8)
    mov     edi, SIGINT
    lea     rsi, [rel sa_new]
    lea     rdx, [rel sa_old]
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    leave
    ret

; =============================================================
; cleanup — ทำ cleanup ก่อน exit
; =============================================================
cleanup:
    push    rbp
    mov     rbp, rsp

    ; แสดงข้อความ cleanup
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_cleanup]
    mov     edx, msg_cleanup_len
    syscall

    ; จำลองการทำ cleanup (sleep 200ms)
    lea     rdi, [rel sleep_ts]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall
    mov     eax, SYS_nanosleep
    syscall

    ; แสดงข้อความ done
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_done]
    mov     edx, msg_done_len
    syscall

    leave
    ret

; =============================================================
; _start — entry point หลัก
; =============================================================
_start:
    ; ติดตั้ง signal handler
    call    install_sigint_handler

    ; ตั้ง flag เริ่มทำงาน
    mov     byte [rel g_running], 1

    ; แสดงข้อความเริ่มต้น
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_start]
    mov     edx, msg_start_len
    syscall

    ; loop หลัก
    xor     ebx, ebx                ; iteration counter
.main_loop:
    ; ตรวจสอบ flag
    movzx   eax, byte [rel g_running]
    test    eax, eax
    jz      .loop_done

    inc     ebx

    ; sleep 500ms
    lea     rdi, [rel sleep_ts]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall

    ; วนซ้ำ
    jmp     .main_loop

.loop_done:
    ; ทำ cleanup
    call    cleanup

    ; exit(0)
    xor     edi, edi
    mov     eax, SYS_exit
    syscall
```

---

## 12. โปรแกรมสมบูรณ์: SIGALRM Timeout

```nasm
; =============================================================
; ไฟล์: sigalrm_timeout.asm
; คำอธิบาย: ใช้ SIGALRM สำหรับ timeout operation
; สร้าง: nasm -f elf64 sigalrm_timeout.asm -o sigalrm_timeout.o
;         ld sigalrm_timeout.o -o sigalrm_timeout
; =============================================================

section .data
    msg_wait    db  "รอ input จาก stdin (timeout 5 วินาที)...", 10
    msg_wait_l  equ $ - msg_wait
    msg_timeout db  10, "TIMEOUT! ไม่มี input ใน 5 วินาที", 10
    msg_timeout_l equ $ - msg_timeout
    msg_got_in  db  "ได้รับ input: ", 0
    msg_newline db  10, 0
    msg_ok      db  "เสร็จสิ้น (ก่อน timeout)", 10
    msg_ok_l    equ $ - msg_ok

section .bss
    g_timed_out resb 1          ; 1 = timeout เกิดขึ้น
    input_buf   resb 256
    sa_sigalrm  resb 152

section .text
global _start

%define SYS_read            0
%define SYS_write           1
%define SYS_exit            60
%define SYS_alarm           37
%define SYS_rt_sigaction    13
%define SYS_rt_sigreturn    15
%define SIGALRM             14
%define SA_RESTORER         0x04000000

__restore_rt_alrm:
    mov     eax, SYS_rt_sigreturn
    syscall

; =============================================================
; sigalrm_handler — จัดการ SIGALRM
; =============================================================
sigalrm_handler:
    mov     byte [rel g_timed_out], 1   ; บันทึกว่า timeout

    ; แสดงข้อความ timeout
    mov     eax, SYS_write
    mov     edi, 2                      ; stderr
    lea     rsi, [rel msg_timeout]
    mov     edx, msg_timeout_l
    syscall
    ret

; =============================================================
; install_sigalrm — ติดตั้ง SIGALRM handler
; =============================================================
install_sigalrm:
    push    rbp
    mov     rbp, rsp

    ; ล้าง struct
    lea     rdi, [rel sa_sigalrm]
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    lea     rax, [rel sigalrm_handler]
    mov     [rel sa_sigalrm + 0], rax   ; sa_handler
    mov     qword [rel sa_sigalrm + 8], SA_RESTORER
    lea     rax, [rel __restore_rt_alrm]
    mov     [rel sa_sigalrm + 16], rax  ; sa_restorer

    mov     edi, SIGALRM
    lea     rsi, [rel sa_sigalrm]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    leave
    ret

; =============================================================
; read_with_timeout — อ่าน stdin พร้อม timeout
; rdi = buffer, rsi = size, rdx = timeout_seconds
; return: bytes read หรือ -1 ถ้า timeout
; =============================================================
read_with_timeout:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12
    push    r13

    mov     rbx, rdi            ; buffer
    mov     r12, rsi            ; size
    mov     r13, rdx            ; timeout

    ; ติดตั้ง SIGALRM handler
    call    install_sigalrm

    ; ล้าง flag
    mov     byte [rel g_timed_out], 0

    ; ตั้ง alarm
    mov     rdi, r13
    mov     eax, SYS_alarm
    syscall

    ; อ่าน stdin (จะถูก interrupt โดย SIGALRM ถ้า timeout)
    mov     rdi, 0              ; stdin
    mov     rsi, rbx            ; buffer
    mov     rdx, r12            ; size
    mov     eax, SYS_read
    syscall
    push    rax                 ; เก็บ return value

    ; ยกเลิก alarm
    xor     edi, edi
    mov     eax, SYS_alarm
    syscall

    pop     rax                 ; คืน return value

    ; ตรวจสอบว่า timeout หรือเปล่า
    movzx   ecx, byte [rel g_timed_out]
    test    ecx, ecx
    jz      .no_timeout

    ; timeout เกิดขึ้น
    mov     rax, -1
    jmp     .done

.no_timeout:
    ; อ่านสำเร็จ (rax = bytes read)

.done:
    pop     r13
    pop     r12
    pop     rbx
    leave
    ret

; =============================================================
; _start
; =============================================================
_start:
    ; แสดงข้อความ
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_wait]
    mov     edx, msg_wait_l
    syscall

    ; อ่านพร้อม timeout 5 วินาที
    lea     rdi, [rel input_buf]
    mov     rsi, 255
    mov     rdx, 5
    call    read_with_timeout

    ; ตรวจผลลัพธ์
    cmp     rax, -1
    je      .timeout_exit

    ; ได้รับ input
    mov     rcx, rax            ; เก็บขนาด input

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_got_in]
    mov     rdx, 14
    syscall

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel input_buf]
    mov     rdx, rcx
    syscall

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_ok]
    mov     edx, msg_ok_l
    syscall

    xor     edi, edi
    jmp     .exit

.timeout_exit:
    mov     edi, 1              ; exit code 1 = timeout

.exit:
    mov     eax, SYS_exit
    syscall
```

---

## 13. โปรแกรมสมบูรณ์: SIGCHLD Child Reaper

```nasm
; =============================================================
; ไฟล์: sigchld_reaper.asm
; คำอธิบาย: fork หลาย children และ reap ด้วย SIGCHLD handler
; สร้าง: nasm -f elf64 sigchld_reaper.asm -o sigchld_reaper.o
;         ld sigchld_reaper.o -o sigchld_reaper
; =============================================================

section .data
    msg_parent_start    db  "Parent PID: ", 0
    msg_forking         db  "กำลัง fork child #", 0
    msg_child_running   db  "  Child กำลังทำงาน (PID: ", 0
    msg_child_done      db  ") เสร็จแล้ว", 10, 0
    msg_child_reaped    db  "SIGCHLD: reaped child PID=", 0
    msg_status          db  " status=", 0
    msg_newline         db  10, 0
    msg_all_done        db  "Children ทั้งหมดได้รับการ reap แล้ว", 10
    msg_all_done_l      equ $ - msg_all_done

    num_children        dq  3   ; จำนวน children ที่จะ fork

section .bss
    g_reaped_count  resq 1      ; จำนวน children ที่ reap แล้ว
    sa_sigchld      resb 152
    num_str_buf     resb 32

section .text
global _start

%define SYS_write           1
%define SYS_exit            60
%define SYS_fork            57
%define SYS_waitpid         61
%define SYS_getpid          39
%define SYS_nanosleep       35
%define SYS_rt_sigaction    13
%define SYS_rt_sigreturn    15
%define SIGCHLD             17
%define SA_RESTORER         0x04000000
%define SA_RESTART          0x10000000
%define SA_NOCLDSTOP        0x00000001
%define WNOHANG             1

section .data
    sleep_200ms:
        .sec  dq 0
        .nsec dq 200000000

section .text

__restore_rt_chld:
    mov     eax, SYS_rt_sigreturn
    syscall

; =============================================================
; uint64_to_str — แปลง uint64 เป็น string
; rdi = number, rsi = buffer
; return: rax = pointer to string start, rcx = length
; =============================================================
uint64_to_str:
    push    rbx
    mov     rbx, rdi
    lea     rdi, [rsi + 19]
    mov     byte [rdi], 0
    xor     ecx, ecx

    test    rbx, rbx
    jnz     .loop
    dec     rdi
    mov     byte [rdi], '0'
    mov     ecx, 1
    mov     rax, rdi
    pop     rbx
    ret

.loop:
    test    rbx, rbx
    jz      .done
    xor     edx, edx
    mov     rax, rbx
    mov     r8d, 10
    div     r8
    mov     rbx, rax
    add     dl, '0'
    dec     rdi
    mov     [rdi], dl
    inc     ecx
    jmp     .loop

.done:
    mov     rax, rdi
    pop     rbx
    ret

; =============================================================
; print_str — พิมพ์ null-terminated string
; rdi = pointer to string
; =============================================================
print_str:
    push    rbx
    mov     rbx, rdi

    ; หา length
    xor     ecx, ecx
.find_len:
    cmp     byte [rbx + rcx], 0
    je      .found_len
    inc     ecx
    jmp     .find_len
.found_len:
    test    ecx, ecx
    jz      .done

    mov     eax, SYS_write
    mov     edi, 1
    mov     rsi, rbx
    mov     edx, ecx
    syscall
.done:
    pop     rbx
    ret

; =============================================================
; print_num — พิมพ์ตัวเลข
; rdi = number
; =============================================================
print_num:
    push    rbx
    push    r12
    sub     rsp, 32

    mov     rbx, rdi
    lea     r12, [rsp]

    mov     rdi, rbx
    mov     rsi, r12
    call    uint64_to_str

    ; rax = pointer, rcx = length
    push    rcx
    mov     eax, SYS_write
    mov     edi, 1
    mov     rsi, rax
    pop     rdx
    syscall

    add     rsp, 32
    pop     r12
    pop     rbx
    ret

; =============================================================
; sigchld_handler — จัดการ SIGCHLD
; ต้องใช้ waitpid เพื่อ reap children
; =============================================================
sigchld_handler:
    push    rbx
    push    r12
    sub     rsp, 8              ; เก็บ wait status

.reap_loop:
    ; waitpid(-1, &status, WNOHANG)
    ; รอทุก children แบบ non-blocking
    mov     edi, -1             ; pid = -1 (ทุก children)
    lea     rsi, [rsp]          ; &status
    mov     edx, WNOHANG        ; ไม่บล็อก
    mov     eax, SYS_waitpid
    syscall

    ; ถ้า return <= 0 ไม่มี children แล้ว
    test    rax, rax
    jle     .done_reaping

    mov     rbx, rax            ; เก็บ pid ที่ reap

    ; นับจำนวน reaped
    inc     qword [rel g_reaped_count]

    ; แสดงข้อความ
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_child_reaped]
    mov     edx, 26
    syscall

    mov     rdi, rbx
    call    print_num

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_status]
    mov     edx, 8
    syscall

    movzx   rdi, word [rsp]     ; status
    call    print_num

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_newline]
    mov     edx, 1
    syscall

    jmp     .reap_loop

.done_reaping:
    add     rsp, 8
    pop     r12
    pop     rbx
    ret

; =============================================================
; install_sigchld — ติดตั้ง SIGCHLD handler
; =============================================================
install_sigchld:
    push    rbp
    mov     rbp, rsp

    lea     rdi, [rel sa_sigchld]
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    lea     rax, [rel sigchld_handler]
    mov     [rel sa_sigchld + 0], rax

    ; SA_RESTART | SA_NOCLDSTOP
    mov     qword [rel sa_sigchld + 8], SA_RESTORER | SA_RESTART | SA_NOCLDSTOP

    lea     rax, [rel __restore_rt_chld]
    mov     [rel sa_sigchld + 16], rax

    mov     edi, SIGCHLD
    lea     rsi, [rel sa_sigchld]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    leave
    ret

; =============================================================
; child_work — งานที่ child ทำ
; rdi = child number
; =============================================================
child_work:
    push    rbx
    mov     rbx, rdi

    ; แสดง "Child กำลังทำงาน"
    mov     rdi, rbx
    ; ... (แสดงข้อความ)

    ; sleep ตามหมายเลข child (200ms * child_num)
    lea     rdi, [rel sleep_200ms]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall

    ; exit ด้วย exit code = child number
    mov     edi, ebx
    mov     eax, SYS_exit
    syscall

    pop     rbx
    ret

; =============================================================
; _start
; =============================================================
_start:
    ; ติดตั้ง SIGCHLD handler
    call    install_sigchld

    ; ล้าง reap counter
    mov     qword [rel g_reaped_count], 0

    ; fork children
    xor     ebx, ebx            ; child counter
.fork_loop:
    cmp     rbx, [rel num_children]
    jge     .wait_children

    ; แสดง "Forking child #N"
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_forking]
    mov     edx, 19
    syscall
    mov     rdi, rbx
    call    print_num
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_newline]
    mov     edx, 1
    syscall

    ; fork
    mov     eax, SYS_fork
    syscall

    test    rax, rax
    jz      .child_process      ; rax=0 ใน child

    ; parent: เพิ่ม counter
    inc     rbx
    jmp     .fork_loop

.child_process:
    ; child process
    mov     rdi, rbx
    call    child_work
    ; child_work ไม่ return (มี exit syscall)

.wait_children:
    ; รอจนกว่า children ทั้งหมดจะ reap
.wait_loop:
    mov     rax, [rel g_reaped_count]
    cmp     rax, [rel num_children]
    jge     .all_reaped

    ; pause — รอ signal
    lea     rdi, [rel sleep_200ms]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall
    jmp     .wait_loop

.all_reaped:
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_all_done]
    mov     edx, msg_all_done_l
    syscall

    xor     edi, edi
    mov     eax, SYS_exit
    syscall
```

---

## 14. โปรแกรมสมบูรณ์: SIGSEGV Handler พร้อม Backtrace

```nasm
; =============================================================
; ไฟล์: sigsegv_handler.asm
; คำอธิบาย: จัดการ SIGSEGV แสดง fault address และ register state
;            ใช้ SA_SIGINFO เพื่อรับ siginfo_t และ ucontext_t
; สร้าง: nasm -f elf64 sigsegv_handler.asm -o sigsegv_handler.o
;         ld sigsegv_handler.o -o sigsegv_handler
; =============================================================

section .data
    msg_crash       db  10, "=== SEGMENTATION FAULT ===", 10
    msg_crash_l     equ $ - msg_crash
    msg_fault_addr  db  "Fault address : 0x", 0
    msg_rip         db  "RIP           : 0x", 0
    msg_rsp         db  "RSP           : 0x", 0
    msg_rax_str     db  "RAX           : 0x", 0
    msg_rbx_str     db  "RBX           : 0x", 0
    msg_si_code     db  "si_code       : ", 0
    msg_maperr      db  "  (SEGV_MAPERR: address not mapped)", 10
    msg_maperr_l    equ $ - msg_maperr
    msg_accerr      db  "  (SEGV_ACCERR: invalid permissions)", 10
    msg_accerr_l    equ $ - msg_accerr
    msg_nl          db  10, 0
    msg_eq          db  "========================", 10, 10
    msg_eq_l        equ $ - msg_eq
    msg_testing     db  "ทดสอบ SIGSEGV handler...", 10
    msg_testing_l   equ $ - msg_testing
    msg_bad_ptr     db  "กำลัง dereference NULL pointer...", 10
    msg_bad_ptr_l   equ $ - msg_bad_ptr

section .bss
    sa_sigsegv      resb 152
    hex_buf         resb 32

section .text
global _start

%define SYS_write           1
%define SYS_exit            60
%define SYS_rt_sigaction    13
%define SYS_rt_sigreturn    15
%define SIGSEGV             11
%define SA_SIGINFO          0x00000004
%define SA_RESTORER         0x04000000

; ucontext offsets บน x86-64
%define UC_MCONTEXT         40
%define GREGS_BASE          UC_MCONTEXT
%define REG_R8              (GREGS_BASE + 0*8)
%define REG_R9              (GREGS_BASE + 1*8)
%define REG_R10             (GREGS_BASE + 2*8)
%define REG_R11             (GREGS_BASE + 3*8)
%define REG_R12             (GREGS_BASE + 4*8)
%define REG_R13             (GREGS_BASE + 5*8)
%define REG_R14             (GREGS_BASE + 6*8)
%define REG_R15             (GREGS_BASE + 7*8)
%define REG_RDI             (GREGS_BASE + 8*8)
%define REG_RSI             (GREGS_BASE + 9*8)
%define REG_RBP             (GREGS_BASE + 10*8)
%define REG_RBX             (GREGS_BASE + 11*8)
%define REG_RDX             (GREGS_BASE + 12*8)
%define REG_RAX             (GREGS_BASE + 13*8)
%define REG_RCX             (GREGS_BASE + 14*8)
%define REG_RSP             (GREGS_BASE + 15*8)
%define REG_RIP             (GREGS_BASE + 16*8)

; siginfo_t offsets
%define SIGINFO_SIGNO   0
%define SIGINFO_ERRNO   4
%define SIGINFO_CODE    8
%define SIGINFO_ADDR    16      ; สำหรับ SIGSEGV

__restore_rt_segv:
    mov     eax, SYS_rt_sigreturn
    syscall

; =============================================================
; uint64_to_hex — แปลง uint64 เป็น hex string
; rdi = value, rsi = output buffer (min 18 bytes: "0x" + 16 hex + null)
; return: rax = pointer, rcx = length
; =============================================================
uint64_to_hex:
    push    rbx
    mov     rbx, rdi

    ; เขียน "0x" จะไม่เขียนที่นี่ เขียน hex ล้วนๆ
    lea     rdi, [rsi + 16]
    mov     byte [rdi], 0
    mov     ecx, 16

.hex_loop:
    mov     al, bl
    and     al, 0x0F
    cmp     al, 10
    jl      .digit
    add     al, 'a' - 10
    jmp     .store
.digit:
    add     al, '0'
.store:
    dec     rdi
    mov     [rdi], al
    shr     rbx, 4
    dec     ecx
    jnz     .hex_loop

    ; rdi ชี้ไปยัง start ของ hex string
    mov     rax, rdi
    mov     ecx, 16

    pop     rbx
    ret

; =============================================================
; print_hex_val — พิมพ์ label แล้วตามด้วย 0x... hex value
; rdi = label string (null-terminated)
; rsi = value
; =============================================================
print_hex_val:
    push    rbx
    push    r12
    sub     rsp, 32

    mov     rbx, rsi            ; เก็บ value

    ; พิมพ์ label
    push    rdi
    ; หา length ของ label
    mov     r12, rdi
    xor     ecx, ecx
.find_label_len:
    cmp     byte [r12 + rcx], 0
    je      .label_found
    inc     ecx
    jmp     .find_label_len
.label_found:
    mov     eax, SYS_write
    mov     edi, 2              ; stderr
    mov     rsi, r12
    mov     edx, ecx
    syscall
    pop     rdi

    ; แปลงเป็น hex
    mov     rdi, rbx
    lea     rsi, [rsp]
    call    uint64_to_hex

    ; พิมพ์ hex string
    mov     eax, SYS_write
    mov     edi, 2
    ; rax ยังชี้อยู่ที่ hex string
    mov     rsi, rax
    mov     edx, 16
    syscall

    ; newline
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_nl]
    mov     edx, 1
    syscall

    add     rsp, 32
    pop     r12
    pop     rbx
    ret

; =============================================================
; sigsegv_handler — SA_SIGINFO handler
; rdi = signo, rsi = siginfo_t*, rdx = ucontext_t*
; =============================================================
sigsegv_handler:
    push    rbx
    push    r12
    push    r13

    mov     rbx, rsi            ; siginfo_t*
    mov     r12, rdx            ; ucontext_t*

    ; แสดง header
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_crash]
    mov     edx, msg_crash_l
    syscall

    ; แสดง fault address จาก siginfo_t
    lea     rdi, [rel msg_fault_addr]
    mov     rsi, [rbx + SIGINFO_ADDR]
    call    print_hex_val

    ; แสดง si_code
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_si_code]
    mov     edx, 16
    syscall

    movsx   rax, dword [rbx + SIGINFO_CODE]
    cmp     eax, 1
    je      .maperr
    cmp     eax, 2
    je      .accerr
    jmp     .code_done

.maperr:
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_maperr]
    mov     edx, msg_maperr_l
    syscall
    jmp     .code_done

.accerr:
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_accerr]
    mov     edx, msg_accerr_l
    syscall

.code_done:
    ; แสดง RIP จาก ucontext_t
    lea     rdi, [rel msg_rip]
    mov     rsi, [r12 + REG_RIP]
    call    print_hex_val

    ; แสดง RSP
    lea     rdi, [rel msg_rsp]
    mov     rsi, [r12 + REG_RSP]
    call    print_hex_val

    ; แสดง RAX
    lea     rdi, [rel msg_rax_str]
    mov     rsi, [r12 + REG_RAX]
    call    print_hex_val

    ; แสดง RBX
    lea     rdi, [rel msg_rbx_str]
    mov     rsi, [r12 + REG_RBX]
    call    print_hex_val

    ; separator
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_eq]
    mov     edx, msg_eq_l
    syscall

    pop     r13
    pop     r12
    pop     rbx

    ; exit ด้วย code 139 (128 + SIGSEGV)
    mov     edi, 139
    mov     eax, SYS_exit
    syscall

; =============================================================
; install_sigsegv — ติดตั้ง SIGSEGV handler
; =============================================================
install_sigsegv:
    push    rbp
    mov     rbp, rsp

    lea     rdi, [rel sa_sigsegv]
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    lea     rax, [rel sigsegv_handler]
    mov     [rel sa_sigsegv + 0], rax

    ; SA_SIGINFO | SA_RESTORER
    mov     qword [rel sa_sigsegv + 8], SA_SIGINFO | SA_RESTORER

    lea     rax, [rel __restore_rt_segv]
    mov     [rel sa_sigsegv + 16], rax

    mov     edi, SIGSEGV
    lea     rsi, [rel sa_sigsegv]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    leave
    ret

; =============================================================
; _start — ทดสอบ SIGSEGV handler
; =============================================================
_start:
    ; ติดตั้ง handler
    call    install_sigsegv

    ; แสดงข้อความทดสอบ
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_testing]
    mov     edx, msg_testing_l
    syscall

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_bad_ptr]
    mov     edx, msg_bad_ptr_l
    syscall

    ; ทำให้เกิด SIGSEGV โดยตั้งใจ
    ; อ่านจาก NULL pointer
    xor     rax, rax
    mov     rbx, [rax]      ; ← SIGSEGV เกิดที่นี่!

    ; ไม่ถึงบรรทัดนี้เพราะ handler เรียก exit
    xor     edi, edi
    mov     eax, SYS_exit
    syscall
```

---

## 15. โปรแกรมสมบูรณ์: Watchdog ด้วย SIGALRM + setitimer

```nasm
; =============================================================
; ไฟล์: watchdog_timer.asm
; คำอธิบาย: Watchdog timer ที่ใช้ setitimer สำหรับ periodic check
;            ตรวจสอบ "heartbeat" ทุก 1 วินาที
;            ถ้าไม่มี heartbeat ใน 3 ครั้ง ถือว่า process ค้าง
; สร้าง: nasm -f elf64 watchdog_timer.asm -o watchdog_timer.o
;         ld watchdog_timer.o -o watchdog_timer
; =============================================================

section .data
    msg_watchdog    db  "=== Watchdog Timer Demo ===", 10
    msg_watchdog_l  equ $ - msg_watchdog
    msg_tick        db  "SIGALRM tick #", 0
    msg_heartbeat   db  "  Heartbeat OK", 10
    msg_heartbeat_l equ $ - msg_heartbeat
    msg_no_hb       db  "  WARNING: No heartbeat! (", 0
    msg_no_hb2      db  "/3)", 10, 0
    msg_dead        db  10, "CRITICAL: Process appears dead! Restarting...", 10
    msg_dead_l      equ $ - msg_dead
    msg_main_work   db  "Main: ทำงาน iteration ", 0
    msg_main_done   db  10, "Main: เสร็จสิ้น", 10
    msg_main_done_l equ $ - msg_main_done

section .bss
    g_heartbeat         resq 1  ; counter heartbeat จาก main loop
    g_last_heartbeat    resq 1  ; heartbeat ที่ watchdog เห็นล่าสุด
    g_miss_count        resq 1  ; จำนวนครั้งที่ไม่มี heartbeat
    g_tick_count        resq 1  ; จำนวน SIGALRM ที่รับ
    g_running           resb 1  ; main loop flag
    sa_watchdog         resb 152
    sa_sigint_wd        resb 152

section .text
global _start

%define SYS_write           1
%define SYS_exit            60
%define SYS_nanosleep       35
%define SYS_setitimer       38
%define SYS_rt_sigaction    13
%define SYS_rt_sigreturn    15
%define SIGALRM             14
%define SIGINT              2
%define SA_RESTORER         0x04000000
%define SA_RESTART          0x10000000
%define ITIMER_REAL         0

section .data
    ; itimerval: 1 วินาที periodic
    wd_itimerval:
        .int_sec    dq  1       ; interval: 1 วินาที
        .int_usec   dq  0
        .val_sec    dq  1       ; initial: 1 วินาที
        .val_usec   dq  0

    ; nanosleep: 300ms
    sleep_300ms:
        .sec    dq  0
        .nsec   dq  300000000

section .text

__restore_rt_wd:
    mov     eax, SYS_rt_sigreturn
    syscall

; =============================================================
; print_num_wd — พิมพ์ตัวเลข (สำหรับใช้ใน handler)
; rdi = number, rsi = fd
; =============================================================
print_num_wd:
    push    rbx
    push    r12
    push    r13
    sub     rsp, 32

    mov     rbx, rdi            ; number
    mov     r12, rsi            ; fd
    lea     r13, [rsp]

    ; แปลงเป็นตัวเลข
    lea     rdi, [r13 + 19]
    mov     byte [rdi], 0
    xor     ecx, ecx

    test    rbx, rbx
    jnz     .num_loop
    dec     rdi
    mov     byte [rdi], '0'
    mov     ecx, 1
    jmp     .num_done

.num_loop:
    test    rbx, rbx
    jz      .num_done
    xor     edx, edx
    mov     rax, rbx
    mov     r8d, 10
    div     r8
    mov     rbx, rax
    add     dl, '0'
    dec     rdi
    mov     [rdi], dl
    inc     ecx
    jmp     .num_loop

.num_done:
    mov     eax, SYS_write
    mov     edi, r12d
    mov     rsi, rdi
    mov     edx, ecx
    syscall

    add     rsp, 32
    pop     r13
    pop     r12
    pop     rbx
    ret

; =============================================================
; watchdog_handler — SIGALRM handler
; ตรวจสอบ heartbeat ทุกครั้งที่ timer ทำงาน
; =============================================================
watchdog_handler:
    push    rbx
    push    r12

    ; เพิ่ม tick counter
    inc     qword [rel g_tick_count]
    mov     rbx, [rel g_tick_count]

    ; แสดง "SIGALRM tick #N"
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_tick]
    mov     edx, 14
    syscall

    mov     rdi, rbx
    mov     rsi, 1
    call    print_num_wd

    ; ตรวจสอบ heartbeat
    mov     rax, [rel g_heartbeat]
    mov     rbx, [rel g_last_heartbeat]

    cmp     rax, rbx
    jg      .heartbeat_ok           ; heartbeat เพิ่มขึ้น = OK

    ; ไม่มี heartbeat
    inc     qword [rel g_miss_count]
    mov     r12, [rel g_miss_count]

    ; แสดง "No heartbeat (N/3)"
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_no_hb]
    mov     edx, 26
    syscall

    mov     rdi, r12
    mov     rsi, 1
    call    print_num_wd

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_no_hb2]
    mov     edx, 4
    syscall

    ; ถ้า miss 3 ครั้งติดกัน → ถือว่าค้าง
    cmp     r12, 3
    jl      .done

    ; แสดง critical message
    mov     eax, SYS_write
    mov     edi, 2
    lea     rsi, [rel msg_dead]
    mov     edx, msg_dead_l
    syscall

    ; หยุด main loop
    mov     byte [rel g_running], 0

    ; ปิด watchdog timer
    ; setitimer(ITIMER_REAL, {0,0,0,0}, NULL)
    sub     rsp, 32
    xor     eax, eax
    mov     [rsp], rax
    mov     [rsp+8], rax
    mov     [rsp+16], rax
    mov     [rsp+24], rax
    mov     edi, ITIMER_REAL
    mov     rsi, rsp
    xor     edx, edx
    mov     eax, SYS_setitimer
    syscall
    add     rsp, 32
    jmp     .done

.heartbeat_ok:
    ; อัพเดต last_heartbeat
    mov     [rel g_last_heartbeat], rax
    mov     qword [rel g_miss_count], 0   ; reset miss count

    ; แสดง "Heartbeat OK"
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_heartbeat]
    mov     edx, msg_heartbeat_l
    syscall

.done:
    pop     r12
    pop     rbx
    ret

; =============================================================
; sigint_wd_handler — จัดการ SIGINT ใน watchdog demo
; =============================================================
sigint_wd_handler:
    mov     byte [rel g_running], 0
    ret

; =============================================================
; setup_watchdog — ตั้งค่า watchdog timer
; =============================================================
setup_watchdog:
    push    rbp
    mov     rbp, rsp

    ; ติดตั้ง SIGALRM handler
    lea     rdi, [rel sa_watchdog]
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    lea     rax, [rel watchdog_handler]
    mov     [rel sa_watchdog + 0], rax
    mov     qword [rel sa_watchdog + 8], SA_RESTORER | SA_RESTART
    lea     rax, [rel __restore_rt_wd]
    mov     [rel sa_watchdog + 16], rax

    mov     edi, SIGALRM
    lea     rsi, [rel sa_watchdog]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    ; ติดตั้ง SIGINT handler
    lea     rdi, [rel sa_sigint_wd]
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    lea     rax, [rel sigint_wd_handler]
    mov     [rel sa_sigint_wd + 0], rax
    mov     qword [rel sa_sigint_wd + 8], SA_RESTORER
    lea     rax, [rel __restore_rt_wd]
    mov     [rel sa_sigint_wd + 16], rax

    mov     edi, SIGINT
    lea     rsi, [rel sa_sigint_wd]
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    ; เริ่ม periodic timer ทุก 1 วินาที
    mov     edi, ITIMER_REAL
    lea     rsi, [rel wd_itimerval]
    xor     edx, edx
    mov     eax, SYS_setitimer
    syscall

    leave
    ret

; =============================================================
; _start — main watchdog demo
; =============================================================
_start:
    ; ตั้งค่าเริ่มต้น
    mov     qword [rel g_heartbeat], 0
    mov     qword [rel g_last_heartbeat], 0
    mov     qword [rel g_miss_count], 0
    mov     qword [rel g_tick_count], 0
    mov     byte [rel g_running], 1

    ; ตั้งค่า watchdog
    call    setup_watchdog

    ; แสดง header
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_watchdog]
    mov     edx, msg_watchdog_l
    syscall

    ; main loop: ทำงานแล้วส่ง heartbeat
    ; จำลองว่าจะหยุดส่ง heartbeat หลัง 5 iterations
    xor     ebx, ebx            ; iteration counter
.main_loop:
    movzx   eax, byte [rel g_running]
    test    eax, eax
    jz      .main_done

    ; แสดง "Main: ทำงาน iteration N"
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_main_work]
    mov     edx, 22
    syscall
    mov     rdi, rbx
    mov     rsi, 1
    call    print_num_wd
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_main_done]
    mov     edx, 1              ; เฉพาะ newline
    syscall

    inc     rbx

    ; ส่ง heartbeat เฉพาะ 5 iteration แรก
    cmp     rbx, 5
    jg      .no_heartbeat
    inc     qword [rel g_heartbeat]
.no_heartbeat:

    ; sleep 300ms
    lea     rdi, [rel sleep_300ms]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall

    jmp     .main_loop

.main_done:
    ; แสดงข้อความจบ
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_main_done]
    mov     edx, msg_main_done_l
    syscall

    xor     edi, edi
    mov     eax, SYS_exit
    syscall
```

---

## 16. โปรแกรมสมบูรณ์: Signal-based IPC

```nasm
; =============================================================
; ไฟล์: signal_ipc.asm
; คำอธิบาย: การสื่อสารระหว่าง processes โดยใช้ SIGUSR1/SIGUSR2
;            Parent fork child แล้วใช้ signal แทน pipe
;            SIGUSR1 = "เริ่มทำงาน"
;            SIGUSR2 = "เสร็จแล้ว"
; สร้าง: nasm -f elf64 signal_ipc.asm -o signal_ipc.o
;         ld signal_ipc.o -o signal_ipc
; =============================================================

section .data
    msg_parent_wait     db  "[Parent] รอ SIGUSR2 จาก child...", 10
    msg_parent_wait_l   equ $ - msg_parent_wait
    msg_parent_recv     db  "[Parent] ได้รับ SIGUSR2 — child เสร็จแล้ว!", 10
    msg_parent_recv_l   equ $ - msg_parent_recv
    msg_parent_send     db  "[Parent] ส่ง SIGUSR1 → child (เริ่มทำงาน)", 10
    msg_parent_send_l   equ $ - msg_parent_send
    msg_child_recv      db  "[Child]  ได้รับ SIGUSR1 — เริ่มทำงาน!", 10
    msg_child_recv_l    equ $ - msg_child_recv
    msg_child_working   db  "[Child]  กำลังทำงาน...", 10
    msg_child_working_l equ $ - msg_child_working
    msg_child_done      db  "[Child]  เสร็จแล้ว ส่ง SIGUSR2 → parent", 10
    msg_child_done_l    equ $ - msg_child_done
    msg_ipc_done        db  "[Parent] IPC สำเร็จ! จบโปรแกรม", 10
    msg_ipc_done_l      equ $ - msg_ipc_done

section .bss
    g_got_sigusr1   resb 1      ; child: ได้รับ SIGUSR1
    g_got_sigusr2   resb 1      ; parent: ได้รับ SIGUSR2
    g_parent_pid    resq 1      ; PID ของ parent
    g_child_pid     resq 1      ; PID ของ child
    sa_usr1         resb 152
    sa_usr2         resb 152
    sigwait_mask    resb 16     ; sigset สำหรับ sigsuspend

section .text
global _start

%define SYS_write           1
%define SYS_exit            60
%define SYS_fork            57
%define SYS_getpid          39
%define SYS_kill            62
%define SYS_nanosleep       35
%define SYS_rt_sigaction    13
%define SYS_rt_sigprocmask  14
%define SYS_rt_sigsuspend   130
%define SYS_waitpid         61
%define SYS_rt_sigreturn    15
%define SIGUSR1             10
%define SIGUSR2             12
%define SA_RESTORER         0x04000000
%define SA_RESTART          0x10000000
%define SIG_BLOCK           0

section .data
    sleep_500ms:
        .sec    dq  0
        .nsec   dq  500000000

section .text

__restore_rt_ipc:
    mov     eax, SYS_rt_sigreturn
    syscall

; =============================================================
; sigusr1_handler — child รับจาก parent
; =============================================================
sigusr1_handler:
    mov     byte [rel g_got_sigusr1], 1

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_child_recv]
    mov     edx, msg_child_recv_l
    syscall
    ret

; =============================================================
; sigusr2_handler — parent รับจาก child
; =============================================================
sigusr2_handler:
    mov     byte [rel g_got_sigusr2], 1

    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_parent_recv]
    mov     edx, msg_parent_recv_l
    syscall
    ret

; =============================================================
; install_signal — ติดตั้ง handler สำหรับ signal
; rdi = signum, rsi = handler, rdx = sa buffer
; =============================================================
install_signal:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    r12
    push    r13

    mov     rbx, rdi            ; signum
    mov     r12, rsi            ; handler
    mov     r13, rdx            ; sa buffer

    ; ล้าง buffer
    mov     rdi, r13
    xor     esi, esi
    mov     ecx, 152/8
    rep     stosq

    mov     [r13 + 0], r12      ; sa_handler
    mov     qword [r13 + 8], SA_RESTORER | SA_RESTART
    lea     rax, [rel __restore_rt_ipc]
    mov     [r13 + 16], rax     ; sa_restorer

    mov     rdi, rbx
    mov     rsi, r13
    xor     edx, edx
    mov     r10, 8
    mov     eax, SYS_rt_sigaction
    syscall

    pop     r13
    pop     r12
    pop     rbx
    leave
    ret

; =============================================================
; wait_for_signal_ipc — รอ signal โดยใช้ sigsuspend
; rdi = signal flag address (byte), rsi = signal to wait (ใช้ mask)
; =============================================================
wait_for_signal_ipc:
    push    rbx
    push    r12

    mov     rbx, rdi            ; flag address

    ; สร้าง mask ที่บล็อกทุกอย่าง
    ; (sigsuspend จะรอ signal ที่ไม่ถูก block)
    ; เราจะสร้าง mask ที่บล็อกทุกอย่างยกเว้น signal ที่รอ
    ; แต่ตอนนี้ทำแบบง่ายๆ: ใช้ flag polling กับ nanosleep

.poll_loop:
    movzx   eax, byte [rbx]
    test    eax, eax
    jnz     .got_signal

    ; sleep 10ms รอ signal
    lea     rdi, [rel sleep_500ms]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall
    jmp     .poll_loop

.got_signal:
    pop     r12
    pop     rbx
    ret

; =============================================================
; child_process_ipc — โค้ดสำหรับ child
; =============================================================
child_process_ipc:
    push    rbp
    mov     rbp, rsp

    ; ติดตั้ง SIGUSR1 handler
    mov     rdi, SIGUSR1
    lea     rsi, [rel sigusr1_handler]
    lea     rdx, [rel sa_usr1]
    call    install_signal

    ; รอ SIGUSR1 จาก parent
    lea     rdi, [rel g_got_sigusr1]
    xor     esi, esi
    call    wait_for_signal_ipc

    ; เริ่มทำงาน
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_child_working]
    mov     edx, msg_child_working_l
    syscall

    ; จำลองการทำงาน (500ms)
    lea     rdi, [rel sleep_500ms]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall

    ; แสดงว่าเสร็จ
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_child_done]
    mov     edx, msg_child_done_l
    syscall

    ; ส่ง SIGUSR2 ไปยัง parent
    mov     rdi, [rel g_parent_pid]
    mov     rsi, SIGUSR2
    mov     eax, SYS_kill
    syscall

    ; child exit
    xor     edi, edi
    mov     eax, SYS_exit
    syscall

    leave
    ret

; =============================================================
; _start — main IPC demo
; =============================================================
_start:
    ; เก็บ PID ของ parent
    mov     eax, SYS_getpid
    syscall
    mov     [rel g_parent_pid], rax

    ; ล้าง flags
    mov     byte [rel g_got_sigusr1], 0
    mov     byte [rel g_got_sigusr2], 0

    ; ติดตั้ง SIGUSR2 handler (สำหรับ parent)
    mov     rdi, SIGUSR2
    lea     rsi, [rel sigusr2_handler]
    lea     rdx, [rel sa_usr2]
    call    install_signal

    ; fork child
    mov     eax, SYS_fork
    syscall

    test    rax, rax
    jz      child_process_ipc   ; rax=0 ใน child

    ; parent: เก็บ child PID
    mov     [rel g_child_pid], rax

    ; แสดงว่ากำลังส่ง SIGUSR1
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_parent_send]
    mov     edx, msg_parent_send_l
    syscall

    ; sleep เล็กน้อยก่อนส่ง signal
    lea     rdi, [rel sleep_500ms]
    xor     esi, esi
    mov     eax, SYS_nanosleep
    syscall

    ; ส่ง SIGUSR1 ไปยัง child
    mov     rdi, [rel g_child_pid]
    mov     rsi, SIGUSR1
    mov     eax, SYS_kill
    syscall

    ; แสดงว่ารอ SIGUSR2
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_parent_wait]
    mov     edx, msg_parent_wait_l
    syscall

    ; รอ SIGUSR2 จาก child
    lea     rdi, [rel g_got_sigusr2]
    xor     esi, esi
    call    wait_for_signal_ipc

    ; reap child
    mov     edi, -1
    xor     esi, esi
    xor     edx, edx
    mov     eax, SYS_waitpid
    syscall

    ; IPC สำเร็จ
    mov     eax, SYS_write
    mov     edi, 1
    lea     rsi, [rel msg_ipc_done]
    mov     edx, msg_ipc_done_l
    syscall

    xor     edi, edi
    mov     eax, SYS_exit
    syscall
```

---

## 17. Realtime Signals (SIGRTMIN ถึง SIGRTMAX)

```nasm
; Realtime signals (Linux)
; SIGRTMIN = 34 (ต้องตรวจจาก runtime ไม่ใช่ hardcode!)
; SIGRTMAX = 64
; คุณสมบัติพิเศษ:
;   - Queued: ถ้าส่ง N ครั้ง จะ deliver N ครั้ง (ต่างจาก standard signals)
;   - Priority: SIGRTMIN+0 สูงกว่า SIGRTMIN+1
;   - สามารถส่ง data ไปพร้อมกับ signal ได้ (ผ่าน sigqueue)

; sigqueue syscall
; syscall number: 129 (x86-64)
; rdi = pid
; rsi = signum
; rdx = value (จะส่งใน siginfo_t.si_value)

%define SYS_sigqueue    129

; ตัวอย่าง: ส่ง realtime signal พร้อม data
send_rt_signal:
    ; rdi = pid, rsi = signum, rdx = data value
    push    rbx
    mov     rbx, rdx            ; เก็บ data

    ; sigqueue(pid, signum, sigval)
    ; rdx ยังคง = data (sigval เป็น union, ใช้ int/ptr)
    mov     eax, SYS_sigqueue
    syscall

    pop     rbx
    ret

; รับ realtime signal พร้อม data ใช้ SA_SIGINFO
; ใน handler (rdi=signo, rsi=siginfo*, rdx=ucontext*):
;   mov eax, [rsi + 16]    ; si_value.sival_int
;   หรือ
;   mov rax, [rsi + 16]    ; si_value.sival_ptr
```

---

## 18. Signal Safety และ Best Practices

### 18.1 Pattern: Flag-based Handler

```nasm
; แนวทางที่ปลอดภัยที่สุด: handler แค่ตั้ง flag
; แล้วให้ main loop ทำงานจริง

section .bss
    g_sig_flags resq 32         ; 32 signals, แต่ละตัว 1 byte ใน qword

; handler สำหรับทุก signal
generic_handler:
    ; rdi = signum (ถ้าใช้ simple handler)
    ; เพียงแค่ตั้ง flag
    mov     qword [rel g_sig_flags + rdi*8], 1
    ret

; main loop ตรวจ flags
check_signals:
    ; ตรวจ SIGINT
    cmp     qword [rel g_sig_flags + 2*8], 0
    je      .no_sigint
    mov     qword [rel g_sig_flags + 2*8], 0
    ; จัดการ SIGINT ที่นี่ (ปลอดภัย เพราะเป็น main context)
    call    handle_sigint_safely
.no_sigint:

    ; ตรวจ SIGTERM
    cmp     qword [rel g_sig_flags + 15*8], 0
    je      .no_sigterm
    mov     qword [rel g_sig_flags + 15*8], 0
    call    handle_sigterm_safely
.no_sigterm:

    ; ... ตรวจ signals อื่นๆ
    ret
```

### 18.2 Atomic Operations ใน Signal Handler

```nasm
; การ set/check flag ใน signal handler ต้องใช้ volatile-like access
; ใน C ใช้ volatile sig_atomic_t
; ใน Assembly ใช้ memory operations โดยตรง (ไม่มี register caching)

; การอ่าน flag (atomic บน aligned access)
read_flag:
    ; xchg หรือ cmpxchg สำหรับ atomic read-clear
    xor     eax, eax
    xchg    dword [rel g_signal_flag], eax  ; atomic read and clear
    ret

; การเขียน flag ใน handler
set_flag:
    mov     dword [rel g_signal_flag], 1    ; aligned store = atomic บน x86
    ; ไม่ต้องการ lock prefix สำหรับ store เดียว
    ; แต่ถ้าต้องการ memory barrier:
    ; mfence  ; หรือ
    ; lock or dword [rel g_signal_flag], 0  ; no-op แต่มี memory barrier
    ret
```

### 18.3 Alternate Signal Stack

```nasm
; ใช้ stack แยกต่างหากสำหรับ signal handler
; สำคัญมากสำหรับ SIGSEGV handler (ถ้า stack overflow ทำให้ SIGSEGV)

; stack_t structure:
; ss_sp    resq 1   ; pointer to stack
; ss_flags resd 1   ; 0 = enable, SS_DISABLE = disable
; padding  resd 1
; ss_size  resq 1   ; stack size

%define SYS_sigaltstack     131
%define SS_DISABLE          2
ALT_STACK_SIZE  equ 16384   ; 16KB

section .bss
    alt_stack_memory    resb ALT_STACK_SIZE
    alt_stack_info      resb 24         ; stack_t

section .text
setup_altstack:
    push    rbp
    mov     rbp, rsp

    ; ตั้งค่า stack_t
    lea     rax, [rel alt_stack_memory + ALT_STACK_SIZE - 8]
    ; align to 16 bytes
    and     rax, ~15

    ; ss_sp = ต้นของ alt stack (แต่ stack เติบโตลง ดังนั้นต้อง top)
    lea     rcx, [rel alt_stack_memory]
    mov     [rel alt_stack_info + 0], rcx   ; ss_sp = base

    ; ss_flags = 0
    mov     dword [rel alt_stack_info + 8], 0

    ; ss_size = ALT_STACK_SIZE
    mov     qword [rel alt_stack_info + 16], ALT_STACK_SIZE

    ; sigaltstack(&new_stack, NULL)
    lea     rdi, [rel alt_stack_info]
    xor     esi, esi
    mov     eax, SYS_sigaltstack
    syscall

    leave
    ret

; เมื่อติดตั้ง handler ที่ใช้ alt stack:
; sa_flags |= SA_ONSTACK
```

---

## 19. Signal Masking ใน Multi-threaded Programs

```nasm
; ใน multi-threaded program:
; - Signal mask เป็นแบบ per-thread
; - Signal delivery ไปยัง thread ที่ไม่ได้ block signal นั้น
; - ใช้ pthread_sigmask แทน sigprocmask สำหรับ threads
;   (แต่ kernel syscall เหมือนกัน คือ rt_sigprocmask)

; แนวทางที่ดี:
; 1. Block ทุก signals ใน main thread ก่อน create threads
; 2. สร้าง dedicated signal-handling thread
; 3. Signal-handling thread ใช้ sigwait() รอรับ signals

; sigwait syscall (via rt_sigtimedwait)
%define SYS_rt_sigtimedwait  128

; rt_sigtimedwait(sigset*, siginfo*, timespec*, sigsetsize)
; rdi = pointer to sigset (signals to wait for)
; rsi = pointer to siginfo_t (output, หรือ NULL)
; rdx = pointer to timespec (timeout, หรือ NULL = infinite)
; r10 = sizeof(sigset_t) = 8
; return: signal number หรือ -errno

wait_for_signals_sync:
    ; rdi = sigset* (ชุด signals ที่รอ)
    ; rsi = siginfo* (output)
    push    rbp
    mov     rbp, rsp
    push    rbx

    mov     rbx, rdi

    mov     rdi, rbx
    ; rsi = siginfo (ถ้ามี)
    xor     edx, edx            ; timeout = NULL (infinite)
    mov     r10, 8
    mov     eax, SYS_rt_sigtimedwait
    syscall

    ; rax = signal number ที่ได้รับ

    pop     rbx
    leave
    ret
```

---

## 20. สรุปและ Reference Sheet

### 20.1 Syscall Numbers (x86-64 Linux)

```
Syscall               Number   ใช้สำหรับ
-------               ------   ---------
rt_sigaction          13       ติดตั้ง/ถาม signal handler
rt_sigprocmask        14       block/unblock signals
rt_sigreturn          15       return จาก signal handler
rt_sigpending         127      ตรวจ pending signals
rt_sigtimedwait       128      รอ signal (synchronous)
sigqueue              129      ส่ง signal พร้อม data
rt_sigsuspend         130      suspend รอ signal
sigaltstack           131      ตั้ง alternate signal stack
kill                  62       ส่ง signal ไปยัง process/group
tkill                 200      ส่ง signal ไปยัง thread
tgkill                234      ส่ง signal ไปยัง thread ใน group
alarm                 37       ตั้ง alarm (วินาที)
setitimer             38       ตั้ง interval timer
getitimer             36       อ่าน interval timer
```

### 20.2 Signal Disposition Values

```
SIG_DFL  equ 0    ; default action (ขึ้นกับแต่ละ signal)
SIG_IGN  equ 1    ; ignore signal
; ค่าอื่นๆ คือ pointer ไปยัง handler function
```

### 20.3 Common Patterns สรุป

```nasm
; Pattern 1: Simple handler (ไม่รับ info)
; sa_flags: SA_RESTORER
; handler signature: void handler(int signo)

; Pattern 2: Extended handler (รับ siginfo + ucontext)  
; sa_flags: SA_SIGINFO | SA_RESTORER
; handler signature: void handler(int signo, siginfo_t*, ucontext_t*)

; Pattern 3: One-shot handler (reset หลัง delivery)
; sa_flags: SA_RESTORER | SA_RESETHAND

; Pattern 4: Restartable syscalls
; sa_flags: SA_RESTORER | SA_RESTART

; Pattern 5: No-stop child notification
; sa_flags: SA_RESTORER | SA_NOCLDSTOP   ; สำหรับ SIGCHLD

; Pattern 6: Alternate stack (สำหรับ stack overflow recovery)
; sa_flags: SA_RESTORER | SA_ONSTACK
; ต้อง setup altstack ก่อนด้วย sigaltstack()
```

### 20.4 Build Commands สรุป

```bash
# compile
nasm -f elf64 signal_program.asm -o signal_program.o

# link (no libc)
ld signal_program.o -o signal_program

# ถ้าต้องการ debug symbols
nasm -f elf64 -g -F dwarf signal_program.asm -o signal_program.o
ld signal_program.o -o signal_program

# ทดสอบ
./signal_program &
PID=$!
kill -SIGUSR1 $PID    # ส่ง SIGUSR1
kill -SIGTERM $PID    # ส่ง SIGTERM
kill -9 $PID          # force kill

# ดู signals ที่ process รับ
cat /proc/$PID/status | grep -E 'Sig(Pnd|Blk|Ign|Cgt)'
```

### 20.5 /proc/PID/status Signal Fields

```
SigPnd: 0000000000000000   — pending signals (ใน thread)
ShdPnd: 0000000000000000   — pending signals (shared กับ process)
SigBlk: 0000000000000000   — blocked signals
SigIgn: 0000000000000000   — ignored signals  
SigCgt: 0000000000000000   — caught signals (มี handler)
```

แต่ละ field เป็น hex 64-bit bitmask
bit (N-1) = signal N
เช่น SigCgt: 0000000000000004 = bit 2 set = SIGINT(3) caught

---

## 21. ข้อควรระวังและ Common Mistakes

### 21.1 Stack Alignment ใน Signal Handler

```nasm
; Signal handler ถูกเรียกโดย kernel ที่ push extra frame บน stack
; Stack alignment อาจเปลี่ยนไป ต้องตรวจสอบเสมอ

signal_handler_safe:
    ; ตรวจ stack alignment
    mov     rax, rsp
    and     rax, 0xF
    test    rax, rax
    jz      .aligned
    ; ถ้าไม่ align ให้ adjust
    and     rsp, ~0xF
    push    rax             ; dummy push เพื่อ alignment
.aligned:
    push    rbp
    mov     rbp, rsp
    ; ตอนนี้ stack align แล้ว ปลอดภัยที่จะเรียก SSE instructions

    ; ... handler body ...

    leave
    ret
```

### 21.2 ระวัง errno ใน Signal Handler

```nasm
; ถ้า handler ถูก interrupt syscall ที่กำลังทำงาน
; การเรียก syscall ใน handler อาจ overwrite errno ของ main code
; ใน C: บันทึก/คืนค่า errno
; ใน Assembly: ระวังการใช้ rax หลัง syscall ใน handler
; ที่ return ค่า error

save_restore_errno:
    ; ถ้าใช้ C runtime errno (rip-relative __errno_location)
    ; ต้อง save errno ก่อน และ restore หลัง
    ; แต่สำหรับ pure Assembly ไม่มี errno global
    ; ค่า return จาก syscall (-EINTR ฯลฯ) อยู่ใน rax เท่านั้น
    ret
```

### 21.3 Re-entrant Code

```nasm
; signal handler ไม่ควรเรียกฟังก์ชันที่ใช้ global state
; เช่น ถ้า malloc กำลังทำงานแล้วเกิด signal
; แล้ว handler เรียก malloc อีก → deadlock หรือ corruption

; ปลอดภัย: ใช้ stack variables เท่านั้น
; อันตราย: แก้ไข global variables ที่ main code ก็ใช้
;           (ยกเว้น atomic flag แบบเดียว)
```

---

## 22. Advanced: Self-modifying Signal Handling

```nasm
; เทคนิค: ใช้ signal handler เพื่อ implement hot-reload
; เมื่อรับ SIGHUP ให้ reload configuration

section .bss
    config_version  resq 1
    g_config_path   resb 256
    g_reload_flag   resb 1

; sighup_handler
sighup_handler:
    mov     byte [rel g_reload_flag], 1
    ret

; main loop ตรวจ reload flag
check_reload:
    cmp     byte [rel g_reload_flag], 0
    je      .no_reload
    mov     byte [rel g_reload_flag], 0
    ; โหลด configuration ใหม่ที่นี่ (ปลอดภัยใน main context)
    call    reload_configuration
.no_reload:
    ret
```

---

## 23. Debugging Signal Issues

### 23.1 strace สำหรับ Signals

```bash
# ติดตาม signals ที่ process รับ
strace -e signal ./my_program

# ติดตาม syscalls ที่เกี่ยวกับ signal ทั้งหมด
strace -e trace=signal,kill ./my_program

# ดู signal handling แบบ verbose
strace -v -e rt_sigaction,rt_sigprocmask ./my_program
```

### 23.2 gdb สำหรับ Signal Debugging

```
(gdb) info signals              # ดู signal dispositions
(gdb) handle SIGINT nostop      # อย่าหยุดที่ SIGINT
(gdb) handle SIGSEGV stop       # หยุดเมื่อเกิด SIGSEGV
(gdb) signal SIGTERM            # ส่ง SIGTERM ไปยัง program
(gdb) catch signal SIGINT       # break เมื่อรับ SIGINT
```

### 23.3 ตรวจสอบด้วย /proc

```bash
# ดู signal mask ของ process
PID=1234
cat /proc/$PID/status | grep Sig

# แปลง hex bitmask
python3 -c "
mask = 0x0000000000000004
for i in range(64):
    if mask & (1 << i):
        print(f'Signal {i+1}')
"
```

---

## 24. บทสรุป (Conclusion)

Signal handling ใน Assembly ต้องการความเข้าใจลึกในหลายด้าน:

1. **โครงสร้างข้อมูล**: sigaction, siginfo_t, ucontext_t ต้องรู้ offsets แม่นยำ
2. **Syscall Interface**: rt_sigaction, rt_sigprocmask, rt_sigsuspend และอื่นๆ
3. **Safety Constraints**: เฉพาะ async-signal-safe functions ใน handler
4. **Kernel Trampoline**: ต้องมี `__restore_rt` เมื่อใช้ SA_RESTORER
5. **Stack Management**: stack alignment และ alternate stack สำหรับ SIGSEGV
6. **Atomic Operations**: การ set/read flags ต้องเป็น atomic

การเขียน signal handling ที่ถูกต้องใน Assembly เป็นทักษะที่สำคัญสำหรับ:
- System programming ระดับต่ำ
- การเขียน runtime environments
- Security research และ exploit development
- Performance-critical applications

การฝึกฝนด้วยโปรแกรมตัวอย่างในบทนี้จะช่วยให้เข้าใจกลไก signal
อย่างลึกซึ้งมากกว่าการใช้ wrapper functions ของ C library

---

*จบ Part 056: Signal Handling ใน Assembly*
