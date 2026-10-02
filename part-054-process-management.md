# Part 054: Process Management ใน Assembly

## บทนำ (Introduction)

Process Management เป็นหัวใจสำคัญของระบบปฏิบัติการ Linux การเข้าใจว่า process ทำงานอย่างไรในระดับ assembly จะช่วยให้เราเขียนโปรแกรมที่มีประสิทธิภาพและเข้าใจกลไกภายในของ OS ได้ลึกซึ้งขึ้น

ในบทนี้เราจะศึกษา:
- การสร้าง process ใหม่ด้วย `fork()`
- การแทนที่ process image ด้วย `exec` family
- การรอ child process ด้วย `wait/waitpid`
- การจัดการ signals
- การอ่านข้อมูล process จาก `/proc`
- การเขียนโปรแกรมประยุกต์ เช่น simple shell, daemon, supervisor

---

## 1. System Call Numbers ที่เกี่ยวข้อง

```nasm
; syscall numbers สำหรับ x86_64 Linux
; ดูได้จาก /usr/include/asm/unistd_64.h

SYS_read        equ 0
SYS_write       equ 1
SYS_open        equ 2
SYS_close       equ 3
SYS_waitpid     equ 7       ; waitpid (เก่า, บาง distro ใช้ wait4)
SYS_execve      equ 59      ; execve(path, argv, envp)
SYS_exit        equ 60      ; exit(status)
SYS_wait4       equ 61      ; wait4(pid, status, options, rusage)
SYS_kill        equ 62      ; kill(pid, sig)
SYS_getpid      equ 39      ; getpid()
SYS_getppid     equ 110     ; getppid()
SYS_getuid      equ 102     ; getuid()
SYS_geteuid     equ 107     ; geteuid()
SYS_getgid      equ 104     ; getgid()
SYS_getegid     equ 108     ; getegid()
SYS_setuid      equ 105     ; setuid(uid)
SYS_setgid      equ 106     ; setgid(gid)
SYS_fork        equ 57      ; fork()
SYS_chdir       equ 80      ; chdir(path)
SYS_getcwd      equ 79      ; getcwd(buf, size)
SYS_chroot      equ 161     ; chroot(path)
SYS_getrlimit   equ 97      ; getrlimit(resource, rlim)
SYS_setrlimit   equ 160     ; setrlimit(resource, rlim)
SYS_setsid      equ 112     ; setsid() - สร้าง new session
SYS_brk         equ 12      ; brk() - จัดการ heap
SYS_mmap        equ 9       ; mmap()
SYS_munmap      equ 11      ; munmap()
SYS_nanosleep   equ 35      ; nanosleep()
SYS_alarm       equ 37      ; alarm()
SYS_pause       equ 34      ; pause()
SYS_sigaction   equ 13      ; rt_sigaction (จริงๆ คือ 13)
SYS_rt_sigaction equ 13     ; rt_sigaction
SYS_umask       equ 95      ; umask()
SYS_dup2        equ 33      ; dup2(oldfd, newfd)
SYS_pipe        equ 22      ; pipe(pipefd)
```

---

## 2. fork() - Process Duplication

### ความเข้าใจพื้นฐาน

`fork()` คือ system call ที่สร้าง process ใหม่ (child) โดยการ copy process ปัจจุบัน (parent) เกือบทั้งหมด ได้แก่:
- Virtual memory space (ใช้ Copy-on-Write)
- File descriptors
- Signal handlers
- Process attributes (uid, gid, ฯลฯ)

**สิ่งที่แตกต่างกัน:**
- PID (Process ID) - child ได้ PID ใหม่
- PPID (Parent PID) - child มี PPID = PID ของ parent
- Return value ของ fork()

### Return Value ของ fork()

```
ใน parent process: RAX = PID ของ child (> 0)
ใน child process:  RAX = 0
ถ้า error:         RAX = -1 (เป็น negative errno)
```

### Copy-on-Write (COW)

เมื่อ fork() ถูกเรียก kernel **ไม่ได้คัดลอก** memory ทันที แต่ทั้ง parent และ child จะแชร์ page เดียวกันก่อน เมื่อใดที่ process ใดพยายามเขียนลงใน page นั้น kernel จะ copy page นั้นมาให้ process นั้นโดยเฉพาะ วิธีนี้ทำให้ fork() เร็วมาก

```
ก่อน fork():
  Parent: [Page A] [Page B] [Page C]
                    แชร์ได้

หลัง fork() (ก่อน write):
  Parent: [Page A] [Page B] [Page C]  <- ชี้ไปหน้าเดียวกัน
  Child:  [Page A] [Page B] [Page C]  <- เขียน read-only flag

เมื่อ child เขียน Page B:
  Parent: [Page A] [Page B] [Page C]
  Child:  [Page A] [Page B'] [Page C]  <- kernel copy Page B -> Page B'
```

### โปรแกรม: fork() พื้นฐาน

```nasm
; ไฟล์: fork_basic.asm
; สาธิต fork() และความแตกต่างระหว่าง parent/child
; คอมไพล์: nasm -f elf64 fork_basic.asm -o fork_basic.o && ld fork_basic.o -o fork_basic

section .data
    ; ข้อความสำหรับ parent
    msg_parent      db "ฉันคือ Parent Process, PID = ", 0
    msg_parent_len  equ $ - msg_parent
    
    ; ข้อความสำหรับ child
    msg_child       db "ฉันคือ Child Process, PID = ", 0
    msg_child_len   equ $ - msg_child
    
    ; ข้อความหลัง fork
    msg_child_pid   db "Child PID จาก parent = ", 0
    msg_child_pid_len equ $ - msg_child_pid
    
    newline         db 10, 0
    
section .bss
    pid_buf         resb 20     ; buffer สำหรับแปลง PID เป็น string

section .text
    global _start

_start:
    ; เรียก fork()
    mov rax, 57         ; SYS_fork
    syscall
    
    ; ตรวจสอบ return value
    cmp rax, 0
    je  child_process   ; ถ้า RAX = 0 แสดงว่าเราคือ child
    js  fork_error      ; ถ้า RAX < 0 (negative) แสดงว่า error
    
    ; ===== PARENT PROCESS =====
    ; RAX มีค่า PID ของ child
    push rax            ; เก็บ child PID ไว้ก่อน
    
    ; แสดงข้อความ parent
    mov rdi, 1          ; stdout
    mov rsi, msg_child_pid
    mov rdx, msg_child_pid_len
    mov rax, 1          ; SYS_write
    syscall
    
    ; แปลง child PID เป็น string และแสดง
    pop rdi             ; restore child PID
    call print_number
    
    ; แสดง newline
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
    ; แสดง PID ของตัวเอง
    mov rdi, 1
    mov rsi, msg_parent
    mov rdx, msg_parent_len
    mov rax, 1
    syscall
    
    mov rax, 39         ; SYS_getpid
    syscall
    mov rdi, rax        ; PID ของ parent
    call print_number
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
    ; Parent รอ child ด้วย wait4
    ; wait4(pid=-1, status, options=0, rusage=NULL)
    mov rdi, -1         ; รอ child ใดก็ได้
    mov rsi, 0          ; ไม่ต้องการ status
    mov rdx, 0          ; options = 0 (block)
    mov r10, 0          ; rusage = NULL
    mov rax, 61         ; SYS_wait4
    syscall
    
    ; exit(0)
    xor rdi, rdi
    mov rax, 60
    syscall

    ; ===== CHILD PROCESS =====
child_process:
    ; แสดงข้อความ child พร้อม PID ของตัวเอง
    mov rdi, 1
    mov rsi, msg_child
    mov rdx, msg_child_len
    mov rax, 1
    syscall
    
    mov rax, 39         ; SYS_getpid - ดึง PID ของ child
    syscall
    mov rdi, rax
    call print_number
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
    ; child exit
    xor rdi, rdi
    mov rax, 60
    syscall

fork_error:
    ; แสดง error
    mov rdi, 1
    mov rsi, msg_error
    mov rdx, msg_error_len
    mov rax, 1
    syscall
    
    mov rdi, 1
    mov rax, 60
    syscall

; ฟังก์ชัน: print_number
; input: RDI = number ที่จะแสดง (unsigned 64-bit)
print_number:
    push rbx
    push rcx
    push rdx
    push rsi
    
    ; แปลงตัวเลขเป็น ASCII
    mov rbx, pid_buf
    add rbx, 19         ; ชี้ไปปลาย buffer (เติมจากขวาไปซ้าย)
    mov byte [rbx], 10  ; newline ที่ปลาย (ไม่ใช้)
    dec rbx
    
    mov rax, rdi        ; ตัวเลขที่จะแปลง
    mov rcx, 10         ; ตัวหาร
    
.loop:
    xor rdx, rdx
    div rcx             ; RAX = quotient, RDX = remainder
    add dl, '0'         ; แปลง digit เป็น ASCII
    mov [rbx], dl
    dec rbx
    test rax, rax
    jnz .loop
    
    inc rbx             ; ชี้ไปที่ตัวแรก
    
    ; คำนวณความยาว
    lea rcx, [pid_buf + 19]
    sub rcx, rbx        ; length
    
    ; write syscall
    push rdi
    mov rdi, 1
    mov rsi, rbx
    mov rdx, rcx
    mov rax, 1
    syscall
    pop rdi
    
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

section .data
    msg_error       db "fork() failed!", 10, 0
    msg_error_len   equ $ - msg_error
```

---

## 3. exec Family - แทนที่ Process Image

### ความเข้าใจ exec

exec ไม่ได้สร้าง process ใหม่ แต่ **แทนที่** image ของ process ปัจจุบันด้วยโปรแกรมใหม่ หลังจาก exec สำเร็จ:
- Code segment เปลี่ยนเป็น program ใหม่
- Data/Stack/Heap ถูกรีเซ็ต
- PID ยังคงเดิม
- File descriptors ส่วนใหญ่ยังคงอยู่ (ยกเว้น FD_CLOEXEC)

### execve() System Call

```c
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

**Arguments:**
- `pathname`: path ของโปรแกรมที่จะรัน (ต้องเป็น absolute หรือ relative path)
- `argv`: array ของ pointer ไปยัง argument strings (ต้องจบด้วย NULL)
- `envp`: array ของ pointer ไปยัง environment strings (ต้องจบด้วย NULL)

### การสร้าง argv Array ใน Assembly

argv เป็น array ของ pointers ที่จบด้วย NULL pointer:

```
argv[0] -> "ls\0"      ; ชื่อโปรแกรม
argv[1] -> "-la\0"     ; argument แรก
argv[2] -> "/tmp\0"    ; argument สอง
argv[3] = 0            ; NULL terminator (จำเป็นมาก!)
```

```nasm
; ไฟล์: execve_demo.asm
; สาธิต execve() - รัน /bin/ls -la /tmp
; คอมไพล์: nasm -f elf64 execve_demo.asm -o execve_demo.o && ld execve_demo.o -o execve_demo

