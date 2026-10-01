# Part 031: ARM Registers และ CPSR (Current Program Status Register)

## บทนำ (Introduction)

ในบทนี้เราจะศึกษา **Registers** ของสถาปัตยกรรม ARM อย่างละเอียด ทั้ง ARM32 (AArch32) และ AArch64
Registers คือหน่วยความจำความเร็วสูงที่อยู่ภายใน CPU โดยตรง ซึ่งเป็นหัวใจสำคัญของการเขียนโปรแกรม Assembly

---

## 1. ARM32 General-Purpose Registers (R0-R15)

### 1.1 ภาพรวมของ Registers ใน ARM32

ARM32 มี registers ทั่วไป 16 ตัว ตั้งแต่ R0 ถึง R15 แต่ละตัวมีขนาด 32 บิต

```
┌─────────────────────────────────────────────────────────────┐
│              ARM32 Register File                            │
├──────────┬─────────────┬───────────────────────────────────┤
│ Register │ Alias        │ คำอธิบาย                          │
├──────────┼─────────────┼───────────────────────────────────┤
│ R0       │ a1          │ Argument/Result register 1        │
│ R1       │ a2          │ Argument/Result register 2        │
│ R2       │ a3          │ Argument register 3               │
│ R3       │ a4          │ Argument register 4               │
│ R4       │ v1          │ Variable register 1 (callee-save) │
│ R5       │ v2          │ Variable register 2 (callee-save) │
│ R6       │ v3          │ Variable register 3 (callee-save) │
│ R7       │ v4/SYSCALL  │ Variable register 4 / Syscall num │
│ R8       │ v5          │ Variable register 5 (callee-save) │
│ R9       │ v6/SB/TR    │ Variable register 6               │
│ R10      │ v7/SL       │ Variable register 7               │
│ R11      │ v8/FP       │ Frame Pointer                     │
│ R12      │ IP          │ Intra-Procedure-call scratch reg  │
│ R13      │ SP          │ Stack Pointer                     │
│ R14      │ LR          │ Link Register                     │
│ R15      │ PC          │ Program Counter                   │
└──────────┴─────────────┴───────────────────────────────────┘
```

### 1.2 AAPCS - ARM Application Procedure Call Standard

```asm
@ ====================================================
@ ไฟล์: arm32_registers_demo.s
@ อธิบาย: การใช้งาน Registers ใน ARM32
@ คอมไพล์: arm-linux-gnueabi-as -o arm32_registers_demo.o arm32_registers_demo.s
@           arm-linux-gnueabi-ld -o arm32_registers_demo arm32_registers_demo.o
@ ====================================================

.section .data
msg_hello:    .ascii "Hello from ARM32!\n"
msg_hello_len = . - msg_hello

msg_r0:       .ascii "R0 = Argument/Return register\n"
msg_r0_len = . - msg_r0

.section .text
.global _start

_start:
    @ --- แสดงการใช้ R0-R3 เป็น Arguments ---
    @ R0 = first argument (file descriptor)
    @ R1 = second argument (buffer pointer)
    @ R2 = third argument (length)
    @ R7 = syscall number (Linux ARM)
    
    mov r7, #4              @ syscall write (sys_write = 4)
    mov r0, #1              @ file descriptor = 1 (stdout)
    ldr r1, =msg_hello      @ โหลดที่อยู่ของ string
    mov r2, #18             @ ความยาว string
    swi #0                  @ เรียก system call (Software Interrupt)
    
    @ --- ทดสอบ R4-R11 (callee-saved registers) ---
    @ ตาม AAPCS: ฟังก์ชันต้องเก็บค่า R4-R11 ก่อนใช้ และคืนค่าเดิมก่อน return
    
    push {r4, r5, r6, r7, r8}   @ บันทึก callee-saved registers
    
    mov r4, #100            @ เก็บค่าไว้ใน callee-saved register
    mov r5, #200
    mov r6, #300
    
    @ ทำการคำนวณ
    add r0, r4, r5          @ r0 = r4 + r5 = 300
    mul r8, r6, r4          @ r8 = r6 * r4 = 30000
    
    pop {r4, r5, r6, r7, r8}    @ คืนค่า callee-saved registers
    
    @ --- แสดงการใช้ R12 (IP - Intra-Procedure scratch) ---
    @ R12 ใช้เป็น scratch register ระหว่าง procedure call
    @ ไม่จำเป็นต้องเก็บค่า (caller-saved)
    ldr r12, =0xDEADBEEF    @ ใช้ร่วมกับ linker veneer
    
    @ --- จบโปรแกรม ---
    mov r7, #1              @ syscall exit
    mov r0, #0              @ exit code = 0
    swi #0
```

### 1.3 Special Purpose Registers

```asm
@ ====================================================
@ ไฟล์: arm32_special_regs.s
@ อธิบาย: Special Purpose Registers: SP, LR, PC
@ ====================================================

.section .data
result_msg: .ascii "Sum = "
result_len = . - result_msg

.section .text
.global _start

@ ----------------------------------------
@ ฟังก์ชัน: add_numbers
@ Input: R0 = a, R1 = b
@ Output: R0 = a + b
@ ----------------------------------------
add_numbers:
    @ LR (R14) ถูก set โดย BL instruction โดยอัตโนมัติ
    @ เก็บ return address ไว้ใน LR
    
    @ ถ้าฟังก์ชันนี้ไม่เรียกฟังก์ชันอื่น ก็ไม่ต้อง push LR
    add r0, r0, r1          @ r0 = r0 + r1
    
    @ BX LR = Branch to address in LR (return)
    bx lr                   @ กลับไปที่ผู้เรียก

@ ----------------------------------------
@ ฟังก์ชัน: nested_function (เรียกฟังก์ชันอื่น)
@ Input: R0 = a, R1 = b, R2 = c
@ Output: R0 = (a + b) + c
@ ----------------------------------------
nested_function:
    @ ต้อง push LR เพราะจะใช้ BL ซึ่งจะเขียนทับ LR
    push {lr}               @ บันทึก return address
    
    @ เรียก add_numbers(a, b)
    bl add_numbers           @ LR = PC+4 (next instruction address)
    
    @ ตอนนี้ R0 = a + b
    @ R2 ยังคงมีค่า c
    add r0, r0, r2          @ R0 = (a+b) + c
    
    pop {pc}                @ คืนค่า LR -> PC (return)
    @ เทียบเท่า: pop {lr}; bx lr

_start:
    @ ทดสอบ LR (Link Register)
    @ ก่อน BL: LR ไม่มีความหมาย
    @ หลัง BL: LR = address ถัดจาก BL instruction
    
    mov r0, #10             @ argument a = 10
    mov r1, #20             @ argument b = 20
    bl add_numbers          @ เรียกฟังก์ชัน, LR จะถูก set
    @ หลังจาก return: R0 = 30
    
    @ ทดสอบ nested function
    mov r0, #5
    mov r1, #6
    mov r2, #7
    bl nested_function      @ R0 = 18
    
    @ --- สาธิต SP (Stack Pointer) ---
    @ SP ชี้ไปที่ top of stack (ARM ใช้ full-descending stack)
    @ PUSH ลด SP ก่อน แล้วเขียนข้อมูล
    @ POP อ่านข้อมูล แล้วเพิ่ม SP
    
    mov r1, #0xAAAA         @ ข้อมูลทดสอบ
    mov r2, #0xBBBB
    mov r3, #0xCCCC
    
    push {r1, r2, r3}       @ SP -= 12, เขียน R1, R2, R3 ลง stack
    @ Stack layout: [SP+8]=R3, [SP+4]=R2, [SP+0]=R1
    
    @ อ่านค่าจาก stack โดยตรง
    ldr r4, [sp]            @ R4 = R1 (0xAAAA)
    ldr r5, [sp, #4]        @ R5 = R2 (0xBBBB)
    ldr r6, [sp, #8]        @ R6 = R3 (0xCCCC)
    
    pop {r1, r2, r3}        @ SP += 12, คืนค่า
    
    @ --- สาธิต PC (Program Counter) ---
    @ PC ชี้ไปที่ current instruction + 8 (เนื่องจาก pipeline)
    @ ใน ARM32: PC = address of current instruction + 8
    @ ใน Thumb: PC = address of current instruction + 4
    
    @ การอ่าน PC:
    mov r0, pc              @ R0 = address ของ instruction นี้ + 8
    sub r0, r0, #8          @ R0 = address จริงๆ ของ instruction ข้างบน
    
    @ การใช้ PC สำหรับ position-independent code:
    ldr r1, [pc, #8]        @ อ่านค่าจาก address ที่อยู่ห่าง 8 bytes จาก PC
    
    @ จบโปรแกรม
    mov r7, #1
    mov r0, #0
    swi #0
    
    @ ข้อมูล literal pool (ข้อมูลที่ฝังไว้ใน code section)
    .word 0x12345678        @ ข้อมูลที่ ldr ข้างบนอ่าน
```

---

## 2. CPSR - Current Program Status Register

### 2.1 โครงสร้างของ CPSR

CPSR เป็น register พิเศษขนาด 32 บิตที่เก็บสถานะปัจจุบันของ processor

```
CPSR Bit Layout:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 31 │ 30 │ 29 │ 28 │ 27 │ .. │  9 │  8 │  7 │  6 │  5 │  4 │  3 │  2 │  1 │  0 │
│  N │  Z │  C │  V │  Q │    │  E │  A │  I │  F │  T │ M4 │ M3 │ M2 │ M1 │ M0 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

N = Negative flag      (bit 31)
Z = Zero flag          (bit 30)
C = Carry flag         (bit 29)
V = Overflow flag      (bit 28)
Q = Saturation flag    (bit 27) - DSP extensions
E = Endianness bit     (bit 9)  - 0=little, 1=big
A = Async abort disable (bit 8)
I = IRQ disable        (bit 7)
F = FIQ disable        (bit 6)
T = Thumb state bit    (bit 5)  - 0=ARM, 1=Thumb
M[4:0] = Mode bits     (bits 4:0)
```

### 2.2 Condition Flags (N, Z, C, V)

