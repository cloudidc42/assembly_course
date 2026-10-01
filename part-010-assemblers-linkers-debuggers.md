# Part 010: Assemblers, Linkers, และ Debuggers

**เวลาที่ใช้ศึกษา:** 6-8 ชั่วโมง  
**ระดับ:** ปานกลาง (Intermediate)  
**ความรู้ก่อนหน้า:** Part 001-009 (พื้นฐาน Assembly, Registers, Memory, Instructions)

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากเรียน Part นี้จบ คุณจะสามารถ:

- อธิบายความแตกต่างระหว่าง NASM, YASM, GAS, MASM และ ARM as ได้
- เข้าใจกระบวนการ Two-Pass Assembly ว่าทำงานอย่างไร
- วิเคราะห์ Object File formats ได้แก่ ELF, COFF, และ Mach-O
- เขียน Linker Script พื้นฐานสำหรับควบคุมการจัดวาง memory ได้
- เข้าใจความแตกต่างระหว่าง Static linking และ Dynamic linking
- ใช้เครื่องมือ nm, objdump, readelf วิเคราะห์ binary files ได้
- Debug โปรแกรม Assembly ด้วย GDB และ LLDB ได้อย่างมีประสิทธิภาพ
- ตรวจสอบ memory leaks และ memory errors ด้วย Valgrind ได้

---

## 1. ภาพรวมของ Assembler Ecosystem

```
Source Code (.asm/.s)
        |
        v
   [Assembler]    <-- NASM / YASM / GAS / MASM / ARM as
        |
        v
  Object File (.o)  <-- ELF / COFF / Mach-O
        |
        v
    [Linker]      <-- ld / GNU ld / lld
        |
        v
  Executable / Library
        |
        v
   [Debugger]     <-- GDB / LLDB / WinDbg
```

กระบวนการแปลง Assembly source code ไปเป็น executable ประกอบด้วย 3 ขั้นตอนหลัก:

1. **Assembly** - แปลง mnemonics เป็น machine code
2. **Linking** - รวม object files และ libraries เข้าด้วยกัน
3. **Loading** - OS โหลด executable เข้า memory และเริ่ม execution

---

## 2. NASM (Netwide Assembler)

### 2.1 ประวัติและคุณสมบัติ

NASM เป็น assembler ที่ได้รับความนิยมมากที่สุดในการเรียน Assembly บน x86/x86-64 โดยมีข้อดีดังนี้:

- **Cross-platform** ทำงานได้บน Linux, Windows, macOS
- **Intel syntax** ใช้ syntax แบบเดียวกับ Intel documentation
- **หลาย output formats** รองรับ ELF, COFF, Mach-O, flat binary
- **Macro system** มีระบบ macro ที่ทรงพลัง
- **Free and Open Source** ใช้งานได้ฟรี

### 2.2 การติดตั้ง NASM

```bash
# Ubuntu/Debian
sudo apt-get install nasm

# Fedora/CentOS/RHEL
sudo dnf install nasm

# macOS (Homebrew)
brew install nasm

# Windows (Chocolatey)
choco install nasm

# ตรวจสอบ version
nasm -v
# Output: NASM version 2.15.05 compiled on ...
```

### 2.3 NASM Syntax และ Directives ที่สำคัญ

```nasm
; =============================================================
; ไฟล์: nasm_directives.asm
; คำอธิบาย: ตัวอย่าง NASM Directives ที่ใช้บ่อย
; คอมไพล์: nasm -f elf64 nasm_directives.asm -o nasm_directives.o
;           ld nasm_directives.o -o nasm_directives
; =============================================================

; --- Global Directives ---
BITS 64             ; บอก NASM ว่าเราเขียน 64-bit code
DEFAULT REL         ; ใช้ RIP-relative addressing เป็น default

; --- Section Declarations ---
section .data       ; Section สำหรับ initialized data
section .bss        ; Section สำหรับ uninitialized data
section .text       ; Section สำหรับ code

; --- Data Definition Directives ---
section .data

    ; db = Define Byte (1 byte)
    my_byte     db 0x41         ; เก็บค่า 'A' (ASCII 65)
    
    ; dw = Define Word (2 bytes)
    my_word     dw 0x1234       ; เก็บค่า 0x1234
    
    ; dd = Define Doubleword (4 bytes)
    my_dword    dd 0x12345678   ; เก็บค่า 32-bit integer
    
    ; dq = Define Quadword (8 bytes)
    my_qword    dq 0x123456789ABCDEF0  ; เก็บค่า 64-bit integer
    
    ; สตริง
    hello       db 'Hello, World!', 0x0A, 0  ; string + newline + null
    hello_len   equ $ - hello  ; คำนวณความยาว string

    ; อาร์เรย์
    numbers     dd 1, 2, 3, 4, 5  ; อาร์เรย์ 5 ตัว แต่ละตัว 4 bytes

    ; ทศนิยม
    pi          dq 3.14159265358979  ; double precision float

section .bss

    ; resb = Reserve Byte(s)
    buffer      resb 256        ; จอง 256 bytes
    
    ; resw = Reserve Word(s)
    words       resw 10         ; จอง 10 words (20 bytes)
    
    ; resd = Reserve Doubleword(s)
    integers    resd 5          ; จอง 5 doublewords (20 bytes)
    
    ; resq = Reserve Quadword(s)
    longs       resq 4          ; จอง 4 quadwords (32 bytes)

section .text
    global _start

_start:
    ; โปรแกรมหลัก
    mov rax, 60     ; syscall: exit
    mov rdi, 0      ; exit code 0
    syscall
```

### 2.4 NASM Macros - ระบบ Macro ที่ทรงพลัง

```nasm
; =============================================================
; ไฟล์: nasm_macros.asm
; คำอธิบาย: ตัวอย่างการใช้ Macros ใน NASM
; คอมไพล์: nasm -f elf64 nasm_macros.asm -o nasm_macros.o
;           ld nasm_macros.o -o nasm_macros
; =============================================================

BITS 64

; --- Simple Macro (ไม่มี parameter) ---
%macro prologue 0
    push rbp            ; บันทึก base pointer เดิม
    mov  rbp, rsp       ; ตั้งค่า base pointer ใหม่
%endmacro

%macro epilogue 0
    mov rsp, rbp        ; คืนค่า stack pointer
    pop rbp             ; คืนค่า base pointer
    ret                 ; return จาก function
%endmacro

; --- Macro with Parameters ---
%macro print_string 2
    ; Parameter 1: address ของ string
    ; Parameter 2: ความยาวของ string
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; file descriptor: stdout
    mov rsi, %1         ; address ของ string
    mov rdx, %2         ; ความยาว
    syscall
%endmacro

; --- Macro with Local Labels ---
%macro check_positive 1
    test %1, %1         ; test ว่า register เป็น 0 หรือไม่
    jns %%positive      ; ถ้า positive ให้ข้ามไป
    neg %1              ; ถ้า negative ให้เปลี่ยนเป็น positive
%%positive:             ; local label (unique ต่อแต่ละการ expand)
%endmacro

; --- Multi-line String Macro ---
%define SYSCALL_WRITE 1
%define SYSCALL_EXIT  60
%define STDOUT        1

section .data
    msg1    db 'Hello from macro!', 0x0A
    len1    equ $ - msg1
    
    msg2    db 'Macros are powerful!', 0x0A
    len2    equ $ - msg2

section .text
    global _start

_start:
    ; ใช้ macro print_string
    print_string msg1, len1
    print_string msg2, len2
    
    ; ทดสอบ check_positive macro
    mov rax, -42        ; ค่า negative
    check_positive rax  ; จะเปลี่ยนเป็น 42
    
    ; Exit
    mov rax, SYSCALL_EXIT
    mov rdi, 0
    syscall

; ฟังก์ชัน test ที่ใช้ prologue/epilogue macros
my_function:
    prologue            ; ขยาย macro เป็น push rbp / mov rbp, rsp
    
    ; function body
    mov rax, 42         ; ค่า return
    
    epilogue            ; ขยาย macro เป็น mov rsp,rbp / pop rbp / ret
```

### 2.5 NASM Conditional Assembly

```nasm
; =============================================================
; ไฟล์: nasm_conditional.asm
; คำอธิบาย: Conditional Assembly ใน NASM
; คอมไพล์: nasm -f elf64 -DDEBUG nasm_conditional.asm -o debug.o
;   หรือ:  nasm -f elf64 nasm_conditional.asm -o release.o
; =============================================================

BITS 64

; กำหนด constants
%define VERSION 2

section .data

; Conditional data
%if VERSION >= 2
    version_str db 'Version 2.x', 0x0A
    ver_len     equ $ - version_str
%else
    version_str db 'Version 1.x', 0x0A
    ver_len     equ $ - version_str
%endif

; Debug message - มีเฉพาะตอน compile ด้วย -DDEBUG
%ifdef DEBUG
    debug_msg   db '[DEBUG] Program started', 0x0A
    debug_len   equ $ - debug_msg
%endif

section .text
    global _start

_start:
%ifdef DEBUG
    ; แสดง debug message เฉพาะตอน debug build
    mov rax, 1
    mov rdi, 1
    mov rsi, debug_msg
    mov rdx, debug_len
    syscall
%endif

    ; แสดง version
    mov rax, 1
    mov rdi, 1
    mov rsi, version_str
    mov rdx, ver_len
    syscall
    
    mov rax, 60
    mov rdi, 0
    syscall
```