section .data
    ; Path ของโปรแกรมที่จะรัน
    prog_path   db "/bin/ls", 0
    
    ; Argument strings
    arg0        db "/bin/ls", 0     ; argv[0] = ชื่อโปรแกรม (convention)
    arg1        db "-la", 0         ; argv[1]
    arg2        db "/tmp", 0        ; argv[2]
    
    ; Environment strings
    env0        db "HOME=/root", 0
    env1        db "PATH=/bin:/usr/bin", 0
    
    msg_before  db "กำลังจะ exec /bin/ls...", 10, 0
    msg_before_len equ $ - msg_before
    
    ; สิ่งที่ไม่ควรเห็น: ถ้า exec สำเร็จ code นี้จะไม่ถูกรัน
    msg_after   db "ถ้าเห็นข้อความนี้ exec ล้มเหลว!", 10, 0
    msg_after_len equ $ - msg_after

section .bss
    ; argv array: 4 pointers (arg0, arg1, arg2, NULL)
    ; แต่ละ pointer = 8 bytes บน 64-bit
    argv_array  resq 4
    
    ; envp array: 3 pointers (env0, env1, NULL)
    envp_array  resq 3

section .text
    global _start

_start:
    ; แสดงข้อความก่อน exec
    mov rdi, 1
    mov rsi, msg_before
    mov rdx, msg_before_len
    mov rax, 1
    syscall
    
    ; สร้าง argv array
    mov qword [argv_array + 0*8], arg0   ; argv[0] = "/bin/ls"
    mov qword [argv_array + 1*8], arg1   ; argv[1] = "-la"
    mov qword [argv_array + 2*8], arg2   ; argv[2] = "/tmp"
    mov qword [argv_array + 3*8], 0      ; argv[3] = NULL (สำคัญมาก!)
    
    ; สร้าง envp array
    mov qword [envp_array + 0*8], env0   ; envp[0] = "HOME=/root"
    mov qword [envp_array + 1*8], env1   ; envp[1] = "PATH=..."
    mov qword [envp_array + 2*8], 0      ; envp[2] = NULL
    
    ; เรียก execve()
    mov rdi, prog_path      ; pathname
    mov rsi, argv_array     ; argv
    mov rdx, envp_array     ; envp
    mov rax, 59             ; SYS_execve
    syscall
    
    ; ถ้ามาถึงที่นี่ แสดงว่า exec ล้มเหลว
    mov rdi, 1
    mov rsi, msg_after
    mov rdx, msg_after_len
    mov rax, 1
    syscall
    
    ; exit ด้วย error code
    mov rdi, 1
    mov rax, 60
    syscall
```

### การสร้าง envp ที่ inherit จาก parent

ในทางปฏิบัติ เวลาที่เราต้องการส่ง environment เดิมไปให้ child process เราต้องเข้าถึง `environ` ซึ่งเก็บอยู่ที่ stack เมื่อโปรแกรมเริ่มทำงาน:

```nasm
; โครงสร้าง stack เมื่อโปรแกรมเริ่มทำงาน (_start):
; [RSP+0]  = argc
; [RSP+8]  = argv[0]
; [RSP+16] = argv[1]
; ...
; [RSP+8*(argc+1)] = NULL
; [RSP+8*(argc+2)] = envp[0]   <- environment variables เริ่มที่นี่
; ...

_start:
    mov rbx, [rsp]          ; argc
    lea rcx, [rsp + 8]      ; &argv[0]
    
    ; คำนวณตำแหน่ง envp
    ; envp อยู่หลัง argv array + NULL terminator
    lea rdx, [rsp + 8 + rbx*8 + 8]  ; &envp[0]
    ; rdx ชี้ไปที่ envp เดิมของ process
    ; สามารถส่งตรงๆ ให้ execve ได้เลย
```

---

## 4. wait() และ waitpid() - รอ Child Process

### ความสำคัญของการ wait

เมื่อ child process สิ้นสุด kernel จะเก็บ exit status ไว้ใน process table จนกว่า parent จะมา "รับ" ด้วย wait() ระหว่างนี้ child จะอยู่ในสถานะ **Zombie**

ถ้า parent ไม่เคย wait() เลย:
- Child กลายเป็น **Zombie process** ตลอดไป (จนกว่า parent จะตาย)
- Zombie ใช้ทรัพยากรน้อยมากแต่ยังครองที่ใน process table

### wait4() System Call (เวอร์ชัน Linux ที่ใช้จริง)

```c
pid_t wait4(pid_t pid, int *wstatus, int options, struct rusage *rusage);
```

**pid parameter:**
- `pid < -1`: รอ child ใดก็ได้ใน process group |pid|
- `pid = -1`: รอ child ใดก็ได้ (เหมือน wait())
- `pid = 0`: รอ child ในกลุ่มเดียวกับ caller
- `pid > 0`: รอ child ที่มี PID ตรงกัน

**options flags:**
```nasm
WNOHANG     equ 1   ; return ทันทีถ้าไม่มี child หยุด
WUNTRACED   equ 2   ; return ถ้า child หยุด (SIGSTOP)
WCONTINUED  equ 8   ; return ถ้า child continue (SIGCONT)
```

### การ Decode Wait Status

```nasm
; wstatus เป็น int 32-bit ที่มี encoding ดังนี้:
;
; ถ้า WIFEXITED(status) = true:
;   bits[7:0]  = 0
;   bits[15:8] = exit code (0-255)
;
; ถ้า WIFSIGNALED(status) = true:
;   bits[6:0]  = signal number
;   bit[7]     = core dump flag
;   bits[15:8] = 0
;
; ถ้า WIFSTOPPED(status) = true:
;   bits[7:0]  = 0x7F
;   bits[15:8] = stop signal

; Macros สำหรับ decode:
; WIFEXITED(s)   = ((s & 0x7F) == 0)        -> ออกปกติ
; WEXITSTATUS(s) = ((s >> 8) & 0xFF)         -> exit code
; WIFSIGNALED(s) = (((s & 0x7F) != 0) && ((s & 0x7F) != 0x7F))  -> ถูก signal kill
; WTERMSIG(s)    = (s & 0x7F)               -> signal ที่ kill
; WIFSTOPPED(s)  = ((s & 0xFF) == 0x7F)     -> หยุดชั่วคราว
; WSTOPSIG(s)    = ((s >> 8) & 0xFF)         -> signal ที่ stop

; ในภาษา Assembly:
decode_wait_status:
    ; input: RDI = wstatus (32-bit value)
    ; แก้ไขเพื่อ 32-bit operation
    mov eax, edi
    
    ; ตรวจสอบ WIFEXITED
    and eax, 0x7F
    test eax, eax
    jz .exited_normally     ; ถ้า lower 7 bits = 0 -> ออกปกติ
    
    ; ตรวจสอบ WIFSTOPPED
    cmp eax, 0x7F
    je .stopped             ; ถ้า lower 8 bits = 0x7F -> stopped
    
    ; WIFSIGNALED
    mov eax, edi
    and eax, 0x7F           ; signal number อยู่ใน lower 7 bits
    ret
    
.exited_normally:
    mov eax, edi
    shr eax, 8              ; WEXITSTATUS = bits 15:8
    and eax, 0xFF
    ret
    
.stopped:
    mov eax, edi
    shr eax, 8              ; WSTOPSIG
    and eax, 0xFF
    ret
```

### โปรแกรม: wait status demo

```nasm
; ไฟล์: wait_demo.asm
; สาธิต fork() + execve() + waitpid() พร้อม status decoding

section .data
    prog_ls     db "/bin/ls", 0
    arg_ls0     db "ls", 0
    arg_ls1     db "--invalid-option", 0    ; จงใจทำให้ error
    
    msg_fork    db "Parent: forked child", 10, 0
    msg_fork_len equ $ - msg_fork
    
    msg_waiting db "Parent: waiting for child...", 10, 0
    msg_waiting_len equ $ - msg_waiting
    
    msg_exited  db "Child exited normally with code: ", 0
    msg_exited_len equ $ - msg_exited
    
    msg_killed  db "Child killed by signal: ", 0
    msg_killed_len equ $ - msg_killed
    
    newline     db 10
    
section .bss
    wstatus     resd 1          ; wait status (4 bytes)
    argv_buf    resq 4          ; argv array
    envp_buf    resq 2          ; envp array (ว่าง + NULL)
    num_buf     resb 20

section .text
    global _start

_start:
    ; เตรียม argv
    mov qword [argv_buf + 0], arg_ls0
    mov qword [argv_buf + 1*8], arg_ls1
    mov qword [argv_buf + 2*8], 0       ; NULL terminator
    
    ; เตรียม envp (ว่าง - แค่ NULL)
    mov qword [envp_buf + 0], 0
    
    ; fork()
    mov rax, 57
    syscall
    
    test rax, rax
    jz .child
    js .fork_failed
    
    ; ===== PARENT =====
    push rax            ; เก็บ child PID
    
    mov rdi, 1
    mov rsi, msg_fork
    mov rdx, msg_fork_len
    mov rax, 1
    syscall
    
    mov rdi, 1
    mov rsi, msg_waiting
    mov rdx, msg_waiting_len
    mov rax, 1
    syscall
    
    ; wait4(child_pid, &wstatus, 0, NULL)
    pop rdi             ; child PID
    lea rsi, [wstatus]  ; &wstatus
    xor rdx, rdx        ; options = 0
    xor r10, r10        ; rusage = NULL
    mov rax, 61         ; SYS_wait4
    syscall
    
    ; decode wstatus
    mov eax, [wstatus]
    
    ; WIFEXITED: lower 7 bits == 0?
    mov ecx, eax
    and ecx, 0x7F
    test ecx, ecx
    jnz .check_signaled
    
    ; ออกปกติ - แสดง exit code
    mov rdi, 1
    mov rsi, msg_exited
    mov rdx, msg_exited_len
    mov rax, 1
    syscall
    
    ; WEXITSTATUS(wstatus) = (wstatus >> 8) & 0xFF
    mov ecx, [wstatus]
    shr ecx, 8
    and ecx, 0xFF
    movzx rdi, cl
    call print_number
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    jmp .parent_done
    
.check_signaled:
    ; WIFSIGNALED
    cmp ecx, 0x7F
    je .parent_done     ; stopped (ข้ามก่อน)
    
    ; แสดง signal
    mov rdi, 1
    mov rsi, msg_killed
    mov rdx, msg_killed_len
    mov rax, 1
    syscall
    
    ; WTERMSIG(wstatus) = wstatus & 0x7F
    mov eax, [wstatus]
    and eax, 0x7F
    movzx rdi, al
    call print_number
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
.parent_done:
    xor rdi, rdi
    mov rax, 60
    syscall
    
    ; ===== CHILD =====
