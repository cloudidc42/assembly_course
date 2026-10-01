# Part 004: ประเภทข้อมูลและการจัดการหน่วยความจำ
## (Data Types and Memory Management in Assembly)

**เวลาที่ใช้ศึกษา:** 4-6 ชั่วโมง  
**ระดับ:** พื้นฐาน-กลาง  
**Prerequisites:** Part 001-003 (การติดตั้ง, registers, คำสั่งพื้นฐาน)

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากศึกษา Part นี้แล้ว คุณจะสามารถ:

- อธิบายและใช้งานประเภทข้อมูลต่างๆ ใน Assembly ได้ (Byte, Word, DWORD, QWORD, TBYTE, OWORD, YWORD)
- เข้าใจความแตกต่างระหว่าง Signed และ Unsigned data types
- อธิบาย Memory Layout ของโปรแกรม (Stack, Heap, Code, Data segments)
- เข้าใจ Data Alignment และ Stack Alignment
- ใช้งาน Memory Addressing ในรูปแบบต่างๆ ได้ (absolute, relative, indexed)
- เข้าใจ Endianness และผลกระทบต่อการเก็บข้อมูล
- สร้างและจัดการ Arrays, Structs, และ Pointers ใน Assembly ได้
- ใช้ GDB ตรวจสอบ Memory Layout ของโปรแกรมได้

---

## ส่วนที่ 1: ประเภทข้อมูลพื้นฐาน (Basic Data Types)

### 1.1 Byte (8-bit)

**Byte** คือหน่วยข้อมูลขนาด 8 bits หรือ 1 byte เป็นหน่วยที่เล็กที่สุดที่ CPU สามารถจัดการได้โดยตรงในการ address memory

| ประเภท | ขนาด | ช่วงค่า (Unsigned) | ช่วงค่า (Signed) | NASM keyword |
|--------|------|---------------------|------------------|--------------|
| Byte   | 8-bit  | 0 ถึง 255           | -128 ถึง 127     | `db`, `byte` |
| Word   | 16-bit | 0 ถึง 65,535        | -32,768 ถึง 32,767 | `dw`, `word` |
| DWORD  | 32-bit | 0 ถึง 4,294,967,295 | -2,147,483,648 ถึง 2,147,483,647 | `dd`, `dword` |
| QWORD  | 64-bit | 0 ถึง 18,446,744,073,709,551,615 | -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807 | `dq`, `qword` |

### 1.2 การประกาศข้อมูลใน NASM

ใน NASM เราใช้ directive ต่อไปนี้ในการประกาศข้อมูลใน data section:

```nasm
; ตัวอย่างการประกาศข้อมูลทุกประเภท
; Filename: data_types.asm

section .data
    ; Byte (8-bit) - 1 byte
    my_byte     db  42          ; ประกาศ byte ค่า 42
    my_byte2    db  0xFF        ; ประกาศ byte ค่า 255 (hex)
    my_byte3    db  'A'         ; ประกาศ byte เป็น ASCII code ของ 'A' = 65
    
    ; Word (16-bit) - 2 bytes
    my_word     dw  1000        ; ประกาศ word ค่า 1000
    my_word2    dw  0xABCD      ; ประกาศ word ค่า hex ABCD
    
    ; DWORD (32-bit) - 4 bytes
    my_dword    dd  100000      ; ประกาศ dword ค่า 100,000
    my_dword2   dd  0xDEADBEEF  ; ค่า hex ที่โปรแกรมเมอร์นิยมใช้ test
    
    ; QWORD (64-bit) - 8 bytes
    my_qword    dq  1000000000  ; ประกาศ qword ค่า 1 พันล้าน
    my_qword2   dq  0xCAFEBABEDEADBEEF  ; ค่า 64-bit hex
    
    ; String (array of bytes)
    my_string   db  "Hello, World!", 0  ; null-terminated string
    my_string2  db  72, 101, 108, 108, 111, 0  ; "Hello" ใน ASCII codes

section .bss
    ; ประกาศตัวแปรที่ยังไม่ได้กำหนดค่า
    buffer      resb    256     ; จองพื้นที่ 256 bytes
    num_word    resw    1       ; จองพื้นที่ 1 word (2 bytes)
    num_dword   resd    1       ; จองพื้นที่ 1 dword (4 bytes)
    num_qword   resq    1       ; จองพื้นที่ 1 qword (8 bytes)
```

### 1.3 TBYTE (80-bit) สำหรับ FPU

**TBYTE** หรือ Ten-Byte คือประเภทข้อมูลขนาด 80 bits (10 bytes) ที่ใช้สำหรับ x87 Floating Point Unit (FPU) โดยเฉพาะ

```
โครงสร้างของ 80-bit Extended Precision Float:
┌─────┬──────────────────────┬──────────────────────────────────────────────────────────────┐
│Sign │    Exponent (15 bit) │                    Mantissa (64 bit)                        │
│ (1) │                      │                                                              │
└─────┴──────────────────────┴──────────────────────────────────────────────────────────────┘
  bit 79  bits 78-64                        bits 63-0
```

```nasm
; การใช้ TBYTE ใน NASM
section .data
    float80     dt  3.14159265358979323846  ; ประกาศ 80-bit float
    float80_2   dt  1.0                     ; ค่า 1.0 ใน 80-bit format

section .text
global _start

_start:
    ; โหลดค่า 80-bit float เข้า FPU stack
    fld     tword [float80]     ; โหลด float80 เข้า ST(0)
    fld     tword [float80_2]   ; โหลด float80_2 เข้า ST(0), ST(0) เดิมกลายเป็น ST(1)
    
    fadd                        ; ST(0) = ST(0) + ST(1)
    
    ; เก็บผลลัพธ์กลับ memory
    fstp    tword [float80]     ; เก็บ ST(0) ลง float80 และ pop stack
```

### 1.4 OWORD (128-bit) และ YWORD (256-bit) สำหรับ SIMD

**OWORD** (Octa-word, 128-bit) และ **YWORD** (256-bit) ใช้สำหรับ SIMD instructions ของ SSE และ AVX:

```nasm
; OWORD (128-bit) สำหรับ SSE/SSE2
section .data
    align 16                    ; ต้อง align 16 bytes สำหรับ SSE
    xmm_data    oword   0x000102030405060708090A0B0C0D0E0F
    
    ; หรือประกาศเป็น float array
    float_array dd  1.0, 2.0, 3.0, 4.0   ; 4 ค่า float ใน 128 bits
    
; YWORD (256-bit) สำหรับ AVX
    align 32                    ; ต้อง align 32 bytes สำหรับ AVX
    ymm_data    yword   0       ; 256-bit ค่า 0

section .text
global _start

_start:
    ; SSE: โหลด 4 float พร้อมกัน
    movaps  xmm0, [float_array]  ; โหลด 4 float (128 bits) เข้า xmm0
    
    ; บวก 4 float พร้อมกัน
    movaps  xmm1, [float_array]
    addps   xmm0, xmm1           ; xmm0 = xmm0 + xmm1 (4 floats บวกพร้อมกัน!)
```

---

## ส่วนที่ 2: Signed vs Unsigned

### 2.1 การแสดงค่า Signed ด้วย Two's Complement

ในการแทนค่าลบ CPU ใช้ระบบ **Two's Complement**:

```
ตัวอย่างสำหรับ 8-bit:
ค่า +5  = 00000101
ค่า -5  = 11111011  (Two's complement ของ 00000101)

วิธีคำนวณ Two's Complement:
1. เริ่มจากค่า positive: 00000101 (5)
2. กลับ bits ทั้งหมด:   11111010 (One's complement)
3. บวก 1:               11111011 (-5)

การ verify: 5 + (-5) = 0
  00000101
+ 11111011
----------
 100000000 → overflow bit ทิ้ง → 00000000 = 0 ✓
```

### 2.2 ตัวอย่างโค้ด: Signed vs Unsigned Operations