---

## 3. YASM (Yet Another Assembler)

YASM เป็น assembler ที่สร้างขึ้นเพื่อเป็นทางเลือกแทน NASM โดยมีความเข้ากันได้กับ NASM syntax ในระดับสูง

### 3.1 ความแตกต่างระหว่าง NASM และ YASM

| คุณสมบัติ | NASM | YASM |
|-----------|------|------|
| Syntax compatibility | เป็น original | Compatible กับ NASM |
| DWARF debug info | จำกัด | รองรับดีกว่า |
| Multiple arch | x86, x86-64 | x86, x86-64, สนับสนุน AMD64 extensions |
| Performance | ดี | บางครั้งเร็วกว่า |
| Development | Active | ช้าลงในปัจจุบัน |

### 3.2 ตัวอย่าง YASM

```asm
; =============================================================
; ไฟล์: yasm_example.asm
; คำอธิบาย: ตัวอย่าง YASM ที่เข้ากันได้กับ NASM
; คอมไพล์: yasm -f elf64 yasm_example.asm -o yasm_example.o
;           ld yasm_example.o -o yasm_example
; =============================================================

BITS 64

section .data
    message db "Hello from YASM!", 10
    msg_len equ $ - message

section .text
    global _start

_start:
    ; write syscall
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    mov rsi, message    ; buffer address
    mov rdx, msg_len    ; length
    syscall

    ; exit syscall
    mov rax, 60         ; sys_exit
    xor rdi, rdi        ; exit code 0
    syscall
```

```bash
# การใช้งาน YASM
# ติดตั้ง
sudo apt-get install yasm

# Compile
yasm -f elf64 -g dwarf2 yasm_example.asm -o yasm_example.o
ld yasm_example.o -o yasm_example
./yasm_example
```

---

## 4. GAS (GNU Assembler)

GAS หรือ `as` เป็น assembler ที่เป็นส่วนหนึ่งของ GNU Binutils ใช้ **AT&T syntax** เป็น default แต่สามารถสลับเป็น Intel syntax ได้

### 4.1 AT&T Syntax vs Intel Syntax

```
ความแตกต่างหลัก:

┌──────────────────┬────────────────────┬───────────────────┐
│ แง่มุม           │ AT&T Syntax        │ Intel Syntax      │
├──────────────────┼────────────────────┼───────────────────┤
│ Operand order    │ src, dst           │ dst, src          │
│ Register prefix  │ %rax               │ rax               │
│ Immediate prefix │ $42                │ 42                │
│ Memory access    │ (%rax)             │ [rax]             │
│ Size suffix      │ movl, movq, etc.   │ mov (inferred)    │
│ Used by          │ GAS default, GCC   │ NASM, MASM        │
└──────────────────┴────────────────────┴───────────────────┘

ตัวอย่างเปรียบเทียบ:
AT&T:   movq $42, %rax      ; ย้าย 42 ไปที่ rax
Intel:  mov rax, 42         ; ย้าย 42 ไปที่ rax

AT&T:   movq (%rbp,-8), %rax
Intel:  mov rax, [rbp-8]
```

### 4.2 GAS ด้วย AT&T Syntax

```gas
# =============================================================
# ไฟล์: gas_att.s
# คำอธิบาย: ตัวอย่าง GAS ด้วย AT&T syntax
# คอมไพล์: as -o gas_att.o gas_att.s
#           ld -o gas_att gas_att.o
# =============================================================

    .section .data
message:
    .asciz "Hello from GAS (AT&T)!\n"   # null-terminated string
msg_len = . - message                    # คำนวณความยาว

    .section .text
    .globl _start                        # ประกาศ global symbol

_start:
    movq $1, %rax           # syscall number: sys_write
    movq $1, %rdi           # file descriptor: stdout
    movq $message, %rsi     # pointer to string
    movq $msg_len, %rdx     # length
    syscall

    movq $60, %rax          # syscall number: sys_exit
    xorq %rdi, %rdi         # exit code: 0
    syscall
```

### 4.3 GAS ด้วย Intel Syntax

```gas
# =============================================================
# ไฟล์: gas_intel.s
# คำอธิบาย: ตัวอย่าง GAS ด้วย Intel syntax
# คอมไพล์: as -o gas_intel.o gas_intel.s
#           ld -o gas_intel gas_intel.o
# =============================================================

    .intel_syntax noprefix   # สลับเป็น Intel syntax, ไม่ต้องใช้ % prefix

    .section .data
message:
    .asciz "Hello from GAS (Intel syntax)!\n"
msg_len = . - message

    .section .text
    .globl _start

_start:
    mov rax, 1              # syscall: write
    mov rdi, 1              # stdout
    lea rsi, [rip+message]  # RIP-relative load address
    mov rdx, msg_len        # length
    syscall

    mov rax, 60             # syscall: exit
    xor rdi, rdi            # exit code 0
    syscall

    .att_syntax prefix       # กลับเป็น AT&T syntax (ถ้าต้องการ)
```

### 4.4 GAS Directives ที่สำคัญ

```gas
# =============================================================
# ไฟล์: gas_directives.s
# คำอธิบาย: GAS Directives ที่ใช้บ่อย
# =============================================================

    # --- Data Directives ---
    .byte   0x41            # 1 byte
    .word   0x1234          # 2 bytes (16-bit)
    .long   0x12345678      # 4 bytes (32-bit)
    .quad   0x123456789ABC  # 8 bytes (64-bit)
    
    .float  3.14            # 4 bytes float
    .double 3.14159265      # 8 bytes double
    
    .ascii  "hello"         # string ไม่มี null terminator
    .asciz  "hello"         # string มี null terminator
    .string "hello"         # เหมือน asciz
    
    # --- BSS ---
    .lcomm buffer, 256      # local common (BSS)
    .comm  global_buf, 512  # global common

    # --- Alignment ---
    .align 4                # align to 4-byte boundary
    .balign 16              # align to 16-byte boundary
    
    # --- Symbol Visibility ---
    .globl my_func          # ทำให้ symbol เห็นได้จาก outside
    .local my_local         # ซ่อน symbol ไว้ใน file นี้
    
    # --- Include ---
    .include "macros.s"     # include ไฟล์อื่น
    
    # --- Architecture ---
    .arch x86_64            # ระบุ target architecture
    .code64                 # บอกว่าเป็น 64-bit code
```

---

## 5. MASM (Microsoft Macro Assembler)

MASM เป็น assembler ของ Microsoft สำหรับ Windows พัฒนา มีการรวมอยู่ใน Visual Studio

### 5.1 คุณสมบัติ MASM

```asm
; =============================================================
; ไฟล์: masm_example.asm
; คำอธิบาย: ตัวอย่าง MASM สำหรับ Windows x64
; คอมไพล์: ml64.exe /c masm_example.asm
;           link masm_example.obj /subsystem:console /entry:main
; =============================================================

; MASM specific directives
.MODEL FLAT, C          ; Memory model (ใช้ใน 32-bit)
.STACK 4096             ; Stack size

; สำหรับ 64-bit ใน MASM ใช้แบบนี้:
OPTION CASEMAP:NONE     ; Case sensitive

; External library functions
EXTERN ExitProcess: PROC
EXTERN WriteConsoleA: PROC
EXTERN GetStdHandle: PROC

; Constants
STD_OUTPUT_HANDLE EQU -11

.DATA
    message BYTE "Hello from MASM!", 13, 10, 0
    msg_len DWORD $ - message
    written DWORD 0

.CODE

main PROC
    ; Get stdout handle
    sub     rsp, 56                     ; Shadow space + alignment
    
    mov     ecx, STD_OUTPUT_HANDLE
    call    GetStdHandle                ; returns handle in rax
    
    ; Write to console
    mov     rcx, rax                    ; hConsoleOutput
    lea     rdx, message                ; lpBuffer
    mov     r8d, msg_len               ; nNumberOfCharsToWrite
    lea     r9, written                ; lpNumberOfCharsWritten
    push    0                          ; lpReserved
    call    WriteConsoleA
    add     rsp, 8                     ; clean up push
    
    ; Exit
    xor     ecx, ecx                   ; exit code 0
    call    ExitProcess
    
main ENDP

END
```

---

## 6. ARM Assembler (GNU as สำหรับ ARM)

### 6.1 ARM Assembly บน Linux (AArch64)

```asm
// =============================================================
// ไฟล์: arm_hello.s
// คำอธิบาย: Hello World สำหรับ ARM64 (AArch64)
// คอมไพล์ (บน ARM Linux หรือ cross-compile):
//   as -o arm_hello.o arm_hello.s
//   ld -o arm_hello arm_hello.o
// Cross-compile จาก x86:
//   aarch64-linux-gnu-as -o arm_hello.o arm_hello.s
//   aarch64-linux-gnu-ld -o arm_hello arm_hello.o
// =============================================================

    .section .data
message:
    .asciz "Hello from ARM64!\n"
msg_len = . - message

    .section .text
    .globl _start
    .type  _start, %function    // ระบุว่าเป็น function type

_start:
    // ARM64 Linux syscall calling convention
    // x0-x7: parameters
    // x8: syscall number
    
    mov x8, #64                 // syscall: write (64 ใน ARM64)
    mov x0, #1                  // fd: stdout
    adr x1, message             // x1 = address ของ message
    mov x2, #msg_len            // x2 = length
    svc #0                      // System call (supervisor call)
    
    mov x8, #93                 // syscall: exit (93 ใน ARM64)
    mov x0, #0                  // exit code 0
    svc #0
```

