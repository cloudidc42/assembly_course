# Part 007: ARM Architecture เบื้องต้น
## หลักสูตร Assembly Programming ตั้งแต่พื้นฐานถึงระดับโลก

**ระดับ:** พื้นฐาน-กลาง  
**เวลาที่ใช้:** ประมาณ 5-7 ชั่วโมง  
**ความต่อเนื่อง:** ต่อจาก Part 006 (x86 Addressing Modes)

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

เมื่อเรียนจบ Part นี้แล้ว ผู้เรียนจะสามารถ:

- อธิบายประวัติและวิวัฒนาการของ ARM architecture ตั้งแต่ ARM1 จนถึงปัจจุบัน
- เข้าใจความแตกต่างระหว่าง ARM Architecture Profiles (A, R, M)
- ใช้งาน ARM32 registers ทั้งหมดได้อย่างถูกต้อง รวมถึง CPSR flags
- เขียนโปรแกรม ARM32 assembly ด้วย Thumb และ Thumb-2 instruction sets
- ใช้งาน AArch64 (ARM64) registers และ NZCV flags
- เปรียบเทียบความแตกต่างระหว่าง ARM และ x86 architecture
- เขียนโปรแกรมสำหรับ Cortex-M4 embedded systems เบื้องต้น
- ทำความเข้าใจว่า ARM ถูกใช้งานในอุปกรณ์ประเภทใดบ้าง

---

## 1. ประวัติและวิวัฒนาการของ ARM

### 1.1 จุดเริ่มต้นของ ARM (1983-1985)

ARM ย่อมาจาก **Acorn RISC Machine** (ต่อมาเปลี่ยนเป็น **Advanced RISC Machine**)

บริษัท Acorn Computers ของอังกฤษเริ่มพัฒนา ARM processor ในปี ค.ศ. 1983 โดยมีเป้าหมายหลักคือ:
- สร้าง processor ที่มีประสิทธิภาพสูงแต่ใช้พลังงานต่ำ
- ใช้แนวคิด RISC (Reduced Instruction Set Computer) เพื่อให้ instruction set เรียบง่าย
- ลดจำนวน transistors เพื่อลดต้นทุนและพลังงาน

**ARM1** (1985) เป็น chip ตัวแรก ทำงานที่ 8 MHz มี transistors เพียง 25,000 ตัว (เทียบกับ Intel 286 ที่มี 134,000 ตัว)

### 1.2 Timeline วิวัฒนาการ ARM

| ปี | รุ่น | ความสำคัญ |
|----|------|-----------|
| 1985 | ARM1 | ARM processor ตัวแรก, 8 MHz |
| 1986 | ARM2 | ใช้ใน Acorn Archimedes, 8 MIPS |
| 1989 | ARM3 | เพิ่ม cache, ประสิทธิภาพดีขึ้น |
| 1991 | ARM6 | รองรับ 32-bit addressing เต็มรูปแบบ |
| 1993 | ARM7 | Thumb instruction set เริ่มต้นใน ARM7T |
| 1997 | ARM9 | Pipeline 5 stages, ความถี่สูงขึ้น |
| 1998 | ARM Ltd. ก่อตั้ง | แยกออกจาก Acorn เป็นบริษัทแยก |
| 2001 | ARM10 | VFP floating point |
| 2004 | ARM11 | ARMv6, ใช้ใน iPhone รุ่นแรก |
| 2005 | Cortex-M3 | ARM สำหรับ microcontroller |
| 2007 | Cortex-A8 | ARMv7-A, Thumb-2, NEON SIMD |
| 2009 | Cortex-M4 | DSP instructions, FPU optional |
| 2011 | Cortex-A15 | ARMv7-A, Big.LITTLE ครั้งแรก |
| 2012 | Cortex-A53/A57 | ARMv8-A, AArch64 (64-bit) |
| 2016 | Cortex-A73 | ประสิทธิภาพสูง, พลังงานต่ำ |
| 2019 | Cortex-A77 | IPC เพิ่มขึ้น 20% |
| 2020 | Cortex-X1 | "Premium" core สำหรับ flagship |
| 2021 | Cortex-X2 | ARMv9, SVE2 |
| 2022 | Cortex-X3 | ปรับปรุงประสิทธิภาพ |
| 2023 | Cortex-X4 | ARMv9.2, ประสิทธิภาพสูงสุดในตระกูล |

### 1.3 ARM ใน Products ที่มีชื่อเสียง

```
iPhone/iPad:   Apple A-series (A4 = ARM Cortex-A8, A14/A15/A16/A17 = Apple custom ARMv8.x)
Android:       Qualcomm Snapdragon (Cortex-X/A series)
               Samsung Exynos (Cortex-X/A series)
               MediaTek Dimensity (Cortex-X/A series)
MacBook (M1+): Apple Silicon (custom ARMv8.5+ cores)
Raspberry Pi:  Cortex-A53 (Pi 3), Cortex-A72 (Pi 4)
Arduino:       ATmega328 (AVR ไม่ใช่ ARM), Due (Cortex-M3), Zero (Cortex-M0+)
STM32:         Cortex-M0/M3/M4/M7
AWS Graviton:  Custom ARMv8/v9 สำหรับ cloud server
```

---

## 2. ARM Architecture Profiles

ARM แบ่ง architecture ออกเป็น 3 profiles หลัก:

### 2.1 Profile A - Application Profile (Cortex-A)

```
ลักษณะ: ประสิทธิภาพสูง, รองรับ OS แบบเต็มรูปแบบ
ตัวอย่าง: Cortex-A5, A7, A8, A9, A53, A55, A72, A78, X1, X2, X3, X4
การใช้งาน: Smartphone, Tablet, Laptop (Apple M-series), Server

Memory: รองรับ Virtual Memory Management Unit (MMU)
OS: Linux, Android, iOS, Windows on ARM
Instruction: ARM32 + Thumb + Thumb-2 หรือ AArch64
```

### 2.2 Profile R - Real-time Profile (Cortex-R)

```
ลักษณะ: Real-time performance, deterministic behavior
ตัวอย่าง: Cortex-R4, R5, R7, R8, R52, R82
การใช้งาน: Automotive (ABS, Airbag), Medical devices, Storage controllers

Memory: Memory Protection Unit (MPU) แทน MMU
OS: RTOS (FreeRTOS, QNX, VxWorks)
Requirement: ต้องการ latency ที่แน่นอน (deterministic)
```

### 2.3 Profile M - Microcontroller Profile (Cortex-M)

```
ลักษณะ: ใช้พลังงานต่ำ, ราคาถูก, โปรแกรมง่าย
ตัวอย่าง: Cortex-M0, M0+, M3, M4, M7, M23, M33, M55, M85
การใช้งาน: IoT, Wearables, Sensors, Industrial control

Memory: MPU (optional), ไม่มี MMU
OS: Bare-metal หรือ RTOS ขนาดเล็ก
ISA: Thumb-2 เป็นหลัก (ไม่รองรับ ARM32)
```

---

## 3. ARM32 Registers

### 3.1 General Purpose Registers (R0-R15)

ARM32 มี 16 registers ขนาด 32-bit ทั้งหมด:

```
Register | Alias | หน้าที่หลัก
---------|-------|-------------
R0       | -     | Argument/Return value, Scratch register
R1       | -     | Argument/Return value, Scratch register
R2       | -     | Argument, Scratch register
R3       | -     | Argument, Scratch register
R4       | -     | Callee-saved (ต้องรักษาค่า)
R5       | -     | Callee-saved
R6       | -     | Callee-saved
R7       | -     | Callee-saved (Thumb syscall number)
R8       | -     | Callee-saved
R9       | -     | Platform register (อาจเป็น thread pointer)
R10      | -     | Callee-saved
R11      | FP    | Frame Pointer (optional)
R12      | IP    | Intra-Procedure call scratch register
R13      | SP    | Stack Pointer
R14      | LR    | Link Register (return address)
R15      | PC    | Program Counter
```

### 3.2 CPSR - Current Program Status Register

CPSR เก็บสถานะปัจจุบันของโปรเซสเซอร์:

```
Bit 31: N  = Negative flag
Bit 30: Z  = Zero flag
Bit 29: C  = Carry flag
Bit 28: V  = Overflow flag
Bit 27: Q  = Saturation flag (DSP)
Bit 26-25: IT[1:0] = If-Then execution state
Bit 24: J  = Jazelle state bit (Java bytecode)
Bit 23-20: Reserved
Bit 19-16: GE[3:0] = Greater-than or Equal flags (SIMD)
Bit 15-10: IT[7:2] = If-Then execution state
Bit 9: E   = Endianness (0=Little, 1=Big)
Bit 8: A   = Asynchronous abort disable
Bit 7: I   = IRQ disable
Bit 6: F   = FIQ disable
Bit 5: T   = Thumb state (0=ARM, 1=Thumb)
Bit 4-0: M[4:0] = Processor mode
```

**Processor Modes:**
```
M[4:0] | Mode         | คำอธิบาย
10000  | User         | โหมดปกติสำหรับ application
10001  | FIQ          | Fast Interrupt Request handler
10010  | IRQ          | Normal Interrupt Request handler
10011  | Supervisor   | OS kernel, SVC instruction
10110  | Monitor      | TrustZone security
10111  | Abort        | Memory access fault handler
11010  | Hypervisor   | Virtualization
11011  | Undefined    | Undefined instruction handler
11111  | System       | Privileged user mode
```

### 3.3 SPSR - Saved Program Status Register

เมื่อเกิด exception, CPSR จะถูกบันทึกไว้ใน SPSR เพื่อให้ return กลับมาได้ถูกต้อง

---

## 4. ARM32 Instruction Set

### 4.1 รูปแบบ Instruction

ARM32 ใช้ instructions ขนาด 32-bit คงที่ทุก instruction:

```
Bit 31-28: Condition code (4 bits) - เงื่อนไขการทำงาน
Bit 27-25: Instruction type (3 bits)
Bit 24-...: ส่วนที่เหลือขึ้นอยู่กับประเภท instruction
```

**Condition Codes:**
```
Suffix | Code | Condition          | Flags tested
-------|------|-------------------|------------------
EQ     | 0000 | Equal             | Z=1
NE     | 0001 | Not Equal         | Z=0
CS/HS  | 0010 | Carry Set         | C=1
CC/LO  | 0011 | Carry Clear       | C=0
MI     | 0100 | Minus/Negative    | N=1
PL     | 0101 | Plus/Positive     | N=0
VS     | 0110 | Overflow Set      | V=1
VC     | 0111 | Overflow Clear    | V=0
HI     | 1000 | Higher (unsigned) | C=1 and Z=0
LS     | 1001 | Lower or Same     | C=0 or Z=1
GE     | 1010 | Greater or Equal  | N=V
LT     | 1011 | Less Than         | N!=V
GT     | 1100 | Greater Than      | Z=0 and N=V
LE     | 1101 | Less or Equal     | Z=1 or N!=V
AL     | 1110 | Always (default)  | (any)
```

---

## 5. โค้ดตัวอย่าง ARM32 เบื้องต้น

### 5.1 Hello World บน ARM32 Linux

```asm
@ ไฟล์: hello_arm32.s
@ คอมไพล์: as -o hello_arm32.o hello_arm32.s && ld -o hello_arm32 hello_arm32.o
@ รัน: ./hello_arm32
@ หมายเหตุ: @ ใช้สำหรับ comment ใน ARM GAS syntax

.section .data
    @ กำหนด string สำหรับแสดงผล
    msg:    .ascii "Hello, ARM32 World!\n"  @ ข้อความที่จะแสดง
    msg_len = . - msg                        @ คำนวณความยาวของข้อความ

.section .text
    .global _start  @ ประกาศ entry point สำหรับ linker

_start:
    @ === Write syscall ===
    @ ARM32 Linux syscall convention:
    @ R7 = syscall number
    @ R0 = arg1, R1 = arg2, R2 = arg3

    MOV R7, #4          @ syscall number 4 = sys_write
    MOV R0, #1          @ file descriptor: 1 = stdout
    LDR R1, =msg        @ pointer ไปยัง string
    MOV R2, #msg_len    @ จำนวน bytes ที่จะเขียน
    SWI #0              @ Software Interrupt = เรียก syscall (หรือใช้ SVC #0)

    @ === Exit syscall ===
    MOV R7, #1          @ syscall number 1 = sys_exit
    MOV R0, #0          @ exit code = 0 (สำเร็จ)
    SWI #0              @ เรียก syscall
```

**ผลลัพธ์ที่คาดหวัง:**
```
Hello, ARM32 World!
```

### 5.2 การดำเนินการทางคณิตศาสตร์ ARM32

```asm
@ ไฟล์: arithmetic_arm32.s
@ สาธิตการใช้งาน arithmetic instructions

.section .text
    .global _start

_start:
    @ === การบวก (ADD) ===
    MOV R0, #10         @ R0 = 10
    MOV R1, #20         @ R1 = 20
    ADD R2, R0, R1      @ R2 = R0 + R1 = 30

    @ === การลบ (SUB) ===
    MOV R3, #50         @ R3 = 50
    MOV R4, #15         @ R4 = 15
    SUB R5, R3, R4      @ R5 = R3 - R4 = 35

    @ === การคูณ (MUL) ===
    MOV R6, #7          @ R6 = 7
    MOV R7, #8          @ R7 = 8
    MUL R8, R6, R7      @ R8 = R6 * R7 = 56
    @ หมายเหตุ: destination ต้องไม่ใช่ operand เดียวกับ source ใน ARM32

    @ === การหาร ===
    @ ARM32 ไม่มี DIV instruction โดยตรง
    @ ต้องใช้ UDIV/SDIV (ARMv7-A ขึ้นไป) หรือ software division
    MOV R0, #100        @ R0 = 100 (dividend)
    MOV R1, #5          @ R1 = 5 (divisor)
    UDIV R2, R0, R1     @ R2 = R0 / R1 = 20 (unsigned divide)

    @ === Shift Operations ===
    MOV R0, #1          @ R0 = 1
    LSL R1, R0, #3      @ R1 = R0 << 3 = 8 (Logical Shift Left)
    LSR R2, R1, #1      @ R2 = R1 >> 1 = 4 (Logical Shift Right)
    ASR R3, #-8, #1     @ Arithmetic Shift Right (sign preserved)
    ROR R4, R1, #2      @ R4 = Rotate Right R1 by 2 bits

    @ === Bitwise Operations ===
    MOV R0, #0xFF       @ R0 = 0xFF = 11111111
    MOV R1, #0x0F       @ R1 = 0x0F = 00001111
    AND R2, R0, R1      @ R2 = R0 AND R1 = 0x0F (AND)
    ORR R3, R0, R1      @ R3 = R0 OR R1  = 0xFF (OR)
    EOR R4, R0, R1      @ R4 = R0 XOR R1 = 0xF0 (XOR)
    BIC R5, R0, R1      @ R5 = R0 AND NOT(R1) = 0xF0 (Bit Clear)
    MVN R6, R0          @ R6 = NOT(R0) = 0xFFFFFF00 (Move NOT)

    @ === การเปรียบเทียบ (โดยไม่บันทึกผล) ===
    MOV R0, #10
    MOV R1, #10
    CMP R0, R1          @ R0 - R1, อัพเดท flags แต่ไม่บันทึกผล
    @ ถ้า R0 == R1: Z=1, N=0, C=1, V=0

    @ จบโปรแกรม
    MOV R7, #1          @ sys_exit
    MOV R0, #0          @ exit code 0
    SWI #0
```

### 5.3 การใช้ Conditional Execution ใน ARM32

ARM32 มีฟีเจอร์พิเศษที่ x86 ไม่มี คือทุก instruction สามารถมี condition code ได้

