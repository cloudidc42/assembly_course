# Part 008: โปรแกรม Assembly แรก (x86)

**เวลาที่ใช้เรียน:** 4-6 ชั่วโมง  
**ระดับ:** พื้นฐาน-กลาง  
**Prerequisites:** Part 001-007 (ความรู้พื้นฐาน Registers, Memory, CPU)

---

## วัตถุประสงค์การเรียนรู้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:

- เขียนโปรแกรม Hello World ด้วย NASM syntax บน Linux 32-bit และ 64-bit
- เข้าใจโครงสร้าง sections ใน Assembly (`.text`, `.data`, `.bss`)
- Compile และ Link โปรแกรม Assembly ด้วย NASM และ ld
- เขียนโปรแกรม Hello World บน Windows ด้วย Win32 API
- ใช้ GDB เพื่อ debug และดู execution ของโปรแกรม
- ใช้ `objdump` และ `readelf` วิเคราะห์ binary files
- วิเคราะห์ทุก byte ในโปรแกรม Hello World
- เขียนโปรแกรมรับ input จาก stdin
- เขียน Calculator อย่างง่ายด้วย Assembly

---

## 1. โครงสร้างพื้นฐานของ NASM Syntax

### 1.1 Sections คืออะไร?

ใน Assembly โปรแกรมถูกแบ่งออกเป็น **sections** (ส่วน) ต่างๆ แต่ละ section มีหน้าที่เฉพาะ:

```
+------------------+
|   .text section  |  ← โค้ด (instructions)
+------------------+
|   .data section  |  ← ข้อมูลที่กำหนดค่าไว้แล้ว (initialized data)
+------------------+
|   .bss section   |  ← ข้อมูลที่ยังไม่มีค่า (uninitialized data)
+------------------+
|   Stack          |  ← ข้อมูลชั่วคราว (ระหว่าง runtime)
+------------------+
|   Heap           |  ← Dynamic memory allocation
+------------------+
```

### 1.2 รายละเอียดแต่ละ Section

| Section | หน้าที่ | สิทธิ์การเข้าถึง | ตัวอย่าง |
|---------|---------|-----------------|---------|
| `.text` | เก็บโค้ดโปรแกรม | Read + Execute | `mov eax, 1` |
| `.data` | ข้อมูลที่มีค่าเริ่มต้น | Read + Write | `msg db "Hello"` |
| `.bss` | ข้อมูลที่ยังไม่มีค่า | Read + Write | `buffer resb 64` |
| `.rodata` | ข้อมูลอ่านอย่างเดียว | Read only | constant strings |

### 1.3 โครงสร้าง Labels

Label คือชื่อที่ใช้อ้างอิงตำแหน่งในโค้ดหรือข้อมูล:

```nasm
; รูปแบบ label ใน NASM
label_name:          ; label ทั่วไป
.local_label:        ; local label (ขึ้นต้นด้วยจุด)
_start:             ; entry point ของโปรแกรม (global label)
```

### 1.4 ประเภทของ Instructions ใน NASM

```nasm
; 1. Data Movement Instructions
mov  dst, src       ; คัดลอกข้อมูล
push value          ; เพิ่มข้อมูลบน stack
pop  register       ; นำข้อมูลออกจาก stack
lea  reg, [mem]     ; Load Effective Address

; 2. Arithmetic Instructions  
add  dst, src       ; บวก
sub  dst, src       ; ลบ
mul  src            ; คูณ (unsigned)
div  src            ; หาร (unsigned)
inc  dst            ; เพิ่มค่าทีละ 1
dec  dst            ; ลดค่าทีละ 1

; 3. Logic Instructions
and  dst, src       ; AND
or   dst, src       ; OR
xor  dst, src       ; XOR
not  dst            ; NOT
shl  dst, count     ; Shift Left
shr  dst, count     ; Shift Right

; 4. Control Flow Instructions
jmp  label          ; กระโดดไปยัง label
je   label          ; กระโดดถ้าเท่ากัน
jne  label          ; กระโดดถ้าไม่เท่ากัน
call function       ; เรียก function
ret                 ; กลับจาก function
```

---

## 2. Data Directives ใน NASM

### 2.1 Initialized Data (ใน .data section)

```nasm
section .data
    ; กำหนดตัวแปรพร้อมค่าเริ่มต้น
    
    ; ตัวอักษร 1 ไบต์ (8-bit)
    byte_var   db 42          ; db = define byte
    
    ; ตัวเลข 2 ไบต์ (16-bit word)
    word_var   dw 1000        ; dw = define word
    
    ; ตัวเลข 4 ไบต์ (32-bit doubleword)
    dword_var  dd 100000      ; dd = define doubleword
    
    ; ตัวเลข 8 ไบต์ (64-bit quadword)
    qword_var  dq 1000000000  ; dq = define quadword
    
    ; String (ลำดับของ bytes)
    message    db "Hello, World!", 10  ; 10 = newline (\n)
    msg_len    equ $ - message         ; คำนวณความยาว string อัตโนมัติ
    
    ; Array ของตัวเลข
    numbers    dd 10, 20, 30, 40, 50
    
    ; Float (32-bit)
    float_val  dd 3.14        ; dd ใช้สำหรับ float ด้วย
    
    ; Double (64-bit)
    double_val dq 3.14159265  ; dq ใช้สำหรับ double ด้วย
```

### 2.2 Uninitialized Data (ใน .bss section)

```nasm
section .bss
    ; จองพื้นที่สำหรับตัวแปร (ยังไม่มีค่า)
    
    ; resb = reserve bytes
    buffer     resb 256       ; จอง 256 bytes
    
    ; resw = reserve words (2 bytes each)
    word_buf   resw 10        ; จอง 10 words = 20 bytes
    
    ; resd = reserve doublewords (4 bytes each)
    dword_buf  resd 5         ; จอง 5 dwords = 20 bytes
    
    ; resq = reserve quadwords (8 bytes each)
    qword_buf  resq 3         ; จอง 3 qwords = 24 bytes
```

### 2.3 Constants และ EQU Directive

```nasm
; EQU กำหนดค่าคงที่ (ไม่ใช้ memory)
SYS_EXIT    equ 1      ; syscall number สำหรับ exit
SYS_WRITE   equ 4      ; syscall number สำหรับ write
STDOUT      equ 1      ; file descriptor สำหรับ stdout
STDIN       equ 0      ; file descriptor สำหรับ stdin
NEWLINE     equ 10     ; ASCII code ของ newline

; % define คล้าย #define ใน C
%define MAX_SIZE 1024
%define NULL 0
```

---

## 3. Hello World บน Linux 32-bit (int 0x80)

### 3.1 System Calls บน Linux 32-bit

บน Linux 32-bit การเรียก system call ใช้ **interrupt 0x80**:

| Register | หน้าที่ |
|----------|---------|
| `eax` | syscall number |
| `ebx` | argument 1 |
| `ecx` | argument 2 |
| `edx` | argument 3 |
| `esi` | argument 4 |
| `edi` | argument 5 |

**System calls ที่ใช้บ่อย (32-bit):**

| Syscall | Number | หน้าที่ |
|---------|--------|---------|
| `sys_exit` | 1 | จบโปรแกรม |
| `sys_read` | 3 | อ่านข้อมูล |
| `sys_write` | 4 | เขียนข้อมูล |

### 3.2 โปรแกรม Hello World แรก (32-bit)