```nasm
; Filename: signed_unsigned.asm
; สาธิตความแตกต่างระหว่าง Signed และ Unsigned operations
;
; Compile: nasm -f elf64 signed_unsigned.asm -o signed_unsigned.o
; Link:    ld signed_unsigned.o -o signed_unsigned
; Run:     ./signed_unsigned

section .data
    fmt_signed   db  "Signed division: %d / %d = %d", 10, 0
    fmt_unsigned db  "Unsigned division: %u / %u = %u", 10, 0
    newline      db  10, 0

section .text
global _start
extern printf

_start:
    ; ===== Signed Division =====
    ; idiv ใช้สำหรับ signed division
    mov     eax, -100       ; dividend (ตัวตั้ง)
    cdq                     ; sign-extend eax เป็น edx:eax
                            ; (ถ้า eax < 0, edx = 0xFFFFFFFF)
    mov     ecx, 3          ; divisor (ตัวหาร)
    idiv    ecx             ; eax = quotient (ผลหาร), edx = remainder (เศษ)
                            ; เมื่อ -100 / 3: eax = -33, edx = -1
    
    ; ===== Unsigned Division =====
    ; div ใช้สำหรับ unsigned division
    mov     eax, 0xFF       ; 255 (unsigned)
    xor     edx, edx        ; ล้าง edx ก่อน (สำหรับ unsigned ต้อง zero-extend)
    mov     ecx, 10
    div     ecx             ; eax = 25 (255/10), edx = 5 (255 mod 10)
    
    ; ===== Jump Instructions: Signed vs Unsigned =====
    mov     eax, -1         ; -1 ใน signed = 0xFFFFFFFF ใน unsigned
    mov     ebx, 1
    
    cmp     eax, ebx        ; เปรียบเทียบ eax กับ ebx
    
    ; Signed comparison
    jl      signed_less     ; Jump if Less (signed): -1 < 1? ใช่ → jump
    jmp     not_signed_less
    
signed_less:
    ; โค้ดนี้จะทำงาน
    nop

not_signed_less:
    ; Unsigned comparison  
    jb      unsigned_below  ; Jump if Below (unsigned): 0xFFFFFFFF < 1? ไม่ใช่ → ไม่ jump
    jmp     unsigned_above  ; Jump if Above (unsigned): 0xFFFFFFFF > 1? ใช่ → jump
    
unsigned_below:
    nop
    
unsigned_above:
    ; โค้ดนี้จะทำงาน
    nop
    
    ; Exit
    mov     eax, 60         ; syscall: exit
    xor     edi, edi
    syscall
```

### 2.3 Signed vs Unsigned Comparison Table

| Situation | Signed Jump | Unsigned Jump |
|-----------|-------------|---------------|
| a < b     | JL (Jump Less) | JB (Jump Below) |
| a <= b    | JLE (Jump Less/Equal) | JBE (Jump Below/Equal) |
| a > b     | JG (Jump Greater) | JA (Jump Above) |
| a >= b    | JGE (Jump Greater/Equal) | JAE (Jump Above/Equal) |
| a == b    | JE (Jump Equal) | JE (Jump Equal) |
| a != b    | JNE (Jump Not Equal) | JNE (Jump Not Equal) |

---

## ส่วนที่ 3: Memory Layout ของโปรแกรม

### 3.1 โครงสร้าง Virtual Memory

เมื่อโปรแกรมทำงาน OS จะจัดสรร Virtual Memory ให้ดังนี้:

```
Virtual Address Space (64-bit Linux)
┌─────────────────────────────────┐  ← High address (0x7FFFFFFFFFFF)
│           Stack                 │  ← เติบโตลงมา (↓)
│         (grows down)            │
│  ┌─────────────────────────┐    │
│  │  Stack Frame n          │    │
│  │  - Return address       │    │
│  │  - Saved registers      │    │
│  │  - Local variables      │    │
│  └─────────────────────────┘    │
│                                  │
│       ↓ Stack grows down        │
│                                  │
│         (free space)            │
│                                  │
│       ↑ Heap grows up           │
│                                  │
│  ┌─────────────────────────┐    │
│  │         Heap            │    │  ← เติบโตขึ้นไป (↑)
│  │   (dynamic memory)      │    │
│  └─────────────────────────┘    │
├─────────────────────────────────┤
│         .bss segment            │  ← ตัวแปร uninitialized
│   (uninitialized globals)       │
├─────────────────────────────────┤
│         .data segment           │  ← ตัวแปร initialized
│   (initialized globals)         │
├─────────────────────────────────┤
│         .text segment           │  ← โปรแกรม code (read-only)
│         (program code)          │
├─────────────────────────────────┤
│         .rodata segment         │  ← ข้อมูล read-only
│       (constants, strings)      │
└─────────────────────────────────┘  ← Low address (0x400000)
```

### 3.2 ตัวอย่างโค้ดแสดง Memory Layout

```nasm
; Filename: memory_layout.asm
; สาธิต memory layout ของโปรแกรม Assembly
;
; Compile: nasm -f elf64 memory_layout.asm -o memory_layout.o
; Link:    gcc memory_layout.o -o memory_layout -no-pie
; Run:     ./memory_layout

section .data
    ; Initialized data - อยู่ใน .data segment
    global_var1  dd  100        ; Global variable ค่า 100
    global_var2  dq  0xDEADBEEF ; Global variable 64-bit
    
    ; สตริงสำหรับ output
    msg_text     db  "Text section addr: 0x%016llx", 10, 0
    msg_data     db  "Data section addr: 0x%016llx", 10, 0
    msg_bss      db  "BSS  section addr: 0x%016llx", 10, 0
    msg_stack    db  "Stack addr:        0x%016llx", 10, 0
    msg_heap     db  "Heap addr:         0x%016llx", 10, 0

section .bss
    ; Uninitialized data - อยู่ใน .bss segment
    global_buf   resb    1024    ; Buffer ขนาด 1KB

section .text
global main
extern printf, malloc, free

main:
    push    rbp
    mov     rbp, rsp
    
    ; แสดง address ของ text section
    lea     rsi, [main]         ; address ของ function main (ใน .text)
    lea     rdi, [msg_text]
    xor     eax, eax
    call    printf
    
    ; แสดง address ของ data section
    lea     rsi, [global_var1]  ; address ของ global variable (ใน .data)
    lea     rdi, [msg_data]
    xor     eax, eax
    call    printf
    
    ; แสดง address ของ bss section
    lea     rsi, [global_buf]   ; address ของ buffer (ใน .bss)
    lea     rdi, [msg_bss]
    xor     eax, eax
    call    printf
    
    ; แสดง address ของ stack
    mov     rsi, rsp            ; address ของ stack pointer ปัจจุบัน
    lea     rdi, [msg_stack]
    xor     eax, eax
    call    printf
    
    ; Allocate heap memory และแสดง address
    mov     rdi, 256            ; ขอ malloc 256 bytes
    call    malloc
    mov     rbx, rax            ; เก็บ pointer ไว้ใน rbx
    
    mov     rsi, rax            ; address ของ heap memory
    lea     rdi, [msg_heap]
    xor     eax, eax
    call    printf
    
    ; Free heap memory
    mov     rdi, rbx
    call    free
    
    pop     rbp
    xor     eax, eax
    ret

; Expected output:
; Text section addr: 0x0000000000401150
; Data section addr: 0x0000000000404020
; BSS  section addr: 0x0000000000404060
; Stack addr:        0x00007ffd12345678
; Heap addr:         0x000055a123456260
; (ค่า address จะแตกต่างกันในแต่ละเครื่อง)
```

### 3.3 Stack Growth Direction และ Alignment

Stack ใน x86-64 เติบโตจาก **High address** ลงมา **Low address**:

```nasm
; Filename: stack_demo.asm
; สาธิตการทำงานของ Stack
;
; Compile: nasm -f elf64 stack_demo.asm -o stack_demo.o
; Link:    gcc stack_demo.o -o stack_demo
; Run:     ./stack_demo

section .data
    fmt_rsp     db  "RSP before push: 0x%016llx", 10, 0
    fmt_rsp2    db  "RSP after push:  0x%016llx", 10, 0
    fmt_rsp3    db  "RSP after pop:   0x%016llx", 10, 0
    fmt_val     db  "Value on stack:  %d", 10, 0

section .text
global main
extern printf

main:
    push    rbp
    mov     rbp, rsp
    
    ; แสดง RSP ก่อน push
    mov     rsi, rsp
    lea     rdi, [fmt_rsp]
    xor     eax, eax
    call    printf
    
    ; Push ค่าลง stack
    push    qword 42            ; RSP ลดลง 8 bytes (จาก high → low)
    
    ; แสดง RSP หลัง push (ค่าจะน้อยกว่า 8 bytes)
    mov     rsi, rsp
    lea     rdi, [fmt_rsp2]
    xor     eax, eax
    call    printf
    
    ; แสดงค่าบน stack
    mov     rsi, [rsp]          ; อ่านค่าจาก top of stack
    lea     rdi, [fmt_val]
    xor     eax, eax
    call    printf
    
    ; Pop ค่าออกจาก stack
    pop     rax                 ; RSP เพิ่มขึ้น 8 bytes (กลับไป high)
    
    ; แสดง RSP หลัง pop (ค่าจะกลับมาเท่าเดิม)
    mov     rsi, rsp
    lea     rdi, [fmt_rsp3]
    xor     eax, eax
    call    printf
    
    pop     rbp
    xor     eax, eax
    ret

; ===== Stack Frame Structure =====
; 
; ก่อน function call:
; ┌─────────────────┐ ← high address
; │   ...           │
; │   caller RSP    │ ← RSP ชี้ที่นี่
; └─────────────────┘
;
; หลัง CALL instruction:
; ┌─────────────────┐
; │   ...           │
; │   return addr   │ ← RSP ชี้ที่นี่ (RSP - 8)
; └─────────────────┘
;
; หลัง push rbp / mov rbp, rsp:
; ┌─────────────────┐
; │   ...           │
; │   return addr   │
; │   saved rbp     │ ← RSP ชี้ที่นี่ (RSP - 16)
; └─────────────────┘
```

---