```asm
@ ไฟล์: conditional_arm32.s
@ สาธิต conditional execution ซึ่งเป็นจุดเด่นของ ARM

.section .data
    result_msg: .ascii "Greater!\n"
    result_len = . - result_msg
    equal_msg:  .ascii "Equal!\n"
    equal_len = . - equal_msg

.section .text
    .global _start

_start:
    @ === ตัวอย่าง 1: if-else แบบ ARM ===
    @ เทียบเท่ากับ: if (R0 > R1) { R2 = 1; } else { R2 = 0; }

    MOV R0, #15         @ R0 = 15
    MOV R1, #10         @ R1 = 10
    CMP R0, R1          @ เปรียบเทียบ R0 กับ R1

    @ conditional execution: instruction จะทำงานเฉพาะเมื่อ condition เป็น true
    MOVGT R2, #1        @ ถ้า R0 > R1: R2 = 1  (GT = Greater Than)
    MOVLE R2, #0        @ ถ้า R0 <= R1: R2 = 0 (LE = Less or Equal)

    @ === ตัวอย่าง 2: Loop แบบ ARM ===
    @ นับจาก 0 ถึง 4 (loop 5 ครั้ง)

    MOV R0, #0          @ counter = 0
    MOV R1, #5          @ limit = 5

loop:
    ADD R0, R0, #1      @ counter++
    CMP R0, R1          @ เปรียบเทียบ counter กับ limit
    BNE loop            @ ถ้า counter != limit ให้วนซ้ำ

    @ === ตัวอย่าง 3: Conditional Block (IT block ใน Thumb-2) ===
    @ ใน ARM32 ธรรมดาไม่ต้องใช้ IT block

    MOV R0, #5
    MOV R1, #5
    CMP R0, R1          @ เปรียบเทียบ

    @ ถ้า equal ให้พิมพ์ข้อความ
    BNE not_equal       @ ถ้าไม่เท่ากัน ข้ามไป

    @ พิมพ์ "Equal!"
    MOV R7, #4
    MOV R0, #1
    LDR R1, =equal_msg
    MOV R2, #equal_len
    SWI #0
    B done              @ กระโดดไป done

not_equal:
    @ พิมพ์ "Greater!"
    MOV R7, #4
    MOV R0, #1
    LDR R1, =result_msg
    MOV R2, #result_len
    SWI #0

done:
    MOV R7, #1
    MOV R0, #0
    SWI #0
```

### 5.4 การใช้ Stack และ Function Call ใน ARM32

```asm
@ ไฟล์: functions_arm32.s
@ สาธิตการเรียก function และการใช้ Stack ใน ARM32

.section .data
    newline: .ascii "\n"

.section .text
    .global _start

@ ===== Function: add_numbers =====
@ Input:  R0 = number1, R1 = number2
@ Output: R0 = sum
@ Clobbers: ไม่มี (ใช้แค่ R0, R1, R2)
add_numbers:
    PUSH {R2, LR}       @ บันทึก R2 และ Link Register ลง stack
    @ LR (R14) เก็บ return address ที่ caller ส่งมา

    ADD R2, R0, R1      @ R2 = R0 + R1
    MOV R0, R2          @ ผลลัพธ์ไปใน R0

    POP {R2, PC}        @ คืนค่า R2 และ jump กลับโดย pop ไปยัง PC
    @ การ POP ไปยัง PC เท่ากับ return จาก function

@ ===== Function: factorial =====
@ Input:  R0 = n
@ Output: R0 = n!
@ สาธิต recursive function
factorial:
    PUSH {R4, LR}       @ บันทึก R4 และ LR

    CMP R0, #1          @ ถ้า n <= 1
    BLE factorial_base  @ ไปที่ base case

    MOV R4, R0          @ R4 = n (บันทึกไว้ก่อน recursive call)
    SUB R0, R0, #1      @ R0 = n - 1
    BL factorial        @ recursive call: factorial(n-1)
    @ ตอนนี้ R0 = factorial(n-1)

    MUL R0, R4, R0      @ R0 = n * factorial(n-1)
    B factorial_end

factorial_base:
    MOV R0, #1          @ factorial(0) = factorial(1) = 1

factorial_end:
    POP {R4, PC}        @ คืนค่าและ return

@ ===== Main Program =====
_start:
    @ เรียก add_numbers(15, 25)
    MOV R0, #15
    MOV R1, #25
    BL add_numbers      @ BL = Branch with Link (บันทึก return address ใน LR)
    @ ตอนนี้ R0 = 40

    @ เรียก factorial(5)
    MOV R0, #5
    BL factorial
    @ ตอนนี้ R0 = 120

    @ Exit
    MOV R7, #1
    MOV R0, #0
    SWI #0
```

---

## 6. Thumb Instruction Set

### 6.1 ทำไมต้องมี Thumb?

ARM32 ใช้ instruction ขนาด 32-bit ทุก instruction ซึ่งทำให้:
- Code size ใหญ่ (เปลืองหน่วยความจำ)
- ไม่เหมาะกับ embedded systems ที่มี ROM/Flash จำกัด

Thumb แก้ปัญหาด้วยการใช้ instruction ขนาด **16-bit** ซึ่งลด code size ลงประมาณ 30-40%

### 6.2 Thumb vs ARM32

```
Feature              | ARM32    | Thumb
---------------------|----------|--------
Instruction size     | 32-bit   | 16-bit
Registers accessible | R0-R15   | R0-R7 (mostly)
Conditional execution| ทุก instr| BL, B เท่านั้น (Thumb1)
Code density         | ต่ำ      | สูง (ดีกว่า 30%)
Performance          | สูง      | ต่ำกว่าเล็กน้อย
```

### 6.3 Thumb-2 Extension

Thumb-2 (ARMv6T2 ขึ้นไป) รวมเอา 16-bit และ 32-bit instructions เข้าด้วยกัน:
- ได้ทั้ง code density ของ Thumb
- ได้ทั้ง performance ของ ARM32
- Cortex-M series ใช้ Thumb-2 เป็นหลัก

### 6.4 ตัวอย่าง Thumb Code

```asm
@ ไฟล์: thumb_example.s
@ คอมไพล์: arm-linux-gnueabi-as --thumb -o thumb.o thumb_example.s
@          arm-linux-gnueabi-ld -o thumb thumb.o

.syntax unified    @ ใช้ unified syntax (รองรับทั้ง ARM และ Thumb)
.thumb             @ เปิดใช้ Thumb mode

.section .data
    msg: .ascii "Thumb Mode!\n"
    len = . - msg

.section .text
    .global _start

_start:
    @ Thumb syscall: ใช้ R7 = syscall number เหมือน ARM32
    MOVS R7, #4        @ sys_write (MOVS อัพเดท flags ด้วย)
    MOVS R0, #1        @ stdout
    LDR  R1, =msg      @ pointer
    MOVS R2, #len      @ length
    SVC  #0            @ syscall (Thumb ใช้ SVC แทน SWI)

    MOVS R7, #1        @ sys_exit
    MOVS R0, #0
    SVC  #0
```

### 6.5 IT Block ใน Thumb-2

Thumb-2 ต้องการ IT (If-Then) block สำหรับ conditional execution:

```asm
@ ตัวอย่าง IT block

.syntax unified
.thumb

    CMP R0, R1          @ เปรียบเทียบ
    IT GT               @ If Greater Than
    MOVGT R2, #1        @ ทำงานเฉพาะเมื่อ GT

    @ IT block แบบหลาย instructions (ITTEE = If-Then-Then-Else-Else)
    CMP R0, #0
    ITTEE EQ            @ If Equal: 2 then, 2 else
    MOVEQ R1, #1        @ then (EQ)
    ADDEQ R2, R2, #1   @ then (EQ)
    MOVNE R1, #0        @ else (NE)
    SUBNE R2, R2, #1   @ else (NE)
```

---

## 7. AArch64 (ARM64) Architecture

### 7.1 AArch64 Overview

AArch64 เป็น 64-bit execution state ที่เพิ่มมาใน **ARMv8-A** (2011) มีความแตกต่างจาก ARM32 อย่างมีนัยสำคัญ:

```
การเปลี่ยนแปลงหลัก:
- Registers เพิ่มจาก 16 เป็น 31 general purpose registers
- Register ขนาดเพิ่มจาก 32-bit เป็น 64-bit
- ไม่มี condition code สำหรับทุก instruction (เหมือน ARM32)
- ลบ LDM/STM ออก ใช้ LDP/STP แทน
- ไม่มี IT block สำหรับ Thumb-2 (AArch64 ไม่มี Thumb mode)
- Instruction set เรียบง่ายและสม่ำเสมอมากขึ้น
```

### 7.2 AArch64 Registers

```
Register  | 64-bit | 32-bit | หน้าที่
----------|--------|--------|------------------
X0        | X0     | W0     | Argument 1, Return value
X1        | X1     | W1     | Argument 2
X2        | X2     | W2     | Argument 3
X3        | X3     | W3     | Argument 4
X4        | X4     | W4     | Argument 5
X5        | X5     | W5     | Argument 6
X6        | X6     | W6     | Argument 7
X7        | X7     | W7     | Argument 8
X8        | X8     | W8     | Indirect result, syscall number
X9-X15    | X9-X15 | W9-W15 | Caller-saved (scratch)
X16-X17   | X16-17 | W16-17 | Intra-procedure call scratch
X18       | X18    | W18    | Platform register (อาจใช้เป็น thread pointer)
X19-X28   | X19-28 | W19-28 | Callee-saved
X29       | X29    | W29    | Frame Pointer (FP)
X30       | X30    | W30    | Link Register (LR)
XZR/WZR   | XZR    | WZR    | Zero register (อ่านได้ 0, เขียนทิ้ง)
SP        | SP     | WSP    | Stack Pointer
PC        | PC     | -      | Program Counter (ไม่เข้าถึงตรงๆ)
```