.child:
    ; exec ls ที่จะ fail ด้วย exit code ไม่ใช่ 0
    mov rdi, prog_ls
    mov rsi, argv_buf
    mov rdx, envp_buf
    mov rax, 59
    syscall
    
    ; ถ้า exec ล้มเหลว ออกด้วย code 127 (convention)
    mov rdi, 127
    mov rax, 60
    syscall
    
.fork_failed:
    mov rdi, 1
    mov rax, 60
    syscall

print_number:
    push rbx
    push rcx
    push rdx
    
    lea rbx, [num_buf + 19]
    mov byte [rbx], 0
    dec rbx
    
    mov rax, rdi
    mov rcx, 10
    
.pn_loop:
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rbx], dl
    dec rbx
    test rax, rax
    jnz .pn_loop
    
    inc rbx
    lea rcx, [num_buf + 19]
    sub rcx, rbx
    
    push rdi
    mov rdi, 1
    mov rsi, rbx
    mov rdx, rcx
    mov rax, 1
    syscall
    pop rdi
    
    pop rdx
    pop rcx
    pop rbx
    ret
```

---

## 5. Zombie Process และ Orphan Process

### Zombie Process

**Zombie** คือ process ที่ตายแล้วแต่ยังมีรายการอยู่ใน process table เพราะ parent ยังไม่ได้ wait()

```
สถานะ Zombie:
- ปรากฎใน ps aux เป็น <defunct> หรือ Z
- ไม่ใช้ CPU
- ไม่ใช้ memory (ยกเว้น process table entry เล็กน้อย)
- ไม่สามารถ kill ได้ด้วย signal
- หายไปเองเมื่อ parent ตาย (init จะ wait() ให้)
```

**วิธีป้องกัน Zombie:**

1. Parent เรียก wait() หรือ waitpid()
2. ตั้ง SIGCHLD handler ที่เรียก waitpid() ใน non-blocking mode
3. Ignore SIGCHLD (kernel จะ auto-reap)

```nasm
; วิธีที่ 1: รอด้วย waitpid() แบบ non-blocking
; ใช้ WNOHANG เพื่อ poll แทนการ block

; WNOHANG = 1
.reap_loop:
    mov rdi, -1         ; รอ child ใดก็ได้
    lea rsi, [wstatus]
    mov rdx, 1          ; WNOHANG
    xor r10, r10        ; rusage = NULL
    mov rax, 61         ; SYS_wait4
    syscall
    
    test rax, rax
    jz .no_child_ready  ; return 0 = ไม่มี child พร้อม
    js .no_children     ; return -ECHILD = ไม่มี child เลย
    ; rax > 0: เก็บ child PID ที่ reap แล้ว
    jmp .reap_loop      ; ลอง reap ต่อ
    
.no_child_ready:
    ; ยังมี child อยู่แต่ยังไม่หยุด
    ret
    
.no_children:
    ; ไม่มี child แล้ว
    ret
```

### Orphan Process

**Orphan** คือ child process ที่ parent ตายไปก่อน:
- Kernel จะ reparent orphan ไปให้ init (PID 1) หรือ subreaper
- init จะคอย wait() เพื่อเก็บ exit status
- Orphan ยังทำงานต่อได้ปกติ ไม่ใช่ปัญหา

```nasm
; ตัวอย่าง Orphan Process
; Parent ออกไปก่อนที่ child จะเสร็จ

section .data
    msg_orphan  db "ฉันกำลังกลายเป็น orphan...", 10, 0
    msg_orphan_len equ $ - msg_orphan

_start:
    mov rax, 57     ; fork
    syscall
    
    test rax, rax
    jz .child
    
    ; Parent: ออกทันทีโดยไม่ wait
    xor rdi, rdi
    mov rax, 60
    syscall
    
.child:
    ; Child ยังทำงานอยู่ แต่ parent ตายแล้ว
    ; PPID จะกลายเป็น 1 (init)
    mov rdi, 1
    mov rsi, msg_orphan
    mov rdx, msg_orphan_len
    mov rax, 1
    syscall
    
    ; Sleep ก่อนออก
    ; nanosleep(&timespec, NULL)
    ; timespec: { tv_sec=2, tv_nsec=0 }
    mov qword [timespec_sec], 2
    mov qword [timespec_nsec], 0
    lea rdi, [timespec_sec]
    xor rsi, rsi
    mov rax, 35     ; SYS_nanosleep
    syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall

section .bss
    timespec_sec    resq 1
    timespec_nsec   resq 1
```

---

## 6. getpid / getppid - ดึง Process ID

```nasm
; getpid() และ getppid() ไม่มี arguments
; return value อยู่ใน RAX

; ไฟล์: pid_info.asm
section .data
    msg_pid     db "My PID  : ", 0
    msg_pid_len equ $ - msg_pid
    msg_ppid    db "My PPID : ", 0
    msg_ppid_len equ $ - msg_ppid
    newline     db 10

section .bss
    num_buf     resb 20

section .text
    global _start

_start:
    ; แสดง PID
    mov rdi, 1
    mov rsi, msg_pid
    mov rdx, msg_pid_len
    mov rax, 1
    syscall
    
    mov rax, 39         ; SYS_getpid
    syscall
    mov rdi, rax
    call print_number
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
    ; แสดง PPID
    mov rdi, 1
    mov rsi, msg_ppid
    mov rdx, msg_ppid_len
    mov rax, 1
    syscall
    
    mov rax, 110        ; SYS_getppid
    syscall
    mov rdi, rax
    call print_number
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall

print_number:
    lea rbx, [num_buf + 19]
    mov byte [rbx], 0
    dec rbx
    mov rax, rdi
    mov rcx, 10
.loop:
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rbx], dl
    dec rbx
    test rax, rax
    jnz .loop
    inc rbx
    lea rcx, [num_buf + 19]
    sub rcx, rbx
    push rdi
    mov rdi, 1
    mov rsi, rbx
    mov rdx, rcx
    mov rax, 1
    syscall
    pop rdi
    ret
```

---

## 7. getuid / geteuid / getgid / getegid

### ความแตกต่างระหว่าง Real ID และ Effective ID

```
Real UID (RUID):    UID ของผู้ที่ login หรือสร้าง process
Effective UID (EUID): UID ที่ใช้ตรวจสอบ permission จริงๆ
Saved UID (SUID):   UID ที่ถูก save ไว้ก่อน setuid()

ปกติ: RUID = EUID = UID ของ user ที่รัน
SUID bit: ถ้าโปรแกรมมี SUID bit, EUID = UID ของเจ้าของไฟล์ (เช่น root)
```

```nasm
; ไฟล์: uid_info.asm
; แสดงข้อมูล UID/GID ทั้งหมด

section .data
    msg_uid     db "UID  (real)      : ", 0
    msg_uid_len equ $ - msg_uid
    msg_euid    db "EUID (effective) : ", 0
    msg_euid_len equ $ - msg_euid
    msg_gid     db "GID  (real)      : ", 0
    msg_gid_len equ $ - msg_gid
    msg_egid    db "EGID (effective) : ", 0
    msg_egid_len equ $ - msg_egid
    newline     db 10

section .bss
    num_buf     resb 20

section .text
    global _start

_start:
    ; getuid()
    mov rdi, 1
    mov rsi, msg_uid
    mov rdx, msg_uid_len
    mov rax, 1
    syscall
    
    mov rax, 102        ; SYS_getuid
    syscall
    mov rdi, rax
    call print_number
    call print_newline
    
    ; geteuid()
    mov rdi, 1
    mov rsi, msg_euid
    mov rdx, msg_euid_len
    mov rax, 1
    syscall
    
    mov rax, 107        ; SYS_geteuid
    syscall
    mov rdi, rax
    call print_number
    call print_newline
    
    ; getgid()
    mov rdi, 1
    mov rsi, msg_gid
    mov rdx, msg_gid_len
    mov rax, 1
    syscall
    
    mov rax, 104        ; SYS_getgid
    syscall
    mov rdi, rax
    call print_number
    call print_newline
    
    ; getegid()
    mov rdi, 1
    mov rsi, msg_egid
    mov rdx, msg_egid_len
    mov rax, 1
    syscall
    
    mov rax, 108        ; SYS_getegid
    syscall
    mov rdi, rax
    call print_number
    call print_newline
    
    xor rdi, rdi
    mov rax, 60
    syscall

print_number:
    push rbx
    push rcx
    push rdx
    lea rbx, [num_buf + 19]
    mov byte [rbx], 0
    dec rbx
    mov rax, rdi
    mov rcx, 10
.loop:
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rbx], dl
    dec rbx
    test rax, rax
    jnz .loop
    inc rbx
    lea rcx, [num_buf + 19]
    sub rcx, rbx
    push rdi
    mov rdi, 1
    mov rsi, rbx
    mov rdx, rcx
    mov rax, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rbx
    ret

print_newline:
    push rax
    push rdi
    push rsi
    push rdx
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    ret
```

---

## 8. setuid / setgid - Privilege Dropping

### Privilege Dropping คืออะไร

หลักการ "least privilege": โปรแกรมที่ต้องการสิทธิ์ root เพียงช่วงสั้นๆ ควร drop สิทธิ์ทันทีหลังทำงานที่ต้องการ root เสร็จ

ตัวอย่าง: web server อาจต้องการ root เพื่อ bind port 80 แต่หลังจากนั้นควร setuid() เป็น www-data

```nasm
; ไฟล์: priv_drop.asm
; ตัวอย่าง privilege dropping
; หมายเหตุ: ต้องรันด้วย root หรือมี CAP_SETUID

section .data
    msg_before  db "กำลัง drop privileges...", 10, 0
    msg_before_len equ $ - msg_before
    msg_after   db "Drop เรียบร้อย!", 10, 0
    msg_after_len equ $ - msg_after
    msg_error   db "setuid() ล้มเหลว (ต้องรัน as root)", 10, 0
    msg_error_len equ $ - msg_error

section .text
    global _start

_start:
    ; แสดงข้อความก่อน
    mov rdi, 1
    mov rsi, msg_before
    mov rdx, msg_before_len
    mov rax, 1
    syscall
    
    ; setgid(1000) - ต้อง drop GID ก่อน UID!
    ; (ถ้า drop UID ก่อน จะไม่มีสิทธิ์ drop GID แล้ว)
    mov rdi, 1000       ; GID = 1000 (user ทั่วไป)
    mov rax, 106        ; SYS_setgid
    syscall
    
    test rax, rax
    js .error
    
    ; setuid(1000) - drop UID สุดท้าย
    mov rdi, 1000       ; UID = 1000
    mov rax, 105        ; SYS_setuid
    syscall
    
    test rax, rax
    js .error
    
    ; ตอนนี้เป็น unprivileged user แล้ว
    mov rdi, 1
    mov rsi, msg_after
    mov rdx, msg_after_len
    mov rax, 1
    syscall
    
    ; ลองกลับเป็น root - จะล้มเหลว!
    mov rdi, 0          ; UID = 0 (root)
    mov rax, 105        ; SYS_setuid
    syscall
    ; รัน rax = -1 (EPERM) - ไม่สามารถกลับได้
    
    xor rdi, rdi
    mov rax, 60
    syscall
    