```asm
@ ====================================================
@ ไฟล์: arm32_cpsr_flags.s
@ อธิบาย: CPSR Condition Flags N, Z, C, V
@ ====================================================

.section .data
flag_n_msg:   .ascii "N (Negative) flag set\n\0"
flag_z_msg:   .ascii "Z (Zero) flag set\n\0"
flag_c_msg:   .ascii "C (Carry) flag set\n\0"
flag_v_msg:   .ascii "V (Overflow) flag set\n\0"
no_flag_msg:  .ascii "No flags set\n\0"

.section .text
.global _start

@ ----------------------------------------
@ สาธิต N Flag (Negative)
@ N = 1 เมื่อผลลัพธ์เป็นลบ (bit 31 = 1)
@ ----------------------------------------
demo_n_flag:
    @ การลบที่ได้ผลลัพธ์เป็นลบ
    mov r0, #5              @ r0 = 5
    mov r1, #10             @ r1 = 10
    subs r2, r0, r1         @ r2 = 5 - 10 = -5, N=1, Z=0, C=0, V=0
    @ หลังจากนี้ N flag = 1
    
    @ ตรวจสอบ N flag
    bpl positive_result     @ branch if plus (N=0) -> ข้ามไปถ้าผลบวก
    @ ถ้ามาถึงที่นี่ N=1 (ผลลัพธ์เป็นลบ)
    bx lr
    
positive_result:
    @ N=0 (ผลลัพธ์เป็นบวก)
    bx lr

@ ----------------------------------------
@ สาธิต Z Flag (Zero)
@ Z = 1 เมื่อผลลัพธ์เป็น 0
@ ----------------------------------------
demo_z_flag:
    mov r0, #42
    subs r1, r0, #42        @ r1 = 42 - 42 = 0, Z=1
    @ Z flag = 1
    
    @ ตรวจสอบ Z flag
    beq equal               @ branch if equal (Z=1)
    bne not_equal           @ branch if not equal (Z=0)
    
equal:
    bx lr
not_equal:
    bx lr

@ ----------------------------------------
@ สาธิต C Flag (Carry)
@ C = 1 เมื่อ unsigned addition มี carry out
@ C = 0 เมื่อ unsigned subtraction มี borrow
@ ----------------------------------------
demo_c_flag:
    @ Carry ใน addition:
    mov r0, #0xFFFFFFFF     @ max unsigned 32-bit value
    adds r1, r0, #1         @ 0xFFFFFFFF + 1 = 0x100000000 (ล้น), C=1
    @ C flag = 1, r1 = 0
    
    bcs carry_set           @ branch if carry set (C=1)
    bcc carry_clear         @ branch if carry clear (C=0)
    
    @ Carry ใน subtraction (ใช้เป็น NOT-borrow):
    mov r2, #10
    subs r3, r2, #5         @ 10 - 5 = 5, ไม่มี borrow, C=1
    
    mov r4, #5
    subs r5, r4, #10        @ 5 - 10 = -5, มี borrow, C=0
    
carry_set:
    bx lr
carry_clear:
    bx lr

@ ----------------------------------------
@ สาธิต V Flag (Overflow)
@ V = 1 เมื่อ signed arithmetic overflow
@ ----------------------------------------
demo_v_flag:
    @ Positive + Positive = Negative (overflow!)
    mov r0, #0x7FFFFFFF     @ max signed 32-bit = 2,147,483,647
    adds r1, r0, #1         @ 0x7FFFFFFF + 1 = 0x80000000 (ล้นไป negative)
    @ V = 1 (signed overflow!)
    
    bvs overflow_set        @ branch if overflow set (V=1)
    bvc overflow_clear      @ branch if overflow clear (V=0)
    
    @ Negative + Negative = Positive (overflow!)
    mov r2, #0x80000000     @ min signed = -2,147,483,648
    subs r3, r2, #1         @ -2147483648 - 1 = overflow!
    @ V = 1
    
overflow_set:
    bx lr
overflow_clear:
    bx lr

_start:
    @ ========================================
    @ ทดสอบ N, Z, C, V flags อย่างละเอียด
    @ ========================================
    
    @ --- TEST 1: Zero flag ---
    mov r0, #100
    subs r1, r0, #100       @ 100 - 100 = 0
    @ CPSR: Z=1, N=0, C=1 (no borrow), V=0
    
    @ อ่าน CPSR โดยตรง (ต้องใช้ MRS)
    mrs r10, cpsr           @ r10 = CPSR value
    @ ตรวจสอบ Z bit (bit 30)
    tst r10, #(1 << 30)     @ ทดสอบ bit 30
    bne z_is_set
    b z_not_set
z_is_set:
    @ Z flag = 1
    nop
z_not_set:
    
    @ --- TEST 2: Negative flag ---
    mov r0, #5
    subs r1, r0, #10        @ 5 - 10 = -5
    @ CPSR: N=1, Z=0, C=0, V=0
    
    mrs r10, cpsr
    tst r10, #(1 << 31)     @ ตรวจสอบ N bit (bit 31)
    bne n_is_set
n_is_set:
    
    @ --- TEST 3: Carry flag (unsigned overflow) ---
    ldr r0, =0xFFFFFFFF
    adds r1, r0, #2         @ 0xFFFFFFFF + 2 = overflow
    @ CPSR: C=1, Z=0, N=0, V=0
    
    mrs r10, cpsr
    tst r10, #(1 << 29)     @ ตรวจสอบ C bit (bit 29)
    bne c_is_set
c_is_set:
    
    @ --- TEST 4: Overflow flag (signed overflow) ---
    ldr r0, =0x7FFFFFFF     @ max signed int
    adds r1, r0, #1         @ signed overflow!
    @ CPSR: V=1, C=1, N=1, Z=0
    
    mrs r10, cpsr
    tst r10, #(1 << 28)     @ ตรวจสอบ V bit (bit 28)
    bne v_is_set
v_is_set:
    
    @ จบโปรแกรม
    mov r7, #1
    mov r0, #0
    swi #0
```

### 2.3 การใช้ Condition Flags กับ Conditional Execution

```asm
@ ====================================================
@ ไฟล์: arm32_conditional_exec.s
@ อธิบาย: Conditional Execution ใน ARM32
@ ARM สามารถทำ conditional execution กับทุก instruction
@ ====================================================

.section .text
.global _start

@ ----------------------------------------
@ ARM Condition Codes:
@ EQ - Equal (Z=1)
@ NE - Not Equal (Z=0)
@ CS/HS - Carry Set / Unsigned Higher or Same (C=1)
@ CC/LO - Carry Clear / Unsigned Lower (C=0)
@ MI - Minus / Negative (N=1)
@ PL - Plus / Positive or Zero (N=0)
@ VS - Overflow Set (V=1)
@ VC - Overflow Clear (V=0)
@ HI - Unsigned Higher (C=1 and Z=0)
@ LS - Unsigned Lower or Same (C=0 or Z=1)
@ GE - Signed Greater or Equal (N=V)
@ LT - Signed Less Than (N!=V)
@ GT - Signed Greater Than (Z=0 and N=V)
@ LE - Signed Less or Equal (Z=1 or N!=V)
@ AL - Always (default)
@ ----------------------------------------

@ ฟังก์ชัน: absolute_value (ค่าสัมบูรณ์)
@ Input: R0 = number
@ Output: R0 = |number|
abs_value:
    cmp r0, #0              @ เปรียบเทียบกับ 0
    @ ถ้า r0 < 0 (N=1, V=0): ทำ negation
    rsblt r0, r0, #0        @ R0 = 0 - R0 (ถ้า LessThan)
    @ rsblt = RSB (Reverse Subtract) with LT condition
    bx lr

@ ฟังก์ชัน: max_value
@ Input: R0 = a, R1 = b
@ Output: R0 = max(a, b)
max_value:
    cmp r0, r1              @ เปรียบเทียบ a กับ b
    movlt r0, r1            @ ถ้า a < b: R0 = b
    @ movlt = MOV with LT condition
    bx lr

@ ฟังก์ชัน: min_value
@ Input: R0 = a, R1 = b
@ Output: R0 = min(a, b)
min_value:
    cmp r0, r1
    movgt r0, r1            @ ถ้า a > b: R0 = b
    bx lr

@ ฟังก์ชัน: clamp (จำกัดค่าในช่วง)
@ Input: R0 = value, R1 = min, R2 = max
@ Output: R0 = clamp(value, min, max)
clamp_value:
    cmp r0, r1              @ value vs min
    movlt r0, r1            @ ถ้า value < min: value = min
    cmp r0, r2              @ value vs max
    movgt r0, r2            @ ถ้า value > max: value = max
    bx lr

@ ฟังก์ชัน: sign_of_number
@ Input: R0 = number
@ Output: R0 = -1 ถ้า negative, 0 ถ้า zero, 1 ถ้า positive
sign_of:
    cmp r0, #0
    movgt r0, #1            @ ถ้า > 0: return 1
    movlt r0, #-1           @ ถ้า < 0: return -1
    @ ถ้า == 0: R0 ยังคงเป็น 0
    bx lr

@ ฟังก์ชัน: is_between (ตรวจสอบว่าอยู่ในช่วง)
@ Input: R0 = value, R1 = low, R2 = high
@ Output: R0 = 1 ถ้าอยู่ในช่วง [low, high], 0 ถ้าไม่อยู่
is_between:
    cmp r0, r1              @ value >= low?
    blt not_in_range        @ ถ้า value < low: ออก
    cmp r0, r2              @ value <= high?
    bgt not_in_range        @ ถ้า value > high: ออก
    mov r0, #1              @ อยู่ในช่วง
    bx lr
not_in_range:
    mov r0, #0
    bx lr

@ ----------------------------------------
@ Conditional execution ช่วยลด branches
@ เปรียบเทียบกับ if-else ปกติ
@ ----------------------------------------
conditional_vs_branch:
    @ วิธีที่ 1: ใช้ branch (แบบดั้งเดิม)
    cmp r0, r1
    bne skip_move
    mov r2, #100
skip_move:
    
    @ วิธีที่ 2: ใช้ conditional move (ดีกว่า - ไม่มี branch penalty)
    cmp r0, r1
    moveq r2, #100          @ ทำงานเฉพาะเมื่อ EQ
    
    bx lr

_start:
    @ --- ทดสอบ abs_value ---
    mov r0, #-42
    bl abs_value            @ ควรได้ R0 = 42
    
    mov r0, #100
    bl abs_value            @ ควรได้ R0 = 100
    
    @ --- ทดสอบ max_value ---
    mov r0, #15
    mov r1, #25
    bl max_value            @ ควรได้ R0 = 25
    
    @ --- ทดสอบ clamp ---
    mov r0, #-10
    mov r1, #0
    mov r2, #100
    bl clamp_value          @ ควรได้ R0 = 0 (clamp to min)
    
    mov r0, #150
    mov r1, #0
    mov r2, #100
    bl clamp_value          @ ควรได้ R0 = 100 (clamp to max)
    
    mov r0, #50
    mov r1, #0
    mov r2, #100
    bl clamp_value          @ ควรได้ R0 = 50 (ไม่เปลี่ยนแปลง)
    
    @ จบโปรแกรม
    mov r7, #1
    mov r0, #0
    swi #0
```

---

## 3. CPSR Control Bits

### 3.1 I, F, T Bits และ Mode Bits