## ส่วนที่ 4: Data Alignment

### 4.1 ทำไม Alignment ถึงสำคัญ?

CPU อ่านข้อมูลได้เร็วที่สุดเมื่อข้อมูลอยู่ใน **aligned address** คือ address ที่หารด้วยขนาดข้อมูลลงตัว:

```
การ Access ข้อมูล 4 bytes:
- Aligned (address 0x4):    0x0004 → อ่าน 1 ครั้ง ✓ เร็ว
- Misaligned (address 0x3): 0x0003 → ต้องอ่าน 2 ครั้ง แล้วรวมกัน ✗ ช้า

Memory Layout ของ 4-byte int:
Address: 0x00  0x01  0x02  0x03  0x04  0x05  0x06  0x07
         ──────────────────────────────────────────────────
Aligned: [────────── int ──────────]                     ← อ่านครั้งเดียว
         0x00  0x01  0x02  0x03

Misaligned:     [── part1 ──] [── part2 ──]             ← อ่าน 2 ครั้ง
                0x03  0x04  0x05  0x06
```

### 4.2 Alignment Rules

| ประเภทข้อมูล | ขนาด | Alignment ที่แนะนำ |
|-------------|------|-------------------|
| char/byte   | 1 byte | 1 byte (ไม่ต้อง align พิเศษ) |
| short/word  | 2 bytes | 2 bytes |
| int/dword   | 4 bytes | 4 bytes |
| long/qword  | 8 bytes | 8 bytes |
| SSE (XMM)  | 16 bytes | 16 bytes |
| AVX (YMM)  | 32 bytes | 32 bytes |
| AVX-512 (ZMM) | 64 bytes | 64 bytes |

### 4.3 ตัวอย่างการใช้ align directive ใน NASM

```nasm
; Filename: alignment_demo.asm
; สาธิต Data Alignment และผลกระทบต่อ performance
;
; Compile: nasm -f elf64 alignment_demo.asm -o alignment_demo.o
; Link:    gcc alignment_demo.o -o alignment_demo
; Run:     ./alignment_demo

section .data
    ; ตัวอย่าง aligned data
    align 4                     ; align ถึง 4-byte boundary
    int_val     dd  12345678    ; DWORD ที่ aligned
    
    align 8                     ; align ถึง 8-byte boundary
    long_val    dq  1234567890  ; QWORD ที่ aligned
    
    align 16                    ; align ถึง 16-byte boundary สำหรับ SSE
    sse_data    dd  1.0, 2.0, 3.0, 4.0  ; 4 floats สำหรับ SSE
    
    align 32                    ; align ถึง 32-byte boundary สำหรับ AVX
    avx_data    dd  1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0  ; 8 floats สำหรับ AVX

    ; ตัวอย่าง misaligned data (ไม่ดี!)
    bad_byte    db  0xFF        ; byte เดี่ยว ทำให้ต่อไปนี้ misaligned
    ; หลัง bad_byte, address จะเป็น odd number
    ; ถ้าประกาศ word หลัง bad_byte โดยไม่ align จะเป็น misaligned!
    
    align 2                     ; แก้ปัญหาด้วย align
    good_word   dw  1000        ; Word ที่ aligned ถูกต้อง

section .text
global main
extern printf

main:
    push    rbp
    mov     rbp, rsp
    
    ; ===== SSE operation กับ aligned data =====
    movaps  xmm0, [sse_data]    ; movaps = Move Aligned Packed Single-precision
                                 ; ถ้า data ไม่ aligned → General Protection Fault!
    
    ; คูณแต่ละ float ด้วย 2.0
    movaps  xmm1, xmm0
    addps   xmm0, xmm1          ; xmm0 = [2.0, 4.0, 6.0, 8.0]
    
    ; ===== เปรียบเทียบ movaps vs movups =====
    ; movaps = aligned, เร็วกว่า แต่ crash ถ้าไม่ aligned
    ; movups = unaligned, ช้ากว่าแต่ปลอดภัยกว่า
    
    pop     rbp
    xor     eax, eax
    ret
```

### 4.4 Stack Alignment ใน x86-64 System V ABI

```nasm
; ก่อน CALL instruction, RSP ต้อง aligned ที่ 16 bytes
; แต่ CALL จะ push return address (8 bytes) ดังนั้น
; ภายใน function, RSP จะเป็น 16n - 8 (misaligned by 8)
; นั่นคือทำไมเราต้อง push rbp ก่อน เพื่อทำให้ RSP กลับมา 16n-aligned

; Example function prologue:
my_function:
    push    rbp         ; RSP - 8 → เนื่องจาก call push return addr (8 bytes)
                        ; แล้ว push rbp อีก 8 bytes → รวม -16 → aligned!
    mov     rbp, rsp    ; บันทึก frame pointer
    
    ; ถ้าต้องการ local variables, ต้อง sub rsp ทีละ 16 bytes
    sub     rsp, 32     ; จอง 32 bytes สำหรับ local vars (ต้องเป็น multiple ของ 16)
    
    ; ... body of function ...
    
    add     rsp, 32     ; คืน stack space
    pop     rbp
    ret
```

---

## ส่วนที่ 5: Memory Addressing Modes

### 5.1 รูปแบบการ Address Memory ทั้งหมด

ใน x86-64 สามารถ address memory ได้หลายวิธี:

```nasm
; Filename: addressing_modes.asm
; สาธิต Memory Addressing Modes ทั้งหมดใน x86-64
;
; Compile: nasm -f elf64 addressing_modes.asm -o addressing_modes.o
; Link:    gcc addressing_modes.o -o addressing_modes
; Run:     ./addressing_modes

section .data
    array       dd  10, 20, 30, 40, 50  ; array ของ 5 integers
    matrix      dd  1, 2, 3, 4, 5, 6, 7, 8, 9  ; matrix 3x3
    value       dd  100

section .text
global main

main:
    push    rbp
    mov     rbp, rsp
    
    ; ===== 1. Immediate Addressing =====
    ; ค่า operand คือค่าตรงๆ ไม่ได้อ้างถึง memory
    mov     eax, 42             ; eax = 42 (immediate value)
    mov     ebx, 0xFF           ; ebx = 255
    
    ; ===== 2. Register Addressing =====
    ; operand คือ register
    mov     ecx, eax            ; ecx = eax (register to register)
    add     eax, ebx            ; eax = eax + ebx
    
    ; ===== 3. Direct (Absolute) Addressing =====
    ; address คือค่า constant
    mov     eax, [value]        ; โหลดค่าจาก address ของ value
    mov     [value], ebx        ; เก็บ ebx ลง address ของ value
    
    ; ===== 4. Register Indirect Addressing =====
    ; Register เก็บ address
    lea     rbx, [array]        ; rbx = address ของ array
    mov     eax, [rbx]          ; โหลดค่าจาก address ใน rbx (= array[0] = 10)
    mov     [rbx], dword 99     ; เก็บ 99 ไว้ที่ array[0]
    
    ; ===== 5. Base + Displacement Addressing =====
    ; Base register + constant offset
    lea     rbx, [array]        ; rbx = address ของ array
    mov     eax, [rbx + 4]      ; โหลด array[1] (offset 4 bytes จาก start)
    mov     eax, [rbx + 8]      ; โหลด array[2] (offset 8 bytes)
    mov     eax, [rbx + 12]     ; โหลด array[3]
    mov     eax, [rbx + 16]     ; โหลด array[4]
    
    ; ===== 6. Base + Index Addressing =====
    ; Base register + Index register
    lea     rbx, [array]        ; base = address ของ array
    mov     rcx, 2              ; index = 2
    mov     eax, [rbx + rcx*4]  ; โหลด array[2] (index * sizeof(int) = 2*4 = 8)
    ; [base + index * scale] โดย scale = 1, 2, 4, หรือ 8
    
    ; ===== 7. Base + Index + Displacement Addressing =====
    ; Base + Index * Scale + Displacement
    lea     rbx, [matrix]       ; base = address ของ matrix
    mov     rcx, 1              ; row = 1
    mov     rdx, 2              ; col = 2
    ; element [row][col] = matrix + row*3*4 + col*4
    ; สำหรับ row=1, col=2: matrix + 1*12 + 2*4 = matrix + 20
    mov     eax, [rbx + rcx*12 + rdx*4]    ; matrix[1][2] = 6
    
    ; ===== 8. RIP-relative Addressing (x86-64 เท่านั้น) =====
    ; อ้างถึง data แบบ relative to Instruction Pointer
    ; ใช้ใน Position Independent Code (PIC)
    lea     rax, [rel value]    ; rax = address ของ value (relative to RIP)
    mov     eax, [rel value]    ; โหลดค่าจาก value แบบ RIP-relative
    
    pop     rbp
    xor     eax, eax
    ret

; สรุป Address Forms ใน Intel syntax:
; [register]                        - Register indirect
; [register + displacement]         - Base + displacement
; [register + register]             - Base + index
; [register + register*scale]       - Base + scaled index
; [register + register*scale + disp] - Full form
; [address]                         - Direct addressing
; scale = 1, 2, 4, 8 เท่านั้น!
```