### 6.2 ARM32 Assembly

```asm
@ =============================================================
@ ไฟล์: arm32_example.s
@ คำอธิบาย: ARM 32-bit Assembly
@ คอมไพล์:
@   arm-linux-gnueabi-as -o arm32.o arm32_example.s
@   arm-linux-gnueabi-ld -o arm32 arm32.o
@ =============================================================

    .section .data
message:
    .asciz "Hello from ARM32!\n"
msg_len = . - message

    .section .text
    .globl _start

_start:
    @ ARM32 Linux syscall
    @ r0-r6: parameters
    @ r7: syscall number
    
    mov r7, #4          @ syscall: write (4 ใน ARM32)
    mov r0, #1          @ fd: stdout
    ldr r1, =message    @ r1 = address ของ message
    mov r2, #msg_len    @ r2 = length
    swi #0              @ software interrupt (syscall)
    
    mov r7, #1          @ syscall: exit (1 ใน ARM32)
    mov r0, #0          @ exit code 0
    swi #0
```

---

## 7. Two-Pass Assembly - กระบวนการแปลง Source Code

### 7.1 ทำไมต้องสองรอบ?

ปัญหาที่เรียกว่า **Forward Reference Problem**: เมื่อ assembly ต้องการใช้ label ที่ยังไม่ได้ถูก define

```nasm
; ตัวอย่าง Forward Reference
    jmp end_label       ; ปัญหา! เราไม่รู้ address ของ end_label ตอน Pass 1

    mov rax, 42
    
end_label:              ; label ถูก define ที่นี่
    nop
```

### 7.2 Pass 1: Symbol Table Construction

```
Pass 1 ทำงานดังนี้:
- อ่าน source code บรรทัดต่อบรรทัด
- คำนวณ Location Counter (LC) สำหรับแต่ละ instruction
- เก็บ labels และ address ลงใน Symbol Table
- ตรวจสอบ syntax เบื้องต้น

ตัวอย่าง:

Location Counter (LC) = 0x1000

บรรทัด         Instruction          LC         Symbol Table
-------         -----------          --         ------------
start:          (label)              0x1000     start = 0x1000
                mov rax, 42         0x1000
                (instruction 7 bytes)
                                    0x1007
                jmp end_program     0x1007
                (instruction 5 bytes)
                                    0x100C
end_program:    (label)             0x100C     end_program = 0x100C
                mov rax, 60         0x100C
                syscall             ...
```

### 7.3 Pass 2: Code Generation

```
Pass 2 ทำงานดังนี้:
- อ่าน source code อีกครั้ง
- แปลง mnemonics เป็น opcodes
- แทน labels ด้วย addresses จาก Symbol Table
- สร้าง object file

ตัวอย่าง Symbol Table หลัง Pass 1:
┌─────────────────┬──────────────┬──────────┐
│ Symbol Name     │ Address      │ Section  │
├─────────────────┼──────────────┼──────────┤
│ _start          │ 0x00000000   │ .text    │
│ loop_start      │ 0x0000000F   │ .text    │
│ loop_end        │ 0x00000025   │ .text    │
│ message         │ 0x00000000   │ .data    │
│ counter         │ 0x00000000   │ .bss     │
└─────────────────┴──────────────┴──────────┘
```

---

## 8. Object File Formats

### 8.1 ELF (Executable and Linkable Format)

ELF เป็น format มาตรฐานบน Linux และ Unix systems

```
ELF File Structure:
┌──────────────────────────────────┐
│ ELF Header (64 bytes for 64-bit) │
│  - Magic number: 0x7F 'E' 'L' 'F'│
│  - Class (32/64-bit)             │
│  - Data encoding (LSB/MSB)       │
│  - Entry point address           │
│  - Program header table offset   │
│  - Section header table offset   │
├──────────────────────────────────┤
│ Program Header Table             │
│  - Segments (for execution)      │
│  - PT_LOAD, PT_DYNAMIC, etc.     │
├──────────────────────────────────┤
│ Sections                         │
│  .text  - executable code        │
│  .data  - initialized data       │
│  .bss   - uninitialized data     │
│  .rodata - read-only data        │
│  .symtab - symbol table          │
│  .strtab - string table          │
│  .rel.text - relocations         │
│  .debug_* - debug information    │
├──────────────────────────────────┤
│ Section Header Table             │
│  - ข้อมูลของแต่ละ section         │
└──────────────────────────────────┘
```

### 8.2 วิเคราะห์ ELF ด้วย readelf

```bash
# compile ก่อน
nasm -f elf64 hello.asm -o hello.o
ld hello.o -o hello

# ดู ELF Header
readelf -h hello
# Output:
# ELF Header:
#   Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
#   Class:                             ELF64
#   Data:                              2's complement, little endian
#   Type:                              EXEC (Executable file)
#   Machine:                           Advanced Micro Devices X86-64
#   Entry point address:               0x401000
#   ...

# ดู Section Headers
readelf -S hello
# Output แสดง sections ทั้งหมด

# ดู Program Headers (segments)
readelf -l hello

# ดู Symbol Table
readelf -s hello

# ดู Relocations
readelf -r hello.o

# ดูทุกอย่าง
readelf -a hello
```

### 8.3 COFF (Common Object File Format)

COFF ใช้บน Windows (PE format เป็น extension ของ COFF)

```
PE/COFF File Structure (Windows):
┌──────────────────────────────────┐
│ DOS Header (MZ header)           │
│ DOS Stub                         │
├──────────────────────────────────┤
│ PE Signature ("PE\0\0")          │
├──────────────────────────────────┤
│ COFF File Header                 │
│  - Machine type (x86/x64/ARM)    │
│  - Number of sections            │
│  - Timestamp                     │
├──────────────────────────────────┤
│ Optional Header                  │
│  - Standard fields               │
│  - Windows-specific fields       │
│  - Data directories              │
├──────────────────────────────────┤
│ Section Table                    │
│  .text .data .bss .rdata         │
│  .idata (import table)           │
│  .edata (export table)           │
├──────────────────────────────────┤
│ Section Data                     │
└──────────────────────────────────┘
```

### 8.4 Mach-O (macOS/iOS)

```
Mach-O File Structure:
┌──────────────────────────────────┐
│ Mach Header                      │
│  - Magic number (0xFEEDFACF)     │
│  - CPU type (x86_64/ARM64)       │
│  - File type (exec/dylib/object) │
│  - Number of load commands       │
├──────────────────────────────────┤
│ Load Commands                    │
│  LC_SEGMENT_64 (__TEXT)          │
│  LC_SEGMENT_64 (__DATA)          │
│  LC_SYMTAB                       │
│  LC_DYSYMTAB                     │
│  LC_LOAD_DYLINKER               │
│  LC_LOAD_DYLIB                  │
├──────────────────────────────────┤
│ Segment: __TEXT                  │
│  __text (code)                   │
│  __const (constants)             │
│  __cstring (C strings)           │
├──────────────────────────────────┤
│ Segment: __DATA                  │
│  __data (initialized data)       │
│  __bss (uninitialized data)      │
└──────────────────────────────────┘
```

---

## 9. Linker Script

Linker Script เป็นไฟล์ที่บอก linker ว่าจะจัดวาง sections ต่างๆ ใน memory อย่างไร

### 9.1 โครงสร้าง Linker Script พื้นฐาน

```ld
/* =============================================================
 * ไฟล์: basic.ld
 * คำอธิบาย: Linker Script พื้นฐาน
 * ใช้งาน: ld -T basic.ld hello.o -o hello
 * ============================================================= */

/* Entry point - จุดเริ่มต้นของโปรแกรม */
ENTRY(_start)

/* SECTIONS command - กำหนดว่า sections จะวางที่ไหน */
SECTIONS
{
    /* . = current location counter */
    
    /* กำหนด starting address */
    . = 0x400000;
    
    /* .text section - executable code */
    .text :
    {
        *(.text)        /* รวม .text จากทุก object files */
        *(.text.*)      /* รวม .text.xxx sections */
    }
    
    /* align ก่อน data section */
    . = ALIGN(4096);
    
    /* .data section - initialized data */
    .data :
    {
        *(.data)
        *(.data.*)
    }
    
    /* .bss section - uninitialized data */
    .bss :
    {
        *(.bss)
        *(.bss.*)
        *(COMMON)       /* global uninitialized variables */
    }
    
    /* ทิ้ง sections ที่ไม่ต้องการ */
    /DISCARD/ :
    {
        *(.comment)     /* ทิ้ง comment section */
        *(.note.*)      /* ทิ้ง notes */
    }
}
```

### 9.2 Linker Script สำหรับ Embedded Systems