```asm
@ ====================================================
@ ไฟล์: arm32_cpsr_control.s
@ อธิบาย: CPSR Control Bits (I, F, T, Mode)
@ หมายเหตุ: ต้องรันในโหมด privileged (kernel mode)
@ ====================================================

.section .text
.global _start

@ ----------------------------------------
@ CPSR Mode Bits (M[4:0]):
@ 0b10000 (0x10) = User mode (USR)
@ 0b10001 (0x11) = FIQ mode
@ 0b10010 (0x12) = IRQ mode
@ 0b10011 (0x13) = Supervisor mode (SVC)
@ 0b10111 (0x17) = Abort mode (ABT)
@ 0b11011 (0x1B) = Undefined mode (UND)
@ 0b11111 (0x1F) = System mode (SYS)
@ ----------------------------------------

.equ MODE_USR, 0x10         @ User mode
.equ MODE_FIQ, 0x11         @ Fast Interrupt mode
.equ MODE_IRQ, 0x12         @ Interrupt mode
.equ MODE_SVC, 0x13         @ Supervisor mode
.equ MODE_ABT, 0x17         @ Abort mode
.equ MODE_UND, 0x1B         @ Undefined mode
.equ MODE_SYS, 0x1F         @ System mode

.equ I_BIT,   0x80          @ IRQ disable bit (bit 7)
.equ F_BIT,   0x40          @ FIQ disable bit (bit 6)
.equ T_BIT,   0x20          @ Thumb state bit (bit 5)

@ ----------------------------------------
@ อ่านค่า CPSR และแสดงโหมดปัจจุบัน
@ ----------------------------------------
read_current_mode:
    mrs r0, cpsr            @ อ่าน CPSR -> R0
    and r1, r0, #0x1F       @ เอาแค่ Mode bits (bits 4:0)
    
    @ ตรวจสอบแต่ละโหมด
    cmp r1, #MODE_USR
    beq mode_is_user
    cmp r1, #MODE_FIQ
    beq mode_is_fiq
    cmp r1, #MODE_IRQ
    beq mode_is_irq
    cmp r1, #MODE_SVC
    beq mode_is_svc
    
mode_is_user:
    @ กำลังรันใน User mode
    bx lr
mode_is_fiq:
    bx lr
mode_is_irq:
    bx lr
mode_is_svc:
    bx lr

@ ----------------------------------------
@ การอ่าน/เขียน I bit (IRQ disable)
@ ใน User mode: ไม่สามารถแก้ไข control bits ได้
@ ต้องอยู่ใน privileged mode
@ ----------------------------------------
disable_irq:
    mrs r0, cpsr            @ อ่าน CPSR
    orr r0, r0, #I_BIT      @ set I bit = disable IRQ
    msr cpsr_c, r0          @ เขียนกลับ (cpsr_c = control bits only)
    bx lr

enable_irq:
    mrs r0, cpsr
    bic r0, r0, #I_BIT      @ clear I bit = enable IRQ
    msr cpsr_c, r0
    bx lr

disable_fiq:
    mrs r0, cpsr
    orr r0, r0, #F_BIT      @ set F bit = disable FIQ
    msr cpsr_c, r0
    bx lr

@ ----------------------------------------
@ T bit: Thumb state
@ ไม่ควรแก้ไขตรงๆ - ใช้ BX instruction แทน
@ BX Rn: ถ้า bit 0 ของ Rn = 1 -> switch to Thumb
@          ถ้า bit 0 ของ Rn = 0 -> switch to ARM
@ ----------------------------------------
switch_to_thumb:
    @ เปลี่ยนไป Thumb mode
    adr r0, thumb_code + 1  @ +1 เพื่อ set bit 0 (Thumb indicator)
    bx r0                   @ switch to Thumb mode

.thumb
thumb_code:
    @ นี่คือ Thumb code
    mov r0, #42
    bx lr                   @ กลับไป ARM mode (ถ้า LR ไม่มี bit 0)
.arm

_start:
    @ อ่าน CPSR ปัจจุบัน
    mrs r0, cpsr
    
    @ ตรวจสอบ T bit (bit 5)
    tst r0, #T_BIT
    bne in_thumb_mode       @ ถ้า T=1: Thumb mode
    @ ถ้ามาถึงที่นี่: ARM mode
    
in_thumb_mode:
    
    @ ตรวจสอบ I bit (bit 7)
    tst r0, #I_BIT
    bne irq_disabled        @ ถ้า I=1: IRQ ถูก disable
    @ IRQ enabled
irq_disabled:
    
    @ ตรวจสอบโหมดปัจจุบัน
    and r1, r0, #0x1F
    cmp r1, #MODE_USR       @ ตรวจว่าอยู่ใน User mode?
    beq is_user_mode
    @ อยู่ใน privileged mode
is_user_mode:
    
    @ จบโปรแกรม
    mov r7, #1
    mov r0, #0
    swi #0
```

---

## 4. SPSR - Saved Program Status Register

### 4.1 บทบาทของ SPSR

```asm
@ ====================================================
@ ไฟล์: arm32_spsr_demo.s
@ อธิบาย: SPSR (Saved Program Status Register)
@ 
@ เมื่อเกิด exception (interrupt, abort, syscall):
@ 1. CPU บันทึก CPSR -> SPSR_<mode>
@ 2. CPU เปลี่ยนโหมดเป็น exception mode
@ 3. CPU เก็บ return address ใน LR_<mode>
@ 4. ทำงาน exception handler
@ 5. ใช้ MOVS PC, LR หรือ SUBS PC, LR, #4 เพื่อ return
@    (MOVS PC, LR คือ copy LR -> PC และ copy SPSR -> CPSR)
@ ====================================================

.section .text
.global _start

@ ----------------------------------------
@ Exception Handler Template
@ ----------------------------------------
irq_handler:
    @ เมื่อ IRQ เกิดขึ้น:
    @ - CPU อยู่ใน IRQ mode
    @ - LR_irq = PC + 4 (address หลัง interrupted instruction)
    @ - SPSR_irq = CPSR ก่อนเกิด interrupt
    
    @ บันทึก registers (LR ต้องลดลง 4 สำหรับ IRQ)
    sub lr, lr, #4          @ adjust return address
    push {r0-r12, lr}       @ บันทึก caller registers + LR
    
    @ ... ทำงาน interrupt handler ...
    
    @ กลับจาก interrupt
    ldm sp!, {r0-r12, pc}^  @ ^ = copy SPSR -> CPSR เมื่อ load PC
    @ เทียบเท่า: pop {r0-r12, lr}; movs pc, lr
    
syscall_handler:
    @ เมื่อ SWI เกิดขึ้น:
    @ - CPU อยู่ใน SVC mode
    @ - LR_svc = PC ของ instruction ถัดจาก SWI
    @ - SPSR_svc = CPSR ก่อนเกิด SWI
    
    push {r0-r12, lr}
    
    @ อ่านหมายเลข syscall จาก SWI instruction
    ldr r0, [lr, #-4]       @ อ่าน SWI instruction
    bic r0, r0, #0xFF000000 @ เอา 24 bits ล่าง = syscall number
    
    @ ... ทำงาน syscall handler ...
    
    ldm sp!, {r0-r12, pc}^  @ return from SVC

@ ----------------------------------------
@ การอ่าน/เขียน SPSR (เฉพาะใน exception mode)
@ ----------------------------------------
read_spsr_example:
    @ ต้องอยู่ใน exception mode (FIQ, IRQ, SVC, ABT, UND)
    mrs r0, spsr            @ อ่าน SPSR ของ mode ปัจจุบัน
    @ r0 ตอนนี้มีค่า CPSR ที่ถูกบันทึกไว้
    
    @ ตรวจสอบว่า code ที่ถูก interrupt อยู่ใน Thumb mode หรือ ARM mode
    tst r0, #0x20           @ T bit
    bne was_in_thumb
    @ was in ARM mode
was_in_thumb:
    
    bx lr

write_spsr_example:
    @ แก้ไข SPSR ก่อน return (เพื่อเปลี่ยน flags ของ caller)
    mrs r0, spsr
    orr r0, r0, #0x80000000 @ set N flag ใน SPSR
    msr spsr_f, r0          @ เขียนเฉพาะ flag bits (spsr_f)
    bx lr

_start:
    @ โปรแกรมนี้แสดงให้เห็นว่า SPSR ทำงานอย่างไร
    @ ในทางปฏิบัติ เราจะจัดการ SPSR ใน exception handlers
    
    @ อ่าน CPSR ปัจจุบัน
    mrs r0, cpsr
    
    @ ใน User mode ไม่สามารถอ่าน SPSR ได้
    @ การพยายามอ่านจะได้ค่าที่ไม่แน่นอน
    
    mov r7, #1
    mov r0, #0
    swi #0
```

---

## 5. Banked Registers

### 5.1 ระบบ Banked Registers ใน ARM32

```
ARM32 Banked Register System:

         USR/SYS  FIQ      IRQ      SVC      ABT      UND
         ─────────────────────────────────────────────────
R0       R0       R0       R0       R0       R0       R0
R1       R1       R1       R1       R1       R1       R1
R2       R2       R2       R2       R2       R2       R2
R3       R3       R3       R3       R3       R3       R3
R4       R4       R4       R4       R4       R4       R4
R5       R5       R5       R5       R5       R5       R5
R6       R6       R6       R6       R6       R6       R6
R7       R7       R7       R7       R7       R7       R7
R8       R8       R8_fiq   R8       R8       R8       R8
R9       R9       R9_fiq   R9       R9       R9       R9
R10      R10      R10_fiq  R10      R10      R10      R10
R11      R11      R11_fiq  R11      R11      R11      R11
R12      R12      R12_fiq  R12      R12      R12      R12
R13(SP)  SP       SP_fiq   SP_irq   SP_svc   SP_abt   SP_und
R14(LR)  LR       LR_fiq   LR_irq   LR_svc   LR_abt   LR_und
R15(PC)  PC       PC       PC       PC       PC       PC
CPSR     CPSR     CPSR     CPSR     CPSR     CPSR     CPSR
SPSR     -        SPSR_fiq SPSR_irq SPSR_svc SPSR_abt SPSR_und

FIQ mode มี banked registers มากที่สุด (R8-R14)
ทำให้ Fast Interrupt handler ไม่ต้อง push/pop registers มาก
```