```nasm
; ========================================
; hello32.asm - Hello World บน Linux 32-bit
; Compile: nasm -f elf32 hello32.asm -o hello32.o
; Link:    ld -m elf_i386 hello32.o -o hello32
; Run:     ./hello32
; ========================================

; ประกาศ global symbol (entry point ของโปรแกรม)
global _start

; ========================================
; .data section: ข้อมูลที่มีค่าเริ่มต้น
; ========================================
section .data
    ; กำหนด string "Hello, World!\n"
    ; db = define byte
    ; 10 คือ ASCII code ของ newline (\n)
    message db "Hello, World!", 10
    
    ; คำนวณความยาวของ string อัตโนมัติ
    ; $ = ตำแหน่งปัจจุบัน
    ; $ - message = ความยาวตั้งแต่ต้น message จนถึงตำแหน่งปัจจุบัน
    msg_len equ $ - message

; ========================================
; .text section: โค้ดโปรแกรม
; ========================================
section .text

_start:
    ; -----------------------------------------------
    ; System call: write(stdout, message, msg_len)
    ; เขียน string ไปที่หน้าจอ
    ; -----------------------------------------------
    
    mov eax, 4          ; syscall number 4 = sys_write
    mov ebx, 1          ; file descriptor 1 = stdout (หน้าจอ)
    mov ecx, message    ; pointer ไปยัง string ที่ต้องการแสดง
    mov edx, msg_len    ; จำนวน bytes ที่ต้องการเขียน
    int 0x80            ; เรียก kernel (software interrupt)
    
    ; -----------------------------------------------
    ; System call: exit(0)
    ; จบโปรแกรมอย่างถูกต้อง
    ; -----------------------------------------------
    
    mov eax, 1          ; syscall number 1 = sys_exit
    mov ebx, 0          ; exit code 0 = สำเร็จ
    int 0x80            ; เรียก kernel
```

**วิธี Compile และ Run:**

```bash
# Step 1: Assemble (แปลง .asm เป็น .o object file)
nasm -f elf32 hello32.asm -o hello32.o

# Step 2: Link (รวม object files เป็น executable)
ld -m elf_i386 hello32.o -o hello32

# Step 3: Run
./hello32

# ผลลัพธ์:
# Hello, World!
```

**อธิบายทุก instruction:**

```
mov eax, 4    → เก็บ 4 ใน register EAX (4 = sys_write)
mov ebx, 1    → เก็บ 1 ใน register EBX (1 = stdout)
mov ecx, msg  → เก็บ address ของ message ใน ECX
mov edx, len  → เก็บความยาว ใน EDX
int 0x80      → เรียก kernel ด้วย interrupt 0x80
               kernel อ่าน EAX เพื่อรู้ว่าต้องทำอะไร
               kernel อ่าน EBX, ECX, EDX เป็น arguments
```

---

## 4. Hello World บน Linux 64-bit (syscall instruction)

### 4.1 System Calls บน Linux 64-bit

บน Linux 64-bit การเรียก system call ใช้ **instruction `syscall`** (แทน `int 0x80`):

| Register | หน้าที่ |
|----------|---------|
| `rax` | syscall number |
| `rdi` | argument 1 |
| `rsi` | argument 2 |
| `rdx` | argument 3 |
| `r10` | argument 4 |
| `r8` | argument 5 |
| `r9` | argument 6 |

**System calls ที่ใช้บ่อย (64-bit):**

| Syscall | Number | หน้าที่ |
|---------|--------|---------|
| `sys_read` | 0 | อ่านข้อมูล |
| `sys_write` | 1 | เขียนข้อมูล |
| `sys_exit` | 60 | จบโปรแกรม |

**สังเกตว่า syscall numbers ต่างกันระหว่าง 32-bit และ 64-bit!**

### 4.2 โปรแกรม Hello World (64-bit)

```nasm
; ========================================
; hello64.asm - Hello World บน Linux 64-bit
; Compile: nasm -f elf64 hello64.asm -o hello64.o
; Link:    ld hello64.o -o hello64
; Run:     ./hello64
; ========================================

global _start

section .data
    ; กำหนด string พร้อม newline
    message db "Hello, World!", 10
    msg_len equ $ - message

section .text

_start:
    ; -----------------------------------------------
    ; System call: write(1, message, msg_len)
    ; ใน 64-bit: syscall number ต่างกัน, registers ต่างกัน
    ; -----------------------------------------------
    
    mov rax, 1          ; syscall number 1 = sys_write (64-bit)
    mov rdi, 1          ; argument 1: file descriptor = stdout
    mov rsi, message    ; argument 2: pointer to string
    mov rdx, msg_len    ; argument 3: number of bytes to write
    syscall             ; เรียก kernel (ใช้ syscall แทน int 0x80)
    
    ; -----------------------------------------------
    ; System call: exit(0)
    ; ใน 64-bit: exit syscall number = 60
    ; -----------------------------------------------
    
    mov rax, 60         ; syscall number 60 = sys_exit (64-bit)
    mov rdi, 0          ; argument 1: exit code = 0 (success)
    syscall             ; เรียก kernel
```

**วิธี Compile และ Run:**

```bash
# Assemble สำหรับ 64-bit ELF
nasm -f elf64 hello64.asm -o hello64.o

# Link (ไม่ต้องระบุ -m เพราะ default คือ 64-bit)
ld hello64.o -o hello64

# Run
./hello64

# ผลลัพธ์:
# Hello, World!
```

### 4.3 เปรียบเทียบ 32-bit vs 64-bit

```
          32-bit (int 0x80)    64-bit (syscall)
          ─────────────────    ────────────────
syscall#  EAX                  RAX
arg 1     EBX                  RDI
arg 2     ECX                  RSI
arg 3     EDX                  RDX
arg 4     ESI                  R10
arg 5     EDI                  R8
arg 6     EBP                  R9

write#    4                    1
read#     3                    0
exit#     1                    60
```

---

## 5. Hello World บน Windows (Win32 API)

### 5.1 โครงสร้างโปรแกรม Windows Assembly

บน Windows เราใช้ **Win32 API** แทน Linux syscalls:

```nasm
; ========================================
; hello_win.asm - Hello World บน Windows
; Compile: nasm -f win32 hello_win.asm -o hello_win.obj
; Link:    link hello_win.obj /subsystem:console kernel32.lib
;   หรือ   gcc hello_win.obj -o hello_win.exe -lkernel32 -mwindows
; ========================================

; Import functions จาก Windows DLL
extern GetStdHandle
extern WriteConsoleA
extern ExitProcess

global _start

section .data
    message db "Hello, World!", 13, 10  ; 13=CR, 10=LF (Windows line ending)
    msg_len equ $ - message
    
section .bss
    ; จองพื้นที่สำหรับเก็บจำนวน bytes ที่เขียนได้
    bytes_written resd 1    ; 4 bytes สำหรับ DWORD

section .text

_start:
    ; -----------------------------------------------
    ; หา handle ของ stdout
    ; GetStdHandle(STD_OUTPUT_HANDLE)
    ; STD_OUTPUT_HANDLE = -11 (0xFFFFFFF5)
    ; -----------------------------------------------
    push -11                    ; STD_OUTPUT_HANDLE = -11
    call GetStdHandle           ; เรียก Windows API
    ; ผลลัพธ์อยู่ใน EAX (handle ของ stdout)
    
    ; -----------------------------------------------
    ; เขียน string ไปที่ console
    ; WriteConsoleA(handle, buffer, length, bytesWritten, reserved)
    ; -----------------------------------------------
    push 0                      ; lpReserved = NULL
    push bytes_written          ; lpNumberOfCharsWritten (output)
    push msg_len                ; nNumberOfCharsToWrite
    push message                ; lpBuffer
    push eax                    ; hConsoleOutput (จาก GetStdHandle)
    call WriteConsoleA          ; เรียก Windows API
    
    ; -----------------------------------------------
    ; จบโปรแกรม
    ; ExitProcess(0)
    ; -----------------------------------------------
    push 0                      ; uExitCode = 0
    call ExitProcess            ; เรียก Windows API
```

**หมายเหตุ:** บน Windows:
- ใช้ `CRLF` (`\r\n`, byte 13 และ 10) เป็น newline
- ใช้ `cdecl` calling convention: arguments push ลง stack ก่อน call
- Stack alignment ต้องเป็น 4 bytes (32-bit)

---

## 6. การ Compile และ Link อย่างละเอียด

### 6.1 ขั้นตอนการสร้าง Executable