**หมายเหตุสำคัญ:**
- ใช้ `X0-X30` สำหรับ 64-bit operations
- ใช้ `W0-W30` สำหรับ 32-bit operations (lower 32 bits ของ X registers)
- `XZR`/`WZR` = Zero Register ใช้ในหลายบริบท เช่น `MOV X0, XZR` เท่ากับ `MOV X0, #0`

### 7.3 NZCV Flags ใน AArch64

AArch64 ใช้ NZCV flags (เหมือน ARM32 แต่ไม่มี Q, IT, GE bits ใน PSTATE):

```
N = Negative: ผลลัพธ์เป็นลบ (bit 31/63 = 1)
Z = Zero:     ผลลัพธ์เป็น 0
C = Carry:    เกิด carry หรือ borrow
V = oVerflow: เกิด signed overflow
```

**การตั้งค่า flags:**
```asm
@ AArch64 instruction ส่วนใหญ่ไม่อัพเดท flags โดยอัตโนมัติ
@ ต้องใช้ variants ที่ลงท้ายด้วย S

ADDS X0, X1, X2     @ Add และอัพเดท flags
SUBS X0, X1, X2     @ Subtract และอัพเดท flags
ANDS X0, X1, X2     @ AND และอัพเดท flags

CMP X0, X1          @ เทียบเท่ากับ SUBS XZR, X0, X1
CMN X0, X1          @ เทียบเท่ากับ ADDS XZR, X0, X1
TST X0, X1          @ เทียบเท่ากับ ANDS XZR, X0, X1
```

### 7.4 System Registers ใน AArch64

```
PSTATE    = Processor State (แทน CPSR ใน ARM32)
NZCV      = Condition flags (subset ของ PSTATE)
DAIF      = Debug/Abort/IRQ/FIQ disable flags
CurrentEL = Current Exception Level (EL0-EL3)
SPSel     = Stack Pointer Selection
TPIDR_EL0 = Thread ID register (user space)
TPIDR_EL1 = Thread ID register (kernel)
VBAR_EL1  = Vector Base Address Register
MAIR_EL1  = Memory Attribute Indirection Register
TCR_EL1   = Translation Control Register
SCTLR_EL1 = System Control Register
```

**Exception Levels:**
```
EL0 = User space (แอพพลิเคชัน)
EL1 = OS Kernel
EL2 = Hypervisor
EL3 = Secure Monitor (TrustZone)
```

---

## 8. โค้ดตัวอย่าง AArch64

### 8.1 Hello World บน AArch64 Linux

```asm
// ไฟล์: hello_arm64.s
// คอมไพล์: as -o hello_arm64.o hello_arm64.s && ld -o hello_arm64 hello_arm64.o
// รัน: ./hello_arm64
// หมายเหตุ: AArch64 GAS ใช้ // หรือ /* */ สำหรับ comment

.section .data
    msg:     .ascii "Hello, ARM64 World!\n"
    msg_len = . - msg

.section .text
    .global _start

_start:
    // === Write syscall (AArch64 Linux) ===
    // AArch64 syscall convention:
    // X8 = syscall number
    // X0-X7 = arguments

    mov x8, #64         // syscall number 64 = write (AArch64 ต่างจาก ARM32!)
    mov x0, #1          // file descriptor: 1 = stdout
    adr x1, msg         // address ของ string
    mov x2, #msg_len    // จำนวน bytes
    svc #0              // syscall

    // === Exit syscall ===
    mov x8, #93         // syscall number 93 = exit (AArch64)
    mov x0, #0          // exit code = 0
    svc #0
```

**หมายเหตุ syscall numbers:**
```
                  | ARM32  | AArch64
------------------|--------|--------
write             | 4      | 64
read              | 3      | 63
exit              | 1      | 93
open              | 5      | 56
close             | 6      | 57
```

### 8.2 การดำเนินการทางคณิตศาสตร์ AArch64

```asm
// ไฟล์: arithmetic_arm64.s
// สาธิต arithmetic operations ใน AArch64

.section .text
    .global _start

_start:
    // === การบวก ===
    mov x0, #100                // x0 = 100
    mov x1, #200                // x1 = 200
    add x2, x0, x1              // x2 = x0 + x1 = 300
    adds x3, x0, x1             // x3 = x0 + x1 และอัพเดท NZCV flags

    // === การลบ ===
    mov x4, #500
    mov x5, #150
    sub x6, x4, x5              // x6 = x4 - x5 = 350
    subs x7, x4, x5             // x7 = 350 พร้อมอัพเดท flags

    // === การคูณ ===
    mov x0, #12
    mov x1, #13
    mul x2, x0, x1              // x2 = x0 * x1 = 156
    madd x3, x0, x1, xzr        // x3 = x0 * x1 + 0 (MAdd = Multiply-Add)

    // === การหาร ===
    mov x0, #100
    mov x1, #7
    udiv x2, x0, x1             // x2 = 100 / 7 = 14 (unsigned)
    sdiv x3, x0, x1             // x3 = 100 / 7 = 14 (signed)

    // คำนวณ remainder: remainder = dividend - (quotient * divisor)
    msub x4, x2, x1, x0         // x4 = x0 - (x2 * x1) = 100 - 98 = 2

    // === Shift Operations ===
    mov x0, #1
    lsl x1, x0, #10             // x1 = x0 << 10 = 1024
    lsr x2, x1, #2              // x2 = x1 >> 2 = 256 (unsigned)
    asr x3, x1, #2              // x3 = x1 >> 2 = 256 (signed, sign-extended)
    ror x4, x1, #4              // x4 = rotate right 4 bits

    // === Immediate values ===
    // AArch64 immediate ใน MOV มีข้อจำกัด
    // ใช้ MOVZ, MOVK, MOVN สำหรับค่า immediate ขนาดใหญ่

    movz x0, #0x1234            // x0 = 0x0000000000001234
    movk x0, #0x5678, lsl #16  // x0 = 0x0000000056781234
    movk x0, #0x9ABC, lsl #32  // x0 = 0x00009ABC56781234
    movk x0, #0xDEF0, lsl #48  // x0 = 0xDEF09ABC56781234

    // === Logical Operations ===
    mov x0, #0xFF00FF00
    mov x1, #0x0F0F0F0F
    and x2, x0, x1              // AND
    orr x3, x0, x1              // OR
    eor x4, x0, x1              // XOR
    bic x5, x0, x1              // Bit Clear (AND NOT)
    orn x6, x0, x1              // OR NOT
    eon x7, x0, x1              // EOR NOT (XNOR)

    // Exit
    mov x8, #93
    mov x0, #0
    svc #0
```

### 8.3 Function Call Convention ใน AArch64

```asm
// ไฟล์: functions_arm64.s
// สาธิต ARM64 AAPCS64 calling convention

.section .text
    .global _start

// ===== Function: compute_sum =====
// Input:  x0 = array pointer, x1 = count
// Output: x0 = sum
// Callee-saved: x19-x28 (ต้องบันทึกถ้าใช้)
compute_sum:
    stp x19, x20, [sp, #-16]!  // Push x19, x20 ลง stack (pre-indexed)
    // stp = Store Pair, ! หมายถึง update SP

    mov x19, x0                 // x19 = array pointer (บันทึก caller's arg)
    mov x20, x1                 // x20 = count
    mov x0, #0                  // x0 = sum = 0 (accumulator)
    mov x2, #0                  // x2 = index = 0

sum_loop:
    cmp x2, x20                 // เปรียบเทียบ index กับ count
    b.ge sum_done               // ถ้า index >= count ออกจาก loop

    ldr x3, [x19, x2, lsl #3]  // x3 = array[index] (8 bytes per element, lsl #3 = *8)
    add x0, x0, x3              // sum += array[index]
    add x2, x2, #1              // index++
    b sum_loop

sum_done:
    ldp x19, x20, [sp], #16    // Pop x19, x20 จาก stack (post-indexed)
    ret                         // return (jump ไป LR = X30)

// ===== Function: swap =====
// Input:  x0 = pointer to a, x1 = pointer to b
// Output: ค่าที่ x0 และ x1 ชี้ถูก swap
swap:
    ldr x2, [x0]               // x2 = *a
    ldr x3, [x1]               // x3 = *b
    str x3, [x0]               // *a = x3
    str x2, [x1]               // *b = x2
    ret

// ===== Main =====
_start:
    // สร้าง array ใน stack
    sub sp, sp, #64            // จอง 64 bytes (8 x int64)

    // ใส่ค่าลง array
    mov x0, #10
    str x0, [sp]               // array[0] = 10
    mov x0, #20
    str x0, [sp, #8]           // array[1] = 20
    mov x0, #30
    str x0, [sp, #16]          // array[2] = 30
    mov x0, #40
    str x0, [sp, #24]          // array[3] = 40
    mov x0, #50
    str x0, [sp, #32]          // array[4] = 50

    // เรียก compute_sum
    mov x0, sp                 // x0 = pointer to array
    mov x1, #5                 // x1 = count = 5
    bl compute_sum             // call function
    // x0 = 150

    // คืน stack
    add sp, sp, #64

    // Exit
    mov x8, #93
    mov x0, #0
    svc #0
```