---

## ส่วนที่ 6: Endianness

### 6.1 Little-Endian vs Big-Endian

**Endianness** คือลำดับการเก็บ bytes ใน memory:

```
ตัวอย่าง: เก็บค่า 0x12345678 (32-bit integer)

Little-Endian (x86):
Address:  0x00  0x01  0x02  0x03
Data:     0x78  0x56  0x34  0x12
           LSB                MSB
(Least Significant Byte อยู่ที่ address ต่ำสุด)

Big-Endian (Network byte order):
Address:  0x00  0x01  0x02  0x03
Data:     0x12  0x34  0x56  0x78
           MSB                LSB
(Most Significant Byte อยู่ที่ address ต่ำสุด)
```

### 6.2 ตัวอย่างโค้ด: Endianness ใน Assembly

```nasm
; Filename: endianness_demo.asm
; สาธิต Endianness และการแปลงค่า
;
; Compile: nasm -f elf64 endianness_demo.asm -o endianness_demo.o
; Link:    gcc endianness_demo.o -o endianness_demo
; Run:     ./endianness_demo

section .data
    value32     dd  0x12345678  ; 32-bit value
    value64     dq  0x0102030405060708  ; 64-bit value
    
    fmt_byte    db  "Byte at offset %d: 0x%02x", 10, 0
    fmt_val     db  "Value = 0x%08x", 10, 0

section .text
global main
extern printf

main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    
    ; อ่านค่า 32-bit ที่เก็บใน memory
    ; ใน Little-Endian: 0x12345678 จะเก็บเป็น 78 56 34 12
    
    lea     rbx, [value32]
    
    ; อ่านทีละ byte เพื่อดู endianness
    movzx   esi, byte [rbx]         ; byte แรก (offset 0) = 0x78 (LSB!)
    mov     edi, 0
    lea     rdi, [fmt_byte]
    call    printf
    
    movzx   esi, byte [rbx + 1]     ; byte สอง (offset 1) = 0x56
    mov     edi, 1
    lea     rdi, [fmt_byte]
    call    printf
    
    movzx   esi, byte [rbx + 2]     ; byte สาม (offset 2) = 0x34
    mov     edi, 2
    lea     rdi, [fmt_byte]
    call    printf
    
    movzx   esi, byte [rbx + 3]     ; byte สี่ (offset 3) = 0x12 (MSB!)
    mov     edi, 3
    lea     rdi, [fmt_byte]
    call    printf
    
    ; ===== การแปลง Endianness (BSWAP instruction) =====
    mov     eax, [value32]          ; โหลด 0x12345678
    bswap   eax                     ; swap bytes: 0x78563412
    ; ตอนนี้ eax = 0x78563412
    ; ซึ่งเป็น big-endian representation ของ 0x12345678
    
    ; ===== Network byte order conversion =====
    ; htonl = host to network long (little-endian → big-endian)
    ; ntohl = network to host long (big-endian → little-endian)
    ; ทั้งสองทำ bswap นั่นเอง
    
    pop     rbp
    xor     eax, eax
    ret

; Output:
; Byte at offset 0: 0x78   ← LSB อยู่ที่ address ต่ำสุด (Little-endian!)
; Byte at offset 1: 0x56
; Byte at offset 2: 0x34
; Byte at offset 3: 0x12   ← MSB อยู่ที่ address สูงสุด
```

---

## ส่วนที่ 7: Global vs Local Variables

### 7.1 Global Variables ใน Assembly

```nasm
; Filename: variables_demo.asm
; สาธิต Global vs Local Variables
;
; Compile: nasm -f elf64 variables_demo.asm -o variables_demo.o
; Link:    gcc variables_demo.o -o variables_demo
; Run:     ./variables_demo

; ===== GLOBAL VARIABLES อยู่ใน .data หรือ .bss segment =====
section .data
    ; Global initialized variable
    global_int      dd  42          ; int global_int = 42;
    global_char     db  'A'         ; char global_char = 'A';
    global_str      db  "Hello", 0  ; char* global_str = "Hello";
    global_float    dd  3.14        ; float global_float = 3.14;
    
    ; สำหรับ output
    fmt_global  db  "Global int: %d, Global char: %c", 10, 0
    fmt_local   db  "Local int: %d", 10, 0
    fmt_ptr     db  "Pointer value: %d", 10, 0

section .bss
    ; Global uninitialized variable (initialized to 0 โดย OS)
    global_buf      resb    100     ; char global_buf[100];
    global_counter  resd    1       ; int global_counter; (= 0)

section .text
global main
extern printf

; ===== LOCAL VARIABLES อยู่ใน Stack Frame =====
main:
    push    rbp
    mov     rbp, rsp
    
    ; จอง stack space สำหรับ local variables
    ; local_int (4 bytes) + local_char (1 byte + 3 padding) = 8 bytes
    ; ต้อง align ที่ 16 bytes → จอง 16 bytes
    sub     rsp, 16
    
    ; ประกาศ local variables บน stack
    ; rbp-4  = local_int (4 bytes)
    ; rbp-5  = local_char (1 byte)
    ; ตำแหน่งอ้างอิงจาก RBP (base pointer)
    
    ; กำหนดค่า local variables
    mov     dword [rbp - 4], 100    ; local_int = 100;
    mov     byte  [rbp - 5], 'B'    ; local_char = 'B';
    
    ; ===== ใช้งาน Global Variables =====
    mov     esi, [global_int]       ; โหลด global_int
    movzx   edx, byte [global_char] ; โหลด global_char
    lea     rdi, [fmt_global]
    xor     eax, eax
    call    printf
    
    ; ===== แก้ไข Global Variable =====
    mov     dword [global_int], 999 ; global_int = 999;
    inc     dword [global_counter]  ; global_counter++;
    
    ; ===== ใช้งาน Local Variables =====
    mov     esi, [rbp - 4]          ; โหลด local_int
    lea     rdi, [fmt_local]
    xor     eax, eax
    call    printf
    
    ; Cleanup
    add     rsp, 16                 ; คืน stack space
    pop     rbp
    xor     eax, eax
    ret
```

---

## ส่วนที่ 8: Arrays ใน Assembly

### 8.1 การประกาศและเข้าถึง Arrays

```nasm
; Filename: arrays_demo.asm
; สาธิตการใช้งาน Arrays ใน Assembly
;
; Compile: nasm -f elf64 arrays_demo.asm -o arrays_demo.o
; Link:    gcc arrays_demo.o -o arrays_demo
; Run:     ./arrays_demo

section .data
    ; ===== Static Arrays =====
    ; int arr[5] = {10, 20, 30, 40, 50};
    int_arr     dd  10, 20, 30, 40, 50
    INT_SIZE    equ 4                   ; sizeof(int) = 4 bytes
    INT_COUNT   equ 5                   ; จำนวน elements
    
    ; char str[] = "Hello";
    char_arr    db  'H', 'e', 'l', 'l', 'o', 0
    
    ; float arr[] = {1.0, 2.0, 3.0, 4.0};
    align 16
    float_arr   dd  1.0, 2.0, 3.0, 4.0
    
    ; 2D array: int matrix[3][4] = {...}
    ; เก็บแบบ row-major order (C style)
    matrix      dd  1,  2,  3,  4,   ; row 0
                dd  5,  6,  7,  8,   ; row 1
                dd  9, 10, 11, 12    ; row 2
    COLS        equ 4                 ; จำนวน columns
    
    fmt_elem    db  "arr[%d] = %d", 10, 0
    fmt_matrix  db  "matrix[%d][%d] = %d", 10, 0

section .text
global main
extern printf

; ===== ฟังก์ชัน: Sum of Array =====
; input: rdi = pointer to array, rsi = count
; output: rax = sum
sum_array:
    push    rbp
    mov     rbp, rsp
    
    xor     eax, eax        ; sum = 0
    xor     ecx, ecx        ; i = 0
    
.loop:
    cmp     ecx, esi        ; i < count?
    jge     .done
    
    mov     edx, [rdi + rcx*4]  ; edx = arr[i] (4 bytes per element)
    add     eax, edx            ; sum += arr[i]
    inc     ecx                 ; i++
    jmp     .loop
    
.done:
    pop     rbp
    ret

; ===== ฟังก์ชัน: Print Array =====
; input: rdi = pointer to array, rsi = count
print_array:
    push    rbp
    mov     rbp, rsp
    push    rbx                 ; save rbx
    push    r12
    push    r13
    
    mov     rbx, rdi            ; rbx = array pointer
    mov     r12, rsi            ; r12 = count
    xor     r13, r13            ; r13 = index = 0
    
.print_loop:
    cmp     r13, r12
    jge     .print_done
    
    ; printf("arr[%d] = %d\n", i, arr[i]);
    mov     rsi, r13            ; arg2 = index
    mov     edx, [rbx + r13*4] ; arg3 = arr[i]
    lea     rdi, [fmt_elem]     ; arg1 = format string
    xor     eax, eax
    call    printf
    
    inc     r13
    jmp     .print_loop
    
.print_done:
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret

main:
    push    rbp
    mov     rbp, rsp
    
    ; ===== Access Array Elements =====
    ; element ที่ i = base + i * sizeof(element)
    lea     rbx, [int_arr]      ; rbx = base address ของ int_arr
    
    ; arr[0]
    mov     eax, [rbx]          ; eax = int_arr[0] = 10
    ; arr[1]
    mov     eax, [rbx + 4]      ; eax = int_arr[1] = 20
    ; arr[2]
    mov     eax, [rbx + 8]      ; eax = int_arr[2] = 30
    
    ; Array index ด้วย register
    mov     rcx, 3              ; index = 3
    mov     eax, [rbx + rcx*4]  ; eax = int_arr[3] = 40
    
    ; ===== Print Array =====
    lea     rdi, [int_arr]
    mov     rsi, INT_COUNT
    call    print_array
    
    ; ===== Calculate Sum =====
    lea     rdi, [int_arr]
    mov     esi, INT_COUNT
    call    sum_array
    ; rax = 10+20+30+40+50 = 150
    
    ; ===== 2D Array Access =====
    ; element [row][col] = base + (row * COLS + col) * sizeof(int)
    lea     rbx, [matrix]
    
    mov     rcx, 1              ; row = 1
    mov     rdx, 2              ; col = 2
    ; offset = (1 * 4 + 2) * 4 = 6 * 4 = 24
    imul    rcx, COLS           ; rcx = row * COLS
    add     rcx, rdx            ; rcx = row * COLS + col
    mov     eax, [rbx + rcx*4]  ; eax = matrix[1][2] = 7
    
    pop     rbp
    xor     eax, eax
    ret
```