.error:
    mov rdi, 1
    mov rsi, msg_error
    mov rdx, msg_error_len
    mov rax, 1
    syscall
    
    mov rdi, 1
    mov rax, 60
    syscall
```

---

## 9. chdir / getcwd - Change and Get Working Directory

```nasm
; ไฟล์: dir_ops.asm
; สาธิต chdir() และ getcwd()

section .data
    target_dir  db "/tmp", 0
    msg_cd      db "เปลี่ยน directory ไปที่ /tmp", 10, 0
    msg_cd_len  equ $ - msg_cd
    msg_cwd     db "Current directory: ", 0
    msg_cwd_len equ $ - msg_cwd
    newline     db 10

section .bss
    cwd_buf     resb 4096   ; buffer สำหรับ path

section .text
    global _start

_start:
    ; แสดง cwd ปัจจุบันก่อน
    call show_cwd
    
    ; แสดงข้อความ
    mov rdi, 1
    mov rsi, msg_cd
    mov rdx, msg_cd_len
    mov rax, 1
    syscall
    
    ; chdir("/tmp")
    mov rdi, target_dir     ; path
    mov rax, 80             ; SYS_chdir
    syscall
    
    test rax, rax
    js .chdir_failed
    
    ; แสดง cwd ใหม่
    call show_cwd
    
    xor rdi, rdi
    mov rax, 60
    syscall

.chdir_failed:
    mov rdi, 1
    mov rax, 60
    syscall

; ฟังก์ชัน: แสดง current working directory
show_cwd:
    push rax
    push rdi
    push rsi
    push rdx
    
    ; แสดง prefix
    mov rdi, 1
    mov rsi, msg_cwd
    mov rdx, msg_cwd_len
    mov rax, 1
    syscall
    
    ; getcwd(buf, size)
    ; return: pointer ไปยัง buf (หรือ NULL ถ้า error)
    mov rdi, cwd_buf    ; buffer
    mov rsi, 4096       ; buffer size
    mov rax, 79         ; SYS_getcwd
    syscall
    
    ; RAX = จำนวนตัวอักษรที่เขียนลงใน buf (รวม null terminator)
    ; หรือ negative ถ้า error
    test rax, rax
    js .cwd_error
    
    ; แสดง path (ไม่รวม null terminator, ใส่ newline แทน)
    dec rax             ; ไม่นับ null terminator
    push rax
    mov rdi, 1
    mov rsi, cwd_buf
    mov rdx, rax
    mov rax, 1
    syscall
    pop rax
    
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    
.cwd_error:
    pop rdx
    pop rsi
    pop rdi
    pop rax
    ret
```

---

## 10. chroot - Change Root Directory

### chroot คืออะไร

`chroot()` เปลี่ยน root directory ("/") ของ process เป็น path ที่กำหนด ใช้สำหรับ:
- สร้าง **chroot jail** เพื่อ isolate โปรแกรม
- ทดสอบในสภาพแวดล้อมที่แยกจากกัน
- Recovery mode

**ข้อจำกัด:**
- ต้องมีสิทธิ์ root (CAP_SYS_CHROOT)
- chroot ไม่ใช่ security boundary ที่แน่นหนา (privileged process สามารถหลบหนีได้)
- ต้องใช้ chdir() ร่วมกับ chroot() เสมอ

```nasm
; ไฟล์: chroot_demo.asm
; สาธิต chroot() - ต้องรันด้วย root

section .data
    jail_path   db "/tmp/jail", 0       ; directory สำหรับ jail
    root_path   db "/", 0
    msg_chroot  db "เปลี่ยน root เป็น /tmp/jail", 10, 0
    msg_chroot_len equ $ - msg_chroot
    msg_ok      db "chroot สำเร็จ!", 10, 0
    msg_ok_len  equ $ - msg_ok
    msg_fail    db "chroot ล้มเหลว (ต้องรันด้วย root)", 10, 0
    msg_fail_len equ $ - msg_fail

section .text
    global _start

_start:
    mov rdi, 1
    mov rsi, msg_chroot
    mov rdx, msg_chroot_len
    mov rax, 1
    syscall
    
    ; chroot("/tmp/jail")
    ; หมายเหตุ: /tmp/jail ต้องมีอยู่จริง
    mov rdi, jail_path
    mov rax, 161        ; SYS_chroot
    syscall
    
    test rax, rax
    js .failed
    
    ; ต้อง chdir("/") หลัง chroot เสมอ!
    ; ถ้าไม่ทำ working directory จะยังอยู่นอก jail
    mov rdi, root_path
    mov rax, 80         ; SYS_chdir
    syscall
    
    mov rdi, 1
    mov rsi, msg_ok
    mov rdx, msg_ok_len
    mov rax, 1
    syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall
    
.failed:
    mov rdi, 1
    mov rsi, msg_fail
    mov rdx, msg_fail_len
    mov rax, 1
    syscall
    
    mov rdi, 1
    mov rax, 60
    syscall
```

---

## 11. Process Limits - getrlimit / setrlimit

### Resource Limit Structures

```c
struct rlimit {
    rlim_t rlim_cur;   /* Soft limit - 8 bytes */
    rlim_t rlim_max;   /* Hard limit - 8 bytes */
};
```

**RLIM_INFINITY = 0xFFFFFFFFFFFFFFFF**

### Resource Constants

```nasm
RLIMIT_CPU      equ 0   ; CPU time ใน seconds
RLIMIT_FSIZE    equ 1   ; Max file size ใน bytes
RLIMIT_DATA     equ 2   ; Max data segment size
RLIMIT_STACK    equ 3   ; Max stack size
RLIMIT_CORE     equ 4   ; Max core dump size
RLIMIT_RSS      equ 5   ; Max resident set size
RLIMIT_NPROC    equ 6   ; Max number of processes
RLIMIT_NOFILE   equ 7   ; Max number of open files
RLIMIT_MEMLOCK  equ 8   ; Max locked memory
RLIMIT_AS       equ 9   ; Max address space size
RLIMIT_LOCKS    equ 10  ; Max file locks
RLIMIT_SIGPENDING equ 11 ; Max pending signals
RLIMIT_MSGQUEUE equ 12  ; Max POSIX message queue
RLIMIT_NICE     equ 13  ; Max nice priority
RLIMIT_RTPRIO   equ 14  ; Max real-time priority
RLIMIT_RTTIME   equ 15  ; Max real-time CPU time (microsecs)
```

```nasm
; ไฟล์: rlimit_demo.asm
; อ่านและแสดง resource limits

section .data
    msg_nofile  db "RLIMIT_NOFILE (max open files):", 10
                db "  soft: ", 0
    msg_nofile_len equ $ - msg_nofile
    msg_hard    db "  hard: ", 0
    msg_hard_len equ $ - msg_hard
    newline     db 10
    infinity    db "unlimited", 10, 0
    infinity_len equ $ - infinity

section .bss
    rlim_struct resq 2      ; rlimit: cur (8 bytes) + max (8 bytes)
    num_buf     resb 20

section .text
    global _start

_start:
    ; getrlimit(RLIMIT_NOFILE, &rlim)
    mov rdi, 7              ; RLIMIT_NOFILE
    lea rsi, [rlim_struct]  ; &rlimit struct
    mov rax, 97             ; SYS_getrlimit
    syscall
    
    test rax, rax
    js .error
    
    ; แสดง header
    mov rdi, 1
    mov rsi, msg_nofile
    mov rdx, msg_nofile_len
    mov rax, 1
    syscall
    
    ; แสดง soft limit
    mov rax, [rlim_struct]          ; rlim_cur
    mov rcx, 0xFFFFFFFFFFFFFFFF     ; RLIM_INFINITY
    cmp rax, rcx
    je .soft_unlimited
    
    mov rdi, rax
    call print_number
    call print_newline
    jmp .show_hard
    
.soft_unlimited:
    mov rdi, 1
    mov rsi, infinity
    mov rdx, infinity_len
    mov rax, 1
    syscall
    
.show_hard:
    mov rdi, 1
    mov rsi, msg_hard
    mov rdx, msg_hard_len
    mov rax, 1
    syscall
    
    ; แสดง hard limit
    mov rax, [rlim_struct + 8]      ; rlim_max
    mov rcx, 0xFFFFFFFFFFFFFFFF
    cmp rax, rcx
    je .hard_unlimited
    
    mov rdi, rax
    call print_number
    call print_newline
    jmp .done
    
.hard_unlimited:
    mov rdi, 1
    mov rsi, infinity
    mov rdx, infinity_len
    mov rax, 1
    syscall
    
.done:
    ; ตัวอย่าง setrlimit - ลด RLIMIT_NOFILE เป็น 64
    ; (process นี้จะเปิดไฟล์ได้สูงสุด 64 ไฟล์)
    mov qword [rlim_struct], 64     ; soft limit = 64
    mov qword [rlim_struct + 8], 64 ; hard limit = 64
    
    mov rdi, 7              ; RLIMIT_NOFILE
    lea rsi, [rlim_struct]
    mov rax, 160            ; SYS_setrlimit
    syscall
    ; (ตรวจสอบ error ด้วยถ้าต้องการ)
    
    xor rdi, rdi
    mov rax, 60
    syscall
    
.error:
    mov rdi, 1
    mov rax, 60
    syscall

print_number:
    push rbx
    push rcx
    push rdx
    lea rbx, [num_buf + 19]
    mov byte [rbx], 0
    dec rbx
    mov rax, rdi
    mov rcx, 10
.loop:
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rbx], dl
    dec rbx
    test rax, rax
    jnz .loop
    inc rbx
    lea rcx, [num_buf + 19]
    sub rcx, rbx
    push rdi
    mov rdi, 1
    mov rsi, rbx
    mov rdx, rcx
    mov rax, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rbx
    ret