### 8.4 Memory Access ใน AArch64

```asm
// ไฟล์: memory_arm64.s
// สาธิต load/store operations ใน AArch64

.section .data
    array:  .quad 1, 2, 3, 4, 5    // 64-bit integer array
    bytes:  .byte 10, 20, 30, 40   // byte array
    words:  .hword 100, 200, 300   // 16-bit array

.section .text
    .global _start

_start:
    // === Load/Store ขนาดต่างๆ ===
    ldr x0, =array              // x0 = address ของ array

    ldrb w1, [x0]              // Load byte (8-bit) zero-extended
    ldrh w2, [x0]              // Load halfword (16-bit) zero-extended
    ldr  w3, [x0]              // Load word (32-bit)
    ldr  x4, [x0]              // Load doubleword (64-bit)

    ldrsb w5, [x0]             // Load byte sign-extended
    ldrsh w6, [x0]             // Load halfword sign-extended
    ldrsw x7, [x0]             // Load word sign-extended to 64-bit

    // === Addressing Modes ===
    // Base Register
    ldr x1, [x0]               // Load จาก address ใน x0

    // Base + Immediate Offset
    ldr x1, [x0, #8]           // Load จาก x0 + 8

    // Base + Register Offset
    mov x2, #3
    ldr x1, [x0, x2, lsl #3]  // Load จาก x0 + (x2 * 8)

    // Pre-indexed (อัพเดท base ก่อน load)
    ldr x1, [x0, #8]!          // x0 = x0 + 8, แล้ว load จาก x0

    // Post-indexed (load ก่อน แล้วอัพเดท base)
    ldr x1, [x0], #8           // load จาก x0, แล้ว x0 = x0 + 8

    // Load Pair (2 registers ในครั้งเดียว)
    ldp x1, x2, [x0]           // x1 = mem[x0], x2 = mem[x0+8]
    ldp x3, x4, [x0, #16]      // x3 = mem[x0+16], x4 = mem[x0+24]

    // === Store Operations ===
    mov x1, #0xFF
    strb w1, [x0]              // Store byte
    strh w1, [x0]              // Store halfword
    str  w1, [x0]              // Store word
    str  x1, [x0]              // Store doubleword
    stp  x1, x2, [x0]         // Store pair

    // Exit
    mov x8, #93
    mov x0, #0
    svc #0
```

---

## 9. Cortex-M4 สำหรับ Embedded Systems

### 9.1 Cortex-M4 Overview

Cortex-M4 เป็น ARM processor ที่ออกแบบสำหรับ embedded systems โดยเฉพาะ:

```
Architecture: ARMv7E-M
ISA: Thumb-2 (ทั้ง 16-bit และ 32-bit)
หน่วยความจำ: ไม่มี cache ทั่วไป (อาจมี TCM)
FPU: Single-precision floating point (optional)
DSP: DSP instruction extensions
Nested Vectored Interrupt Controller (NVIC)
SysTick timer
ใช้ใน: STM32F4, LPC43xx, Kinetis K-series
```

### 9.2 Memory Map ของ Cortex-M

```
Address Range       | Region
--------------------|------------------
0x00000000-0x1FFFFFFF| Code (512MB)
0x20000000-0x3FFFFFFF| SRAM (512MB)
0x40000000-0x5FFFFFFF| Peripherals (512MB)
0x60000000-0x9FFFFFFF| External RAM (1GB)
0xA0000000-0xDFFFFFFF| External Device (1GB)
0xE0000000-0xFFFFFFFF| System (512MB)
  0xE000E000-0xE000EFFF| System Control Space (SCS)
    0xE000E010| SysTick
    0xE000E100| NVIC
    0xE000ED00| SCB (System Control Block)
```

### 9.3 โค้ดตัวอย่าง Cortex-M4 (Bare Metal)

```asm
@ ไฟล์: stm32f4_blink.s
@ Toggle LED บน STM32F4 Discovery board
@ คอมไพล์: arm-none-eabi-as -mcpu=cortex-m4 -mthumb -o blink.o stm32f4_blink.s
@          arm-none-eabi-ld -T stm32f4.ld -o blink.elf blink.o
@          arm-none-eabi-objcopy -O binary blink.elf blink.bin

.syntax unified
.cpu cortex-m4
.thumb

@ ===== Peripheral Base Addresses =====
.equ RCC_BASE,    0x40023800   @ Reset and Clock Control
.equ GPIOD_BASE,  0x40020C00   @ GPIO Port D base address

@ RCC Registers
.equ RCC_AHB1ENR, 0x40023830   @ AHB1 peripheral clock enable

@ GPIOD Registers
.equ GPIOD_MODER, 0x40020C00   @ Mode register
.equ GPIOD_ODR,   0x40020C14   @ Output data register

@ Bit definitions
.equ GPIODEN,     (1 << 3)     @ Enable GPIOD clock (bit 3 ของ AHB1ENR)
.equ PIN12_MODE,  (1 << 24)    @ Set pin 12 as output (bits 24-25 ของ MODER)
.equ PIN12_OUT,   (1 << 12)    @ Pin 12 output bit ใน ODR

.section .text
    .global Reset_Handler

@ Vector Table
.section .vectors
    .word 0x20020000    @ Initial Stack Pointer (end of SRAM)
    .word Reset_Handler @ Reset handler address

.section .text

@ ===== Delay Function =====
@ Input: R0 = delay count
delay:
    PUSH {R1, LR}
delay_loop:
    SUBS R0, R0, #1     @ R0-- และอัพเดท flags
    BNE delay_loop      @ วนซ้ำถ้าไม่เป็น 0
    POP {R1, PC}

@ ===== Main Reset Handler =====
Reset_Handler:
    @ --- Enable GPIOD clock ---
    LDR R0, =RCC_AHB1ENR    @ โหลด address ของ RCC_AHB1ENR
    LDR R1, [R0]             @ อ่านค่าปัจจุบัน
    ORR R1, R1, #GPIODEN     @ เปิด bit สำหรับ GPIOD
    STR R1, [R0]             @ เขียนกลับ

    @ --- Configure GPIOD Pin 12 as output ---
    LDR R0, =GPIOD_MODER    @ โหลด address ของ MODER
    LDR R1, [R0]             @ อ่านค่าปัจจุบัน
    ORR R1, R1, #PIN12_MODE  @ ตั้งค่า pin 12 เป็น output (01)
    BIC R1, R1, #(1 << 25)  @ clear bit 25 (ให้เป็น 01 ไม่ใช่ 11)
    STR R1, [R0]             @ เขียนกลับ

    @ --- Main LED Blink Loop ---
blink_loop:
    @ Turn ON LED (Pin 12 HIGH)
    LDR R0, =GPIOD_ODR
    LDR R1, [R0]
    ORR R1, R1, #PIN12_OUT  @ Set bit 12
    STR R1, [R0]

    @ Delay
    LDR R0, =500000         @ delay ~500ms
    BL delay

    @ Turn OFF LED (Pin 12 LOW)
    LDR R0, =GPIOD_ODR
    LDR R1, [R0]
    BIC R1, R1, #PIN12_OUT  @ Clear bit 12
    STR R1, [R0]

    @ Delay
    LDR R0, =500000
    BL delay

    B blink_loop            @ วนซ้ำตลอด
```

### 9.4 Exception Handling ใน Cortex-M