```
Source Code (.asm)
       ↓
   Assembler (NASM)
       ↓
  Object File (.o)
       ↓
    Linker (ld)
       ↓
  Executable file
```

### 6.2 NASM Options ที่สำคัญ

```bash
# Format options:
nasm -f elf32  file.asm    # Linux 32-bit
nasm -f elf64  file.asm    # Linux 64-bit
nasm -f win32  file.asm    # Windows 32-bit
nasm -f win64  file.asm    # Windows 64-bit
nasm -f bin    file.asm    # Raw binary (no format)
nasm -f macho  file.asm    # macOS 32-bit
nasm -f macho64 file.asm   # macOS 64-bit

# Debug options:
nasm -g        file.asm    # เพิ่ม debug information
nasm -F dwarf  file.asm    # ใช้ DWARF debug format

# Listing file (ดู machine code):
nasm -l list.lst file.asm  # สร้าง listing file

# Preprocessor:
nasm -D SYMBOL     # Define macro
nasm -I /path      # Include directory

# ตัวอย่าง compile พร้อม debug info:
nasm -f elf64 -g -F dwarf hello64.asm -o hello64.o
```

### 6.3 Linker (ld) Options ที่สำคัญ

```bash
# ลิงค์ขั้นพื้นฐาน
ld hello.o -o hello

# กำหนด entry point (ถ้าไม่ใช่ _start)
ld -e main hello.o -o hello

# ลิงค์กับ C standard library
ld -lc hello.o -o hello

# ลิงค์ 32-bit บน 64-bit system
ld -m elf_i386 hello32.o -o hello32

# ลิงค์หลาย object files
ld file1.o file2.o file3.o -o program

# กำหนด linker script
ld -T script.ld hello.o -o hello

# ดู linker version
ld --version
```

### 6.4 Makefile สำหรับ Assembly Projects

```makefile
# Makefile สำหรับโปรแกรม Assembly
# ใช้: make hello32 หรือ make hello64

# Tools
NASM = nasm
LD   = ld
GDB  = gdb

# Flags
NASM_FLAGS_32 = -f elf32 -g -F dwarf
NASM_FLAGS_64 = -f elf64 -g -F dwarf
LD_FLAGS_32   = -m elf_i386
LD_FLAGS_64   =

# Targets
all: hello32 hello64

hello32: hello32.o
	$(LD) $(LD_FLAGS_32) $< -o $@

hello64: hello64.o
	$(LD) $(LD_FLAGS_64) $< -o $@

%.o: %.asm
	$(NASM) $(NASM_FLAGS_64) $< -o $@

hello32.o: hello32.asm
	$(NASM) $(NASM_FLAGS_32) $< -o $@

clean:
	rm -f *.o hello32 hello64

debug32: hello32
	$(GDB) hello32

debug64: hello64
	$(GDB) hello64

.PHONY: all clean debug32 debug64
```

---

## 7. GAS (AT&T) Syntax Version

### 7.1 ความแตกต่างระหว่าง NASM และ GAS

| ลักษณะ | NASM (Intel) | GAS (AT&T) |
|--------|-------------|------------|
| Operand order | `dst, src` | `src, dst` |
| Register prefix | ไม่มี (`eax`) | มี `%` (`%eax`) |
| Immediate prefix | ไม่มี (`42`) | มี `$` (`$42`) |
| Memory access | `[addr]` | `addr` หรือ `(reg)` |
| Size suffix | ไม่มี | `b/w/l/q` |

### 7.2 Hello World ด้วย GAS Syntax

```asm
# ========================================
# hello_gas.s - Hello World ด้วย GAS syntax
# Compile: as --32 hello_gas.s -o hello_gas.o
# Link:    ld -m elf_i386 hello_gas.o -o hello_gas
# หรือใช้ gcc:
# gcc -m32 -nostdlib hello_gas.s -o hello_gas
# ========================================

# .section แทน section ของ NASM
.section .data
    message:
        .ascii "Hello, World!\n"    # .ascii ไม่เพิ่ม null terminator
        # .asciz จะเพิ่ม null terminator โดยอัตโนมัติ
    msg_len = . - message           # . คือตำแหน่งปัจจุบัน (เหมือน $ ใน NASM)

.section .text

# .globl ใช้แทน global ของ NASM
.globl _start

_start:
    # write(1, message, msg_len)
    # ใน AT&T syntax: src อยู่ก่อน, dst อยู่หลัง
    movl $4, %eax           # $4 = immediate value 4 (syscall write)
                             # %eax = register eax
    movl $1, %ebx           # stdout = 1
    movl $message, %ecx     # address ของ message
    movl $msg_len, %edx     # ความยาวของ message
    int  $0x80              # software interrupt
    
    # exit(0)
    movl $1, %eax           # syscall exit = 1
    movl $0, %ebx           # exit code = 0
    int  $0x80
```

### 7.3 Hello World 64-bit ด้วย GAS

```asm
# ========================================
# hello_gas64.s - Hello World 64-bit ด้วย GAS
# Compile: as hello_gas64.s -o hello_gas64.o
# Link:    ld hello_gas64.o -o hello_gas64
# ========================================

.section .data
    message:
        .ascii "Hello, World!\n"
    msg_len = . - message

.section .text
.globl _start

_start:
    # write(1, message, msg_len) - 64-bit syscall
    movq $1, %rax           # syscall 1 = write (ใช้ q = quadword = 64-bit)
    movq $1, %rdi           # arg1: stdout
    movq $message, %rsi     # arg2: buffer address
    movq $msg_len, %rdx     # arg3: length
    syscall                 # เรียก kernel
    
    # exit(0) - 64-bit
    movq $60, %rax          # syscall 60 = exit (64-bit)
    movq $0, %rdi           # exit code = 0
    syscall
```

---

## 8. การใช้ GDB ดู Program Execution

### 8.1 Compile พร้อม Debug Information

```bash
# ต้องเพิ่ม -g flag เพื่อให้มี debug information
nasm -f elf64 -g -F dwarf hello64.asm -o hello64.o
ld hello64.o -o hello64
```

### 8.2 คำสั่ง GDB พื้นฐาน

```bash
# เริ่ม GDB
gdb ./hello64

# คำสั่งภายใน GDB:
(gdb) list             # แสดงโค้ด source
(gdb) break _start     # ตั้ง breakpoint ที่ _start
(gdb) run              # รันโปรแกรม
(gdb) info registers   # ดูค่า registers ทั้งหมด
(gdb) info reg rax     # ดูค่า RAX เท่านั้น
(gdb) next             # ทำ instruction ถัดไป (step over)
(gdb) stepi            # ทำ instruction ถัดไป (step into)
(gdb) nexti            # เหมือน next แต่สำหรับ assembly
(gdb) x/10x $rsp       # ดูหน่วยความจำที่ RSP (hex format)
(gdb) x/s $rsi         # ดูหน่วยความจำเป็น string
(gdb) disassemble      # ดู disassembly ของ function ปัจจุบัน
(gdb) quit             # ออกจาก GDB
```

### 8.3 GDB Session ตัวอย่าง

```
$ gdb ./hello64
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1
(gdb) break _start
Breakpoint 1 at 0x401000

(gdb) run
Starting program: /home/user/hello64 

Breakpoint 1, 0x0000000000401000 in _start ()

(gdb) info registers
rax            0x0                 0
rbx            0x0                 0
rcx            0x0                 0
rdx            0x0                 0
rsi            0x0                 0
rdi            0x0                 0
rip            0x401000            0x401000 <_start>
...

(gdb) nexti
0x0000000000401007 in _start ()

(gdb) info reg rax
rax            0x1                 1
```

### 8.4 การดู Memory ด้วย GDB

```bash
# รูปแบบ: x/[count][format][size] address
# format: x=hex, d=decimal, s=string, i=instruction
# size: b=byte, h=halfword(2), w=word(4), g=giant(8)

(gdb) x/20xb &message    # ดู 20 bytes ที่ message (hex)
(gdb) x/s &message       # ดู string ที่ message
(gdb) x/10i _start       # ดู 10 instructions จาก _start
(gdb) x/4xw $rsp         # ดู stack (4 dwords ใน hex)
```