---

## ส่วนที่ 9: Structs/Records ใน Assembly

### 9.1 การสร้าง Struct ใน NASM ด้วย struc

```nasm
; Filename: structs_demo.asm
; สาธิตการใช้งาน Structs ใน Assembly
;
; Compile: nasm -f elf64 structs_demo.asm -o structs_demo.o
; Link:    gcc structs_demo.o -o structs_demo
; Run:     ./structs_demo

; ===== ประกาศ Struct =====
; เทียบกับ C:
; struct Point {
;     int x;      // offset 0, 4 bytes
;     int y;      // offset 4, 4 bytes
; };
struc Point
    .x    resd    1   ; int x; (4 bytes)
    .y    resd    1   ; int y; (4 bytes)
endstruc
; Point_size = 8 bytes

; struct Person {
;     char name[32]; // offset 0,  32 bytes
;     int  age;      // offset 32,  4 bytes
;     float height;  // offset 36,  4 bytes
; };
struc Person
    .name   resb    32  ; char name[32]
    .age    resd    1   ; int age
    .height resd    1   ; float height (ใช้ resd สำหรับ float)
endstruc
; Person_size = 40 bytes

section .data
    ; ===== Initialized Struct =====
    ; สร้าง Point struct ด้วย istruc
    point1  istruc Point
        at Point.x, dd  10      ; point1.x = 10
        at Point.y, dd  20      ; point1.y = 20
    iend
    
    ; สร้าง Point อีกตัว
    point2  istruc Point
        at Point.x, dd  5
        at Point.y, dd  15
    iend
    
    fmt_point   db  "Point(%d, %d)", 10, 0
    fmt_dist    db  "Distance squared: %d", 10, 0

section .bss
    ; Uninitialized struct
    result_point    resb    Point_size  ; Point result;
    my_person       resb    Person_size ; Person my_person;

section .text
global main
extern printf

; ===== ฟังก์ชัน: Print Point =====
; input: rdi = pointer to Point struct
print_point:
    push    rbp
    mov     rbp, rsp
    
    mov     rbx, rdi                    ; rbx = pointer to Point
    mov     esi, [rbx + Point.x]        ; arg2 = point.x
    mov     edx, [rbx + Point.y]        ; arg3 = point.y
    lea     rdi, [fmt_point]
    xor     eax, eax
    call    printf
    
    pop     rbp
    ret

; ===== ฟังก์ชัน: Add Points =====
; input: rdi = Point* a, rsi = Point* b, rdx = Point* result
add_points:
    push    rbp
    mov     rbp, rsp
    
    ; result.x = a.x + b.x
    mov     eax, [rdi + Point.x]    ; eax = a.x
    add     eax, [rsi + Point.x]    ; eax = a.x + b.x
    mov     [rdx + Point.x], eax    ; result.x = a.x + b.x
    
    ; result.y = a.y + b.y
    mov     eax, [rdi + Point.y]    ; eax = a.y
    add     eax, [rsi + Point.y]    ; eax = a.y + b.y
    mov     [rdx + Point.y], eax    ; result.y = a.y + b.y
    
    pop     rbp
    ret

main:
    push    rbp
    mov     rbp, rsp
    
    ; ===== พิมพ์ point1 =====
    lea     rdi, [point1]
    call    print_point             ; Point(10, 20)
    
    ; ===== พิมพ์ point2 =====
    lea     rdi, [point2]
    call    print_point             ; Point(5, 15)
    
    ; ===== บวก point1 + point2 =====
    lea     rdi, [point1]
    lea     rsi, [point2]
    lea     rdx, [result_point]
    call    add_points
    
    ; ===== พิมพ์ result =====
    lea     rdi, [result_point]
    call    print_point             ; Point(15, 35)
    
    ; ===== Struct Array =====
    ; สร้าง array ของ Points บน stack
    sub     rsp, Point_size * 3     ; จอง space สำหรับ 3 Points
    
    ; points[0].x = 1, points[0].y = 2
    mov     dword [rsp + Point_size*0 + Point.x], 1
    mov     dword [rsp + Point_size*0 + Point.y], 2
    
    ; points[1].x = 3, points[1].y = 4
    mov     dword [rsp + Point_size*1 + Point.x], 3
    mov     dword [rsp + Point_size*1 + Point.y], 4
    
    ; points[2].x = 5, points[2].y = 6
    mov     dword [rsp + Point_size*2 + Point.x], 5
    mov     dword [rsp + Point_size*2 + Point.y], 6
    
    ; print points[1]
    lea     rdi, [rsp + Point_size*1]
    call    print_point
    
    add     rsp, Point_size * 3     ; คืน stack space
    
    pop     rbp
    xor     eax, eax
    ret

; Output:
; Point(10, 20)
; Point(5, 15)
; Point(15, 35)
; Point(3, 4)
```

### 9.2 Struct Padding และ Packing

```nasm
; ตัวอย่าง Struct Padding:
; struct BadLayout {
;     char   a;     // 1 byte  → offset 0
;     // 3 bytes padding
;     int    b;     // 4 bytes → offset 4
;     char   c;     // 1 byte  → offset 8
;     // 3 bytes padding
;     int    d;     // 4 bytes → offset 12
; }; // total = 16 bytes!

; struct GoodLayout {
;     int    b;     // 4 bytes → offset 0
;     int    d;     // 4 bytes → offset 4
;     char   a;     // 1 byte  → offset 8
;     char   c;     // 1 byte  → offset 9
;     // 2 bytes padding
; }; // total = 12 bytes!

; NASM struct:
struc BadLayout
    .a    resb    1
    align 4             ; padding ให้ครบ 4 bytes
    .b    resd    1
    .c    resb    1
    align 4             ; padding ให้ครบ 4 bytes
    .d    resd    1
endstruc
```

---

## ส่วนที่ 10: Pointers และ Pointer Arithmetic

### 10.1 Pointers คืออะไร?

**Pointer** คือตัวแปรที่เก็บ **address** ของตัวแปรอื่น ใน Assembly ทุก register ที่ใช้ชี้ไปยัง memory เป็น pointer ทั้งนั้น

