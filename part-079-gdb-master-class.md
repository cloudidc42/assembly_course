# Part 079: GDB Master Class - คู่มือ GDB ฉบับสมบูรณ์

## บทนำ

GDB (GNU Debugger) คือเครื่องมือ debugging ที่ทรงพลังที่สุดสำหรับโปรแกรมภาษา C/C++ และ Assembly บน Linux  
ในบทนี้เราจะเรียนรู้การใช้ GDB ตั้งแต่พื้นฐานไปจนถึงขั้นสูง รวมถึง plugin เช่น PEDA, PWNDBG และ GEF  
ที่ใช้กันมากในการแข่งขัน CTF (Capture The Flag) และ Security Research

---

## สารบัญ

1. [GDB Startup และการเปิดไฟล์](#gdb-startup)
2. [Basic Commands พื้นฐาน](#basic-commands)
3. [Info Commands](#info-commands)
4. [Breakpoints](#breakpoints)
5. [Watchpoints](#watchpoints)
6. [Catchpoints](#catchpoints)
7. [x Command - Examine Memory](#x-command)
8. [disassemble Command](#disassemble-command)
9. [print Command](#print-command)
10. [backtrace และ Frame Navigation](#backtrace)
11. [TUI Mode](#tui-mode)
12. [Python Scripting](#python-scripting)
13. [PEDA Plugin](#peda-plugin)
14. [PWNDBG Plugin](#pwndbg-plugin)
15. [GEF Plugin](#gef-plugin)
16. [Remote Debugging](#remote-debugging)
17. [CTF Techniques](#ctf-techniques)
18. [Advanced Tips](#advanced-tips)

---

## 1. GDB Startup {#gdb-startup}

### การเปิด Binary ธรรมดา

```bash
# เปิด binary file
gdb ./binary
gdb ./vulnerable_program

# เปิดพร้อม arguments
gdb --args ./binary arg1 arg2 arg3
gdb --args ./program -f input.txt --verbose

# ตัวอย่าง: program ที่รับ arguments
gdb --args ./ctf_challenge AAAA BBBB

# Quiet mode (ไม่แสดง copyright notice)
gdb -q ./binary
gdb --quiet ./binary

# เปิดและรัน script อัตโนมัติ
gdb -x script.gdb ./binary
gdb --command=init.gdb ./binary

# Batch mode (ไม่ interactive)
gdb -batch -ex "run" -ex "bt" ./binary
```

### Attach กับ Running Process

```bash
# Attach โดยใช้ PID
gdb -p 1234
gdb --pid=1234

# หา PID ก่อน
ps aux | grep program_name
pidof program_name
pgrep program_name

# ตัวอย่าง workflow
$ pidof vulnerable_server
4521
$ gdb -p 4521
# GDB จะ attach และ pause process นั้น

# Attach แบบ non-invasive (อ่านอย่างเดียว)
gdb -p 1234 --readonly
```

### Core Files

Core dump คือ snapshot ของ memory ขณะที่ program crash  
มีประโยชน์มากในการวิเคราะห์ crash หลังจากเกิดขึ้นแล้ว

```bash
# เปิด core file
gdb binary core
gdb binary core.1234
gdb ./program /var/crash/core.5678

# ตัวอย่าง
$ ./vulnerable_program
Segmentation fault (core dumped)
$ gdb ./vulnerable_program core
# GDB จะโหลด core และแสดงว่า crash ที่ไหน

# เปิดใช้ core dump (ต้องทำก่อน)
ulimit -c unlimited
# หรือใน /etc/security/limits.conf
# * soft core unlimited

# กำหนด pattern ของ core file
echo "/tmp/core-%e-%p-%t" > /proc/sys/kernel/core_pattern
# %e = executable name
# %p = PID
# %t = timestamp

# อ่าน core dump โดยไม่มี binary (limited)
gdb -c core
```

### GDB Init Files

```bash
# ~/.gdbinit - รันทุกครั้งที่เปิด GDB
# ./.gdbinit - รันเมื่อเปิดในโฟลเดอร์นั้น

# ตัวอย่าง ~/.gdbinit
cat ~/.gdbinit
set disassembly-flavor intel
set pagination off
set print pretty on
set print array on
```

---

## 2. Basic Commands พื้นฐาน {#basic-commands}

### run (r) - รันโปรแกรม

```gdb
# รันโปรแกรม
(gdb) run
(gdb) r

# รันพร้อม arguments
(gdb) run arg1 arg2 arg3
(gdb) r "hello world" file.txt

# รันพร้อม stdin redirect
(gdb) run < input.txt
(gdb) r <<< "AAAAAAAAAA"

# รันพร้อม environment variable
(gdb) set environment LD_PRELOAD=./hook.so
(gdb) run

# รันซ้ำด้วย arguments เดิม
(gdb) run
# (ถ้ารันครั้งแรกแล้ว ครั้งต่อไปใช้ arguments เดิม)
```

### continue (c) - ดำเนินต่อ

```gdb
# ดำเนินโปรแกรมต่อจาก breakpoint ปัจจุบัน
(gdb) continue
(gdb) c

# ดำเนินต่อ N ครั้ง (ข้าม breakpoint N-1 ครั้ง)
(gdb) continue 5
(gdb) c 3

# ตัวอย่าง: มี loop ที่ hit breakpoint หลายครั้ง
# ต้องการ continue จนถึงครั้งที่ 10
(gdb) c 9
```

### next (n) - Step Over

```gdb
# Execute บรรทัดปัจจุบัน แล้วหยุดที่บรรทัดถัดไป
# ถ้าเป็น function call จะ "ข้าม" เข้าไปใน function นั้น
(gdb) next
(gdb) n

# next N บรรทัด
(gdb) next 5
(gdb) n 3

# ตัวอย่าง: debug loop
(gdb) b main
(gdb) r
Breakpoint 1, main () at prog.c:10
10    for (int i = 0; i < 100; i++) {
(gdb) n
11        process(i);
(gdb) n        # ข้าม process() ไปเลย
12    }
```

### step (s) - Step Into

```gdb
# Execute บรรทัดปัจจุบัน และเข้าไปใน function call
(gdb) step
(gdb) s

# step N ครั้ง
(gdb) step 5

# ตัวอย่าง: เข้าไปใน function
(gdb) n
11        process(i);
(gdb) s        # เข้าไปใน process()
process (x=0) at prog.c:25
25    int result = x * 2;
```

### finish - รันจนจบ function ปัจจุบัน

```gdb
# รันต่อจนออกจาก function ปัจจุบัน
(gdb) finish

# ตัวอย่าง
(gdb) s
process (x=0) at prog.c:25
25    int result = x * 2;
(gdb) finish
Run till exit from #0  process (x=0) at prog.c:25
Value returned is $1 = 0
main () at prog.c:12
12    }
```

### until - รันจนถึงบรรทัดที่กำหนด

```gdb
# รันจนถึงบรรทัดถัดไป (ออกจาก loop)
(gdb) until

# รันจนถึงบรรทัดที่กำหนด
(gdb) until 50
(gdb) until prog.c:75

# ตัวอย่าง: ออกจาก loop โดยไม่ต้อง step ทีละบรรทัด
(gdb) b main
(gdb) r
(gdb) n
# อยู่ที่ loop
(gdb) until 20   # รันถึงบรรทัด 20 (หลัง loop)
```

### jump - กระโดดไปยัง address/line

```gdb
# กระโดดไปรันที่ address นั้น (อันตราย!)
(gdb) jump *0x401234
(gdb) jump 50      # กระโดดไปบรรทัด 50

# ใช้ใน CTF เพื่อข้าม check
(gdb) jump *0x4011a5   # กระโดดข้าม authentication check
```

### set - เปลี่ยนค่า

```gdb
# เปลี่ยนค่า variable
(gdb) set variable x = 10
(gdb) set var result = 0x1337

# เปลี่ยน register
(gdb) set $rax = 0
(gdb) set $eip = 0x401234

# เปลี่ยน memory
(gdb) set {int}0x601080 = 42
(gdb) set {char[10]}0x601080 = "AAAAAAAAAA"
```

---

## 3. Info Commands {#info-commands}

### info registers

```gdb
# แสดง registers ทั้งหมด
(gdb) info registers
(gdb) i r

# แสดง register เฉพาะ
(gdb) info registers rax rbx rcx
(gdb) i r rip rsp rbp

# แสดง all registers รวม floating point
(gdb) info all-registers
(gdb) i all-r

# ตัวอย่าง output:
# rax            0x0                 0
# rbx            0x0                 0
# rcx            0x7ffff7af4224      140737348813348
# rdx            0x7ffff7dcf8c0      140737351841984
# rsi            0x602260            6300256
# rdi            0x1                 1
# rbp            0x7fffffffde90      0x7fffffffde90
# rsp            0x7fffffffde70      0x7fffffffde70
# r8             0x7ffff7fef4c0      140737354072256
# rip            0x401176            0x401176 <main+14>
# eflags         0x246               [ PF ZF IF ]
# cs             0x33                51
# ss             0x2b                43
```

### info breakpoints

```gdb
# แสดง breakpoints ทั้งหมด
(gdb) info breakpoints
(gdb) i b

# ตัวอย่าง output:
# Num     Type           Disp Enb Address            What
# 1       breakpoint     keep y   0x0000000000401162 in main at prog.c:8
#         breakpoint already hit 1 time
# 2       breakpoint     keep y   0x00000000004011a5 in check_password at prog.c:25
# 3       watchpoint     keep y   global_var
```

### info locals

```gdb
# แสดง local variables ของ frame ปัจจุบัน
(gdb) info locals
(gdb) i locals

# ตัวอย่าง output:
# i = 5
# buffer = "Hello\000\000\000\000\000"
# result = 0
# ptr = 0x602260
```

### info args

```gdb
# แสดง arguments ของ function ปัจจุบัน
(gdb) info args
(gdb) i args

# ตัวอย่าง output:
# argc = 2
# argv = 0x7fffffffe088
```

### info threads

```gdb
# แสดง threads ทั้งหมด
(gdb) info threads
(gdb) i threads

# ตัวอย่าง output:
#   Id   Target Id         Frame 
# * 1    Thread 0x7ffff7fd7740 (LWP 4521) "program" main () at prog.c:10
#   2    Thread 0x7ffff77d4700 (LWP 4522) "program" 0x00007ffff7bc6dc3 in ...
#   3    Thread 0x7ffff6fd3700 (LWP 4523) "program" 0x00007ffff7bc6dc3 in ...

# เปลี่ยน thread
(gdb) thread 2
(gdb) t 3
```

### info proc

```gdb
# แสดงข้อมูล process
(gdb) info proc
(gdb) i proc

# แสดง memory map
(gdb) info proc mappings
(gdb) i proc map

# ตัวอย่าง output:
# Start Addr   End Addr       Size     Offset objfile
# 0x400000   0x401000     0x1000        0x0 /home/user/program
# 0x600000   0x601000     0x1000        0x0 /home/user/program
# 0x601000   0x602000     0x1000     0x1000 /home/user/program
# 0x7ffff7a0d000 0x7ffff7bcd000   0x1c0000    0x0 /lib/x86_64-linux-gnu/libc.so.6
# 0x7ffffffde000 0x7ffffffff000    0x21000    0x0 [stack]

# แสดง status
(gdb) info proc status

# แสดง files ที่เปิดอยู่
(gdb) info proc files
```

### info functions / variables / types

```gdb
# แสดง functions ทั้งหมด
(gdb) info functions
(gdb) i functions

# ค้นหา function ด้วย regex
(gdb) info functions ^main
(gdb) info functions check.*

# แสดง global variables
(gdb) info variables
(gdb) i variables

# แสดง types
(gdb) info types
```

### info sharedlibrary

```gdb
# แสดง shared libraries ที่โหลดอยู่
(gdb) info sharedlibrary
(gdb) i shared

# ตัวอย่าง output:
# From                To                  Syms Read   Shared Object Library
# 0x00007ffff7dd7ac0  0x00007ffff7df5cc8  Yes         /lib64/ld-linux-x86-64.so.2
# 0x00007ffff7a52740  0x00007ffff7b9f52c  Yes         /lib/x86_64-linux-gnu/libc.so.6
```

---

## 4. Breakpoints {#breakpoints}

### การตั้ง Breakpoint

```gdb
# Break ที่ function
(gdb) break main
(gdb) b main
(gdb) break check_password
(gdb) b vulnerable_func

# Break ที่ address (hex)
(gdb) break *0x401234
(gdb) b *0x4011a5
(gdb) b *main+20

# Break ที่บรรทัดใน source file
(gdb) break prog.c:25
(gdb) b 30          # บรรทัดปัจจุบัน file

# Break ที่ offset จาก function
(gdb) break *main+50
(gdb) b *check_password+0x10

# Break ด้วย condition
(gdb) break main if argc > 1
(gdb) b check_password if strcmp(password, "secret") == 0
(gdb) b loop_func if i == 99

# ตัวอย่าง CTF: break เมื่อ input ถูกต้อง
(gdb) b check_flag if $rax == 1
```

### Conditional Breakpoints

```gdb
# ตั้ง condition หลังจากตั้ง breakpoint แล้ว
(gdb) break vulnerable
(gdb) condition 1 buf_size > 100

# เปลี่ยน condition
(gdb) condition 1 new_condition

# ลบ condition (breakpoint ยังอยู่)
(gdb) condition 1

# Breakpoint พร้อม ignore count
(gdb) break loop
(gdb) ignore 1 999   # hit breakpoint 999 ครั้งก่อน

# ดูสถานะ
(gdb) info breakpoints
```

### Enable / Disable / Delete Breakpoints

```gdb
# Disable breakpoint (ยังคงอยู่แต่ไม่ทำงาน)
(gdb) disable 1
(gdb) disable 1 2 3
(gdb) disable breakpoints    # ทั้งหมด

# Enable breakpoint
(gdb) enable 1
(gdb) enable 1 2 3
(gdb) enable breakpoints     # ทั้งหมด

# Enable once (enable แล้ว disable ตัวเองหลัง hit ครั้งแรก)
(gdb) enable once 1

# Delete breakpoint
(gdb) delete 1
(gdb) d 1 2 3
(gdb) delete breakpoints    # ลบทั้งหมด
(gdb) clear main            # ลบ breakpoint ที่ main
(gdb) clear *0x401234

# ลบและ log
(gdb) delete 1
Deleted breakpoint 1
```

### Breakpoint Commands

```gdb
# รัน commands เมื่อ hit breakpoint
(gdb) break main
(gdb) commands 1
> print $rax
> print $rsp
> continue
> end

# ตัวอย่าง: log ค่า argument ทุกครั้ง
(gdb) break process_input
(gdb) commands 1
> printf "Input: %s\n", (char*)$rdi
> continue
> end

# Silent breakpoint (ไม่แสดง message)
(gdb) break loop
(gdb) commands 1
> silent
> print i
> continue
> end
```

### Temporary Breakpoints

```gdb
# Breakpoint ที่ลบตัวเองหลัง hit ครั้งแรก
(gdb) tbreak main
(gdb) tb *0x401234
(gdb) thbreak main   # hardware temporary breakpoint
```

---

## 5. Watchpoints {#watchpoints}

Watchpoint คือ breakpoint ที่ทำงานเมื่อ memory location มีการเปลี่ยนแปลง

### watch - Watch เมื่อมีการ Write

```gdb
# Watch variable (หยุดเมื่อค่า write)
(gdb) watch variable_name
(gdb) watch global_counter
(gdb) watch array[5]

# Watch memory address
(gdb) watch *0x601080
(gdb) watch *(int*)0x7fffffffde84

# Watch expression
(gdb) watch (int)(*ptr)
(gdb) watch strcmp(buffer, "admin")

# ตัวอย่าง: ติดตามการเปลี่ยนแปลงของ password buffer
(gdb) watch password_buffer
Watchpoint 2: password_buffer
(gdb) c
Continuing.
Hardware watchpoint 2: password_buffer
Old value = "password123"
New value = "newpass"
0x00000000004011f5 in set_password (new_pwd=0x7fffffffde70 "newpass") at prog.c:42
42        strcpy(password_buffer, new_pwd);
```

### rwatch - Watch เมื่อมีการ Read

```gdb
# หยุดเมื่อมีการ read memory location
(gdb) rwatch variable_name
(gdb) rwatch *0x601080
(gdb) rwatch secret_key

# ตัวอย่าง: ติดตามว่าใครอ่าน secret key
(gdb) rwatch secret_flag
Hardware read watchpoint 3: secret_flag
# เมื่อ code อ่าน secret_flag จะหยุด
```

### awatch - Watch เมื่อ Read หรือ Write

```gdb
# หยุดทั้งเมื่อ read และ write
(gdb) awatch variable_name
(gdb) awatch *0x601080

# ตัวอย่าง: ติดตาม access ทั้งหมดของ critical variable
(gdb) awatch admin_flag
```

### Hardware vs Software Watchpoints

```gdb
# Hardware watchpoints: เร็วกว่า, จำกัดจำนวน (ปกติ 4 บน x86)
# Software watchpoints: ช้ากว่ามาก (GDB check ทุก single step)

# ดูจำนวน hardware watchpoints ที่รองรับ
(gdb) show can-use-hw-watchpoints

# บังคับใช้ software watchpoint
(gdb) set can-use-hw-watchpoints 0

# Hardware watchpoint จะใช้ DR0-DR3 registers ของ x86
# ดู debug registers
(gdb) info registers dr0 dr1 dr2 dr3 dr6 dr7

# จำกัดของ hardware watchpoints:
# - x86: 4 watchpoints
# - แต่ละ watchpoint ดู 1, 2, 4, หรือ 8 bytes
# - ถ้าเกิน limit จะ fallback เป็น software watchpoint
```

---

## 6. Catchpoints {#catchpoints}

Catchpoint คือ breakpoint สำหรับ events พิเศษ เช่น syscalls, exceptions, signals

### catch syscall

```gdb
# หยุดเมื่อ program เรียก syscall
(gdb) catch syscall
(gdb) catch syscall read
(gdb) catch syscall write
(gdb) catch syscall open
(gdb) catch syscall execve

# หยุดเมื่อเรียก syscall หลายตัว
(gdb) catch syscall mmap mprotect mmap2

# ดู syscall numbers
(gdb) catch syscall 0   # read (syscall #0)
(gdb) catch syscall 1   # write (syscall #1)
(gdb) catch syscall 59  # execve (syscall #59)

# ตัวอย่าง: ติดตามการ open files
(gdb) catch syscall open openat
(gdb) r
Catchpoint 1 (call to syscall openat), ...
# แสดงว่า program เปิด file อะไร

# Catch เมื่อ syscall return (หลัง execute)
(gdb) catch syscall
# จะหยุดทั้งก่อนและหลัง syscall
```

### catch fork / vfork / exec

```gdb
# หยุดเมื่อ program fork
(gdb) catch fork
(gdb) catch vfork

# หยุดเมื่อ exec
(gdb) catch exec

# Follow child process หลัง fork
(gdb) set follow-fork-mode child
(gdb) set follow-fork-mode parent    # default

# Detach parent เมื่อ follow child
(gdb) set detach-on-fork on

# ตัวอย่าง: debug daemon ที่ fork
(gdb) catch fork
(gdb) set follow-fork-mode child
(gdb) r
# GDB จะ follow child process
```

### catch signal

```gdb
# หยุดเมื่อรับ signal
(gdb) catch signal SIGSEGV
(gdb) catch signal SIGILL
(gdb) catch signal all

# ตัวอย่าง: debug segfault
(gdb) catch signal SIGSEGV
(gdb) r
Catchpoint 1 (signal SIGSEGV), ...
# แสดงว่า segfault เกิดที่ไหน

# จัดการ signal
(gdb) handle SIGSEGV stop    # หยุดเมื่อรับ
(gdb) handle SIGINT nostop   # ไม่หยุดเมื่อรับ
(gdb) handle SIGALRM pass    # ส่ง signal ไปให้ program
(gdb) handle SIGPIPE nopass  # ไม่ส่ง signal ไปให้ program

# ดูสถานะ signal handling
(gdb) info signals
```

### catch throw / catch

```gdb
# C++ exceptions
(gdb) catch throw           # หยุดเมื่อ throw exception
(gdb) catch catch           # หยุดเมื่อ catch exception
(gdb) catch throw std::runtime_error
```

---

## 7. x Command - Examine Memory {#x-command}

`x` command ใช้ตรวจสอบ memory - เป็นหนึ่งใน commands ที่ใช้บ่อยที่สุด

### Format ของ x command

```
x/[count][format][size] address

Format:
  x = hex (default)
  d = decimal (signed)
  u = unsigned decimal
  o = octal
  t = binary
  f = float
  a = address (pointer)
  c = character
  s = string (null-terminated)
  i = instruction (disassemble)
  z = hex with leading zeros

Size:
  b = byte (1 byte)
  h = halfword (2 bytes)
  w = word (4 bytes)
  g = giant/quadword (8 bytes)
```

### ตัวอย่างการใช้ x command

```gdb
# ดู 10 values แบบ hex ขนาด 8 bytes (quadword)
(gdb) x/10xg 0x7fffffffde80
0x7fffffffde80: 0x0000000000000001  0x00007fffffffe288
0x7fffffffde90: 0x0000000000000000  0x0000000000400626
0x7fffffffdea0: 0x0000000000000001  0x00007fffffffe288
0x7fffffffdeb0: 0x00007fffffffe298  0x0000000000000000
0x7fffffffdec0: 0x00007ffff7de59a0  0x0000000000000000

# ดู 20 instructions ที่ address
(gdb) x/20i 0x401162
   0x401162 <main>:      push   rbp
   0x401163 <main+1>:    mov    rbp,rsp
   0x401166 <main+4>:    sub    rsp,0x50
   ...

# ดู string ที่ address
(gdb) x/s 0x402004
0x402004: "Hello, World!"

# ดู string ที่ register
(gdb) x/s $rdi
0x402004: "Hello, World!"

# ดู 10 bytes
(gdb) x/10xb 0x601080
0x601080: 0x41  0x41  0x41  0x41  0x41  0x41  0x41  0x41
0x601088: 0x00  0x00

# ดู 10 words (4 bytes each) แบบ decimal
(gdb) x/10dw 0x601080

# ดู hex พร้อม leading zeros
(gdb) x/4xg $rsp
0x7fffffffde80: 0x0000000000000001  0x00007fffffffe288
0x7fffffffde90: 0x0000000000000000  0x0000000000400626

# ดู stack 20 entries
(gdb) x/20xg $rsp

# ดู characters
(gdb) x/20c 0x402004
0x402004: 72 'H'  101 'e'  108 'l'  108 'l'  111 'o'  44 ','  32 ' '

# ดู binary
(gdb) x/4tb $rsp
0x7fffffffde80: 00000001  00000000  00000000  00000000

# ดู float
(gdb) x/4f 0x601080
0x601080: 3.14159  2.71828  1.41421  1.73205

# ดู pointer/address
(gdb) x/4a $rsp
0x7fffffffde80: 0x1  0x7fffffffe288
0x7fffffffde90: 0x0  0x400626 <main>
```

### x command กับ relative address

```gdb
# ดูจาก symbol
(gdb) x/10xg &global_var
(gdb) x/s &string_var

# ดูจาก register offset
(gdb) x/10xg $rbp-0x50
(gdb) x/4xg $rsp+8

# ดูจาก pointer
(gdb) x/s *((char**)$rdi)

# เทคนิค: ดู GOT table
(gdb) x/20xg 0x601018   # GOT address
(gdb) x/20xg &_GLOBAL_OFFSET_TABLE_

# เทคนิค: ดู heap
(gdb) x/100xg 0x602000  # heap start (ประมาณ)
```

---

## 8. disassemble Command {#disassemble-command}

### disassemble พื้นฐาน

```gdb
# Disassemble function ปัจจุบัน
(gdb) disassemble
(gdb) disas

# Disassemble function ที่กำหนด
(gdb) disassemble main
(gdb) disas check_password
(gdb) disas vulnerable_func

# Disassemble address range
(gdb) disassemble 0x401162, 0x4011a5
(gdb) disas 0x401162, +100    # +100 = 100 bytes หลัง start

# ตั้งค่า flavor (Intel vs AT&T)
(gdb) set disassembly-flavor intel    # แนะนำสำหรับ CTF
(gdb) set disassembly-flavor att      # default

# ดู flavor ปัจจุบัน
(gdb) show disassembly-flavor
```

### disas /m - Mixed Source and Assembly

```gdb
# แสดง source code พร้อม assembly (ต้องมี debug symbols)
(gdb) disassemble /m main
(gdb) disas /m check_password

# ตัวอย่าง output:
# 8       int main(int argc, char *argv[]) {
#    0x0000000000401162 <+0>:     push   rbp
#    0x0000000000401163 <+1>:     mov    rbp,rsp
#
# 9           char buffer[64];
#    0x0000000000401166 <+4>:     sub    rsp,0x50
#
# 10          scanf("%s", buffer);
#    0x000000000040116a <+8>:     lea    rdi,[rip+0xe93]
#    0x0000000000401171 <+15>:    ...
```

### disas /r - Show Raw Bytes

```gdb
# แสดง raw bytes พร้อม assembly
(gdb) disassemble /r main
(gdb) disas /r check_password

# ตัวอย่าง output:
# 0x0000000000401162 <+0>:     55                      push   rbp
# 0x0000000000401163 <+1>:     48 89 e5                mov    rbp,rsp
# 0x0000000000401166 <+4>:     48 83 ec 50             sub    rsp,0x50
# 0x000000000040116a <+8>:     48 8d 3d 93 0e 00 00    lea    rdi,[rip+0xe93]

# Combine /m และ /r
(gdb) disas /mr main
```

### x/i สำหรับ Disassemble

```gdb
# Disassemble N instructions ที่ address
(gdb) x/20i 0x401162
(gdb) x/20i main
(gdb) x/10i $rip       # instructions จาก current PC

# Disassemble backward (ดู instructions ก่อน current)
(gdb) x/10i $rip-30
```

---

## 9. print Command {#print-command}

### print พื้นฐาน

```gdb
# Print ค่า variable
(gdb) print variable_name
(gdb) p var

# Print expression
(gdb) print 1 + 2
(gdb) p argc + argv[0][0]

# Print ค่า register
(gdb) print $rax
(gdb) p $rip
(gdb) p/x $rsp    # hex format

# Print ค่า memory
(gdb) print *ptr
(gdb) p *array

# Print type cast
(gdb) print (char*)$rdi
(gdb) p (int*)0x601080
(gdb) p *(struct node*)0x602260
```

### print formats

```gdb
# /x = hex
(gdb) print/x variable
(gdb) p/x $rax

# /d = decimal (signed)
(gdb) print/d variable
(gdb) p/d $rbx

# /u = unsigned decimal
(gdb) print/u variable

# /o = octal
(gdb) print/o variable

# /t = binary
(gdb) print/t $rflags    # ดู flags register แบบ binary
(gdb) p/t 255            # = 11111111

# /f = float
(gdb) print/f $xmm0

# /c = character
(gdb) print/c 65         # = 65 'A'
(gdb) p/c $rdi

# /a = address
(gdb) print/a 0x401162   # = 0x401162 <main>

# /s = string
(gdb) print/s 0x402004
(gdb) p/s (char*)$rdi
```

### print address / pointer

```gdb
# ดู address ของ variable
(gdb) print &variable
(gdb) p &global_var
(gdb) p &local_var

# Cast และ dereference
(gdb) print (int*)0x601080
(gdb) p *(int*)0x601080
(gdb) p *(char**)($rsp+8)

# Array access
(gdb) print array[5]
(gdb) p *((int*)0x601080 + 5)

# Struct member
(gdb) print struct_var.member
(gdb) p ptr->member

# ตัวอย่าง: ดู argc และ argv
(gdb) p argc
(gdb) p *argv
(gdb) p argv[0]
(gdb) p argv[1]
```

### GDB History Variables

```gdb
# ผลลัพธ์จาก print ถูก save เป็น $1, $2, $3, ...
(gdb) p $rax
$1 = 0
(gdb) p $1 + 10
$2 = 10

# $ = ผลลัพธ์ล่าสุด
(gdb) p array
$3 = {1, 2, 3, 4, 5}
(gdb) p $[2]    # element ที่ 2 ของผลลัพธ์ล่าสุด
$4 = 3

# ใช้ใน commands
(gdb) p $rsp
$5 = 0x7fffffffde80
(gdb) x/20xg $5
```

### display - Auto-print

```gdb
# Auto print ทุก step
(gdb) display $rip
(gdb) display $rax
(gdb) display buffer

# ดู display list
(gdb) info display

# Disable/enable display
(gdb) disable display 1
(gdb) enable display 1

# Delete display
(gdb) undisplay 1
(gdb) delete display 1
```

---

## 10. backtrace และ Frame Navigation {#backtrace}

### backtrace (bt)

```gdb
# แสดง call stack
(gdb) backtrace
(gdb) bt

# แสดง N frames จาก top
(gdb) bt 5
(gdb) backtrace 5

# แสดง N frames จาก bottom
(gdb) bt -5
(gdb) backtrace -5

# แสดงพร้อม full locals/args
(gdb) bt full
(gdb) backtrace full

# ตัวอย่าง output:
# #0  vulnerable_func (buf=0x7fffffffde40 "AAAA") at prog.c:15
# #1  0x00000000004011d5 in process (input=0x7fffffffe280 "AAAA") at prog.c:35
# #2  0x0000000000401209 in main (argc=2, argv=0x7fffffffe088) at prog.c:50
```

### frame navigation

```gdb
# ดู frame ปัจจุบัน
(gdb) frame
(gdb) f

# เปลี่ยนไป frame ที่กำหนด
(gdb) frame 0    # top frame (ปัจจุบัน)
(gdb) frame 1    # frame ที่ 1
(gdb) f 2

# ขึ้น (ไป caller)
(gdb) up
(gdb) up 3    # ขึ้น 3 frames

# ลง (ไป callee)
(gdb) down
(gdb) down 2

# select-frame (เหมือน frame แต่ไม่แสดง info)
(gdb) select-frame 1

# ดูข้อมูล frame ปัจจุบัน
(gdb) info frame
# ตัวอย่าง:
# Stack level 0, frame at 0x7fffffffde50:
#  rip = 0x401176 in main (prog.c:10); saved rip = 0x7ffff7a05b97
#  source language c.
#  Arglist at 0x7fffffffde40, args: argc=1, argv=0x7fffffffe098
#  Locals at 0x7fffffffde40, Previous frame's sp is 0x7fffffffde50
#  Saved registers:
#   rbp at 0x7fffffffde40, rip at 0x7fffffffde48
```

### where และ info stack

```gdb
# เหมือน backtrace
(gdb) where
(gdb) info stack

# ดู frame ทั้งหมดพร้อม addresses
(gdb) info frame 0
(gdb) info frame 1
```

---

## 11. TUI Mode {#tui-mode}

TUI (Text User Interface) Mode แสดง source code, assembly และ registers พร้อมกัน

### เปิด/ปิด TUI Mode

```gdb
# เปิด TUI Mode
(gdb) tui enable
# หรือกด Ctrl+X A

# ปิด TUI Mode
(gdb) tui disable
# หรือกด Ctrl+X A อีกครั้ง

# Toggle TUI
Ctrl+X A
```

### Layout

```gdb
# ดู source code
(gdb) layout src
(gdb) layout source

# ดู assembly
(gdb) layout asm

# ดู registers
(gdb) layout regs

# ดู source + assembly
(gdb) layout split

# ดู source + registers
(gdb) layout src
(gdb) tui reg general   # + registers

# Next layout
(gdb) layout next
(gdb) layout prev
```

### Keyboard Shortcuts ใน TUI Mode

```
Ctrl+X A    - Toggle TUI mode
Ctrl+X 1    - Layout src
Ctrl+X 2    - Layout asm / split (สลับ)
Ctrl+X O    - เปลี่ยน active window
Ctrl+L      - Refresh screen
PageUp       - Scroll source up
PageDown     - Scroll source down
Up/Down      - Scroll active window
```

### TUI Commands เพิ่มเติม

```gdb
# Focus window
(gdb) focus src
(gdb) focus asm
(gdb) focus regs
(gdb) focus cmd

# Register groups
(gdb) tui reg general      # general purpose registers
(gdb) tui reg float        # floating point registers
(gdb) tui reg system       # system registers
(gdb) tui reg next         # next register group

# Refresh
(gdb) refresh

# ปรับขนาด (บน terminal)
(gdb) winheight src +5
(gdb) winheight asm -3
```

---

## 12. Python Scripting {#python-scripting}

GDB มี Python API ที่ทรงพลัง ช่วยให้เขียน automation scripts ได้

### Python basics ใน GDB

```gdb
# รัน Python inline
(gdb) python print("Hello from Python!")
(gdb) py print(gdb.selected_frame().pc())

# Multi-line Python
(gdb) python
> import gdb
> frame = gdb.selected_frame()
> print(frame.name())
> end

# โหลด Python script
(gdb) source script.py
(gdb) python exec(open('script.py').read())
```

### gdb.parse_and_eval

```python
# Evaluate expression ใน context ของ program
import gdb

# ดึงค่า variable
val = gdb.parse_and_eval("variable_name")
print(val)

# ดึงค่า register
rax = gdb.parse_and_eval("$rax")
rsp = gdb.parse_and_eval("$rsp")
print(f"RAX = {int(rax):#x}")

# Cast และ dereference
ptr = gdb.parse_and_eval("(char*)$rdi")
print(ptr.string())

# ดึง struct member
member = gdb.parse_and_eval("my_struct.field")

# ตัวอย่าง: ดึงทุก arguments
frame = gdb.selected_frame()
block = frame.block()
for sym in block:
    if sym.is_argument:
        val = sym.value(frame)
        print(f"{sym.name} = {val}")
```

### Breakpoint Hooks ด้วย Python

```python
# สร้าง breakpoint ด้วย Python
import gdb

class MyBreakpoint(gdb.Breakpoint):
    def __init__(self, spec):
        super().__init__(spec)
    
    def stop(self):
        # code นี้รันทุกครั้งที่ hit breakpoint
        rdi = gdb.parse_and_eval("$rdi")
        print(f"[*] Function called with rdi = {int(rdi):#x}")
        
        # ดึง string argument
        try:
            arg_str = gdb.parse_and_eval("(char*)$rdi").string()
            print(f"[*] String argument: {arg_str}")
        except:
            pass
        
        return False  # False = continue, True = stop

# ติดตั้ง breakpoint
bp = MyBreakpoint("check_password")
```

### Advanced Python Scripts

```python
# Script: ติดตาม function calls
import gdb

class FunctionTracer(gdb.Breakpoint):
    call_count = {}
    
    def __init__(self, func_name):
        super().__init__(func_name)
        self.func_name = func_name
        FunctionTracer.call_count[func_name] = 0
    
    def stop(self):
        FunctionTracer.call_count[self.func_name] += 1
        count = FunctionTracer.call_count[self.func_name]
        frame = gdb.selected_frame()
        print(f"[TRACE] {self.func_name} called (#{count})")
        print(f"  PC: {frame.pc():#x}")
        return False

# ติดตาม functions ที่น่าสนใจ
FunctionTracer("malloc")
FunctionTracer("free")
FunctionTracer("strcpy")
FunctionTracer("gets")

print("[*] Function tracing enabled")
```

```python
# Script: Memory leak detector
import gdb

allocs = {}

class MallocBP(gdb.Breakpoint):
    def __init__(self):
        super().__init__("malloc", internal=True)
    
    def stop(self):
        size = int(gdb.parse_and_eval("$rdi"))
        # malloc argument อยู่ใน rdi
        gdb.post_event(lambda: self.record_alloc(size))
        return False
    
    def record_alloc(self, size):
        # ดึง return value หลัง malloc return
        ret = int(gdb.parse_and_eval("$rax"))
        if ret != 0:
            allocs[ret] = size
            print(f"[ALLOC] {ret:#x} size={size}")

class FreeBP(gdb.Breakpoint):
    def __init__(self):
        super().__init__("free", internal=True)
    
    def stop(self):
        ptr = int(gdb.parse_and_eval("$rdi"))
        if ptr in allocs:
            print(f"[FREE] {ptr:#x}")
            del allocs[ptr]
        elif ptr != 0:
            print(f"[DOUBLE FREE?] {ptr:#x}")
        return False

MallocBP()
FreeBP()
```

```python
# Script: Dump function arguments ทุกครั้งที่ call
import gdb

def dump_args(func_name):
    """Dump all arguments when function is called"""
    
    class ArgDumper(gdb.Breakpoint):
        def stop(self):
            frame = gdb.selected_frame()
            print(f"\n[*] {func_name} called:")
            
            # ดู register arguments (x86-64 calling convention)
            regs = ['rdi', 'rsi', 'rdx', 'rcx', 'r8', 'r9']
            for i, reg in enumerate(regs):
                val = int(gdb.parse_and_eval(f"${reg}"))
                print(f"  arg{i} ({reg}) = {val:#x} ({val})")
            
            # ลอง interpret เป็น string
            try:
                s = gdb.parse_and_eval("(char*)$rdi").string()
                print(f"  arg0 as string: {repr(s)}")
            except:
                pass
            
            return False
    
    ArgDumper(func_name)

dump_args("check_input")
dump_args("process_command")
```

### GDB Python API อื่น ๆ

```python
import gdb

# ดูข้อมูล current frame
frame = gdb.selected_frame()
print(f"Function: {frame.name()}")
print(f"PC: {frame.pc():#x}")
print(f"File: {frame.find_sal().symtab.filename}")
print(f"Line: {frame.find_sal().line}")

# Read memory
inferior = gdb.selected_inferior()
data = inferior.read_memory(0x401162, 16)
print(bytes(data).hex())

# Write memory
inferior.write_memory(0x601080, b'\x00' * 8)

# Thread info
for thread in gdb.inferiors()[0].threads():
    print(f"Thread {thread.num}: {thread.name}")

# Lookup symbol
sym, _ = gdb.lookup_symbol("main")
print(f"main @ {sym.value():#x}")

# Type system
t = gdb.lookup_type("int")
print(f"int size: {t.sizeof}")
```

---

## 13. PEDA Plugin {#peda-plugin}

PEDA (Python Exploit Development Assistance) เป็น GDB plugin ที่ยอดนิยมสำหรับ CTF

### การติดตั้ง PEDA

```bash
git clone https://github.com/longld/peda.git ~/peda
echo "source ~/peda/peda.py" >> ~/.gdbinit

# ตรวจสอบ
gdb -q
# จะเห็น PEDA banner
```

### pattern_create และ pattern_offset

```gdb
# สร้าง cyclic pattern เพื่อหา offset ของ buffer overflow
(gdb) pattern_create 200
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAA'

# ส่ง pattern เข้า program แล้วดู crash
(gdb) r
Enter input: AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAA

Program received signal SIGSEGV
# RIP = 0x6141414541414134 (ค่า pattern ที่ overwrite RIP)

# หา offset
(gdb) pattern_offset 0x6141414541414134
6141414541414134 found at offset: 72

# หรือใช้ RIP โดยตรง
(gdb) pattern_offset $rip
```

### checksec - ตรวจสอบ Security Features

```gdb
# ตรวจสอบ security protections
(gdb) checksec
CANARY    : ENABLED
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial

# ตีความ:
# CANARY    = Stack canary (ป้องกัน stack overflow)
# FORTIFY   = Source fortification
# NX        = No-Execute (ป้องกัน shellcode บน stack)
# PIE       = Position Independent Executable (ASLR)
# RELRO     = Relocation Read-Only (ป้องกันการแก้ GOT)
```

### rop - ROP Gadget Search

```gdb
# ค้นหา ROP gadgets
(gdb) rop
# แสดง gadgets ทั้งหมด

# ค้นหา gadget เฉพาะ
(gdb) rop --string "pop rdi"
(gdb) rop --string "ret"
(gdb) rop --string "pop rdi; ret"

# ค้นหาใน libc
(gdb) rop --string "pop rdi" libc

# ตัวอย่าง output:
# Gadget not found
# OR
# 0x0000000000401234 : pop rdi ; ret

# ค้นหา gadgets ที่มีประโยชน์
(gdb) rop --string "pop rdi; ret"    # set rdi = first arg
(gdb) rop --string "pop rsi; ret"    # set rsi = second arg
(gdb) rop --string "pop rdx; ret"    # set rdx = third arg
```

### คำสั่งอื่น ๆ ของ PEDA

```gdb
# แสดงข้อมูล crash (สวยงามกว่า default)
(gdb) context
# แสดง registers, code, stack พร้อมกัน

# Stack examination
(gdb) stack 20
# แสดง 20 entries บน stack

# ดู memory sections
(gdb) vmmap
# แสดง virtual memory map แบบสวยงาม

# Find string ใน memory
(gdb) find "flag"
(gdb) find 0x41414141

# Find address
(gdb) searchmem "CTF"
(gdb) searchmem 0x41414141

# Dump memory
(gdb) dumpmem output.bin 0x601080 0x6010a0
(gdb) hexdump 0x601080 64

# Shellcode generation
(gdb) shellcode generate x86/exec

# เข้า GDB Python shell
(gdb) python
```

---

## 14. PWNDBG Plugin {#pwndbg-plugin}

PWNDBG เป็น GDB plugin ที่ทรงพลังมาก เน้น heap analysis และ exploitation

### การติดตั้ง PWNDBG

```bash
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh

# หรือใช้ pip
pip install pwndbg
```

### heap - Heap Analysis

```gdb
# ดู heap chunks ทั้งหมด
(gdb) heap
# แสดง malloc chunks บน heap

# ดู allocated chunks
(gdb) heap --all

# ตัวอย่าง output:
# Allocated chunk | PREV_INUSE
# Addr: 0x602000
# Size: 0x21 (with flag bits: 0x21)
# 
# Allocated chunk | PREV_INUSE
# Addr: 0x602020
# Size: 0x31 (with flag bits: 0x31)
#
# Top chunk | PREV_INUSE
# Addr: 0x602050
# Size: 0x20fb1 (with flag bits: 0x20fb1)

# ดู specific heap
(gdb) heap 0x602000
```

### bins - Freelist Analysis

```gdb
# ดู bins ทั้งหมด (freelist)
(gdb) bins

# ดู tcache bins
(gdb) tcachebins
# ตัวอย่าง output:
# tcachebins
# 0x20 [  1]:  0x602260 ◂— 0x0

# ดู fastbins
(gdb) fastbins
# ตัวอย่าง:
# fastbins
# 0x20: 0x602260 ◂— 0x0

# ดู smallbins
(gdb) smallbins

# ดู largebins
(gdb) largebins

# ดู unsorted bin
(gdb) unsortedbin

# ดู main_arena
(gdb) main_arena
```

### procinfo - Process Information

```gdb
# ดูข้อมูล process แบบละเอียด
(gdb) procinfo
# แสดง:
# - PID
# - Exe path
# - Arguments
# - Memory maps
# - Open file descriptors

# ดู memory map
(gdb) vmmap
# ตัวอย่าง output:
# LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
#          0x400000           0x401000 r-xp     1000 0      /home/user/program
#          0x600000           0x601000 r--p     1000 0      /home/user/program
#          0x601000           0x602000 rw-p     1000 1000   /home/user/program
#          0x602000           0x623000 rw-p    21000 0      [heap]
#    0x7ffff7a0d000     0x7ffff7bcd000 r-xp   1c0000 0      /lib/.../libc.so.6
#    0x7ffffffde000     0x7ffffffff000 rw-p    21000 0      [stack]
```

### PWNDBG commands เพิ่มเติม

```gdb
# Context display (สวยงามมาก)
(gdb) context
# แสดง:
# - Registers พร้อม highlight ค่าที่เปลี่ยน
# - Disassembly รอบ RIP
# - Stack
# - Backtrace

# Find gadgets
(gdb) rop --grep "pop rdi"
(gdb) ropper --search "pop rdi; ret"

# Format string analysis
(gdb) fmtstr-payload 6 {0x601080: b'\x00\x00\x00\x00\x00\x00\x00\x00'}

# Canary value
(gdb) canary

# ค้นหา pattern
(gdb) cyclic 200
(gdb) cyclic -l 0x6161616e  # หา offset

# GOT table
(gdb) got
# แสดง GOT entries ทั้งหมด

# PLT
(gdb) plt
# แสดง PLT entries

# ดู ELF info
(gdb) elfheader
(gdb) elfsections

# Hexdump
(gdb) hexdump 0x601080 64
(gdb) hd 0x601080 64

# Search memory
(gdb) search -s "flag"
(gdb) search -x 0x41414141
(gdb) search -p 0x401162   # ค้นหา pointer

# Telescope - smart memory display
(gdb) telescope $rsp 20
# แสดง stack พร้อม pointer resolution
```

---

## 15. GEF Plugin {#gef-plugin}

GEF (GDB Enhanced Features) เป็น plugin ที่มีฟีเจอร์ครบครัน

### การติดตั้ง GEF

```bash
# ติดตั้งด้วย pip
pip3 install gef

# หรือ wget
wget -q https://gef.blah.cat/py -O ~/.gdbinit-gef.py
echo "source ~/.gdbinit-gef.py" >> ~/.gdbinit

# ตรวจสอบ
gdb -q
# จะเห็น GEF banner
```

### GEF Features หลัก

```gdb
# Context display (ดีมาก)
(gdb) context
# แสดงข้อมูล comprehensive: registers, code, stack, trace

# Memory view แบบ smart
(gdb) dereference $rsp 10
# Smart dereference ทุก pointer

# Heap analysis
(gdb) heap
(gdb) heap chunks
(gdb) heap bins
(gdb) heap arenas

# Format string helper
(gdb) format-string-helper
(gdb) fmtstr-helper

# ROP gadgets
(gdb) rop
(gdb) rop --grep "pop rdi"

# Assembly/Shellcode
(gdb) assemble "pop rdi; ret"
(gdb) shellcode get execve

# ค้นหา pattern
(gdb) pattern create 200
(gdb) pattern search 0x6161616e

# Memory search
(gdb) search-pattern "flag"
(gdb) search-pattern 0x41414141

# Process info
(gdb) process-search
(gdb) xinfo $rsp     # smart info เกี่ยวกับ address

# Checksec
(gdb) checksec

# GOT/PLT
(gdb) got
(gdb) plt

# Elf info
(gdb) elf-info

# Canary
(gdb) canary

# Entry point
(gdb) entry-break
(gdb) entrypoint

# บันทึก session
(gdb) gef-record start
(gdb) gef-record stop

# Unicorn emulation (ถ้ามี unicorn)
(gdb) unicorn-emulate
```

### GEF Configuration

```gdb
# ดู config ทั้งหมด
(gdb) gef config

# เปลี่ยน config
(gdb) gef config context.layout "legend regs code stack args source memory threads trace extra"
(gdb) gef config context.nb_lines_code 10
(gdb) gef config context.nb_lines_stack 8

# บันทึก config
(gdb) gef save

# โหลด config
(gdb) gef restore
```

---

## 16. Remote Debugging {#remote-debugging}

### gdbserver

gdbserver ช่วยให้ debug remotely ได้ - มีประโยชน์มากในการ debug บน embedded systems  
หรือ remote CTF servers

```bash
# บน target machine:
# รัน program ด้วย gdbserver
gdbserver :1234 ./program
gdbserver :1234 ./program arg1 arg2

# Attach กับ running process
gdbserver :1234 --attach 4521

# Listen บน specific host
gdbserver 0.0.0.0:1234 ./program

# Multi-process mode
gdbserver --multi :1234
```

```gdb
# บน debugging machine:
# Connect ไปยัง gdbserver
(gdb) target remote localhost:1234
(gdb) target remote 192.168.1.100:1234

# Extended remote (รองรับ process control)
(gdb) target extended-remote :1234

# ตรวจสอบ connection
(gdb) info target

# ใช้ remote commands
(gdb) remote put local_file remote_file  # upload file
(gdb) remote get remote_file local_file  # download file

# ตัวอย่าง workflow สมบูรณ์:
# Target:
$ gdbserver :4444 ./vulnerable_service

# Debugger:
$ gdb ./vulnerable_service
(gdb) target remote target_ip:4444
(gdb) b check_auth
(gdb) c
```

### QEMU + GDB

ใช้สำหรับ debug kernel หรือ embedded programs ที่รันบน QEMU

```bash
# รัน QEMU พร้อม gdb stub
qemu-system-x86_64 -kernel bzImage -s -S
# -s = เปิด gdbserver ที่ port 1234
# -S = หยุดรอ debugger ก่อนเริ่ม

# Debug ARM binary บน x86
qemu-arm -g 1234 ./arm_binary

# Debug MIPS binary
qemu-mips -g 1234 ./mips_binary

# Debug x86-64 binary (ต้อง static link)
qemu-x86_64 -g 1234 ./static_binary
```

```gdb
# Connect GDB ไปยัง QEMU
$ gdb vmlinux   # kernel symbols
(gdb) target remote :1234
(gdb) hbreak *0xffffffff81000000  # hardware breakpoint สำหรับ kernel
(gdb) c

# สำหรับ ARM debugging
$ gdb-multiarch ./arm_binary
(gdb) set architecture arm
(gdb) target remote :1234
```

### Remote Debugging ผ่าน SSH Tunnel

```bash
# สร้าง SSH tunnel
ssh -L 1234:localhost:1234 user@remote_server &

# บน remote server
gdbserver :1234 ./program

# บน local machine
gdb ./program
(gdb) target remote localhost:1234
```

---

## 17. CTF Techniques {#ctf-techniques}

### Buffer Overflow Analysis

```gdb
# Step 1: หา overflow offset ด้วย cyclic pattern
(gdb) pattern create 200   # PEDA/PWNDBG
(gdb) cyclic 200           # GEF
(gdb) r
Enter: <paste pattern>
Program received signal SIGSEGV
# ดูค่าใน RIP/RSP

# PEDA
(gdb) pattern offset $rip

# PWNDBG  
(gdb) cyclic -l $(p64 $rip).decode()

# Step 2: ตรวจสอบ protections
(gdb) checksec

# Step 3: ค้นหา ROP gadgets
(gdb) rop --string "pop rdi; ret"
(gdb) rop --string "pop rsi; pop r15; ret"
(gdb) rop --string "ret"

# Step 4: หา system() และ "/bin/sh"
(gdb) print system
(gdb) find "/bin/sh"   # GDB command
(gdb) searchmem "/bin/sh"   # PEDA
(gdb) search-pattern "/bin/sh"   # GEF
```

### Format String Analysis

```gdb
# Step 1: หา offset ของ format string
(gdb) b printf
(gdb) r
# ใส่ "%p %p %p %p %p %p %p %p" เป็น input

# Step 2: ดูว่า input อยู่ที่ offset เท่าไร
# ถ้า format string offset = 6:
# ใช้ "%6$p" จะอ่านค่าจาก stack position 6

# Step 3: หา GOT address ที่ต้องการ overwrite
(gdb) got                   # ดู GOT entries
(gdb) x/20xg 0x601018      # ดู GOT manually

# Step 4: คำนวณ write primitive
# %<value>c%<offset>$hn สำหรับ 2-byte write
# ใช้ pwntools ช่วยคำนวณ

# ตัวอย่าง: overwrite GOT entry ของ exit() ให้ชี้ไป win()
(gdb) p &exit@got.plt
$1 = 0x601040
(gdb) p win
$2 = {<text variable, no debug info>} 0x401162 <win>
```

### Heap Exploitation Analysis

```gdb
# ดู heap state
(gdb) heap             # PWNDBG
(gdb) heap chunks      # GEF

# ดู freelist
(gdb) bins             # PWNDBG
(gdb) heap bins        # GEF

# ตรวจสอบ tcache
(gdb) tcachebins

# ดู heap หลัง free
(gdb) heap
# หา freed chunk ดูว่า fd/bk ชี้ไปไหน

# Use-After-Free analysis
(gdb) b malloc
(gdb) b free
(gdb) commands 1
> printf "malloc(%ld) = %p\n", $rdi, $rax
> c
> end
(gdb) commands 2
> printf "free(%p)\n", $rdi
> c
> end

# Heap leak
(gdb) x/10xg <freed_chunk_addr>
# fd pointer ใน unsorted bin ชี้ไป main_arena (ใน libc)
# ใช้ offset นี้คำนวณ libc base
```

### Return Oriented Programming (ROP)

```gdb
# ค้นหา gadgets ต่าง ๆ
# x86-64 calling convention: rdi, rsi, rdx, rcx, r8, r9

# Gadget สำหรับ system("/bin/sh")
(gdb) rop --string "pop rdi; ret"      # load "/bin/sh" addr ไป rdi
(gdb) rop --string "ret"               # stack alignment

# ค้นหา one_gadget (libc)
$ one_gadget /lib/x86_64-linux-gnu/libc.so.6

# ใน GDB: ค้นหา constraints ของ one_gadget
(gdb) x/20i <one_gadget_address>

# ตรวจสอบ gadget chain
# ROP chain: pop rdi; ret -> "/bin/sh" addr -> system addr
(gdb) x/4xg $rsp  # ดู stack ก่อน return
# ควรเห็น:
# [rsp+0]  : addr ของ pop rdi; ret gadget
# [rsp+8]  : addr ของ "/bin/sh" string
# [rsp+16] : addr ของ system()
```

### ASLR Bypass

```gdb
# ปิด ASLR เพื่อ development
$ echo 0 > /proc/sys/kernel/randomize_va_space

# ใน GDB (ปิดแค่ session นี้)
(gdb) set disable-randomization on
(gdb) set disable-randomization off

# หา libc base จาก leak
(gdb) vmmap libc      # ดู libc base address
(gdb) p printf        # ดู address ของ printf
(gdb) p system        # ดู address ของ system

# คำนวณ offset
# libc_base = leaked_addr - offset_in_libc
(gdb) p &printf - <libc_base_address>
```

### Stack Canary Bypass

```gdb
# ดู canary value
(gdb) canary        # PWNDBG/GEF

# หา canary ใน stack
(gdb) x/20xg $rsp
# canary อยู่ก่อน saved rbp และ return address
# มักลงท้ายด้วย 0x00 (null byte)

# Format string leak canary
# %<stack_offset>$p -> ค่า canary
(gdb) p/x *(long*)($rbp-8)   # canary ก่อน rbp

# Brute force canary (ถ้า program fork)
# ทำได้เพราะ child มี canary เดิมจาก parent
```

---

## 18. Advanced Tips {#advanced-tips}

### GDB Settings ที่แนะนำ

```gdb
# ไฟล์ ~/.gdbinit
set disassembly-flavor intel
set pagination off
set print pretty on
set print array on
set print array-indexes on
set follow-fork-mode child
set confirm off
set history save on
set history size 10000
set history filename ~/.gdb_history

# Aliases ที่มีประโยชน์
define xxd
  if $argc == 2
    x/$arg1xb $arg2
  else
    x/16xb $arg0
  end
end

define stack
  x/20xg $rsp
end

define regs
  info registers
end
```

### Scripting และ Automation

```bash
# รัน GDB commands แบบ non-interactive
gdb -batch \
  -ex "file ./program" \
  -ex "break main" \
  -ex "run arg1" \
  -ex "info registers" \
  -ex "quit"

# Script file
cat > debug.gdb << 'EOF'
set pagination off
file ./program
break *0x401234
run < /tmp/input
info registers
x/20xg $rsp
quit
EOF
gdb -batch -x debug.gdb
```

### Attach to Docker Container

```bash
# หา PID ของ process ใน docker
docker ps
docker exec container_name ps aux | grep program
PID=$(docker exec container_name pidof program)

# Attach
sudo gdb -p $PID

# หรือ nsenter
nsenter -t $PID -p -m gdb -p $PID
```

### Debugging Stripped Binaries

```gdb
# Binary ไม่มี symbols - ใช้ addresses โดยตรง
(gdb) x/50i 0x401162   # entry point ประมาณ

# หา main จาก __libc_start_main
# โดยปกติ argument แรกของ __libc_start_main คือ main
(gdb) b _start
(gdb) r
(gdb) ni    # step จนถึง call __libc_start_main
(gdb) x/1xg $rsp   # argument แรก = main
# หรือดู rdi ก่อน call

# Ghidra/IDA integration
# นำ addresses จาก Ghidra มาใช้ใน GDB
(gdb) b *0x401234   # address จาก Ghidra (อาจต้อง +base_addr ถ้า PIE)

# PIE base address
(gdb) vmmap
# ดู base address ของ executable
(gdb) p 0x555555554000 + 0x1234   # base + ghidra_offset
```

### Multi-thread Debugging

```gdb
# ดู threads
(gdb) info threads

# เปลี่ยน thread
(gdb) thread 2

# All-stop vs Non-stop mode
(gdb) set non-stop on   # threads อื่นรันต่อ ขณะ debug หนึ่ง thread
(gdb) set non-stop off  # หยุดทุก thread เมื่อ hit breakpoint (default)

# Thread-specific breakpoint
(gdb) break func thread 2

# Scheduler locking
(gdb) set scheduler-locking on    # รันแค่ thread ปัจจุบัน
(gdb) set scheduler-locking step  # lock เฉพาะตอน step
(gdb) set scheduler-locking off   # default

# Apply command ทุก thread
(gdb) thread apply all bt
(gdb) thread apply all info locals
(gdb) thread apply 1 2 3 print $rax
```

### Signal Handling

```gdb
# จัดการ signals
(gdb) handle SIGPIPE nostop noprint nopass
(gdb) handle SIGUSR1 stop print pass
(gdb) handle SIGSEGV stop print nopass

# ส่ง signal ไปให้ program
(gdb) signal SIGUSR1
(gdb) signal 0   # ไม่ส่ง signal (resume จาก signal handler)

# ดูสถานะ signal handling
(gdb) info signals
```

### Memory Manipulation สำหรับ CTF

```gdb
# Patch binary ใน memory (bypass checks)
# เปลี่ยน jump instruction
(gdb) set {char[6]}0x401234 = {0x90, 0x90, 0x90, 0x90, 0x90, 0x90}  # NOP

# เปลี่ยน je เป็น jmp
# je = 0x74, jmp = 0xeb
(gdb) set {char}0x401234 = 0xeb

# เปลี่ยน return value
(gdb) b check_password
(gdb) commands 1
> set $rax = 1   # force return 1 (success)
> continue
> end

# เปลี่ยน flag variable
(gdb) b main
(gdb) commands 1
> set is_admin = 1
> continue
> end

# Skip function (return ทันที)
(gdb) b dangerous_function
(gdb) commands 1
> return 0    # return ทันที
> end
```

### Convenience Functions

```gdb
# Define custom commands
define mystack
  x/20xg $rsp
end

define myregs
  printf "RAX: %lx  RBX: %lx  RCX: %lx  RDX: %lx\n", $rax, $rbx, $rcx, $rdx
  printf "RSI: %lx  RDI: %lx  RBP: %lx  RSP: %lx\n", $rsi, $rdi, $rbp, $rsp
  printf "RIP: %lx  RFLAGS: %lx\n", $rip, $eflags
end

# ดู function จาก PLT
define gotshow
  info plt
end
```

---

## สรุป Command Quick Reference

### Startup
| Command | Description |
|---------|-------------|
| `gdb ./binary` | เปิด binary |
| `gdb -p PID` | Attach process |
| `gdb --args ./prog arg1` | เปิดพร้อม args |
| `gdb binary core` | เปิด core dump |
| `gdb -q` | Quiet mode |
| `gdb -x script.gdb` | รัน script |

### Execution
| Command | Description |
|---------|-------------|
| `r / run` | รัน program |
| `c / continue` | ดำเนินต่อ |
| `n / next` | Step over |
| `s / step` | Step into |
| `finish` | จบ function |
| `until N` | รันถึงบรรทัด N |
| `jump *addr` | กระโดดไป address |

### Breakpoints
| Command | Description |
|---------|-------------|
| `b main` | Break ที่ main |
| `b *0x401234` | Break ที่ address |
| `b func if cond` | Conditional break |
| `enable/disable N` | เปิด/ปิด breakpoint |
| `delete N` | ลบ breakpoint |
| `tbreak` | Temporary breakpoint |

### Inspection
| Command | Description |
|---------|-------------|
| `i r` | Registers |
| `i b` | Breakpoints |
| `i locals` | Local variables |
| `i args` | Arguments |
| `i threads` | Threads |
| `i proc map` | Memory map |

### Memory
| Command | Description |
|---------|-------------|
| `x/10xg addr` | 10 quadwords hex |
| `x/20i addr` | 20 instructions |
| `x/s addr` | String |
| `x/10xb addr` | 10 bytes hex |
| `p var` | Print variable |
| `p/x $rax` | Print register hex |
| `p &var` | Print address |

### Navigation
| Command | Description |
|---------|-------------|
| `bt / backtrace` | Call stack |
| `frame N / f N` | เลือก frame |
| `up / down` | เลื่อน frames |
| `info frame` | ข้อมูล frame ปัจจุบัน |

### TUI
| Command | Description |
|---------|-------------|
| `Ctrl+X A` | Toggle TUI |
| `layout src` | Source view |
| `layout asm` | Assembly view |
| `layout regs` | Registers view |
| `layout split` | Split view |

### PEDA/PWNDBG/GEF
| Command | Description |
|---------|-------------|
| `checksec` | Security info |
| `pattern create N` | Cyclic pattern |
| `pattern offset` | หา offset |
| `rop` | ROP gadgets |
| `heap` | Heap analysis |
| `bins` | Freelist |
| `vmmap` | Memory map |

---

## ตัวอย่าง CTF Workflow สมบูรณ์

### ตัวอย่าง: Stack Buffer Overflow

```bash
# Step 1: Reconnaissance
file ./vulnerable
checksec --file=./vulnerable
strings ./vulnerable | grep -E "flag|win|system"

# Step 2: Static Analysis
objdump -d ./vulnerable | grep -A 20 "<main>"

# Step 3: Dynamic Analysis ด้วย GDB
gdb -q ./vulnerable
```

```gdb
# Step 4: หา overflow
(gdb) pattern create 200
# copy pattern
(gdb) r
Enter: <paste pattern>
Program received signal SIGSEGV, Segmentation fault.
# ดู RIP

# Step 5: หา offset
(gdb) pattern offset $rip
# Output: 72

# Step 6: หา win function
(gdb) p win
$1 = {<text variable, no debug info>} 0x4011c6 <win>

# Step 7: ตรวจสอบ rop gadget ถ้าจำเป็น (เช่น stack alignment)
(gdb) rop --string "ret"
0x000000000040101a : ret

# Step 8: สร้าง exploit
# exit GDB แล้วเขียน exploit script
```

```python
# exploit.py
from pwn import *

elf = ELF('./vulnerable')
p = process('./vulnerable')

offset = 72
win = p64(elf.sym['win'])
ret = p64(0x40101a)  # ret gadget สำหรับ alignment

payload = b'A' * offset + ret + win
p.sendline(payload)
p.interactive()
```

### ตัวอย่าง: Format String Exploit

```gdb
# หา format string vulnerability
(gdb) b printf
(gdb) r
# ใส่ "%p %p %p %p %p %p %p %p"

# ดูว่า input อยู่ที่ offset ไหน
# ถ้าเราเห็นค่าจาก buffer ที่ stack position N
# ใช้ %N$p เพื่ออ่าน

# หา GOT entry ของ function ที่อยากแก้
(gdb) x/xg &exit@got.plt
0x601040:   0x00007ffff7a629d5

# ใช้ format string เพื่อ overwrite
# %<low_16bit>c%<offset>$hn  # write lower 16 bits
# %<high_16bit>c%<offset+1>$hn  # write upper 16 bits
```

---

## เคล็ดลับสุดท้าย

1. **เริ่มต้นด้วย checksec** - รู้ว่า protections อะไรมีบ้างก่อน
2. **ใช้ `set pagination off`** - ป้องกัน GDB หยุดรอ input
3. **ใช้ `set disassembly-flavor intel`** - อ่านง่ายกว่า AT&T
4. **บันทึก GDB session** - `(gdb) set logging on`
5. **ใช้ Python scripts** - automation ช่วยได้มาก
6. **เรียน pwntools** - ควบคู่กับ GDB
7. **วิเคราะห์ใน static ก่อน** - Ghidra/IDA ช่วยลด time

---

*ส่วนนี้เป็นเนื้อหาการศึกษาด้านความปลอดภัย (Security Education) และการแข่งขัน CTF (Capture The Flag) เท่านั้น*  
*ความรู้นี้ควรนำไปใช้อย่างถูกต้องตามกฎหมาย เช่น การ debug โปรแกรมตัวเอง, การเรียนรู้ด้าน security, และการแข่งขัน CTF*