---

## 9. การดู Disassembly ด้วย objdump

### 9.1 คำสั่ง objdump พื้นฐาน

```bash
# ดู disassembly ของ executable
objdump -d hello64

# ดู disassembly พร้อม source code (ต้องมี debug info)
objdump -d -S hello64

# ดูเฉพาะ section .text
objdump -d -j .text hello64

# ดู all sections (รวม .data, .bss)
objdump -D hello64

# ดูในรูปแบบ Intel syntax (แทน AT&T)
objdump -d -M intel hello64

# ดู symbol table
objdump -t hello64

# ดู section headers
objdump -h hello64
```

### 9.2 ตัวอย่าง Output ของ objdump

```
$ objdump -d -M intel hello64

hello64:     file format elf64-x86-64

Disassembly of section .text:

0000000000401000 <_start>:
  401000:	b8 01 00 00 00       	mov    eax,0x1
  401005:	bf 01 00 00 00       	mov    edi,0x1
  40100a:	48 be 00 20 40 00 00 	movabs rsi,0x402000
  401011:	00 00 00 
  401014:	ba 0e 00 00 00       	mov    edx,0xe
  401019:	0f 05                	syscall
  40101b:	b8 3c 00 00 00       	mov    eax,0x3c
  401020:	bf 00 00 00 00       	mov    edi,0x0
  401025:	0f 05                	syscall
```

**อ่าน output:**
- `401000:` = ตำแหน่งในหน่วยความจำ (virtual address)
- `b8 01 00 00 00` = machine code (hex bytes)
- `mov eax,0x1` = assembly instruction

---

## 10. การดู ELF Header ด้วย readelf

### 10.1 โครงสร้าง ELF File

```
ELF File Structure:
┌─────────────────┐
│   ELF Header    │ ← ข้อมูลพื้นฐานของไฟล์
├─────────────────┤
│ Program Headers │ ← บอก OS ว่าจะ load โปรแกรมอย่างไร
├─────────────────┤
│  .text section  │ ← โค้ด
├─────────────────┤
│  .data section  │ ← ข้อมูล
├─────────────────┤
│  .bss section   │ ← ข้อมูลว่าง
├─────────────────┤
│ Section Headers │ ← ข้อมูลของแต่ละ section
└─────────────────┘
```

### 10.2 คำสั่ง readelf

```bash
# ดู ELF Header
readelf -h hello64

# ดู Section Headers
readelf -S hello64

# ดู Program Headers (Segments)
readelf -l hello64

# ดู Symbol Table
readelf -s hello64

# ดูทั้งหมด
readelf -a hello64

# ดู hex dump ของ section
readelf -x .data hello64
readelf -x .text hello64
```

### 10.3 ตัวอย่าง ELF Header Output

```
$ readelf -h hello64
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Entry point address:               0x401000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          4096 (bytes into file)
  ...
```

**อธิบาย Magic bytes:**
- `7f 45 4c 46` = `\x7fELF` (ELF signature)
- `02` = 64-bit (01 = 32-bit)
- `01` = little endian (02 = big endian)

---

## 11. Analysis: ทุก Byte ในโปรแกรม Hello World

### 11.1 ดู Machine Code ของโปรแกรม

```bash
# ดู hex dump ของ executable
xxd hello64 | head -50

# หรือ
hexdump -C hello64 | head -50
```

### 11.2 วิเคราะห์ทุก instruction

```
Instruction: mov rax, 1
Machine code: 48 B8 01 00 00 00 00 00 00 00

แยกวิเคราะห์:
48          = REX prefix (บอกว่า operand เป็น 64-bit)
             48 = 0100 1000 binary
             bit 3 (W) = 1 → 64-bit operand
B8          = opcode สำหรับ MOV reg, imm64 (B8+reg)
             B8 = MOV RAX, imm64
01 00 00 00 = value 1 ใน little-endian format
00 00 00 00   (รวม 8 bytes = 64-bit immediate)
```

```
Instruction: mov edi, 1
Machine code: BF 01 00 00 00

แยกวิเคราะห์:
BF          = opcode MOV EDI, imm32
             (ไม่มี REX prefix เพราะ 32-bit zero-extends ไปยัง 64-bit อัตโนมัติ)
01 00 00 00 = value 1 ใน little-endian (4 bytes = 32-bit)
```

```
Instruction: movabs rsi, 0x402000  (address ของ message)
Machine code: 48 BE 00 20 40 00 00 00 00 00

แยกวิเคราะห์:
48          = REX prefix (64-bit)
BE          = opcode MOV RSI, imm64
00 20 40 00 = address 0x402000 ใน little-endian
00 00 00 00   (8 bytes)
```

```
Instruction: syscall
Machine code: 0F 05

แยกวิเคราะห์:
0F 05       = สองไบต์ opcode สำหรับ SYSCALL instruction
             เมื่อ CPU เห็น 0F 05 จะทำ:
             1. เซฟ RIP ลงใน RCX
             2. เซฟ RFLAGS ลงใน R11
             3. กระโดดไปยัง syscall handler ใน kernel
```

### 11.3 ขนาดโปรแกรม Hello World

```bash
$ size hello64
   text    data     bss     dec     hex filename
     39      14       0      53      35 hello64

$ ls -la hello64
-rwxr-xr-x 1 user user 4744 hello64

# โปรแกรมมีขนาด 4744 bytes แต่โค้ดจริงๆ มีแค่ 39 bytes!
# ส่วนที่เหลือเป็น ELF headers และ metadata
```

---

## 12. โปรแกรมรับ Input จาก stdin

### 12.1 โปรแกรมรับชื่อแล้วทักทาย (32-bit)

```nasm
; ========================================
; greet32.asm - รับชื่อแล้วทักทาย (32-bit)
; Compile: nasm -f elf32 greet32.asm -o greet32.o
; Link:    ld -m elf_i386 greet32.o -o greet32
; Run:     ./greet32
; ========================================

global _start

section .data
    ; String สำหรับถามชื่อ
    prompt      db "Enter your name: ", 0
    prompt_len  equ $ - prompt
    
    ; String สำหรับทักทาย
    greeting    db "Hello, "
    greet_len   equ $ - greeting
    
    ; Newline
    newline     db 10
    nl_len      equ 1

section .bss
    ; จองพื้นที่สำหรับรับ input (สูงสุด 64 bytes)
    name_buf    resb 64
    name_len    resd 1          ; เก็บจำนวน bytes ที่อ่านได้

section .text

_start:
    ; -----------------------------------------------
    ; แสดง prompt: "Enter your name: "
    ; sys_write(stdout, prompt, prompt_len)
    ; -----------------------------------------------
    mov eax, 4                  ; sys_write
    mov ebx, 1                  ; stdout
    mov ecx, prompt             ; address ของ prompt string
    mov edx, prompt_len         ; ความยาว
    int 0x80
    
    ; -----------------------------------------------
    ; รับ input จาก keyboard
    ; sys_read(stdin, buffer, max_size)
    ; return: จำนวน bytes ที่อ่านได้ใน EAX
    ; -----------------------------------------------
    mov eax, 3                  ; syscall 3 = sys_read
    mov ebx, 0                  ; stdin = 0
    mov ecx, name_buf           ; buffer สำหรับเก็บ input
    mov edx, 64                 ; อ่านสูงสุด 64 bytes
    int 0x80
    
    ; เซฟจำนวน bytes ที่อ่านได้
    mov [name_len], eax         ; เก็บค่า return ใน name_len
    
    ; -----------------------------------------------
    ; แสดง "Hello, "
    ; -----------------------------------------------
    mov eax, 4
    mov ebx, 1
    mov ecx, greeting
    mov edx, greet_len
    int 0x80
    
    ; -----------------------------------------------
    ; แสดงชื่อที่รับมา
    ; -----------------------------------------------
    mov eax, 4
    mov ebx, 1
    mov ecx, name_buf
    mov edx, [name_len]         ; ใช้ความยาวที่อ่านมาได้จริง
    int 0x80
    
    ; -----------------------------------------------
    ; จบโปรแกรม
    ; -----------------------------------------------
    mov eax, 1
    mov ebx, 0
    int 0x80
```