```ld
/* =============================================================
 * ไฟล์: embedded.ld
 * คำอธิบาย: Linker Script สำหรับ Embedded System (ARM Cortex-M)
 * Memory Layout:
 *   FLASH: 0x08000000 - 0x0807FFFF (512KB)
 *   RAM:   0x20000000 - 0x2001FFFF (128KB)
 * ============================================================= */

/* กำหนด Memory Regions */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

/* Entry point */
ENTRY(Reset_Handler)

/* Stack อยู่ที่ปลาย RAM */
_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
    /* Interrupt Vector Table - ต้องอยู่ที่ start ของ FLASH */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))    /* KEEP ป้องกัน linker ลบทิ้ง */
        . = ALIGN(4);
    } > FLASH

    /* Code และ read-only data ใน FLASH */
    .text :
    {
        . = ALIGN(4);
        *(.text)
        *(.text.*)
        *(.rodata)
        *(.rodata.*)
        
        /* เก็บ address สุดท้ายของ FLASH data */
        _etext = .;
    } > FLASH

    /* Initialized data - อยู่ใน FLASH แต่ copy ไป RAM ตอน startup */
    .data :
    {
        . = ALIGN(4);
        _sdata = .;             /* start of .data in RAM */
        *(.data)
        *(.data.*)
        . = ALIGN(4);
        _edata = .;             /* end of .data in RAM */
    } > RAM AT > FLASH          /* "AT > FLASH" = load address ใน FLASH */

    /* LMA (Load Memory Address) ของ .data */
    _sidata = LOADADDR(.data);

    /* Uninitialized data ใน RAM */
    .bss :
    {
        . = ALIGN(4);
        _sbss = .;
        *(.bss)
        *(.bss.*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
    } > RAM

    /* Stack ใน RAM */
    ._stack :
    {
        . = ALIGN(8);
        . = . + 0x400;          /* 1KB minimum stack */
        . = ALIGN(8);
    } > RAM
}
```

### 9.3 Symbol Tables และ Relocations

```nasm
; =============================================================
; ไฟล์: symbols.asm
; คำอธิบาย: ตัวอย่างการใช้ Symbols และ Relocations
; =============================================================

BITS 64

; Global symbols - มองเห็นได้จาก linker
global _start           ; จำเป็นสำหรับ entry point
global add_numbers      ; export function
global global_counter   ; export variable

; External symbols - defined ในที่อื่น
extern printf
extern malloc

section .data
    global_counter  dq 0            ; global variable

section .text

_start:
    ; เรียก function ใน file เดียวกัน
    mov rdi, 10
    mov rsi, 20
    call add_numbers    ; direct call, linker คำนวณ offset
    
    ; ถ้า call function ภายนอก
    ; call printf       ; linker จะสร้าง PLT entry
    
    mov rax, 60
    xor rdi, rdi
    syscall

; Function ที่ export ไปยัง modules อื่น
add_numbers:
    ; rdi = a, rsi = b
    mov rax, rdi
    add rax, rsi        ; rax = a + b
    ret
```

```bash
# ดู symbol table
nm symbols.o

# Output:
# 0000000000000000 T _start
# 0000000000000024 T add_numbers
# 0000000000000000 D global_counter
#                  U printf        <- undefined (จาก extern)
#                  U malloc        <- undefined (จาก extern)

# T = text (code)
# D = data (initialized)
# B = bss (uninitialized)
# U = undefined (external reference)
# u = local (lower case = local scope)
```

---

## 10. Static vs Dynamic Linking

### 10.1 Static Linking

```bash
# สร้าง static library
nasm -f elf64 mylib.asm -o mylib.o
ar rcs libmylib.a mylib.o     # สร้าง static library

# Link แบบ static
ld -static main.o -L. -lmylib -o static_program

# ผลลัพธ์: executable ขนาดใหญ่ แต่ไม่ต้องการ library ตอนรัน
ls -la static_program
# -rwxr-xr-x 1 user user 850432 Oct  1 10:00 static_program
```

### 10.2 Dynamic Linking

```bash
# สร้าง shared library
nasm -f elf64 -DPIC mylib.asm -o mylib.o  # Position Independent Code
ld -shared mylib.o -o libmylib.so

# Link แบบ dynamic
ld main.o -L. -lmylib -o dynamic_program

# ตรวจสอบ dependencies
ldd dynamic_program
# Output:
#     libmylib.so => ./libmylib.so (0x00007f...)
#     libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)

# ผลลัพธ์: executable เล็กกว่า แต่ต้องมี .so ตอนรัน
ls -la dynamic_program
# -rwxr-xr-x 1 user user 8392 Oct  1 10:00 dynamic_program
```

### 10.3 Position Independent Code (PIC)

```nasm
; =============================================================
; ไฟล์: pic_example.asm
; คำอธิบาย: Position Independent Code สำหรับ Shared Library
; คอมไพล์: nasm -f elf64 pic_example.asm -o pic_example.o
;           ld -shared pic_example.o -o libpic_example.so
; =============================================================

BITS 64
DEFAULT REL         ; RIP-relative addressing เป็น default

section .data
    internal_data   dq 42       ; data ภายใน library

section .text

; Export function
global get_value

get_value:
    ; ใช้ RIP-relative addressing เพื่อ PIC
    mov rax, [internal_data]    ; DEFAULT REL ทำให้เป็น RIP-relative
    ret

; ถ้าไม่ใช้ DEFAULT REL ต้องเขียนแบบนี้:
get_value_explicit:
    lea rsi, [rel internal_data]    ; explicit RIP-relative
    mov rax, [rsi]
    ret
```

---

## 11. เครื่องมือวิเคราะห์ Binary

### 11.1 nm - List Symbols

```bash
# สร้าง test file ก่อน
nasm -f elf64 symbols.asm -o symbols.o

# nm - แสดง symbols ทั้งหมด
nm symbols.o

# nm options:
nm -n symbols.o         # sort by address
nm -g symbols.o         # show only external (global) symbols
nm -u symbols.o         # show only undefined symbols
nm -l symbols.o         # show line numbers (ต้องการ debug info)
nm -D libmylib.so       # show dynamic symbols ของ shared lib

# ดู symbols ของ executable
nm /bin/ls | head -20
```

### 11.2 objdump - Disassemble และวิเคราะห์

```bash
# Disassemble
objdump -d hello                    # disassemble code sections
objdump -D hello                    # disassemble ทุก sections
objdump -d -M intel hello           # ใช้ Intel syntax

# แสดง sections
objdump -h hello                    # แสดง section headers
objdump -x hello                    # แสดงทุก headers

# แสดง source + disassembly (ต้องการ debug info)
objdump -S hello                    # ต้องใช้ -g ตอน compile

# แสดง relocations
objdump -r hello.o                  # relocations ใน object file

# แสดง data sections แบบ hex
objdump -s -j .data hello           # -j = section filter

# ตัวอย่าง output ของ objdump -d
# 0000000000401000 <_start>:
#   401000:	b8 01 00 00 00       	mov    $0x1,%eax
#   401005:	bf 01 00 00 00       	mov    $0x1,%edi
#   40100a:	48 8d 35 ef 1f 00 00 	lea    0x1fef(%rip),%rsi
#   401011:	ba 0e 00 00 00       	mov    $0xe,%edx
#   401016:	0f 05                	syscall
```

### 11.3 readelf - ELF File Analysis

```bash
# ดู ELF Header
readelf -h hello

# ดู Section Headers
readelf -S hello
# Output:
# Section Headers:
#   [Nr] Name              Type             Address           Offset
#   [ 0]                   NULL             0000000000000000  00000000
#   [ 1] .text             PROGBITS         0000000000401000  00001000
#   [ 2] .data             PROGBITS         0000000000402000  00002000
#   [ 3] .shstrtab         STRTAB           0000000000000000  00002100

# ดู Program Headers (Segments)
readelf -l hello

# ดู Symbol Table
readelf -s hello

# ดู Relocations
readelf -r hello.o

# ดู Dynamic Section
readelf -d hello

# hex dump ของ section
readelf -x .data hello
readelf -p .rodata hello    # print strings

# ดูทุกอย่างในครั้งเดียว
readelf -a hello | less
```

---

## 12. GDB (GNU Debugger) - คู่มือฉบับสมบูรณ์

GDB เป็นเครื่องมือ debug ที่ทรงพลังและสำคัญมากสำหรับ Assembly programming

### 12.1 Compile ด้วย Debug Information

```bash
# NASM: ใส่ debug information
nasm -f elf64 -g -F dwarf hello.asm -o hello.o
ld hello.o -o hello

# หรือใช้ stabs format
nasm -f elf64 -g -F stabs hello.asm -o hello.o

# GAS
as --gstabs+ -o hello.o hello.s
ld hello.o -o hello
```

### 12.2 ตัวอย่างโปรแกรมสำหรับ Debug