```asm
@ ====================================================
@ ไฟล์: arm32_banked_regs.s
@ อธิบาย: การเข้าถึง Banked Registers
@ ====================================================

.section .text
.global _start

@ ----------------------------------------
@ ตัวอย่าง FIQ Handler (ใช้ banked registers)
@ ----------------------------------------
fiq_handler:
    @ ใน FIQ mode: R8-R14 เป็น banked registers
    @ ไม่ต้อง push R8-R12 เพราะเป็น private ของ FIQ mode
    
    @ ทำงานโดยตรงโดยไม่ต้อง save/restore
    mov r8, #0              @ R8_fiq (ไม่กระทบ R8 ของ user mode)
    mov r9, #0              @ R9_fiq
    
    @ อ่านข้อมูลจาก hardware (ตัวอย่าง)
    @ ldr r8, =UART_DATA_REG @ โหลดที่อยู่ UART
    @ ldr r9, [r8]           @ อ่านข้อมูล
    
    @ Return จาก FIQ
    @ LR_fiq ถูก set เป็น PC+4 โดย hardware
    subs pc, lr, #4         @ return from FIQ interrupt
    @ SUBS กับ PC: copy SPSR_fiq -> CPSR (restore เป็น user mode)

@ ----------------------------------------
@ IRQ Handler (ต้อง save R0-R12 เพราะ shared)
@ ----------------------------------------
irq_handler_banked:
    @ ใน IRQ mode: มีแค่ SP_irq และ LR_irq ที่เป็น banked
    @ R0-R12 เป็น shared กับ user mode
    
    sub lr, lr, #4          @ adjust return address
    push {r0-r12, lr}       @ ต้อง save เพราะ shared
    
    @ ... ทำงาน IRQ handler ...
    
    ldm sp!, {r0-r12, pc}^  @ restore และ return

_start:
    @ โปรแกรมนี้เป็น user mode ทั่วไป
    @ เราไม่สามารถเปลี่ยนโหมดเองได้ใน user mode
    
    @ อ่าน CPSR เพื่อดูโหมดปัจจุบัน
    mrs r0, cpsr
    and r1, r0, #0x1F       @ เอา mode bits
    
    @ r1 ควรเป็น 0x10 = User mode
    
    mov r7, #1
    mov r0, #0
    swi #0
```

---

## 6. AArch64 Registers

### 6.1 ภาพรวมของ AArch64 (ARM64) Registers

```
AArch64 Register File:
┌─────────────────────────────────────────────────────────────────┐
│ General-Purpose Registers (64-bit)                              │
├──────────┬──────────┬──────────────────────────────────────────┤
│ 64-bit   │ 32-bit   │ คำอธิบาย                                │
├──────────┼──────────┼──────────────────────────────────────────┤
│ X0       │ W0       │ Argument/Result 1, Caller-saved          │
│ X1       │ W1       │ Argument/Result 2, Caller-saved          │
│ X2       │ W2       │ Argument 3, Caller-saved                 │
│ X3       │ W3       │ Argument 4, Caller-saved                 │
│ X4       │ W4       │ Argument 5, Caller-saved                 │
│ X5       │ W5       │ Argument 6, Caller-saved                 │
│ X6       │ W6       │ Argument 7, Caller-saved                 │
│ X7       │ W7       │ Argument 8, Caller-saved                 │
│ X8       │ W8       │ Indirect result / syscall number        │
│ X9-X15   │ W9-W15   │ Caller-saved temp registers              │
│ X16(IP0) │ W16      │ Intra-procedure scratch                 │
│ X17(IP1) │ W17      │ Intra-procedure scratch                 │
│ X18      │ W18      │ Platform register (OS specific)          │
│ X19-X28  │ W19-W28  │ Callee-saved registers                   │
│ X29(FP)  │ W29      │ Frame Pointer                            │
│ X30(LR)  │ W30      │ Link Register (return address)           │
│ SP       │ WSP      │ Stack Pointer (64/32 bit)                │
│ PC       │ -        │ Program Counter (ไม่สามารถ mov โดยตรง) │
│ XZR      │ WZR      │ Zero Register (อ่านได้ 0, เขียนทิ้ง)   │
└──────────┴──────────┴──────────────────────────────────────────┘
```

### 6.2 ตัวอย่าง AArch64 Registers

```asm
// ====================================================
// ไฟล์: aarch64_registers.s
// อธิบาย: การใช้งาน AArch64 Registers
// คอมไพล์: aarch64-linux-gnu-as -o aarch64_regs.o aarch64_registers.s
//           aarch64-linux-gnu-ld -o aarch64_regs aarch64_regs.o
// ====================================================

.section .data
hello_msg:  .ascii "Hello from AArch64!\n"
hello_len = . - hello_msg

.section .text
.global _start

// ----------------------------------------
// AArch64 Calling Convention (AAPCS64):
// X0-X7: Arguments (up to 8 args)
// X8: Indirect result location (HFA/HVA)
// X8: Also used for Linux syscall number
// X9-X15: Caller-saved temps
// X16-X17: IP0, IP1 (scratch)
// X18: Platform register
// X19-X28: Callee-saved
// X29: Frame pointer
// X30: Link register (return address)
// XZR/WZR: Zero register
// ----------------------------------------

// ฟังก์ชัน: add64
// Input: X0 = a, X1 = b
// Output: X0 = a + b (64-bit)
add64:
    add x0, x0, x1          // 64-bit addition
    ret                      // = br x30 (branch to link register)

// ฟังก์ชัน: add32
// Input: W0 = a, W1 = b (32-bit)
// Output: W0 = a + b (32-bit)
add32:
    add w0, w0, w1           // 32-bit addition (upper 32 bits zeroed)
    ret

// ฟังก์ชัน: use_xzr (ตัวอย่างการใช้ Zero Register)
// XZR อ่านได้ 0 เสมอ, WZR เช่นกัน
// การเขียนไปยัง XZR/WZR ไม่มีผล
demo_xzr:
    // ใช้ XZR เพื่อ zero-initialize
    mov x0, xzr             // x0 = 0
    mov x1, xzr             // x1 = 0
    
    // ใช้ WZR ในการ compare กับ 0
    cmp w0, wzr              // เปรียบเทียบ w0 กับ 0
    
    // ใช้ XZR ใน arithmetic
    sub x0, x1, xzr          // x0 = x1 - 0 = x1
    
    // เขียนไปยัง XZR = ทิ้งผลลัพธ์
    add xzr, x0, x1          // คำนวณแต่ทิ้งผล (ยังคง set flags ถ้าใช้ adds)
    
    ret

// ฟังก์ชัน: nested64 (ใช้ X19-X28 เป็น callee-saved)
// Input: X0-X2 = a, b, c
// Output: X0 = a*b + c
nested64:
    // บันทึก callee-saved registers ที่จะใช้
    stp x19, x20, [sp, #-16]!   // push X19, X20 (ลด SP 16 bytes)
    stp x21, x30, [sp, #-16]!   // push X21, LR
    
    // บันทึก arguments ใน callee-saved registers
    mov x19, x0                  // x19 = a
    mov x20, x1                  // x20 = b
    mov x21, x2                  // x21 = c
    
    // คำนวณ a*b
    mul x0, x19, x20             // x0 = a * b
    
    // เพิ่ม c
    add x0, x0, x21              // x0 = a*b + c
    
    // คืนค่า callee-saved registers
    ldp x21, x30, [sp], #16     // pop X21, LR
    ldp x19, x20, [sp], #16     // pop X19, X20
    
    ret

_start:
    // --- ทดสอบ 64-bit registers ---
    mov x0, #0x1234567890ABCDEF  // ใส่ค่า 64-bit (ต้องใช้ movz/movk)
    // จริงๆ ต้องใช้:
    movz x0, #0xCDEF
    movk x0, #0xAB00, lsl #8    // ไม่ถูกต้อง, ใช้เป็นตัวอย่าง
    
    // วิธีที่ถูกต้องสำหรับ 64-bit immediate:
    movz x1, #0x1234, lsl #48   // bits 63-48
    movk x1, #0x5678, lsl #32   // bits 47-32
    movk x1, #0x9ABC, lsl #16   // bits 31-16
    movk x1, #0xDEF0             // bits 15-0
    // x1 = 0x123456789ABCDEF0
    
    // --- ทดสอบ W registers (32-bit view) ---
    mov x2, #0xFFFFFFFFFFFFFFFF  // ใส่ทุก bit
    mov w2, #0x12345678          // เขียน W2 -> zero-extends ไปยัง X2
    // ตอนนี้ X2 = 0x0000000012345678
    
    // --- ทดสอบ Stack Pointer ---
    // SP ต้องจัดเรียงที่ 16 bytes ใน AArch64
    mov x3, sp                   // อ่าน SP
    
    // Push ข้อมูลลง stack
    sub sp, sp, #32              // allocate 32 bytes
    str x0, [sp]                 // เก็บ x0 ที่ SP+0
    str x1, [sp, #8]             // เก็บ x1 ที่ SP+8
    str x2, [sp, #16]            // เก็บ x2 ที่ SP+16
    
    // Pop
    ldr x0, [sp]
    ldr x1, [sp, #8]
    add sp, sp, #32              // deallocate
    
    // --- ทดสอบ add64 ---
    mov x0, #1000000000          // 1 billion
    mov x1, #2000000000          // 2 billion
    bl add64                      // x0 = 3 billion
    
    // --- ทดสอบ nested64 ---
    mov x0, #10
    mov x1, #20
    mov x2, #5
    bl nested64                   // x0 = 10*20 + 5 = 205
    
    // --- System call ใน AArch64 ---
    // Linux AArch64: x8 = syscall number, x0-x5 = arguments
    mov x0, #1                   // fd = 1 (stdout)
    adr x1, hello_msg            // buffer
    mov x2, #20                  // length
    mov x8, #64                  // sys_write = 64 ใน AArch64 (ต่างจาก ARM32!)
    svc #0                       // supervisor call
    
    // Exit
    mov x0, #0                   // exit code
    mov x8, #93                  // sys_exit = 93 ใน AArch64
    svc #0
```

---

## 7. NZCV Register ใน AArch64

### 7.1 NZCV แทน CPSR ใน AArch64