print_newline:
    push rax
    push rdi
    push rsi
    push rdx
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    mov rax, 1
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    ret
```

---

## 12. Process Memory - อ่าน /proc/self/maps และ /proc/self/status

### /proc/self/maps

ไฟล์นี้แสดง memory mappings ของ process ปัจจุบัน รูปแบบแต่ละบรรทัด:

```
ที่อยู่เริ่ม-ที่อยู่สิ้นสุด  permissions  offset  dev  inode  path
55a1b2c3d000-55a1b2c3e000  r-xp  00000000  08:01  12345  /bin/myprogram
```

**Permissions:**
- `r` = read, `w` = write, `x` = execute, `p` = private, `s` = shared

### /proc/self/status

แสดงสถานะต่างๆ ของ process เช่น:
- `VmRSS`: Resident Set Size (RAM ที่ใช้จริง)
- `VmVSZ`: Virtual Memory Size
- `Threads`: จำนวน threads

```nasm
; ไฟล์: proc_maps.asm
; อ่านและแสดง /proc/self/maps

section .data
    maps_path   db "/proc/self/maps", 0
    status_path db "/proc/self/status", 0
    msg_maps    db "=== /proc/self/maps ===", 10, 0
    msg_maps_len equ $ - msg_maps
    msg_status  db "=== /proc/self/status ===", 10, 0
    msg_status_len equ $ - msg_status
    err_msg     db "ไม่สามารถเปิดไฟล์ได้", 10, 0
    err_msg_len equ $ - err_msg

section .bss
    read_buf    resb 4096   ; buffer สำหรับอ่านข้อมูล

section .text
    global _start

_start:
    ; แสดง header สำหรับ maps
    mov rdi, 1
    mov rsi, msg_maps
    mov rdx, msg_maps_len
    mov rax, 1
    syscall
    
    ; open("/proc/self/maps", O_RDONLY)
    mov rdi, maps_path
    xor rsi, rsi            ; O_RDONLY = 0
    xor rdx, rdx            ; mode (ไม่จำเป็นสำหรับ O_RDONLY)
    mov rax, 2              ; SYS_open
    syscall
    
    test rax, rax
    js .open_failed
    
    mov rbx, rax            ; เก็บ fd
    
    ; อ่านและแสดงทีละ block
.read_maps_loop:
    mov rdi, rbx            ; fd
    mov rsi, read_buf       ; buffer
    mov rdx, 4096           ; max bytes
    mov rax, 0              ; SYS_read
    syscall
    
    test rax, rax
    jle .close_maps         ; EOF หรือ error
    
    push rax
    ; write to stdout
    mov rdi, 1
    mov rsi, read_buf
    mov rdx, rax
    mov rax, 1
    syscall
    pop rax
    
    cmp rax, 4096
    je .read_maps_loop      ; อ่านต่อถ้าเต็ม buffer
    
.close_maps:
    ; close(fd)
    mov rdi, rbx
    mov rax, 3              ; SYS_close
    syscall
    
    ; แสดง header สำหรับ status
    mov rdi, 1
    mov rsi, msg_status
    mov rdx, msg_status_len
    mov rax, 1
    syscall
    
    ; open("/proc/self/status", O_RDONLY)
    mov rdi, status_path
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 2
    syscall
    
    test rax, rax
    js .open_failed
    
    mov rbx, rax
    
.read_status_loop:
    mov rdi, rbx
    mov rsi, read_buf
    mov rdx, 4096
    mov rax, 0
    syscall
    
    test rax, rax
    jle .close_status
    
    push rax
    mov rdi, 1
    mov rsi, read_buf
    mov rdx, rax
    mov rax, 1
    syscall
    pop rax
    
    cmp rax, 4096
    je .read_status_loop
    
.close_status:
    mov rdi, rbx
    mov rax, 3
    syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall
    
.open_failed:
    mov rdi, 1
    mov rsi, err_msg
    mov rdx, err_msg_len
    mov rax, 1
    syscall
    
    mov rdi, 1
    mov rax, 60
    syscall
```

---

## 13. Signals - kill syscall, raise

### Signal Numbers ที่สำคัญ

```nasm
SIGHUP      equ 1   ; Hangup - terminal ปิด
SIGINT      equ 2   ; Interrupt - Ctrl+C
SIGQUIT     equ 3   ; Quit - Ctrl+\ (core dump)
SIGILL      equ 4   ; Illegal instruction
SIGTRAP     equ 5   ; Trace/breakpoint trap
SIGABRT     equ 6   ; Abort
SIGBUS      equ 7   ; Bus error
SIGFPE      equ 8   ; Floating point exception
SIGKILL     equ 9   ; Kill (ไม่สามารถ catch/ignore ได้)
SIGUSR1     equ 10  ; User-defined signal 1
SIGSEGV     equ 11  ; Segmentation fault
SIGUSR2     equ 12  ; User-defined signal 2
SIGPIPE     equ 13  ; Broken pipe
SIGALRM     equ 14  ; Timer signal (alarm())
SIGTERM     equ 15  ; Termination - graceful shutdown
SIGCHLD     equ 17  ; Child status changed
SIGCONT     equ 18  ; Continue if stopped
SIGSTOP     equ 19  ; Stop (ไม่สามารถ catch/ignore ได้)
SIGTSTP     equ 20  ; Terminal stop - Ctrl+Z
SIGTTIN     equ 21  ; Background read from terminal
SIGTTOU     equ 22  ; Background write to terminal
```

### kill() syscall

```c
int kill(pid_t pid, int sig);
```

**pid values พิเศษ:**
- `pid > 0`: ส่ง signal ไปยัง process ที่มี PID ตรงกัน
- `pid = 0`: ส่งไปทุก process ในกลุ่มเดียวกัน
- `pid = -1`: ส่งไปทุก process ที่มีสิทธิ์ส่ง
- `pid < -1`: ส่งไปทุก process ใน process group |pid|

```nasm
; ไฟล์: signals_demo.asm
; สาธิต kill() และ raise()

section .data
    msg_kill    db "ส่ง SIGTERM ไปยัง process...", 10, 0
    msg_kill_len equ $ - msg_kill
    msg_raise   db "ส่ง signal ให้ตัวเอง (SIGUSR1)...", 10, 0
    msg_raise_len equ $ - msg_raise
    msg_alive   db "ยังมีชีวิตอยู่หลัง SIGUSR1", 10, 0
    msg_alive_len equ $ - msg_alive
    msg_handled db "SIGUSR1 ถูก handle แล้ว!", 10, 0
    msg_handled_len equ $ - msg_handled
    newline     db 10

section .bss
    sigaction_struct resb 152   ; struct sigaction (ขนาด 152 bytes บน x86_64)

section .text
    global _start

_start:
    ; ตั้ง SIGUSR1 handler ก่อน
    ; struct sigaction:
    ;   sa_handler  (8 bytes) - pointer ไปยัง handler function
    ;   sa_flags    (8 bytes)
    ;   sa_restorer (8 bytes) - pointer ไปยัง restorer (ต้องมี)
    ;   sa_mask     (128 bytes) - signal mask
    
    ; เคลียร์ struct
    lea rdi, [sigaction_struct]
    mov rcx, 152/8
    xor rax, rax
    rep stosq
    
    ; ตั้งค่า handler
    lea rax, [sigusr1_handler]
    mov [sigaction_struct], rax         ; sa_handler
    
    ; SA_RESTORER = 0x04000000
    ; ต้องตั้ง sa_restorer ด้วย
    mov qword [sigaction_struct + 8], 0x04000000  ; sa_flags = SA_RESTORER
    lea rax, [signal_restorer]
    mov [sigaction_struct + 16], rax    ; sa_restorer
    
    ; rt_sigaction(SIGUSR1, &new_action, NULL, 8)
    mov rdi, 10                 ; SIGUSR1
    lea rsi, [sigaction_struct] ; &new action
    xor rdx, rdx                ; NULL (ไม่ต้องการ old action)
    mov r10, 8                  ; sigset_t size
    mov rax, 13                 ; SYS_rt_sigaction
    syscall
    
    ; แสดงข้อความก่อน raise
    mov rdi, 1
    mov rsi, msg_raise
    mov rdx, msg_raise_len
    mov rax, 1
    syscall
    
    ; raise(SIGUSR1) = kill(getpid(), SIGUSR1)
    ; หรือใช้ kill(0, SIGUSR1) ก็ได้
    mov rax, 39         ; SYS_getpid
    syscall
    
    mov rdi, rax        ; pid = getpid()
    mov rsi, 10         ; SIGUSR1
    mov rax, 62         ; SYS_kill
    syscall
    
    ; แสดงข้อความหลัง signal ถูก handle
    mov rdi, 1
    mov rsi, msg_alive
    mov rdx, msg_alive_len
    mov rax, 1
    syscall
    
    ; ส่ง SIGTERM ไปยังตัวเอง (จะทำให้ตาย)
    ; mov rax, 39
    ; syscall
    ; mov rdi, rax
    ; mov rsi, 15     ; SIGTERM
    ; mov rax, 62
    ; syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall

; Signal handler สำหรับ SIGUSR1
sigusr1_handler:
    ; ใน signal handler ใช้ syscall ที่ async-signal-safe เท่านั้น
    push rax
    push rdi
    push rsi
    push rdx
    
    mov rdi, 1
    mov rsi, msg_handled
    mov rdx, msg_handled_len
    mov rax, 1
    syscall
    
    pop rdx
    pop rsi
    pop rdi
    pop rax
    ret

; Signal restorer (จำเป็นสำหรับ SA_RESTORER)
signal_restorer:
    mov rax, 15         ; SYS_rt_sigreturn
    syscall
```

---

## 14. โปรแกรมประยุกต์: Simple Shell

```nasm
; ไฟล์: simple_shell.asm
; Simple shell: รับ command จาก user, fork+exec+wait
; รองรับ: commandline แบบง่าย (ไม่มี pipe, redirect, ฯลฯ)
; คอมไพล์: nasm -f elf64 simple_shell.asm -o simple_shell.o && ld simple_shell.o -o simple_shell

section .data
    prompt          db "ash> ", 0
    prompt_len      equ $ - prompt
    newline         db 10, 0
    
    ; Messages
    msg_exit        db "ลาก่อน!", 10, 0
    msg_exit_len    equ $ - msg_exit
    
    msg_exec_fail   db "ไม่สามารถรันคำสั่งได้", 10, 0
    msg_exec_fail_len equ $ - msg_exec_fail
    
    msg_fork_fail   db "fork() ล้มเหลว", 10, 0
    msg_fork_fail_len equ $ - msg_fork_fail
    
    ; exit command string
    str_exit        db "exit", 0
    
    ; Shell environment
    env_path        db "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 0
    env_home        db "HOME=/root", 0
    env_term        db "TERM=xterm-256color", 0
    
section .bss
    ; Input buffer สำหรับรับ command
    input_buf       resb 1024
    
    ; argv array (max 64 arguments)
    argv_ptrs       resq 65
    
    ; envp array
    envp_ptrs       resq 4
    
    ; Path สำหรับ exec
    exec_path       resb 256
    
    ; Wait status
    wait_status     resd 1
    
    ; Parsed token pointers
    token_count     resq 1
    