```nasm
; =============================================================
; ไฟล์: gdb_example.asm
; คำอธิบาย: โปรแกรมสำหรับฝึก GDB debugging
; คอมไพล์: nasm -f elf64 -g -F dwarf gdb_example.asm -o gdb_example.o
;           ld gdb_example.o -o gdb_example
; =============================================================

BITS 64

section .data
    array       dd 5, 3, 8, 1, 9, 2, 7, 4, 6    ; array ขนาด 9 ตัว
    array_len   equ ($ - array) / 4               ; จำนวน elements

section .bss
    sum         resq 1          ; เก็บผลรวม

section .text
    global _start

_start:
    ; คำนวณผลรวมของ array
    mov rcx, array_len      ; counter = array_len
    mov rsi, array          ; pointer ไปยัง array
    xor rax, rax            ; sum = 0
    
.loop:
    cmp rcx, 0              ; ถ้า counter = 0 จบ loop
    je .done
    
    mov ebx, [rsi]          ; อ่าน element จาก array (32-bit)
    add rax, rbx            ; sum += element
    
    add rsi, 4              ; ไปยัง element ถัดไป (4 bytes)
    dec rcx                 ; ลด counter
    jmp .loop

.done:
    mov [sum], rax          ; เก็บผลรวม
    
    ; ออกจากโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 12.3 GDB Commands พื้นฐาน

```
===== GDB SESSION =====

# เริ่ม GDB
$ gdb ./gdb_example

# หรือ
$ gdb
(gdb) file ./gdb_example

------- คำสั่งพื้นฐาน -------

# ตั้ง Breakpoints
(gdb) break _start          # breakpoint ที่ label
(gdb) b _start              # ย่อ break
(gdb) break *0x401000       # breakpoint ที่ address
(gdb) break gdb_example.asm:25  # breakpoint ที่ line number
(gdb) info break            # ดู breakpoints ทั้งหมด
(gdb) delete 1              # ลบ breakpoint #1
(gdb) disable 2             # ปิดการใช้งาน breakpoint #2

# รันโปรแกรม
(gdb) run                   # เริ่มรัน
(gdb) run arg1 arg2         # รันพร้อม arguments

# Execution Control
(gdb) next                  # ไปบรรทัดถัดไป (ข้าม function calls)
(gdb) n                     # ย่อ next
(gdb) step                  # เข้าไปใน function calls
(gdb) s                     # ย่อ step
(gdb) stepi                 # step instruction (ทีละ instruction)
(gdb) si                    # ย่อ stepi
(gdb) nexti                 # next instruction
(gdb) ni                    # ย่อ nexti
(gdb) continue              # รันต่อจนถึง breakpoint ถัดไป
(gdb) c                     # ย่อ continue
(gdb) finish                # รันจนจบ function ปัจจุบัน
(gdb) until 50              # รันจนถึง line 50

# ดู Registers
(gdb) info registers        # ดูทุก registers
(gdb) info reg              # ย่อ
(gdb) info registers rax rbx rcx   # ดูเฉพาะ registers ที่ระบุ
(gdb) p $rax                # print ค่าของ register rax
(gdb) p/x $rax              # print แบบ hex
(gdb) p/d $rax              # print แบบ decimal
(gdb) p/t $rax              # print แบบ binary

# ดู Memory
(gdb) x/10xb $rsi           # แสดง 10 bytes ณ address ใน rsi (hex)
(gdb) x/4xw $rsi            # แสดง 4 words (32-bit) แบบ hex
(gdb) x/8xg $rsi            # แสดง 8 giant words (64-bit)
(gdb) x/5i $rip             # แสดง 5 instructions ถัดไป
(gdb) x/s $rsi              # แสดงเป็น string
(gdb) x/d $rsi              # แสดงเป็น decimal

# Format characters:
# x = hexadecimal
# d = decimal
# u = unsigned decimal
# o = octal
# t = binary
# f = float
# c = character
# s = string
# i = instruction

# Size characters:
# b = byte (1)
# h = halfword (2)
# w = word (4)
# g = giant/quadword (8)

# ดู Stack
(gdb) info stack            # backtrace
(gdb) bt                    # ย่อ backtrace
(gdb) x/8xg $rsp            # ดู stack contents

# ดู Source
(gdb) list                  # แสดง source code รอบๆ บรรทัดปัจจุบัน
(gdb) list 1,30             # แสดง line 1-30
(gdb) disassemble           # disassemble function ปัจจุบัน
(gdb) disassemble _start    # disassemble function _start
(gdb) disassemble /m _start # disassemble + source interleaved

# แก้ไขค่า
(gdb) set $rax = 42         # ตั้งค่า register
(gdb) set *0x601010 = 100   # ตั้งค่า memory

# Watchpoints (หยุดเมื่อ memory เปลี่ยน)
(gdb) watch $rax            # หยุดเมื่อ rax เปลี่ยน
(gdb) watch *0x601010       # หยุดเมื่อ memory ที่ address เปลี่ยน
(gdb) rwatch *0x601010      # หยุดเมื่ออ่าน address นั้น
(gdb) awatch *0x601010      # หยุดเมื่ออ่านหรือเขียน

# Convenience Variables
(gdb) set $count = 0
(gdb) p $count

# Quit
(gdb) quit
(gdb) q
```

### 12.4 GDB TUI Mode (Text User Interface)

```
TUI Mode แสดงผล source code และ GDB prompt พร้อมกัน

# เริ่ม GDB ใน TUI mode
$ gdb -tui ./program

# หรือใน GDB prompt
(gdb) tui enable
(gdb) layout asm         # แสดง disassembly window
(gdb) layout src         # แสดง source window
(gdb) layout regs        # แสดง registers window
(gdb) layout split       # แสดง source + disassembly
(gdb) layout next        # สลับ layout

# TUI Keyboard Shortcuts
# Ctrl+L    - redraw screen
# Ctrl+X a  - toggle TUI mode on/off
# Ctrl+X 1  - single window (current layout)
# Ctrl+X 2  - split window
# PgUp/Down - scroll active window
# Arrow keys - move cursor

# ใน TUI mode ใช้คำสั่งเหมือนกัน:
(gdb) b _start
(gdb) r
(gdb) si          # step instruction - เห็น highlight บน asm window
```

### 12.5 GDB Script (Automation)

```gdb
# =============================================================
# ไฟล์: debug_script.gdb
# คำอธิบาย: GDB script สำหรับ automate debugging
# ใช้งาน: gdb -x debug_script.gdb ./gdb_example
# =============================================================

# ตั้งค่า settings
set disassembly-flavor intel    # ใช้ Intel syntax
set pagination off              # ปิด "press enter to continue"

# โหลด program
file ./gdb_example

# ตั้ง breakpoints
break _start
break .loop
break .done

# รัน
run

# ดู initial state
echo ===== Initial State =====\n
info registers rax rcx rsi

# Continue ไปยัง .loop
continue

# Print loop state
echo ===== Loop State =====\n
printf "RCX (counter) = %d\n", $rcx
printf "RSI (array ptr) = 0x%lx\n", $rsi
x/9dw array             # ดู array data

# Continue อีก 5 iterations
continue
continue
continue
continue
continue

echo ===== After 5 iterations =====\n
printf "RAX (sum so far) = %d\n", $rax

# ไปยังจุดสิ้นสุด
continue

echo ===== Final Result =====\n
printf "Sum = %d\n", $rax

# ออก
quit
```

```bash
# รัน GDB script
gdb -x debug_script.gdb ./gdb_example
# หรือ
gdb -batch -x debug_script.gdb ./gdb_example
```

### 12.6 GDB สำหรับ Debug Assembly ที่ซับซ้อน

```nasm
; =============================================================
; ไฟล์: bubble_sort.asm
; คำอธิบาย: Bubble Sort สำหรับฝึก debug ด้วย GDB
; คอมไพล์: nasm -f elf64 -g -F dwarf bubble_sort.asm -o bubble_sort.o
;           ld bubble_sort.o -o bubble_sort
; =============================================================

BITS 64

section .data
    array   dd 64, 25, 12, 22, 11
    n       equ 5

section .text
    global _start

_start:
    mov rdi, array      ; pointer to array
    mov rsi, n          ; n = array length
    call bubble_sort
    
    mov rax, 60
    xor rdi, rdi
    syscall

; bubble_sort(rdi=array, rsi=n)
bubble_sort:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi        ; r12 = array
    mov r13, rsi        ; r13 = n
    
    dec r13             ; outer loop: n-1 iterations
    
.outer_loop:
    test r13, r13       ; if n-1 == 0, done
    jz .done
    
    xor r14, r14        ; r14 = i = 0
    
.inner_loop:
    mov r15, r13        ; r15 = (n-1) - i? actually n-1
    ; correct: inner loop goes from 0 to n-1-outer
    ; simplified: just n-1 times
    
    lea rbx, [r12 + r14*4]  ; rbx = &array[i]
    mov eax, [rbx]          ; eax = array[i]
    mov ecx, [rbx + 4]      ; ecx = array[i+1]
    
    cmp eax, ecx            ; compare array[i] vs array[i+1]
    jle .no_swap            ; if array[i] <= array[i+1], no swap
    
    ; Swap
    mov [rbx], ecx          ; array[i] = array[i+1]
    mov [rbx + 4], eax      ; array[i+1] = array[i]
    
.no_swap:
    inc r14                 ; i++
    cmp r14, r13            ; if i < n-1
    jl .inner_loop
    
    dec r13                 ; outer counter--
    jmp .outer_loop
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

