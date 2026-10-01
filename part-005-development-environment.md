# Part 005: การตั้งค่า Development Environment

**เวลาที่ใช้เรียน:** 3-4 ชั่วโมง  
**ระดับ:** พื้นฐาน (Beginner)  
**หัวข้อก่อนหน้า:** Part 004 - Architecture Overview  
**หัวข้อถัดไป:** Part 006 - Registers และ Memory Model

---

## วัตถุประสงค์การเรียนรู้

หลังจากเรียนจบ Part นี้แล้ว ผู้เรียนจะสามารถ:

- ติดตั้งและกำหนดค่า NASM, GAS, และ MASM ได้อย่างถูกต้อง
- ตั้งค่า ARM cross-compiler toolchain สำหรับ x86 และ AArch64
- ติดตั้งและใช้งาน GDB debugger เบื้องต้นได้
- ใช้งาน QEMU สำหรับจำลอง ARM environment
- ตั้งค่า VS Code สำหรับการพัฒนา Assembly
- เขียน Makefile สำหรับ Assembly projects ได้
- Compile และ Link โปรแกรม Assembly ได้ทั้ง x86 และ ARM
- รัน Hello World บน Linux 32-bit, 64-bit, และ ARM emulator ได้

---

## สารบัญ

1. [ทำไมต้องตั้งค่า Development Environment ให้ดี?](#section-1)
2. [การติดตั้ง NASM บน Linux/Windows/macOS](#section-2)
3. [การติดตั้ง GAS (GNU Assembler)](#section-3)
4. [การติดตั้ง MASM บน Windows](#section-4)
5. [ARM Cross-Compiler Toolchain](#section-5)
6. [การติดตั้งและใช้งาน GDB](#section-6)
7. [การใช้งาน QEMU สำหรับ ARM Emulation](#section-7)
8. [VS Code Extensions สำหรับ Assembly](#section-8)
9. [Makefile สำหรับ Assembly Projects](#section-9)
10. [การ Compile และ Link](#section-10)
11. [Hello World โปรแกรมแรก](#section-11)
12. [ข้อผิดพลาดที่พบบ่อย](#section-12)
13. [แบบฝึกหัด](#section-13)
14. [สรุปและ Key Takeaways](#section-14)
15. [แหล่งอ้างอิง](#section-15)

---

<a name="section-1"></a>
## 1. ทำไมต้องตั้งค่า Development Environment ให้ดี?

การมี Development Environment ที่ดีเป็นรากฐานสำคัญของการพัฒนาโปรแกรม Assembly อย่างมีประสิทธิภาพ ต่างจากภาษา High-Level อย่าง Python หรือ JavaScript ที่มี IDE ครบครัน การพัฒนา Assembly ต้องการเครื่องมือหลายอย่างประกอบกัน

### เครื่องมือที่จำเป็น

| เครื่องมือ | หน้าที่ | Platform |
|-----------|---------|----------|
| NASM | Assembler สำหรับ x86/x86-64 | Linux, Windows, macOS |
| GAS (as) | GNU Assembler สำหรับ AT&T syntax | ทุก Platform |
| MASM | Assembler ของ Microsoft | Windows เท่านั้น |
| GDB | Debugger สำหรับ debug โปรแกรม | Linux, macOS |
| ld | Linker สำหรับสร้าง executable | Linux, macOS |
| QEMU | Emulator สำหรับ ARM | ทุก Platform |
| Make | Build automation tool | ทุก Platform |

### Workflow การพัฒนา Assembly

```
Source Code (.asm/.s) 
        ↓
   Assembler (NASM/GAS)
        ↓
Object File (.o)
        ↓
    Linker (ld)
        ↓
  Executable (ELF/PE)
        ↓
    Program runs!
```

---

<a name="section-2"></a>
## 2. การติดตั้ง NASM (Netwide Assembler)

NASM เป็น Assembler ที่นิยมใช้มากที่สุดสำหรับ x86/x86-64 รองรับ Intel syntax ซึ่งอ่านง่ายกว่า AT&T syntax ของ GAS

### 2.1 การติดตั้งบน Linux (Ubuntu/Debian)

```bash
# อัปเดต package list ก่อน
sudo apt update

# ติดตั้ง NASM
sudo apt install -y nasm

# ตรวจสอบการติดตั้ง
nasm --version
# ผลลัพธ์ที่คาดหวัง: NASM version 2.15.05 compiled on ...

# ดูความช่วยเหลือ
nasm --help
```

```bash
# สำหรับ Fedora/RHEL/CentOS
sudo dnf install nasm

# หรือสำหรับ Arch Linux
sudo pacman -S nasm

# สำหรับ openSUSE
sudo zypper install nasm
```

### 2.2 การติดตั้งบน macOS

```bash
# ใช้ Homebrew (แนะนำ)
# ติดตั้ง Homebrew ก่อนถ้ายังไม่มี
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง NASM
brew install nasm

# ตรวจสอบการติดตั้ง
nasm --version
# ผลลัพธ์: NASM version 2.16.xx

# หมายเหตุ: macOS มี NASM ที่ล้าสมัยติดมาด้วย ต้องใช้ brew install เพื่อได้เวอร์ชันใหม่
which nasm
# ควรแสดง: /usr/local/bin/nasm (Intel) หรือ /opt/homebrew/bin/nasm (Apple Silicon)
```

### 2.3 การติดตั้งบน Windows

#### วิธีที่ 1: ติดตั้งจาก Official Installer

```
1. ไปที่ https://www.nasm.us/pub/nasm/releasebuilds/
2. ดาวน์โหลดไฟล์ nasm-2.xx.xx-installer-x64.exe
3. รัน installer และทำตาม wizard
4. เพิ่ม path ใน System Environment Variables:
   - C:\Program Files\NASM
5. เปิด Command Prompt ใหม่แล้วพิมพ์:
   nasm --version
```

#### วิธีที่ 2: ใช้ Chocolatey (Package Manager)

```powershell
# เปิด PowerShell ด้วยสิทธิ์ Administrator
# ติดตั้ง Chocolatey ก่อน
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง NASM
choco install nasm

# ตรวจสอบ
nasm --version
```

#### วิธีที่ 3: ใช้ WSL2 (Windows Subsystem for Linux) - แนะนำสำหรับ Development

```powershell
# เปิด PowerShell ด้วยสิทธิ์ Administrator
wsl --install

# รีสตาร์ทคอมพิวเตอร์แล้วตั้งค่า Ubuntu
# จากนั้นใน WSL terminal:
sudo apt update && sudo apt install nasm
```

### 2.4 ทดสอบ NASM

สร้างไฟล์ทดสอบ `test.asm`:

```nasm
; test.asm - ไฟล์ทดสอบ NASM เบื้องต้น
; คอมไพล์: nasm -f elf64 test.asm -o test.o

section .data
    msg db "NASM works!", 0Ah    ; ข้อความ + newline character

section .text
    global _start

_start:
    ; เรียก sys_write (system call หมายเลข 1)
    mov rax, 1          ; syscall number: sys_write
    mov rdi, 1          ; file descriptor: stdout (1)
    mov rsi, msg        ; pointer ไปที่ข้อความ
    mov rdx, 12         ; จำนวน bytes ที่จะเขียน
    syscall             ; เรียก kernel

    ; เรียก sys_exit (system call หมายเลข 60)
    mov rax, 60         ; syscall number: sys_exit
    xor rdi, rdi        ; exit code: 0 (success)
    syscall             ; เรียก kernel
```

```bash
# คอมไพล์และรัน
nasm -f elf64 test.asm -o test.o
ld test.o -o test
./test
# ผลลัพธ์: NASM works!
```

---

<a name="section-3"></a>
## 3. การติดตั้ง GAS (GNU Assembler) - binutils

GAS เป็นส่วนหนึ่งของ GNU binutils package ใช้ AT&T syntax เป็นค่าเริ่มต้น มักใช้กับ GCC compiler

### 3.1 การติดตั้งบน Linux

```bash
# ติดตั้ง binutils (มี GAS อยู่ใน package นี้)
sudo apt install -y binutils

# ตรวจสอบว่า as (assembler) พร้อมใช้งาน
as --version
# ผลลัพธ์: GNU assembler (GNU Binutils for Ubuntu) 2.38

# ดูเครื่องมือใน binutils
ls /usr/bin/ | grep -E "^(ar|as|ld|nm|objdump|objcopy|readelf|strip)"
```

### 3.2 ติดตั้ง build-essential (สำหรับ Linux)

```bash
# ติดตั้ง build-essential ซึ่งรวม gcc, g++, make, binutils
sudo apt install -y build-essential

# ตรวจสอบ
gcc --version
make --version
ld --version
```

### 3.3 ตัวอย่างโค้ด GAS (AT&T Syntax)

```gas
# hello_gas.s - ตัวอย่าง GAS (AT&T syntax)
# คอมไพล์: as -o hello_gas.o hello_gas.s && ld -o hello_gas hello_gas.o

    .section .data
msg:
    .ascii "Hello from GAS!\n"   # ข้อความ (ไม่มี null terminator)
msg_len = . - msg                # คำนวณความยาวข้อความ

    .section .text
    .global _start

_start:
    # AT&T syntax: source อยู่ซ้าย, destination อยู่ขวา
    # ตรงข้ามกับ NASM/Intel syntax
    
    movl $1, %eax       # syscall: sys_write (ใช้ % นำหน้า register)
    movl $1, %ebx       # file descriptor: stdout
    movl $msg, %ecx     # pointer ไปที่ข้อความ
    movl $msg_len, %edx # ความยาวข้อความ
    int $0x80           # interrupt 80h (32-bit Linux syscall)

    movl $1, %eax       # syscall: sys_exit
    xorl %ebx, %ebx     # exit code: 0
    int $0x80
```

```bash
# คอมไพล์ GAS 32-bit
as --32 -o hello_gas.o hello_gas.s
ld -m elf_i386 -o hello_gas hello_gas.o
./hello_gas
```

### 3.4 ความแตกต่างระหว่าง NASM และ GAS Syntax

| คุณสมบัติ | NASM (Intel) | GAS (AT&T) |
|----------|-------------|-----------|
| Operand order | `dest, src` | `src, dest` |
| Register prefix | ไม่มี | `%` (เช่น `%eax`) |
| Immediate prefix | ไม่มี | `$` (เช่น `$42`) |
| Memory access | `[ebx]` | `(%ebx)` |
| Size suffix | ไม่จำเป็น | `movl`, `movb`, `movw` |
| Comment | `;` | `#` |
| Section | `section .data` | `.section .data` |

---

<a name="section-4"></a>
## 4. การติดตั้ง MASM (Microsoft Macro Assembler) บน Windows

MASM เป็น Assembler ของ Microsoft รวมอยู่ใน Visual Studio

### 4.1 ติดตั้งผ่าน Visual Studio

```
1. ดาวน์โหลด Visual Studio Community (ฟรี) จาก https://visualstudio.microsoft.com/
2. ระหว่าง installation เลือก:
   - "Desktop development with C++"
   - หรือ Individual components: "MSVC v143 - VS 2022 C++ x64/x86 build tools"
3. MASM (ml64.exe) จะถูกติดตั้งอัตโนมัติ
4. เปิด "Developer Command Prompt for VS 2022"
5. พิมพ์: ml64 /?
```

### 4.2 ติดตั้ง MASM แบบ Standalone

```powershell
# ดาวน์โหลด Build Tools for Visual Studio
# จาก https://visualstudio.microsoft.com/downloads/
# เลือก "Build Tools for Visual Studio 2022"

# หลังติดตั้ง ใช้ Visual Studio Developer Command Prompt
# หรือเพิ่ม PATH:
# C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.xx.xxxx\bin\Hostx64\x64
```

### 4.3 ตัวอย่างโค้ด MASM

```masm
; hello_masm.asm - ตัวอย่าง MASM สำหรับ Windows
; คอมไพล์: ml64 /c hello_masm.asm
; Link: link hello_masm.obj /entry:main /subsystem:console kernel32.lib

; หรือใช้ Visual Studio

ExitProcess PROTO

.data
    msg db "Hello from MASM!", 0Dh, 0Ah, 0   ; 0Dh = CR, 0Ah = LF

.code
main PROC
    ; Windows ใช้ Win32 API ไม่ใช่ Linux syscalls
    ; ต้อง call ExitProcess จาก kernel32.dll
    
    sub rsp, 28h            ; จอง shadow space ตาม Windows ABI
    
    ; แสดงข้อความ (ต้องใช้ WriteConsole หรือ printf จาก C runtime)
    
    xor ecx, ecx            ; exit code = 0
    call ExitProcess        ; จบโปรแกรม
main ENDP

END
```

---

<a name="section-5"></a>
## 5. ARM Cross-Compiler Toolchain

การพัฒนาโปรแกรม ARM บนเครื่อง x86 ต้องใช้ Cross-Compiler ที่สามารถสร้าง ARM binary จากเครื่อง x86

### 5.1 ความเข้าใจเรื่อง Cross-Compilation

```
เครื่อง Host (x86-64)           Target Platform (ARM)
┌─────────────────────┐        ┌──────────────────────┐
│   Source Code .s    │        │   Raspberry Pi       │
│         ↓           │        │   Android Device     │
│  arm-linux-gnueabi  │──────→ │   Embedded System    │
│   cross-compiler    │ binary │   QEMU emulator      │
│         ↓           │        └──────────────────────┘
│   ARM binary        │
└─────────────────────┘
```

### 5.2 ติดตั้ง ARM 32-bit Toolchain (arm-linux-gnueabi)

```bash
# สำหรับ Ubuntu/Debian
sudo apt update
sudo apt install -y gcc-arm-linux-gnueabi
sudo apt install -y gcc-arm-linux-gnueabihf    # สำหรับ hard-float ABI

# ตรวจสอบการติดตั้ง
arm-linux-gnueabi-gcc --version
# ผลลัพธ์: arm-linux-gnueabi-gcc (Ubuntu ...) 11.x.x

# ดูเครื่องมือที่มีทั้งหมด
ls /usr/bin/arm-linux-gnueabi-*
```

```bash
# เครื่องมือที่ได้มา:
# arm-linux-gnueabi-gcc     - C compiler สำหรับ ARM 32-bit
# arm-linux-gnueabi-as      - Assembler สำหรับ ARM 32-bit
# arm-linux-gnueabi-ld      - Linker สำหรับ ARM 32-bit
# arm-linux-gnueabi-objdump - Object file analyzer
# arm-linux-gnueabi-gdb     - Debugger สำหรับ ARM
# arm-linux-gnueabi-strip   - Strip symbols จาก binary
```

### 5.3 ติดตั้ง AArch64 Toolchain (aarch64-linux-gnu)

```bash
# ติดตั้ง AArch64 (ARM 64-bit) toolchain
sudo apt install -y gcc-aarch64-linux-gnu
sudo apt install -y binutils-aarch64-linux-gnu

# ตรวจสอบ
aarch64-linux-gnu-gcc --version
# ผลลัพธ์: aarch64-linux-gnu-gcc (Ubuntu ...) 11.x.x

# ดูเครื่องมือ
ls /usr/bin/aarch64-linux-gnu-*
```

### 5.4 ทดสอบ ARM Cross-Compilation

```gas
# arm_test.s - ทดสอบ ARM 32-bit Assembly
# คอมไพล์: arm-linux-gnueabi-as -o arm_test.o arm_test.s
# Link: arm-linux-gnueabi-ld -o arm_test arm_test.o
# รัน: qemu-arm ./arm_test

    .text
    .global _start

_start:
    @ ARM ใช้ @ สำหรับ comment (แทน ; ของ x86)
    
    @ sys_write (syscall หมายเลข 4 สำหรับ ARM 32-bit)
    mov r7, #4          @ r7 คือ syscall number register ใน ARM Linux
    mov r0, #1          @ file descriptor: stdout
    ldr r1, =msg        @ pointer ไปที่ข้อความ
    mov r2, #14         @ ความยาวข้อความ
    swi 0               @ software interrupt (เรียก syscall)

    @ sys_exit (syscall หมายเลข 1)
    mov r7, #1          @ syscall: exit
    mov r0, #0          @ exit code: 0
    swi 0               @ เรียก syscall

    .data
msg:
    .ascii "ARM 32-bit OK!\n"   @ ข้อความทดสอบ
```

```gas
# aarch64_test.s - ทดสอบ AArch64 Assembly
# คอมไพล์: aarch64-linux-gnu-as -o aarch64_test.o aarch64_test.s
# Link: aarch64-linux-gnu-ld -o aarch64_test aarch64_test.o
# รัน: qemu-aarch64 ./aarch64_test

    .text
    .global _start

_start:
    // AArch64 ใช้ // สำหรับ comment (หรือ /* */ แบบ C)
    
    // sys_write (syscall หมายเลข 64 สำหรับ AArch64 Linux)
    mov x8, #64         // x8 คือ syscall number register
    mov x0, #1          // file descriptor: stdout
    ldr x1, =msg        // pointer ไปที่ข้อความ
    mov x2, #17         // ความยาวข้อความ
    svc #0              // supervisor call (เรียก syscall)

    // sys_exit (syscall หมายเลข 93)
    mov x8, #93         // syscall: exit
    mov x0, #0          // exit code: 0
    svc #0

    .data
msg:
    .ascii "AArch64 64-bit OK!\n"
```

---

<a name="section-6"></a>
## 6. การติดตั้งและใช้งาน GDB Debugger

GDB (GNU Debugger) เป็นเครื่องมือสำคัญสำหรับ debug โปรแกรม Assembly ช่วยให้เห็นสถานะของ registers, memory และการทำงานแต่ละ instruction

### 6.1 การติดตั้ง GDB

```bash
# Ubuntu/Debian
sudo apt install -y gdb

# Fedora/RHEL
sudo dnf install gdb

# macOS
brew install gdb
# หมายเหตุ: ต้อง code-sign GDB บน macOS (ดูขั้นตอนเพิ่มเติม)

# ตรวจสอบ
gdb --version
# ผลลัพธ์: GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1
```

### 6.2 ติดตั้ง GDB สำหรับ ARM debugging

```bash
# GDB ที่รองรับ ARM (multi-arch)
sudo apt install -y gdb-multiarch

# ตรวจสอบ
gdb-multiarch --version

# สำหรับ debug ARM binary บน x86 ต้องใช้ QEMU + GDB server
# (ดูรายละเอียดในส่วน QEMU)
```

### 6.3 การใช้งาน GDB เบื้องต้น

```bash
# คอมไพล์ด้วย debug symbols (-g flag)
nasm -f elf64 -g -F dwarf hello.asm -o hello.o
ld hello.o -o hello

# เริ่ม GDB
gdb ./hello
```

```gdb
# คำสั่ง GDB ที่ใช้บ่อยสำหรับ Assembly

# แสดง layout สำหรับ Assembly debugging
(gdb) layout asm          # แสดง Assembly code
(gdb) layout regs         # แสดง registers
(gdb) layout src          # แสดง source code (ถ้ามี)

# กำหนด breakpoint
(gdb) break _start        # break ที่ label _start
(gdb) break *0x401000     # break ที่ address
(gdb) info breakpoints    # แสดง breakpoints ทั้งหมด

# เริ่มรัน
(gdb) run                 # รันโปรแกรม
(gdb) start               # รันแล้ว break ที่ main

# ควบคุมการทำงาน
(gdb) stepi               # ทำงานทีละ instruction (si)
(gdb) nexti               # ทำงานทีละ instruction (ไม่เข้า function call)
(gdb) continue            # รันต่อจนถึง breakpoint (c)
(gdb) finish              # รันจนจบ function ปัจจุบัน

# ดู registers
(gdb) info registers      # แสดง registers ทั้งหมด
(gdb) print $rax          # แสดงค่าใน rax
(gdb) print /x $rax       # แสดงในรูป hexadecimal
(gdb) print /d $rax       # แสดงในรูป decimal

# ดู memory
(gdb) x/10x $rsp          # แสดง memory จาก rsp 10 ค่า (hex)
(gdb) x/s msg             # แสดง string จาก address msg
(gdb) x/10i $rip          # แสดง instructions 10 ตัวจาก rip

# ออกจาก GDB
(gdb) quit                # ออก (q)
```

### 6.4 Script GDB สำหรับ Assembly Debugging

สร้างไฟล์ `.gdbinit` ในโฟลเดอร์ project:

```gdb
# .gdbinit - GDB initialization script สำหรับ Assembly debugging

# แสดง registers อัตโนมัติหลังแต่ละคำสั่ง
define hook-stepi
  info registers rax rbx rcx rdx rsi rdi rsp rbp rip
end

# shortcut สำหรับดู stack
define stack
  x/16x $rsp
end

# shortcut สำหรับดู registers หลัก
define regs
  printf "RAX: 0x%016lx  RBX: 0x%016lx\n", $rax, $rbx
  printf "RCX: 0x%016lx  RDX: 0x%016lx\n", $rcx, $rdx
  printf "RSP: 0x%016lx  RBP: 0x%016lx\n", $rsp, $rbp
  printf "RIP: 0x%016lx  RFL: 0x%016lx\n", $rip, $eflags
end

# ตั้งค่า Intel syntax (อ่านง่ายกว่า AT&T สำหรับ NASM user)
set disassembly-flavor intel

# แสดงจำนวน instructions มากขึ้นใน TUI
set height 50
```

---

<a name="section-7"></a>
## 7. การใช้งาน QEMU สำหรับ ARM Emulation

QEMU (Quick Emulator) ช่วยให้เราสามารถรัน ARM binary บนเครื่อง x86 ได้โดยไม่ต้องมี hardware ARM จริง

### 7.1 การติดตั้ง QEMU

```bash
# ติดตั้ง QEMU user-mode emulation (สำหรับรัน ARM binary บน Linux)
sudo apt install -y qemu-user qemu-user-static

# ติดตั้ง QEMU system emulation (สำหรับ emulate ทั้ง OS)
sudo apt install -y qemu-system-arm qemu-system-aarch64

# ตรวจสอบ
qemu-arm --version
qemu-aarch64 --version
# ผลลัพธ์: qemu-arm version 6.2.0 (Debian ...)
```

### 7.2 รัน ARM Binary ด้วย QEMU User Mode

```bash
# ขั้นตอนที่ 1: เขียน ARM Assembly
cat > hello_arm.s << 'EOF'
    .text
    .global _start

_start:
    mov r7, #4          @ sys_write
    mov r0, #1          @ stdout
    ldr r1, =msg
    mov r2, #18
    swi 0

    mov r7, #1          @ sys_exit
    mov r0, #0
    swi 0

    .data
msg:
    .ascii "Hello ARM World!\n\0"
EOF

# ขั้นตอนที่ 2: Assemble
arm-linux-gnueabi-as -o hello_arm.o hello_arm.s

# ขั้นตอนที่ 3: Link
arm-linux-gnueabi-ld -o hello_arm hello_arm.o

# ขั้นตอนที่ 4: รันด้วย QEMU
qemu-arm ./hello_arm
# ผลลัพธ์: Hello ARM World!

# ดูไฟล์ binary
file hello_arm
# ผลลัพธ์: hello_arm: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV)...
```

### 7.3 รัน AArch64 Binary

```bash
# สร้าง AArch64 binary
cat > hello_arm64.s << 'EOF'
    .text
    .global _start

_start:
    mov x8, #64         // sys_write
    mov x0, #1          // stdout
    ldr x1, =msg
    mov x2, #20
    svc #0

    mov x8, #93         // sys_exit
    mov x0, #0
    svc #0

    .data
msg:
    .ascii "Hello AArch64 World!\n"
EOF

aarch64-linux-gnu-as -o hello_arm64.o hello_arm64.s
aarch64-linux-gnu-ld -o hello_arm64 hello_arm64.o
qemu-aarch64 ./hello_arm64
# ผลลัพธ์: Hello AArch64 World!
```

### 7.4 Debug ARM Binary ด้วย QEMU + GDB

```bash
# Terminal 1: รัน QEMU ใน debug mode (-g ระบุ port สำหรับ GDB)
qemu-arm -g 1234 ./hello_arm

# Terminal 2: เชื่อมต่อ GDB
gdb-multiarch hello_arm
```

```gdb
# ใน GDB
(gdb) set architecture arm     # กำหนด architecture
(gdb) target remote :1234      # เชื่อมต่อกับ QEMU GDB server
(gdb) break _start             # กำหนด breakpoint
(gdb) continue                 # รันจนถึง breakpoint
(gdb) layout asm               # แสดง ARM assembly
(gdb) stepi                    # ทำงานทีละ instruction
(gdb) info registers           # ดู ARM registers
```

### 7.5 QEMU System Mode (Emulate ARM Board)

```bash
# ดาวน์โหลด Raspberry Pi OS image (สำหรับ development ขั้นสูง)
# https://www.raspberrypi.org/downloads/

# ตัวอย่าง run Raspberry Pi OS บน QEMU
qemu-system-arm \
    -M versatilepb \
    -cpu arm1176 \
    -m 256 \
    -kernel kernel.img \
    -dtb versatile-pb.dtb \
    -drive format=raw,file=rootfs.img \
    -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw" \
    -serial stdio \
    -net nic -net user
```

---

<a name="section-8"></a>
## 8. VS Code Extensions สำหรับ Assembly

VS Code เป็น Editor ที่นิยมใช้กับ Assembly เพราะมี Extensions ที่ดี

### 8.1 Extensions ที่แนะนำ

```json
// ติดตั้งผ่าน VS Code Extensions marketplace
// หรือใช้ command line:

// 1. ASM Code Lens - แสดง references และ hover info
code --install-extension maziac.asm-code-lens

// 2. x86 and x86_64 Assembly - syntax highlighting สำหรับ NASM/GAS
code --install-extension 13xforever.language-x86-64-assembly

// 3. ARM Assembly - syntax highlighting สำหรับ ARM
code --install-extension dan-c-underwood.arm

// 4. Native Debug - GDB integration ใน VS Code
code --install-extension webfreak.debug

// 5. Hex Editor - ดู binary files
code --install-extension ms-vscode.hexeditor

// 6. Better Comments - color-coded comments
code --install-extension aaron-bond.better-comments
```

### 8.2 VS Code settings.json สำหรับ Assembly

สร้างไฟล์ `.vscode/settings.json` ในโฟลเดอร์ project:

```json
{
    // ตั้งค่า file associations
    "files.associations": {
        "*.asm": "asm-intel-x86-generic",
        "*.s": "asm-intel-x86-generic",
        "*.S": "asm-intel-x86-generic",
        "*.inc": "asm-intel-x86-generic"
    },
    
    // ตั้งค่า tab สำหรับ Assembly (4 spaces)
    "[asm-intel-x86-generic]": {
        "editor.tabSize": 4,
        "editor.insertSpaces": true,
        "editor.detectIndentation": false
    },
    
    // แสดง line numbers
    "editor.lineNumbers": "on",
    
    // highlight ตัวอักษรปัจจุบัน
    "editor.wordSeparators": "`~!@#$%^&*()-=+[{]}\\|;:'\",.<>/?",
    
    // ตั้งค่า terminal
    "terminal.integrated.defaultProfile.linux": "bash"
}
```

### 8.3 VS Code launch.json สำหรับ GDB Debug

สร้างไฟล์ `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug x86-64 Assembly",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "ใช้ Intel syntax",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                },
                {
                    "description": "แสดง TUI layout",
                    "text": "layout asm",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "build"
        },
        {
            "name": "Debug ARM32 with QEMU",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/${fileBasenameNoExtension}_arm",
            "miDebuggerPath": "/usr/bin/gdb-multiarch",
            "miDebuggerServerAddress": "localhost:1234",
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "text": "set architecture arm",
                    "ignoreFailures": false
                }
            ]
        }
    ]
}
```

### 8.4 VS Code tasks.json สำหรับ Build

สร้างไฟล์ `.vscode/tasks.json`:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "command": "make",
            "args": ["${fileBasenameNoExtension}"],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            },
            "problemMatcher": {
                "owner": "nasm",
                "fileLocation": ["relative", "${workspaceFolder}"],
                "pattern": {
                    "regexp": "^(.*):(\\d+):\\s+error:\\s+(.*)$",
                    "file": 1,
                    "line": 2,
                    "message": 3
                }
            }
        },
        {
            "label": "clean",
            "type": "shell",
            "command": "make clean",
            "group": "build"
        },
        {
            "label": "build ARM",
            "type": "shell",
            "command": "make",
            "args": ["${fileBasenameNoExtension}_arm"],
            "group": "build"
        }
    ]
}
```

---

<a name="section-9"></a>
## 9. Makefile สำหรับ Assembly Projects

Makefile ช่วยทำให้กระบวนการ compile อัตโนมัติ ไม่ต้องพิมพ์คำสั่งยาวๆ ซ้ำๆ

### 9.1 Makefile พื้นฐาน

```makefile
# Makefile พื้นฐาน สำหรับ x86-64 Assembly
# วิธีใช้: make <target>

# ตัวแปรหลัก
ASM     = nasm                    # assembler
ASFLAGS = -f elf64                # flags: ELF 64-bit format
LD      = ld                      # linker
LDFLAGS =                         # linker flags (ว่างไว้)

# สีสำหรับ output (optional)
GREEN  = \033[0;32m
YELLOW = \033[0;33m
RED    = \033[0;31m
NC     = \033[0m  # No Color

# หา .asm files ทั้งหมดโดยอัตโนมัติ
SRCS    = $(wildcard *.asm)
OBJS    = $(SRCS:.asm=.o)
TARGETS = $(SRCS:.asm=)

# Default target: build ทุกอย่าง
.PHONY: all clean help

all: $(TARGETS)
	@echo "$(GREEN)Build complete!$(NC)"

# Rule: .asm → .o (Assemble)
%.o: %.asm
	@echo "$(YELLOW)Assembling $<...$(NC)"
	$(ASM) $(ASFLAGS) $< -o $@

# Rule: .o → executable (Link)
%: %.o
	@echo "$(YELLOW)Linking $@...$(NC)"
	$(LD) $(LDFLAGS) $< -o $@

# Clean: ลบ object files และ executables
clean:
	@echo "$(RED)Cleaning...$(NC)"
	rm -f $(OBJS) $(TARGETS)

# Help: แสดงวิธีใช้
help:
	@echo "Usage: make [target]"
	@echo ""
	@echo "Targets:"
	@echo "  all       - Build all .asm files (default)"
	@echo "  clean     - Remove built files"
	@echo "  <name>    - Build specific file (e.g., make hello)"
	@echo "  help      - Show this help"
```

### 9.2 Makefile แบบขยาย (สำหรับ Project ใหญ่)

```makefile
# Makefile แบบ advanced สำหรับ Assembly project
# รองรับทั้ง x86, x86-64, ARM32, และ AArch64

# ===== คอนฟิกหลัก =====
PROJECT = myproject
VERSION = 1.0.0

# ===== x86-64 Tools =====
NASM        = nasm
NASM_FLAGS  = -f elf64 -g -F dwarf   # รวม debug info
LD          = ld
LD_FLAGS    = -m elf_x86_64

# ===== x86 32-bit Tools =====
NASM32_FLAGS = -f elf32
LD32_FLAGS   = -m elf_i386

# ===== ARM 32-bit Tools =====
ARM_AS      = arm-linux-gnueabi-as
ARM_LD      = arm-linux-gnueabi-ld
ARM_FLAGS   = -march=armv7-a

# ===== AArch64 Tools =====
ARM64_AS    = aarch64-linux-gnu-as
ARM64_LD    = aarch64-linux-gnu-ld

# ===== QEMU =====
QEMU_ARM    = qemu-arm
QEMU_ARM64  = qemu-aarch64

# ===== Directories =====
SRC_DIR     = src
OBJ_DIR     = obj
BIN_DIR     = bin

# สร้าง directories ถ้ายังไม่มี
$(shell mkdir -p $(OBJ_DIR) $(BIN_DIR))

# ===== Source Files =====
SRCS_X64    = $(wildcard $(SRC_DIR)/*.asm)
SRCS_ARM    = $(wildcard $(SRC_DIR)/*.arm.s)
SRCS_ARM64  = $(wildcard $(SRC_DIR)/*.arm64.s)

OBJS_X64    = $(patsubst $(SRC_DIR)/%.asm, $(OBJ_DIR)/%.o, $(SRCS_X64))
TARGETS_X64 = $(patsubst $(SRC_DIR)/%.asm, $(BIN_DIR)/%, $(SRCS_X64))

# ===== Targets =====
.PHONY: all x64 arm arm64 clean test run debug help

all: x64 arm arm64
	@echo ""
	@echo "All targets built successfully!"

# Build x86-64
x64: $(TARGETS_X64)

$(OBJ_DIR)/%.o: $(SRC_DIR)/%.asm
	@echo "[NASM] $< → $@"
	@$(NASM) $(NASM_FLAGS) $< -o $@

$(BIN_DIR)/%: $(OBJ_DIR)/%.o
	@echo "[LD]   $< → $@"
	@$(LD) $(LD_FLAGS) $< -o $@

# Build ARM 32-bit
arm: $(SRC_DIR)/hello.arm.s
	@echo "[ARM-AS] Assembling ARM 32-bit..."
	@$(ARM_AS) $(ARM_FLAGS) -o $(OBJ_DIR)/hello_arm.o $<
	@$(ARM_LD) -o $(BIN_DIR)/hello_arm $(OBJ_DIR)/hello_arm.o
	@echo "[ARM-LD] Done: $(BIN_DIR)/hello_arm"

# Build AArch64
arm64: $(SRC_DIR)/hello.arm64.s
	@echo "[ARM64-AS] Assembling AArch64..."
	@$(ARM64_AS) -o $(OBJ_DIR)/hello_arm64.o $<
	@$(ARM64_LD) -o $(BIN_DIR)/hello_arm64 $(OBJ_DIR)/hello_arm64.o
	@echo "[ARM64-LD] Done: $(BIN_DIR)/hello_arm64"

# Clean
clean:
	@echo "Cleaning build artifacts..."
	@rm -rf $(OBJ_DIR)/* $(BIN_DIR)/*
	@echo "Done."

# Test: รัน binaries ทั้งหมด
test: all
	@echo "=== Testing x86-64 ==="
	@for f in $(BIN_DIR)/*; do \
		if [ -f "$$f" ] && [ -x "$$f" ]; then \
			echo "Running: $$f"; \
			./$$f; \
		fi; \
	done

# Run specific target
run-%: $(BIN_DIR)/%
	@echo "Running $<..."
	@$<

# Debug: เปิด GDB
debug-%: $(BIN_DIR)/%
	@echo "Starting GDB for $<..."
	gdb -tui $<

# Help
help:
	@echo "$(PROJECT) v$(VERSION) - Build System"
	@echo ""
	@echo "Usage: make [target]"
	@echo ""
	@echo "Build Targets:"
	@echo "  all      - Build x64, ARM, AArch64 (default)"
	@echo "  x64      - Build x86-64 targets only"
	@echo "  arm      - Build ARM 32-bit only"
	@echo "  arm64    - Build AArch64 only"
	@echo ""
	@echo "Other Targets:"
	@echo "  clean    - Remove build artifacts"
	@echo "  test     - Run all built programs"
	@echo "  run-NAME - Run specific program"
	@echo "  debug-NAME - Debug with GDB"
	@echo "  help     - Show this message"
```

---

<a name="section-10"></a>
## 10. การ Compile และ Link

### 10.1 x86-64 Compilation Process

```
hello.asm
    │
    │ NASM: nasm -f elf64 hello.asm -o hello.o
    ↓
hello.o (Object File - ELF format)
    │
    │ LD: ld hello.o -o hello
    ↓
hello (Executable - ELF format)
    │
    │ OS Loader
    ↓
Running Process
```

```bash
# ขั้นตอนที่ 1: Assemble - แปลง Assembly เป็น Object File
nasm -f elf64 hello.asm -o hello.o

# ตรวจสอบ Object File
file hello.o
# ผลลัพธ์: hello.o: ELF 64-bit LSB relocatable, x86-64, ...

# ดูรายละเอียด Object File
objdump -d hello.o     # disassemble
nm hello.o             # ดู symbols
readelf -a hello.o     # ดู ELF information
```

```bash
# ขั้นตอนที่ 2: Link - รวม Object Files เป็น Executable
ld hello.o -o hello

# ตรวจสอบ Executable
file hello
# ผลลัพธ์: hello: ELF 64-bit LSB executable, x86-64, ...

# ดูขนาดของแต่ละ section
size hello

# รัน
./hello
```

### 10.2 การ Link กับ C Library (libc)

```bash
# ปกติ Assembly ที่ใช้ syscall โดยตรงไม่ต้อง link กับ libc
# แต่ถ้าต้องการใช้ printf, malloc ฯลฯ ต้อง link กับ libc

# Compile โปรแกรม Assembly ที่เรียก C functions
nasm -f elf64 mixed.asm -o mixed.o

# Link กับ libc (ต้องระบุ dynamic linker ด้วย)
ld mixed.o -o mixed \
    -lc \
    -dynamic-linker /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2

# หรือใช้ gcc เป็น linker (ง่ายกว่า)
gcc mixed.o -o mixed -no-pie -nostartfiles
```

### 10.3 Compilation Flags ที่สำคัญ

```bash
# NASM flags
nasm -f elf64          # ELF 64-bit format (Linux 64-bit)
nasm -f elf32          # ELF 32-bit format (Linux 32-bit)
nasm -f macho64        # Mach-O 64-bit format (macOS)
nasm -f win64          # PE 64-bit format (Windows)
nasm -f bin            # Raw binary (OS development, bootloaders)
nasm -g                # รวม debug symbols
nasm -F dwarf          # ใช้ DWARF debug format (สำหรับ GDB)
nasm -l hello.lst      # สร้าง listing file (ดูผลลัพธ์การ assemble)
nasm -E hello.asm      # Preprocess เท่านั้น (expand macros)
nasm -M hello.asm      # สร้าง Makefile dependencies

# LD flags
ld -m elf_x86_64       # สร้าง ELF x86-64 executable
ld -m elf_i386         # สร้าง ELF i386 executable
ld -s                  # Strip debug symbols (ลดขนาดไฟล์)
ld -e main             # ระบุ entry point เป็น 'main' แทน '_start'
ld -T linker.ld        # ใช้ custom linker script
```

---

<a name="section-11"></a>
## 11. Hello World โปรแกรมแรก

### 11.1 Hello World บน Linux x86-64 (NASM)

```nasm
; hello_x64.asm - Hello World สำหรับ Linux 64-bit
; คอมไพล์: nasm -f elf64 hello_x64.asm -o hello_x64.o
; Link:     ld hello_x64.o -o hello_x64
; รัน:      ./hello_x64
; ผลลัพธ์:  Hello, World! (Assembly x86-64)

; ======================================================
; Section .data - ข้อมูลที่กำหนดไว้ล่วงหน้า (initialized data)
; ======================================================
section .data
    msg     db  "Hello, World! (Assembly x86-64)", 0Ah
    ; db = define byte
    ; 0Ah = 10 decimal = newline character (\n)
    
    msg_len equ $ - msg
    ; $ = ตำแหน่งปัจจุบัน
    ; msg_len = ความยาวของ msg = 34 bytes

; ======================================================
; Section .bss - ข้อมูลที่ยังไม่ได้กำหนดค่า (uninitialized)
; ======================================================
section .bss
    ; (ไม่ใช้ในตัวอย่างนี้)

; ======================================================
; Section .text - โค้ดโปรแกรม
; ======================================================
section .text
    global _start       ; ประกาศให้ linker รู้ว่า _start คือจุดเริ่มต้น
    
_start:
    ; ==========================================
    ; System Call: sys_write (write to file)
    ; Syscall number: 1 (ใน /usr/include/asm/unistd_64.h)
    ; Arguments:
    ;   rdi = file descriptor (1 = stdout)
    ;   rsi = pointer to buffer
    ;   rdx = number of bytes to write
    ; ==========================================
    
    mov rax, 1          ; syscall number: sys_write = 1
    mov rdi, 1          ; arg1: file descriptor = 1 (stdout)
    mov rsi, msg        ; arg2: pointer to message buffer
    mov rdx, msg_len    ; arg3: number of bytes to write
    syscall             ; เรียก kernel (interrupt 0x80 แบบเก่า ไม่ใช้แล้วใน 64-bit)

    ; ==========================================
    ; System Call: sys_exit (terminate process)
    ; Syscall number: 60
    ; Arguments:
    ;   rdi = exit status code (0 = success)
    ; ==========================================
    
    mov rax, 60         ; syscall number: sys_exit = 60
    xor rdi, rdi        ; arg1: exit code = 0 (xor เร็วกว่า mov rdi, 0)
    syscall             ; เรียก kernel
```

```bash
# Build และรัน
nasm -f elf64 hello_x64.asm -o hello_x64.o
ld hello_x64.o -o hello_x64
./hello_x64
# ผลลัพธ์: Hello, World! (Assembly x86-64)

# ดูขนาด binary
ls -la hello_x64
# ประมาณ 700-900 bytes (เล็กมาก เพราะไม่มี C runtime)

# ดู system calls ที่เรียก
strace ./hello_x64
# ผลลัพธ์:
# execve("./hello_x64", ["./hello_x64"], ...) = 0
# write(1, "Hello, World! (Assembly x86-64)\n", 33) = 33
# exit(0) = ?
```

### 11.2 Hello World บน Linux x86 32-bit (NASM)

```nasm
; hello_x86.asm - Hello World สำหรับ Linux 32-bit
; คอมไพล์: nasm -f elf32 hello_x86.asm -o hello_x86.o
; Link:     ld -m elf_i386 hello_x86.o -o hello_x86
; รัน:      ./hello_x86
; หมายเหตุ: อาจต้องติดตั้ง lib32-dev: sudo apt install gcc-multilib

section .data
    msg     db  "Hello, World! (Assembly x86 32-bit)", 0Ah
    msg_len equ $ - msg

section .text
    global _start

_start:
    ; ==========================================
    ; ใน 32-bit Linux ใช้ int 0x80 แทน syscall
    ; Registers ใช้ eax, ebx, ecx, edx (32-bit registers)
    ; syscall numbers แตกต่างจาก 64-bit!
    ; ==========================================
    
    ; sys_write syscall number = 4 (ใน 32-bit ต่างจาก 64-bit ที่ใช้ 1)
    mov eax, 4          ; syscall: sys_write = 4 (32-bit)
    mov ebx, 1          ; arg1: stdout
    mov ecx, msg        ; arg2: pointer to message
    mov edx, msg_len    ; arg3: length
    int 0x80            ; interrupt 80h = Linux syscall (32-bit method)

    ; sys_exit syscall number = 1 (ใน 32-bit ต่างจาก 64-bit ที่ใช้ 60)
    mov eax, 1          ; syscall: sys_exit = 1 (32-bit)
    xor ebx, ebx        ; exit code = 0
    int 0x80
```

### 11.3 Hello World บน ARM 32-bit

```gas
@ hello_arm32.s - Hello World สำหรับ ARM 32-bit Linux
@ คอมไพล์: arm-linux-gnueabi-as -o hello_arm32.o hello_arm32.s
@ Link:     arm-linux-gnueabi-ld -o hello_arm32 hello_arm32.o
@ รัน:      qemu-arm ./hello_arm32

    .text
    .global _start

_start:
    @ ==========================================
    @ ARM 32-bit syscall convention:
    @ r7 = syscall number
    @ r0 = arg1, r1 = arg2, r2 = arg3
    @ swi 0 = software interrupt (เรียก syscall)
    @ ==========================================
    
    @ sys_write (syscall number = 4 สำหรับ ARM Linux 32-bit)
    mov r7, #4          @ syscall: sys_write
    mov r0, #1          @ arg1: stdout (file descriptor 1)
    ldr r1, =msg        @ arg2: address ของ message (ldr ใช้สำหรับ load address)
    mov r2, #27         @ arg3: ความยาว message
    swi 0               @ software interrupt = เรียก syscall

    @ sys_exit (syscall number = 1)
    mov r7, #1          @ syscall: sys_exit
    mov r0, #0          @ arg1: exit code 0
    swi 0               @ เรียก syscall

    .data
msg:
    .ascii "Hello, World! (ARM 32-bit)\n"   @ ข้อความ (27 chars)
```

### 11.4 Hello World บน AArch64 (ARM 64-bit)

```gas
// hello_aarch64.s - Hello World สำหรับ AArch64 (ARM 64-bit)
// คอมไพล์: aarch64-linux-gnu-as -o hello_aarch64.o hello_aarch64.s
// Link:     aarch64-linux-gnu-ld -o hello_aarch64 hello_aarch64.o
// รัน:      qemu-aarch64 ./hello_aarch64

    .text
    .global _start

_start:
    // ==========================================
    // AArch64 syscall convention:
    // x8 = syscall number
    // x0 = arg1, x1 = arg2, x2 = arg3
    // svc #0 = supervisor call (เรียก syscall)
    // syscall numbers ต่างจาก x86 และ ARM32!
    // ==========================================
    
    // sys_write (syscall number = 64 สำหรับ AArch64 Linux)
    mov x8, #64         // syscall: sys_write (64-bit ARM ใช้ 64, ไม่ใช่ 4)
    mov x0, #1          // arg1: stdout
    ldr x1, =msg        // arg2: address ของ message
    mov x2, #30         // arg3: ความยาว message
    svc #0              // supervisor call

    // sys_exit (syscall number = 93 สำหรับ AArch64 Linux)
    mov x8, #93         // syscall: sys_exit (64-bit ARM ใช้ 93, ไม่ใช่ 60)
    mov x0, #0          // exit code 0
    svc #0

    .data
msg:
    .ascii "Hello, World! (AArch64 64-bit)\n"    // 31 chars
```

### 11.5 ตาราง Syscall Numbers

| Syscall | x86 32-bit | x86-64 | ARM 32-bit | AArch64 |
|---------|-----------|--------|-----------|---------|
| sys_read | 3 | 0 | 3 | 63 |
| sys_write | 4 | 1 | 4 | 64 |
| sys_open | 5 | 2 | 5 | 56 |
| sys_close | 6 | 3 | 6 | 57 |
| sys_exit | 1 | 60 | 1 | 93 |
| sys_fork | 2 | 57 | 2 | - |
| sys_mmap | 90 | 9 | 90 | 222 |

### 11.6 รัน Hello World ทั้งหมด

```bash
#!/bin/bash
# build_and_run_all.sh - Script สำหรับ build และรัน Hello World ทุก architecture

set -e  # หยุดทันทีถ้ามี error

echo "=== Building and Testing Assembly Hello World ==="
echo ""

# ===== x86-64 =====
echo "--- x86-64 ---"
nasm -f elf64 hello_x64.asm -o hello_x64.o
ld hello_x64.o -o hello_x64
echo -n "Output: "
./hello_x64
echo ""

# ===== x86 32-bit =====
echo "--- x86 32-bit ---"
nasm -f elf32 hello_x86.asm -o hello_x86.o
ld -m elf_i386 hello_x86.o -o hello_x86
echo -n "Output: "
./hello_x86
echo ""

# ===== ARM 32-bit =====
echo "--- ARM 32-bit (via QEMU) ---"
arm-linux-gnueabi-as -o hello_arm32.o hello_arm32.s
arm-linux-gnueabi-ld -o hello_arm32 hello_arm32.o
echo -n "Output: "
qemu-arm ./hello_arm32
echo ""

# ===== AArch64 =====
echo "--- AArch64 (via QEMU) ---"
aarch64-linux-gnu-as -o hello_aarch64.o hello_aarch64.s
aarch64-linux-gnu-ld -o hello_aarch64 hello_aarch64.o
echo -n "Output: "
qemu-aarch64 ./hello_aarch64
echo ""

echo "=== All tests passed! ==="
```

---

<a name="section-12"></a>
## 12. ข้อผิดพลาดที่พบบ่อยและวิธีแก้ไข

### 12.1 NASM Errors

```
error: parser: instruction expected
```
**สาเหตุ:** ลืมเว้นบรรทัดว่างหลัง section label หรือ syntax ผิด  
**แก้ไข:**
```nasm
; ผิด
section .text
_start:mov rax, 1   ; ผิด: ไม่เว้นบรรทัด

; ถูก
section .text
_start:
    mov rax, 1      ; ถูก
```

```
error: symbol `_start' undefined
```
**สาเหตุ:** ลืม declare `global _start`  
**แก้ไข:**
```nasm
section .text
    global _start   ; ต้องมีบรรทัดนี้!
_start:
    ...
```

```
ld: warning: cannot find entry symbol _start; defaulting to 0000000000401000
```
**สาเหตุ:** Linker ไม่เจอ entry point  
**แก้ไข:** ตรวจสอบว่า `global _start` อยู่ใน `.text` section

### 12.2 Segmentation Fault (Segfault)

```bash
./hello
Segmentation fault (core dumped)
```

**สาเหตุที่พบบ่อย:**

```nasm
; ปัญหา 1: เข้าถึง memory address ผิด
mov rax, [0x12345678]   ; ผิด: address นั้นอาจไม่ valid

; ปัญหา 2: ลืม syscall ตอนจบ
_start:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg
    mov rdx, 13
    syscall
    ; ลืม exit syscall! โปรแกรมจะรันต่อไปใน memory ที่ไม่รู้จัก

; วิธีแก้: ต้องมี exit เสมอ
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 12.3 Wrong Architecture

```bash
./hello_arm
bash: ./hello_arm: cannot execute binary file: Exec format error
```

**สาเหตุ:** พยายามรัน ARM binary บน x86 โดยตรง  
**แก้ไข:** ใช้ QEMU
```bash
qemu-arm ./hello_arm    # ถูกต้อง
```

### 12.4 Missing Libraries

```bash
./hello_mixed
./hello_mixed: error while loading shared libraries: libc.so.6: cannot open shared object file
```

**สาเหตุ:** โปรแกรม link กับ libc แบบ dynamic แต่ library path ผิด  
**แก้ไข:**
```bash
# วิธีที่ 1: static linking
ld -static hello.o -lc -o hello

# วิธีที่ 2: ระบุ dynamic linker
ld hello.o -o hello \
   -lc \
   -dynamic-linker /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

### 12.5 Stack Alignment Issue

```bash
# โปรแกรมที่เรียก C functions อาจ crash ถ้า stack ไม่ align
```

```nasm
; ปัญหา: stack ต้อง align 16 bytes ก่อนเรียก C function
_start:
    ; ใน _start, stack อาจ align 8 bytes (ไม่ใช่ 16)
    ; ต้อง align ก่อนเรียก C function
    
    and rsp, -16        ; align stack เป็น 16 bytes (AND ด้วย -16 = 0xFFF...0)
    sub rsp, 8          ; จอง space (optional แต่ช่วยให้ align)
    
    ; ตอนนี้ถึงจะเรียก C functions ได้
    call printf
```

### 12.6 ตาราง Format สำหรับ NASM

| Platform | Format Flag | Linker Flag |
|---------|------------|------------|
| Linux 64-bit | `-f elf64` | (ค่าเริ่มต้น) |
| Linux 32-bit | `-f elf32` | `-m elf_i386` |
| macOS 64-bit | `-f macho64` | (ใช้ `ld` ของ macOS) |
| Windows 64-bit | `-f win64` | (ใช้ MSVC link.exe) |
| Raw binary | `-f bin` | (ไม่ต้อง link) |

---

<a name="section-13"></a>
## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: ตรวจสอบการติดตั้ง

ทำตามขั้นตอนต่อไปนี้และบันทึกผลลัพธ์:

```bash
# ตรวจสอบทุกเครื่องมือ
nasm --version
as --version
ld --version
gdb --version
qemu-arm --version 2>/dev/null || echo "qemu-arm not installed"
arm-linux-gnueabi-gcc --version 2>/dev/null || echo "ARM toolchain not installed"

# บันทึกผลลัพธ์ทั้งหมดไปยังไฟล์
nasm --version > tool_versions.txt
as --version >> tool_versions.txt
ld --version >> tool_versions.txt
cat tool_versions.txt
```

**คำถาม:** เครื่องมือไหนติดตั้งแล้ว? เครื่องมือไหนยังขาด?

### แบบฝึกหัดที่ 2: Hello World ด้วยข้อมูลของตัวเอง

เขียนโปรแกรม Assembly ที่แสดงชื่อและประโยคต้อนรับของคุณเอง:

```nasm
; exercise2.asm - แสดงข้อมูลส่วนตัว
; แก้ไข: เปลี่ยน YOUR_NAME และ YOUR_MESSAGE เป็นข้อมูลจริง

section .data
    ; เปลี่ยนตรงนี้:
    name     db  "YOUR_NAME", 0Ah        ; ชื่อของคุณ + newline
    message  db  "YOUR_MESSAGE", 0Ah     ; ข้อความ + newline
    
    name_len    equ $ - name
    ; แก้ไข: message_len ต้องคำนวณอย่างไร?

section .text
    global _start

_start:
    ; แสดงชื่อ
    ; TODO: เติม syscall สำหรับ sys_write
    
    ; แสดงข้อความ
    ; TODO: เติม syscall สำหรับ sys_write
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

**ผลลัพธ์ที่คาดหวัง:**
```
[ชื่อของคุณ]
[ข้อความของคุณ]
```

### แบบฝึกหัดที่ 3: สร้าง Makefile

สร้าง Makefile สำหรับ project directory ที่มีโครงสร้างดังนี้:

```
myproject/
├── src/
│   ├── hello.asm
│   └── goodbye.asm
├── obj/     (สร้างอัตโนมัติ)
├── bin/     (สร้างอัตโนมัติ)
└── Makefile  ← สร้างไฟล์นี้
```

**Requirements:**
- `make` หรือ `make all` → build ทั้ง hello และ goodbye ไปที่ `bin/`
- `make clean` → ลบทุกอย่างใน `obj/` และ `bin/`
- `make run-hello` → build แล้วรัน hello
- Output ของ make ต้องมีสีสัน (เขียว/เหลือง/แดง)

### แบบฝึกหัดที่ 4: Debug ด้วย GDB

ใช้โปรแกรม hello_x64.asm จากบทเรียน แล้วทำตามขั้นตอน:

```bash
# 1. Compile ด้วย debug symbols
nasm -f elf64 -g -F dwarf hello_x64.asm -o hello_x64.o
ld hello_x64.o -o hello_x64

# 2. เปิด GDB
gdb ./hello_x64

# 3. ใน GDB ทำตามขั้นตอน:
# - break _start
# - run
# - ดูค่าใน rax, rdi, rsi, rdx
# - stepi 5 ครั้ง
# - ดู memory ที่ msg ชี้ไป
# - continue จนจบ
```

**บันทึก:** ค่าใน register แต่ละตัวในแต่ละ step คืออะไร?

### แบบฝึกหัดที่ 5: Cross-compile สำหรับ ARM

เขียนและรัน Hello World สำหรับ ARM บนเครื่อง x86:

```bash
# 1. เขียน ARM Assembly (ใช้ template จากบทเรียน)
# 2. Assemble ด้วย arm-linux-gnueabi-as
# 3. Link ด้วย arm-linux-gnueabi-ld
# 4. ตรวจสอบว่าเป็น ARM binary ด้วย: file hello_arm
# 5. รันด้วย qemu-arm
```

**Challenge:** แก้ไขข้อความให้เป็น "สวัสดีชาวโลก!" (Unicode Thai text)  
**Hint:** ต้องระบุความยาวของ UTF-8 string ให้ถูกต้อง (ภาษาไทยใช้ 3 bytes ต่อตัวอักษร)

### แบบฝึกหัดที่ 6: เปรียบเทียบ Binary Size

Build Hello World ด้วยวิธีต่างๆ แล้วเปรียบเทียบขนาด:

```bash
# Assembly
nasm -f elf64 hello_x64.asm -o hello_x64_asm.o
ld hello_x64_asm.o -o hello_asm
ls -la hello_asm

# C (ไม่ optimize)
echo '#include <stdio.h>\nint main() { printf("Hello!\\n"); return 0; }' > hello.c
gcc hello.c -o hello_c
ls -la hello_c

# C (static)
gcc hello.c -static -o hello_c_static
ls -la hello_c_static

# C (optimize + strip)
gcc -O3 -s hello.c -o hello_c_opt
ls -la hello_c_opt

# เปรียบเทียบ
echo "Assembly: $(du -b hello_asm | cut -f1) bytes"
echo "C (dynamic): $(du -b hello_c | cut -f1) bytes"
echo "C (static): $(du -b hello_c_static | cut -f1) bytes"
```

**คำถาม:** ทำไม Assembly binary ถึงเล็กที่สุด?

### แบบฝึกหัดที่ 7: ตรวจสอบ Syscalls

ใช้ `strace` เพื่อดู system calls ที่โปรแกรมเรียก:

```bash
# ดู syscalls ของ Assembly program
strace ./hello_asm 2>&1 | head -20

# ดู syscalls ของ C program
strace ./hello_c 2>&1 | head -20

# เปรียบเทียบจำนวน syscalls
strace -c ./hello_asm 2>&1
strace -c ./hello_c 2>&1
```

**คำถาม:** โปรแกรม C มี syscall อะไรเพิ่มเติมที่ Assembly ไม่มี และทำไม?

---

<a name="section-14"></a>
## 14. สรุปและ Key Takeaways

### สิ่งที่เรียนรู้ใน Part นี้

1. **NASM** คือ Assembler หลักที่ใช้ในคอร์สนี้ รองรับ Intel syntax ที่อ่านง่าย
2. **GAS** เป็น GNU Assembler ที่มากับ binutils ใช้ AT&T syntax
3. **Cross-compiler** ช่วยให้เราสร้าง ARM binary บนเครื่อง x86 ได้
4. **QEMU** ช่วยให้รัน ARM binary บน x86 โดยไม่ต้องมี hardware จริง
5. **GDB** เป็นเครื่องมือ debug ที่จำเป็นสำหรับ Assembly
6. **Makefile** ช่วยทำให้ build process อัตโนมัติ
7. **Syscall numbers** ต่างกันตาม architecture และ OS

### การเลือก Assembler ที่เหมาะสม

```
ต้องการเรียน x86/x86-64?
    └─→ ใช้ NASM (Intel syntax - อ่านง่ายกว่า)

ต้องการทำงานกับ GCC?
    └─→ ใช้ GAS (AT&T syntax - compat กับ GCC)

ต้องการ Windows development?
    └─→ ใช้ MASM หรือ NASM (win64 format)

ต้องการ ARM development?
    └─→ ใช้ GAS ผ่าน cross-compiler toolchain
```

### Checklist สำหรับ Environment ที่สมบูรณ์

```bash
# รัน script นี้เพื่อตรวจสอบ environment
#!/bin/bash

check_tool() {
    if command -v $1 &> /dev/null; then
        echo "✓ $1 installed: $($1 --version 2>&1 | head -1)"
    else
        echo "✗ $1 NOT installed"
    fi
}

echo "=== Assembly Development Environment Check ==="
check_tool nasm
check_tool as
check_tool ld
check_tool gdb
check_tool make
check_tool qemu-arm
check_tool qemu-aarch64
check_tool arm-linux-gnueabi-as
check_tool aarch64-linux-gnu-as
echo "=============================================="
```

---

<a name="section-15"></a>
## 15. แหล่งอ้างอิง

### เอกสารทางการ
- **NASM Manual:** https://nasm.us/doc/nasmdoc.html
- **GAS Manual:** https://sourceware.org/binutils/docs/as/
- **GDB Manual:** https://www.gnu.org/software/gdb/documentation/
- **QEMU Documentation:** https://www.qemu.org/docs/master/

### Linux Syscall References
- **x86-64 syscalls:** https://filippo.io/linux-syscall-table/
- **ARM syscalls:** https://syscalls.w3challs.com/?arch=arm_strong
- **AArch64 syscalls:** https://syscalls.w3challs.com/?arch=arm64

### Cross-Compiler Resources
- **ARM GCC Toolchain:** https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain
- **Linaro Toolchain:** https://www.linaro.org/downloads/

### เครื่องมือออนไลน์ (ไม่ต้องติดตั้ง)
- **Compiler Explorer (godbolt.org):** https://godbolt.org - ดู Assembly output จาก C code
- **Online NASM:** https://www.tutorialspoint.com/compile_assembly_online.php
- **Defuse.ca:** https://defuse.ca/online-x86-assembler.htm

### หนังสือแนะนำ
- **"Programming from the Ground Up"** - Jonathan Bartlett (ฟรี)
- **"Introduction to 64 Bit Intel Assembly Language"** - Ray Seyfarth
- **"ARM Assembly Language"** - William Hohl

---

## Quick Reference Card

```
╔══════════════════════════════════════════════════════════════╗
║              ASSEMBLY QUICK REFERENCE                       ║
╠══════════════════════════════════════════════════════════════╣
║ COMPILE x86-64:                                             ║
║   nasm -f elf64 file.asm -o file.o && ld file.o -o file     ║
║                                                             ║
║ COMPILE x86 32-bit:                                         ║
║   nasm -f elf32 file.asm -o file.o                          ║
║   ld -m elf_i386 file.o -o file                             ║
║                                                             ║
║ COMPILE ARM 32-bit:                                         ║
║   arm-linux-gnueabi-as -o file.o file.s                     ║
║   arm-linux-gnueabi-ld -o file file.o                       ║
║   qemu-arm ./file                                           ║
║                                                             ║
║ COMPILE AArch64:                                            ║
║   aarch64-linux-gnu-as -o file.o file.s                     ║
║   aarch64-linux-gnu-ld -o file file.o                       ║
║   qemu-aarch64 ./file                                       ║
║                                                             ║
║ DEBUG:                                                      ║
║   gdb ./file                                                ║
║   (gdb) break _start                                        ║
║   (gdb) run                                                 ║
║   (gdb) stepi                                               ║
║   (gdb) info registers                                      ║
╚══════════════════════════════════════════════════════════════╝
```

---

**หน้าถัดไป:** [Part 006 - Registers และ Memory Model](part-006-registers-memory.md)

*Part 005 สมบูรณ์แล้ว - ยินดีด้วย! ตอนนี้คุณมี Development Environment ที่พร้อมสำหรับการเรียน Assembly แล้ว*