```asm
// ====================================================
// ไฟล์: aarch64_nzcv.s
// อธิบาย: NZCV Register ใน AArch64
// ====================================================

.section .text
.global _start

// ----------------------------------------
// AArch64 ใช้ PSTATE แทน CPSR
// Condition flags อยู่ใน NZCV register
// (ส่วนหนึ่งของ PSTATE)
//
// N = bit 31 (Negative)
// Z = bit 30 (Zero)
// C = bit 29 (Carry)
// V = bit 28 (Overflow)
// ----------------------------------------

demo_nzcv:
    // อ่าน NZCV
    mrs x0, nzcv             // x0 = NZCV flags
    
    // ตรวจสอบ Z flag
    tst x0, #(1 << 30)
    bne zero_flag_set
zero_flag_set:
    
    // ตั้งค่า flags โดยตรง
    mov x1, #0
    movz x1, #0, lsl #28    // NZCV = 0000 (clear all)
    msr nzcv, x1             // เขียน NZCV ตรงๆ
    
    // ตั้ง Z flag
    movz x1, #(1 << 14)     // bit 30 ใน MSR encoding
    // หรือใช้:
    cmp xzr, xzr            // 0 == 0 -> sets Z=1
    
    ret

// ----------------------------------------
// System Registers ใน AArch64
// ใช้ MRS/MSR ในการอ่าน/เขียน
// ----------------------------------------
demo_system_registers:
    // อ่าน Current EL (Exception Level)
    mrs x0, currentel        // x0 = current exception level * 4
    lsr x0, x0, #2          // x0 = actual EL (0=EL0, 1=EL1, 2=EL2, 3=EL3)
    
    // อ่าน Counter (Timestamp)
    mrs x1, cntvct_el0       // x1 = virtual counter value
    
    // อ่าน Processor ID
    mrs x2, midr_el1         // x2 = Main ID Register (EL1 required)
    
    // อ่าน Thread ID (TLS)
    mrs x3, tpidr_el0        // x3 = Thread ID Register (user-accessible)
    
    ret

// ----------------------------------------
// ตัวอย่างการใช้ conditional execution ใน AArch64
// (AArch64 ใช้ conditional instruction น้อยกว่า ARM32)
// ----------------------------------------
demo_cond_exec_a64:
    // AArch64 มี conditional instructions น้อยกว่า ARM32
    // หลักๆ ใช้: CSEL, CSINC, CSINV, CSNEG, CSET, CSETM
    
    cmp x0, x1
    
    // CSEL: Conditional Select
    // x2 = (condition) ? x0 : x1
    csel x2, x0, x1, gt     // x2 = (x0 > x1) ? x0 : x1 = max(x0, x1)
    
    // CSINC: Conditional Select Increment
    // x2 = (condition) ? x0 : x1+1
    csinc x2, x0, x1, eq    // x2 = (x0==x1) ? x0 : x1+1
    
    // CSET: Conditional Set (1 if condition, 0 otherwise)
    cset x2, gt              // x2 = (x0 > x1) ? 1 : 0
    
    // CSETM: Conditional Set Mask (-1 if condition, 0 otherwise)
    csetm x2, lt             // x2 = (x0 < x1) ? -1 : 0
    
    // CSINV: Conditional Select Invert
    csinv x2, x0, x1, ne    // x2 = (x0 != x1) ? x0 : ~x1
    
    // CSNEG: Conditional Select Negate
    csneg x2, x0, x1, mi    // x2 = (N flag) ? x0 : -x1
    
    ret

_start:
    bl demo_nzcv
    bl demo_cond_exec_a64
    
    // Exit
    mov x0, #0
    mov x8, #93
    svc #0
```

---

## 8. Floating-Point Registers และ FPCR/FPSR

### 8.1 VFP/NEON Registers ใน ARM32

```asm
@ ====================================================
@ ไฟล์: arm32_fpregs.s
@ อธิบาย: Floating-Point Registers ใน ARM32
@ ====================================================

.section .data
fp_value:   .float 3.14159
fp_result:  .float 0.0

.section .text
.global _start

@ ----------------------------------------
@ ARM32 FP Registers:
@ S0-S31: Single-precision (32-bit float) - 32 registers
@ D0-D15: Double-precision (64-bit float) - 16 registers
@ D0 = {S1, S0}, D1 = {S3, S2}, ...
@ Q0-Q7: Quad-precision (128-bit SIMD) - 8 registers
@ Q0 = {D1, D0}, Q1 = {D3, D2}, ...
@ ----------------------------------------

demo_fp_arm32:
    @ ใช้ VMOV เพื่อโหลดค่า
    vldr s0, =1.0            @ S0 = 1.0
    vldr s1, =2.5            @ S1 = 2.5
    
    @ คำนวณ float
    vadd.f32 s2, s0, s1      @ S2 = S0 + S1 = 3.5
    vmul.f32 s3, s0, s1      @ S3 = S0 * S1 = 2.5
    vsub.f32 s4, s1, s0      @ S4 = S1 - S0 = 1.5
    vdiv.f32 s5, s1, s0      @ S5 = S1 / S0 = 2.5
    
    @ Double precision
    vldr d0, =1.5            @ D0 = 1.5 (double)
    vldr d1, =2.5            @ D1 = 2.5 (double)
    vadd.f64 d2, d0, d1      @ D2 = 4.0 (double)
    
    @ อ่าน FPSCR (Floating-Point Status and Control Register)
    vmrs r0, fpscr           @ R0 = FPSCR
    
    @ FPSCR bits:
    @ Bit 31: N (Negative comparison result)
    @ Bit 30: Z (Zero comparison result)
    @ Bit 29: C (Carry/comparison result)
    @ Bit 28: V (Overflow/comparison result)
    @ Bit 27: QC (Cumulative saturation flag) - NEON
    @ Bit 26: AHP (Alternative Half-Precision)
    @ Bit 25: DN (Default NaN mode)
    @ Bit 24: FZ (Flush-to-zero mode)
    @ Bits 23-22: RMode (Rounding mode: 00=nearest, 01=+inf, 10=-inf, 11=zero)
    @ Bit 8: IOE (Invalid operation exception enable)
    @ Bit 7: DZE (Divide by zero exception enable)
    @ Bit 4: IDC (Input denormal cumulative flag)
    @ Bit 3: IXC (Inexact cumulative flag)
    @ Bit 1: DZC (Divide by zero cumulative flag)
    @ Bit 0: IOC (Invalid operation cumulative flag)
    
    @ ตั้งค่า rounding mode = round toward zero
    orr r0, r0, #(3 << 22)  @ RMode = 11 = round toward zero
    vmsr fpscr, r0           @ เขียนกลับ
    
    @ VFP comparison
    vcmp.f32 s0, s1          @ เปรียบเทียบ S0 กับ S1
    vmrs apsr_nzcv, fpscr   @ copy FP flags -> APSR (application program status register)
    @ ตอนนี้ CPSR flags ถูก update จากผลการ compare FP
    blt s0_less_than_s1
s0_less_than_s1:
    
    bx lr

_start:
    bl demo_fp_arm32
    
    mov r7, #1
    mov r0, #0
    swi #0
```

### 8.2 FPCR/FPSR ใน AArch64

```asm
// ====================================================
// ไฟล์: aarch64_fpcr_fpsr.s
// อธิบาย: FPCR และ FPSR ใน AArch64
// ====================================================

.section .text
.global _start

// ----------------------------------------
// AArch64 FP/SIMD Registers:
// V0-V31: 128-bit vector registers
// - สามารถใช้เป็น:
//   Bn (8-bit byte)
//   Hn (16-bit half-precision float)
//   Sn (32-bit single-precision float)
//   Dn (64-bit double-precision float)
//   Qn (128-bit quad)
//
// FPCR: Floating-Point Control Register
// FPSR: Floating-Point Status Register
// ----------------------------------------

demo_fp_aarch64:
    // โหลดค่า float (single precision)
    fmov s0, #1.5            // S0 = 1.5
    fmov s1, #2.5            // S1 = 2.5
    
    // คำนวณ
    fadd s2, s0, s1          // S2 = 4.0
    fmul s3, s0, s1          // S3 = 3.75
    fsub s4, s1, s0          // S4 = 1.0
    fdiv s5, s1, s0          // S5 = 5/3 ≈ 1.666
    fsqrt s6, s1             // S6 = sqrt(2.5)
    
    // Double precision
    fmov d0, #3.14           // D0 = 3.14
    fmov d1, #2.71           // D1 = 2.71
    fadd d2, d0, d1          // D2 = 5.85
    
    // อ่าน FPCR (Floating-Point Control Register)
    mrs x0, fpcr             // X0 = FPCR
    // FPCR ควบคุม:
    // Bit 26: AHP (Alternative Half-Precision)
    // Bit 25: DN (Default NaN mode)
    // Bit 24: FZ (Flush-to-zero)
    // Bits 23-22: RMode (Rounding mode)
    // Bit 12: IDE (Input Denormal exception Enable)
    // Bit 11: EBF (Extended BFloat16)
    // Bit 9: IXE (Inexact exception Enable)
    // Bit 8: UFE (Underflow exception Enable)
    // Bit 7: OFE (Overflow exception Enable)
    // Bit 2: DZE (Divide-by-Zero exception Enable)
    // Bit 0: IOE (Invalid Operation exception Enable)
    
    // อ่าน FPSR (Floating-Point Status Register)
    mrs x1, fpsr             // X1 = FPSR
    // FPSR เก็บ status:
    // Bit 31: N (Negative)
    // Bit 30: Z (Zero)
    // Bit 29: C (Carry)
    // Bit 28: V (Overflow)
    // Bit 27: QC (SIMD saturation cumulative)
    // Bit 7: IDC (Input Denormal Cumulative)
    // Bit 4: IXC (Inexact Cumulative)
    // Bit 3: UFC (Underflow Cumulative)
    // Bit 2: OFC (Overflow Cumulative)
    // Bit 1: DZC (Divide-by-Zero Cumulative)
    // Bit 0: IOC (Invalid Operation Cumulative)
    
    // ตั้งค่า Round-to-Zero mode ใน FPCR
    mrs x0, fpcr
    orr x0, x0, #(3 << 22)  // RMode = 0b11 = round toward zero
    msr fpcr, x0
    
    // FP comparison
    fcmp s0, s1              // เปรียบเทียบ S0 กับ S1
    // ผลอยู่ใน NZCV flags
    blt s0_lt_s1
    bgt s0_gt_s1
    beq s0_eq_s1
s0_lt_s1:
    b done_compare
s0_gt_s1:
    b done_compare
s0_eq_s1:
done_compare:
    
    ret

demo_fpcr_exceptions:
    // เปิดใช้ FP exceptions
    mrs x0, fpcr
    
    // เปิด divide-by-zero exception
    orr x0, x0, #(1 << 1)   // DZE bit
    msr fpcr, x0
    
    // ลองหาร 0 (จะ trigger exception ถ้า DZE=1)
    fmov s0, #1.0
    fmov s1, wzr             // s1 = 0.0
    // fdiv s2, s0, s1       // จะเกิด exception!
    
    // ปิด exceptions
    mrs x0, fpcr
    and x0, x0, #~(1 << 1)  // clear DZE
    msr fpcr, x0
    
    // clear status flags ใน FPSR
    msr fpsr, xzr            // FPSR = 0 (clear all flags)
    
    ret

_start:
    bl demo_fp_aarch64
    bl demo_fpcr_exceptions
    
    // Exit
    mov x0, #0
    mov x8, #93
    svc #0
```

---

## 9. Practical: อ่าน/เขียน CPSR

### 9.1 โปรแกรมแสดงค่า CPSR ทุก Field