### 12.2 โปรแกรมรับ Input (64-bit) พร้อม Error Handling

```nasm
; ========================================
; greet64.asm - รับชื่อแล้วทักทาย (64-bit)
; Compile: nasm -f elf64 greet64.asm -o greet64.o
; Link:    ld greet64.o -o greet64
; ========================================

global _start

; ค่าคงที่ (constants)
SYS_READ    equ 0       ; syscall number สำหรับ read
SYS_WRITE   equ 1       ; syscall number สำหรับ write
SYS_EXIT    equ 60      ; syscall number สำหรับ exit
STDIN       equ 0       ; file descriptor stdin
STDOUT      equ 1       ; file descriptor stdout
BUF_SIZE    equ 64      ; ขนาด buffer

section .data
    prompt      db "Enter your name: "
    prompt_len  equ $ - prompt
    
    greeting    db "Hello, "
    greet_len   equ $ - greeting
    
    ; Error message ถ้า read ล้มเหลว
    err_msg     db "Error reading input!", 10
    err_len     equ $ - err_msg

section .bss
    name_buf    resb BUF_SIZE

section .text

_start:
    ; แสดง prompt
    mov rax, SYS_WRITE
    mov rdi, STDOUT
    mov rsi, prompt
    mov rdx, prompt_len
    syscall
    
    ; อ่าน input
    mov rax, SYS_READ
    mov rdi, STDIN
    mov rsi, name_buf
    mov rdx, BUF_SIZE
    syscall
    
    ; ตรวจสอบ error (ถ้า return value < 0 = error)
    test rax, rax           ; ทดสอบว่า rax = 0 หรือไม่
    js .error               ; ถ้า negative (sign flag set) = error
    jz .error               ; ถ้า 0 bytes อ่านได้ = error/EOF
    
    ; เซฟจำนวน bytes ที่อ่านได้
    push rax                ; เก็บ return value ไว้บน stack
    
    ; แสดง "Hello, "
    mov rax, SYS_WRITE
    mov rdi, STDOUT
    mov rsi, greeting
    mov rdx, greet_len
    syscall
    
    ; นำจำนวน bytes กลับมาจาก stack
    pop rdx                 ; จำนวน bytes ที่ต้องแสดง
    
    ; แสดงชื่อ
    mov rax, SYS_WRITE
    mov rdi, STDOUT
    mov rsi, name_buf
    syscall
    
    ; จบโปรแกรมด้วย exit code 0
    mov rax, SYS_EXIT
    mov rdi, 0
    syscall

.error:
    ; แสดง error message
    mov rax, SYS_WRITE
    mov rdi, STDOUT
    mov rsi, err_msg
    mov rdx, err_len
    syscall
    
    ; จบโปรแกรมด้วย exit code 1 (error)
    mov rax, SYS_EXIT
    mov rdi, 1
    syscall
```

---

## 13. โปรแกรม Calculator อย่างง่าย

### 13.1 Calculator รับตัวเลขสองตัวแล้วบวก (64-bit)

```nasm
; ========================================
; calc64.asm - Calculator อย่างง่าย (64-bit)
; รับตัวเลขสองตัวจาก stdin แล้วแสดงผลลัพธ์
; Compile: nasm -f elf64 calc64.asm -o calc64.o
; Link:    ld calc64.o -o calc64
; Run:     ./calc64
; ========================================

global _start

SYS_READ    equ 0
SYS_WRITE   equ 1
SYS_EXIT    equ 60
STDIN       equ 0
STDOUT      equ 1

section .data
    prompt1     db "Enter first number: "
    prompt1_len equ $ - prompt1
    
    prompt2     db "Enter second number: "
    prompt2_len equ $ - prompt2
    
    ; ข้อความผลลัพธ์
    result_msg  db "Sum = "
    result_len  equ $ - result_msg
    
    newline     db 10           ; newline character

section .bss
    buf1        resb 16         ; buffer สำหรับตัวเลขที่ 1
    buf2        resb 16         ; buffer สำหรับตัวเลขที่ 2
    result_buf  resb 20         ; buffer สำหรับผลลัพธ์

section .text

; ========================================
; Function: atoi (ASCII to Integer)
; Input:  RSI = pointer to string, RDX = length
; Output: RAX = integer value
; ========================================
atoi:
    xor rax, rax        ; rax = 0 (ผลลัพธ์)
    xor rcx, rcx        ; rcx = 0 (counter)
    
.loop:
    cmp rcx, rdx        ; ถ้า counter >= length ออกจาก loop
    jge .done
    
    movzx rbx, byte [rsi + rcx]  ; อ่าน byte ที่ตำแหน่ง rsi+rcx
    
    ; ตรวจสอบว่าเป็น newline หรือ null
    cmp rbx, 10         ; 10 = newline
    je  .done
    cmp rbx, 0          ; 0 = null terminator
    je  .done
    
    ; แปลง ASCII เป็นตัวเลข: '0'=48, '1'=49, ... '9'=57
    sub rbx, 48         ; ลบ 48 ('0') เพื่อได้ตัวเลขจริง
    
    ; ตรวจว่าเป็นตัวเลขจริงหรือไม่ (0-9)
    cmp rbx, 0
    jl  .done           ; ถ้าน้อยกว่า 0 = ไม่ใช่ตัวเลข
    cmp rbx, 9
    jg  .done           ; ถ้ามากกว่า 9 = ไม่ใช่ตัวเลข
    
    ; คำนวณ: result = result * 10 + digit
    imul rax, rax, 10   ; rax = rax * 10
    add  rax, rbx       ; rax = rax + digit
    
    inc rcx             ; เพิ่ม counter
    jmp .loop
    
.done:
    ret

; ========================================
; Function: itoa (Integer to ASCII)
; Input:  RAX = integer, RSI = output buffer
; Output: RDX = length of string written
; ========================================
itoa:
    push rax            ; เซฟ registers
    push rbx
    push rcx
    push rdi
    
    mov  rdi, rsi       ; เซฟ pointer ไปยัง buffer
    add  rsi, 19        ; เริ่มเขียนจากท้าย buffer (เขียนย้อนกลับ)
    mov  byte [rsi], 0  ; null terminator
    dec  rsi
    
    ; ถ้า 0 เขียน '0' แล้วจบ
    test rax, rax
    jnz  .convert
    mov  byte [rsi], '0'
    dec  rsi
    jmp  .finish
    
.convert:
    test rax, rax       ; ตรวจว่าหมดแล้วหรือยัง
    jz   .finish
    
    xor  rdx, rdx       ; rdx = 0 (สำหรับ div)
    mov  rbx, 10
    div  rbx            ; rax = rax / 10, rdx = rax % 10
    
    add  rdx, 48        ; แปลงเป็น ASCII: 0+'0'=48, 1+'0'=49, ...
    mov  byte [rsi], dl ; เขียนตัวอักษร
    dec  rsi
    jmp  .convert
    
.finish:
    inc  rsi            ; ปรับ pointer กลับ (เพิ่มขึ้น 1)
    
    ; คำนวณความยาว
    mov  rdx, rdi       ; rdx = start of buffer
    add  rdx, 20        ; rdx = end of buffer
    sub  rdx, rsi       ; rdx = end - current = ความยาว
    dec  rdx            ; ลด 1 เพราะ null terminator
    
    ; ย้าย string ไปยังต้น buffer
    push rsi            ; เซฟ source pointer
    mov  rdi_src, rsi
    
    ; Simple copy loop
    xor  rcx, rcx
.copy:
    cmp  rcx, rdx
    jge  .done_copy
    mov  al, byte [rsi + rcx]
    mov  byte [rdi + rcx], al
    inc  rcx
    jmp  .copy
.done_copy:
    
    pop  rsi
    pop  rdi
    pop  rcx
    pop  rbx
    pop  rax
    ret
```