```nasm
; Filename: pointers_demo.asm
; สาธิต Pointers และ Pointer Arithmetic
;
; Compile: nasm -f elf64 pointers_demo.asm -o pointers_demo.o
; Link:    gcc pointers_demo.o -o pointers_demo
; Run:     ./pointers_demo

section .data
    value       dd  42              ; int value = 42;
    array       dd  10, 20, 30, 40, 50
    
    fmt_val     db  "Value: %d", 10, 0
    fmt_ptr     db  "Pointer address: 0x%016llx", 10, 0
    fmt_deref   db  "Dereferenced: %d", 10, 0

section .text
global main
extern printf

; ===== ฟังก์ชัน: increment value through pointer =====
; input: rdi = pointer to int
; ทำ (*ptr)++
increment_ptr:
    push    rbp
    mov     rbp, rsp
    
    ; dereference pointer และ increment
    inc     dword [rdi]     ; (*ptr)++
    
    pop     rbp
    ret

main:
    push    rbp
    mov     rbp, rsp
    
    ; ===== Basic Pointer Operations =====
    
    ; int* ptr = &value;  (ptr เก็บ address ของ value)
    lea     rbx, [value]        ; rbx = &value
    
    ; printf("Pointer address: 0x%016llx\n", ptr);
    mov     rsi, rbx
    lea     rdi, [fmt_ptr]
    xor     eax, eax
    call    printf
    
    ; *ptr = 100;  (กำหนดค่าผ่าน pointer)
    mov     dword [rbx], 100    ; *ptr = 100
    
    ; printf("Value: %d\n", value);
    mov     esi, [value]
    lea     rdi, [fmt_val]
    xor     eax, eax
    call    printf              ; Value: 100
    
    ; ===== Pointer Arithmetic =====
    lea     rbx, [array]        ; ptr = array (ชี้ที่ array[0])
    
    ; ptr++ สำหรับ int* ทำให้ address เพิ่ม sizeof(int) = 4
    mov     esi, [rbx]          ; *ptr = array[0] = 10
    add     rbx, 4              ; ptr++ (4 bytes สำหรับ int)
    mov     esi, [rbx]          ; *ptr = array[1] = 20
    add     rbx, 4              ; ptr++
    mov     esi, [rbx]          ; *ptr = array[2] = 30
    
    ; ptr + n  = address + n * sizeof(element)
    lea     rbx, [array]        ; กลับไปที่ array[0]
    mov     rcx, 3              ; n = 3
    mov     esi, [rbx + rcx*4]  ; *(ptr+3) = array[3] = 40
    
    ; ===== ส่ง Pointer ไปยัง Function =====
    lea     rdi, [value]        ; ส่ง address ของ value
    call    increment_ptr       ; value จะถูก increment เป็น 101
    
    mov     esi, [value]
    lea     rdi, [fmt_val]
    xor     eax, eax
    call    printf              ; Value: 101
    
    ; ===== Pointer to Pointer (Double Pointer) =====
    lea     rbx, [value]        ; rbx = &value (pointer to int)
    ; ถ้าจะสร้าง pointer to pointer (int**):
    ; เก็บ address ของ rbx ลงใน stack
    push    rbx                 ; stack now contains address of value
    mov     rcx, rsp            ; rcx = address of (pointer to value)
    ; *(*rcx) = dereference twice
    mov     rbx, [rcx]          ; rbx = *rcx (= &value)
    mov     eax, [rbx]          ; eax = **rcx (= value = 101)
    pop     rbx                 ; cleanup
    
    pop     rbp
    xor     eax, eax
    ret
```

---

## ส่วนที่ 11: ดู Memory Layout ด้วย GDB

### 11.1 คำสั่ง GDB สำหรับตรวจสอบ Memory

```bash
# Compile with debug info
nasm -f elf64 -g -F dwarf program.asm -o program.o
gcc program.o -o program -g

# เริ่มต้น GDB
gdb ./program

# คำสั่ง GDB ที่ใช้บ่อย
(gdb) break main          # breakpoint ที่ main
(gdb) run                 # รันโปรแกรม
(gdb) info registers      # แสดง register ทั้งหมด
(gdb) info registers rsp  # แสดง RSP เท่านั้น

# ===== ดู Memory =====
# x/<count><format><size> <address>
# format: d=decimal, x=hex, c=char, s=string, i=instruction
# size: b=byte, h=halfword(2), w=word(4), g=giant(8)

(gdb) x/4xw $rsp          # ดู 4 words (32-bit) บน stack ในรูป hex
(gdb) x/8xb $rsp          # ดู 8 bytes ในรูป hex
(gdb) x/4xg $rsp          # ดู 4 quadwords (64-bit)
(gdb) x/s $rdi            # ดู string ที่ rdi ชี้
(gdb) x/10i $pc           # ดู 10 instructions ถัดจาก PC

# ===== ตัวอย่างการดู Struct =====
(gdb) x/8xb &point1       # ดู 8 bytes ของ point1 struct
(gdb) p point1.x          # print field x ของ point1 (ถ้ามี debug info)

# ===== ดู Stack Frame =====
(gdb) info frame          # แสดงข้อมูล stack frame ปัจจุบัน
(gdb) backtrace           # แสดง call stack
(gdb) info locals         # แสดง local variables

# ===== Layout Visualization =====
(gdb) layout regs         # แสดง register window
(gdb) layout asm          # แสดง assembly window
(gdb) layout src          # แสดง source code window
```

### 11.2 ตัวอย่าง GDB Session เต็ม

```nasm
; Filename: gdb_demo.asm
; ไฟล์สำหรับทดสอบ GDB
;
; Compile: nasm -f elf64 -g -F dwarf gdb_demo.asm -o gdb_demo.o
; Link:    gcc gdb_demo.o -o gdb_demo -g
; Debug:   gdb ./gdb_demo

section .data
    arr     dd  10, 20, 30, 40, 50
    msg     db  "Hello, GDB!", 0

section .bss
    result  resd    1

section .text
global main

main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    
    ; กำหนดค่า local variable บน stack
    mov     dword [rbp - 4], 0  ; int sum = 0;
    
    ; Loop: sum ของ arr
    xor     ecx, ecx
.loop:
    cmp     ecx, 5
    jge     .done
    
    mov     eax, [arr + rcx*4]
    add     [rbp - 4], eax
    inc     ecx
    jmp     .loop
    
.done:
    mov     eax, [rbp - 4]
    mov     [result], eax
    
    add     rsp, 16
    pop     rbp
    xor     eax, eax
    ret

; ===== GDB Session Example =====
; $ gdb ./gdb_demo
; (gdb) break main
; Breakpoint 1 at 0x401120: file gdb_demo.asm, line 22.
;
; (gdb) run
; Starting program: /home/user/gdb_demo
; Breakpoint 1, main () at gdb_demo.asm:22
;
; (gdb) info registers rsp rbp
; rsp            0x7fffffffe480
; rbp            0x7fffffffe480
;
; (gdb) x/5xw &arr
; 0x404020:  0x0000000a  0x00000014  0x0000001e  0x00000028
; 0x404030:  0x00000032
; ; แปลว่า: 10, 20, 30, 40, 50 (ใน Little-endian)
;
; (gdb) x/s &msg
; 0x404034:  "Hello, GDB!"
;
; (gdb) next                    ; step ไปทีละ instruction
; (gdb) continue                ; รันต่อจนถึง breakpoint ถัดไป
; (gdb) x/xw &result            ; ดูค่า result
; 0x404060:  0x0000006e         ; = 110 = 10+20+30+40+50 ✓
```

---

## ส่วนที่ 12: ARM Assembly - Memory Management

### 12.1 ARM Data Types และ Load/Store Instructions

```asm
@ Filename: arm_data_types.s
@ ARM Assembly สาธิต Data Types
@ 
@ Compile (ARM Linux):  as -o arm_data_types.o arm_data_types.s
@ Link:                 ld arm_data_types.o -o arm_data_types
@ หรือใช้ gcc:          gcc -o arm_data_types arm_data_types.s

.section .data
    byte_val:   .byte   42          @ 8-bit value
    hword_val:  .hword  1000        @ 16-bit halfword
    word_val:   .word   100000      @ 32-bit word
    dword_val:  .quad   1000000     @ 64-bit doubleword (ARM64)
    float_val:  .float  3.14        @ 32-bit float
    double_val: .double 2.718281828 @ 64-bit double
    
    array:      .word   1, 2, 3, 4, 5
    string:     .asciz  "Hello ARM!"

.section .text
.global main

main:
    @ Save registers
    push    {r4-r7, lr}     @ Save callee-saved registers + link register
    
    @ ===== Load/Store Byte =====
    ldr     r0, =byte_val   @ r0 = address ของ byte_val
    ldrb    r1, [r0]        @ r1 = *r0 (โหลด byte, zero-extend)
    ldrbs   r1, [r0]        @ r1 = *r0 (โหลด byte, sign-extend) - ARM32
    
    @ แก้ไขและเก็บกลับ
    add     r1, r1, #1      @ r1++
    strb    r1, [r0]        @ *r0 = r1 (เก็บ byte)
    
    @ ===== Load/Store Halfword =====
    ldr     r0, =hword_val
    ldrh    r1, [r0]        @ โหลด 16-bit, zero-extend
    ldrsh   r1, [r0]        @ โหลด 16-bit, sign-extend
    strh    r1, [r0]        @ เก็บ 16-bit
    
    @ ===== Load/Store Word =====
    ldr     r0, =word_val
    ldr     r1, [r0]        @ โหลด 32-bit word
    str     r1, [r0]        @ เก็บ 32-bit word
    
    @ ===== Load Multiple (LDM) =====
    @ โหลด r4, r5, r6 จาก memory พร้อมกัน (ประหยัด cycles)
    ldr     r0, =array
    ldm     r0, {r4, r5, r6}   @ r4=array[0], r5=array[1], r6=array[2]
    
    @ ===== Addressing Modes ใน ARM =====
    ldr     r0, =array
    
    @ Offset addressing: [Rn, #offset]
    ldr     r1, [r0, #4]    @ โหลด array[1] (4 bytes offset)
    ldr     r1, [r0, #8]    @ โหลด array[2]
    
    @ Register offset: [Rn, Rm]
    mov     r2, #12         @ offset = 12 bytes
    ldr     r1, [r0, r2]    @ โหลด array[3]
    
    @ Scaled register offset: [Rn, Rm, LSL #n]
    mov     r2, #2          @ index = 2
    ldr     r1, [r0, r2, LSL #2]   @ โหลด array[2] (2 << 2 = 8 bytes offset)
    
    @ Pre-indexed: [Rn, #offset]! (update Rn ก่อน access)
    ldr     r1, [r0, #4]!   @ r0 += 4, แล้วโหลดจาก r0 (= array[1])
    
    @ Post-indexed: [Rn], #offset (access ก่อน แล้ว update Rn)
    ldr     r1, [r0], #4    @ โหลดจาก r0 แล้ว r0 += 4
    
    @ Restore registers and return
    pop     {r4-r7, lr}
    mov     r0, #0          @ return 0
    bx      lr              @ return to caller
```