```asm
@ ====================================================
@ ไฟล์: cpsr_dump.s
@ อธิบาย: อ่านและแสดงค่า CPSR ทุก field
@ ====================================================

.section .data
.equ STDOUT, 1
.equ SYS_WRITE, 4
.equ SYS_EXIT, 1

@ Strings สำหรับแสดงผล
str_cpsr:    .ascii "CPSR Value Analysis:\n"
str_cpsr_l = . - str_cpsr

str_n_set:   .ascii "  N (Negative): SET\n"
str_n_set_l = . - str_n_set
str_n_clr:   .ascii "  N (Negative): CLEAR\n"
str_n_clr_l = . - str_n_clr

str_z_set:   .ascii "  Z (Zero):     SET\n"
str_z_set_l = . - str_z_set
str_z_clr:   .ascii "  Z (Zero):     CLEAR\n"
str_z_clr_l = . - str_z_clr

str_c_set:   .ascii "  C (Carry):    SET\n"
str_c_set_l = . - str_c_set
str_c_clr:   .ascii "  C (Carry):    CLEAR\n"
str_c_clr_l = . - str_c_clr

str_v_set:   .ascii "  V (Overflow): SET\n"
str_v_set_l = . - str_v_set
str_v_clr:   .ascii "  V (Overflow): CLEAR\n"
str_v_clr_l = . - str_v_clr

str_mode_usr: .ascii "  Mode: USER (0x10)\n"
str_mode_usr_l = . - str_mode_usr
str_mode_svc: .ascii "  Mode: SVC (0x13)\n"
str_mode_svc_l = . - str_mode_svc
str_mode_irq: .ascii "  Mode: IRQ (0x12)\n"
str_mode_irq_l = . - str_mode_irq
str_mode_fiq: .ascii "  Mode: FIQ (0x11)\n"
str_mode_fiq_l = . - str_mode_fiq
str_mode_unk: .ascii "  Mode: UNKNOWN\n"
str_mode_unk_l = . - str_mode_unk

str_thumb:    .ascii "  T (Thumb):    SET\n"
str_thumb_l = . - str_thumb
str_arm:      .ascii "  T (Thumb):    CLEAR (ARM mode)\n"
str_arm_l = . - str_arm

str_irq_dis:  .ascii "  I (IRQ):      DISABLED\n"
str_irq_dis_l = . - str_irq_dis
str_irq_en:   .ascii "  I (IRQ):      ENABLED\n"
str_irq_en_l = . - str_irq_en

str_newline:  .ascii "\n"

.section .bss
cpsr_val: .space 4          @ เก็บค่า CPSR

.section .text
.global _start

@ ----------------------------------------
@ write_string: เขียน string ไปยัง stdout
@ R0 = string address
@ R1 = length
@ ----------------------------------------
write_string:
    push {r7}
    mov r2, r1              @ length
    mov r1, r0              @ buffer
    mov r0, #STDOUT
    mov r7, #SYS_WRITE
    swi #0
    pop {r7}
    bx lr

@ ----------------------------------------
@ dump_cpsr: แสดงค่า CPSR ทุก field
@ ----------------------------------------
dump_cpsr:
    push {r4, r5, lr}
    
    @ อ่าน CPSR
    mrs r4, cpsr
    str r4, cpsr_val        @ บันทึกค่าไว้
    
    @ แสดงหัวเรื่อง
    ldr r0, =str_cpsr
    mov r1, #str_cpsr_l
    bl write_string
    
    @ --- ตรวจสอบ N flag (bit 31) ---
    tst r4, #0x80000000
    bne show_n_set
    ldr r0, =str_n_clr
    mov r1, #str_n_clr_l
    b show_n
show_n_set:
    ldr r0, =str_n_set
    mov r1, #str_n_set_l
show_n:
    bl write_string
    
    @ --- ตรวจสอบ Z flag (bit 30) ---
    tst r4, #0x40000000
    bne show_z_set
    ldr r0, =str_z_clr
    mov r1, #str_z_clr_l
    b show_z
show_z_set:
    ldr r0, =str_z_set
    mov r1, #str_z_set_l
show_z:
    bl write_string
    
    @ --- ตรวจสอบ C flag (bit 29) ---
    tst r4, #0x20000000
    bne show_c_set
    ldr r0, =str_c_clr
    mov r1, #str_c_clr_l
    b show_c
show_c_set:
    ldr r0, =str_c_set
    mov r1, #str_c_set_l
show_c:
    bl write_string
    
    @ --- ตรวจสอบ V flag (bit 28) ---
    tst r4, #0x10000000
    bne show_v_set
    ldr r0, =str_v_clr
    mov r1, #str_v_clr_l
    b show_v
show_v_set:
    ldr r0, =str_v_set
    mov r1, #str_v_set_l
show_v:
    bl write_string
    
    @ --- ตรวจสอบ T bit (bit 5) ---
    tst r4, #0x20
    bne show_thumb
    ldr r0, =str_arm
    mov r1, #str_arm_l
    b show_t
show_thumb:
    ldr r0, =str_thumb
    mov r1, #str_thumb_l
show_t:
    bl write_string
    
    @ --- ตรวจสอบ I bit (bit 7) ---
    tst r4, #0x80
    bne show_irq_dis
    ldr r0, =str_irq_en
    mov r1, #str_irq_en_l
    b show_i
show_irq_dis:
    ldr r0, =str_irq_dis
    mov r1, #str_irq_dis_l
show_i:
    bl write_string
    
    @ --- ตรวจสอบ Mode bits ---
    and r5, r4, #0x1F       @ เอา mode bits
    cmp r5, #0x10
    beq show_usr_mode
    cmp r5, #0x13
    beq show_svc_mode
    cmp r5, #0x12
    beq show_irq_mode
    cmp r5, #0x11
    beq show_fiq_mode
    
    ldr r0, =str_mode_unk
    mov r1, #str_mode_unk_l
    bl write_string
    b mode_done
    
show_usr_mode:
    ldr r0, =str_mode_usr
    mov r1, #str_mode_usr_l
    bl write_string
    b mode_done
    
show_svc_mode:
    ldr r0, =str_mode_svc
    mov r1, #str_mode_svc_l
    bl write_string
    b mode_done
    
show_irq_mode:
    ldr r0, =str_mode_irq
    mov r1, #str_mode_irq_l
    bl write_string
    b mode_done
    
show_fiq_mode:
    ldr r0, =str_mode_fiq
    mov r1, #str_mode_fiq_l
    bl write_string
    
mode_done:
    pop {r4, r5, pc}

_start:
    @ แสดง CPSR ก่อนทำอะไร
    bl dump_cpsr
    
    @ --- ตั้งค่า flags โดยทำ arithmetic ---
    ldr r0, =str_newline
    mov r1, #1
    bl write_string
    
    @ สร้าง Zero flag
    mov r0, #10
    subs r0, r0, #10        @ 10 - 10 = 0, Z=1
    bl dump_cpsr
    
    ldr r0, =str_newline
    mov r1, #1
    bl write_string
    
    @ สร้าง Negative flag
    mov r0, #5
    subs r0, r0, #10        @ 5 - 10 = -5, N=1
    bl dump_cpsr
    
    ldr r0, =str_newline
    mov r1, #1
    bl write_string
    
    @ สร้าง Carry flag
    ldr r0, =0xFFFFFFFF
    adds r0, r0, #1         @ overflow unsigned, C=1
    bl dump_cpsr
    
    @ จบโปรแกรม
    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0
```

---

## 10. AArch64 System Registers (MRS/MSR)

### 10.1 การใช้งาน System Registers

```asm
// ====================================================
// ไฟล์: aarch64_sysregs.s
// อธิบาย: System Registers ใน AArch64
// ====================================================

.section .text
.global _start

// ----------------------------------------
// System Register Encoding:
// op0[1:0] : op1[2:0] : CRn[3:0] : CRm[3:0] : op2[2:0]
// MRS X<t>, <systemreg>
// MSR <systemreg>, X<t>
//
// User-accessible registers (EL0):
// - CTR_EL0: Cache Type Register
// - DCZID_EL0: DC ZVA ID Register
// - CNTVCT_EL0: Virtual Counter
// - CNTFRQ_EL0: Counter Frequency
// - NZCV: Condition flags
// - DAIF: Debug/SError/IRQ/FIQ mask bits
// - FPCR: FP Control
// - FPSR: FP Status
// - TPIDR_EL0: Thread ID
// ----------------------------------------

demo_sysregs:
    // อ่าน Cache Type Register (ข้อมูล cache ของ processor)
    mrs x0, ctr_el0
    // bits 3:0 = Instruction cache minimum line size (2^n words)
    // bits 19:16 = Data cache minimum line size (2^n words)
    and x1, x0, #0xF        // I-cache line size exponent
    ubfx x2, x0, #16, #4   // D-cache line size exponent
    
    // อ่าน Counter Frequency
    mrs x3, cntfrq_el0      // x3 = frequency (Hz)
    
    // อ่าน Virtual Counter (timestamp)
    mrs x4, cntvct_el0      // x4 = current timestamp
    
    // ทำงานบางอย่าง
    add x5, x3, x4
    
    // อ่าน timestamp อีกครั้ง
    mrs x6, cntvct_el0      // x6 = timestamp หลังทำงาน
    sub x7, x6, x4          // x7 = elapsed time in counter ticks
    
    // แปลงเป็น nanoseconds:
    // ns = ticks * 1,000,000,000 / frequency
    mov x8, #1000000000     // 1 billion
    mul x7, x7, x8          // x7 = ticks * 1e9
    udiv x7, x7, x3         // x7 = (ticks * 1e9) / freq = nanoseconds
    
    ret

demo_daif:
    // DAIF = Debug, SError, IRQ, FIQ mask bits
    // อ่านใน EL0 ได้ แต่เขียนได้บางส่วน
    
    mrs x0, daif
    // Bit 9: D (Debug exception mask)
    // Bit 8: A (SError/Async abort mask)
    // Bit 7: I (IRQ mask)
    // Bit 6: F (FIQ mask)
    
    // ตรวจสอบ IRQ mask
    tst x0, #(1 << 7)
    bne irq_masked
    // IRQ enabled
    b check_done
irq_masked:
    // IRQ disabled
check_done:
    
    ret

demo_tpidr:
    // Thread ID Register (TLS - Thread Local Storage)
    // Kernel เขียนค่านี้เมื่อ context switch
    // User space สามารถอ่านได้เพื่อรู้ TLS pointer
    
    mrs x0, tpidr_el0       // x0 = TLS pointer (set by OS)
    // x0 ชี้ไปยัง Thread Control Block (TCB)
    
    ret

_start:
    bl demo_sysregs
    bl demo_daif
    bl demo_tpidr
    
    // Exit
    mov x0, #0
    mov x8, #93
    svc #0
```

---

## 11. โปรแกรมตัวอย่าง: Benchmark ด้วย Counter Register