### 13.2 Calculator แบบง่าย (ใช้ C Library)

เพื่อความง่าย ตัวอย่างนี้ใช้ `printf` และ `scanf` จาก C library:

```nasm
; ========================================
; calc_libc.asm - Calculator ใช้ C library
; Compile: nasm -f elf64 calc_libc.asm -o calc_libc.o
; Link:    gcc calc_libc.o -o calc_libc  (ใช้ gcc เพราะต้องการ libc)
; หรือ:   ld calc_libc.o -o calc_libc -lc -dynamic-linker /lib64/ld-linux-x86-64.so.2
; ========================================

; นำเข้า C functions
extern printf
extern scanf
extern exit

global main

section .data
    ; Format strings สำหรับ printf/scanf
    fmt_prompt1 db "Enter first number: ", 0   ; null-terminated string
    fmt_prompt2 db "Enter second number: ", 0
    fmt_input   db "%d", 0          ; scanf format
    fmt_result  db "Sum = %d", 10, 0 ; printf format
    
    fmt_add     db "Add result: %d + %d = %d", 10, 0
    fmt_sub     db "Sub result: %d - %d = %d", 10, 0
    fmt_mul     db "Mul result: %d * %d = %d", 10, 0

section .bss
    num1        resd 1      ; int num1
    num2        resd 1      ; int num2

section .text

main:
    ; สร้าง stack frame
    push rbp
    mov  rbp, rsp
    sub  rsp, 32        ; จองพื้นที่ใน stack
    
    ; printf("Enter first number: ")
    mov  rdi, fmt_prompt1   ; arg1: format string
    xor  eax, eax           ; rax = 0 (ไม่มี float arguments)
    call printf
    
    ; scanf("%d", &num1)
    mov  rdi, fmt_input     ; arg1: format string
    lea  rsi, [num1]        ; arg2: pointer to num1
    xor  eax, eax
    call scanf
    
    ; printf("Enter second number: ")
    mov  rdi, fmt_prompt2
    xor  eax, eax
    call printf
    
    ; scanf("%d", &num2)
    mov  rdi, fmt_input
    lea  rsi, [num2]
    xor  eax, eax
    call scanf
    
    ; อ่านค่าทั้งสองมา
    mov  eax, [num1]        ; eax = num1
    mov  ebx, [num2]        ; ebx = num2
    
    ; ===== การบวก =====
    mov  ecx, eax           ; ecx = num1
    add  ecx, ebx           ; ecx = num1 + num2
    
    ; printf("Add result: %d + %d = %d\n", num1, num2, sum)
    mov  rdi, fmt_add
    mov  esi, eax           ; arg2: num1
    mov  edx, ebx           ; arg3: num2
    mov  ecx, ecx           ; arg4: sum
    xor  eax, eax
    call printf
    
    ; ===== การลบ =====
    mov  ecx, eax           ; ecx = num1
    sub  ecx, ebx           ; ecx = num1 - num2
    
    mov  rdi, fmt_sub
    mov  esi, eax
    mov  edx, ebx
    ; ecx ยังมีค่า result จากการลบ
    xor  eax, eax
    call printf
    
    ; ===== การคูณ =====
    mov  ecx, eax           ; ecx = num1
    imul ecx, ebx           ; ecx = num1 * num2
    
    mov  rdi, fmt_mul
    mov  esi, eax
    mov  edx, ebx
    xor  eax, eax
    call printf
    
    ; exit(0)
    mov  rdi, 0
    call exit
```

### 13.3 Calculator แบบสมบูรณ์ (Pure Assembly, 64-bit)