section .text
    global _start

_start:
    ; เตรียม envp
    mov qword [envp_ptrs + 0], env_path
    mov qword [envp_ptrs + 8], env_home
    mov qword [envp_ptrs + 16], env_term
    mov qword [envp_ptrs + 24], 0       ; NULL terminator
    
.shell_loop:
    ; แสดง prompt
    mov rdi, 1
    mov rsi, prompt
    mov rdx, prompt_len
    mov rax, 1
    syscall
    
    ; อ่าน input จาก stdin
    mov rdi, 0              ; stdin
    mov rsi, input_buf
    mov rdx, 1023           ; max bytes
    mov rax, 0              ; SYS_read
    syscall
    
    ; ตรวจสอบ EOF (Ctrl+D)
    test rax, rax
    jle .shell_done
    
    ; ลบ newline ที่ท้าย
    lea rbx, [input_buf]
    add rbx, rax
    dec rbx                 ; ชี้ไปตัวสุดท้าย
    cmp byte [rbx], 10      ; newline?
    jne .no_newline
    mov byte [rbx], 0       ; แทนที่ด้วย null
.no_newline:
    
    ; ถ้า input ว่าง ข้ามไป
    cmp byte [input_buf], 0
    je .shell_loop
    
    ; ตรวจสอบคำสั่ง "exit"
    lea rdi, [input_buf]
    lea rsi, [str_exit]
    call str_equal
    test rax, rax
    jnz .shell_done
    
    ; Parse command เป็น arguments (tokenize by spaces)
    lea rdi, [input_buf]
    call tokenize
    ; token_count = จำนวน tokens
    ; argv_ptrs = array ของ pointers
    
    ; ตรวจสอบว่ามี argument อย่างน้อย 1 ตัว
    mov rax, [token_count]
    test rax, rax
    jz .shell_loop
    
    ; สร้าง exec path จาก argv[0]
    ; ถ้า argv[0] ไม่เริ่มด้วย '/' ให้ใส่ /usr/bin/ ข้างหน้า
    mov rax, [argv_ptrs]    ; argv[0]
    cmp byte [rax], '/'
    je .absolute_path
    
    ; สร้าง path: "/usr/bin/" + command
    lea rdi, [exec_path]
    lea rsi, [usr_bin_prefix]
    call str_copy
    
    ; append command name
    lea rdi, [exec_path]
    call str_end            ; หา end of string
    ; rdi ชี้ไปที่ null terminator
    mov rsi, [argv_ptrs]    ; argv[0]
    call str_copy
    jmp .do_exec
    
.absolute_path:
    lea rdi, [exec_path]
    mov rsi, [argv_ptrs]
    call str_copy
    
.do_exec:
    ; fork()
    mov rax, 57
    syscall
    
    test rax, rax
    jz .child_exec
    js .fork_err
    
    ; ===== PARENT =====
    ; รอ child
    mov rdi, rax            ; child PID
    lea rsi, [wait_status]
    xor rdx, rdx            ; options = 0
    xor r10, r10            ; rusage = NULL
    mov rax, 61
    syscall
    
    jmp .shell_loop
    
.child_exec:
    ; ===== CHILD =====
    ; execve(exec_path, argv_ptrs, envp_ptrs)
    lea rdi, [exec_path]
    lea rsi, [argv_ptrs]
    lea rdx, [envp_ptrs]
    mov rax, 59
    syscall
    
    ; exec ล้มเหลว
    mov rdi, 2
    mov rsi, msg_exec_fail
    mov rdx, msg_exec_fail_len
    mov rax, 1
    syscall
    
    mov rdi, 127
    mov rax, 60
    syscall
    
.fork_err:
    mov rdi, 2
    mov rsi, msg_fork_fail
    mov rdx, msg_fork_fail_len
    mov rax, 1
    syscall
    jmp .shell_loop
    
.shell_done:
    mov rdi, 1
    mov rsi, msg_exit
    mov rdx, msg_exit_len
    mov rax, 1
    syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall

; ===== Helper Functions =====

; tokenize(rdi = string)
; แยก string ด้วย space, เก็บ pointer ใน argv_ptrs
; result: token_count = จำนวน tokens
tokenize:
    push rbx
    push rcx
    push rdx
    push rsi
    
    mov rbx, rdi            ; current position
    xor rcx, rcx            ; token count
    mov rdx, 0              ; in_token = false
    
.tok_loop:
    mov al, [rbx]
    
    test al, al             ; null terminator?
    jz .tok_end_check
    
    cmp al, ' '             ; space?
    je .tok_space
    cmp al, 9               ; tab?
    je .tok_space
    
    ; ตัวอักษรปกติ
    test rdx, rdx
    jnz .tok_continue       ; ถ้ากำลัง in token อยู่แล้ว ข้ามไป
    
    ; เริ่ม token ใหม่
    mov [argv_ptrs + rcx*8], rbx   ; เก็บ pointer
    inc rcx
    mov rdx, 1              ; in_token = true
    jmp .tok_continue
    
.tok_space:
    test rdx, rdx
    jz .tok_continue        ; ไม่ได้ใน token
    
    ; สิ้นสุด token
    mov byte [rbx], 0       ; null-terminate token
    mov rdx, 0              ; in_token = false
    jmp .tok_continue
    
.tok_continue:
    inc rbx
    jmp .tok_loop
    
.tok_end_check:
    ; NULL terminate argv array
    mov qword [argv_ptrs + rcx*8], 0
    mov [token_count], rcx
    
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

; str_equal(rdi, rsi) -> rax (0=not equal, 1=equal)
str_equal:
    push rbx
    push rcx
.se_loop:
    mov al, [rdi]
    mov bl, [rsi]
    cmp al, bl
    jne .se_not_equal
    test al, al
    jz .se_equal
    inc rdi
    inc rsi
    jmp .se_loop
.se_equal:
    mov rax, 1
    pop rcx
    pop rbx
    ret
.se_not_equal:
    xor rax, rax
    pop rcx
    pop rbx
    ret

; str_copy(rdi=dest, rsi=src)
str_copy:
    push rax
.sc_loop:
    mov al, [rsi]
    mov [rdi], al
    test al, al
    jz .sc_done
    inc rdi
    inc rsi
    jmp .sc_loop
.sc_done:
    pop rax
    ret

; str_end(rdi=string) -> rdi = pointer to null terminator
str_end:
    cmp byte [rdi], 0
    je .done
    inc rdi
    jmp str_end
.done:
    ret

section .data
    usr_bin_prefix  db "/usr/bin/", 0
```

---

## 15. โปรแกรมประยุกต์: Daemon Process (Double Fork)

### Daemon คืออะไร

Daemon คือ process ที่ทำงานอยู่ใน background โดยไม่มี controlling terminal:
- ไม่ถูก kill เมื่อ user logout
- ทำงานเป็น background service
- ตัวอย่าง: sshd, nginx, cron, syslogd

### Double Fork Technique

```
ขั้นตอน:
1. fork() ครั้งแรก -> parent exit ทันที (orphan child)
2. setsid() -> สร้าง session ใหม่ (ตัดความสัมพันธ์กับ terminal)
3. fork() ครั้งสอง -> parent exit ทันที
   (ป้องกัน daemon จากการ acquire controlling terminal ได้อีก)
4. chdir("/") -> ไม่ block unmount
5. umask(0) -> ควบคุม file permissions เอง
6. ปิด stdin/stdout/stderr (fd 0,1,2) และเปิดใหม่ไปยัง /dev/null
7. เริ่มทำงานจริง
```

```nasm
; ไฟล์: daemon_demo.asm
; Daemon process ด้วย double fork technique
; จะ log ไปที่ /tmp/daemon.log ทุก 5 วินาที

section .data
    log_path        db "/tmp/daemon.log", 0
    root_path       db "/", 0
    dev_null        db "/dev/null", 0
    
    ; Log messages
    log_start       db "Daemon started", 10, 0
    log_start_len   equ $ - log_start
    log_tick        db "Daemon tick", 10, 0
    log_tick_len    equ $ - log_tick
    
    err_fork        db "fork failed", 10, 0
    err_fork_len    equ $ - err_fork
    err_setsid      db "setsid failed", 10, 0
    err_setsid_len  equ $ - err_setsid

section .bss
    log_fd          resd 1
    null_fd         resd 1
    wait_status     resd 1
    timespec        resq 2      ; tv_sec + tv_nsec

section .text
    global _start

_start:
    ; ===== STEP 1: First Fork =====
    mov rax, 57             ; fork()
    syscall
    
    test rax, rax
    js .fork1_failed
    jnz .parent1_exit       ; parent ออกทันที
    
    ; ===== ใน First Child =====
    
    ; STEP 2: setsid() - สร้าง session ใหม่
    mov rax, 112            ; SYS_setsid
    syscall
    test rax, rax
    js .setsid_failed
    
    ; STEP 3: Second Fork
    mov rax, 57
    syscall
    
    test rax, rax
    js .fork2_failed
    jnz .parent2_exit       ; first child ออก
    
    ; ===== ใน Daemon Process (Second Child) =====
    
    ; STEP 4: chdir("/")
    mov rdi, root_path
    mov rax, 80
    syscall
    
    ; STEP 5: umask(0) - ให้ daemon ควบคุม permissions เอง
    mov rdi, 0
    mov rax, 95             ; SYS_umask
    syscall
    
    ; STEP 6: ปิด stdin/stdout/stderr
    ; close(0), close(1), close(2)
    xor rdi, rdi
    mov rax, 3
    syscall
    
    mov rdi, 1
    mov rax, 3
    syscall
    
    mov rdi, 2
    mov rax, 3
    syscall
    
    ; เปิด /dev/null และ dup2 ไปยัง 0,1,2
    ; open("/dev/null", O_RDWR)
    mov rdi, dev_null
    mov rsi, 2              ; O_RDWR
    xor rdx, rdx
    mov rax, 2
    syscall
    
    test rax, rax
    js .daemon_work         ; ถ้าเปิดไม่ได้ก็ข้าม
    
    mov [null_fd], eax      ; เก็บ fd
    
    ; dup2(null_fd, 0) - stdin -> /dev/null
    movsx rdi, dword [null_fd]
    mov rsi, 0
    mov rax, 33             ; SYS_dup2
    syscall
    
    ; dup2(null_fd, 1) - stdout -> /dev/null
    movsx rdi, dword [null_fd]
    mov rsi, 1
    mov rax, 33
    syscall
    
    ; dup2(null_fd, 2) - stderr -> /dev/null
    movsx rdi, dword [null_fd]
    mov rsi, 2
    mov rax, 33
    syscall
    
    ; ปิด null_fd ต้นฉบับ (ถ้าไม่ใช่ 0,1,2)
    movsx rdi, dword [null_fd]
    cmp rdi, 2
    jle .daemon_work
    mov rax, 3
    syscall
    
    ; STEP 7: เปิด log file