```asm
// ====================================================
// ไฟล์: aarch64_benchmark.s
// อธิบาย: การวัดเวลาด้วย Counter Register
// คอมไพล์: aarch64-linux-gnu-as -o bench.o aarch64_benchmark.s
//           aarch64-linux-gnu-ld -o bench bench.o
// รัน: qemu-aarch64 ./bench
// ====================================================

.section .data
bench_msg:   .ascii "Benchmark: Fibonacci calculation\n"
bench_len = . - bench_msg
result_msg:  .ascii "Result computed\n"
result_len = . - result_msg

.section .text
.global _start

// ----------------------------------------
// ฟังก์ชัน: fibonacci_iterative
// Input: X0 = n
// Output: X0 = fib(n)
// ----------------------------------------
fibonacci_iterative:
    cmp x0, #1
    ble fib_return           // ถ้า n <= 1: return n
    
    mov x1, #0              // a = fib(0) = 0
    mov x2, #1              // b = fib(1) = 1
    mov x3, x0              // counter = n
    
fib_loop:
    add x4, x1, x2          // temp = a + b
    mov x1, x2              // a = b
    mov x2, x4              // b = temp
    subs x3, x3, #1         // counter--
    bne fib_loop            // ถ้ายังไม่ถึง 0 วนซ้ำ
    
    mov x0, x1              // return a
fib_return:
    ret

// ----------------------------------------
// ฟังก์ชัน: benchmark_fib
// วัดเวลาการคำนวณ fibonacci
// ----------------------------------------
benchmark_fib:
    stp x19, x20, [sp, #-16]!
    stp x21, x30, [sp, #-16]!
    
    // อ่าน start timestamp
    mrs x19, cntvct_el0     // x19 = start time
    mrs x20, cntfrq_el0     // x20 = frequency
    
    // คำนวณ fibonacci หลายครั้ง
    mov x21, #1000          // ทำ 1000 ครั้ง
bench_loop:
    mov x0, #40             // fib(40)
    bl fibonacci_iterative
    subs x21, x21, #1
    bne bench_loop
    
    // อ่าน end timestamp
    mrs x1, cntvct_el0      // x1 = end time
    
    // คำนวณ elapsed
    sub x0, x1, x19         // x0 = elapsed ticks
    
    // แปลงเป็น microseconds
    mov x1, #1000000        // 1 million
    mul x0, x0, x1          // ticks * 1,000,000
    udiv x0, x0, x20        // / frequency = microseconds
    
    ldp x21, x30, [sp], #16
    ldp x19, x20, [sp], #16
    ret

_start:
    // แสดงข้อความ
    mov x0, #1
    adr x1, bench_msg
    mov x2, #bench_len
    mov x8, #64
    svc #0
    
    // รัน benchmark
    bl benchmark_fib
    // x0 = microseconds
    
    // แสดงผลลัพธ์
    mov x0, #1
    adr x1, result_msg
    mov x2, #result_len
    mov x8, #64
    svc #0
    
    // Exit
    mov x0, #0
    mov x8, #93
    svc #0
```

---

## 12. การ Build และรันด้วย QEMU

### 12.1 สำหรับ ARM32

```bash
#!/bin/bash
# build_arm32.sh

echo "=== Building ARM32 Programs ==="

# ติดตั้ง tools (Ubuntu/Debian)
# sudo apt install gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi qemu-user

# สร้างไฟล์ทดสอบ
cat > arm32_test.s << 'EOF'
.section .data
msg: .ascii "ARM32 Registers Test OK\n"
msg_len = . - msg

.section .text
.global _start

_start:
    @ ทดสอบ registers ทั้งหมด
    mov r0, #1
    mov r1, #2
    mov r2, #3
    mov r3, #4
    mov r4, #5
    mov r5, #6
    mov r6, #7
    mov r7, #8
    mov r8, #9
    mov r9, #10
    mov r10, #11
    mov r11, #12
    
    @ อ่าน CPSR
    mrs r12, cpsr
    
    @ print message
    mov r7, #4          @ sys_write
    mov r0, #1          @ stdout
    ldr r1, =msg
    mov r2, #msg_len
    swi #0
    
    @ exit
    mov r7, #1
    mov r0, #0
    swi #0
EOF

# Assemble
arm-linux-gnueabi-as -o arm32_test.o arm32_test.s
echo "Assembly: $?"

# Link
arm-linux-gnueabi-ld -o arm32_test arm32_test.o
echo "Linking: $?"

# Run with QEMU
echo "Running:"
qemu-arm ./arm32_test
echo "Exit code: $?"

echo ""
echo "=== ARM32 Build Complete ==="
```

### 12.2 สำหรับ AArch64

```bash
#!/bin/bash
# build_aarch64.sh

echo "=== Building AArch64 Programs ==="

# ติดตั้ง tools
# sudo apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu qemu-user

# สร้างไฟล์ทดสอบ
cat > aarch64_test.s << 'EOF'
.section .data
msg: .ascii "AArch64 Registers Test OK\n"
msg_len = . - msg

.section .text
.global _start

_start:
    // ทดสอบ registers ทั้งหมด
    mov x0, #1
    mov x1, #2
    mov x2, #3
    mov x3, #4
    
    // ทดสอบ XZR
    mov x5, xzr             // x5 = 0
    add x6, x0, xzr         // x6 = x0 + 0 = 1
    
    // อ่าน NZCV
    mrs x7, nzcv
    
    // อ่าน counter
    mrs x8, cntvct_el0
    
    // print message
    mov x0, #1
    adr x1, msg
    mov x2, #msg_len
    mov x8, #64              // sys_write
    svc #0
    
    // exit
    mov x0, #0
    mov x8, #93              // sys_exit
    svc #0
EOF

# Assemble
aarch64-linux-gnu-as -o aarch64_test.o aarch64_test.s
echo "Assembly: $?"

# Link
aarch64-linux-gnu-ld -o aarch64_test aarch64_test.o
echo "Linking: $?"

# Run with QEMU
echo "Running:"
qemu-aarch64 ./aarch64_test
echo "Exit code: $?"

echo ""
echo "=== AArch64 Build Complete ==="
```

### 12.3 Makefile รวม

```makefile
# Makefile สำหรับ ARM Assembly Course - Part 031

# Tools
ARM32_AS = arm-linux-gnueabi-as
ARM32_LD = arm-linux-gnueabi-ld
ARM64_AS = aarch64-linux-gnu-as
ARM64_LD = aarch64-linux-gnu-ld
QEMU_ARM = qemu-arm
QEMU_A64 = qemu-aarch64

# Flags
ARM32_ASFLAGS = -march=armv7-a
ARM64_ASFLAGS = -march=armv8-a

# Targets
ARM32_TARGETS = arm32_registers_demo arm32_cpsr_flags arm32_conditional_exec cpsr_dump
ARM64_TARGETS = aarch64_registers aarch64_nzcv aarch64_fpcr_fpsr aarch64_sysregs

.PHONY: all clean arm32 arm64 run

all: arm32 arm64

arm32: $(ARM32_TARGETS)
arm64: $(ARM64_TARGETS)

# ARM32 rules
%: %.s
	@echo "Building ARM32: $@"
	$(ARM32_AS) $(ARM32_ASFLAGS) -o $@.o $<
	$(ARM32_LD) -o $@ $@.o
	@echo "Done: $@"

# Special AArch64 rules
aarch64_%: aarch64_%.s
	@echo "Building AArch64: $@"
	$(ARM64_AS) $(ARM64_ASFLAGS) -o $@.o $<
	$(ARM64_LD) -o $@ $@.o
	@echo "Done: $@"

run-arm32: arm32
	@for t in $(ARM32_TARGETS); do \
		echo "Running $$t:"; \
		$(QEMU_ARM) ./$$t || true; \
		echo "---"; \
	done

run-arm64: arm64
	@for t in $(ARM64_TARGETS); do \
		echo "Running $$t:"; \
		$(QEMU_A64) ./$$t || true; \
		echo "---"; \
	done

clean:
	rm -f *.o $(ARM32_TARGETS) $(ARM64_TARGETS)

help:
	@echo "ARM Assembly Course - Part 031"
	@echo "Targets:"
	@echo "  make all      - Build all"
	@echo "  make arm32    - Build ARM32 programs"
	@echo "  make arm64    - Build AArch64 programs"
	@echo "  make run-arm32 - Run all ARM32 programs"
	@echo "  make run-arm64 - Run all AArch64 programs"
	@echo "  make clean    - Remove built files"
```

---

## 13. แบบฝึกหัด (Exercises)

### Exercise 1: Flag Analysis

```asm
@ ====================================================
@ แบบฝึกหัด 1: ทำนาย flags ก่อนรัน
@ ====================================================

.section .text
.global _start

exercise_1:
    @ สำหรับแต่ละ operation ด้านล่าง ให้ทำนายค่า N, Z, C, V
    @ แล้วตรวจสอบโดยอ่าน CPSR หลังจากรัน
    
    @ ----- Operation A -----
    mov r0, #0xFF
    add r1, r0, #1          @ = 0x100
    @ N=?, Z=?, C=?, V=?
    @ ตอบ: N=0, Z=0, C=0, V=0 (ใช้ ADD ธรรมดา ไม่ set flags)
    
    adds r1, r0, #1         @ ใช้ ADDS เพื่อ set flags
    @ N=0, Z=0, C=0, V=0 (0x100 ไม่ overflow ใน 32 bits)
    
    @ ----- Operation B -----
    ldr r0, =0xFFFFFFFF
    adds r1, r0, #1         @ 0xFFFFFFFF + 1 = 0x100000000 (overflow unsigned)
    @ N=0, Z=1, C=1, V=0
    @ Z=1 เพราะ lower 32 bits = 0
    @ C=1 เพราะ carry out จาก bit 31
    
    @ ----- Operation C -----
    ldr r0, =0x7FFFFFFF     @ max positive signed
    adds r1, r0, #1
    @ N=1, Z=0, C=0, V=1
    @ N=1 เพราะ bit 31 กลายเป็น 1
    @ V=1 เพราะ signed overflow (positive + positive = negative)
    
    @ ----- Operation D -----
    ldr r0, =0x80000000     @ min negative signed = -2147483648
    subs r1, r0, #1
    @ N=?, Z=?, C=?, V=?
    @ 0x80000000 - 1 = 0x7FFFFFFF
    @ N=0, Z=0, C=1 (no borrow), V=1 (signed overflow: negative - positive = positive)
    
    bx lr

_start:
    bl exercise_1
    
    mov r7, #1
    mov r0, #0
    swi #0
```

### Exercise 2: CPSR Manipulation

```asm
@ ====================================================
@ แบบฝึกหัด 2: การปรับแต่ง CPSR
@ ให้เขียนฟังก์ชันที่กำหนด
@ ====================================================

.section .text
.global _start

@ TODO: เขียนฟังก์ชัน set_flags
@ Input: R0 = bitmask ของ flags ที่ต้องการ set
@        (ใช้ bit 31=N, bit 30=Z, bit 29=C, bit 28=V)
@ Output: ไม่มี (แก้ไข CPSR โดยตรง)
set_flags:
    @ TODO: ใช้ MRS/MSR เพื่อแก้ไข CPSR flags
    @ Hint: mrs r1, cpsr
    @       orr r1, r1, r0   <- set bits ที่ต้องการ
    @       msr cpsr_f, r1   <- cpsr_f = flag field only
    bx lr

@ TODO: เขียนฟังก์ชัน clear_flags
@ Input: R0 = bitmask ของ flags ที่ต้องการ clear
clear_flags:
    @ TODO
    bx lr

@ TODO: เขียนฟังก์ชัน get_flag
@ Input: R0 = bit position (31=N, 30=Z, 29=C, 28=V)
@ Output: R0 = 1 ถ้า flag set, 0 ถ้า flag clear
get_flag:
    @ TODO
    bx lr

@ เฉลย (ซ่อนไว้):
set_flags_solution:
    mrs r1, cpsr
    orr r1, r1, r0
    msr cpsr_f, r1
    bx lr

clear_flags_solution:
    mrs r1, cpsr
    bic r1, r1, r0          @ bic = bit clear
    msr cpsr_f, r1
    bx lr

get_flag_solution:
    mrs r1, cpsr
    mov r2, #1
    lsl r2, r2, r0          @ r2 = 1 << bit_position
    tst r1, r2
    movne r0, #1
    moveq r0, #0
    bx lr

_start:
    @ ทดสอบฟังก์ชัน
    
    @ Set Z flag
    mov r0, #0x40000000     @ bit 30 = Z flag
    bl set_flags_solution
    
    @ ตรวจสอบ Z flag
    bne z_not_set            @ ถ้า Z=0 ไป not_set
    @ Z=1 ตามต้องการ
z_not_set:
    
    @ Clear Z flag
    mov r0, #0x40000000
    bl clear_flags_solution
    
    @ เรียก get_flag
    mov r0, #30             @ Z bit position
    bl get_flag_solution    @ ควรได้ 0
    
    mov r7, #1
    mov r0, #0
    swi #0
```