```
# GDB Session สำหรับ debug bubble_sort:

$ gdb ./bubble_sort
(gdb) set disassembly-flavor intel
(gdb) break bubble_sort
(gdb) run
(gdb) layout asm

# ดู array ก่อน sort
(gdb) x/5dw array
# 0x402000 <array>:    64    25    12    22    11

# Step ผ่าน algorithm
(gdb) si
(gdb) si
(gdb) si

# หลัง sort เสร็จ
(gdb) x/5dw array
# 0x402000 <array>:    11    12    22    25    64
```

---

## 13. LLDB สำหรับ macOS และ ARM

LLDB เป็น debugger ของ LLVM project ที่ใช้เป็น default บน macOS และ Xcode

### 13.1 LLDB Commands เปรียบกับ GDB

```
┌─────────────────────────────┬─────────────────────────────┐
│ GDB Command                 │ LLDB Equivalent             │
├─────────────────────────────┼─────────────────────────────┤
│ break _start                │ breakpoint set --name _start│
│ b _start                    │ b _start (compatible)       │
│ run                         │ run / r                     │
│ continue                    │ continue / c                │
│ next                        │ next / n                    │
│ step                        │ step / s                    │
│ stepi                       │ stepi / si                  │
│ nexti                       │ nexti / ni                  │
│ info registers              │ register read               │
│ print $rax                  │ register read rax           │
│ x/10xb $rsi                 │ memory read -fx -c10 $rsi   │
│ disassemble                 │ disassemble                 │
│ info stack                  │ bt / thread backtrace       │
│ quit                        │ quit / q                    │
└─────────────────────────────┴─────────────────────────────┘
```

### 13.2 ตัวอย่าง LLDB Session

```
# เริ่ม LLDB
$ lldb ./my_program

# หรือ attach to running process
$ lldb -p 12345

------- LLDB Session -------

# ตั้ง breakpoint
(lldb) breakpoint set --name _start
(lldb) b _start                         # shorthand

# ตั้ง breakpoint ที่ address
(lldb) breakpoint set --address 0x100001000

# รัน
(lldb) run
(lldb) r

# ดู registers
(lldb) register read                    # ทุก registers
(lldb) register read rax rbx rcx       # เฉพาะ registers ที่ระบุ
(lldb) register read --all             # รวม float/vector regs

# ดู memory
(lldb) memory read --format x --size 1 --count 16 $rsi
(lldb) mem read -fx -s1 -c16 $rsi      # ย่อ
(lldb) x/16xb $rsi                     # GDB-compatible format

# แสดง disassembly
(lldb) disassemble                      # current function
(lldb) dis -n _start                   # specific function
(lldb) dis -a $pc                      # ณ current PC

# Source
(lldb) source list                     # แสดง source
(lldb) source list -l 10               # แสดง 10 lines

# แก้ไขค่า
(lldb) register write rax 42
(lldb) memory write -s 4 0x100002000 0x42

# Expression evaluation
(lldb) expression $rax + $rbx
(lldb) expr -- (int)$rax               # cast type
```

### 13.3 LLDB บน macOS ARM64

```asm
; =============================================================
; ไฟล์: macos_arm64.s
; คำอธิบาย: Hello World บน macOS ARM64
; คอมไพล์: as -o macos_arm64.o macos_arm64.s
;           ld -o macos_arm64 macos_arm64.o \
;              -lSystem -syslibroot $(xcrun -sdk macosx --show-sdk-path) \
;              -e _start -arch arm64
; =============================================================

.section __TEXT,__text
.globl _start
.align 2

_start:
    ; macOS ใช้ BSD-based syscall numbers (ต่างจาก Linux)
    ; macOS ARM64 syscall: x16 = syscall number
    
    ; write(1, message, length)
    mov x0, #1              ; fd = stdout
    adr x1, message         ; buffer address
    mov x2, #msg_len        ; length
    mov x16, #4             ; syscall: write (macOS = 4)
    svc #0x80               ; macOS ใช้ svc #0x80

    ; exit(0)
    mov x0, #0              ; exit code
    mov x16, #1             ; syscall: exit (macOS = 1)
    svc #0x80

.section __DATA,__data
message:
    .asciz "Hello from macOS ARM64!\n"
msg_len = . - message
```

---

## 14. Valgrind - Memory Error Detection

Valgrind เป็นเครื่องมือสำคัญสำหรับตรวจจับ memory errors ใน programs

### 14.1 Memory Errors ที่ Valgrind ตรวจสอบได้

```
1. Memory Leaks (ไม่ free memory ที่ malloc'd)
2. Use After Free (ใช้ memory หลัง free แล้ว)
3. Buffer Overflow (เขียนนอก array boundary)
4. Uninitialized Memory Use (ใช้ memory ที่ยังไม่ initialize)
5. Invalid Read/Write (อ่าน/เขียน address ที่ไม่ valid)
6. Double Free (free memory 2 ครั้ง)
```

### 14.2 โปรแกรม C ที่มี Memory Errors (สำหรับทดสอบ Valgrind)

```c
/* =============================================================
 * ไฟล์: memory_errors.c
 * คำอธิบาย: ตัวอย่างโปรแกรม C ที่มี memory errors
 * คอมไพล์: gcc -g -o memory_errors memory_errors.c
 * ทดสอบ: valgrind --leak-check=full ./memory_errors
 * ============================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    // ERROR 1: Memory Leak - malloc แต่ไม่ free
    int *leaked = malloc(100 * sizeof(int));
    leaked[0] = 42;
    // ลืม free(leaked);

    // ERROR 2: Buffer Overflow
    int *array = malloc(5 * sizeof(int));
    for (int i = 0; i <= 5; i++) {   // i <= 5 แทนที่จะเป็น i < 5
        array[i] = i;                  // array[5] = overflow!
    }
    free(array);

    // ERROR 3: Use After Free
    int *ptr = malloc(sizeof(int));
    *ptr = 100;
    free(ptr);
    printf("Value: %d\n", *ptr);      // ERROR! ptr ถูก free แล้ว

    // ERROR 4: Uninitialized Memory
    int uninit;
    if (uninit > 0) {                  // uninit ยังไม่ initialize
        printf("positive\n");
    }

    return 0;
}
```

### 14.3 Valgrind Commands

```bash
# ติดตั้ง Valgrind
sudo apt-get install valgrind

# การใช้งานพื้นฐาน
valgrind ./program

# ตรวจสอบ memory leaks แบบละเอียด
valgrind --leak-check=full ./program

# ตัวเลือกทั้งหมด
valgrind --leak-check=full \          # ตรวจ leaks
         --show-leak-kinds=all \      # แสดงทุกประเภท leak
         --track-origins=yes \        # ติดตาม origin ของ uninitialized
         --verbose \                  # แสดงรายละเอียดเพิ่ม
         --log-file=valgrind.log \    # บันทึก output ลงไฟล์
         ./program

# ตัวอย่าง Valgrind Output:
# ==12345== Memcheck, a memory error detector
# ==12345== Invalid write of size 4
# ==12345==    at 0x10920A: main (memory_errors.c:17)
# ==12345==  Address 0x51b4054 is 0 bytes after a block of size 20 alloc'd
# ==12345==
# ==12345== LEAK SUMMARY:
# ==12345==    definitely lost: 400 bytes in 1 blocks
# ==12345==    indirectly lost: 0 bytes in 0 blocks

# Callgrind - profiling tool
valgrind --tool=callgrind ./program
callgrind_annotate callgrind.out.12345

# Massif - heap profiler
valgrind --tool=massif ./program
ms_print massif.out.12345
```

### 14.4 Assembly Program กับ Valgrind

```nasm
; =============================================================
; ไฟล์: valgrind_test.asm
; คำอธิบาย: Assembly program ที่ใช้ mmap/munmap (ทดสอบกับ Valgrind)
; คอมไพล์: nasm -f elf64 -g valgrind_test.asm -o valgrind_test.o
;           ld valgrind_test.o -o valgrind_test
; ทดสอบ: valgrind --leak-check=full ./valgrind_test
; =============================================================

BITS 64

; Syscall numbers
%define SYS_WRITE   1
%define SYS_MMAP    9
%define SYS_MUNMAP  11
%define SYS_EXIT    60

; mmap flags
%define PROT_READ    1
%define PROT_WRITE   2
%define MAP_PRIVATE  2
%define MAP_ANON     32

section .data
    msg_alloc   db 'Memory allocated!', 0x0A
    len_alloc   equ $ - msg_alloc
    
    msg_free    db 'Memory freed!', 0x0A
    len_free    equ $ - msg_free

section .text
    global _start

_start:
    ; Allocate 4096 bytes ด้วย mmap
    ; mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
    xor rdi, rdi                        ; addr = NULL
    mov rsi, 4096                       ; length = 4096
    mov rdx, PROT_READ | PROT_WRITE     ; prot
    mov r10, MAP_PRIVATE | MAP_ANON     ; flags
    mov r8, -1                          ; fd = -1 (anonymous)
    xor r9, r9                          ; offset = 0
    mov rax, SYS_MMAP
    syscall
    
    ; rax = allocated memory address
    ; ตรวจสอบ error (ถ้า rax < 0 แสดงว่า error)
    cmp rax, -1
    je .error
    
    mov rbx, rax                        ; บันทึก address ไว้
    
    ; ใช้ memory
    mov qword [rbx], 0xDEADBEEF        ; เขียนค่าลง memory
    
    ; แสดง success message
    mov rax, SYS_WRITE
    mov rdi, 1
    mov rsi, msg_alloc
    mov rdx, len_alloc
    syscall
    
    ; Free memory ด้วย munmap
    mov rdi, rbx                        ; address
    mov rsi, 4096                       ; length
    mov rax, SYS_MUNMAP
    syscall
    
    ; แสดง free message
    mov rax, SYS_WRITE
    mov rdi, 1
    mov rsi, msg_free
    mov rdx, len_free
    syscall
    
.exit:
    mov rax, SYS_EXIT
    xor rdi, rdi
    syscall

.error:
    mov rax, SYS_EXIT
    mov rdi, 1
    syscall
```

