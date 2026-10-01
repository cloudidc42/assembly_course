# Part 035: ARM Stack และ Subroutines

## ARM Stack Management and Calling Conventions

---

## สารบัญ (Table of Contents)

1. [ARM Stack พื้นฐาน](#arm-stack-พื้นฐาน)
2. [Full Descending Stack](#full-descending-stack)
3. [STMFD/LDMFD และ PUSH/POP](#stmfdldmfd-และ-pushpop)
4. [AAPCS Calling Convention](#aapcs-calling-convention)
5. [Stack Frame Setup](#stack-frame-setup)
6. [Nested Function Calls](#nested-function-calls)
7. [Local Variables on Stack](#local-variables-on-stack)
8. [VFP Push/Pop](#vfp-pushpop)
9. [ARM PCS - Procedure Call Standard](#arm-pcs---procedure-call-standard)
10. [Parameter Passing Beyond R3](#parameter-passing-beyond-r3)
11. [Large Struct Passing](#large-struct-passing)
12. [Return Struct in Memory](#return-struct-in-memory)
13. [AArch64 Stack และ Calling Convention](#aarch64-stack-และ-calling-convention)
14. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ARM Stack พื้นฐาน

### Stack คืออะไร?

Stack เป็นโครงสร้างข้อมูลแบบ LIFO (Last In, First Out) ที่ใช้สำหรับ:
- เก็บค่า registers ชั่วคราว
- ส่งผ่าน parameters ไปยัง functions
- เก็บ local variables
- เก็บ return address

```
Memory Layout:
High Address  ┌─────────────────┐
              │   Stack         │  ← SP เริ่มต้นที่นี่
              │   ↓ grows down  │
              │                 │
              │   (free space)  │
              │                 │
              │   ↑ grows up    │
              │   Heap          │
              ├─────────────────┤
              │   BSS           │
              ├─────────────────┤
              │   Data          │
              ├─────────────────┤
              │   Text (Code)   │
Low Address   └─────────────────┘
```

### ARM Stack ใน 4 รูปแบบ

ARM รองรับ stack 4 แบบ แต่ที่ใช้บ่อยที่สุดคือ **Full Descending (FD)**:

| ชื่อ | Abbreviation | คำอธิบาย |
|------|-------------|----------|
| Full Descending | FD | SP ชี้ที่ข้อมูลล่าสุด, stack เติบโตลงมา |
| Full Ascending | FA | SP ชี้ที่ข้อมูลล่าสุด, stack เติบโตขึ้นไป |
| Empty Descending | ED | SP ชี้ที่ช่องว่าง, stack เติบโตลงมา |
| Empty Ascending | EA | SP ชี้ที่ช่องว่าง, stack เติบโตขึ้นไป |

```
Full Descending Stack (ที่ ARM Linux ใช้):

ก่อน PUSH:          หลัง PUSH R0:
                    
High  ┌────────┐    High  ┌────────┐
      │ data1  │ ← SP          │ data1  │
      │ data2  │          │ data2  │
Low   └────────┘    Low   ├────────┤
                          │  R0    │ ← SP (ลดลง 4 bytes)
                          └────────┘
```

---

## Full Descending Stack

### การทำงานของ Full Descending Stack

ใน ARM Linux (AAPCS), stack เป็นแบบ **Full Descending**:
- SP (R13) ชี้ไปยัง **ข้อมูลล่าสุดที่ push** (Full)
- Stack **เติบโตไปทางที่อยู่ต่ำกว่า** (Descending)

```asm
@ ============================================================
@ full_descending_demo.s
@ สาธิตการทำงานของ Full Descending Stack
@ ============================================================
.section .text
.global _start

_start:
    @ สมมติว่า SP = 0x1000 (เริ่มต้น)
    
    @ PUSH R0 - SP ลดลงก่อน แล้วจึง store
    @ เทียบเท่า:
    @   SUB SP, SP, #4    @ SP = 0x0FFC
    @   STR R0, [SP]      @ เก็บ R0 ที่ 0x0FFC
    
    MOV R0, #100        @ R0 = 100
    MOV R1, #200        @ R1 = 200
    MOV R2, #300        @ R2 = 300
    
    @ PUSH หลายตัวพร้อมกัน (เรียงจาก high register ก่อน)
    PUSH {R0, R1, R2}   @ push R2 ก่อน, แล้ว R1, แล้ว R0
    
    @ Stack ตอนนี้:
    @ SP+0: R0 (100)
    @ SP+4: R1 (200)
    @ SP+8: R2 (300)
    
    MOV R0, #0          @ เปลี่ยนค่า
    MOV R1, #0
    MOV R2, #0
    
    @ POP คืนค่า
    POP {R0, R1, R2}    @ pop R0 ก่อน, แล้ว R1, แล้ว R2
    
    @ R0=100, R1=200, R2=300 (ค่าเดิม)
    
    @ จบโปรแกรม
    MOV R7, #1          @ syscall exit
    SWI #0
```

### STMFD และ LDMFD (รูปแบบเก่า)

```asm
@ ============================================================
@ stmfd_ldmfd_demo.s
@ สาธิต STMFD/LDMFD vs PUSH/POP
@ ============================================================
.section .text
.global _start

_start:
    @ ===== รูปแบบเก่า (ARM classic) =====
    @ STMFD = Store Multiple, Full Descending
    @ LDMFD = Load Multiple, Full Descending
    
    MOV R0, #10
    MOV R1, #20
    MOV R2, #30
    MOV R3, #40
    
    @ STMFD SP!, {R0-R3} เทียบเท่า PUSH {R0-R3}
    @ ! หมายถึง write-back (อัปเดต SP)
    STMFD SP!, {R0-R3}      @ เก็บ R0-R3 ลง stack
    
    MOV R0, #0
    MOV R1, #0
    MOV R2, #0
    MOV R3, #0
    
    @ LDMFD SP!, {R0-R3} เทียบเท่า POP {R0-R3}
    LDMFD SP!, {R0-R3}      @ โหลด R0-R3 จาก stack
    
    @ ===== รูปแบบใหม่ (Thumb2/ARMv7) =====
    @ PUSH เทียบเท่ากับ STMFD SP!
    @ POP เทียบเท่ากับ LDMFD SP!
    
    PUSH {R0-R3}            @ เก็บ R0-R3
    POP  {R0-R3}            @ โหลด R0-R3
    
    @ ===== ตารางเปรียบเทียบ =====
    @ STMFD SP!, {regs}  =  PUSH {regs}
    @ LDMFD SP!, {regs}  =  POP {regs}
    @ STMIA SP!, {regs}  =  (stack ascending, ไม่ค่อยใช้)
    @ STMDB SP!, {regs}  =  STMFD SP!, {regs}  (เหมือนกัน)
    @ LDMIA SP!, {regs}  =  LDMFD SP!, {regs}  (เหมือนกัน)
    
    MOV R7, #1
    SWI #0
```

---

## STMFD/LDMFD และ PUSH/POP

### รายละเอียดคำสั่ง STM/LDM

```asm
@ ============================================================
@ stm_ldm_variations.s
@ รูปแบบต่างๆ ของ STM/LDM
@ ============================================================
.section .text
.global _start

_start:
    @ ===== STM (Store Multiple) =====
    @ Syntax: STM{cond}<mode> Rn{!}, {registers}
    
    @ mode = IA (Increment After)  - default
    @        IB (Increment Before)
    @        DA (Decrement After)
    @        DB (Decrement Before) = FD stack
    
    @ Stack operations:
    @ STMFD = STMDB (Decrement Before = Full Descending)
    @ LDMFD = LDMIA (Increment After = Full Descending restore)
    
    @ ===== ตัวอย่าง STMIA =====
    @ เก็บ R0-R3 โดยเพิ่ม address หลังเก็บแต่ละตัว
    @ ใช้ R5 เป็น base pointer (ไม่ใช่ SP)
    LDR R5, =buffer         @ โหลด address ของ buffer
    
    MOV R0, #1
    MOV R1, #2
    MOV R2, #3
    MOV R3, #4
    
    STMIA R5, {R0-R3}       @ เก็บที่ buffer+0, +4, +8, +12
                             @ R5 ไม่เปลี่ยน (ไม่มี !)
    
    STMIA R5!, {R0-R3}      @ เก็บ แล้ว R5 += 16
                             @ R5 ชี้ไปหลัง R3
    
    @ ===== ตัวอย่าง LDMIA =====
    LDR R5, =buffer
    MOV R0, #0
    MOV R1, #0
    MOV R2, #0
    MOV R3, #0
    
    LDMIA R5, {R0-R3}       @ โหลดจาก buffer
    LDMIA R5!, {R0-R3}      @ โหลด แล้ว R5 += 16
    
    @ ===== Full Function Example =====
    BL my_function
    
    MOV R7, #1
    SWI #0

@ ฟังก์ชันตัวอย่าง
my_function:
    @ บันทึก registers ที่จะใช้
    STMFD SP!, {R4-R7, LR}  @ บันทึก R4-R7 และ LR
    
    @ ทำงาน...
    MOV R4, #100
    MOV R5, #200
    ADD R4, R4, R5          @ R4 = 300
    
    @ คืนค่า registers
    LDMFD SP!, {R4-R7, PC}  @ คืน R4-R7 และ jump ไป LR (= return)
                             @ PC = LR = return address

.section .bss
buffer: .space 32
```

### การ PUSH/POP หลาย Registers

```asm
@ ============================================================
@ push_pop_multiple.s
@ PUSH/POP หลาย registers
@ ============================================================
.section .text
.global _start

_start:
    @ ===== Syntax ของ PUSH/POP =====
    @ PUSH {Rn, Rm, ...}     @ push registers
    @ POP  {Rn, Rm, ...}     @ pop registers
    
    @ registers ใน {} เรียงตามตัวเลข (ไม่ใช่ลำดับที่ push)
    
    MOV R0, #0xAA
    MOV R1, #0xBB
    MOV R2, #0xCC
    MOV R3, #0xDD
    MOV R4, #0xEE
    MOV R5, #0xFF
    
    @ PUSH registers (R5 push ก่อน = highest address, R0 push หลัง = lowest address)
    PUSH {R0-R5}    @ SP -= 24, เก็บ R0...R5
    
    @ Stack layout (SP ชี้ที่ lowest address):
    @ SP+0:  R0 (0xAA)
    @ SP+4:  R1 (0xBB)
    @ SP+8:  R2 (0xCC)
    @ SP+12: R3 (0xDD)
    @ SP+16: R4 (0xEE)
    @ SP+20: R5 (0xFF)
    
    MOV R0, #0
    MOV R1, #0
    MOV R2, #0
    MOV R3, #0
    MOV R4, #0
    MOV R5, #0
    
    POP {R0-R5}     @ คืนค่าทั้งหมด
    
    @ ===== PUSH/POP กับ LR/PC =====
    @ สำหรับ function:
    @ PUSH {R4-R11, LR}   @ บันทึก callee-saved + return address
    @ POP  {R4-R11, PC}   @ คืนค่า + return (PC = old LR)
    
    MOV R7, #1
    SWI #0
```

---

## AAPCS Calling Convention

### ARM Architecture Procedure Call Standard (AAPCS)

AAPCS กำหนดกฎการเรียกฟังก์ชันใน ARM:

```
Register Usage ตาม AAPCS:
┌──────────┬──────────────┬──────────────────────────────────┐
│ Register │ AAPCS Name   │ การใช้งาน                        │
├──────────┼──────────────┼──────────────────────────────────┤
│ R0       │ a1           │ Argument 1 / Return value (32-bit)│
│ R1       │ a2           │ Argument 2 / Return value high    │
│ R2       │ a3           │ Argument 3                        │
│ R3       │ a4           │ Argument 4                        │
│ R4       │ v1           │ Variable (callee-saved)           │
│ R5       │ v2           │ Variable (callee-saved)           │
│ R6       │ v3           │ Variable (callee-saved)           │
│ R7       │ v4           │ Variable (callee-saved)           │
│ R8       │ v5           │ Variable (callee-saved)           │
│ R9       │ v6/SB        │ Variable / Static Base (platform) │
│ R10      │ v7/SL        │ Variable / Stack Limit            │
│ R11      │ v8/FP        │ Variable / Frame Pointer          │
│ R12      │ IP           │ Intra-Procedure scratch           │
│ R13      │ SP           │ Stack Pointer                     │
│ R14      │ LR           │ Link Register (return address)    │
│ R15      │ PC           │ Program Counter                   │
└──────────┴──────────────┴──────────────────────────────────┘
```

### Caller vs Callee Saved Registers

```
Caller-Saved (Scratch):  R0, R1, R2, R3, R12, LR
  - ผู้เรียก (caller) ต้องบันทึกเองถ้าต้องการใช้ค่าหลัง function call
  
Callee-Saved:  R4, R5, R6, R7, R8, R9, R10, R11
  - ฟังก์ชันที่ถูกเรียก (callee) ต้องบันทึกและคืนค่าเหล่านี้
```

```asm
@ ============================================================
@ aapcs_demo.s
@ สาธิต AAPCS Calling Convention
@ ============================================================
.section .text
.global _start
.global add_numbers
.global compute_sum

_start:
    @ ===== เรียกฟังก์ชัน add_numbers(10, 20) =====
    MOV R0, #10         @ arg1 = 10
    MOV R1, #20         @ arg2 = 20
    BL add_numbers      @ เรียกฟังก์ชัน, LR = return address
    @ R0 = ผลลัพธ์ = 30
    
    @ ===== เรียกฟังก์ชัน compute_sum(1, 2, 3, 4) =====
    MOV R0, #1          @ arg1
    MOV R1, #2          @ arg2
    MOV R2, #3          @ arg3
    MOV R3, #4          @ arg4
    BL compute_sum      @ เรียก
    @ R0 = 1+2+3+4 = 10
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int add_numbers(int a, int b)
@ R0 = a, R1 = b
@ return R0 = a + b
@ ============================================================
add_numbers:
    @ ฟังก์ชันนี้ไม่ใช้ callee-saved registers
    @ ดังนั้นไม่ต้อง PUSH/POP
    ADD R0, R0, R1      @ R0 = a + b
    BX LR               @ return (BX = Branch and eXchange)

@ ============================================================
@ int compute_sum(int a, int b, int c, int d)
@ R0=a, R1=b, R2=c, R3=d
@ return R0 = a+b+c+d
@ ============================================================
compute_sum:
    @ ฟังก์ชันนี้ใช้ R4 (callee-saved) จึงต้องบันทึก
    PUSH {R4, LR}       @ บันทึก R4 และ LR
    
    @ คำนวณ a+b
    ADD R4, R0, R1      @ R4 = a + b (ใช้ R4 เก็บผลชั่วคราว)
    
    @ คำนวณ (a+b) + c + d
    ADD R4, R4, R2      @ R4 = a + b + c
    ADD R0, R4, R3      @ R0 = a + b + c + d (ผลลัพธ์)
    
    POP {R4, PC}        @ คืน R4, return (PC = old LR)
```

### ตัวอย่าง AAPCS ครบถ้วน

```asm
@ ============================================================
@ aapcs_full.s
@ ตัวอย่าง AAPCS ครบถ้วนพร้อม caller/callee registers
@ Compile: arm-linux-gnueabi-as aapcs_full.s -o aapcs_full.o
@          arm-linux-gnueabi-ld aapcs_full.o -o aapcs_full
@ ============================================================
.section .data
result_msg: .ascii "Result: %d\n\0"
fmt_str:    .ascii "Value: %d, %d\n\0"

.section .text
.global _start

_start:
    @ ===== เรียก function ที่ใช้ callee-saved registers =====
    MOV R0, #5
    MOV R1, #7
    MOV R2, #3
    BL multiply_add     @ result = (a * b) + c = 5*7+3 = 38
    @ R0 = 38
    
    @ R0-R3 อาจเปลี่ยนหลัง call (caller-saved)
    @ ถ้าต้องการค่าเดิม ต้อง PUSH ก่อน BL
    
    @ ===== ตัวอย่าง caller บันทึกค่า =====
    MOV R4, #100        @ R4 เป็น callee-saved (จะคืนค่าหลัง call)
    
    MOV R0, #10
    MOV R1, #20
    BL add_numbers      @ R0 = 30, R4 ยังคงเป็น 100
    @ R4 = 100 (ไม่เปลี่ยน เพราะ callee-saved)
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int multiply_add(int a, int b, int c)
@ R0=a, R1=b, R2=c
@ return (a * b) + c
@ ============================================================
multiply_add:
    PUSH {R4, R5, LR}   @ บันทึก callee-saved ที่จะใช้
    
    MOV R4, R0          @ R4 = a (บันทึกไว้ใน callee-saved)
    MOV R5, R1          @ R5 = b
    
    @ คูณ a * b
    MUL R0, R4, R5      @ R0 = a * b
    ADD R0, R0, R2      @ R0 = (a * b) + c
    
    POP {R4, R5, PC}    @ คืนค่า และ return

@ ============================================================
@ int add_numbers(int a, int b)
@ ============================================================
add_numbers:
    @ ฟังก์ชันนี้ไม่ต้องการ PUSH เพราะไม่ใช้ callee-saved
    ADD R0, R0, R1
    BX LR
```

---

## Stack Frame Setup

### Stack Frame คืออะไร?

Stack Frame (หรือ Activation Record) คือส่วนของ stack ที่จัดสรรให้แต่ละฟังก์ชัน ประกอบด้วย:
- Saved registers
- Local variables
- (บางครั้ง) incoming arguments

```
Stack Frame Layout:
                    ┌────────────────┐
                    │  arg5+        │  ← passed on stack
                    │  arg4+        │
                    ├────────────────┤ ← caller's SP
                    │  saved LR     │
                    │  saved FP(R11)│
                    ├────────────────┤ ← FP (R11)
                    │  saved R4     │
                    │  saved R5     │
                    │  saved R6     │
                    │  saved R7     │
                    ├────────────────┤
                    │  local var 1  │
                    │  local var 2  │
                    │  local var 3  │
                    └────────────────┘ ← SP (current)
```

```asm
@ ============================================================
@ stack_frame.s
@ การสร้าง Stack Frame อย่างสมบูรณ์
@ ============================================================
.section .data
msg: .ascii "Stack frame demo\n\0"

.section .text
.global _start

_start:
    MOV R0, #3          @ arg1
    MOV R1, #7          @ arg2
    MOV R2, #1          @ arg3
    BL complex_function
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int complex_function(int a, int b, int c)
@ สร้าง stack frame อย่างสมบูรณ์
@ ============================================================
complex_function:
    @ ===== Function Prologue =====
    PUSH {R4-R8, R11, LR}   @ บันทึก callee-saved + FP + LR
    ADD  R11, SP, #20        @ ตั้ง FP ให้ชี้หลัง saved registers
                              @ (6 regs * 4 = 24 bytes, แต่ FP+LR อยู่ที่ offset 20)
    SUB  SP, SP, #16         @ จัดสรร local variables 4 ตัว (4*4=16 bytes)
    
    @ Stack ตอนนี้:
    @ R11+4:  saved LR
    @ R11+0:  saved R11 (FP)
    @ R11-4:  saved R8
    @ R11-8:  saved R7
    @ R11-12: saved R6
    @ R11-16: saved R5
    @ R11-20: saved R4
    @ R11-24: local var 1  ← SP+12
    @ R11-28: local var 2  ← SP+8
    @ R11-32: local var 3  ← SP+4
    @ R11-36: local var 4  ← SP+0
    
    @ เก็บ parameters ใน callee-saved registers
    MOV R4, R0              @ R4 = a
    MOV R5, R1              @ R5 = b
    MOV R6, R2              @ R6 = c
    
    @ เก็บ local variables ลง stack
    MOV R7, #100
    STR R7, [SP, #0]        @ local1 = 100
    MOV R7, #200
    STR R7, [SP, #4]        @ local2 = 200
    MOV R7, #300
    STR R7, [SP, #8]        @ local3 = 300
    MOV R7, #400
    STR R7, [SP, #12]       @ local4 = 400
    
    @ ทำงาน...
    LDR R7, [SP, #0]        @ โหลด local1
    ADD R0, R4, R5          @ a + b
    ADD R0, R0, R6          @ a + b + c
    ADD R0, R0, R7          @ a + b + c + local1
    
    @ ===== Function Epilogue =====
    SUB SP, R11, #20        @ คืน SP กลับ (ก่อน local vars)
    POP {R4-R8, R11, PC}    @ คืน registers และ return

@ ============================================================
@ ตัวอย่างที่ง่ายกว่า: ไม่ใช้ FP
@ ============================================================
simple_function:
    @ Prologue
    PUSH {R4, R5, LR}       @ 3 registers = 12 bytes
    SUB SP, SP, #8          @ local variables 2 ตัว
    
    @ body...
    MOV R4, R0
    STR R4, [SP, #0]        @ local1 = arg1
    MOV R5, #42
    STR R5, [SP, #4]        @ local2 = 42
    
    LDR R0, [SP, #0]        @ โหลด local1
    LDR R1, [SP, #4]        @ โหลด local2
    ADD R0, R0, R1          @ return local1 + local2
    
    @ Epilogue
    ADD SP, SP, #8          @ ปลด local variables
    POP {R4, R5, PC}        @ return
```

---

## Nested Function Calls

### การเรียกฟังก์ชันซ้อนกัน

เมื่อฟังก์ชันเรียกฟังก์ชันอื่น LR จะถูกเขียนทับ ดังนั้นต้องบันทึก LR ใน stack:

```asm
@ ============================================================
@ nested_calls.s
@ การเรียกฟังก์ชันซ้อนกัน
@ Compile: arm-linux-gnueabi-as nested_calls.s -o nested_calls.o
@          arm-linux-gnueabi-ld nested_calls.o -o nested_calls
@ Run:     qemu-arm ./nested_calls
@ ============================================================
.section .data
newline: .ascii "\n\0"

.section .text
.global _start

_start:
    @ เรียก level1 -> level2 -> level3
    MOV R0, #5
    BL level1           @ LR = return_addr_1
    @ R0 = ผลลัพธ์จาก level1
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int level1(int n) - เรียก level2
@ ============================================================
level1:
    PUSH {R4, LR}       @ บันทึก LR! (เพราะจะเรียก BL อีก)
    MOV R4, R0          @ R4 = n
    
    @ เรียก level2(n + 1)
    ADD R0, R4, #1      @ arg = n + 1
    BL level2           @ LR ถูก overwrite แต่ old LR อยู่ใน stack
    
    @ R0 = result จาก level2
    ADD R0, R0, R4      @ return result + n
    
    POP {R4, PC}        @ คืน R4 และ return

@ ============================================================
@ int level2(int n) - เรียก level3
@ ============================================================
level2:
    PUSH {R4, R5, LR}   @ บันทึก LR
    MOV R4, R0
    
    @ เรียก level3(n * 2)
    MOV R5, #2
    MUL R0, R4, R5      @ arg = n * 2
    BL level3
    
    @ R0 = result จาก level3
    ADD R0, R0, R4      @ return result + n
    
    POP {R4, R5, PC}

@ ============================================================
@ int level3(int n) - ฟังก์ชัน leaf (ไม่เรียกอะไรอีก)
@ ============================================================
level3:
    @ Leaf function: ไม่ต้อง PUSH LR
    ADD R0, R0, #1      @ return n + 1
    BX LR               @ return

@ ============================================================
@ ตัวอย่าง Recursive Function: Factorial
@ ============================================================
@ ============================================================
@ int factorial(int n)
@ if n <= 1, return 1
@ else return n * factorial(n-1)
@ ============================================================
factorial:
    PUSH {R4, LR}       @ บันทึก R4 และ LR (recursive!)
    MOV R4, R0          @ R4 = n
    
    @ Base case: n <= 1
    CMP R0, #1
    BLE fact_base       @ ถ้า n <= 1, ไป base case
    
    @ Recursive case: factorial(n-1)
    SUB R0, R4, #1      @ arg = n - 1
    BL factorial        @ เรียก factorial(n-1), LR อยู่ใน stack
    
    @ R0 = factorial(n-1)
    MUL R0, R4, R0      @ return n * factorial(n-1)
    POP {R4, PC}
    
fact_base:
    MOV R0, #1          @ return 1
    POP {R4, PC}
```

### ตัวอย่าง Fibonacci ด้วย Recursion

```asm
@ ============================================================
@ fibonacci.s
@ Fibonacci แบบ recursive สาธิต nested calls
@ ============================================================
.section .text
.global _start
.global fibonacci

_start:
    @ คำนวณ fib(10) = 55
    MOV R0, #10
    BL fibonacci
    @ R0 = 55
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int fibonacci(int n)
@ fib(0) = 0, fib(1) = 1
@ fib(n) = fib(n-1) + fib(n-2)
@ ============================================================
fibonacci:
    PUSH {R4, LR}
    MOV R4, R0          @ R4 = n
    
    @ Base cases
    CMP R4, #0
    MOVEQ R0, #0        @ fib(0) = 0
    BEQ fib_done
    
    CMP R4, #1
    MOVEQ R0, #1        @ fib(1) = 1
    BEQ fib_done
    
    @ fib(n-1)
    SUB R0, R4, #1
    BL fibonacci
    PUSH {R0}           @ บันทึก fib(n-1) ชั่วคราว
    
    @ fib(n-2)
    SUB R0, R4, #2
    BL fibonacci        @ R0 = fib(n-2)
    
    POP {R1}            @ R1 = fib(n-1)
    ADD R0, R0, R1      @ R0 = fib(n-1) + fib(n-2)
    
fib_done:
    POP {R4, PC}
```

---

## Local Variables on Stack

### การจัดการ Local Variables

```asm
@ ============================================================
@ local_variables.s
@ การใช้ local variables บน stack
@ ============================================================
.section .data
fmt_int:  .ascii "%d\n\0"
fmt_arr:  .ascii "arr[%d] = %d\n\0"

.section .text
.global _start

_start:
    @ เรียกฟังก์ชัน
    BL demo_locals
    BL demo_array_on_stack
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ void demo_locals()
@ สาธิตการใช้ local variables บน stack
@ ============================================================
demo_locals:
    @ Prologue
    PUSH {R4-R6, LR}
    @ จัดสรร local variables:
    @ [SP+0]  = int x
    @ [SP+4]  = int y
    @ [SP+8]  = int z
    @ [SP+12] = int result
    SUB SP, SP, #16         @ 4 local ints
    
    @ x = 10
    MOV R4, #10
    STR R4, [SP, #0]        @ x = 10
    
    @ y = 20
    MOV R4, #20
    STR R4, [SP, #4]        @ y = 20
    
    @ z = 30
    MOV R4, #30
    STR R4, [SP, #8]        @ z = 30
    
    @ result = x + y + z
    LDR R4, [SP, #0]        @ load x
    LDR R5, [SP, #4]        @ load y
    LDR R6, [SP, #8]        @ load z
    ADD R4, R4, R5
    ADD R4, R4, R6
    STR R4, [SP, #12]       @ result = 60
    
    @ Epilogue
    ADD SP, SP, #16         @ ปลด local variables
    POP {R4-R6, PC}

@ ============================================================
@ void demo_array_on_stack()
@ อาร์เรย์บน stack
@ ============================================================
demo_array_on_stack:
    @ Prologue
    PUSH {R4-R7, LR}
    @ จัดสรร array[10] บน stack
    @ 10 * 4 bytes = 40 bytes
    SUB SP, SP, #40
    
    @ เริ่มต้น array: arr[i] = i * 2
    MOV R4, #0              @ i = 0
    
init_loop:
    CMP R4, #10
    BGE init_done
    
    LSL R5, R4, #1          @ R5 = i * 2
    LSL R7, R4, #2          @ offset = i * 4
    ADD R6, SP, R7          @ address = SP + offset
    STR R5, [R6]            @ arr[i] = i * 2
    
    ADD R4, R4, #1          @ i++
    B init_loop
    
init_done:
    @ คำนวณ sum ของ array
    MOV R4, #0              @ i = 0
    MOV R6, #0              @ sum = 0
    
sum_loop:
    CMP R4, #10
    BGE sum_done
    
    LSL R7, R4, #2          @ offset = i * 4
    LDR R5, [SP, R7]        @ R5 = arr[i]
    ADD R6, R6, R5          @ sum += arr[i]
    ADD R4, R4, #1          @ i++
    B sum_loop
    
sum_done:
    @ R6 = sum = 0+2+4+6+8+10+12+14+16+18 = 90
    MOV R0, R6
    
    @ Epilogue
    ADD SP, SP, #40         @ ปลด array
    POP {R4-R7, PC}

@ ============================================================
@ void swap(int *a, int *b)
@ สลับค่าสองตัวบน stack
@ ============================================================
swap:
    @ ไม่ต้อง PUSH เพราะเป็น leaf function ที่ใช้แค่ R2
    LDR R2, [R0]            @ R2 = *a
    LDR R3, [R1]            @ R3 = *b  (R3 = caller-saved, ok)
    STR R3, [R0]            @ *a = *b
    STR R2, [R1]            @ *b = *a
    BX LR
```

### Stack Alignment

```asm
@ ============================================================
@ stack_alignment.s
@ Stack ต้อง align ที่ 8 bytes (AAPCS requirement)
@ ============================================================
.section .text
.global _start

_start:
    @ SP ต้อง align ที่ 8 bytes เมื่อเรียก public functions
    @ (เฉพาะ 4-byte alignment ก็พอสำหรับ ARM internal)
    
    @ PUSH 1 register = 4 bytes → misaligned!
    @ ต้องทำให้ครบ 8 bytes
    
bad_example:
    @ PUSH {LR}           @ SP -= 4 → SP ไม่ align ที่ 8
    @ ถ้าจะเรียก C library ต้อง 8-byte aligned
    
good_example:
    PUSH {R0, LR}           @ SP -= 8 → align ดี (8 bytes)
    @ ทำงาน...
    POP  {R0, PC}
    
@ ตัวอย่างที่ถูกต้อง: align stack ก่อนเรียก printf
call_printf:
    PUSH {R1, LR}           @ push dummy R1 เพื่อ align
    @ ... เรียก printf ...
    POP  {R1, PC}
    BX LR
```

---

## VFP Push/Pop

### VFP (Vector Floating Point) Registers

```asm
@ ============================================================
@ vfp_stack.s
@ การ push/pop VFP floating point registers
@ Compile: arm-linux-gnueabihf-as -mfpu=vfpv3 vfp_stack.s -o vfp_stack.o
@ ============================================================
.section .data
pi_val:  .float 3.14159
e_val:   .float 2.71828

.section .text
.global _start

_start:
    @ Enable VFP (ปกติ OS ทำให้แล้ว)
    
    @ โหลดค่า float
    LDR R0, =pi_val
    VLDR S0, [R0]           @ S0 = 3.14159
    LDR R0, =e_val
    VLDR S1, [R0]           @ S1 = 2.71828
    
    @ เรียก VFP function
    BL vfp_compute
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ VFP Calling Convention:
@ S0-S15 (D0-D7)  = argument/return (caller-saved)
@ S16-S31 (D8-D15) = callee-saved
@ ============================================================

vfp_compute:
    @ บันทึก VFP callee-saved registers
    @ VPUSH บันทึก S/D registers ลง stack
    VPUSH {D8-D11}          @ บันทึก D8-D11 (= S16-S23)
    
    @ ทำการคำนวณ
    VADD.F32 S2, S0, S1     @ S2 = pi + e
    VMUL.F32 D8, D0, D1     @ D8 = pi * e (เป็น double operation)
    
    @ ผลลัพธ์ใน S0 (return value)
    VMOV S0, S2             @ return pi + e
    
    @ คืน VFP callee-saved registers
    VPOP {D8-D11}
    BX LR

@ ============================================================
@ ตัวอย่าง: function ที่มีทั้ง integer และ float parameters
@ float compute(int n, float x, float y)
@ AAPCS-VFP: int args ใน R0-R3, float/double ใน S0-S15/D0-D7
@ ============================================================
mixed_compute:
    @ R0 = n (int)
    @ S0 = x (float)
    @ S1 = y (float)
    
    PUSH {R4, LR}
    VPUSH {S16-S19}         @ บันทึก callee-saved VFP
    
    MOV R4, R0              @ R4 = n
    VMOV S16, S0            @ S16 = x (callee-saved copy)
    VMOV S17, S1            @ S17 = y
    
    @ คำนวณ x + y
    VADD.F32 S0, S16, S17   @ S0 = x + y
    
    @ แปลง int n เป็น float
    VMOV S18, R4            @ S18 = n (bit pattern)
    VCVT.F32.S32 S18, S18   @ S18 = (float)n
    
    @ ผลลัพธ์ = (x + y) * n
    VMUL.F32 S0, S0, S18    @ S0 = (x + y) * n
    
    VPOP {S16-S19}
    POP {R4, PC}

@ ============================================================
@ VFP Double Precision (D registers)
@ ============================================================
double_example:
    PUSH {LR}
    VPUSH {D8-D15}          @ บันทึก D8-D15
    
    @ D0 = first arg (double), D1 = second arg
    VADD.F64 D8, D0, D1     @ D8 = D0 + D1
    VSQRT.F64 D0, D8        @ D0 = sqrt(D0 + D1)  (return)
    
    VPOP {D8-D15}
    POP {PC}
```

---

## ARM PCS - Procedure Call Standard

### ARM PCS กฎพิเศษ

```asm
@ ============================================================
@ arm_pcs.s
@ กฎพิเศษของ ARM Procedure Call Standard
@ ============================================================
.section .text
.global _start

@ ============================================================
@ 1. Thumb interworking: BX vs BL vs BLX
@ ============================================================

_start:
    @ BL: Branch with Link (เรียก ARM function)
    BL arm_function         @ return address ใน LR (bit0 = 0)
    
    @ BLX: Branch with Link and eXchange (สลับ ARM/Thumb)
    @ BLX thumb_function    @ เรียก Thumb function จาก ARM
    
    MOV R7, #1
    SWI #0

arm_function:
    @ ARM mode function
    BX LR                   @ return (BX preserves Thumb bit)

@ ============================================================
@ 2. Inline assembly (C interop)
@ ============================================================
@ ใน C:
@ int arm_add(int a, int b) {
@     return a + b;
@ }
@ 
@ ARM Assembly:
@ .global arm_add
@ arm_add:
@     ADD R0, R0, R1
@     BX LR

@ ============================================================
@ 3. Position Independent Code (PIC)
@ ============================================================
pic_example:
    @ ใน shared library ต้องใช้ PC-relative addressing
    @ LDR R0, =global_var  @ ไม่ใช้ใน PIC!
    
    @ แทน: ใช้ GOT (Global Offset Table)
    LDR R0, [PC, #8]        @ โหลด GOT pointer
    LDR R0, [R0]            @ dereference GOT
    BX LR
    .word 0                  @ GOT offset (ตัวอย่าง)

@ ============================================================
@ 4. Variadic functions (__va_list)
@ ============================================================
@ int sum(int count, ...)
@ R0 = count, R1-R3 = args 2-4, stack = args 5+
@
@ ตัวอย่าง: sum(4, 10, 20, 30, 40) = 100
@ R0=4, R1=10, R2=20, R3=30, [SP]=40

variadic_sum:
    PUSH {R4-R7, LR}
    MOV R4, R0              @ R4 = count
    MOV R5, #0              @ R5 = sum = 0
    MOV R6, #0              @ R6 = arg index
    
    @ เก็บ R1-R3 ลง stack (va_list area)
    @ (ปกติ compiler จัดการให้)
    PUSH {R1-R3}            @ args 2-4 บน stack
    
    MOV R7, SP              @ R7 = pointer to args
    
va_loop:
    CMP R6, R4
    BGE va_done
    
    LDR R0, [R7, R6, LSL #2] @ โหลด arg[i]
    ADD R5, R5, R0
    ADD R6, R6, #1
    B va_loop
    
va_done:
    MOV R0, R5              @ return sum
    ADD SP, SP, #12         @ ปลด R1-R3 ที่ push ไว้
    POP {R4-R7, PC}
```

---

## Parameter Passing Beyond R3

### เมื่อ Arguments มีมากกว่า 4 ตัว

เมื่อฟังก์ชันมีพารามิเตอร์มากกว่า 4 ตัว พารามิเตอร์ที่ 5 เป็นต้นไปจะถูกส่งผ่าน stack:

```asm
@ ============================================================
@ params_beyond_r3.s
@ การส่ง parameters มากกว่า 4 ตัว
@ ============================================================
.section .text
.global _start
.global sum6
.global sum8

_start:
    @ เรียก sum6(1, 2, 3, 4, 5, 6)
    @ R0=1, R1=2, R2=3, R3=4, [SP+0]=5, [SP+4]=6
    
    @ ต้อง push arg6, arg5 ก่อน (push จาก high ไป low)
    MOV R0, #6
    PUSH {R0}               @ arg6 = 6 บน stack
    MOV R0, #5
    PUSH {R0}               @ arg5 = 5 บน stack
    
    @ Stack ตอนนี้:
    @ SP+0: arg5 = 5
    @ SP+4: arg6 = 6
    
    MOV R0, #1              @ arg1
    MOV R1, #2              @ arg2
    MOV R2, #3              @ arg3
    MOV R3, #4              @ arg4
    BL sum6
    
    ADD SP, SP, #8          @ ปลด arg5, arg6 หลัง call
    @ R0 = 1+2+3+4+5+6 = 21
    
    @ เรียก sum8(1, 2, 3, 4, 5, 6, 7, 8)
    MOV R0, #8
    PUSH {R0}               @ arg8
    MOV R0, #7
    PUSH {R0}               @ arg7
    MOV R0, #6
    PUSH {R0}               @ arg6
    MOV R0, #5
    PUSH {R0}               @ arg5
    
    MOV R0, #1
    MOV R1, #2
    MOV R2, #3
    MOV R3, #4
    BL sum8
    
    ADD SP, SP, #16         @ ปลด 4 stack args
    @ R0 = 36
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int sum6(int a, int b, int c, int d, int e, int f)
@ R0=a, R1=b, R2=c, R3=d, [SP+0]=e, [SP+4]=f
@ ============================================================
sum6:
    PUSH {R4, R5, LR}
    @ Stack layout หลัง PUSH:
    @ SP+0:  saved R4
    @ SP+4:  saved R5
    @ SP+8:  saved LR
    @ SP+12: e (arg5)
    @ SP+16: f (arg6)
    
    @ โหลด arg5 และ arg6 จาก stack
    LDR R4, [SP, #12]       @ R4 = e (arg5)
    LDR R5, [SP, #16]       @ R5 = f (arg6)
    
    @ คำนวณผลรวม
    ADD R0, R0, R1          @ a + b
    ADD R0, R0, R2          @ + c
    ADD R0, R0, R3          @ + d
    ADD R0, R0, R4          @ + e
    ADD R0, R0, R5          @ + f
    
    POP {R4, R5, PC}

@ ============================================================
@ int sum8(int a, int b, int c, int d, int e, int f, int g, int h)
@ R0-R3 = a,b,c,d; Stack = e,f,g,h
@ ============================================================
sum8:
    PUSH {R4-R7, LR}
    @ หลัง PUSH 5 regs (20 bytes):
    @ SP+20: e (arg5)
    @ SP+24: f (arg6)
    @ SP+28: g (arg7)
    @ SP+32: h (arg8)
    
    LDR R4, [SP, #20]       @ R4 = e
    LDR R5, [SP, #24]       @ R5 = f
    LDR R6, [SP, #28]       @ R6 = g
    LDR R7, [SP, #32]       @ R7 = h
    
    ADD R0, R0, R1
    ADD R0, R0, R2
    ADD R0, R0, R3
    ADD R0, R0, R4
    ADD R0, R0, R5
    ADD R0, R0, R6
    ADD R0, R0, R7
    
    POP {R4-R7, PC}

@ ============================================================
@ ตัวอย่าง: mixed int/float parameters
@ float compute(float a, float b, int n, float c, float d, int m)
@ Soft-float ABI: ทุกอย่างใน R0-R3 และ stack
@ S0=a→R0, S1=b→R1, int n→R2, S2=c→R3, [SP+0]=d(float), [SP+4]=m(int)
@ Hard-float ABI: float ใน S/D registers
@ ============================================================
```

### ตัวอย่าง C-to-Assembly: 6-argument function

```asm
@ ============================================================
@ c_interop_6args.s
@ ตัวอย่าง C function ที่มี 6 arguments
@ เทียบเท่า C:
@   int process(int a, int b, int c, int d, int e, int f) {
@       int sum = a + b + c + d + e + f;
@       int product = a * b;
@       return sum + product;
@   }
@ ============================================================
.section .text
.global process

process:
    @ R0=a, R1=b, R2=c, R3=d
    @ [SP+0]=e, [SP+4]=f (before our PUSH)
    
    PUSH {R4-R7, LR}
    @ หลัง PUSH: arg5 อยู่ที่ SP+20, arg6 อยู่ที่ SP+24
    
    MOV R4, R0              @ R4 = a
    MOV R5, R1              @ R5 = b
    
    @ โหลด e และ f
    LDR R6, [SP, #20]       @ R6 = e
    LDR R7, [SP, #24]       @ R7 = f
    
    @ คำนวณ sum = a+b+c+d+e+f
    ADD R0, R4, R5          @ a + b
    ADD R0, R0, R2          @ + c
    ADD R0, R0, R3          @ + d
    ADD R0, R0, R6          @ + e
    ADD R0, R0, R7          @ + f
    @ R0 = sum
    
    @ คำนวณ product = a * b
    MUL R1, R4, R5          @ R1 = a * b
    
    ADD R0, R0, R1          @ return sum + product
    
    POP {R4-R7, PC}
```

---

## Large Struct Passing

### การส่ง Struct ขนาดใหญ่

```asm
@ ============================================================
@ struct_passing.s
@ การส่ง Struct เป็น parameter
@ 
@ เทียบเท่า C:
@ typedef struct {
@     int x, y, z, w;   // 16 bytes = 4 registers
@ } Point4D;
@
@ typedef struct {
@     int a, b, c, d, e, f;  // 24 bytes > 4 registers
@ } BigStruct;
@ ============================================================

.section .data
@ ตัวอย่าง struct data
small_point:
    .word 10    @ x
    .word 20    @ y
    .word 30    @ z
    .word 40    @ w

big_struct_data:
    .word 1     @ a
    .word 2     @ b
    .word 3     @ c
    .word 4     @ d
    .word 5     @ e
    .word 6     @ f

.section .text
.global _start
.global sum_point4d
.global process_big_struct

_start:
    @ ===== Small struct (4 words = 4 registers) =====
    @ ส่งผ่าน R0-R3 โดยตรง
    LDR R0, =small_point
    LDMIA R0, {R0-R3}       @ โหลด x,y,z,w เข้า R0-R3
    BL sum_point4d          @ R0=x, R1=y, R2=z, R3=w
    @ R0 = 10+20+30+40 = 100
    
    @ ===== Large struct (> 4 words) =====
    @ ส่งผ่าน pointer ใน R0
    LDR R0, =big_struct_data
    BL process_big_struct
    @ R0 = 1+2+3+4+5+6 = 21
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int sum_point4d(int x, int y, int z, int w)
@ Struct ขนาด <= 16 bytes ส่งใน registers
@ ============================================================
sum_point4d:
    ADD R0, R0, R1
    ADD R0, R0, R2
    ADD R0, R0, R3
    BX LR

@ ============================================================
@ int process_big_struct(BigStruct *s)
@ Struct ขนาด > 16 bytes ส่งเป็น pointer
@ ============================================================
process_big_struct:
    PUSH {R4-R8, LR}
    MOV R4, R0              @ R4 = pointer to struct
    
    @ โหลดฟิลด์ทีละตัว
    LDR R5, [R4, #0]        @ a
    LDR R6, [R4, #4]        @ b
    LDR R7, [R4, #8]        @ c
    LDR R8, [R4, #12]       @ d
    
    ADD R0, R5, R6
    ADD R0, R0, R7
    ADD R0, R0, R8
    
    LDR R5, [R4, #16]       @ e
    LDR R6, [R4, #20]       @ f
    ADD R0, R0, R5
    ADD R0, R0, R6
    
    POP {R4-R8, PC}

@ ============================================================
@ การส่ง struct โดยค่า (pass by value) - copy บน stack
@ void modify_struct_copy(BigStruct s)
@ ===== Caller code =====
@ Caller ต้อง push struct ลง stack ก่อนเรียก
@ ============================================================
call_with_struct_copy:
    PUSH {R4, LR}
    
    @ โหลด struct แล้ว push ลง stack (ลำดับกลับกัน)
    LDR R4, =big_struct_data
    
    @ Push 6 words ลง stack (24 bytes)
    LDR R0, [R4, #20]       @ f
    PUSH {R0}
    LDR R0, [R4, #16]       @ e
    PUSH {R0}
    LDR R0, [R4, #12]       @ d
    PUSH {R0}
    LDR R0, [R4, #8]        @ c
    PUSH {R0}
    LDR R0, [R4, #4]        @ b
    PUSH {R0}
    LDR R0, [R4, #0]        @ a → R0 (first field)
    
    @ struct ขนาด 24 bytes = 4 registers + 2 on stack
    LDR R1, [R4, #4]        @ R1 = b
    LDR R2, [R4, #8]        @ R2 = c
    LDR R3, [R4, #12]       @ R3 = d
    @ R0 = a, R1 = b, R2 = c, R3 = d (แล้ว e, f อยู่บน stack)
    
    BL modify_struct_copy
    ADD SP, SP, #8          @ ปลด e, f จาก stack
    
    POP {R4, PC}

modify_struct_copy:
    @ R0=a, R1=b, R2=c, R3=d, [SP+0]=e, [SP+4]=f
    @ (เป็น copy ของ struct ดังนั้น modify ได้เลย)
    PUSH {LR}
    
    @ ทำอะไรก็ได้กับ struct
    ADD R0, R0, R1          @ a + b (เป็น local copy)
    
    POP {PC}
```

---

## Return Struct in Memory

### การ Return Struct จาก Function

```asm
@ ============================================================
@ return_struct.s
@ การคืนค่า Struct จาก Function
@ 
@ เทียบเท่า C:
@ typedef struct { int x, y; } Point2D;   // 8 bytes
@ typedef struct { int a, b, c; } Triple;  // 12 bytes
@ typedef struct { int data[8]; } Large;   // 32 bytes
@ ============================================================

.section .bss
result_large: .space 32     @ พื้นที่สำหรับ Large struct

.section .text
.global _start
.global make_point2d
.global make_triple
.global make_large

_start:
    @ ===== Return small struct (8 bytes) - ผ่าน R0:R1 =====
    MOV R0, #10             @ arg1
    MOV R1, #20             @ arg2
    BL make_point2d
    @ R0 = x = 10, R1 = y = 20 (struct ใน R0:R1)
    
    @ ===== Return medium struct (12 bytes) - ผ่าน R0:R1:R2 =====
    MOV R0, #1
    MOV R1, #2
    MOV R2, #3
    BL make_triple
    @ R0=1, R1=2, R2=3
    
    @ ===== Return large struct - ผ่าน pointer (R0) =====
    @ Caller ต้องจัดสรรพื้นที่แล้วส่ง pointer ใน R0 (hidden first arg)
    LDR R0, =result_large   @ R0 = pointer to buffer
    MOV R1, #5              @ ค่า initial
    BL make_large
    @ struct ถูกเก็บที่ result_large
    @ R0 = pointer to result (same as input R0)
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ Point2D make_point2d(int x, int y)
@ Return ใน R0 (x) และ R1 (y) - struct ≤ 8 bytes
@ ============================================================
make_point2d:
    @ R0 = x, R1 = y อยู่แล้ว
    @ แค่ return โดยไม่เปลี่ยน R0, R1
    @ (ในกรณีนี้ parameters เป็น return value ด้วย)
    BX LR

@ ============================================================
@ Triple make_triple(int a, int b, int c)  
@ Return ใน R0:R1:R2 - struct ≤ 12 bytes
@ ============================================================
make_triple:
    @ R0, R1, R2 เป็น return value แล้ว
    BX LR

@ ============================================================
@ Large make_large(int init_val) - HIDDEN POINTER CONVENTION
@ 
@ เมื่อ return struct > 4 bytes (บางคอมไพเลอร์ > 8 bytes):
@ Caller ส่ง pointer ใน R0 (hidden first argument)
@ Function parameters ขยับเป็น R1+
@ 
@ Assembly version:
@ R0 = pointer to result (hidden), R1 = init_val
@ ============================================================
make_large:
    @ R0 = result pointer (hidden)
    @ R1 = init_val
    PUSH {R4-R6, LR}
    MOV R4, R0              @ R4 = result pointer
    MOV R5, R1              @ R5 = init_val
    
    @ เติมข้อมูลใน struct
    MOV R6, #0              @ index = 0
fill_large:
    CMP R6, #8
    BGE fill_done
    
    LSL R0, R6, #2          @ offset = index * 4
    ADD R1, R5, R6          @ value = init_val + index
    STR R1, [R4, R0]        @ result->data[index] = value
    ADD R6, R6, #1
    B fill_large
    
fill_done:
    MOV R0, R4              @ return pointer to result
    POP {R4-R6, PC}

@ ============================================================
@ ตัวอย่างครบถ้วน: Complex Return
@ ============================================================
.section .data
result_buffer: .space 64    @ buffer for returned struct

.section .text

@ ============================================================
@ void fill_struct(OutputStruct *out, int count)
@ เติม struct ด้วยข้อมูล
@ R0 = out pointer, R1 = count
@ ============================================================
fill_struct:
    PUSH {R4-R7, LR}
    MOV R4, R0              @ R4 = output pointer
    MOV R5, R1              @ R5 = count
    MOV R6, #0              @ i = 0
    
loop:
    CMP R6, R5
    BGE done
    
    LSL R7, R6, #2          @ offset = i * 4
    ADD R0, R6, #1          @ value = i + 1
    STR R0, [R4, R7]        @ out[i] = i + 1
    ADD R6, R6, #1
    B loop
    
done:
    POP {R4-R7, PC}
```

---

## AArch64 Stack และ Calling Convention

### ARM64 / AArch64 Stack

AArch64 มีกฎ stack ที่เข้มงวดกว่า ARM32:
- Stack ต้อง **16-byte aligned** เสมอ
- ใช้ **STP/LDP** (Store/Load Pair) แทน STM/LDM

```asm
@ ============================================================
@ aarch64_stack.s
@ AArch64 Stack และ Calling Convention
@ Compile: aarch64-linux-gnu-as aarch64_stack.s -o aarch64_stack.o
@          aarch64-linux-gnu-ld aarch64_stack.o -o aarch64_stack
@ Run:     qemu-aarch64 ./aarch64_stack
@ ============================================================
.section .text
.global _start

_start:
    // เรียก function
    MOV X0, #10
    MOV X1, #20
    BL add_a64
    // X0 = 30
    
    MOV X0, #0
    MOV X8, #93         // syscall exit
    SVC #0

// ============================================================
// AArch64 Register Calling Convention:
// X0-X7   = arguments and return values (caller-saved)
// X8      = indirect result location (hidden struct pointer)
// X9-X15  = scratch (caller-saved)
// X16-X17 = intra-procedure scratch (IP0, IP1)
// X18     = platform register (avoid)
// X19-X28 = callee-saved
// X29     = frame pointer (FP)
// X30     = link register (LR)
// SP      = stack pointer (must be 16-byte aligned)
// ============================================================

// ============================================================
// int add_a64(int a, int b)
// W0=a, W1=b (W = 32-bit view of X)
// ============================================================
add_a64:
    ADD W0, W0, W1      // W0 = a + b
    RET                 // = BR X30

// ============================================================
// AArch64 Function Prologue/Epilogue
// ============================================================
complex_a64:
    // Prologue: บันทึก callee-saved registers
    // STP = Store Pair (เก็บ 2 registers พร้อมกัน)
    // Stack ต้อง 16-byte aligned → push ทีละ 2 เสมอ
    STP X29, X30, [SP, #-16]!   // save FP, LR; SP -= 16
    MOV X29, SP                  // FP = SP (frame pointer)
    
    STP X19, X20, [SP, #-16]!   // save X19, X20
    STP X21, X22, [SP, #-16]!   // save X21, X22
    
    // จัดสรร local variables (ต้อง 16-byte aligned)
    SUB SP, SP, #32              // 8 local variables (8*4 = 32 bytes)
    
    // body...
    MOV X19, X0                  // save arg1
    MOV X20, X1                  // save arg2
    
    STR W19, [SP, #0]            // local1 = arg1
    STR W20, [SP, #4]            // local2 = arg2
    
    // คำนวณ
    LDR W0, [SP, #0]
    LDR W1, [SP, #4]
    ADD W0, W0, W1               // result = local1 + local2
    
    // Epilogue
    ADD SP, SP, #32              // ปลด local variables
    LDP X21, X22, [SP], #16     // restore X21, X22; SP += 16
    LDP X19, X20, [SP], #16     // restore X19, X20
    LDP X29, X30, [SP], #16     // restore FP, LR; SP += 16
    RET

// ============================================================
// AArch64 Nested Calls
// ============================================================
outer_a64:
    STP X29, X30, [SP, #-16]!   // บันทึก LR (X30) จำเป็น!
    MOV X29, SP
    
    STP X19, X20, [SP, #-16]!
    
    MOV X19, X0                  // บันทึก arg1
    MOV X20, X1                  // บันทึก arg2
    
    // เรียก inner function
    ADD X0, X19, #1              // arg = arg1 + 1
    BL inner_a64                 // X30 (LR) ถูกแก้ แต่ old LR อยู่ใน stack
    
    MOV X0, X19                  // เรียกใช้ค่าเดิมของ arg1
    
    LDP X19, X20, [SP], #16
    LDP X29, X30, [SP], #16
    RET

inner_a64:
    // Leaf function
    ADD X0, X0, #10
    RET

// ============================================================
// AArch64: Parameters Beyond X7 (> 8 args)
// ============================================================
// int sum9(int a, int b, int c, int d, int e, int f, int g, int h, int i)
// X0=a ... X7=h, [SP+0]=i
// ============================================================
sum9_a64:
    STP X29, X30, [SP, #-16]!
    MOV X29, SP
    
    // โหลด arg9 จาก stack
    // Stack layout: [SP+16] = i (หลัง STP, SP ลดไป 16)
    LDR W9, [SP, #16]
    
    ADD W0, W0, W1
    ADD W0, W0, W2
    ADD W0, W0, W3
    ADD W0, W0, W4
    ADD W0, W0, W5
    ADD W0, W0, W6
    ADD W0, W0, W7
    ADD W0, W0, W9       // + arg9
    
    LDP X29, X30, [SP], #16
    RET

// ============================================================
// AArch64: Return Large Struct
// X8 = hidden result pointer
// ============================================================
make_large_a64:
    // X8 = result pointer (hidden)
    // X0 = init value
    STP X29, X30, [SP, #-16]!
    MOV X29, SP
    STP X19, X20, [SP, #-16]!
    
    MOV X19, X8         // result pointer
    MOV X20, X0         // init value
    
    MOV X9, #0          // i = 0
fill_loop_a64:
    CMP X9, #8
    BGE fill_done_a64
    
    ADD W10, W20, W9    // value = init + i
    STR W10, [X19, X9, LSL #2]  // result[i] = value
    ADD X9, X9, #1
    B fill_loop_a64
    
fill_done_a64:
    // ไม่ต้อง return ค่า (struct อยู่ใน *X8 แล้ว)
    LDP X19, X20, [SP], #16
    LDP X29, X30, [SP], #16
    RET
```

### AArch64 SIMD/NEON Stack

```asm
// ============================================================
// aarch64_neon_stack.s
// NEON registers บน stack
// ============================================================
.section .text
.global neon_function

neon_function:
    // NEON callee-saved: D8-D15 (V8-V15 lower 64 bits)
    // NEON caller-saved: D0-D7, D16-D31
    
    STP X29, X30, [SP, #-16]!
    MOV X29, SP
    
    // บันทึก NEON callee-saved (ใช้ STP สำหรับ D registers)
    STP D8, D9, [SP, #-16]!     // save D8, D9
    STP D10, D11, [SP, #-16]!   // save D10, D11
    
    // ทำงานกับ NEON
    FMOV D8, #1.0               // D8 = 1.0
    FMOV D9, #2.0               // D9 = 2.0
    FADD D0, D8, D9             // D0 = 3.0 (return)
    
    // คืน NEON registers
    LDP D10, D11, [SP], #16
    LDP D8, D9, [SP], #16
    LDP X29, X30, [SP], #16
    RET
```

---

## การ Compile และ Run ด้วย QEMU

### ARM32 (Raspberry Pi / Linux)

```bash
#!/bin/bash
# ============================================================
# compile_arm32.sh
# Script สำหรับ compile และ run ARM32 assembly
# ============================================================

# ติดตั้ง cross-compiler (Ubuntu/Debian)
# sudo apt install gcc-arm-linux-gnueabi qemu-user

# Compile ARM32 assembly
arm-linux-gnueabi-as \
    -march=armv7-a \
    -mfpu=vfpv3 \
    your_program.s \
    -o your_program.o

# Link
arm-linux-gnueabi-ld \
    your_program.o \
    -o your_program

# Run ด้วย QEMU
qemu-arm \
    -L /usr/arm-linux-gnueabi/ \
    ./your_program

echo "Exit code: $?"
```

### ARM64 / AArch64

```bash
#!/bin/bash
# ============================================================
# compile_aarch64.sh
# Script สำหรับ compile และ run AArch64 assembly
# ============================================================

# ติดตั้ง (Ubuntu/Debian)
# sudo apt install gcc-aarch64-linux-gnu qemu-user

# Compile AArch64 assembly
aarch64-linux-gnu-as \
    your_program.s \
    -o your_program.o

# Link
aarch64-linux-gnu-ld \
    your_program.o \
    -o your_program

# Run ด้วย QEMU
qemu-aarch64 \
    -L /usr/aarch64-linux-gnu/ \
    ./your_program

echo "Exit code: $?"
```

### Makefile สำหรับทั้ง ARM32 และ ARM64

```makefile
# ============================================================
# Makefile
# Build และ test ARM assembly programs
# ============================================================

ARM32_AS  = arm-linux-gnueabi-as
ARM32_LD  = arm-linux-gnueabi-ld
ARM32_RUN = qemu-arm -L /usr/arm-linux-gnueabi/

ARM64_AS  = aarch64-linux-gnu-as
ARM64_LD  = aarch64-linux-gnu-ld
ARM64_RUN = qemu-aarch64 -L /usr/aarch64-linux-gnu/

ARM32_FLAGS = -march=armv7-a -mfpu=vfpv3

all: arm32 arm64

arm32: part035_arm32
arm64: part035_arm64

part035_arm32: part035_arm32.o
	$(ARM32_LD) $< -o $@

part035_arm32.o: part035_arm32.s
	$(ARM32_AS) $(ARM32_FLAGS) $< -o $@

part035_arm64: part035_arm64.o
	$(ARM64_LD) $< -o $@

part035_arm64.o: part035_arm64.s
	$(ARM64_AS) $< -o $@

run_arm32: part035_arm32
	$(ARM32_RUN) ./part035_arm32; echo "Exit: $$?"

run_arm64: part035_arm64
	$(ARM64_RUN) ./part035_arm64; echo "Exit: $$?"

clean:
	rm -f *.o part035_arm32 part035_arm64

.PHONY: all arm32 arm64 run_arm32 run_arm64 clean
```

---

## โปรแกรมตัวอย่างครบถ้วน: Calculator

```asm
@ ============================================================
@ calculator.s
@ โปรแกรม Calculator ครบถ้วน แสดงการใช้ Stack และ Subroutines
@ Compile: arm-linux-gnueabi-as calculator.s -o calculator.o
@          arm-linux-gnueabi-ld calculator.o -o calculator
@ ============================================================
.section .data
@ Strings
msg_add:    .ascii "Addition result: "
msg_add_end:
msg_sub:    .ascii "Subtraction result: "
msg_sub_end:
msg_mul:    .ascii "Multiplication result: "
msg_mul_end:
msg_div:    .ascii "Division result: "
msg_div_end:
msg_newline: .ascii "\n"

.section .bss
num_buf:    .space 20   @ buffer สำหรับแสดงตัวเลข

.section .text
.global _start

@ ===== Syscall constants =====
.equ SYS_WRITE, 4
.equ SYS_EXIT,  1
.equ STDOUT,    1

_start:
    @ ============================
    @ ทดสอบการบวก: add(15, 27) = 42
    @ ============================
    MOV R0, #15
    MOV R1, #27
    BL calc_add
    @ R0 = 42
    
    @ แสดงผล "Addition result: 42\n"
    PUSH {R0}               @ บันทึกผลลัพธ์
    
    MOV R0, #STDOUT
    LDR R1, =msg_add
    MOV R2, #(msg_add_end - msg_add)
    MOV R7, #SYS_WRITE
    SWI #0
    
    POP {R0}                @ คืนผลลัพธ์
    BL print_int
    BL print_newline
    
    @ ============================
    @ ทดสอบการลบ: sub(100, 37) = 63
    @ ============================
    MOV R0, #100
    MOV R1, #37
    BL calc_sub
    
    PUSH {R0}
    MOV R0, #STDOUT
    LDR R1, =msg_sub
    MOV R2, #(msg_sub_end - msg_sub)
    MOV R7, #SYS_WRITE
    SWI #0
    POP {R0}
    BL print_int
    BL print_newline
    
    @ ============================
    @ ทดสอบการคูณ: mul(6, 7) = 42
    @ ============================
    MOV R0, #6
    MOV R1, #7
    BL calc_mul
    
    PUSH {R0}
    MOV R0, #STDOUT
    LDR R1, =msg_mul
    MOV R2, #(msg_mul_end - msg_mul)
    MOV R7, #SYS_WRITE
    SWI #0
    POP {R0}
    BL print_int
    BL print_newline
    
    @ ============================
    @ ทดสอบการหาร: div(84, 2) = 42
    @ ============================
    MOV R0, #84
    MOV R1, #2
    BL calc_div
    
    PUSH {R0}
    MOV R0, #STDOUT
    LDR R1, =msg_div
    MOV R2, #(msg_div_end - msg_div)
    MOV R7, #SYS_WRITE
    SWI #0
    POP {R0}
    BL print_int
    BL print_newline
    
    @ จบโปรแกรม
    MOV R0, #0
    MOV R7, #SYS_EXIT
    SWI #0

@ ============================================================
@ int calc_add(int a, int b) - return a + b
@ ============================================================
calc_add:
    ADD R0, R0, R1
    BX LR

@ ============================================================
@ int calc_sub(int a, int b) - return a - b
@ ============================================================
calc_sub:
    SUB R0, R0, R1
    BX LR

@ ============================================================
@ int calc_mul(int a, int b) - return a * b
@ ============================================================
calc_mul:
    MUL R0, R0, R1
    BX LR

@ ============================================================
@ int calc_div(int a, int b) - return a / b (integer division)
@ ============================================================
calc_div:
    @ ARM32 ไม่มี DIV instruction (ยกเว้น ARMv7 SDIV/UDIV)
    @ ใช้ซอฟต์แวร์ division (หรือ SDIV ถ้ามี)
    @ ตรวจสอบ ARMv7-A: ใช้ SDIV ได้
    @ SDIV R0, R0, R1     @ หาร signed
    
    @ สำหรับ CPU ที่ไม่มี SDIV: ใช้ __aeabi_idiv
    @ (เรียกผ่าน BL ต้องมี library)
    
    @ ตัวอย่างนี้ใช้ UDIV (unsigned division, ARMv7-R/M)
    @ หรือทำ software division:
    PUSH {R4, R5, LR}
    MOV R4, R0              @ R4 = dividend
    MOV R5, R1              @ R5 = divisor
    
    MOV R0, #0              @ quotient = 0
    
div_loop:
    CMP R4, R5
    BLT div_done
    SUB R4, R4, R5          @ dividend -= divisor
    ADD R0, R0, #1          @ quotient++
    B div_loop
    
div_done:
    @ R0 = quotient, R4 = remainder
    POP {R4, R5, PC}

@ ============================================================
@ void print_int(int n) - แสดงตัวเลขใน decimal
@ ============================================================
print_int:
    PUSH {R4-R8, LR}
    MOV R4, R0              @ R4 = number
    
    @ จัดการ negative numbers
    CMP R4, #0
    BGE print_positive
    
    @ พิมพ์ '-'
    PUSH {R4}
    MOV R0, #'-'
    BL print_char
    POP {R4}
    NEG R4, R4              @ R4 = |n|

print_positive:
    @ แปลงตัวเลขเป็น string (กลับหลัง)
    LDR R5, =num_buf        @ R5 = buffer pointer
    ADD R5, R5, #19         @ ชี้ที่ท้าย buffer
    MOV R6, #0              @ R6 = digit count
    STRB R6, [R5]           @ null terminator
    
    @ กรณี n = 0
    CMP R4, #0
    BNE convert_loop
    MOV R8, #'0'
    STRB R8, [R5, #-1]!
    ADD R6, R6, #1
    B print_digits
    
convert_loop:
    CMP R4, #0
    BEQ print_digits
    
    @ digit = n % 10
    @ n / 10 (software)
    MOV R7, R4              @ R7 = n
    @ หาร R7 ด้วย 10 (ใช้วิธีคูณ magic number)
    @ R8 = n / 10 (approx via: n * 0xCCCCCCCD >> 35)
    @ สำหรับความง่าย ใช้ loop division:
    MOV R8, #0
div10_loop:
    CMP R7, #10
    BLT div10_done
    SUB R7, R7, #10
    ADD R8, R8, #1
    B div10_loop
div10_done:
    @ R7 = remainder (digit), R8 = quotient
    ADD R7, R7, #'0'        @ แปลงเป็น ASCII
    STRB R7, [R5, #-1]!     @ เก็บใน buffer (backwards)
    ADD R6, R6, #1
    MOV R4, R8              @ n = n / 10
    B convert_loop
    
print_digits:
    @ พิมพ์ digits
    MOV R7, #SYS_WRITE
    MOV R0, #STDOUT
    MOV R1, R5              @ pointer to first digit
    MOV R2, R6              @ length
    SWI #0
    
    POP {R4-R8, PC}

@ ============================================================
@ void print_char(char c) - พิมพ์ตัวอักษร 1 ตัว
@ ============================================================
print_char:
    PUSH {R1-R3, LR}
    STRB R0, [SP, #-4]!     @ เก็บ char ลง stack
    MOV R0, #STDOUT
    MOV R1, SP
    MOV R2, #1
    MOV R7, #SYS_WRITE
    SWI #0
    ADD SP, SP, #4          @ ปลด char
    POP {R1-R3, PC}

@ ============================================================
@ void print_newline() - พิมพ์ newline
@ ============================================================
print_newline:
    PUSH {R0-R3, LR}
    MOV R0, #STDOUT
    LDR R1, =msg_newline
    MOV R2, #1
    MOV R7, #SYS_WRITE
    SWI #0
    POP {R0-R3, PC}
```

---

## โปรแกรมตัวอย่างครบถ้วน: Linked List

```asm
@ ============================================================
@ linked_list.s
@ Linked List ด้วย ARM Assembly
@ สาธิตการใช้ stack, subroutines, และ pointers
@ ============================================================
.section .data
@ Node structure:
@   offset 0: data (int)
@   offset 4: next (pointer)
@ Size = 8 bytes per node

@ Static nodes (จะใช้ dynamic allocation ก็ได้)
node1: .word 10, 0    @ data=10, next=NULL
node2: .word 20, 0    @ data=20, next=NULL
node3: .word 30, 0    @ data=30, next=NULL
node4: .word 40, 0    @ data=40, next=NULL

head_ptr: .word 0     @ pointer to head

.section .text
.global _start

_start:
    @ สร้าง linked list: 10 -> 20 -> 30 -> 40 -> NULL
    
    @ เชื่อม nodes
    LDR R0, =node1
    LDR R1, =node2
    STR R1, [R0, #4]        @ node1.next = &node2
    
    LDR R0, =node2
    LDR R1, =node3
    STR R1, [R0, #4]        @ node2.next = &node3
    
    LDR R0, =node3
    LDR R1, =node4
    STR R1, [R0, #4]        @ node3.next = &node4
    
    @ ตั้ง head
    LDR R0, =node1
    LDR R1, =head_ptr
    STR R0, [R1]             @ head = &node1
    
    @ คำนวณ sum ของ linked list
    LDR R0, [R1]             @ R0 = head
    BL list_sum
    @ R0 = 10 + 20 + 30 + 40 = 100
    
    @ หา node ที่ n (0-indexed)
    LDR R1, =head_ptr
    LDR R0, [R1]             @ head
    MOV R1, #2               @ index = 2
    BL list_get              @ R0 = node at index 2
    @ R0 = pointer to node3 (data=30)
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int list_sum(Node *head)
@ คำนวณผลรวมของ linked list
@ R0 = head pointer
@ return R0 = sum
@ ============================================================
list_sum:
    PUSH {R4, R5, LR}
    MOV R4, R0              @ R4 = current = head
    MOV R5, #0              @ R5 = sum = 0
    
sum_loop:
    CMP R4, #0              @ if current == NULL
    BEQ sum_done
    
    LDR R0, [R4, #0]        @ data = current->data
    ADD R5, R5, R0          @ sum += data
    LDR R4, [R4, #4]        @ current = current->next
    B sum_loop
    
sum_done:
    MOV R0, R5
    POP {R4, R5, PC}

@ ============================================================
@ Node* list_get(Node *head, int index)
@ หา node ที่ index (0-indexed)
@ R0 = head, R1 = index
@ return R0 = pointer to node (NULL if out of range)
@ ============================================================
list_get:
    PUSH {R4, R5, LR}
    MOV R4, R0              @ R4 = current = head
    MOV R5, #0              @ R5 = i = 0
    
get_loop:
    CMP R4, #0
    BEQ get_notfound
    
    CMP R5, R1
    BEQ get_found
    
    LDR R4, [R4, #4]        @ current = current->next
    ADD R5, R5, #1
    B get_loop
    
get_found:
    MOV R0, R4              @ return current
    POP {R4, R5, PC}
    
get_notfound:
    MOV R0, #0              @ return NULL
    POP {R4, R5, PC}

@ ============================================================
@ int list_length(Node *head)
@ นับจำนวน nodes
@ ============================================================
list_length:
    PUSH {R4, LR}
    MOV R4, R0              @ current = head
    MOV R0, #0              @ count = 0
    
length_loop:
    CMP R4, #0
    BEQ length_done
    LDR R4, [R4, #4]        @ current = current->next
    ADD R0, R0, #1          @ count++
    B length_loop
    
length_done:
    POP {R4, PC}
```

---

## สรุปตาราง Calling Convention

### ARM32 (AAPCS) Quick Reference

```
┌──────────────────────────────────────────────────────────────┐
│ ARM32 AAPCS Quick Reference                                    │
├─────────────────┬───────────────────────────────────────────┤
│ Registers       │ Usage                                       │
├─────────────────┼───────────────────────────────────────────┤
│ R0              │ Arg1, Return value (32-bit)                 │
│ R0:R1           │ Return value (64-bit)                       │
│ R1              │ Arg2                                        │
│ R2              │ Arg3                                        │
│ R3              │ Arg4                                        │
│ R4-R11          │ Callee-saved (ต้องบันทึกและคืน)            │
│ R12 (IP)        │ Intra-procedure scratch                     │
│ R13 (SP)        │ Stack Pointer                               │
│ R14 (LR)        │ Link Register                               │
│ R15 (PC)        │ Program Counter                             │
├─────────────────┼───────────────────────────────────────────┤
│ Arg5+           │ Pushed on stack (right-to-left order)       │
│ Stack alignment │ 4-byte (8-byte for public interface)        │
│ Large return    │ Hidden pointer in R0, args shift to R1+     │
│ Small return    │ ≤ 4 bytes in R0, ≤ 8 bytes in R0:R1        │
└─────────────────┴───────────────────────────────────────────┘
```

### AArch64 (AAPCS64) Quick Reference

```
┌──────────────────────────────────────────────────────────────┐
│ AArch64 AAPCS64 Quick Reference                                │
├─────────────────┬───────────────────────────────────────────┤
│ Registers       │ Usage                                       │
├─────────────────┼───────────────────────────────────────────┤
│ X0-X7           │ Args 1-8, Return value                      │
│ X8              │ Indirect result location (hidden struct ptr) │
│ X9-X15          │ Scratch (caller-saved)                      │
│ X16-X17 (IP0,1) │ Intra-procedure scratch                     │
│ X18             │ Platform register (avoid)                   │
│ X19-X28         │ Callee-saved                                │
│ X29 (FP)        │ Frame Pointer                               │
│ X30 (LR)        │ Link Register                               │
│ SP              │ Stack Pointer                               │
├─────────────────┬───────────────────────────────────────────┤
│ NEON/FP Args    │ V0-V7 (D0-D7 / S0-S7)                      │
│ NEON callee-sv  │ V8-V15 (lower 64 bits only)                 │
│ Stack alignment │ 16-byte ALWAYS                              │
│ Arg9+           │ On stack                                    │
└─────────────────┴───────────────────────────────────────────┘
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Stack Operations พื้นฐาน

```asm
@ ============================================================
@ exercise1_template.s
@ แบบฝึกหัด 1: Stack Operations
@ 
@ ให้เขียนฟังก์ชัน reverse_array(int *arr, int len)
@ ที่กลับลำดับ array โดยใช้ stack
@ ============================================================
.section .data
test_array: .word 1, 2, 3, 4, 5

.section .text
.global _start

_start:
    LDR R0, =test_array     @ R0 = pointer to array
    MOV R1, #5              @ R1 = length
    BL reverse_array
    @ ผลลัพธ์: [5, 4, 3, 2, 1]
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ TODO: เขียนฟังก์ชัน reverse_array
@ คำใบ้: push ทุกตัวลง stack แล้ว pop คืนกลับ
@ ============================================================
reverse_array:
    PUSH {R4-R7, LR}
    MOV R4, R0              @ R4 = array pointer
    MOV R5, R1              @ R5 = length
    
    @ ขั้นตอนที่ 1: push ทุกตัวลง stack
    MOV R6, #0
push_loop:
    CMP R6, R5
    BGE push_done
    LDR R7, [R4, R6, LSL #2]  @ โหลด arr[i]
    PUSH {R7}                   @ push ลง stack
    ADD R6, R6, #1
    B push_loop
    
push_done:
    @ ขั้นตอนที่ 2: pop กลับ (กลับลำดับ)
    MOV R6, #0
pop_loop:
    CMP R6, R5
    BGE pop_done
    POP {R7}                    @ pop จาก stack
    STR R7, [R4, R6, LSL #2]   @ arr[i] = popped value
    ADD R6, R6, #1
    B pop_loop
    
pop_done:
    POP {R4-R7, PC}
```

### แบบฝึกหัดที่ 2: Recursive Power Function

```asm
@ ============================================================
@ exercise2.s
@ แบบฝึกหัด 2: Recursive Power (base^exp)
@ เขียนฟังก์ชัน int power(int base, int exp)
@ ============================================================
.section .text
.global _start
.global power

_start:
    @ ทดสอบ power(2, 10) = 1024
    MOV R0, #2
    MOV R1, #10
    BL power
    @ R0 = 1024
    
    @ ทดสอบ power(3, 5) = 243
    MOV R0, #3
    MOV R1, #5
    BL power
    @ R0 = 243
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int power(int base, int exp)
@ ============================================================
power:
    PUSH {R4, R5, LR}
    MOV R4, R0          @ R4 = base
    MOV R5, R1          @ R5 = exp
    
    @ Base case: exp == 0
    CMP R5, #0
    MOVEQ R0, #1        @ return 1
    BEQ power_done
    
    @ Recursive: base * power(base, exp-1)
    SUB R1, R5, #1      @ exp - 1
    MOV R0, R4          @ base
    BL power            @ R0 = power(base, exp-1)
    
    MUL R0, R4, R0      @ return base * power(base, exp-1)
    
power_done:
    POP {R4, R5, PC}
```

### แบบฝึกหัดที่ 3: Bubble Sort

```asm
@ ============================================================
@ exercise3.s
@ แบบฝึกหัด 3: Bubble Sort ด้วย Subroutines
@ ============================================================
.section .data
sort_array: .word 64, 34, 25, 12, 22, 11, 90

.section .text
.global _start

_start:
    LDR R0, =sort_array
    MOV R1, #7              @ length
    BL bubble_sort
    @ ผลลัพธ์: [11, 12, 22, 25, 34, 64, 90]
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ void bubble_sort(int *arr, int len)
@ ============================================================
bubble_sort:
    PUSH {R4-R8, LR}
    MOV R4, R0              @ R4 = arr
    MOV R5, R1              @ R5 = len
    
outer_loop:
    SUBS R5, R5, #1         @ len--
    BLE sort_done
    
    MOV R6, #0              @ i = 0
inner_loop:
    CMP R6, R5              @ if i >= len-1
    BGE outer_loop
    
    @ โหลด arr[i] และ arr[i+1]
    LSL R7, R6, #2
    LDR R0, [R4, R7]        @ R0 = arr[i]
    ADD R7, R7, #4
    LDR R1, [R4, R7]        @ R1 = arr[i+1]
    
    @ เปรียบเทียบ และ swap ถ้าจำเป็น
    CMP R0, R1
    BLE no_swap
    
    @ swap arr[i] และ arr[i+1]
    LSL R8, R6, #2
    STR R1, [R4, R8]        @ arr[i] = arr[i+1]
    ADD R8, R8, #4
    STR R0, [R4, R8]        @ arr[i+1] = arr[i]
    
no_swap:
    ADD R6, R6, #1          @ i++
    B inner_loop
    
sort_done:
    POP {R4-R8, PC}
```

### แบบฝึกหัดที่ 4: String Operations

```asm
@ ============================================================
@ exercise4.s
@ แบบฝึกหัด 4: String Functions ด้วย Stack/Subroutines
@ ============================================================
.section .data
str1: .ascii "Hello, ARM!\0"
str2: .space 50

.section .text
.global _start

_start:
    @ strlen("Hello, ARM!") = 11
    LDR R0, =str1
    BL my_strlen
    @ R0 = 11
    
    @ strcpy(str2, str1)
    LDR R0, =str2
    LDR R1, =str1
    BL my_strcpy
    
    @ strrev(str2) - กลับ string
    LDR R0, =str2
    BL my_strrev
    
    MOV R7, #1
    SWI #0

@ ============================================================
@ int my_strlen(const char *s)
@ ============================================================
my_strlen:
    PUSH {R4, LR}
    MOV R4, R0              @ R4 = s
    MOV R0, #0              @ R0 = count = 0
    
strlen_loop:
    LDRB R1, [R4], #1       @ R1 = *s++
    CMP R1, #0
    BEQ strlen_done
    ADD R0, R0, #1
    B strlen_loop
    
strlen_done:
    POP {R4, PC}

@ ============================================================
@ void my_strcpy(char *dst, const char *src)
@ ============================================================
my_strcpy:
    PUSH {R4, R5, LR}
    MOV R4, R0              @ R4 = dst
    MOV R5, R1              @ R5 = src
    
strcpy_loop:
    LDRB R0, [R5], #1       @ R0 = *src++
    STRB R0, [R4], #1       @ *dst++ = R0
    CMP R0, #0
    BNE strcpy_loop
    
    POP {R4, R5, PC}

@ ============================================================
@ void my_strrev(char *s) - กลับ string in-place
@ ============================================================
my_strrev:
    PUSH {R4-R7, LR}
    MOV R4, R0              @ R4 = start pointer
    
    @ หา end pointer
    MOV R0, R4
    BL my_strlen
    @ R0 = length
    
    ADD R5, R4, R0          @ R5 = end = s + len
    SUB R5, R5, #1          @ R5 = s + len - 1 (last char)
    
    @ กลับ string ด้วย two-pointer technique
reverse_loop:
    CMP R4, R5
    BGE reverse_done
    
    LDRB R6, [R4]           @ R6 = *start
    LDRB R7, [R5]           @ R7 = *end
    STRB R7, [R4]           @ *start = *end
    STRB R6, [R5]           @ *end = *start
    
    ADD R4, R4, #1          @ start++
    SUB R5, R5, #1          @ end--
    B reverse_loop
    
reverse_done:
    POP {R4-R7, PC}
```

### แบบฝึกหัดที่ 5: AArch64 Calling Convention

```asm
// ============================================================
// exercise5_aarch64.s
// แบบฝึกหัด 5: AArch64 Calling Convention
// เขียน function sum_array(int *arr, int len) ใน AArch64
// ============================================================
.section .data
test_arr: .word 1, 2, 3, 4, 5, 6, 7, 8, 9, 10

.section .text
.global _start

_start:
    LDR X0, =test_arr       // X0 = pointer
    MOV W1, #10             // W1 = len = 10
    BL sum_array_a64
    // X0 = 55
    
    MOV X0, #0
    MOV X8, #93
    SVC #0

// ============================================================
// int sum_array_a64(int *arr, int len)
// X0 = arr pointer, W1 = len
// ============================================================
sum_array_a64:
    STP X29, X30, [SP, #-32]!   // บันทึก FP, LR, จัดสรร space
    MOV X29, SP
    STP X19, X20, [SP, #16]     // บันทึก callee-saved
    
    MOV X19, X0                  // X19 = arr
    MOV W20, W1                  // W20 = len
    
    MOV W0, #0                   // sum = 0
    MOV W9, #0                   // i = 0
    
sum_loop_a64:
    CMP W9, W20
    BGE sum_done_a64
    
    LDR W10, [X19, X9, LSL #2]  // W10 = arr[i]
    ADD W0, W0, W10              // sum += arr[i]
    ADD W9, W9, #1               // i++
    B sum_loop_a64
    
sum_done_a64:
    LDP X19, X20, [SP, #16]
    LDP X29, X30, [SP], #32
    RET
```

---

## สรุปบทเรียน (Summary)

### สิ่งที่เรียนรู้ในบทนี้

1. **Full Descending Stack**: ARM Linux ใช้ stack แบบนี้ โดย SP ชี้ที่ข้อมูลล่าสุด และ stack เติบโตลงมา

2. **STMFD/LDMFD = PUSH/POP**: คำสั่งเก่าและใหม่มีความหมายเหมือนกัน

3. **AAPCS Calling Convention**:
   - R0-R3: Arguments (4 ตัวแรก) และ Return value
   - R4-R11: Callee-saved (ต้องบันทึกก่อนใช้)
   - R12: Scratch (caller-saved)
   - LR: Return address (ต้องบันทึกถ้าเรียกฟังก์ชันอื่น)

4. **Stack Frame**: โครงสร้าง PUSH {R4-R11, LR} / LDMFD SP!, {R4-R11, PC}

5. **Nested Calls**: ต้องบันทึก LR ใน stack เสมอเมื่อมี BL

6. **Local Variables**: จัดสรรด้วย SUB SP, SP, #n และปลดด้วย ADD SP, SP, #n

7. **VFP Registers**: บันทึก/คืนด้วย VPUSH/VPOP

8. **Parameters > 4**: ส่งผ่าน stack, callee โหลดจาก [SP + offset]

9. **Large Struct Return**: ใช้ hidden pointer ใน R0 (ARM32) หรือ X8 (AArch64)

10. **AArch64**: Stack ต้อง 16-byte aligned, ใช้ STP/LDP, X0-X7 สำหรับ args

### คำสั่งสำคัญ

```
ARM32:
PUSH {regs}         = STMFD SP!, {regs}
POP  {regs}         = LDMFD SP!, {regs}
POP  {regs, PC}     = return from function
BX LR               = return from leaf function
BL func             = call function (LR = return addr)
VPUSH {Sn-Sm}       = push VFP registers
VPOP  {Sn-Sm}       = pop VFP registers

AArch64:
STP Xn, Xm, [SP, #-16]!    = push two registers
LDP Xn, Xm, [SP], #16      = pop two registers
RET                          = return (BR X30)
BL func                      = call function (X30 = LR)
```

---

*Part 035 จบแล้ว → ต่อด้วย Part 036: ARM Exceptions และ Interrupt Handling*