### Exercise 3: AArch64 Register Operations

```asm
// ====================================================
// แบบฝึกหัด 3: AArch64 Register Operations
// ====================================================

.section .text
.global _start

// TODO: เขียนฟังก์ชัน count_set_bits_64
// Input: X0 = 64-bit value
// Output: X0 = number of set bits (popcount)
// Hint: ใช้ X, W registers mix
count_set_bits_64:
    // TODO:
    // วิธีหนึ่ง: ใช้ CLZ + loop
    // วิธีที่ดีกว่า: ใช้ NEON instruction
    mov x1, #0
    cbz x0, done_count      // ถ้า x0 = 0 ข้าม
count_loop:
    tst x0, #1              // ตรวจ bit ต่ำสุด
    cinc x1, x1, ne        // ถ้า bit=1: x1++
    lsr x0, x0, #1         // shift right 1
    cbnz x0, count_loop    // วนซ้ำถ้ายังมี bits
done_count:
    mov x0, x1
    ret

// TODO: เขียนฟังก์ชัน swap64
// Input: X0 = pointer to value a, X1 = pointer to value b
// Output: ค่าที่ X0 และ X1 ชี้ถูก swap
swap64:
    ldr x2, [x0]            // x2 = *a
    ldr x3, [x1]            // x3 = *b
    str x3, [x0]            // *a = b
    str x2, [x1]            // *b = a
    ret

// TODO: เขียนฟังก์ชัน find_min_max
// Input: X0 = pointer to array, X1 = count
// Output: X0 = min, X1 = max
find_min_max:
    ldr x2, [x0]            // x2 = min = array[0]
    mov x3, x2              // x3 = max = array[0]
    mov x4, #1              // index = 1
find_loop:
    cmp x4, x1              // index >= count?
    bge find_done
    lsl x5, x4, #3          // offset = index * 8
    ldr x6, [x0, x5]        // x6 = array[index]
    cmp x6, x2
    csel x2, x6, x2, lt    // min = (array[i] < min) ? array[i] : min
    cmp x6, x3
    csel x3, x6, x3, gt    // max = (array[i] > max) ? array[i] : max
    add x4, x4, #1
    b find_loop
find_done:
    mov x0, x2
    mov x1, x3
    ret

.section .data
test_array: .quad 5, 3, 8, 1, 9, 2, 7, 4, 6, 10
array_count = (. - test_array) / 8

.section .text

_start:
    // ทดสอบ count_set_bits_64
    mov x0, #0xFF00FF00     // 16 bits set
    bl count_set_bits_64    // X0 ควรเป็น 16
    
    // ทดสอบ find_min_max
    adr x0, test_array
    mov x1, #array_count
    bl find_min_max
    // X0 = 1 (min), X1 = 10 (max)
    
    // Exit
    mov x0, #0
    mov x8, #93
    svc #0
```

---

## 14. สรุปความรู้สำคัญ

### 14.1 สรุป ARM32 Registers

```
ARM32 Registers Summary:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

GENERAL PURPOSE REGISTERS (R0-R15):
  R0-R3   = Arguments / Return values (caller-saved)
  R4-R11  = Variables (callee-saved: ต้อง push/pop ถ้าจะใช้)
  R12     = IP - scratch register (caller-saved)
  R13     = SP - Stack Pointer (full-descending)
  R14     = LR - Link Register (return address จาก BL)
  R15     = PC - Program Counter (current+8 ใน ARM mode)

CPSR BITS:
  [31] N = Negative result
  [30] Z = Zero result
  [29] C = Carry/borrow
  [28] V = Signed overflow
  [27] Q = Saturation (DSP)
  [9]  E = Endianness
  [8]  A = Async abort mask
  [7]  I = IRQ mask (1=disabled)
  [6]  F = FIQ mask (1=disabled)
  [5]  T = Thumb mode (1=Thumb, 0=ARM)
  [4:0] M = Processor mode

MODE BITS:
  10000 = USR    10001 = FIQ    10010 = IRQ
  10011 = SVC    10111 = ABT    11011 = UND
  11111 = SYS

KEY INSTRUCTIONS:
  MRS Rd, CPSR    - อ่าน CPSR
  MSR CPSR_f, Rn  - เขียน CPSR (flags only)
  MSR CPSR_c, Rn  - เขียน CPSR (control only)
  MSR CPSR_fsxc, Rn - เขียน CPSR (ทั้งหมด)
  MRS Rd, SPSR    - อ่าน SPSR (exception mode only)
  MSR SPSR_f, Rn  - เขียน SPSR flags

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 14.2 สรุป AArch64 Registers

```
AArch64 Registers Summary:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

GENERAL PURPOSE:
  X0-X7   = Arguments/Return (caller-saved)
  X8      = Indirect result / Linux syscall number
  X9-X15  = Caller-saved temps
  X16-X17 = IP0, IP1 (scratch)
  X18     = Platform register
  X19-X28 = Callee-saved
  X29     = Frame Pointer
  X30     = Link Register (return address จาก BL)
  SP      = Stack Pointer (must be 16-byte aligned)
  PC      = Program Counter (ไม่สามารถ MOV ได้โดยตรง)
  XZR/WZR = Zero register

FP/SIMD REGISTERS:
  V0-V31  = 128-bit vector registers
  - Bn (byte), Hn (half), Sn (single), Dn (double), Qn (quad)

SYSTEM REGISTERS (via MRS/MSR):
  NZCV    = Condition flags
  FPCR    = FP Control
  FPSR    = FP Status
  CNTVCT_EL0 = Virtual Counter
  CNTFRQ_EL0 = Counter Frequency
  TPIDR_EL0  = Thread ID
  DAIF    = Interrupt mask bits
  CurrentEL  = Current Exception Level

KEY DIFFERENCES FROM ARM32:
  - ไม่มี conditional execution กับทุก instruction
  - ใช้ CSEL/CSINC/CSET แทน conditional suffix
  - RET แทน BX LR
  - STP/LDP สำหรับ store/load pair
  - XZR ใช้แทน #0 ในหลายกรณี
  - Syscall ใช้ SVC #0 (ไม่ใช่ SWI)
  - Syscall numbers ต่างจาก ARM32

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 15. Quick Reference Card

```
╔══════════════════════════════════════════════════════════════╗
║         ARM Registers Quick Reference                        ║
╠══════════════════════════════════════════════════════════════╣
║ ARM32 MRS/MSR                                                ║
║   mrs r0, cpsr        @ Read CPSR                           ║
║   msr cpsr_f, r0      @ Write flags only                    ║
║   msr cpsr_c, r0      @ Write control only                  ║
║   mrs r0, spsr        @ Read SPSR (exception mode)          ║
║                                                              ║
║ ARM32 Condition Codes                                        ║
║   EQ/NE  Z=1/0   (equal/not equal)                         ║
║   CS/CC  C=1/0   (carry set/clear)                         ║
║   MI/PL  N=1/0   (minus/plus)                              ║
║   VS/VC  V=1/0   (overflow set/clear)                      ║
║   HI/LS  C=1&Z=0 / C=0|Z=1  (unsigned higher/lower-same)  ║
║   GE/LT  N=V/N≠V (signed >=/<)                             ║
║   GT/LE  Z=0&N=V / Z=1|N≠V  (signed >/ <=)                ║
║                                                              ║
║ AArch64 MRS/MSR                                              ║
║   mrs x0, nzcv        // Read condition flags               ║
║   msr nzcv, x0        // Write condition flags              ║
║   mrs x0, fpcr        // Read FP control                    ║
║   msr fpcr, x0        // Write FP control                   ║
║   mrs x0, fpsr        // Read FP status                     ║
║   msr fpsr, x0        // Write FP status (clear flags)      ║
║   mrs x0, cntvct_el0  // Read timestamp counter             ║
║                                                              ║
║ AArch64 Conditional Select                                   ║
║   csel  xd, xn, xm, cond  // xd = cond ? xn : xm           ║
║   csinc xd, xn, xm, cond  // xd = cond ? xn : xm+1         ║
║   cset  xd, cond          // xd = cond ? 1 : 0              ║
║   csetm xd, cond          // xd = cond ? -1 : 0             ║
╚══════════════════════════════════════════════════════════════╝
```

---

## บทสรุป (Conclusion)

ในบทนี้เราได้เรียนรู้:

1. **ARM32 Registers (R0-R15)**: การใช้งาน general-purpose registers, calling convention (AAPCS), และ special registers (SP, LR, PC)

2. **CPSR Structure**: โครงสร้างทุก bit ของ CPSR รวมถึง condition flags (N, Z, C, V), control bits (I, F, T), และ mode bits

3. **Condition Flags**: วิธีที่ arithmetic operations set flags, และการใช้ conditional execution ใน ARM32

4. **SPSR**: บทบาทของ SPSR ใน exception handling และวิธีที่ CPU บันทึก/คืนค่า CPSR

5. **Banked Registers**: ระบบ register banking ของแต่ละ processor mode โดยเฉพาะ FIQ mode

6. **AArch64 Registers**: X0-X30, XZR, SP, PC และ W registers (32-bit view)

7. **NZCV Register**: การใช้ condition flags ใน AArch64 และ conditional select instructions

8. **System Registers**: การใช้ MRS/MSR เพื่ออ่าน/เขียน system registers ทั้งใน ARM32 และ AArch64

9. **FPCR/FPSR**: การควบคุม floating-point behavior และการอ่านสถานะ FP exceptions

ความเข้าใจ registers เหล่านี้เป็นพื้นฐานสำคัญสำหรับการเขียน ARM Assembly ทั้งในระดับ application และ system programming

---

**ไฟล์ถัดไป**: Part 032 - ARM Addressing Modes และ Memory Access Patterns

---
*หลักสูตร Assembly Programming - Part 031*
*ระดับ: Intermediate-Advanced*
*ภาษา: ARM32 (GAS) / AArch64*