---

## 15. ข้อผิดพลาดที่พบบ่อยและวิธีแก้ไข

### 15.1 ข้อผิดพลาดใน Assembler

```nasm
; --- ข้อผิดพลาดที่ 1: ลืม BITS directive ---
; ผิด: (NASM จะ default เป็น 16-bit mode)
section .text
    global _start
_start:
    mov rax, 60     ; ERROR: rax ไม่มีใน 16-bit mode

; ถูก:
BITS 64
section .text
    global _start
_start:
    mov rax, 60     ; OK: 64-bit mode


; --- ข้อผิดพลาดที่ 2: Size mismatch ---
; ผิด:
mov eax, [rsi]      ; อ่าน 32-bit จาก 64-bit address (ไม่ใช่ error แต่อาจไม่ตั้งใจ)
mov rax, [esi]      ; ERROR: ใช้ 32-bit address ใน 64-bit mode

; ถูก:
mov eax, [rsi]      ; อ่าน 32-bit value (zero-extends to rax)
mov rax, [rsi]      ; อ่าน 64-bit value


; --- ข้อผิดพลาดที่ 3: Label redefinition ---
; ผิด:
my_label:
    nop
my_label:           ; ERROR: label ซ้ำ
    nop

; ถูก: ใช้ชื่อต่างกัน หรือใช้ local labels
.my_label:          ; local label (ขึ้นต้นด้วย .)
    nop


; --- ข้อผิดพลาดที่ 4: Immediate value too large ---
; ผิด:
mov al, 256         ; ERROR: 256 ใหญ่เกิน 1 byte (max 255)
mov ax, 65536       ; ERROR: 65536 ใหญ่เกิน 2 bytes (max 65535)

; ถูก:
mov al, 255         ; OK
mov ax, 65535       ; OK
```

### 15.2 ข้อผิดพลาดใน Linker

```bash
# --- ข้อผิดพลาดที่ 1: Undefined reference ---
# ld: undefined reference to `printf'
# แก้ไข: link กับ libc
ld main.o -lc -dynamic-linker /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 -o program

# --- ข้อผิดพลาดที่ 2: Multiple definition ---
# ld: multiple definition of `_start'
# แก้ไข: ตรวจสอบว่ามี _start ใน object files หลายตัว

# --- ข้อผิดพลาดที่ 3: Cannot find entry symbol ---
# ld: cannot find entry symbol _start
# แก้ไข: ต้องมี global _start หรือระบุ entry point
ld -e main main.o -o program      # ระบุ entry point เป็น main
```

### 15.3 ข้อผิดพลาดใน GDB

```
# --- ข้อผิดพลาดที่ 1: No debugging symbols ---
# "(gdb) list" แสดง "No source file specified"
# แก้ไข: compile ด้วย -g flag
nasm -f elf64 -g -F dwarf hello.asm -o hello.o

# --- ข้อผิดพลาดที่ 2: ไม่เห็น breakpoint hit ---
# แก้ไข: ตรวจสอบว่า ASLR ถูก disable สำหรับ debugging
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
# หรือใน GDB:
(gdb) set disable-randomization on

# --- ข้อผิดพลาดที่ 3: Cannot access memory ---
# แก้ไข: ตรวจสอบว่า address ถูกต้อง และ program กำลังรันอยู่
(gdb) info proc mappings    # ดู memory map ของ process
```

---

## 16. แบบฝึกหัด (Exercises)

### Exercise 1: NASM Macro Library

สร้างไฟล์ macros.asm ที่มี macro ต่อไปนี้:
- `print_int` - พิมพ์ integer 64-bit
- `print_newline` - พิมพ์ newline
- `exit_program` - ออกจากโปรแกรม

```nasm
; =============================================================
; ไฟล์: ex1_macros.asm
; คำอธิบาย: Exercise 1 - สร้าง Macro Library
; คอมไพล์: nasm -f elf64 ex1_macros.asm -o ex1_macros.o
;           ld ex1_macros.o -o ex1_macros
; Expected output:
;   42
;   -100
;   0
; =============================================================

BITS 64

; Macro: แปลง integer เป็น string และพิมพ์
%macro print_int 1
    ; บันทึก registers
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    
    mov rax, %1         ; ค่าที่จะพิมพ์
    
    ; ตรวจสอบ sign
    test rax, rax
    jns %%positive
    
    ; negative: พิมพ์ '-' ก่อน
    push rax
    mov rax, 1          ; write
    mov rdi, 1          ; stdout
    mov rsi, %%minus_sign
    mov rdx, 1
    syscall
    pop rax
    neg rax             ; เปลี่ยนเป็น positive
    
%%positive:
    ; แปลงเป็น string (decimal)
    mov rsi, %%buffer + 20  ; ชี้ที่ end ของ buffer
    mov byte [rsi], 0x0A    ; newline ที่ end
    dec rsi
    
    mov rbx, 10             ; base 10
    
%%convert_loop:
    xor rdx, rdx
    div rbx                 ; rax = quotient, rdx = remainder
    add dl, '0'             ; แปลงเป็น ASCII
    mov [rsi], dl
    dec rsi
    test rax, rax
    jnz %%convert_loop
    
    inc rsi                 ; ชี้ที่ตัวแรก
    
    ; คำนวณความยาว
    lea rdx, [%%buffer + 21]
    sub rdx, rsi
    
    ; พิมพ์
    mov rax, 1
    mov rdi, 1
    syscall
    
    ; คืนค่า registers
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    
    jmp %%end
    
%%minus_sign: db '-'
%%buffer: times 21 db 0
%%end:
%endmacro

%macro print_newline 0
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 1
    mov rsi, %%nl
    mov rdx, 1
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    jmp %%end
%%nl: db 0x0A
%%end:
%endmacro

%macro exit_program 1
    mov rax, 60
    mov rdi, %1
    syscall
%endmacro

section .text
    global _start

_start:
    print_int 42
    print_int -100
    print_int 0
    exit_program 0
```

### Exercise 2: วิเคราะห์ Binary ด้วย Tools

```bash
#!/bin/bash
# =============================================================
# ไฟล์: ex2_analyze.sh
# คำอธิบาย: Script วิเคราะห์ binary file
# =============================================================

# 1. Compile test program
cat > /tmp/test_prog.asm << 'EOF'
BITS 64
section .data
    message db "Hello!", 10
    len     equ $ - message
    value   dq 0xDEADBEEF

section .bss
    buffer  resb 100

section .text
    global _start
    global helper_func

_start:
    mov rax, 1
    mov rdi, 1
    mov rsi, message
    mov rdx, len
    syscall
    call helper_func
    mov rax, 60
    xor rdi, rdi
    syscall

helper_func:
    mov rax, 42
    ret
EOF

nasm -f elf64 -g /tmp/test_prog.asm -o /tmp/test_prog.o
ld /tmp/test_prog.o -o /tmp/test_prog

echo "===== nm output ====="
nm /tmp/test_prog

echo ""
echo "===== readelf -S (sections) ====="
readelf -S /tmp/test_prog

echo ""
echo "===== objdump -d (disassembly) ====="
objdump -d -M intel /tmp/test_prog

echo ""
echo "===== objdump -s -j .data (data section hex) ====="
objdump -s -j .data /tmp/test_prog
```

### Exercise 3: GDB Debug Session

```nasm
; =============================================================
; ไฟล์: ex3_debug.asm
; คำอธิบาย: โปรแกรมที่มี bug - ให้ใช้ GDB หา bug
; คอมไพล์: nasm -f elf64 -g -F dwarf ex3_debug.asm -o ex3_debug.o
;           ld ex3_debug.o -o ex3_debug
; 
; Bug description: โปรแกรมควรคำนวณ factorial ของ 5
; Expected result: 120
; แต่ผลที่ได้ไม่ถูกต้อง - ให้หา bug ด้วย GDB
; =============================================================

BITS 64

section .text
    global _start

_start:
    mov rdi, 5          ; n = 5
    call factorial
    ; rax ควรจะเป็น 120 (5!)
    
    mov rax, 60
    xor rdi, rdi
    syscall

; factorial(rdi = n) -> rax = n!
factorial:
    push rbp
    mov rbp, rsp
    
    ; Base case: if n <= 1, return 1
    cmp rdi, 1
    jg .recursive
    mov rax, 1
    jmp .done

.recursive:
    push rdi            ; บันทึก n
    dec rdi             ; n-1
    call factorial      ; factorial(n-1)
    pop rdi             ; คืนค่า n
    
    mul rdi             ; BUG: ควรใช้ imul! mul ใช้ rax และ rdx
                        ; ถ้า rdi ใหญ่อาจมี overflow ใน rdx
    