### 12.2 ARM64 (AArch64) Memory Operations

```asm
@ Filename: arm64_memory.s
@ ARM64 Assembly - Memory Operations
@
@ Compile (ARM64 Linux): as -o arm64_memory.o arm64_memory.s
@ Link:                  ld arm64_memory.o -o arm64_memory

.section .data
    value:  .quad   0x0102030405060708  @ 64-bit value
    arr:    .word   100, 200, 300, 400, 500

.section .text
.global _start

_start:
    @ ===== Load/Store ใน ARM64 =====
    
    @ โหลด address ของ value
    adr     x0, value           @ x0 = PC-relative address ของ value
    @ หรือ:
    adrp    x0, value           @ x0 = page address (upper 52 bits)
    add     x0, x0, :lo12:value @ x0 = full address
    
    @ โหลด 64-bit value
    ldr     x1, [x0]            @ x1 = *x0 (64-bit load)
    
    @ โหลดขนาดต่างๆ
    ldrb    w1, [x0]            @ โหลด byte (zero-extend to 32-bit)
    ldrh    w1, [x0]            @ โหลด halfword
    ldr     w1, [x0]            @ โหลด word (32-bit)
    ldr     x1, [x0]            @ โหลด doubleword (64-bit)
    
    @ ===== Signed Load =====
    ldrsb   x1, [x0]            @ โหลด byte, sign-extend to 64-bit
    ldrsh   x1, [x0]            @ โหลด halfword, sign-extend to 64-bit
    ldrsw   x1, [x0]            @ โหลด word, sign-extend to 64-bit
    
    @ ===== Load Pair (LDP) =====
    @ โหลด 2 registers พร้อมกัน
    adr     x0, arr
    ldp     w1, w2, [x0]        @ w1 = arr[0], w2 = arr[1]
    ldp     w3, w4, [x0, #8]    @ w3 = arr[2], w4 = arr[3]
    
    @ ===== Store Pair (STP) =====
    stp     x29, x30, [sp, #-16]!  @ บันทึก frame pointer และ link register
    
    @ ===== Atomic Operations =====
    @ ใช้สำหรับ multi-threaded programming
    adr     x0, value
    mov     x1, #1
    stlr    x1, [x0]            @ Store Release (atomic store)
    ldar    x2, [x0]            @ Load Acquire (atomic load)
    
    @ Exit
    mov     x8, #93             @ syscall: exit
    mov     x0, #0
    svc     #0
```

---

## ส่วนที่ 13: ข้อผิดพลาดที่พบบ่อย (Common Mistakes)

### 13.1 Segmentation Fault จาก Misaligned Access

```nasm
; ผิด: ใช้ movaps กับ data ที่ไม่ aligned
section .data
    db  1               ; byte เดี่ยว ทำให้ต่อไปเป็น odd address
    float_data dd  1.0, 2.0, 3.0, 4.0  ; misaligned!

; ถูก: ใช้ align ก่อน data ที่ต้องการ alignment
section .data
    db  1
    align 16            ; บังคับ 16-byte alignment
    float_data dd  1.0, 2.0, 3.0, 4.0  ; aligned ✓
```

### 13.2 Stack Overflow จาก Infinite Recursion

```nasm
; ผิด: ไม่มี base case ใน recursion
bad_recursive:
    push    rbp
    mov     rbp, rsp
    call    bad_recursive   ; เรียกตัวเองไม่หยุด → Stack Overflow!
    pop     rbp
    ret

; ถูก: มี base case
factorial:
    push    rbp
    mov     rbp, rsp
    
    cmp     edi, 0      ; if (n == 0)
    je      .base_case
    
    ; recursive case
    push    rdi         ; save n
    dec     edi         ; n - 1
    call    factorial   ; factorial(n-1)
    pop     rdi         ; restore n
    imul    eax, edi    ; n * factorial(n-1)
    jmp     .done
    
.base_case:
    mov     eax, 1      ; return 1
    
.done:
    pop     rbp
    ret
```

### 13.3 ลืม sign-extend ก่อน Division

```nasm
; ผิด: ลืม cdq ก่อน idiv
bad_division:
    mov     eax, -100
    ; ลืม cdq !!!
    mov     ecx, 3
    idiv    ecx     ; edx ไม่ได้ถูก set → undefined behavior หรือ exception!

; ถูก: ต้อง sign-extend ก่อนเสมอ
good_division:
    mov     eax, -100
    cdq             ; sign-extend eax → edx:eax
    mov     ecx, 3
    idiv    ecx     ; ถูกต้อง: eax = -33, edx = -1
```

### 13.4 Off-by-One ใน Array Access

```nasm
; array dd 1, 2, 3, 4, 5  (5 elements, index 0-4)

; ผิด: เข้าถึง element ที่ 5 (index 5) ซึ่งไม่มีอยู่!
bad_array_access:
    lea     rbx, [array]
    mov     ecx, 5          ; index = 5 (out of bounds!)
    mov     eax, [rbx + rcx*4]  ; อ่านค่าที่ไม่ถูกต้อง

; ถูก: ตรวจสอบ bounds ก่อนเสมอ
safe_array_access:
    lea     rbx, [array]
    mov     ecx, 5          ; index
    cmp     ecx, 5          ; compare with array size
    jge     .out_of_bounds  ; ถ้า >= 5 ให้ข้าม
    mov     eax, [rbx + rcx*4]  ; ปลอดภัยแล้ว
    jmp     .done
.out_of_bounds:
    ; handle error
.done:
```

### 13.5 หลง Endianness เวลาเปรียบเทียบ

```nasm
; ข้อมูล network packet (big-endian) ที่รับมา:
; 0x00 0x50 = port 80 ใน big-endian

; ผิด: เปรียบเทียบโดยตรงบน little-endian machine
; network_port dw 0x0050    ; ใน memory: 50 00 (little-endian!)
; cmp  word [network_port], 0x0050  ; เปรียบเทียบกับ little-endian 0x0050!
; ผลที่ได้จะผิด!

; ถูก: แปลง byte order ก่อนเปรียบเทียบ
correct_port_check:
    movzx   eax, word [network_port]    ; โหลด 0x5000 (little-endian)
    xchg    al, ah                       ; swap bytes: 0x0050
    cmp     eax, 80                      ; ถูกต้องแล้ว
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Array Statistics
เขียนโปรแกรมที่รับ array ของ integers แล้วหา min, max, และ average

```nasm
; Filename: exercise1_stats.asm
; TODO: ทดลองเขียนและรันโปรแกรมนี้
;
; Compile: nasm -f elf64 exercise1_stats.asm -o exercise1_stats.o
; Link:    gcc exercise1_stats.o -o exercise1_stats

section .data
    arr         dd  45, 12, 78, 23, 56, 9, 34, 67, 89, 11
    arr_len     equ ($ - arr) / 4      ; คำนวณ length อัตโนมัติ
    
    fmt_min     db  "Min: %d", 10, 0
    fmt_max     db  "Max: %d", 10, 0
    fmt_avg     db  "Average: %d", 10, 0

section .text
global main
extern printf

; TODO: เขียน find_min function
; input: rdi = pointer to array, rsi = count
; output: eax = minimum value

; TODO: เขียน find_max function
; input: rdi = pointer to array, rsi = count
; output: eax = maximum value

; TODO: เขียน calc_average function
; input: rdi = pointer to array, rsi = count
; output: eax = average (integer division)

main:
    push    rbp
    mov     rbp, rsp
    
    ; TODO: เรียก find_min และพิมพ์ผล
    ; TODO: เรียก find_max และพิมพ์ผล
    ; TODO: เรียก calc_average และพิมพ์ผล
    
    pop     rbp
    xor     eax, eax
    ret

; Expected output:
; Min: 9
; Max: 89
; Average: 42
```

### Exercise 2: String Reverse

```nasm
; Filename: exercise2_reverse.asm
; เขียนฟังก์ชัน reverse_string ที่กลับ string in-place
;
; Hint: ใช้ two-pointer approach:
;   - pointer left ชี้ที่ตัวแรก
;   - pointer right ชี้ที่ตัวสุดท้าย (ก่อน null)
;   - swap แล้ว left++ right--
;   - หยุดเมื่อ left >= right