```nasm
; ========================================
; calculator.asm - Calculator สมบูรณ์ (Pure Assembly)
; รองรับ: บวก, ลบ, คูณ, หาร
; Compile: nasm -f elf64 calculator.asm -o calculator.o
; Link:    ld calculator.o -o calculator
; ========================================

global _start

SYS_READ    equ 0
SYS_WRITE   equ 1
SYS_EXIT    equ 60
STDIN       equ 0
STDOUT      equ 1

section .data
    ; Menu
    menu        db "=== Assembly Calculator ===", 10
                db "Operations:", 10
                db "  1: Addition (+)", 10
                db "  2: Subtraction (-)", 10
                db "  3: Multiplication (*)", 10
                db "  4: Division (/)", 10
                db "  5: Exit", 10
                db "Choose (1-5): "
    menu_len    equ $ - menu
    
    prompt_a    db "Enter number A: "
    pa_len      equ $ - prompt_a
    
    prompt_b    db "Enter number B: "
    pb_len      equ $ - prompt_b
    
    msg_result  db "Result: "
    mr_len      equ $ - msg_result
    
    msg_divzero db "Error: Division by zero!", 10
    dz_len      equ $ - msg_divzero
    
    newline     db 10

section .bss
    input_buf   resb 32     ; buffer สำหรับรับ input
    output_buf  resb 32     ; buffer สำหรับแสดงผลลัพธ์

section .text

; ===================================================
; _start: entry point
; ===================================================
_start:
.main_loop:
    ; แสดง menu
    call print_menu
    
    ; รับ choice
    call read_number        ; ผลลัพธ์อยู่ใน RAX
    
    ; ตรวจสอบ choice
    cmp rax, 1
    je  .do_add
    cmp rax, 2
    je  .do_sub
    cmp rax, 3
    je  .do_mul
    cmp rax, 4
    je  .do_div
    cmp rax, 5
    je  .exit
    
    jmp .main_loop      ; ถ้าไม่ตรง กลับไปแสดง menu

.do_add:
    call get_two_numbers    ; RAX = num1, RBX = num2
    add  rax, rbx           ; rax = num1 + num2
    call print_result
    jmp  .main_loop

.do_sub:
    call get_two_numbers
    sub  rax, rbx           ; rax = num1 - num2
    call print_result
    jmp  .main_loop

.do_mul:
    call get_two_numbers
    imul rax, rbx           ; rax = num1 * num2
    call print_result
    jmp  .main_loop

.do_div:
    call get_two_numbers
    ; ตรวจสอบ division by zero
    test rbx, rbx
    jz   .div_zero
    
    ; signed divide: rdx:rax / rbx
    cqo                     ; ขยาย rax เป็น rdx:rax (sign extension)
    idiv rbx                ; rax = quotient, rdx = remainder
    call print_result
    jmp  .main_loop

.div_zero:
    ; แสดง error message
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, msg_divzero
    mov  rdx, dz_len
    syscall
    jmp  .main_loop

.exit:
    mov  rax, SYS_EXIT
    mov  rdi, 0
    syscall

; ===================================================
; Function: print_menu
; แสดง menu บนหน้าจอ
; ===================================================
print_menu:
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, menu
    mov  rdx, menu_len
    syscall
    ret

; ===================================================
; Function: read_number
; อ่านตัวเลขจาก stdin
; Output: RAX = number
; ===================================================
read_number:
    ; อ่าน string จาก stdin
    mov  rax, SYS_READ
    mov  rdi, STDIN
    mov  rsi, input_buf
    mov  rdx, 32
    syscall
    
    ; แปลง string เป็นตัวเลข (atoi)
    mov  rsi, input_buf
    mov  rdx, rax           ; จำนวน bytes ที่อ่านได้
    call str_to_int         ; ผลลัพธ์อยู่ใน RAX
    ret

; ===================================================
; Function: get_two_numbers
; ถาม num1 และ num2 จาก user
; Output: RAX = num1, RBX = num2
; ===================================================
get_two_numbers:
    ; แสดง "Enter number A: "
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, prompt_a
    mov  rdx, pa_len
    syscall
    
    ; รับ num1
    call read_number
    push rax                ; เซฟ num1 ไว้บน stack
    
    ; แสดง "Enter number B: "
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, prompt_b
    mov  rdx, pb_len
    syscall
    
    ; รับ num2
    call read_number
    mov  rbx, rax           ; rbx = num2
    
    pop  rax                ; rax = num1
    ret

; ===================================================
; Function: print_result
; แสดงผลลัพธ์
; Input: RAX = result number
; ===================================================
print_result:
    push rax                ; เซฟ result
    
    ; แสดง "Result: "
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, msg_result
    mov  rdx, mr_len
    syscall
    
    pop  rax                ; คืน result
    
    ; แปลงเป็น string
    mov  rsi, output_buf
    call int_to_str         ; RDX = ความยาว string
    
    ; แสดง result
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, output_buf
    syscall                 ; rdx ยังมีค่าจาก int_to_str
    
    ; แสดง newline
    mov  rax, SYS_WRITE
    mov  rdi, STDOUT
    mov  rsi, newline
    mov  rdx, 1
    syscall
    ret

; ===================================================
; Function: str_to_int (atoi)
; Input:  RSI = string pointer, RDX = length
; Output: RAX = integer
; ===================================================
str_to_int:
    xor  rax, rax       ; rax = 0 (ผลลัพธ์)
    xor  rcx, rcx       ; rcx = index
    
    ; ตรวจสอบ negative sign
    xor  r8, r8         ; r8 = sign flag (0 = positive)
    movzx r9, byte [rsi]
    cmp  r9b, '-'
    jne  .atoi_loop
    mov  r8, 1          ; negative
    inc  rcx            ; ข้าม '-'

.atoi_loop:
    cmp  rcx, rdx
    jge  .atoi_done
    
    movzx r9, byte [rsi + rcx]
    
    ; ตรวจสอบ newline/null/space
    cmp  r9b, 10
    je   .atoi_done
    cmp  r9b, 0
    je   .atoi_done
    cmp  r9b, ' '
    je   .atoi_done
    
    ; ตรวจสอบตัวเลข
    sub  r9b, '0'
    cmp  r9b, 0
    jl   .atoi_done
    cmp  r9b, 9
    jg   .atoi_done
    
    imul rax, rax, 10
    add  rax, r9
    
    inc  rcx
    jmp  .atoi_loop

.atoi_done:
    ; ถ้า negative ทำ negation
    test r8, r8
    jz   .atoi_ret
    neg  rax

.atoi_ret:
    ret

; ===================================================
; Function: int_to_str (itoa)
; Input:  RAX = integer, RSI = output buffer
; Output: RDX = length of string, RSI = start of string
; ===================================================
int_to_str:
    push rbx
    push rcx
    push r8
    
    mov  r8, rsi            ; เซฟ buffer start
    
    ; ตรวจสอบ negative
    xor  r9, r9             ; r9 = sign flag
    test rax, rax
    jns  .itoa_positive
    neg  rax                ; ทำ absolute value
    mov  r9, 1              ; negative flag
    
.itoa_positive:
    ; เขียนตัวเลขย้อนกลับ
    mov  rcx, 0             ; counter
    
    ; กรณีพิเศษ: RAX = 0
    test rax, rax
    jnz  .itoa_loop
    mov  byte [rsi], '0'
    inc  rsi
    inc  rcx
    jmp  .itoa_sign
    
.itoa_loop:
    test rax, rax
    jz   .itoa_sign
    
    xor  rdx, rdx
    mov  rbx, 10
    div  rbx                ; rax = rax/10, rdx = rax%10
    
    add  dl, '0'            ; แปลงเป็น ASCII
    mov  byte [rsi], dl
    inc  rsi
    inc  rcx
    jmp  .itoa_loop
    
.itoa_sign:
    ; ถ้า negative เพิ่ม '-'
    test r9, r9
    jz   .itoa_reverse
    mov  byte [rsi], '-'
    inc  rsi
    inc  rcx
    
.itoa_reverse:
    ; string อยู่ในลำดับย้อนกลับ ต้อง reverse
    ; r8 = start, rsi-1 = end
    dec  rsi
    mov  rdi, r8            ; rdi = left pointer
    
.rev_loop:
    cmp  rdi, rsi
    jge  .rev_done
    
    ; swap *rdi และ *rsi
    mov  al, byte [rdi]
    mov  bl, byte [rsi]
    mov  byte [rdi], bl
    mov  byte [rsi], al
    
    inc  rdi
    dec  rsi
    jmp  .rev_loop
    
.rev_done:
    mov  rdx, rcx           ; rdx = ความยาว
    mov  rsi, r8            ; rsi = start of buffer
    
    pop  r8
    pop  rcx
    pop  rbx
    ret
```

---

## 14. ข้อผิดพลาดที่พบบ่อยและวิธีแก้ไข

### 14.1 ตาราง Common Errors

| ข้อผิดพลาด | สาเหตุ | วิธีแก้ |
|-----------|--------|---------|
| `Segmentation fault` | เข้าถึง memory ที่ไม่ได้รับอนุญาต | ตรวจสอบ pointer ว่าถูกต้อง |
| `NASM error: label 'X' undefined` | ใช้ label ที่ยังไม่ได้ประกาศ | ตรวจ spelling ของ label |
| `ld: cannot find -lX` | ไม่พบ library | ติดตั้ง library หรือระบุ path |
| `Illegal instruction` | CPU ไม่รู้จัก instruction | ตรวจสอบว่าใช้ instruction ถูก mode |
| `Bus error` | Alignment error | ตรวจสอบ memory alignment |
| `Stack overflow` | Stack เต็ม | ลด recursion หรือเพิ่ม stack size |
| `Division by zero` | หารด้วย 0 | ตรวจสอบ divisor ก่อน div |

### 14.2 ข้อผิดพลาดใน NASM Syntax

```nasm
; ผิด: ใช้ immediate ขนาดใหญ่เกินไป
mov  al, 256        ; ERROR: al เก็บได้แค่ 0-255

; ถูก:
mov  al, 255        ; OK: max value for byte
mov  eax, 256       ; OK: ใช้ register ใหญ่ขึ้น

; ผิด: ลืม keyword 'byte'/'word'/'dword'
mov  [variable], 5  ; ERROR: ขนาดไม่ชัดเจน

; ถูก:
mov  byte [variable], 5     ; OK: ระบุขนาดชัดเจน
mov  dword [variable], 5    ; OK: ระบุขนาดชัดเจน

; ผิด: ใช้ syscall numbers ผิด
mov  rax, 4         ; ERROR: บน 64-bit, write = 1 ไม่ใช่ 4
syscall

; ถูก:
mov  rax, 1         ; OK: 64-bit sys_write = 1
syscall

; ผิด: ลืม global _start
section .text
_start:             ; linker จะ error: _start not found

; ถูก:
section .text
global _start       ; ต้องประกาศ global ก่อน
_start:
```

### 14.3 ข้อผิดพลาดเกี่ยวกับ Memory

```nasm
; ผิด: อ้างอิง memory address ผิด
mov  eax, [5]       ; ERROR: ไม่ใช่ตัวเลขตรงๆ แต่เป็น memory address

; ถูก (ถ้าต้องการค่า 5):
mov  eax, 5         ; OK: immediate value

; ถูก (ถ้าต้องการอ่านจาก address 5):
mov  eax, [5]       ; เป็น valid syntax แต่ likely จะ segfault

; ผิด: ลืม section
mov  eax, [variable] ; ERROR ถ้า variable ไม่ได้ประกาศใน .data

; ถูก:
section .data
variable dd 42      ; ประกาศก่อน
section .text
mov  eax, [variable] ; OK
```

---

## 15. แบบฝึกหัด (Exercises)

### Exercise 1: Hello in Different Languages (ง่าย)

**โจทย์:** เขียนโปรแกรมแสดงข้อความทักทายทั้งภาษาไทยและอังกฤษ

```nasm
; โครงสร้างที่ต้องกรอก:
global _start

section .data
    ; TODO: เพิ่ม strings ทั้งสองภาษา
    ; "สวัสดีชาวโลก!" (ต้องใช้ UTF-8 encoding)
    ; "Hello, World!"

section .text

_start:
    ; TODO: แสดงทั้งสอง strings
    ; TODO: exit(0)
```