```asm
@ ไฟล์: cortex_m_exceptions.s
@ สาธิต exception/interrupt handling

.syntax unified
.thumb

@ Exception Numbers:
@ 0 = Initial SP
@ 1 = Reset
@ 2 = NMI
@ 3 = HardFault
@ 4 = MemManage
@ 5 = BusFault
@ 6 = UsageFault
@ 11 = SVCall
@ 14 = PendSV
@ 15 = SysTick
@ 16+ = External Interrupts (NVIC)

.section .vectors
    .word 0x20020000        @ Initial SP
    .word Reset_Handler
    .word NMI_Handler
    .word HardFault_Handler
    .word MemManage_Handler
    .word BusFault_Handler
    .word UsageFault_Handler
    .word 0                 @ Reserved
    .word 0
    .word 0
    .word 0
    .word SVC_Handler
    .word 0                 @ Reserved
    .word 0
    .word PendSV_Handler
    .word SysTick_Handler

.section .text

@ Reset Handler
Reset_Handler:
    B main_init

@ NMI Handler
NMI_Handler:
    BX LR               @ Return immediately

@ HardFault Handler (สำหรับ debug)
HardFault_Handler:
    @ อ่าน fault status registers
    LDR R0, =0xE000ED28     @ CFSR (Configurable Fault Status)
    LDR R1, [R0]            @ อ่าน fault status
    LDR R0, =0xE000ED2C     @ HFSR (HardFault Status)
    LDR R2, [R0]            @ อ่าน hard fault status
fault_loop:
    B fault_loop        @ ค้างอยู่ที่นี่ (สำหรับ debugger)

@ SysTick Handler (เรียกทุก 1ms ถ้า configure ไว้)
SysTick_Handler:
    PUSH {LR}
    @ ทำ task ที่ต้องการทุก 1ms
    @ เช่น: เพิ่ม tick counter
    LDR R0, =system_ticks
    LDR R1, [R0]
    ADD R1, R1, #1
    STR R1, [R0]
    POP {PC}

@ SVC Handler
SVC_Handler:
    @ อ่าน SVC number จาก instruction
    TST LR, #4              @ ตรวจสอบ stack (MSP or PSP)
    ITE EQ
    MRSEQ R0, MSP           @ ถ้า MSP: R0 = MSP
    MRSNE R0, PSP           @ ถ้า PSP: R0 = PSP
    LDR R1, [R0, #24]       @ PC ของ caller
    LDRB R0, [R1, #-2]      @ อ่าน SVC number จาก instruction
    @ ตอนนี้ R0 = SVC number
    BX LR

@ Main initialization
main_init:
    @ Configure SysTick สำหรับ 1ms interrupt ที่ 168MHz
    LDR R0, =0xE000E010     @ SysTick base
    LDR R1, =168000         @ 168000 cycles = 1ms @ 168MHz
    STR R1, [R0, #4]        @ SysTick LOAD
    MOV R1, #7              @ Enable, Interrupt enable, Use processor clock
    STR R1, [R0]            @ SysTick CTRL

    B main_loop

main_loop:
    WFI                     @ Wait For Interrupt (ประหยัดพลังงาน)
    B main_loop

.section .bss
    system_ticks: .word 0   @ tick counter
```

---

## 10. ARM vs x86 การเปรียบเทียบ

### 10.1 ความแตกต่างพื้นฐาน

| Feature | ARM | x86/x86-64 |
|---------|-----|------------|
| ISA Type | RISC | CISC |
| Instruction Size | Fixed (ARM32: 32-bit) | Variable (1-15 bytes) |
| Registers | ARM32: 16, AArch64: 31+3 | x86: 8, x86-64: 16 |
| Memory Access | Load/Store only | Any instruction |
| Endianness | BI-endian (default LE) | Little-endian only |
| Condition codes | ทุก instruction (ARM32) | เฉพาะ comparison |
| Power consumption | ต่ำมาก | สูงกว่า |
| Performance/Watt | สูง | ต่ำกว่า |
| Code density | สูง (Thumb) | สูงมาก |
| Floating point | VFP/NEON | x87/SSE/AVX |

### 10.2 Register Comparison

```
x86-64 Register | ขนาด | ARM64 Equivalent
----------------|------|------------------
RAX             | 64   | X0 (return value)
RBX             | 64   | X19 (callee-saved)
RCX             | 64   | X3 (4th argument)
RDX             | 64   | X2 (3rd argument)
RSI             | 64   | X1 (2nd argument)
RDI             | 64   | X0 (1st argument)
RSP             | 64   | SP
RBP             | 64   | X29 (Frame Pointer)
RIP             | 64   | PC (ไม่เข้าถึงตรงๆ)
R8-R15          | 64   | X4-X7 (args), X8-X15
RFLAGS          | 64   | PSTATE/NZCV
```

### 10.3 Instruction Comparison

```asm
@ === การบวก ===
; x86-64                    @ ARM64
add rax, rbx                // add x0, x0, x1

@ === การ Load จาก memory ===
; x86-64                    @ ARM64
mov rax, [rbx]              // ldr x0, [x1]
mov rax, [rbx + 8]          // ldr x0, [x1, #8]
mov rax, [rbx + rcx*8]      // ldr x0, [x1, x2, lsl #3]

@ === Conditional Jump ===
; x86-64                    @ ARM64
cmp rax, rbx                // cmp x0, x1
je  equal_label             // b.eq equal_label
jl  less_label              // b.lt less_label
jg  greater_label           // b.gt greater_label

@ === Function Call ===
; x86-64                    @ ARM64
call function               // bl function
ret                         // ret

@ === Stack Operations ===
; x86-64                    @ ARM64
push rbx                    // str x19, [sp, #-8]! หรือ stp x19, x20, [sp, #-16]!
pop  rbx                    // ldr x19, [sp], #8  หรือ ldp x19, x20, [sp], #16
```

### 10.4 Calling Convention Comparison

```
                | x86-64 Linux (SysV ABI) | AArch64 (AAPCS64)
----------------|------------------------|--------------------
1st arg         | RDI                    | X0
2nd arg         | RSI                    | X1
3rd arg         | RDX                    | X2
4th arg         | RCX                    | X3
5th arg         | R8                     | X4
6th arg         | R9                     | X5
7th+ args       | Stack                  | Stack
Return value    | RAX                    | X0
Callee-saved    | RBX, RBP, R12-R15      | X19-X28, X29 (FP)
Caller-saved    | RAX, RCX, RDX, RSI,    | X0-X18
                | RDI, R8-R11            |
Stack alignment | 16 bytes               | 16 bytes
```

---

## 11. ARM Everywhere: การใช้งานจริง

### 11.1 Mobile: Smartphone และ Tablet

```
Apple:
  A4  (2010) = Cortex-A8 @ 1GHz    → iPhone 4
  A5  (2011) = Cortex-A9 @ 1GHz    → iPhone 4S
  A6  (2012) = Swift @ 1.3GHz      → iPhone 5 (custom ARM core)
  A7  (2013) = Cyclone @ 1.3GHz    → iPhone 5s (AArch64, ตัวแรก!)
  A14 (2020) = Firestorm/Icestorm  → iPhone 12 (5nm)
  A17 Pro (2023) = 3nm             → iPhone 15 Pro

Qualcomm Snapdragon:
  SD888  → Cortex-X1 @ 2.84GHz + A78 + A55
  SD8 Gen 3 → Cortex-X4 @ 3.3GHz + A720 + A520
```

### 11.2 Server: Cloud Computing

```
AWS Graviton:
  Graviton  (2018) = custom ARMv8.0 @ 2.3GHz, 16 cores/socket
  Graviton2 (2020) = custom ARMv8.2 @ 2.5GHz, 64 cores/socket, Neoverse N1
  Graviton3 (2021) = custom ARMv8.4 @ 2.6GHz, 64 cores/socket, Neoverse V1
  Graviton4 (2023) = custom ARMv9.0 @ 2.8GHz, 96 cores/socket, Neoverse V2

Ampere Computing:
  Altra (2021)     = Neoverse N1, 128 cores
  Altra Max        = 192 cores

Apple Silicon for Mac:
  M1 (2020) = 8 cores (4P+4E), เร็วกว่า Intel i9 ในหลาย workload
  M2 Pro (2023) = 12 cores (8P+4E)
  M3 Ultra (2024) = 32 cores (24P+8E)
```

### 11.3 Embedded: IoT และ Microcontroller