section .data
    my_str  db  "Hello, World!", 0
    fmt_str db  "%s", 10, 0

section .text
global main
extern printf, strlen

; TODO: เขียน reverse_string
; input: rdi = pointer to null-terminated string

main:
    push    rbp
    mov     rbp, rsp
    
    ; พิมพ์ก่อน reverse
    lea     rdi, [my_str]
    lea     rdi, [fmt_str]
    ; TODO: printf original string
    
    ; Reverse
    lea     rdi, [my_str]
    ; TODO: call reverse_string
    
    ; พิมพ์หลัง reverse
    ; TODO: printf reversed string
    
    pop     rbp
    xor     eax, eax
    ret

; Expected output:
; Hello, World!
; !dlroW ,olleH
```

### Exercise 3: Struct-based Student Database

```nasm
; Filename: exercise3_students.asm
; สร้างระบบจัดการข้อมูลนักเรียนด้วย struct

; struct Student {
;     char name[32];
;     int  student_id;
;     float gpa;
;     int   age;
; };

struc Student
    .name       resb    32
    .student_id resd    1
    .gpa        resd    1   ; float (ใช้ resd)
    .age        resd    1
endstruc

section .data
    ; TODO: ประกาศ array ของ Student 3 คน
    ; TODO: สร้าง format strings สำหรับแสดงผล

section .bss
    students    resb    Student_size * 10   ; array for 10 students
    count       resd    1                   ; current count

section .text
global main
extern printf, scanf

; TODO: เขียนฟังก์ชัน add_student
; TODO: เขียนฟังก์ชัน print_student
; TODO: เขียนฟังก์ชัน find_highest_gpa

main:
    ; TODO: เพิ่ม 3 นักเรียนและแสดงผล
```

### Exercise 4: Memory Copy (memcpy)

```nasm
; Filename: exercise4_memcpy.asm
; เขียน memcpy ของตัวเอง

section .data
    source  db  "Assembly is awesome!", 0
    src_len equ $ - source
    fmt     db  "Copied: %s", 10, 0

section .bss
    dest    resb    100

section .text
global main
extern printf

; เขียน my_memcpy:
; input: rdi = destination, rsi = source, rdx = count
; คัดลอก rdx bytes จาก source ไป destination
my_memcpy:
    push    rbp
    mov     rbp, rsp
    
    ; Method 1: REP MOVSB (ง่ายที่สุด)
    mov     rcx, rdx        ; count
    ; rep movsb copies [rsi] to [rdi], rdx times, incrementing both
    cld                     ; clear direction flag (increment)
    rep     movsb           ; copy rcx bytes จาก rsi ไป rdi
    
    pop     rbp
    ret

main:
    push    rbp
    mov     rbp, rsp
    
    ; คัดลอก source ไป dest
    lea     rdi, [dest]
    lea     rsi, [source]
    mov     rdx, src_len
    call    my_memcpy
    
    ; พิมพ์ผล
    lea     rdi, [fmt]
    lea     rsi, [dest]
    xor     eax, eax
    call    printf
    
    pop     rbp
    xor     eax, eax
    ret

; Expected output:
; Copied: Assembly is awesome!
```

### Exercise 5: Pointer Chasing Linked List

```nasm
; Filename: exercise5_linked_list.asm
; สร้าง Linked List ด้วย manual memory management

; struct Node {
;     int  value;
;     Node* next;   (pointer ขนาด 8 bytes บน 64-bit)
; };

struc Node
    .value  resd    1       ; int value (4 bytes)
    .pad    resd    1       ; padding (4 bytes) เพื่อ align next pointer
    .next   resq    1       ; Node* next (8 bytes)
endstruc
; Node_size = 16 bytes

section .data
    ; สร้าง linked list แบบ static
    node1   istruc Node
                at Node.value, dd   10
                at Node.pad,   dd   0
                at Node.next,  dq   node2
            iend
    
    node2   istruc Node
                at Node.value, dd   20
                at Node.pad,   dd   0
                at Node.next,  dq   node3
            iend
    
    node3   istruc Node
                at Node.value, dd   30
                at Node.pad,   dd   0
                at Node.next,  dq   0       ; NULL - ท้าย list
            iend
    
    fmt_node    db  "Node value: %d", 10, 0

section .text
global main
extern printf

; TODO: เขียนฟังก์ชัน traverse_list
; input: rdi = pointer to head node
; พิมพ์ทุก node จนถึง NULL

; TODO: เขียนฟังก์ชัน sum_list
; input: rdi = pointer to head node
; output: rax = sum of all values

main:
    push    rbp
    mov     rbp, rsp
    
    ; Traverse list
    lea     rdi, [node1]
    ; TODO: call traverse_list
    
    ; Sum list
    lea     rdi, [node1]
    ; TODO: call sum_list
    ; ผลลัพธ์ควรเป็น 60 (10+20+30)
    
    pop     rbp
    xor     eax, eax
    ret
```

---

## สรุปและ Key Takeaways

### สิ่งที่ควรจำจาก Part นี้:

1. **ขนาดข้อมูล**: Byte=8bit, Word=16bit, DWORD=32bit, QWORD=64bit, TBYTE=80bit (FPU), OWORD=128bit (SSE), YWORD=256bit (AVX)

2. **Two's Complement**: วิธีแทนค่าลบที่ CPU ใช้ - กลับทุก bit แล้วบวก 1

3. **Memory Segments**: Text (code), Data (initialized globals), BSS (uninitialized globals), Heap (dynamic), Stack (local vars, return addresses)

4. **Stack เติบโตลงล่าง**: Push ทำให้ RSP ลดลง, Pop ทำให้ RSP เพิ่มขึ้น

5. **Data Alignment**: ข้อมูล N bytes ควร align ที่ N bytes เพื่อ performance ที่ดีที่สุด

6. **Little-Endian**: x86 เก็บ LSB ที่ address ต่ำกว่าเสมอ

7. **Addressing Modes**: immediate, register, direct, register indirect, base+displacement, base+index*scale+displacement

8. **Pointer = Register**: ใน Assembly register ที่ชี้ไปยัง memory ก็คือ pointer นั่นเอง

9. **struc ใน NASM**: ช่วยจัดการ struct offsets อัตโนมัติ ไม่ต้องคำนวณ offset เอง

10. **GDB**: ใช้ `x/<n><format><size> <addr>` เพื่อดู memory, `info registers` เพื่อดู registers

### Checklist ก่อนไป Part ถัดไป:

- [ ] รัน data_types.asm สำเร็จและเข้าใจ output
- [ ] อธิบาย Two's Complement ของ -5 ได้
- [ ] วาด memory layout ของโปรแกรมตัวเองได้
- [ ] ใช้ GDB ตรวจ memory ได้
- [ ] เขียน array traversal ใน Assembly ได้
- [ ] สร้างและเข้าถึง struct ใน Assembly ได้
- [ ] ทำ Exercise 1-5 อย่างน้อย 3 ข้อ

---

## แหล่งข้อมูลเพิ่มเติม (Resources)

### หนังสือ:
- **"Computer Organization and Architecture" - William Stallings**: อธิบาย memory hierarchy อย่างละเอียด
- **"Programming from the Ground Up" - Jonathan Bartlett**: ฟรี PDF, อธิบาย memory layout ดีมาก
- **"x86-64 Assembly Language Programming with Ubuntu" - Ed Jorgensen**: ฟรี PDF, แนะนำมาก

### Online Resources:
- **Intel® 64 and IA-32 Architectures Software Developer's Manual**: https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
- **System V AMD64 ABI**: https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf (สำหรับ calling convention และ memory layout)
- **NASM Documentation**: https://nasm.us/doc/

### Tools:
```bash
# ตรวจสอบ sections ของ binary
readelf -S ./program

# ดู symbols
nm ./program

# Disassemble
objdump -d ./program

# ดู memory map ของ process ที่กำลังรัน
cat /proc/<pid>/maps

# Valgrind: ตรวจหา memory errors
valgrind --leak-check=full ./program
```

### Commands สำหรับ Compile และ Test:

```bash
# NASM x86-64
nasm -f elf64 program.asm -o program.o
gcc program.o -o program
./program

# NASM พร้อม debug symbols
nasm -f elf64 -g -F dwarf program.asm -o program.o
gcc program.o -o program -g
gdb ./program

# ARM Assembly (ถ้ามี ARM machine หรือ emulator)
as -o program.o program.s
ld program.o -o program

# Cross-compile ARM บน x86 Linux
sudo apt install gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi
arm-linux-gnueabi-as -o program.o program.s
arm-linux-gnueabi-ld program.o -o program
qemu-arm ./program

# ARM64
aarch64-linux-gnu-as -o program.o program.s
aarch64-linux-gnu-ld program.o -o program
qemu-aarch64 ./program
```

---

**Part ถัดไป:** Part 005 - การควบคุมโปรแกรม (Control Flow: Conditions, Loops, Jumps)

---
*Assembly Programming Course - Part 004*  
*ผู้สอน: Assembly Course Series*  
*วันที่อัปเดต: 2026*