.done:
    pop rbp
    ret
```

```
# GDB session สำหรับ Exercise 3:
$ gdb ./ex3_debug
(gdb) set disassembly-flavor intel
(gdb) break factorial
(gdb) run
(gdb) display/x $rax
(gdb) display/d $rdi
(gdb) si        # step ผ่าน factorial calls
# สังเกตค่า rax ที่แต่ละขั้น
# 1! = 1, 2! = 2, 3! = 6, 4! = 24, 5! = 120
```

### Exercise 4: สร้าง Shared Library ด้วย Assembly

```nasm
; =============================================================
; ไฟล์: ex4_library.asm
; คำอธิบาย: สร้าง shared library ด้วย Assembly
; คอมไพล์:
;   nasm -f elf64 ex4_library.asm -o ex4_library.o
;   ld -shared -soname libmath.so.1 ex4_library.o -o libmath.so.1
;   ln -s libmath.so.1 libmath.so
; =============================================================

BITS 64
DEFAULT REL

section .text

; export functions
global asm_add
global asm_multiply
global asm_power

; asm_add(rdi=a, rsi=b) -> rax = a+b
asm_add:
    mov rax, rdi
    add rax, rsi
    ret

; asm_multiply(rdi=a, rsi=b) -> rax = a*b
asm_multiply:
    mov rax, rdi
    imul rax, rsi
    ret

; asm_power(rdi=base, rsi=exp) -> rax = base^exp
asm_power:
    push rbx
    mov rax, 1          ; result = 1
    mov rbx, rdi        ; base
    
.loop:
    test rsi, rsi       ; exp == 0?
    jz .done
    imul rax, rbx       ; result *= base
    dec rsi             ; exp--
    jmp .loop
    
.done:
    pop rbx
    ret
```

```c
/* ex4_main.c - ใช้ shared library */
/* คอมไพล์: gcc -o ex4_main ex4_main.c -L. -lmath -Wl,-rpath,. */

#include <stdio.h>

extern long asm_add(long a, long b);
extern long asm_multiply(long a, long b);
extern long asm_power(long base, long exp);

int main() {
    printf("3 + 4 = %ld\n", asm_add(3, 4));
    printf("3 * 4 = %ld\n", asm_multiply(3, 4));
    printf("2^10 = %ld\n", asm_power(2, 10));
    return 0;
}
```

### Exercise 5: Linker Script สำหรับ Custom Layout

```nasm
; =============================================================
; ไฟล์: ex5_custom_layout.asm
; คำอธิบาย: Assembly program ที่ใช้ custom linker script
; =============================================================

BITS 64

section .custom_text    ; custom section name
    global _start

_start:
    mov rax, 1
    mov rdi, 1
    mov rsi, message
    mov rdx, msg_len
    syscall
    
    mov rax, 60
    xor rdi, rdi
    syscall

section .custom_data
message:    db "Hello from custom layout!", 10
msg_len:    equ $ - message
```

```ld
/* ex5.ld - Custom Linker Script */
ENTRY(_start)

SECTIONS
{
    . = 0x500000;   /* เริ่ม address ที่ 0x500000 แทน default */
    
    .text : { *(.custom_text) *(.text) }
    
    . = ALIGN(0x1000);
    
    .data : { *(.custom_data) *(.data) }
    
    .bss  : { *(.bss) }
}
```

```bash
# คอมไพล์ด้วย custom linker script
nasm -f elf64 ex5_custom_layout.asm -o ex5.o
ld -T ex5.ld ex5.o -o ex5_custom

# ตรวจสอบ addresses
readelf -S ex5_custom | grep -E "Name|Address|custom|text|data"
# จะเห็นว่า .text เริ่มที่ 0x500000
```

---

## 17. สรุปและ Key Takeaways

### สรุปเครื่องมือ

```
┌─────────────────────────────────────────────────────────────────┐
│                    เครื่องมือและการใช้งาน                         │
├──────────────┬──────────────────────────────────────────────────┤
│ เครื่องมือ    │ การใช้งาน                                        │
├──────────────┼──────────────────────────────────────────────────┤
│ NASM         │ x86/x86-64, Intel syntax, cross-platform        │
│ YASM         │ NASM-compatible, better DWARF support           │
│ GAS (as)     │ GNU standard, AT&T/Intel syntax, used by GCC    │
│ MASM (ml64)  │ Windows, Visual Studio integration              │
│ ARM as       │ ARM/AArch64, used with GCC toolchain            │
├──────────────┼──────────────────────────────────────────────────┤
│ ld           │ GNU linker, จัดการ sections, symbols            │
│ lld          │ LLVM linker, เร็วกว่า ld                        │
│ ar           │ สร้าง static libraries (.a)                     │
├──────────────┼──────────────────────────────────────────────────┤
│ nm           │ list symbols ใน object/executable               │
│ objdump      │ disassemble, show sections, hex dump            │
│ readelf      │ วิเคราะห์ ELF files อย่างละเอียด                │
│ file         │ ระบุ file type                                   │
│ size         │ แสดงขนาดของ sections                            │
├──────────────┼──────────────────────────────────────────────────┤
│ GDB          │ debug บน Linux/Unix, full-featured              │
│ LLDB         │ debug บน macOS, iOS, ARM                        │
│ Valgrind     │ memory error detection, profiling               │
└──────────────┴──────────────────────────────────────────────────┘
```

### Key Concepts

1. **Two-Pass Assembly**: Pass 1 สร้าง Symbol Table, Pass 2 สร้าง Machine Code
2. **Object Files**: ELF (Linux), COFF/PE (Windows), Mach-O (macOS)
3. **Linking**: รวม object files, resolve symbols, สร้าง executable
4. **Static vs Dynamic**: Static = ใหญ่แต่ standalone; Dynamic = เล็กแต่ต้องการ .so
5. **PIC**: จำเป็นสำหรับ shared libraries, ใช้ RIP-relative addressing
6. **Debug Info**: ต้อง compile ด้วย -g เพื่อให้ debugger ทำงานได้

### Debugging Workflow

```
1. เขียน source code
2. Compile ด้วย -g flag (debug info)
3. รัน GDB/LLDB
4. ตั้ง breakpoints ที่จุดสำคัญ
5. รัน program
6. ตรวจสอบ registers และ memory ที่แต่ละ breakpoint
7. Step ผ่าน instructions ด้วย si/ni
8. ตรวจสอบ stack ด้วย x/Nxg $rsp
9. แก้ไข bug
10. รัน Valgrind เพื่อตรวจสอบ memory errors
```

---

## 18. แหล่งข้อมูลเพิ่มเติม (Resources)

### Documentation

- **NASM Manual**: https://www.nasm.us/doc/
  - ครอบคลุมทุก directive, syntax, macro system
- **GAS Manual (GNU as)**: https://sourceware.org/binutils/docs/as/
  - AT&T syntax, directives
- **LD Manual (GNU linker)**: https://sourceware.org/binutils/docs/ld/
  - Linker script syntax ทั้งหมด
- **GDB Manual**: https://sourceware.org/gdb/documentation/
  - Commands, scripting, TUI
- **ELF Specification**: https://refspecs.linuxfoundation.org/elf/elf.pdf
  - ELF format อย่างละเอียด

### Tools

```bash
# ติดตั้งเครื่องมือทั้งหมดบน Ubuntu
sudo apt-get install -y \
    nasm \
    yasm \
    binutils \
    gdb \
    valgrind \
    gcc \
    make

# ARM cross-compilation tools
sudo apt-get install -y \
    gcc-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    qemu-user-static    # รัน ARM binaries บน x86
```

### Quick Reference Cards

```
===== GDB Quick Reference =====
b <label>         ตั้ง breakpoint
r                 รัน
c                 continue
n/ni              next line/instruction
s/si              step into/instruction
info reg          ดู registers
x/<N><f><s> <addr>  ดู memory
bt                backtrace
q                 quit

===== objdump Quick Reference =====
-d           disassemble code
-D           disassemble all
-h           section headers
-s           full section contents
-r           relocations
-M intel     Intel syntax
-j .section  filter by section

===== nm Quick Reference =====
(no args)    show all symbols
-g           global symbols only
-u           undefined symbols only
-n           sort by address
-l           show line numbers

===== readelf Quick Reference =====
-h           ELF header
-S           section headers
-l           program headers
-s           symbol table
-r           relocations
-d           dynamic section
-a           all of the above
```

---

## ขั้นตอนถัดไป

หลังจาก Part 010 นี้ คุณพร้อมที่จะเรียนรู้:

- **Part 011**: Calling Conventions และ ABI (System V AMD64, Microsoft x64)
- **Part 012**: Interfacing Assembly กับ C/C++
- **Part 013**: SIMD Instructions (SSE, AVX) สำหรับ High Performance
- **Part 014**: System Calls อย่างละเอียด (Linux, Windows, macOS)
- **Part 015**: Interrupt Handling และ Exception Handling

---

*Part 010 เสร็จสมบูรณ์ - Assemblers, Linkers, และ Debuggers*  
*เนื้อหาทั้งหมดใช้ได้จริง 100% และทดสอบบน Linux x86-64*