```
STM32 Family (STMicroelectronics):
  STM32F0 = Cortex-M0  @ 48MHz,  4-256KB Flash
  STM32F1 = Cortex-M3  @ 72MHz,  16-1MB Flash
  STM32F4 = Cortex-M4F @ 168MHz, 128KB-2MB Flash
  STM32H7 = Cortex-M7  @ 480MHz, 1-2MB Flash

Nordic Semiconductor:
  nRF52840 = Cortex-M4F @ 64MHz (Bluetooth 5.0)
  nRF9160  = Cortex-M33 @ 64MHz (LTE-M/NB-IoT)

Raspberry Pi (SBC):
  Pi Zero W  = ARM11 (ARMv6) @ 1GHz
  Pi 3B+     = Cortex-A53 @ 1.4GHz (ARMv8-A, 64-bit)
  Pi 4B      = Cortex-A72 @ 1.8GHz
  Pi 5       = Cortex-A76 @ 2.4GHz
```

---

## 12. ข้อผิดพลาดที่พบบ่อย (Common Mistakes)

### 12.1 ARM32 Mistakes

```asm
@ ผิด: MUL destination ต้องไม่เหมือนกับ source ใน ARMv5 และเก่ากว่า
MUL R0, R0, R1     @ UNPREDICTABLE ใน ARMv5!

@ ถูก:
MUL R2, R0, R1     @ destination (R2) ต่างจาก source

@ ผิด: ลืมว่า PC (R15) มีค่าเป็น address + 8 ใน ARM32
MOV R0, PC         @ R0 = address ของ instruction นี้ + 8 (ไม่ใช่ +0)

@ ผิด: ใช้ instruction ที่ไม่รองรับใน mode นั้น
@ ARM32 instruction ใน Thumb mode → จะเกิด HardFault

@ ผิด: ลืม update flags ก่อนใช้ conditional branch
ADD R0, R0, #1     @ ไม่อัพเดท flags!
BEQ somewhere      @ ใช้ flags จาก instruction ก่อนหน้านั้น

@ ถูก:
ADDS R0, R0, #1    @ อัพเดท flags (S suffix)
BEQ  somewhere     @ ตอนนี้ใช้ flags ที่ถูกต้อง
```

### 12.2 AArch64 Mistakes

```asm
// ผิด: ลืมว่า AArch64 ไม่มี conditional execution สำหรับทุก instruction
// ใช้ conditional select แทน
ADDEQ x0, x1, x2   // ไม่มีใน AArch64! → Error

// ถูก: ใช้ CSEL (Conditional Select)
CMP x3, #0
CSEL x0, x1, x2, eq   // x0 = (x3 == 0) ? x1 : x2

// ผิด: ลืม alignment สำหรับ LDP/STP
// address ต้อง aligned กับขนาดของ register pair
LDP x0, x1, [x2]   // x2 ต้อง aligned 8 bytes (ใช้ได้)
LDP x0, x1, [x2, #4]  // ผิด! ต้อง aligned 8 bytes

// ผิด: ใช้ W register แต่คิดว่าจะ extend เป็น 64-bit
// W register เขียนทับเฉพาะ lower 32 bits, zero-extends ไปยัง upper
MOV w0, #-1        // x0 = 0x00000000FFFFFFFF (ไม่ใช่ 0xFFFFFFFFFFFFFFFF)
MOV x0, #-1        // x0 = 0xFFFFFFFFFFFFFFFF (ถูก)

// ผิด: ลืม stack alignment requirement
SUB sp, sp, #8     // stack ต้อง 16-byte aligned ตอน call function!
BL some_function   // → อาจเกิด SIGBUS หรือ undefined behavior

// ถูก:
SUB sp, sp, #16    // จอง 16 bytes (aligned)
BL some_function
ADD sp, sp, #16
```

### 12.3 Cortex-M Mistakes

```asm
@ ผิด: ลืม THUMB mode ใน Vector Table
.section .vectors
    .word Reset_Handler   @ ผิด! ต้องมี Thumb bit set

@ ถูก: ใช้ thumb_func attribute
.thumb_func
Reset_Handler:           @ assembler จะ set Thumb bit โดยอัตโนมัติ

@ หรือ: ใช้ .word (Reset_Handler + 1) ใน vector table
.word Reset_Handler + 1  @ Thumb bit set

@ ผิด: ลืม NOP padding สำหรับ pipeline flush
CPSIE I              @ Enable interrupts
@ interrupt อาจ trigger ก่อน pipeline flush
LDR R0, [R1]        @ อาจมีปัญหา

@ ถูก: เพิ่ม ISB (Instruction Synchronization Barrier)
CPSIE I
ISB                  @ flush pipeline
LDR R0, [R1]

@ ผิด: เข้าถึง memory ที่ไม่ได้ enable clock ก่อน
STR R1, [R0]         @ เขียนไปยัง GPIOD ก่อน enable clock → HardFault

@ ถูก: Enable clock ก่อนเสมอ
LDR R2, =RCC_AHB1ENR
LDR R3, [R2]
ORR R3, R3, #GPIODEN
STR R3, [R2]
@ รอ clock stable
NOP
NOP
STR R1, [R0]         @ ตอนนี้ปลอดภัย
```

---

## 13. แบบฝึกหัดปฏิบัติ (Practical Exercises)

### แบบฝึกหัดที่ 1: ARM32 Calculator

```asm
@ แบบฝึกหัด: เขียน calculator สำหรับ ARM32 ที่รองรับ +, -, *, /
@ บันทึกผลใน memory array

.section .data
    num1:   .word 100
    num2:   .word 7
    results: .space 16   @ 4 results x 4 bytes

.section .text
    .global _start

_start:
    @ โหลด operands
    LDR R4, =num1
    LDR R0, [R4]         @ R0 = num1
    LDR R4, =num2
    LDR R1, [R4]         @ R1 = num2

    @ คำนวณ
    ADD R2, R0, R1        @ R2 = num1 + num2
    SUB R3, R0, R1        @ R3 = num1 - num2

    @ บันทึก addition และ subtraction
    LDR R4, =results
    STR R2, [R4]          @ results[0] = sum
    STR R3, [R4, #4]      @ results[1] = difference

    @ คูณ
    MUL R2, R0, R1        @ R2 = num1 * num2
    STR R2, [R4, #8]      @ results[2] = product

    @ หาร (ใช้ UDIV ถ้า ARMv7+)
    UDIV R3, R0, R1       @ R3 = num1 / num2
    STR R3, [R4, #12]     @ results[3] = quotient

    @ คำนวณ remainder
    MUL R5, R3, R1        @ R5 = quotient * num2
    SUB R5, R0, R5        @ R5 = num1 - (quotient * num2) = remainder

    @ Exit
    MOV R7, #1
    MOV R0, #0
    SWI #0
```

**โจทย์:** แก้ไขโปรแกรมด้านบนให้:
1. แสดงผลลัพธ์ทั้ง 4 การดำเนินการบนหน้าจอ
2. รองรับ negative numbers ด้วย SDIV
3. ตรวจสอบ division by zero

### แบบฝึกหัดที่ 2: ARM64 String Operations

```asm
// แบบฝึกหัด: เขียน string length function สำหรับ AArch64

.section .data
    test_str: .asciz "Hello, ARM64!"   // null-terminated string

.section .text
    .global _start

// === strlen function ===
// Input:  x0 = pointer to string
// Output: x0 = string length
my_strlen:
    mov x1, x0          // x1 = pointer (สำรอง x0)
strlen_loop:
    ldrb w2, [x1], #1   // Load byte แล้ว x1++
    cbnz w2, strlen_loop // ถ้าไม่ใช่ null ให้วนซ้ำ
    sub x0, x1, x0      // length = end_ptr - start_ptr
    sub x0, x0, #1      // ลบ null terminator ออก
    ret

_start:
    ldr x0, =test_str
    bl my_strlen
    // x0 = 13 (ความยาวของ "Hello, ARM64!")

    mov x8, #93
    mov x0, #0
    svc #0
```

**โจทย์ขยาย:** เพิ่ม functions:
1. `my_strcpy(dst, src)` - copy string
2. `my_strcmp(s1, s2)` - compare strings (return 0, -1, 1)
3. `my_strcat(dst, src)` - concatenate strings

### แบบฝึกหัดที่ 3: Fibonacci ด้วย ARM32