.daemon_work:
    ; open("/tmp/daemon.log", O_WRONLY|O_CREAT|O_APPEND, 0644)
    mov rdi, log_path
    mov rsi, 0x401          ; O_WRONLY|O_CREAT = 1|0x40 = 0x41, append = 0x400 -> 0x441
    mov rdx, 0644o
    mov rax, 2
    syscall
    
    test rax, rax
    js .daemon_exit
    
    mov [log_fd], eax
    
    ; เขียน start message
    movsx rdi, dword [log_fd]
    mov rsi, log_start
    mov rdx, log_start_len
    mov rax, 1
    syscall
    
    ; Main loop: tick ทุก 5 วินาที
    mov rcx, 10             ; จำนวน ticks (สำหรับ demo)
    
.daemon_loop:
    test rcx, rcx
    jz .daemon_exit
    dec rcx
    
    ; เขียน tick
    movsx rdi, dword [log_fd]
    mov rsi, log_tick
    mov rdx, log_tick_len
    mov rax, 1
    syscall
    
    ; sleep 5 วินาที
    mov qword [timespec], 5   ; tv_sec = 5
    mov qword [timespec + 8], 0  ; tv_nsec = 0
    lea rdi, [timespec]
    xor rsi, rsi
    mov rax, 35             ; SYS_nanosleep
    syscall
    
    jmp .daemon_loop
    
.daemon_exit:
    ; ปิด log file
    movsx rdi, dword [log_fd]
    mov rax, 3
    syscall
    
    xor rdi, rdi
    mov rax, 60
    syscall
    
.parent1_exit:
    ; First parent ออก
    xor rdi, rdi
    mov rax, 60
    syscall
    
.parent2_exit:
    ; First child (second parent) ออก
    xor rdi, rdi
    mov rax, 60
    syscall
    
.fork1_failed:
    mov rdi, 2
    mov rsi, err_fork
    mov rdx, err_fork_len
    mov rax, 1
    syscall
    mov rdi, 1
    mov rax, 60
    syscall
    
.fork2_failed:
    mov rdi, 1
    mov rax, 60
    syscall
    
.setsid_failed:
    mov rdi, 2
    mov rsi, err_setsid
    mov rdx, err_setsid_len
    mov rax, 1
    syscall
    mov rdi, 1
    mov rax, 60
    syscall
```

---

## 16. โปรแกรมประยุกต์: Supervisor Process (Restart on Crash)

```nasm
; ไฟล์: supervisor.asm
; Supervisor: ตรวจสอบ child process และ restart อัตโนมัติเมื่อ crash
; นี่คือ simplified version ของ supervisord/systemd

section .data
    ; โปรแกรมที่จะ supervise
    worker_prog     db "/tmp/worker", 0
    worker_arg0     db "worker", 0
    
    ; Log messages
    msg_starting    db "Supervisor: เริ่ม worker...", 10, 0
    msg_starting_len equ $ - msg_starting
    
    msg_crashed     db "Supervisor: worker crash! กำลัง restart...", 10, 0
    msg_crashed_len equ $ - msg_crashed
    
    msg_exited      db "Supervisor: worker ออกปกติ, ไม่ restart", 10, 0
    msg_exited_len  equ $ - msg_exited
    
    msg_max_restart db "Supervisor: restart ครบ limit แล้ว, หยุด", 10, 0
    msg_max_restart_len equ $ - msg_max_restart
    
    msg_signal      db "Supervisor: worker ถูก signal kill", 10, 0
    msg_signal_len  equ $ - msg_signal
    
    newline         db 10

section .bss
    worker_pid      resd 1
    wait_status     resd 1
    restart_count   resq 1
    argv_arr        resq 3
    envp_arr        resq 2
    env_path        resb 64
    timespec        resq 2

section .data
    max_restarts    equ 5       ; restart ได้สูงสุด 5 ครั้ง
    restart_delay   equ 2       ; รอ 2 วินาทีก่อน restart
    env_path_str    db "PATH=/usr/bin:/bin", 0

section .text
    global _start

_start:
    ; เตรียม argv และ envp
    mov qword [argv_arr], worker_arg0
    mov qword [argv_arr + 8], 0
    
    mov qword [envp_arr], env_path_str
    mov qword [envp_arr + 8], 0
    
    mov qword [restart_count], 0

.supervise_loop:
    ; ตรวจสอบ restart count
    mov rax, [restart_count]
    cmp rax, max_restarts
    jge .max_restart_reached
    
    ; แสดงข้อความ starting
    mov rdi, 1
    mov rsi, msg_starting
    mov rdx, msg_starting_len
    mov rax, 1
    syscall
    
    ; fork() เพื่อสร้าง worker
    mov rax, 57
    syscall
    
    test rax, rax
    jz .worker_child
    js .fork_error
    
    ; ===== SUPERVISOR (PARENT) =====
    mov [worker_pid], eax       ; เก็บ worker PID
    
    ; รอ worker
.wait_worker:
    mov rdi, -1                 ; รอ child ใดก็ได้
    lea rsi, [wait_status]
    xor rdx, rdx                ; options = 0 (block)
    xor r10, r10
    mov rax, 61                 ; SYS_wait4
    syscall
    
    ; ตรวจสอบ wait status
    mov eax, [wait_status]
    
    ; WIFEXITED: lower 7 bits == 0?
    mov ecx, eax
    and ecx, 0x7F
    test ecx, ecx
    jnz .check_signaled
    
    ; Worker ออกปกติ
    ; WEXITSTATUS
    mov ecx, eax
    shr ecx, 8
    and ecx, 0xFF
    
    ; ถ้า exit code = 0 -> ไม่ restart
    test ecx, ecx
    jz .normal_exit
    
    ; exit code != 0 -> ถือว่า crash, restart
    mov rdi, 1
    mov rsi, msg_crashed
    mov rdx, msg_crashed_len
    mov rax, 1
    syscall
    jmp .do_restart
    
.check_signaled:
    cmp ecx, 0x7F
    je .wait_worker     ; stopped, รอต่อ
    
    ; Worker ถูก signal kill -> restart
    mov rdi, 1
    mov rsi, msg_signal
    mov rdx, msg_signal_len
    mov rax, 1
    syscall
    
.do_restart:
    ; รอก่อน restart
    mov qword [timespec], restart_delay
    mov qword [timespec + 8], 0
    lea rdi, [timespec]
    xor rsi, rsi
    mov rax, 35
    syscall
    
    inc qword [restart_count]
    jmp .supervise_loop
    
.normal_exit:
    mov rdi, 1
    mov rsi, msg_exited
    mov rdx, msg_exited_len
    mov rax, 1
    syscall
    jmp .supervisor_done
    
.max_restart_reached:
    mov rdi, 1
    mov rsi, msg_max_restart
    mov rdx, msg_max_restart_len
    mov rax, 1
    syscall
    jmp .supervisor_done
    
    ; ===== WORKER CHILD =====
.worker_child:
    ; exec worker program
    mov rdi, worker_prog
    lea rsi, [argv_arr]
    lea rdx, [envp_arr]
    mov rax, 59
    syscall
    
    ; exec ล้มเหลว -> exit ด้วย error code
    mov rdi, 127
    mov rax, 60
    syscall
    
.fork_error:
    ; fork ล้มเหลว
    mov rdi, 1
    mov rax, 60
    syscall
    
.supervisor_done:
    xor rdi, rdi
    mov rax, 60
    syscall
```

---

## 17. การจัดการ Process Groups และ Sessions

### Concepts

```
Session:
  - กลุ่มของ process groups ที่เกี่ยวข้องกัน
  - มี Session Leader (process ที่เรียก setsid())
  - อาจมี controlling terminal

Process Group:
  - กลุ่มของ process ที่เกี่ยวข้องกัน
  - มี Process Group Leader
  - ใช้สำหรับส่ง signal ไปทั้งกลุ่ม

Job Control:
  - Foreground process group: รับ input จาก terminal
  - Background process groups: ทำงานอยู่เบื้องหลัง
```

```nasm
; setsid() - สร้าง session ใหม่
; เรียกได้เฉพาะถ้า caller ไม่ใช่ process group leader
mov rax, 112    ; SYS_setsid
syscall
; return: session ID ใหม่ หรือ -1 ถ้า error

; setpgid(pid, pgid) - เปลี่ยน process group
; setpgid(0, 0) = สร้าง process group ใหม่โดยใช้ PID เป็น PGID
mov rdi, 0      ; pid = 0 (current process)
mov rsi, 0      ; pgid = 0 (ใช้ PID เป็น PGID)
mov rax, 109    ; SYS_setpgid
syscall

; getpgid(0) - ดึง process group ID
mov rdi, 0
mov rax, 121    ; SYS_getpgid
syscall
```

---

## 18. สรุปโครงสร้าง Process ใน Linux

```
Process State Machine:
                  
  fork()           schedule
NEW -------> READY -------> RUNNING
                ^               |
                |  schedule()   | wait/I/O/signal
                |               v
                +------- WAITING/BLOCKED
                               |
              parent wait()    | I/O complete / signal
                |              v
              ZOMBIE <------ READY
                |
                | parent wait() collects exit status
                v
              REMOVED (process table entry freed)
```

### Process Hierarchy ตัวอย่าง

```
init (PID 1)
├── systemd-journald (PID 234)
├── sshd (PID 456)
│   └── sshd worker (PID 789) <- สำหรับ connection ของเรา
│       └── bash (PID 1000) <- shell ของเรา
│           └── our_program (PID 1234)
│               ├── child1 (PID 1235)
│               └── child2 (PID 1236)
└── cron (PID 567)
    └── [cronjob] (เกิดและตายบ่อย)
```

---

## 19. Best Practices และ Common Pitfalls

### 1. Always check return values

```nasm
; ไม่ดี:
mov rax, 57     ; fork()
syscall
; ใช้ rax โดยไม่ตรวจสอบ error

; ดีกว่า:
mov rax, 57
syscall
test rax, rax
js .fork_failed     ; rax < 0 -> error
jz .child_code      ; rax = 0 -> child
; rax > 0 -> parent, rax = child PID
```

### 2. จัดการ Zombie ด้วย SIGCHLD handler

```nasm
; ตั้ง SIGCHLD handler ที่เรียก waitpid(WNOHANG)
; เพื่อ reap zombie โดยอัตโนมัติ
; หรือ ignore SIGCHLD เพื่อให้ kernel auto-reap:

; sigaction(SIGCHLD, {SIG_IGN}, NULL, 8)
; SIG_IGN = 1
```

### 3. exec หลัง fork: close unused file descriptors

```nasm
; หลัง fork() ใน child:
; ปิด fd ที่ไม่ต้องการก่อน exec
; เพราะ exec จะ inherit fd ทั้งหมด (ยกเว้น FD_CLOEXEC)
; โดยเฉพาะ pipe, socket ที่ parent เปิดไว้
```

### 4. Signal-safe functions ใน handler

```nasm
; ใน signal handler ใช้เฉพาะ async-signal-safe functions:
; write(), read(), fork(), execve(), _exit()
; ห้ามใช้: malloc(), printf(), ฟังก์ชัน C library ทั่วไป
```

### 5. Double fork สำหรับ daemon

```nasm
; เสมอใช้ double fork เพื่อป้องกัน daemon
; จากการ acquire controlling terminal โดยไม่ตั้งใจ
```

---

## 20. ตัวอย่างโปรแกรมสมบูรณ์: Process Monitor

```nasm
; ไฟล์: proc_monitor.asm
; Monitor: อ่าน /proc/<pid>/status ของทุก child process

section .data
    proc_prefix     db "/proc/", 0
    proc_prefix_len equ $ - proc_prefix
    status_suffix   db "/status", 0
    newline         db 10, 0
    sep             db "---", 10, 0
    sep_len         equ $ - sep
    
    msg_header      db "=== Process Status Monitor ===", 10, 0
    msg_header_len  equ $ - msg_header
    
    ; VmRSS label ที่ต้องการหา
    vmrss_label     db "VmRSS:", 0

section .bss
    path_buf        resb 64
    read_buf        resb 4096
    pid_str         resb 16

section .text
    global _start

_start:
    ; แสดง header
    mov rdi, 1
    mov rsi, msg_header
    mov rdx, msg_header_len
    mov rax, 1
    syscall
    
    ; ดึง PID ของตัวเอง
    mov rax, 39
    syscall
    mov rdi, rax
    call read_proc_status
    
    xor rdi, rdi
    mov rax, 60
    syscall

; read_proc_status(rdi = pid)
; อ่านและแสดง /proc/<pid>/status
read_proc_status:
    push rbx
    push r12
    push r13
    
    mov r12, rdi        ; เก็บ pid
    
    ; สร้าง path: "/proc/" + pid + "/status"
    lea rbx, [path_buf]
    
    ; copy "/proc/"
    lea rsi, [proc_prefix]
.copy_prefix:
    mov al, [rsi]
    test al, al
    jz .prefix_done
    mov [rbx], al
    inc rbx
    inc rsi
    jmp .copy_prefix
.prefix_done:
    
    ; แปลง pid เป็น string
    mov rdi, r12
    call num_to_str     ; result ใน pid_str
    
    ; copy pid string
    lea rsi, [pid_str]
.copy_pid:
    mov al, [rsi]
    test al, al
    jz .pid_done
    mov [rbx], al
    inc rbx
    inc rsi
    jmp .copy_pid
.pid_done:
    
    ; copy "/status"
    lea rsi, [status_suffix]
.copy_suffix:
    mov al, [rsi]
    mov [rbx], al
    test al, al
    jz .suffix_done
    inc rbx
    inc rsi
    jmp .copy_suffix
.suffix_done:
    
    ; open(path_buf, O_RDONLY)
    lea rdi, [path_buf]
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 2
    syscall
    
    test rax, rax
    js .open_failed
    
    mov r13, rax        ; fd
    
    ; แสดง separator
    mov rdi, 1
    mov rsi, sep
    mov rdx, sep_len
    mov rax, 1
    syscall
    
    ; อ่านและแสดง
.read_loop:
    mov rdi, r13
    mov rsi, read_buf
    mov rdx, 4096
    mov rax, 0
    syscall
    
    test rax, rax
    jle .close
    
    push rax
    mov rdi, 1
    mov rsi, read_buf
    mov rdx, rax
    mov rax, 1
    syscall
    pop rax
    
    cmp rax, 4096
    je .read_loop
    
.close:
    mov rdi, r13
    mov rax, 3
    syscall
    
.open_failed:
    pop r13
    pop r12
    pop rbx
    ret

; num_to_str(rdi = number)
; เขียนผลลัพธ์ไปยัง pid_str
num_to_str:
    push rbx
    push rcx
    push rdx
    
    lea rbx, [pid_str + 15]
    mov byte [rbx], 0
    dec rbx
    
    mov rax, rdi
    mov rcx, 10
.nts_loop:
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rbx], dl
    dec rbx
    test rax, rax
    jnz .nts_loop
    
    inc rbx
    ; copy to beginning
    lea rdi, [pid_str]
.nts_copy:
    mov al, [rbx]
    mov [rdi], al
    test al, al
    jz .nts_done
    inc rdi
    inc rbx
    jmp .nts_copy
.nts_done:
    
    pop rdx
    pop rcx
    pop rbx
    ret
```

---

## 21. Compile และ Run ทั้งหมด

### สคริปต์ Build

```bash
#!/bin/bash
# build_all.sh - Build ทุก assembly program ใน part-054

PROGS=(
    "fork_basic"
    "execve_demo"
    "wait_demo"
    "pid_info"
    "uid_info"
    "dir_ops"
    "chroot_demo"
    "rlimit_demo"
    "proc_maps"
    "signals_demo"
    "simple_shell"
    "daemon_demo"
    "supervisor"
    "proc_monitor"
)

for prog in "${PROGS[@]}"; do
    if [ -f "${prog}.asm" ]; then
        echo "Building ${prog}..."
        nasm -f elf64 "${prog}.asm" -o "${prog}.o" && \
        ld "${prog}.o" -o "${prog}" && \
        echo "  OK: ${prog}" || \
        echo "  FAILED: ${prog}"
    fi
done

echo "Build complete!"
```

### การทดสอบ

```bash
# ทดสอบ fork_basic
./fork_basic

# ทดสอบ execve_demo (จะรัน /bin/ls)
./execve_demo

# ทดสอบ pid_info
./pid_info

# ทดสอบ uid_info
./uid_info

# ทดสอบ dir_ops
./dir_ops

# ทดสอบ rlimit_demo
./rlimit_demo

# ทดสอบ proc_maps (แสดง memory map)
./proc_maps

# ทดสอบ signals_demo
./signals_demo

# ทดสอบ simple_shell (พิมพ์ exit เพื่อออก)
./simple_shell
# ash> ls
# ash> pwd
# ash> exit

# ทดสอบ daemon_demo (จะ fork ไปทำงาน background)
./daemon_demo
sleep 15
cat /tmp/daemon.log

# ทดสอบ supervisor (ต้องมี /tmp/worker ก่อน)
echo '#!/bin/sh' > /tmp/worker
echo 'sleep 2; exit 1' >> /tmp/worker
chmod +x /tmp/worker
./supervisor

# ทดสอบ proc_monitor
./proc_monitor
```

---

## 22. สรุปและ Key Takeaways

### สิ่งสำคัญที่ต้องจำ

1. **fork() = Copy process**: child ได้ RAX=0, parent ได้ RAX=child_PID
2. **Copy-on-Write**: fork() เร็วเพราะ memory ถูก copy เฉพาะเมื่อเขียน
3. **exec() = Replace image**: ไม่สร้าง process ใหม่ แต่แทนที่ program เดิม
4. **argv/envp ต้องจบด้วย NULL pointer**: ขาดไม่ได้!
5. **wait() ป้องกัน Zombie**: Parent ต้อง wait() เพื่อ collect exit status
6. **Zombie vs Orphan**: Zombie = child ตายแล้วรอ parent collect, Orphan = parent ตายก่อน child
7. **Double fork สำหรับ Daemon**: ป้องกันการ acquire controlling terminal
8. **setgid() ก่อน setuid()**: เมื่อ drop privileges
9. **Signal handler ต้องใช้ async-signal-safe functions เท่านั้น**
10. **chroot ต้องตามด้วย chdir("/")** เสมอ

### Syscall Reference สรุป

| ฟังก์ชัน | Syscall # | Arguments | Return |
|---------|-----------|-----------|--------|
| fork() | 57 | - | child: 0, parent: PID, error: -errno |
| execve() | 59 | path, argv, envp | ไม่ return ถ้าสำเร็จ, -errno ถ้า error |
| wait4() | 61 | pid, *status, options, *rusage | PID ของ child, -errno |
| getpid() | 39 | - | PID |
| getppid() | 110 | - | PPID |
| getuid() | 102 | - | UID |
| geteuid() | 107 | - | EUID |
| getgid() | 104 | - | GID |
| getegid() | 108 | - | EGID |
| setuid() | 105 | uid | 0 หรือ -errno |
| setgid() | 106 | gid | 0 หรือ -errno |
| kill() | 62 | pid, sig | 0 หรือ -errno |
| chdir() | 80 | path | 0 หรือ -errno |
| getcwd() | 79 | buf, size | bytes written |
| chroot() | 161 | path | 0 หรือ -errno |
| getrlimit() | 97 | resource, *rlim | 0 หรือ -errno |
| setrlimit() | 160 | resource, *rlim | 0 หรือ -errno |
| setsid() | 112 | - | session ID หรือ -errno |

---

## ภาคผนวก: โครงสร้างข้อมูลที่เกี่ยวข้อง

### struct timespec

```nasm
; struct timespec {
;     time_t tv_sec;    /* 8 bytes - วินาที */
;     long   tv_nsec;   /* 8 bytes - nanoseconds (0-999999999) */
; };
struc timespec_t
    .tv_sec     resq 1
    .tv_nsec    resq 1
endstruc
```

### struct rlimit

```nasm
; struct rlimit {
;     rlim_t rlim_cur;  /* 8 bytes - soft limit */
;     rlim_t rlim_max;  /* 8 bytes - hard limit */
; };
; RLIM_INFINITY = 0xFFFFFFFFFFFFFFFF
struc rlimit_t
    .rlim_cur   resq 1
    .rlim_max   resq 1
endstruc
```

### struct sigaction (simplified)

```nasm
; struct sigaction {
;     void (*sa_handler)(int);  /* 8 bytes - handler หรือ SIG_DFL(0)/SIG_IGN(1) */
;     unsigned long sa_flags;   /* 8 bytes */
;     void (*sa_restorer)(void); /* 8 bytes */
;     sigset_t sa_mask;          /* 128 bytes */
; };
; ขนาดรวม: 152 bytes
SA_RESTORER     equ 0x04000000
SA_SIGINFO      equ 4
SA_NOCLDWAIT    equ 2

SIG_DFL         equ 0       ; default handler
SIG_IGN         equ 1       ; ignore signal
```

---

*จบ Part 054: Process Management ใน Assembly*

*ส่วนต่อไป: Part 055 - Inter-Process Communication (Pipes, Sockets, Shared Memory)*