**เฉลย:**

```nasm
global _start

section .data
    ; UTF-8 encoding ของ "สวัสดีชาวโลก!\n"
    thai_msg    db 0xE0,0xB8,0xAA,0xE0,0xB8,0xA7,0xE0,0xB8,0xB1,0xE0,0xB8,0xAA
                db 0xE0,0xB8,0x94,0xE0,0xB8,0xB5,0xE0,0xB8,0x8A,0xE0,0xB8,0xB2
                db 0xE0,0xB8,0xA7,0xE0,0xB9,0x82,0xE0,0xB8,0xA5,0xE0,0xB8,0x81
                db "!", 10
    thai_len    equ $ - thai_msg
    
    eng_msg     db "Hello, World!", 10
    eng_len     equ $ - eng_msg

section .text
global _start

_start:
    ; แสดงภาษาไทย
    mov  rax, 1
    mov  rdi, 1
    mov  rsi, thai_msg
    mov  rdx, thai_len
    syscall
    
    ; แสดงภาษาอังกฤษ
    mov  rax, 1
    mov  rdi, 1
    mov  rsi, eng_msg
    mov  rdx, eng_len
    syscall
    
    ; จบโปรแกรม
    mov  rax, 60
    xor  rdi, rdi
    syscall
```

---

### Exercise 2: แสดงตัวเลข 1-10 (กลาง)

**โจทย์:** เขียนโปรแกรมแสดงตัวเลข 1 ถึง 10 ด้วย loop

```nasm
; โครงสร้างที่ต้องกรอก:
global _start

section .data
    ; TODO: เพิ่ม newline string

section .bss
    ; TODO: จอง buffer สำหรับตัวเลข

section .text

_start:
    ; TODO: วน loop จาก 1 ถึง 10
    ;       แสดงแต่ละตัวเลขพร้อม newline
```

**เฉลย:**

```nasm
global _start

section .data
    newline db 10
    
section .bss
    num_buf resb 4

section .text

_start:
    mov  rcx, 1         ; counter = 1
    
.loop:
    cmp  rcx, 11        ; ถ้า counter > 10 หยุด
    jge  .done
    
    ; แปลงตัวเลขเป็น ASCII string
    ; สำหรับ 1-9: เพิ่ม '0' (48)
    ; สำหรับ 10: พิเศษ
    
    push rcx            ; เซฟ counter
    
    cmp  rcx, 10
    je   .print_10
    
    ; 1-9: แค่เพิ่ม '0'
    mov  rax, rcx
    add  al, '0'        ; แปลงเป็น ASCII
    mov  [num_buf], al
    
    ; แสดงตัวเลข
    mov  rax, 1
    mov  rdi, 1
    mov  rsi, num_buf
    mov  rdx, 1
    syscall
    jmp  .print_nl
    
.print_10:
    ; แสดง "10" สองตัวอักษร
    mov  byte [num_buf], '1'
    mov  byte [num_buf+1], '0'
    
    mov  rax, 1
    mov  rdi, 1
    mov  rsi, num_buf
    mov  rdx, 2
    syscall
    
.print_nl:
    ; แสดง newline
    mov  rax, 1
    mov  rdi, 1
    mov  rsi, newline
    mov  rdx, 1
    syscall
    
    pop  rcx
    inc  rcx
    jmp  .loop
    
.done:
    mov  rax, 60
    xor  rdi, rdi
    syscall
```

---

### Exercise 3: โปรแกรมนับ Characters (กลาง)

**โจทย์:** เขียนโปรแกรมรับ string และนับจำนวน characters

**ตัวอย่าง Output:**
```
Enter text: Hello World
Character count: 12
```

```nasm
; โครงสร้างที่ต้องกรอก:
global _start

section .data
    prompt  db "Enter text: "
    p_len   equ $ - prompt
    result  db "Character count: "
    r_len   equ $ - result

section .bss
    buf     resb 256
    
section .text

_start:
    ; TODO: แสดง prompt
    ; TODO: อ่าน input (return value = จำนวน bytes รวม newline)
    ; TODO: ลด 1 (เพื่อไม่นับ newline)
    ; TODO: แสดง result message
    ; TODO: แปลงจำนวนเป็น string แล้วแสดง
    ; TODO: exit(0)
```

---

### Exercise 4: Reverse String (ยาก)

**โจทย์:** เขียนโปรแกรมรับ string แล้วแสดงกลับหัว

**ตัวอย่าง:**
```
Enter text: Hello
Reversed: olleH
```

---

### Exercise 5: เปรียบเทียบสองตัวเลข (กลาง)

**โจทย์:** เขียนโปรแกรมรับตัวเลข 2 ตัว แล้วบอกว่าตัวไหนมากกว่า หรือเท่ากัน

**ตัวอย่าง Output:**
```
Enter number A: 15
Enter number B: 8
15 is greater than 8
```

---

## 16. สรุปและ Key Takeaways

### สิ่งที่ได้เรียนรู้ใน Part นี้

1. **โครงสร้าง NASM Program:**
   - `.data` section สำหรับ initialized data
   - `.bss` section สำหรับ uninitialized data
   - `.text` section สำหรับโค้ด

2. **System Calls:**
   - Linux 32-bit: `int 0x80` (syscall numbers ต่างกัน)
   - Linux 64-bit: `syscall` instruction (sys_write=1, sys_exit=60)
   - Windows: ใช้ Win32 API ผ่าน DLL functions

3. **Compilation Pipeline:**
   ```
   .asm → (NASM) → .o → (ld) → executable
   ```

4. **Tools ที่ใช้:**
   - `nasm`: Assembler
   - `ld`: Linker
   - `gdb`: Debugger
   - `objdump`: Disassembler
   - `readelf`: ELF analyzer

5. **NASM vs GAS Syntax:**
   - NASM ใช้ Intel syntax (dst, src)
   - GAS ใช้ AT&T syntax (src, dst) พร้อม prefix `%` และ `$`

6. **การวิเคราะห์ Machine Code:**
   - REX prefix สำหรับ 64-bit operands
   - Opcodes แต่ละ instruction มีขนาดต่างกัน
   - Little-endian สำหรับ immediate values

### สูตรจำสำหรับ System Calls ที่ใช้บ่อย

```
Linux 32-bit (int 0x80):
  write: EAX=4, EBX=fd, ECX=buf, EDX=len
  read:  EAX=3, EBX=fd, ECX=buf, EDX=len
  exit:  EAX=1, EBX=code

Linux 64-bit (syscall):
  write: RAX=1, RDI=fd, RSI=buf, RDX=len
  read:  RAX=0, RDI=fd, RSI=buf, RDX=len
  exit:  RAX=60, RDI=code
```

---

## 17. References และแหล่งข้อมูลเพิ่มเติม

### เอกสารอ้างอิง

1. **NASM Manual:** https://www.nasm.us/doc/
2. **Linux Syscall Reference (32-bit):** https://syscalls.kernelgrok.com/
3. **Linux Syscall Reference (64-bit):** https://filippo.io/linux-syscall-table/
4. **Intel x86 Manual:** https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
5. **ELF Format Specification:** https://refspecs.linuxbase.org/elf/elf.pdf

### Tools ที่ควรติดตั้ง

```bash
# Ubuntu/Debian
sudo apt-get install nasm gdb binutils

# Fedora/RHEL
sudo dnf install nasm gdb binutils

# ตรวจสอบ versions
nasm --version
ld --version
gdb --version
objdump --version
readelf --version
```

### ขั้นตอนถัดไป (Part 009)

ใน Part ถัดไปเราจะเรียนรู้:
- Arithmetic Operations เชิงลึก
- Flags และ Conditional Jumps
- Integer Overflow และ Carry
- การใช้ SIMD instructions

---

*Part 008 จบแล้ว - ยินดีด้วยที่ได้เขียนโปรแกรม Assembly แรก!*