```asm
@ แบบฝึกหัด: คำนวณ Fibonacci sequence และเก็บใน array

.section .data
    fib_array: .space 40    @ 10 numbers x 4 bytes each

.section .text
    .global _start

_start:
    LDR R4, =fib_array  @ R4 = pointer to array

    @ ค่า seed
    MOV R0, #0          @ fib[0] = 0
    MOV R1, #1          @ fib[1] = 1
    STR R0, [R4]        @ เก็บ fib[0]
    STR R1, [R4, #4]    @ เก็บ fib[1]

    MOV R5, #2          @ index = 2
    MOV R6, #10         @ จะคำนวณ 10 ตัว

fib_loop:
    CMP R5, R6
    BGE fib_done

    ADD R2, R0, R1      @ fib[i] = fib[i-2] + fib[i-1]
    MOV R0, R1          @ อัพเดทค่าก่อนหน้า
    MOV R1, R2

    @ เก็บค่าลง array
    LSL R7, R5, #2      @ offset = index * 4
    STR R2, [R4, R7]

    ADD R5, R5, #1      @ index++
    B fib_loop

fib_done:
    @ ผลลัพธ์: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34

    MOV R7, #1
    MOV R0, #0
    SWI #0
```

**โจทย์ขยาย:** แปลง Fibonacci เป็น version ที่แสดงผลเป็น decimal บนหน้าจอ

### แบบฝึกหัดที่ 4: ARM64 SIMD/NEON (Advanced)

```asm
// แบบฝึกหัด: ใช้ NEON/Advanced SIMD เพิ่ม vectors

.section .data
    .align 4
    vec_a:  .float 1.0, 2.0, 3.0, 4.0   // 4 x float
    vec_b:  .float 5.0, 6.0, 7.0, 8.0
    vec_c:  .space 16                    // result

.section .text
    .global _start

_start:
    // โหลด vectors ด้วย NEON
    ldr x0, =vec_a
    ldr x1, =vec_b
    ldr x2, =vec_c

    ld1 {v0.4s}, [x0]       // โหลด vec_a ลง v0 (4 x 32-bit float)
    ld1 {v1.4s}, [x1]       // โหลด vec_b ลง v1

    // Vector addition
    fadd v2.4s, v0.4s, v1.4s // v2 = v0 + v1 (element-wise)

    // บันทึกผล
    st1 {v2.4s}, [x2]       // เก็บ v2 ลง vec_c

    // ผลลัพธ์: vec_c = {6.0, 8.0, 10.0, 12.0}

    mov x8, #93
    mov x0, #0
    svc #0
```

**โจทย์ขยาย:** เพิ่ม operations:
1. Vector dot product (FMLA)
2. Vector absolute value (FABS)
3. Vector max (FMAX)

### แบบฝึกหัดที่ 5: System Register Access (AArch64)

```asm
// แบบฝึกหัด: อ่าน system registers ใน AArch64

.section .text
    .global _start

_start:
    // อ่าน Exception Level ปัจจุบัน
    mrs x0, CurrentEL       // x0 = CurrentEL
    lsr x0, x0, #2          // shift right 2 เพื่อได้ EL number
    and x0, x0, #3          // mask ให้เหลือ 2 bits
    // x0 = 0 (EL0 = user mode สำหรับ normal process)

    // อ่าน Thread ID (สำหรับ thread-local storage)
    mrs x1, tpidr_el0       // x1 = thread pointer

    // อ่าน CPSR (ใน AArch64 เรียกว่า NZCV)
    mrs x2, nzcv            // x2 = NZCV flags

    // === Counter registers (ต้องมี permission ใน EL1) ===
    // mrs x3, cntpct_el0   // Physical Counter (timer)
    // mrs x4, cntvct_el0   // Virtual Counter

    mov x8, #93
    mov x0, #0
    svc #0
```

---

## 14. สรุปและ Key Takeaways

### สรุปสำคัญ Part 007

1. **ARM เป็น RISC processor** ที่มีข้อดีด้าน power efficiency สูงกว่า x86 มาก ทำให้ครองตลาด mobile และ embedded

2. **Architecture Profiles:** Profile-A สำหรับ application (MMU, Linux), Profile-R สำหรับ realtime, Profile-M สำหรับ microcontroller

3. **ARM32 vs AArch64:**
   - ARM32: 16 registers (R0-R15), conditional execution ทุก instruction
   - AArch64: 31 general registers (X0-X30) + XZR + SP + PC, ไม่มี conditional execution สำหรับทุก instruction

4. **Thumb/Thumb-2:** ลด code size 30-40%, Thumb-2 ผสม 16/32-bit instructions

5. **CPSR/PSTATE:** flags N, Z, C, V ใช้ร่วมกันทั้ง ARM32 และ AArch64 (NZCV)

6. **Cortex-M สำหรับ embedded:** ไม่มี MMU, ใช้ Thumb-2, มี NVIC สำหรับ interrupt

7. **ARM vs x86:** ARM ใช้ Load/Store architecture (memory access เฉพาะ LDR/STR), x86 ทำ computation กับ memory ได้โดยตรง

### ตาราง Cheat Sheet

```
Operation      | ARM32          | AArch64        | x86-64
---------------|----------------|----------------|----------
Load word      | LDR R0, [R1]   | LDR W0, [X1]   | mov eax, [rbx]
Load dword     | LDRD R0,R1,[R2]| LDR X0, [X1]   | mov rax, [rbx]
Store word     | STR R0, [R1]   | STR W0, [X1]   | mov [rbx], eax
Add            | ADD R0,R1,R2   | ADD X0,X1,X2   | add rax, rbx
Call function  | BL func        | BL func        | call func
Return         | BX LR / POP PC | RET            | ret
Syscall        | SWI/SVC #0     | SVC #0         | syscall
```

---

## 15. แหล่งข้อมูลเพิ่มเติม (Resources)

### เอกสารทางการ

1. **ARM Architecture Reference Manual** - เอกสารหลักสำหรับ ARMv7-A/R และ ARMv8-A/AArch64
   - https://developer.arm.com/documentation/

2. **ARM Cortex-M4 Technical Reference Manual**
   - https://developer.arm.com/documentation/100166/latest/

3. **AAPCS64 Procedure Call Standard**
   - https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst

### เครื่องมือและ Development

```bash
# ติดตั้ง ARM toolchain บน Ubuntu/Debian
sudo apt-get install gcc-arm-linux-gnueabihf    # ARM32 cross-compiler
sudo apt-get install gcc-aarch64-linux-gnu      # ARM64 cross-compiler
sudo apt-get install binutils-arm-linux-gnueabi # ARM32 assembler

# Bare metal (Cortex-M)
sudo apt-get install gcc-arm-none-eabi          # ARM none-eabi toolchain

# QEMU สำหรับ emulation
sudo apt-get install qemu-user               # User-mode emulation
sudo apt-get install qemu-system-arm        # System-mode emulation

# Assembling ARM32
arm-linux-gnueabihf-as -o output.o input.s
arm-linux-gnueabihf-ld -o output input.o

# Assembling AArch64
aarch64-linux-gnu-as -o output.o input.s
aarch64-linux-gnu-ld -o output input.o

# รัน ARM binary บน x86 ด้วย QEMU
qemu-arm-static ./arm32_binary
qemu-aarch64-static ./arm64_binary
```

### Online Resources

```
ARM Developer Documentation: https://developer.arm.com/
Godbolt Compiler Explorer:   https://godbolt.org/  (ใส่ ARM target)
ARM Instruction Reference:   https://developer.arm.com/documentation/ddi0596/
NEON Intrinsics Guide:       https://developer.arm.com/architectures/instruction-sets/intrinsics/
```

### หนังสือแนะนำ

1. **"ARM System Developer's Guide"** - Andrew Sloss, Dominic Symes, Chris Wright
2. **"Programming with 64-Bit ARM Assembly Language"** - Stephen Smith
3. **"The Definitive Guide to ARM Cortex-M3 and M4"** - Joseph Yiu

---

## Preview: Part 008

ใน **Part 008** เราจะเรียนรู้:
- **ARM32 Data Processing Instructions** ขั้นสูง
- **Barrel Shifter** ในตัว ARM32 (การ shift/rotate ภายใน instruction)
- **Multiply and Accumulate** instructions (MLA, MLS, UMULL, SMULL)
- **Saturating arithmetic** สำหรับ DSP
- **NEON (Advanced SIMD)** เบื้องต้นสำหรับ ARM32
- **โปรเจกต์จริง:** เขียน image processing routine ด้วย NEON

---

*Part 007 จบแล้ว - ARM Architecture เบื้องต้น*  
*ทำแบบฝึกหัดทั้ง 5 ข้อก่อนไปยัง Part 008*
