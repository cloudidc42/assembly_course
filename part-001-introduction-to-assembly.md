# Assembly Language Programming Course
# Part 001: บทนำสู่ Assembly Language

**เวลาที่ใช้เรียน:** 3-4 ชั่วโมง  
**ระดับ:** Beginner (เริ่มต้น)  
**Prerequisites:** ความรู้พื้นฐานเรื่องคอมพิวเตอร์ทั่วไป

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากเรียนจบ Part นี้ นักเรียนจะสามารถ:

- อธิบายได้ว่า Assembly Language คืออะไร และแตกต่างจากภาษาโปรแกรมอื่นอย่างไร
- เข้าใจประวัติศาสตร์และวิวัฒนาการของ Assembly Language
- อธิบายความแตกต่างระหว่าง Machine Code, Assembly, และ High-level Language
- รู้จักสถาปัตยกรรม x86 และ ARM ในระดับเบื้องต้น
- เข้าใจว่า Assembly ถูกใช้งานอย่างไรในโลกจริง
- อ่าน Assembly output จาก compiler ได้เบื้องต้น
- เตรียมความพร้อมสำหรับการเรียนในส่วนต่อๆ ไป

---

## สารบัญ (Table of Contents)

1. [ทำไมต้องเรียน Assembly ในยุค 2024?](#why-assembly)
2. [ประวัติศาสตร์ Assembly Language](#history)
3. [ความแตกต่างระหว่าง High-level vs Low-level vs Machine Code](#levels)
4. [การใช้งาน Assembly ในโลกจริง](#real-world)
5. [Overview ของ x86 Architecture](#x86-overview)
6. [Overview ของ ARM Architecture](#arm-overview)
7. [Code Examples: Machine Code vs Assembly vs C](#code-examples)
8. [การดู Assembly ใน Compiler Output](#compiler-output)
9. [ข้อผิดพลาดที่พบบ่อย](#common-mistakes)
10. [แบบฝึกหัดท้ายบท](#exercises)
11. [สรุป](#summary)
12. [แหล่งอ้างอิง](#references)

---

## 1. ทำไมต้องเรียน Assembly ในยุค 2024? {#why-assembly}

ในยุคที่มีภาษาโปรแกรมระดับสูงอย่าง Python, JavaScript, Rust, Go ให้ใช้งาน นักเรียนหลายคนอาจตั้งคำถามว่า "ทำไมต้องเรียน Assembly ด้วย?"

### 1.1 เข้าใจ Hardware อย่างลึกซึ้ง

Assembly Language คือภาษาที่ใกล้เคียง Hardware มากที่สุดที่มนุษย์สามารถอ่านได้อย่างสะดวก เมื่อเข้าใจ Assembly คุณจะเข้าใจว่า:

- CPU ทำงานอย่างไรในระดับ Instruction
- Memory ถูก Access อย่างไร
- Register คืออะไรและทำงานอย่างไร
- Stack และ Heap แตกต่างกันอย่างไรในระดับ Machine

### 1.2 Performance Optimization

ในงานที่ต้องการ Performance สูงสุด เช่น:
- Video/Audio Codec (x264, x265, FLAC encoders)
- Cryptography (AES-NI instructions)
- Signal Processing (FFT algorithms)
- Game Engine (Physics calculations)

Assembly หรือ Compiler Intrinsics ที่เขียนจาก Assembly Knowledge ยังคงจำเป็นอยู่

### 1.3 Security และ Reverse Engineering

**Malware Analysis:** นักวิเคราะห์ความปลอดภัยต้องอ่าน Assembly เพื่อเข้าใจว่า Malware ทำอะไร

**CTF (Capture The Flag):** การแข่งขัน Security ส่วนใหญ่ต้องใช้ความรู้ Assembly

**Exploit Development:** การค้นหาและใช้ประโยชน์จาก Vulnerability ต้องการความเข้าใจ Assembly อย่างลึกซึ้ง

**Example - Buffer Overflow:**
```
; ตัวอย่าง Stack Layout ที่ Exploit Developer ต้องเข้าใจ
;
; High Address
; +------------------+
; |   Return Address  |  <-- ถ้าเขียนทับตรงนี้ได้ = Code Execution
; +------------------+
; |   Saved RBP      |
; +------------------+
; |   Local Var 2    |
; +------------------+
; |   Local Var 1    |
; +------------------+
; |   Buffer[64]     |  <-- ถ้า Input ยาวกว่า 64 bytes = Overflow!
; +------------------+
; Low Address
```

### 1.4 Operating System Development

OS Kernel ต้องเขียน Assembly สำหรับ:
- Boot Loader (Bootloader ต้องเขียนด้วย Assembly)
- Interrupt Handlers
- Context Switching (สลับระหว่าง Process)
- Memory Management Unit (MMU) setup
- System Call Interface

### 1.5 Embedded Systems

ใน Microcontroller ที่มี Resource จำกัด เช่น:
- 1KB RAM, 8KB Flash
- No Operating System
- Real-time requirements

Assembly บางครั้งเป็นทางเลือกเดียวที่ทำงานได้

### 1.6 เข้าใจ Compiler ดีขึ้น

เมื่อรู้ Assembly คุณจะเข้าใจว่า:
- `-O2` vs `-O3` optimization ทำอะไรกับ code จริงๆ
- `inline` function ทำงานอย่างไร
- `const` หรือ `volatile` มีผลอย่างไรต่อ Machine Code

---

## 2. ประวัติศาสตร์ Assembly Language {#history}

### 2.1 ยุคก่อน Assembly (1940s)

**1945 - ENIAC (Electronic Numerical Integrator and Computer)**
- คอมพิวเตอร์เครื่องแรกที่ใช้งานได้จริงในเชิงพาณิชย์
- โปรแกรมโดยการต่อสายไฟ (Physical Wiring)
- ไม่มี "Software" ในความหมายสมัยใหม่

**1948 - Manchester Baby (Small-Scale Experimental Machine)**
- คอมพิวเตอร์เครื่องแรกที่ใช้ Stored Program (เก็บโปรแกรมใน Memory)
- โปรแกรมเป็น Binary (0 และ 1) โดยตรง
- โปรแกรมเมอร์ต้องจำ/ดู Opcode Table แปลงด้วยมือ

**ตัวอย่าง Binary Programming (เพื่อความเข้าใจ):**
```
; โปรแกรม "Add 5 + 3" ใน Binary ยุค 1940s (สมมติ)
; โปรแกรมเมอร์ต้องเขียนแบบนี้:
01001000 00000101    ; ความหมาย: โหลดค่า 5 เข้า Register
00000011 00000011    ; ความหมาย: บวก 3 เข้าไป  
11110000 00000000    ; ความหมาย: เก็บผลลัพธ์
; ยากมากที่จะเขียนและ Debug!
```

### 2.2 กำเนิด Assembly Language (1950s)

**1949 - Kathleen Booth เขียน Assembly Language แรก**
- สำหรับ ARC (Automatic Relay Calculator) และ APE(X)C
- เปลี่ยน Opcode Binary เป็น Mnemonic (คำย่อ) ที่มนุษย์อ่านได้

**1951 - Grace Hopper และ A-0 Assembler**
- Grace Hopper เป็นผู้บุกเบิก Assembler โปรแกรมที่แปลง Assembly เป็น Machine Code
- ต่อมาพัฒนาเป็น COBOL

**ตัวอย่าง Assembly ยุคแรก vs Binary:**
```
; Binary (ยุค 1940s) - อ่านยาก ผิดง่าย
01000001 00000101
00000011 00000011

; Assembly (ยุค 1950s) - อ่านง่ายขึ้น
MOV AX, 5    ; โหลด 5 เข้า Register AX
ADD AX, 3    ; บวก 3 เข้า AX
; ผลลัพธ์: AX = 8
```

### 2.3 ยุค Mainframe (1960s-1970s)

**IBM System/360 (1964)**
- เป็น Architecture ที่มีอิทธิพลมากที่สุดในประวัติศาสตร์
- แนะนำ Concept ของ "Family of Computers" ที่ Compatible กัน
- Assembly สำหรับ System/360 ยังถูกใช้ใน Mainframe สมัยใหม่ (z/Architecture)

```asm
; IBM System/360 Assembly - ตัวอย่างการบวกเลข
* Add two numbers in System/360 Assembly
         L     2,NUM1         Load NUM1 into Register 2
         A     2,NUM2         Add NUM2 to Register 2
         ST    2,RESULT       Store result
NUM1     DC    F'10'          Define constant 10
NUM2     DC    F'20'          Define constant 20
RESULT   DS    F              Reserve space for result
```

### 2.4 กำเนิด x86 (1978-1980s)

**Intel 8086 (1978)**
- 16-bit Processor ที่กลายเป็นรากฐานของ x86 Architecture
- IBM ใช้ 8088 (version ที่ถูกกว่า) ใน IBM PC รุ่นแรก (1981)

**Intel 80386 (1985)**
- 32-bit Extension ของ x86
- เรียกว่า IA-32 (Intel Architecture 32-bit)
- Protected Mode, Virtual Memory

**Intel Pentium (1993)**
- Superscalar: Execute หลาย Instruction พร้อมกัน
- เริ่มใช้ SIMD Instructions (MMX ในรุ่นต่อมา)

**AMD Athlon 64 / Intel EM64T (2003-2004)**
- 64-bit Extension ของ x86
- เรียกว่า x86-64 หรือ AMD64
- ขยาย Register จาก 32-bit เป็น 64-bit
- เพิ่ม General Purpose Register จาก 8 เป็น 16 ตัว

### 2.5 กำเนิด ARM Architecture (1983-ปัจจุบัน)

**1983 - Acorn Computers เริ่มพัฒนา ARM**
- ARM = Acorn RISC Machine (ต่อมาเปลี่ยนเป็น Advanced RISC Machine)
- RISC = Reduced Instruction Set Computer
- ออกแบบมาเพื่อ Simple, Low-power

**ARM1 (1985)**
- ใช้ใน Acorn Archimedes Computer
- เป็น 32-bit RISC Processor

**ARM7TDMI (1993)**
- ใช้ใน Nintendo Game Boy Advance
- เป็น Foundation ของ ARM รุ่นสมัยใหม่หลายตัว

**ARMv8/AArch64 (2011)**
- 64-bit ARM Architecture
- ใช้ใน iPhone 5S (2013) เป็นครั้งแรกในโทรศัพท์มือถือ
- ปัจจุบันใช้ใน Apple M1/M2/M3, Raspberry Pi 4/5, Android phones

**ตารางเปรียบเทียบ Architecture:**

| Feature | x86-64 | ARM64 (AArch64) |
|---------|--------|-----------------|
| ประเภท | CISC | RISC |
| Instruction Length | Variable (1-15 bytes) | Fixed (4 bytes) |
| Registers (General) | 16 (RAX, RBX, ...) | 31 (X0-X30) |
| ใช้ใน | Desktop, Server | Mobile, Embedded, Apple Silicon |
| Power Consumption | สูงกว่า | ต่ำกว่า |
| Performance/Watt | ต่ำกว่า | สูงกว่า |
| Raw Performance | สูงมาก | สูงมาก (M-series) |

---

## 3. ความแตกต่างระหว่าง High-level vs Low-level vs Machine Code {#levels}

### 3.1 Abstraction Layers ของ Software

```
┌─────────────────────────────────────────┐
│          Application Layer              │
│   Python, JavaScript, Java, C#         │  <-- High-Level Languages
│   ง่ายต่อการเขียน, Portable            │
├─────────────────────────────────────────┤
│          System Layer                   │
│   C, C++, Rust                         │  <-- Mid-Level Languages
│   ควบคุม Memory ได้, ยังอ่านได้        │
├─────────────────────────────────────────┤
│          Assembly Layer                 │
│   x86 Assembly, ARM Assembly           │  <-- Low-Level Language
│   ใกล้ Hardware, ยังอ่านได้โดยมนุษย์  │
├─────────────────────────────────────────┤
│          Machine Code Layer             │
│   0101001010001011...                  │  <-- Machine Code
│   CPU เข้าใจโดยตรง, มนุษย์อ่านยาก   │
├─────────────────────────────────────────┤
│          Hardware Layer                 │
│   Transistors, Logic Gates             │  <-- Physical Hardware
└─────────────────────────────────────────┘
```

### 3.2 ตัวอย่างเปรียบเทียบ: คำนวณ (a + b) * 2

**Python (High-level):**
```python
# Python - อ่านง่าย เขียนง่าย แต่ไม่รู้ว่า Hardware ทำอะไร
a = 5
b = 3
result = (a + b) * 2
print(result)  # Output: 16
```

**C (Mid-level):**
```c
// C - ยังอ่านได้ แต่ใกล้ Hardware มากขึ้น
#include <stdio.h>

int main() {
    int a = 5;
    int b = 3;
    int result = (a + b) * 2;
    printf("%d\n", result);  // Output: 16
    return 0;
}
```

**x86-64 Assembly (Low-level):**
```nasm
; x86-64 NASM Assembly - เห็นทุก Instruction ที่ CPU จะ Execute
; ทุก operation ต้องระบุชัดเจน

section .data
    fmt     db "%d", 10, 0    ; Format string "%d\n\0"

section .text
    global main
    extern printf

main:
    ; Stack setup
    push    rbp                ; บันทึก Base Pointer เก่า
    mov     rbp, rsp           ; ตั้ง Base Pointer ใหม่
    sub     rsp, 16            ; จองพื้นที่ใน Stack 16 bytes
    
    ; int a = 5
    mov     dword [rbp-4], 5   ; เก็บค่า 5 ที่ตำแหน่ง rbp-4 (a)
    
    ; int b = 3
    mov     dword [rbp-8], 3   ; เก็บค่า 3 ที่ตำแหน่ง rbp-8 (b)
    
    ; result = (a + b) * 2
    mov     eax, [rbp-4]       ; โหลด a เข้า EAX (EAX = 5)
    add     eax, [rbp-8]       ; บวก b เข้า EAX (EAX = 5+3 = 8)
    imul    eax, 2             ; คูณ EAX ด้วย 2 (EAX = 8*2 = 16)
    mov     [rbp-12], eax      ; เก็บ result = 16
    
    ; printf("%d\n", result)
    mov     esi, [rbp-12]      ; Parameter 2: result (value = 16)
    lea     rdi, [rel fmt]     ; Parameter 1: format string
    xor     eax, eax           ; Clear EAX (สำหรับ printf)
    call    printf             ; เรียก printf
    
    ; return 0
    xor     eax, eax           ; EAX = 0 (return value)
    leave                      ; Restore Stack Frame
    ret                        ; Return จาก main

; Output: 16
```

**Machine Code (Binary/Hex) - สิ่งที่ CPU เข้าใจจริงๆ:**
```
; Machine Code ของ "mov eax, 5" (ตัวอย่าง)
; Hex: B8 05 00 00 00
; Binary: 10111000 00000101 00000000 00000000 00000000
;
; B8 = Opcode สำหรับ MOV EAX, imm32
; 05 00 00 00 = ค่า 5 ใน Little-Endian format
;
; มนุษย์ต้องจำ Opcode Table ทั้งหมดถึงจะ "อ่าน" ได้
; นั่นคือเหตุผลที่ Assembly ถูกสร้างขึ้น!
```

### 3.3 ขั้นตอนการแปลง Source Code → Machine Code

```
Python Code
    │
    ▼  (Interpreter/Bytecode Compiler)
Python Bytecode (.pyc)
    │
    ▼  (Python Virtual Machine)
Machine Instructions
    │
    ▼
CPU Execute

─────────────────────────────────

C Code (.c)
    │
    ▼  (Preprocessor: gcc -E)
Preprocessed Code (.i)
    │
    ▼  (Compiler: gcc -S)
Assembly Code (.s / .asm)
    │
    ▼  (Assembler: nasm, as)
Object Code (.o)
    │
    ▼  (Linker: ld)
Executable (a.out, .exe)
    │
    ▼
CPU Execute
```

---

## 4. การใช้งาน Assembly ในโลกจริง {#real-world}

### 4.1 Operating System Kernels

**Linux Kernel Assembly:**
```asm
; จาก Linux Kernel - arch/x86/entry/entry_64.S
; System Call Entry Point (ย่อเพื่อการศึกษา)

; เมื่อ User Program เรียก System Call (เช่น read(), write())
; CPU จะ Jump มาที่นี่

ENTRY(entry_SYSCALL_64)
    swapgs                      ; สลับ GS register (User <-> Kernel)
    movq    %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
    movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp
    
    ; บันทึก Register ของ User Program
    pushq   $__USER_DS          ; User SS
    pushq   PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  ; User RSP
    pushq   %r11                ; User RFLAGS
    pushq   $__USER_CS          ; User CS
    pushq   %rcx                ; User RIP
    
    ; ... Kernel จัดการ System Call ...
END(entry_SYSCALL_64)
```

**Windows NT Kernel Assembly (x86-64):**
```asm
; Context Switch - การสลับระหว่าง Threads
; (ตัวอย่างแบบง่ายเพื่อการศึกษา)

SwapContext:
    ; บันทึก Register ของ Thread เก่า
    mov     [rdi + CONTEXT_RAX], rax    ; บันทึก RAX
    mov     [rdi + CONTEXT_RBX], rbx    ; บันทึก RBX
    mov     [rdi + CONTEXT_RCX], rcx    ; บันทึก RCX
    mov     [rdi + CONTEXT_RDX], rdx    ; บันทึก RDX
    ; ... บันทึก Register อื่นๆ ...
    
    ; โหลด Register ของ Thread ใหม่
    mov     rax, [rsi + CONTEXT_RAX]    ; โหลด RAX
    mov     rbx, [rsi + CONTEXT_RBX]    ; โหลด RBX
    mov     rcx, [rsi + CONTEXT_RCX]    ; โหลด RCX
    mov     rdx, [rsi + CONTEXT_RDX]    ; โหลด RDX
    ; ... โหลด Register อื่นๆ ...
    ret
```

### 4.2 Cryptography - AES-NI Instructions

Intel/AMD CPU มี Hardware Instructions สำหรับ AES encryption โดยเฉพาะ:

```nasm
; AES-128 Encryption ใช้ AES-NI Instructions
; เร็วกว่า Software AES ~10x
section .text
global aes128_encrypt_block

aes128_encrypt_block:
    ; Input:  RDI = pointer to plaintext (16 bytes)
    ;         RSI = pointer to key schedule (176 bytes)
    ; Output: RAX = pointer to ciphertext (in-place)
    
    movdqu  xmm0, [rdi]        ; โหลด Plaintext 128-bit เข้า XMM0
    movdqu  xmm1, [rsi]        ; โหลด Round Key 0
    pxor    xmm0, xmm1         ; XOR กับ Round Key 0 (AddRoundKey)
    
    movdqu  xmm1, [rsi + 16]   ; โหลด Round Key 1
    aesenc  xmm0, xmm1         ; AES Round 1 (SubBytes+ShiftRows+MixColumns+AddRoundKey)
    
    movdqu  xmm1, [rsi + 32]   ; โหลด Round Key 2
    aesenc  xmm0, xmm1         ; AES Round 2
    
    ; ... Round 3-9 ...
    
    movdqu  xmm1, [rsi + 160]  ; โหลด Round Key 10 (Final)
    aesenclast xmm0, xmm1      ; AES Final Round (ไม่มี MixColumns)
    
    movdqu  [rdi], xmm0        ; เก็บ Ciphertext กลับ
    mov     rax, rdi           ; Return pointer
    ret
```

### 4.3 SIMD Optimization - การประมวลผลข้อมูลจำนวนมาก

```nasm
; การบวก Array ของ Integer 8 ตัวพร้อมกันด้วย AVX2
; แทนที่จะทำทีละ 1 ตัว ทำ 8 ตัวพร้อมกัน!

section .text
global add_arrays_avx2

; add_arrays_avx2(int* dst, int* src1, int* src2, int count)
; RDI = dst, RSI = src1, RDX = src2, RCX = count
add_arrays_avx2:
    push    rbp
    mov     rbp, rsp
    
    xor     eax, eax            ; index i = 0
    
.loop:
    cmp     eax, ecx            ; เปรียบเทียบ i กับ count
    jge     .done               ; ถ้า i >= count ออกจาก loop
    
    ; โหลด 8 integers (32-bit each) จาก src1 และ src2 พร้อมกัน
    vmovdqu ymm0, [rsi + rax*4]   ; โหลด src1[i..i+7] เข้า YMM0 (256-bit = 8x32-bit)
    vmovdqu ymm1, [rdx + rax*4]   ; โหลด src2[i..i+7] เข้า YMM1
    
    ; บวก 8 integers พร้อมกัน (SIMD = Single Instruction Multiple Data)
    vpaddd  ymm0, ymm0, ymm1      ; dst[0..7] = src1[0..7] + src2[0..7] (8 additions in 1 instruction!)
    
    ; เก็บผลลัพธ์
    vmovdqu [rdi + rax*4], ymm0   ; เก็บผลลัพธ์ 8 integers
    
    add     eax, 8              ; i += 8 (ข้ามไป 8 elements)
    jmp     .loop
    
.done:
    vzeroupper                  ; Clean up AVX state (สำคัญ!)
    pop     rbp
    ret
```

### 4.4 Embedded Systems - STM32 ARM Cortex-M

```asm
@ ARM Cortex-M4 Assembly (GAS Syntax)
@ LED Blink บน STM32F4 (ไม่ใช้ HAL Library)
@ ควบคุม Hardware โดยตรงผ่าน Memory-Mapped Registers

.equ    RCC_BASE,       0x40023800   @ Reset and Clock Control
.equ    RCC_AHB1ENR,    0x30         @ AHB1 Clock Enable Register offset
.equ    GPIOD_BASE,     0x40020C00   @ GPIOD Base Address
.equ    GPIOD_MODER,    0x00         @ GPIO Mode Register offset
.equ    GPIOD_ODR,      0x14         @ GPIO Output Data Register offset
.equ    LED_PIN,        12           @ LED อยู่ที่ Pin 12 (PD12)

.section .text
.global main
.thumb_func

main:
    @ เปิด Clock สำหรับ GPIOD
    ldr     r0, =RCC_BASE           @ โหลด address ของ RCC Base
    ldr     r1, [r0, #RCC_AHB1ENR] @ อ่านค่า AHB1ENR
    orr     r1, r1, #(1 << 3)      @ Set bit 3 (GPIOD Clock Enable)
    str     r1, [r0, #RCC_AHB1ENR] @ เขียนค่ากลับ

    @ ตั้ง GPIOD Pin 12 เป็น Output
    ldr     r0, =GPIOD_BASE
    ldr     r1, [r0, #GPIOD_MODER] @ อ่าน Mode Register
    bic     r1, r1, #(3 << 24)     @ Clear bits 25:24 (Pin 12 mode)
    orr     r1, r1, #(1 << 24)     @ Set bit 24 = Output Mode
    str     r1, [r0, #GPIOD_MODER] @ เขียนค่ากลับ

loop:
    @ เปิด LED (Set Pin 12)
    ldr     r0, =GPIOD_BASE
    ldr     r1, [r0, #GPIOD_ODR]   @ อ่าน Output Data Register
    orr     r1, r1, #(1 << LED_PIN) @ Set bit 12
    str     r1, [r0, #GPIOD_ODR]   @ เขียน: LED ON

    @ Delay
    ldr     r2, =500000            @ จำนวนรอบ Delay
delay1:
    subs    r2, r2, #1             @ r2 = r2 - 1 และ Set Flags
    bne     delay1                 @ ถ้า r2 != 0 วนซ้ำ

    @ ปิด LED (Clear Pin 12)
    ldr     r1, [r0, #GPIOD_ODR]
    bic     r1, r1, #(1 << LED_PIN) @ Clear bit 12
    str     r1, [r0, #GPIOD_ODR]   @ เขียน: LED OFF

    @ Delay
    ldr     r2, =500000
delay2:
    subs    r2, r2, #1
    bne     delay2

    b       loop                   @ วนซ้ำ ทำให้ LED กระพริบตลอดไป
```

### 4.5 Game Development - Fast Math

```nasm
; Fast Inverse Square Root (ใช้ใน Quake III Arena เวอร์ชัน x86)
; คำนวณ 1/sqrt(x) เร็วกว่า Library เดิมมาก

section .text
global fast_inv_sqrt

; float fast_inv_sqrt(float x)
; xmm0 = x (float)
fast_inv_sqrt:
    ; ขั้นตอนที่ 1: Bit-level hack
    movd    eax, xmm0          ; แปลง float เป็น int bits
    mov     ecx, 0x5f3759df    ; Magic number ที่โด่งดัง!
    sar     eax, 1             ; Shift right 1 (คล้ายกับหาร 2 ใน log domain)
    sub     ecx, eax           ; Magic number - (x >> 1)
    movd    xmm1, ecx          ; แปลงกลับเป็น float
    
    ; ขั้นตอนที่ 2: Newton-Raphson Iteration เพื่อเพิ่มความแม่นยำ
    ; y = y * (1.5 - 0.5 * x * y * y)
    movss   xmm2, [rel half]   ; xmm2 = 0.5
    movss   xmm3, [rel one_5]  ; xmm3 = 1.5
    
    mulss   xmm2, xmm0         ; xmm2 = 0.5 * x
    movaps  xmm0, xmm1         ; xmm0 = y (initial estimate)
    mulss   xmm2, xmm1         ; xmm2 = 0.5 * x * y
    mulss   xmm2, xmm1         ; xmm2 = 0.5 * x * y * y
    subss   xmm3, xmm2         ; xmm3 = 1.5 - 0.5 * x * y * y
    mulss   xmm0, xmm3         ; xmm0 = y * (1.5 - ...)
    
    ret                        ; Return: xmm0 = 1/sqrt(x)

section .data
    half    dd 0.5
    one_5   dd 1.5
```

---

## 5. Overview ของ x86 Architecture {#x86-overview}

### 5.1 Register ของ x86-64

x86-64 มี General Purpose Registers 16 ตัว แต่ละตัวสามารถ Access ได้หลายขนาด:

```
64-bit Register Layout:
┌──────────────────────────────────────────────────────────────┐
│                          RAX (64-bit)                        │
│                          ┌───────────────────────────────────┤
│                          │       EAX (32-bit)                │
│                          │       ┌────────────────────────── ┤
│                          │       │      AX (16-bit)          │
│                          │       │   ┌──────┬───────────────┤
│                          │       │   │ AH   │     AL        │
│                          │       │   │(8-bit│    (8-bit)    │
└──────────────────────────┴───────┴───┴──────┴───────────────┘
Bit:  63              32    31     16  15   8   7            0
```

**ตารางแสดง Register ทั้งหมด:**

| 64-bit | 32-bit | 16-bit | 8-bit High | 8-bit Low | ใช้สำหรับ |
|--------|--------|--------|------------|-----------|-----------|
| RAX | EAX | AX | AH | AL | Accumulator, Return Value |
| RBX | EBX | BX | BH | BL | Base Register |
| RCX | ECX | CX | CH | CL | Counter (loops) |
| RDX | EDX | DX | DH | DL | Data Register |
| RSI | ESI | SI | - | SIL | Source Index |
| RDI | EDI | DI | - | DIL | Destination Index |
| RSP | ESP | SP | - | SPL | Stack Pointer |
| RBP | EBP | BP | - | BPL | Base Pointer |
| R8  | R8D | R8W | - | R8B | General Purpose |
| R9  | R9D | R9W | - | R9B | General Purpose |
| R10 | R10D | R10W | - | R10B | General Purpose |
| R11 | R11D | R11W | - | R11B | General Purpose |
| R12 | R12D | R12W | - | R12B | General Purpose |
| R13 | R13D | R13W | - | R13B | General Purpose |
| R14 | R14D | R14W | - | R14B | General Purpose |
| R15 | R15D | R15W | - | R15B | General Purpose |

**Special Registers:**

| Register | ความหมาย |
|----------|----------|
| RIP | Instruction Pointer - ชี้ไปที่ Instruction ถัดไป |
| RFLAGS | Flags Register - เก็บ Status ของผลการคำนวณ |
| CS, DS, ES, FS, GS, SS | Segment Registers |
| XMM0-XMM15 | 128-bit SIMD Registers (SSE) |
| YMM0-YMM15 | 256-bit SIMD Registers (AVX) |
| ZMM0-ZMM31 | 512-bit SIMD Registers (AVX-512) |

### 5.2 RFLAGS Register

```
RFLAGS Register Bits:
Bit 0:  CF - Carry Flag       (การ Carry/Borrow ใน Arithmetic)
Bit 2:  PF - Parity Flag      (จำนวน 1-bits ใน Result เป็นเลขคู่หรือไม่)
Bit 4:  AF - Auxiliary Flag   (Carry จาก Bit 3 ไป Bit 4 - BCD arithmetic)
Bit 6:  ZF - Zero Flag        (ผลลัพธ์เป็น 0 หรือไม่) **ใช้บ่อยมาก**
Bit 7:  SF - Sign Flag        (ผลลัพธ์เป็นลบหรือไม่)
Bit 8:  TF - Trap Flag        (Single-step mode สำหรับ Debugging)
Bit 9:  IF - Interrupt Flag   (เปิด/ปิด Maskable Interrupts)
Bit 10: DF - Direction Flag   (ทิศทางของ String Operations)
Bit 11: OF - Overflow Flag    (Signed Overflow)
```

**ตัวอย่างการใช้ Flags:**
```nasm
; ตัวอย่างการใช้ Zero Flag (ZF) สำหรับ Loop
section .text
global count_loop

count_loop:
    mov     ecx, 10             ; ตัวนับ = 10

.loop:
    ; ทำงานบางอย่าง...
    dec     ecx                 ; ECX = ECX - 1 และ Set Flags
                                ; ZF = 1 ถ้า ECX = 0
                                ; ZF = 0 ถ้า ECX != 0
    jnz     .loop               ; Jump ถ้า Zero Flag = 0 (Not Zero)
                                ; เมื่อ ECX = 0, ZF = 1, jnz ไม่ Jump → ออกจาก Loop
    ret
```

### 5.3 Memory Addressing Modes

```nasm
; x86-64 Memory Addressing: [Base + Index*Scale + Displacement]

; 1. Direct/Immediate: ค่าตัวเลขตรงๆ
mov     eax, 42             ; EAX = 42 (Immediate)

; 2. Register: ค่าจาก Register
mov     eax, ebx            ; EAX = ค่าใน EBX (Register)

; 3. Memory Direct: ค่าจาก Memory Address ตรงๆ
mov     eax, [0x402000]     ; EAX = ค่าที่ Address 0x402000

; 4. Register Indirect: Address อยู่ใน Register
mov     eax, [rbx]          ; EAX = ค่าที่ Address ใน RBX

; 5. Base + Displacement: Address = Register + Constant
mov     eax, [rbp-4]        ; EAX = ค่าที่ Address (RBP - 4)
                             ; ใช้สำหรับ Local Variables

; 6. Base + Index: Array Element
mov     eax, [rbx + rcx]    ; EAX = ค่าที่ Address (RBX + RCX)

; 7. Base + Index*Scale: Array Element (ขนาดมากกว่า 1 byte)
mov     eax, [rbx + rcx*4]  ; EAX = ค่าที่ Address (RBX + RCX*4)
                             ; ใช้สำหรับ int array (4 bytes/element)

; 8. Full Form: Base + Index*Scale + Displacement
mov     eax, [rbx + rcx*4 + 8]  ; ครบทุก Component
                                  ; เช่น struct array access
```

### 5.4 Calling Conventions (System V AMD64 ABI)

เมื่อเรียก Function ใน Linux/macOS (x86-64):

```
Function Arguments ส่งผ่าน Register ตามลำดับ:
  Integer/Pointer: RDI, RSI, RDX, RCX, R8, R9 (ถ้าเกิน 6 ตัว → Stack)
  Float/Double:   XMM0, XMM1, ..., XMM7

Return Value:
  Integer/Pointer: RAX (และ RDX ถ้า 128-bit)
  Float/Double:    XMM0

Caller-saved (Function สามารถเปลี่ยนได้):
  RAX, RCX, RDX, RSI, RDI, R8, R9, R10, R11

Callee-saved (Function ต้องบันทึกและ Restore):
  RBX, RBP, R12, R13, R14, R15
```

**ตัวอย่าง:**
```nasm
; ฟังก์ชัน: int add(int a, int b, int c)
; a อยู่ใน EDI, b อยู่ใน ESI, c อยู่ใน EDX
; Return value ต้องอยู่ใน EAX

section .text
global add_three

add_three:
    mov     eax, edi           ; EAX = a (EDI)
    add     eax, esi           ; EAX = a + b
    add     eax, edx           ; EAX = a + b + c
    ret                        ; Return: EAX = a + b + c
```

---

## 6. Overview ของ ARM Architecture {#arm-overview}

### 6.1 ARM64 (AArch64) Registers

ARM64 มี General Purpose Registers 31 ตัว (X0-X30) และ Special Registers:

```
ARM64 Register:
X0-X30: 64-bit General Purpose (W0-W30 เป็น 32-bit lower half)
XZR/WZR: Zero Register (อ่านได้เสมอ = 0, เขียนแล้วทิ้ง)
SP:     Stack Pointer (64-bit)
PC:     Program Counter (ไม่สามารถ Access โดยตรง)
LR (X30): Link Register (เก็บ Return Address)
```

**Calling Convention (ARM64 - AAPCS64):**
```
Function Arguments:
  X0-X7: Integer/Pointer (8 ตัวแรก)
  V0-V7: Float/SIMD (8 ตัวแรก)

Return Value:
  X0, X1: Integer/Pointer
  V0:     Float

Callee-saved:
  X19-X28, X29 (Frame Pointer), X30 (Link Register)

Caller-saved:
  X0-X18
```

### 6.2 ARM64 Instruction Set Overview

```asm
@ ARM64 Assembly (GAS Syntax) - Basic Examples

@ 1. Data Movement
mov     x0, #42             @ x0 = 42 (Immediate, ค่าเล็ก)
movz    x0, #0xABCD         @ x0 = 0xABCD (Zero-extend)
movk    x0, #0x1234, lsl #16 @ x0[31:16] = 0x1234 (Keep other bits)
ldr     x0, =0x123456789ABC @ โหลดค่าขนาดใหญ่ (PC-relative)

@ 2. Arithmetic
add     x0, x1, x2          @ x0 = x1 + x2
sub     x0, x1, x2          @ x0 = x1 - x2
mul     x0, x1, x2          @ x0 = x1 * x2 (lower 64 bits)
udiv    x0, x1, x2          @ x0 = x1 / x2 (unsigned)
sdiv    x0, x1, x2          @ x0 = x1 / x2 (signed)

@ 3. Logical
and     x0, x1, x2          @ x0 = x1 AND x2
orr     x0, x1, x2          @ x0 = x1 OR x2
eor     x0, x1, x2          @ x0 = x1 XOR x2
mvn     x0, x1              @ x0 = NOT x1

@ 4. Shift
lsl     x0, x1, #3          @ x0 = x1 << 3 (Logical Shift Left)
lsr     x0, x1, #3          @ x0 = x1 >> 3 (Logical Shift Right)
asr     x0, x1, #3          @ x0 = x1 >> 3 (Arithmetic Shift Right, sign-extend)

@ 5. Memory Access
ldr     x0, [x1]            @ x0 = Memory[x1]
ldr     x0, [x1, #8]        @ x0 = Memory[x1 + 8]
ldr     x0, [x1, x2]        @ x0 = Memory[x1 + x2]
str     x0, [x1]            @ Memory[x1] = x0
stp     x0, x1, [sp, #-16]! @ Push x0 และ x1 ลง Stack
ldp     x0, x1, [sp], #16   @ Pop x0 และ x1 จาก Stack

@ 6. Branch
b       label               @ Unconditional Branch
bl      func                @ Branch with Link (Call Function, เก็บ Return Address ใน X30)
ret                         @ Return (Branch to X30)
cbz     x0, label           @ Branch ถ้า x0 == 0
cbnz    x0, label           @ Branch ถ้า x0 != 0
```

### 6.3 ARM64 ตัวอย่าง Function

```asm
@ ARM64 (AArch64) GAS Syntax
@ Function: int factorial(int n)
@ Arguments: w0 = n
@ Returns: w0 = n!

.section .text
.global factorial
.type factorial, %function

factorial:
    @ ตรวจสอบ Base Case
    cmp     w0, #1              @ เปรียบเทียบ n กับ 1
    ble     .base_case          @ ถ้า n <= 1 ไป base case

    @ Recursive Call: n * factorial(n-1)
    stp     x29, x30, [sp, #-16]!  @ บันทึก Frame Pointer และ Link Register
    mov     x29, sp                @ ตั้ง Frame Pointer

    mov     w19, w0             @ บันทึก n ไว้ใน w19 (Callee-saved)
    sub     w0, w0, #1          @ w0 = n - 1
    bl      factorial           @ เรียก factorial(n-1), ผลอยู่ใน w0

    mul     w0, w19, w0         @ w0 = n * factorial(n-1)

    ldp     x29, x30, [sp], #16 @ Restore Frame Pointer และ Link Register
    ret                         @ Return

.base_case:
    mov     w0, #1              @ Return 1
    ret
```

### 6.4 RISC vs CISC เปรียบเทียบ

```nasm
; CISC (x86) - Instruction ซับซ้อน ทำได้หลายอย่างในครั้งเดียว
; คูณและบวกใน Memory Operations เดียว
imul    eax, [rbx + rcx*4 + 8]  ; EAX = EAX * Memory[RBX + RCX*4 + 8]
                                  ; 1 Instruction แต่ทำหลาย Operation
```

```asm
@ RISC (ARM64) - Instruction เรียบง่าย ทำทีละอย่าง
@ ทำสิ่งเดียวกัน แต่ต้องใช้หลาย Instruction
lsl     x2, x2, #2          @ x2 = rcx * 4
add     x2, x1, x2          @ x2 = rbx + rcx*4
ldr     w3, [x2, #8]        @ w3 = Memory[rbx + rcx*4 + 8]
mul     w0, w0, w3           @ w0 = eax * w3
@ 4 Instructions แต่แต่ละตัวง่ายและรวดเร็ว
```

---

## 7. Code Examples: Machine Code vs Assembly vs C {#code-examples}

### 7.1 Hello World ใน 3 ระดับ

**C Version:**
```c
// hello.c - High-level, อ่านง่าย
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
// Compile: gcc -o hello hello.c
// Run: ./hello
```

**x86-64 Assembly (Linux Syscall):**
```nasm
; hello_x86.asm - Assembly ใช้ Linux System Calls โดยตรง
; ไม่ต้องพึ่ง C Library!
; Compile: nasm -f elf64 hello_x86.asm -o hello_x86.o
; Link:    ld hello_x86.o -o hello_x86
; Run:     ./hello_x86

section .data
    message db "Hello, World!", 10   ; ข้อความ + newline (ASCII 10)
    msg_len equ $ - message          ; คำนวณความยาว: address ปัจจุบัน - address message

section .text
    global _start                    ; Entry point (ไม่ใช่ main เพราะไม่ใช้ C Library)

_start:
    ; System Call: write(1, message, msg_len)
    ; Syscall Number สำหรับ write บน Linux x86-64 = 1
    mov     rax, 1              ; syscall number: write (1)
    mov     rdi, 1              ; fd = 1 (STDOUT)
    lea     rsi, [rel message]  ; buffer = address ของ message
    mov     rdx, msg_len        ; count = ความยาวข้อความ
    syscall                     ; เรียก System Call (เข้า Kernel Mode)

    ; System Call: exit(0)
    ; Syscall Number สำหรับ exit บน Linux x86-64 = 60
    mov     rax, 60             ; syscall number: exit (60)
    xor     rdi, rdi            ; exit code = 0 (สำเร็จ)
    syscall                     ; เรียก System Call

; Expected Output:
; Hello, World!
```

**ARM64 Assembly (Linux Syscall):**
```asm
@ hello_arm64.s - ARM64 Assembly
@ Compile: as -o hello_arm64.o hello_arm64.s
@ Link:    ld hello_arm64.o -o hello_arm64
@ Run:     ./hello_arm64

.section .data
message:
    .ascii "Hello, World!\n"
msg_end:
msg_len = msg_end - message      @ คำนวณความยาวข้อความ

.section .text
.global _start

_start:
    @ System Call: write(1, message, msg_len)
    @ ARM64 Syscall Numbers แตกต่างจาก x86-64!
    mov     x8, #64             @ syscall number: write บน ARM64 = 64
    mov     x0, #1              @ fd = 1 (STDOUT)
    ldr     x1, =message        @ buffer = address ของ message
    mov     x2, #msg_len        @ count = ความยาว
    svc     #0                  @ Supervisor Call (เหมือน syscall บน x86)

    @ System Call: exit(0)
    mov     x8, #93             @ syscall number: exit บน ARM64 = 93
    mov     x0, #0              @ exit code = 0
    svc     #0

@ Expected Output:
@ Hello, World!
```

**Machine Code (Hex) ของ Hello World (x86-64):**
```
; นี่คือสิ่งที่ CPU จริงๆ execute (อ่านยากมาก)
; ไม่แนะนำให้เขียนแบบนี้!

48 c7 c0 01 00 00 00    ; mov rax, 1
48 c7 c7 01 00 00 00    ; mov rdi, 1
48 8d 35 XX XX XX XX    ; lea rsi, [rel message] (XX = offset)
48 c7 c2 0e 00 00 00    ; mov rdx, 14 (length)
0f 05                   ; syscall
48 c7 c0 3c 00 00 00    ; mov rax, 60
48 31 ff                ; xor rdi, rdi
0f 05                   ; syscall
```

### 7.2 ตัวอย่าง Loop: Sum 1 to 100

**C Version:**
```c
// sum.c
#include <stdio.h>

int main() {
    int sum = 0;
    for (int i = 1; i <= 100; i++) {
        sum += i;
    }
    printf("Sum 1 to 100 = %d\n", sum);  // Output: Sum 1 to 100 = 5050
    return 0;
}
```

**x86-64 Assembly:**
```nasm
; sum_x86.asm
; คำนวณ 1 + 2 + 3 + ... + 100

section .data
    fmt     db "Sum 1 to 100 = %d", 10, 0   ; Format string

section .text
    global main
    extern printf

main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16                ; จองพื้นที่ Stack

    ; int sum = 0
    mov     eax, 0                 ; EAX = sum = 0
    
    ; int i = 1
    mov     ecx, 1                 ; ECX = i = 1

.loop:
    ; Loop condition: i <= 100
    cmp     ecx, 100               ; เปรียบเทียบ i กับ 100
    jg      .done                  ; ถ้า i > 100 ออกจาก Loop

    ; sum += i
    add     eax, ecx               ; sum = sum + i

    ; i++
    inc     ecx                    ; i = i + 1
    
    jmp     .loop                  ; วนซ้ำ

.done:
    ; printf("Sum 1 to 100 = %d\n", sum)
    mov     esi, eax               ; Parameter 2: sum
    lea     rdi, [rel fmt]         ; Parameter 1: format string
    xor     eax, eax               ; Clear EAX
    call    printf

    xor     eax, eax               ; return 0
    leave
    ret

; Expected Output: Sum 1 to 100 = 5050
```

**ARM64 Assembly:**
```asm
@ sum_arm64.s
@ คำนวณ 1 + 2 + 3 + ... + 100

.section .data
fmt:
    .asciz "Sum 1 to 100 = %d\n"   @ Format string (null-terminated)

.section .text
.global main
.extern printf

main:
    stp     x29, x30, [sp, #-16]!  @ บันทึก FP, LR
    mov     x29, sp                @ ตั้ง Frame Pointer

    mov     w0, #0                 @ w0 = sum = 0
    mov     w1, #1                 @ w1 = i = 1

.loop:
    cmp     w1, #100               @ เปรียบเทียบ i กับ 100
    bgt     .done                  @ ถ้า i > 100 ออกจาก Loop

    add     w0, w0, w1             @ sum = sum + i
    add     w1, w1, #1             @ i = i + 1
    b       .loop                  @ วนซ้ำ

.done:
    @ printf(fmt, sum)
    mov     w1, w0                 @ Parameter 2: sum
    ldr     x0, =fmt               @ Parameter 1: format string
    bl      printf                 @ เรียก printf

    mov     w0, #0                 @ return 0
    ldp     x29, x30, [sp], #16    @ Restore FP, LR
    ret

@ Expected Output: Sum 1 to 100 = 5050
```

---

## 8. การดู Assembly ใน Compiler Output {#compiler-output}

### 8.1 ใช้ gcc -S

```bash
# สร้างไฟล์ C ก่อน
cat > example.c << 'EOF'
int add(int a, int b) {
    return a + b;
}

int main() {
    int x = 10;
    int y = 20;
    int z = add(x, y);
    return z;
}
EOF

# แปลงเป็น Assembly (ไม่ Optimize)
gcc -S -O0 -o example_O0.s example.c

# แปลงเป็น Assembly (Optimize ระดับ 2)
gcc -S -O2 -o example_O2.s example.c

# ดูความแตกต่าง
diff example_O0.s example_O2.s
```

**Output ของ gcc -S -O0 (ไม่ Optimize):**
```asm
# example_O0.s (AT&T Syntax - ที่ gcc ใช้เป็น default)
# Note: AT&T Syntax: source ก่อน, destination ทีหลัง (ตรงข้าม Intel)
# และมี prefix: r, e, ตาม Register ขนาด; % หน้า Register; $ หน้า Immediate

add:
    pushq   %rbp              # บันทึก RBP เก่า
    movq    %rsp, %rbp        # ตั้ง Frame Pointer ใหม่
    movl    %edi, -4(%rbp)    # เก็บ parameter a ลง Stack
    movl    %esi, -8(%rbp)    # เก็บ parameter b ลง Stack
    movl    -4(%rbp), %edx    # โหลด a เข้า EDX
    movl    -8(%rbp), %eax    # โหลด b เข้า EAX
    addl    %edx, %eax        # EAX = a + b
    popq    %rbp              # Restore RBP
    ret                       # Return (EAX = a + b)

main:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp         # จองพื้นที่ Local Variables
    movl    $10, -4(%rbp)     # int x = 10
    movl    $20, -8(%rbp)     # int y = 20
    movl    -8(%rbp), %edx    # โหลด y
    movl    -4(%rbp), %eax    # โหลด x
    movl    %edx, %esi        # Parameter 2: y
    movl    %eax, %edi        # Parameter 1: x
    call    add               # เรียก add(x, y)
    movl    %eax, -12(%rbp)   # เก็บ z = ผลลัพธ์
    movl    -12(%rbp), %eax   # โหลด z เป็น Return Value
    leave                     # Restore Stack Frame
    ret                       # Return z
```

**Output ของ gcc -S -O2 (Optimize ระดับ 2):**
```asm
# example_O2.s - Compiler ฉลาดมาก Optimize จนไม่เหลือ Memory Access เลย!
add:
    leal    (%rdi,%rsi), %eax  # EAX = a + b (ใน 1 Instruction!)
    ret

main:
    movl    $30, %eax          # Compile-time constant folding: 10+20=30
    ret                        # Return 30 โดยตรง ไม่ต้องเรียก add() เลย!
```

### 8.2 ใช้ Godbolt Compiler Explorer

Godbolt (https://godbolt.org) คือเครื่องมือออนไลน์ที่ดีที่สุดสำหรับดู Assembly Output:

```
วิธีใช้ Godbolt:
1. ไปที่ https://godbolt.org
2. เขียน C/C++ code ด้านซ้าย
3. เลือก Compiler (gcc, clang, msvc, etc.)
4. ใส่ Compiler Flags (เช่น -O2, -O3)
5. ดู Assembly Output ด้านขวา
6. สีของ code บ่งบอกว่า C code บรรทัดไหน → Assembly บรรทัดไหน
```

### 8.3 ใช้ objdump เพื่อ Disassemble Binary

```bash
# Compile ไปเป็น Binary
gcc -O0 -o example example.c

# Disassemble ด้วย objdump (Intel Syntax)
objdump -d -M intel example | head -50

# Output จะมีลักษณะ:
# Address     Opcode Bytes    Mnemonic    Operands
# 0000000000001139 <add>:
#    1139:    55                           push   rbp
#    113a:    48 89 e5                     mov    rbp,rsp
#    113d:    89 7d fc                     mov    DWORD PTR [rbp-0x4],edi
#    1140:    89 75 f8                     mov    DWORD PTR [rbp-0x8],esi
#    1143:    8b 55 fc                     mov    edx,DWORD PTR [rbp-0x4]
#    1146:    8b 45 f8                     mov    eax,DWORD PTR [rbp-0x8]
#    1149:    01 d0                        add    eax,edx
#    114b:    5d                           pop    rbp
#    114c:    c3                           ret
```

### 8.4 ใช้ GDB เพื่อ Debug Assembly

```bash
# Compile พร้อม Debug Info
gcc -g -O0 -o example example.c

# เปิด GDB
gdb ./example

# คำสั่งใน GDB สำหรับ Assembly
(gdb) disassemble main      # Disassemble ฟังก์ชัน main
(gdb) disassemble add       # Disassemble ฟังก์ชัน add
(gdb) break main            # Set Breakpoint ที่ main
(gdb) run                   # Run program
(gdb) stepi                 # Execute 1 Assembly Instruction
(gdb) nexti                 # Execute 1 Assembly Instruction (ไม่เข้า Function)
(gdb) info registers        # ดู Register ทั้งหมด
(gdb) x/10i $rip            # ดู 10 Instructions ถัดจาก RIP
(gdb) x/4xb 0x402000        # ดู Memory 4 bytes ที่ address 0x402000 (hex)
(gdb) layout asm            # เปิด Assembly View
(gdb) layout regs           # เปิด Register View
```

---

## 9. ข้อผิดพลาดที่พบบ่อยสำหรับผู้เริ่มต้น {#common-mistakes}

### 9.1 ลืม Null Terminate String

```nasm
; ผิด! (สำหรับ C functions ที่ต้องการ null-terminated string)
msg     db "Hello, World!", 10   ; ไม่มี null byte ต่อท้าย!

; ถูก!
msg     db "Hello, World!", 10, 0  ; มี null byte (0) ต่อท้าย
; หรือ
msg     db "Hello, World!", 10
        db 0                       ; null terminator บรรทัดแยก
```

### 9.2 ลืมบันทึก Callee-saved Registers

```nasm
; ผิด! ถ้า Function ใช้ RBX แต่ไม่บันทึกค่าเดิม
bad_function:
    mov     rbx, 42        ; ทำลายค่าเดิมของ RBX!
    ; ... ทำงาน ...
    ret

; ถูก! บันทึกและ Restore
good_function:
    push    rbx            ; บันทึกค่า RBX เดิม
    mov     rbx, 42        ; ตอนนี้ปลอดภัยแล้ว
    ; ... ทำงาน ...
    pop     rbx            ; Restore ค่า RBX เดิม
    ret
```

### 9.3 Stack Alignment

```nasm
; x86-64 ABI ต้องการให้ RSP Aligned เป็น 16 bytes ก่อนเรียก Function

; ผิด! (อาจ Crash ใน printf หรือ Function ที่ใช้ XMM)
bad_call:
    push    rbp
    mov     rbp, rsp
    ; RSP ตอนนี้อาจไม่ Aligned!
    call    printf         ; CRASH if using SSE/AVX internally!
    
; ถูก! ตรวจสอบ Alignment
good_call:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 8         ; Align to 16 bytes (push rbp took 8 bytes)
    call    printf
    add     rsp, 8
    pop     rbp
    ret
```

### 9.4 ขนาด Operand ไม่ตรงกัน

```nasm
; ผิด!
mov     eax, rbx       ; Error: ขนาดไม่ตรง (32-bit ← 64-bit)
mov     al, 256        ; Warning/Error: 256 ไม่จุใน 8-bit (max 255)

; ถูก!
mov     eax, ebx       ; 32-bit ← 32-bit ตรงกัน
mov     rax, rbx       ; 64-bit ← 64-bit ตรงกัน
mov     al, 255        ; 8-bit ← ค่าที่จุได้ใน 8-bit
```

### 9.5 ลืม section declarations

```nasm
; ผิด!
msg     db "Hello", 0   ; ถ้าไม่มี section .data จะ Error หรือไม่ทำงานถูกต้อง

_start:                 ; ถ้าไม่มี section .text จะ Error
    mov rax, 1

; ถูก!
section .data
    msg     db "Hello", 0

section .text
    global _start
_start:
    mov rax, 1
```

---

## 10. แบบฝึกหัดท้ายบท {#exercises}

### แบบฝึกหัดที่ 1: ทำความเข้าใจ Assembly Output

**โจทย์:** เขียนโปรแกรม C ต่อไปนี้ แล้วดู Assembly Output ด้วย `gcc -S`

```c
// exercise1.c
int multiply(int x, int y) {
    return x * y;
}

int square(int n) {
    return multiply(n, n);
}

int main() {
    return square(7);  // ควรได้ 49
}
```

**คำถาม:**
1. Compile ด้วย `gcc -S -O0 exercise1.c` - มีกี่ Instruction?
2. Compile ด้วย `gcc -S -O2 exercise1.c` - เปลี่ยนไปอย่างไร?
3. ลองใช้ `gcc -S -O3 exercise1.c` - Compiler ทำ Inlining หรือไม่?

**เฉลยแนวคิด:**
```bash
# วิธีทำ
cat > exercise1.c << 'EOF'
int multiply(int x, int y) { return x * y; }
int square(int n) { return multiply(n, n); }
int main() { return square(7); }
EOF

gcc -S -O0 -o ex1_O0.s exercise1.c
gcc -S -O2 -o ex1_O2.s exercise1.c
gcc -S -O3 -o ex1_O3.s exercise1.c

# ดูความแตกต่าง
wc -l ex1_O0.s ex1_O2.s ex1_O3.s   # นับจำนวนบรรทัด

# ด้วย -O3 ผลลัพธ์น่าจะเป็น:
# main:
#     movl $49, %eax   <- Compiler คำนวณ 7*7=49 ตอน compile เวลา!
#     ret
```

### แบบฝึกหัดที่ 2: ระบุส่วนประกอบของ Assembly

**โจทย์:** อ่าน Assembly ต่อไปนี้ และตอบคำถาม

```nasm
; mystery.asm
section .data
    numbers dd 10, 20, 30, 40, 50    ; Array of 5 integers

section .text
    global mystery

mystery:
    push    rbp
    mov     rbp, rsp
    
    xor     eax, eax           ; A
    mov     ecx, 5             ; B
    lea     rdx, [rel numbers] ; C
    
.loop:
    add     eax, [rdx]         ; D
    add     rdx, 4             ; E
    dec     ecx                ; F
    jnz     .loop              ; G
    
    pop     rbp
    ret
```

**คำถาม:**
1. บรรทัด A ทำอะไร? ทำไมถึงใช้ `xor eax, eax` แทน `mov eax, 0`?
2. บรรทัด B ตั้งค่าอะไร?
3. บรรทัด C ทำอะไร? `rel` คืออะไร?
4. บรรทัด D-G เป็น Loop ที่ทำอะไร?
5. Function นี้ Return ค่าอะไร?

**เฉลย:**
```
A: xor eax, eax = EAX = 0 (สะอาด Zero register)
   ใช้ XOR เพราะ Machine Code สั้นกว่า mov eax, 0
   xor eax, eax = 2 bytes
   mov eax, 0   = 5 bytes
   
B: ECX = 5 (Loop Counter)

C: RDX = Address ของ array "numbers" (PC-relative addressing)
   rel = relative to RIP (สำหรับ Position Independent Code)

D-G: Loop ที่บวกค่าทุกตัวใน Array
   D: EAX += *RDX (บวกค่าที่ RDX ชี้อยู่)
   E: RDX += 4 (ชี้ไปที่ element ถัดไป, int = 4 bytes)
   F: ECX-- (ลด Counter)
   G: ถ้า ECX != 0 วนซ้ำ

Return Value: EAX = 10+20+30+40+50 = 150
```

### แบบฝึกหัดที่ 3: เขียน Hello World

**โจทย์:** เขียน Hello World ด้วย x86-64 Assembly โดย:
1. ใช้ Linux System Calls โดยตรง (ไม่ใช้ printf)
2. แสดงข้อความ "Hello from Assembly!" 
3. Compile และ Run ได้จริง

```bash
# ขั้นตอนการ Setup Environment
# สำหรับ Ubuntu/Debian:
sudo apt update
sudo apt install nasm gcc build-essential gdb

# สร้างไฟล์
nano hello_world.asm    # หรือ editor ที่ชอบ

# Compile
nasm -f elf64 hello_world.asm -o hello_world.o
ld hello_world.o -o hello_world

# Run
./hello_world

# คาดหวังผลลัพธ์:
# Hello from Assembly!
```

**เฉลย:**
```nasm
; hello_solution.asm
section .data
    msg     db "Hello from Assembly!", 10   ; ข้อความ + newline
    msglen  equ $ - msg                     ; ความยาว

section .text
    global _start

_start:
    mov     rax, 1          ; syscall: write
    mov     rdi, 1          ; fd: stdout
    lea     rsi, [rel msg]  ; buffer
    mov     rdx, msglen     ; length
    syscall

    mov     rax, 60         ; syscall: exit
    xor     rdi, rdi        ; exit code: 0
    syscall
```

### แบบฝึกหัดที่ 4: Register เลขคณิต

**โจทย์:** เขียน Assembly Function ที่รับ 3 พารามิเตอร์ (a, b, c) และ Return ค่า `(a + b) - c`

```nasm
; template:
; พารามิเตอร์ส่งผ่าน RDI, RSI, RDX (Linux x86-64 Calling Convention)
; Return Value ต้องอยู่ใน RAX

section .text
global calc

calc:
    ; เขียน code ตรงนี้
    ; ...
    ret
```

**เฉลย:**
```nasm
section .text
global calc

; int calc(int a, int b, int c)
; a = EDI, b = ESI, c = EDX
; return (a + b) - c
calc:
    mov     eax, edi        ; EAX = a
    add     eax, esi        ; EAX = a + b
    sub     eax, edx        ; EAX = (a + b) - c
    ret                     ; Return EAX
```

### แบบฝึกหัดที่ 5: ค้นคว้าเพิ่มเติม

**โจทย์:** ทำการทดลองต่อไปนี้และบันทึกผล:

1. **ดู Syscall Table:**
   ```bash
   # Linux x86-64 Syscall Numbers
   cat /usr/include/x86_64-linux-gnu/asm/unistd_64.h | head -50
   # หรือ
   ausyscall --dump | head -20
   ```

2. **ดู CPU Information:**
   ```bash
   # ดู CPU รุ่นและ Feature Flags
   cat /proc/cpuinfo | grep -E "model name|flags" | head -5
   # ค้นหา flags เช่น: aes, avx, avx2, bmi1, bmi2, sse4_2
   ```

3. **ดู Binary ด้วย hexdump:**
   ```bash
   # Compile โปรแกรมง่ายๆ
   echo 'int main(){return 42;}' > t.c
   gcc -o t t.c
   # ดู Bytes แรก (ELF Header)
   hexdump -C t | head -20
   # จะเห็น: 7f 45 4c 46 = ELF Magic Number
   ```

4. **ดู Assembly ใน Python:**
   ```python
   # Python ก็มี Assembly ภายในเช่นกัน!
   # ใช้ dis module ดู Python Bytecode
   import dis
   
   def add(a, b):
       return a + b
   
   dis.dis(add)
   # Output จะแสดง Bytecode Instructions ของ Python
   ```

---

## 11. สรุปและ Key Takeaways {#summary}

### สิ่งที่ได้เรียนรู้ใน Part นี้

1. **Assembly Language คืออะไร:** ภาษา Low-level ที่แปลง Mnemonic ไป Machine Code ได้ 1:1 (เกือบ)

2. **ทำไมต้องเรียน:**
   - เข้าใจ Hardware ลึกขึ้น
   - Security/Reverse Engineering
   - Performance Optimization
   - OS/Embedded Development
   - เข้าใจ Compiler ดีขึ้น

3. **ประวัติศาสตร์:**
   - Assembly เกิดขึ้นในช่วง 1940s-1950s เพื่อแก้ปัญหา Binary Programming
   - x86 มาจาก Intel 8086 (1978) → IA-32 → x86-64
   - ARM มาจาก Acorn (1983) → ARMv8/AArch64 ที่ใช้ใน Smartphones และ Apple M-series

4. **ความแตกต่างของ Levels:**
   - High-level (Python, Java) → ง่าย แต่ช้ากว่า
   - Mid-level (C, Rust) → สมดุล
   - Assembly → ใกล้ Hardware สูงสุด ควบคุมได้มากสุด
   - Machine Code → CPU เข้าใจโดยตรง มนุษย์อ่านยาก

5. **x86-64 สำคัญ:**
   - 16 General Purpose Registers (RAX-R15)
   - RFLAGS สำหรับ Conditional Branching
   - Complex Addressing Modes
   - Calling Convention: Arguments ใน RDI, RSI, RDX, RCX, R8, R9

6. **ARM64 สำคัญ:**
   - 31 General Purpose Registers (X0-X30)
   - Fixed 4-byte Instruction Size
   - RISC: Simple Instructions
   - Calling Convention: Arguments ใน X0-X7

7. **Tools สำคัญ:**
   - `nasm` - x86 Assembler
   - `as` - GNU Assembler (สำหรับ ARM และ AT&T Syntax)
   - `gcc -S` - ดู Assembly Output
   - `objdump -d` - Disassemble Binary
   - `gdb` - Debugger
   - Godbolt - Online Compiler Explorer

### Preview: Part ต่อไป

**Part 002:** การติดตั้งและ Setup Development Environment
- ติดตั้ง NASM, GAS
- Setup สำหรับ Linux, macOS, Windows (WSL)
- IDE และ Text Editor สำหรับ Assembly
- สร้างและรัน Hello World แรก
- ทำความเข้าใจ ELF/PE Binary Format

---

## 12. แหล่งอ้างอิงและแหล่งเรียนรู้เพิ่มเติม {#references}

### หนังสือแนะนำ

| ชื่อหนังสือ | ผู้แต่ง | ระดับ | หมายเหตุ |
|------------|---------|-------|----------|
| Programming from the Ground Up | Jonathan Bartlett | Beginner | ฟรี, Linux x86 |
| The Art of Assembly Language | Randall Hyde | Beginner-Intermediate | ครอบคลุมมาก |
| Professional Assembly Language | Richard Blum | Intermediate | x86 Linux |
| x86 Assembly Language and C Fundamentals | Joseph Cavanagh | Beginner | มี C เปรียบเทียบ |
| Computer Organization and Design | Patterson & Hennessy | All | ARM Edition ดีมาก |
| Low-Level Programming | Igor Zhirkov | Intermediate | x86-64 Linux |

### Online Resources

- **Godbolt Compiler Explorer:** https://godbolt.org - ดู Assembly Output แบบ Real-time
- **Intel Software Developer Manual:** https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html
- **ARM Architecture Reference Manual:** https://developer.arm.com/documentation/
- **x86 Instruction Reference:** https://www.felixcloutier.com/x86/
- **NASM Manual:** https://www.nasm.us/doc/
- **Linux Syscall Table:** https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md

### ช่องทาง Video

- **Low Level Learning (YouTube)** - ดีมากสำหรับ Assembly และ Systems Programming
- **LiveOverflow (YouTube)** - Binary Exploitation, Assembly
- **Ben Eater (YouTube)** - Hardware Level, How CPU works
- **Davy Wybiral (YouTube)** - Assembly Programming Tutorials

### Practice Platforms

- **pwn.college** - เรียน Assembly ผ่าน CTF challenges
- **microcorruption.com** - MSP430 Assembly Exploitation
- **reversing.kr** - Reverse Engineering challenges
- **exploit.education** - Binary Exploitation labs

### Communities

- **/r/asm** - Reddit Assembly Community
- **Stack Overflow** - ถามคำถาม tag: [assembly], [x86], [arm]
- **OSDev.org Wiki** - สำหรับ OS Development
- **freenode IRC #asm** - Real-time help

---

## Appendix A: Quick Reference Card

### x86-64 Common Instructions

```nasm
; Data Movement
mov  dst, src     ; dst = src
lea  dst, [addr]  ; dst = address (ไม่ dereference)
push src          ; Stack push
pop  dst          ; Stack pop
xchg dst, src     ; Swap values

; Arithmetic
add  dst, src     ; dst += src
sub  dst, src     ; dst -= src
mul  src          ; RDX:RAX = RAX * src (unsigned)
imul dst, src     ; dst *= src (signed)
div  src          ; RAX = RDX:RAX / src; RDX = remainder (unsigned)
idiv src          ; Same แต่ signed
inc  dst          ; dst++
dec  dst          ; dst--
neg  dst          ; dst = -dst

; Logical
and  dst, src     ; dst &= src
or   dst, src     ; dst |= src
xor  dst, src     ; dst ^= src
not  dst          ; dst = ~dst
shl  dst, count   ; dst <<= count
shr  dst, count   ; dst >>= count (logical)
sar  dst, count   ; dst >>= count (arithmetic, sign-extend)

; Comparison & Jump
cmp  a, b         ; Set Flags โดย a-b (ไม่เก็บผล)
test a, b         ; Set Flags โดย a&b (ไม่เก็บผล)
jmp  label        ; Unconditional Jump
je   label        ; Jump if Equal (ZF=1)
jne  label        ; Jump if Not Equal (ZF=0)
jl   label        ; Jump if Less (signed)
jg   label        ; Jump if Greater (signed)
jle  label        ; Jump if Less or Equal (signed)
jge  label        ; Jump if Greater or Equal (signed)
jb   label        ; Jump if Below (unsigned)
ja   label        ; Jump if Above (unsigned)

; Function Call
call label        ; Push RIP, Jump to label
ret               ; Pop RIP, Jump
leave             ; mov rsp,rbp; pop rbp
```

### ARM64 Common Instructions

```asm
@ Data Movement
mov  xN, xM          @ xN = xM
mov  xN, #imm        @ xN = immediate
ldr  xN, [xM]        @ xN = Memory[xM]
ldr  xN, [xM, #off]  @ xN = Memory[xM + off]
str  xN, [xM]        @ Memory[xM] = xN
stp  xN, xM, [xP]    @ Store pair
ldp  xN, xM, [xP]    @ Load pair

@ Arithmetic
add  xN, xM, xP      @ xN = xM + xP
sub  xN, xM, xP      @ xN = xM - xP
mul  xN, xM, xP      @ xN = xM * xP
udiv xN, xM, xP      @ xN = xM / xP (unsigned)
sdiv xN, xM, xP      @ xN = xM / xP (signed)

@ Comparison & Branch
cmp  xN, xM          @ Set flags: xN - xM
b    label           @ Unconditional branch
bl   label           @ Branch with link (call)
ret                  @ Branch to X30 (return)
beq  label           @ Branch if equal
bne  label           @ Branch if not equal
blt  label           @ Branch if less than (signed)
bgt  label           @ Branch if greater than (signed)
cbz  xN, label       @ Branch if xN == 0
cbnz xN, label       @ Branch if xN != 0
```

---

## Appendix B: Linux System Calls (x86-64)

| Syscall | Number | Arguments | คำอธิบาย |
|---------|--------|-----------|----------|
| read | 0 | fd, buf, count | อ่านข้อมูลจาก File Descriptor |
| write | 1 | fd, buf, count | เขียนข้อมูลไปยัง File Descriptor |
| open | 2 | filename, flags, mode | เปิดไฟล์ |
| close | 3 | fd | ปิด File Descriptor |
| mmap | 9 | addr, len, prot, flags, fd, off | Map Memory |
| munmap | 11 | addr, len | Unmap Memory |
| brk | 12 | addr | เปลี่ยนขนาด Heap |
| getpid | 39 | - | ได้รับ Process ID |
| fork | 57 | - | สร้าง Process ใหม่ |
| execve | 59 | filename, argv, envp | รัน Program |
| exit | 60 | status | ออกจาก Process |
| kill | 62 | pid, sig | ส่ง Signal ไปยัง Process |

**วิธีเรียก System Call บน x86-64 Linux:**
```nasm
; Register สำหรับ System Call:
; RAX = Syscall Number
; RDI = Argument 1
; RSI = Argument 2
; RDX = Argument 3
; R10 = Argument 4
; R8  = Argument 5
; R9  = Argument 6
; Return Value: RAX (ถ้าเป็นลบ = Error: errno = -RAX)

; ตัวอย่าง: read(0, buffer, 100) อ่านจาก stdin 100 bytes
section .bss
    buffer  resb 100    ; จองพื้นที่ 100 bytes (ไม่ Initialize)

section .text
    global _start

_start:
    mov     rax, 0              ; syscall: read (0)
    mov     rdi, 0              ; fd = 0 (stdin)
    lea     rsi, [rel buffer]   ; buf = buffer
    mov     rdx, 100            ; count = 100
    syscall                     ; เรียก System Call
    ; RAX = จำนวน bytes ที่อ่านได้ (หรือ Error ถ้าเป็นลบ)
```

---

**จบ Part 001: บทนำสู่ Assembly Language**

ใน Part ถัดไป (Part 002) เราจะลงมือติดตั้ง Development Environment และเขียนโปรแกรมแรกด้วย Assembly จริงๆ

---
*Assembly Language Programming Course - Part 001 of 100+*  
*สร้างโดย: หลักสูตรนี้ครอบคลุมตั้งแต่ระดับ Beginner ถึง World-class Professional*
