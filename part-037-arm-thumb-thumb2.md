# Part 037: ARM Thumb และ Thumb-2

## บทนำ (Introduction)

ARM Thumb เป็น instruction set ที่ถูกออกแบบมาเพื่อเพิ่ม **code density** (ความหนาแน่นของโค้ด) โดยใช้ instruction ขนาด 16-bit แทน 32-bit ของ ARM ปกติ ทำให้โปรแกรมมีขนาดเล็กลงประมาณ 30-40% เหมาะสำหรับระบบ embedded ที่มี memory จำกัด

Thumb-2 ขยายความสามารถของ Thumb ด้วยการผสม instruction 16-bit และ 32-bit เข้าด้วยกัน ทำให้ได้ทั้ง code density ที่ดีและประสิทธิภาพสูง

---

## 1. ทำไมต้องใช้ Thumb: Code Density (Why Thumb: Code Density)

### ปัญหาของ ARM 32-bit Instructions

```
ARM 32-bit instruction ทุก instruction ใช้พื้นที่ 4 bytes เสมอ
แม้ว่า operation จะง่ายมากก็ตาม เช่น:

ADD R0, R0, #1    ; 4 bytes (แค่บวก 1 แต่ใช้ 4 bytes)
MOV R1, #0        ; 4 bytes (แค่กำหนดค่า 0)
BX LR             ; 4 bytes (แค่ return)
```

### Thumb 16-bit Instructions

```
Thumb instruction ส่วนใหญ่ใช้แค่ 2 bytes:

ADDS R0, R0, #1   ; 2 bytes (ประหยัดไป 50%)
MOVS R1, #0       ; 2 bytes
BX LR             ; 2 bytes
```

### การเปรียบเทียบ Code Size

```assembly
@ ===== ARM mode: ฟังก์ชัน factorial =====
@ ใช้พื้นที่: 28 bytes

factorial_arm:
    CMP R0, #1
    MOVLE R0, #1
    BXLE LR
    PUSH {R4, LR}
    MOV R4, R0
    SUB R0, R0, #1
    BL factorial_arm
    MUL R0, R4, R0
    POP {R4, PC}

@ ===== Thumb mode: ฟังก์ชัน factorial เดิม =====
@ ใช้พื้นที่: ~20 bytes (ประหยัดประมาณ 28%)

.thumb
factorial_thumb:
    CMP R0, #1
    BHI .not_one
    MOVS R0, #1
    BX LR
.not_one:
    PUSH {R4, LR}
    MOVS R4, R0
    SUBS R0, R0, #1
    BL factorial_thumb
    MULS R0, R4, R0
    POP {R4, PC}
```

### Code Density Analysis Table

```
Operation          | ARM (bytes) | Thumb (bytes) | ประหยัด
-------------------|-------------|---------------|--------
ADD Rd, Rn, Rm     | 4           | 2             | 50%
MOV Rd, #imm8      | 4           | 2             | 50%
LDR Rd, [Rn, #0]   | 4           | 2             | 50%
STR Rd, [Rn, #0]   | 4           | 2             | 50%
BL target          | 4           | 4 (Thumb BL)  | 0%
เฉลี่ย            | 4           | 2.5           | ~37.5%
```

---

## 2. Thumb State vs ARM State

### การทำงานของ ARM Processor Modes

```
ARM processor มี 2 instruction set states:
1. ARM State   - execute 32-bit instructions, PC[1:0] = 00
2. Thumb State - execute 16-bit instructions, PC[0] = 1 (CPSR T-bit set)

CPSR (Current Program Status Register):
Bit 5 (T-bit): 0 = ARM state, 1 = Thumb state
```

### การตรวจสอบ State ปัจจุบัน

```assembly
@ ตรวจสอบว่าอยู่ใน ARM หรือ Thumb state
@ ARM mode code
check_state:
    MRS R0, CPSR        @ อ่าน CPSR
    AND R0, R0, #0x20   @ mask T-bit (bit 5)
    CMP R0, #0x20
    BEQ in_thumb_state
    @ เราอยู่ใน ARM state
    B done_check
in_thumb_state:
    @ เราอยู่ใน Thumb state
done_check:
    BX LR
```

### CPSR T-bit Diagram

```
CPSR Register:
Bit: 31 30 29 28 27 ... 7  6  5  4  3  2  1  0
     N  Z  C  V  Q  ... I  F  T  M4 M3 M2 M1 M0
                           ^
                           T-bit: 0=ARM, 1=Thumb
```

---

## 3. BX Instruction: การสลับระหว่าง ARM และ Thumb State

### BX (Branch and Exchange)

```assembly
@ BX Rn - สลับ state ตาม bit[0] ของ Rn
@ ถ้า Rn[0] = 1: สลับไป Thumb state แล้ว branch
@ ถ้า Rn[0] = 0: สลับไป ARM state แล้ว branch

@ ===== ตัวอย่าง: เรียกฟังก์ชัน Thumb จาก ARM code =====

.section .text
.global main

@ ARM code
.arm
main:
    LDR R0, =thumb_function + 1  @ +1 เพื่อ set bit[0] = 1 (Thumb)
    BLX R0                        @ เรียกฟังก์ชัน Thumb
    @ กลับมาที่นี่หลัง Thumb function return
    MOV R7, #1
    SWI #0

@ Thumb code
.thumb
.thumb_func
thumb_function:
    @ ตอนนี้อยู่ใน Thumb state
    MOVS R0, #42
    BX LR                @ กลับไป ARM state (เพราะ LR[0] = 0)

@ ===== การเรียก ARM function จาก Thumb code =====
.thumb
.thumb_func
call_arm_from_thumb:
    LDR R0, =arm_function  @ arm_function address (bit[0] = 0)
    BLX R0                  @ เรียก ARM function, สลับ state อัตโนมัติ
    BX LR
```

### BLX Instruction

```assembly
@ BLX - Branch with Link and Exchange
@ ใช้สำหรับเรียกฟังก์ชันที่อยู่ใน state อื่น

.arm
caller_arm:
    @ เรียก Thumb function
    ADR R0, thumb_func + 1   @ +1 สำหรับ Thumb address
    BLX R0                    @ สลับไป Thumb และบันทึก return address ใน LR
    @ LR จะ point กลับมา ARM code ที่นี่
    BX LR

.thumb
.thumb_func
thumb_func:
    ADDS R0, R0, #1
    BX LR              @ BX LR จะสลับกลับ ARM (เพราะ LR[0] = 0)
```

### Interworking Address Table

```
Address bit[0] | การทำงาน
---------------|------------------------------------------
0              | branch ไป ARM code, T-bit = 0
1              | branch ไป Thumb code, T-bit = 1, PC = addr & ~1
```

---

## 4. Thumb Registers: การใช้งานที่จำกัด

### Thumb Register Constraints

```
ARM มี registers R0-R15
Thumb มีข้อจำกัดสำคัญ:

Low registers (Lo regs):  R0-R7   - ใช้ได้กับเกือบทุก instruction
High registers (Hi regs): R8-R15  - ใช้ได้เฉพาะ instruction บางตัว

R0-R7:  ใช้ได้กับ arithmetic, logic, load/store ทั้งหมด
R8-R12: ใช้ได้กับ MOV, CMP, ADD (เฉพาะ high register forms)
R13 (SP): ใช้กับ PUSH/POP, ADD/SUB SP
R14 (LR): ใช้กับ BX LR, MOV PC, LR
R15 (PC): ใช้กับ MOV PC, Rn (indirect branch)
```

### ตัวอย่างการใช้ Register ใน Thumb

```assembly
.thumb
.thumb_func
thumb_register_demo:
    @ Low registers - ใช้ได้เต็มที่
    MOVS R0, #10        @ OK: Lo reg
    MOVS R1, #20        @ OK: Lo reg
    ADDS R2, R0, R1     @ OK: ทั้ง 3 เป็น Lo regs
    
    @ High registers - จำกัด
    MOV R8, R0          @ OK: MOV hi <- lo
    MOV R0, R8          @ OK: MOV lo <- hi
    ADD R0, R0, R8      @ OK: ADD lo, lo, hi (special form)
    CMP R0, R8          @ OK: CMP lo, hi
    
    @ ไม่สามารถทำได้ใน Thumb (16-bit):
    @ ADDS R8, R9, R10  @ ERROR: arithmetic with all hi regs
    @ LDR R8, [R0]      @ ERROR: load to hi reg (ใน Thumb-1)
    
    BX LR
```

### Register Usage Summary

```
Instruction Type      | Registers ที่ใช้ได้
----------------------|----------------------------------
ADDS, SUBS, MULS      | R0-R7 เท่านั้น
ANDS, ORRS, EORS, etc | R0-R7 เท่านั้น
LSLS, LSRS, ASRS, RORS| R0-R7 เท่านั้น
LDR, STR, LDRB, STRB  | R0-R7 (base และ dest)
PUSH                  | R0-R7, LR
POP                   | R0-R7, PC
MOV                   | R0-R15 (any register)
ADD                   | R0-R15 (high register form)
CMP                   | R0-R15 (high register form)
BX                    | R0-R15
```

---

## 5. Thumb Instructions: 16-bit Subset of ARM

### Data Processing Instructions

```assembly
.thumb
.thumb_func
thumb_data_processing:
    @ Arithmetic
    MOVS R0, #255       @ R0 = 255, flags updated
    ADDS R1, R0, #1     @ R1 = R0 + 1, flags updated
    SUBS R2, R1, #10    @ R2 = R1 - 10, flags updated
    MULS R3, R0, R1     @ R3 = R0 * R1 (Lo regs only)
    
    @ Logic
    ANDS R4, R4, R0     @ R4 = R4 AND R0
    ORRS R5, R5, R1     @ R5 = R5 OR R1
    EORS R6, R6, R2     @ R6 = R6 XOR R2
    BICS R7, R7, R3     @ R7 = R7 AND NOT R3
    
    @ Shift
    LSLS R0, R0, #2     @ Logical shift left 2
    LSRS R1, R1, #1     @ Logical shift right 1
    ASRS R2, R2, #3     @ Arithmetic shift right 3
    RORS R3, R3, R0     @ Rotate right by R0
    
    @ Compare (ไม่เปลี่ยน register แค่ set flags)
    CMP R0, #100        @ set flags จาก R0 - 100
    CMN R1, R2          @ set flags จาก R1 + R2
    TST R0, R1          @ set flags จาก R0 AND R1
    
    BX LR

@ หมายเหตุ: Thumb instructions ส่วนใหญ่จะ update flags อัตโนมัติ
@ ต่างจาก ARM ที่ต้องใช้ 'S' suffix (ADDS, SUBS ฯลฯ)
```

### Load/Store Instructions

```assembly
.thumb
.thumb_func
thumb_load_store:
    @ LDR/STR พื้นฐาน
    LDR R0, [R1]        @ R0 = [R1]
    STR R0, [R1]        @ [R1] = R0
    
    @ Offset addressing
    LDR R0, [R1, #4]    @ R0 = [R1+4] (offset ต้องเป็น multiple of 4)
    STR R0, [R1, #8]    @ [R1+8] = R0
    
    @ Register offset
    LDR R0, [R1, R2]    @ R0 = [R1+R2]
    STR R0, [R1, R2]    @ [R1+R2] = R0
    
    @ Byte/halfword
    LDRB R0, [R1]       @ R0 = zero-extended byte จาก [R1]
    STRB R0, [R1]       @ [R1] = byte ล่างสุดของ R0
    LDRH R0, [R1]       @ R0 = zero-extended halfword จาก [R1]
    STRH R0, [R1]       @ [R1] = halfword ล่างสุดของ R0
    
    @ Signed load
    LDRSB R0, [R1]      @ R0 = sign-extended byte จาก [R1]
    LDRSH R0, [R1]      @ R0 = sign-extended halfword จาก [R1]
    
    @ PC-relative load (constant pool)
    LDR R0, =0x12345678 @ โหลดค่าคงที่ขนาดใหญ่
    
    @ SP-relative
    LDR R0, [SP, #8]    @ โหลดจาก stack
    STR R0, [SP, #4]    @ เก็บไปยัง stack
    
    BX LR
```

### Branch Instructions

```assembly
.thumb
.thumb_func
thumb_branches:
    @ Conditional branches (range: -256 to +254 bytes)
    CMP R0, #0
    BEQ equal_label     @ Branch if equal (Z=1)
    BNE not_equal       @ Branch if not equal (Z=0)
    BGT greater         @ Branch if greater (Z=0, N=V)
    BLT less_than       @ Branch if less than (N!=V)
    BGE greater_equal   @ Branch if >= (N=V)
    BLE less_equal      @ Branch if <= (Z=1 or N!=V)
    BCS carry_set       @ Branch if carry set
    BCC carry_clear     @ Branch if carry clear
    BMI minus           @ Branch if minus (N=1)
    BPL plus            @ Branch if plus (N=0)
    
    @ Unconditional branch (range: -2048 to +2046 bytes)
    B target_label
    
    @ Branch with Link (range: -4MB to +4MB, 4-byte instruction)
    BL function_name
    
    @ Branch and Exchange (state switching)
    BX R0               @ branch ไป address ใน R0 (สลับ state ถ้า bit[0]=1)
    BX LR               @ return
    
equal_label:
not_equal:
greater:
less_than:
greater_equal:
less_equal:
carry_set:
carry_clear:
minus:
plus:
target_label:
function_name:
    BX LR
```

---

## 6. PUSH/POP ใน Thumb

### PUSH และ POP

```assembly
.thumb
.thumb_func
@ PUSH/POP ใน Thumb ง่ายกว่า ARM เพราะมี syntax ที่กระชับ

function_with_push_pop:
    @ PUSH: บันทึก registers ลง stack (SP ลดลง)
    PUSH {R4, R5, R6, R7, LR}  @ บันทึก 5 registers + return address
    
    @ ทำงาน...
    MOVS R4, #10
    MOVS R5, #20
    ADDS R4, R4, R5
    
    @ POP: กู้คืน registers จาก stack (SP เพิ่มขึ้น)
    POP {R4, R5, R6, R7, PC}   @ กู้คืน 5 registers และ return (PC = LR เก่า)
    @ หมายเหตุ: POP {PC} = BX LR (เพราะ PC ถูก load ด้วย LR เก่า)

@ PUSH/POP กับ registers น้อยลง
small_function:
    PUSH {R4, LR}       @ บันทึกแค่ R4 และ LR
    
    MOV R4, R0          @ ใช้งาน R4
    BL other_function   @ เรียกฟังก์ชันอื่น (LR ถูกบันทึกไว้แล้ว)
    
    POP {R4, PC}        @ กู้คืนและ return

@ ตัวอย่าง: nested function calls
.thumb_func
outer_function:
    PUSH {R4, R5, LR}
    MOVS R4, #5
    MOVS R5, #10
    
    MOVS R0, R4
    BL inner_function   @ R0 = inner_function(5)
    ADDS R5, R5, R0     @ R5 = R5 + result
    
    MOVS R0, R5
    POP {R4, R5, PC}    @ return R5 เป็น result

.thumb_func
inner_function:
    ADDS R0, R0, #1     @ return R0 + 1
    BX LR

other_function:
    BX LR
```

### Stack Operations ใน Thumb

```assembly
.thumb
@ Stack pointer manipulation
sp_operations:
    @ ปรับ SP สำหรับ local variables
    SUB SP, SP, #16     @ จอง 16 bytes สำหรับ local vars
    
    @ ใช้งาน local variables
    MOVS R0, #42
    STR R0, [SP, #0]    @ local var 1
    MOVS R0, #100
    STR R0, [SP, #4]    @ local var 2
    
    @ อ่าน local variables
    LDR R0, [SP, #0]
    LDR R1, [SP, #4]
    ADDS R0, R0, R1     @ R0 = 42 + 100 = 142
    
    @ คืน stack
    ADD SP, SP, #16
    BX LR
```

---

## 7. Thumb-2 Extension: Mixed 16/32-bit Instructions

### Thumb-2 คืออะไร

```
Thumb-2 (ARMv6T2 และใหม่กว่า) เพิ่มความสามารถโดย:
1. เพิ่ม 32-bit instructions ลงใน Thumb instruction set
2. ไม่ต้องสลับระหว่าง ARM และ Thumb state
3. ทำให้ Cortex-M processor ใช้ Thumb-2 เพียง ISA เดียว

Thumb-2 ผสม:
- Thumb 16-bit instructions (เดิม)
- Thumb-2 32-bit instructions (ใหม่)

ทั้งหมดทำงานใน Thumb state เดียวกัน!
```

### ตัวอย่าง Thumb-2 Code

```assembly
.thumb
.syntax unified    @ ใช้ unified assembly syntax

.thumb_func
thumb2_demo:
    @ 16-bit instructions (Thumb เดิม)
    MOVS R0, #10        @ 2 bytes
    ADDS R1, R0, #5     @ 2 bytes
    
    @ 32-bit instructions (Thumb-2 ใหม่)
    MOVW R2, #0x1234    @ 4 bytes: load 16-bit immediate
    MOVT R2, #0x5678    @ 4 bytes: load upper 16-bit
    @ ตอนนี้ R2 = 0x56781234
    
    @ 32-bit การทำ barrel shift ซับซ้อน
    MOV R3, R0, LSL #16  @ 4 bytes: R3 = R0 << 16
    
    @ 32-bit load/store ที่ ARM mode เท่านั้น
    LDRD R4, R5, [SP, #8] @ 4 bytes: load doubleword
    STRD R4, R5, [SP, #8] @ 4 bytes: store doubleword
    
    BX LR

@ ตัวอย่างที่แสดงว่า Thumb-2 มีประสิทธิภาพดีกว่า Thumb-1
.thumb_func
efficient_thumb2:
    @ Thumb-1: ต้องใช้หลาย instructions
    @ MOVS R0, #0xFF
    @ LSLS R0, R0, #8
    @ ORRS R0, R0, #0x12  @ ยุ่งยาก
    
    @ Thumb-2: ทำได้ใน instruction เดียว
    MOV R0, #0xFF12     @ โหลด 16-bit value โดยตรง
    
    BX LR
```

---

## 8. IT (If-Then) Block

### IT Block คืออะไร

```
IT block ให้ Thumb code สามารถทำ conditional execution
เหมือน ARM mode (ที่มี condition codes ใน instruction)

ไวยากรณ์: IT{x}{y}{z} cond
- cond: เงื่อนไข (EQ, NE, GT, LT ฯลฯ)
- x, y, z: T (Then) หรือ E (Else) สำหรับ instructions ที่ตามมา

IT   cond         = 1 conditional instruction
ITT  cond         = 2 Then instructions
ITE  cond         = 1 Then + 1 Else instruction
ITTE cond         = 2 Then + 1 Else instruction
ITET cond         = 1 Then + 1 Else + 1 Then instruction
ITEE cond         = 1 Then + 2 Else instructions
ITTTT cond        = 4 Then instructions
```

### ตัวอย่าง IT Block

```assembly
.thumb
.syntax unified

.thumb_func
it_block_examples:
    @ ตัวอย่าง 1: ITE (if-then-else)
    CMP R0, #10
    ITE GT              @ if R0 > 10
    MOVGT R1, #1        @   then: R1 = 1
    MOVLE R1, #0        @   else: R1 = 0
    
    @ ตัวอย่าง 2: ITT (if-then-then)
    CMP R0, #0
    ITT NE              @ if R0 != 0
    MULNE R2, R0, R1    @   then: R2 = R0 * R1
    ADDNE R2, R2, #1    @   then: R2 = R2 + 1
    
    @ ตัวอย่าง 3: ITTE
    CMP R0, R1
    ITTE EQ             @ if R0 == R1
    MOVEQ R2, #1        @   then: R2 = 1
    ADDEQ R3, R0, #1    @   then: R3 = R0 + 1
    MOVNE R2, #0        @   else: R2 = 0
    
    @ ตัวอย่าง 4: nested conditions
    CMP R0, #5
    ITT LT
    CMPLT R0, #0        @ ถ้า R0 < 5: ตรวจสอบ R0 < 0
    MOVLT R4, #-1       @ ถ้า R0 < 5 AND R0 < 0: R4 = -1
    
    @ ตัวอย่าง: if-else กับ multiple registers
    CMP R0, #100
    ITE GE
    SUBGE R1, R0, #100  @ R0 >= 100: R1 = R0 - 100
    MOVLT R1, R0        @ R0 < 100:  R1 = R0
    
    BX LR

@ ตัวอย่าง: การแปลง if-else เป็น IT block
.thumb_func
absolute_value:
    @ if (R0 < 0) R0 = -R0
    CMP R0, #0
    IT MI               @ if negative
    NEGMI R0, R0        @   negate R0
    BX LR

@ เทียบกับ branch-based approach
.thumb_func
absolute_value_branch:
    CMP R0, #0
    BGE .not_negative
    NEGS R0, R0
.not_negative:
    BX LR
    @ IT block ใช้ bytes น้อยกว่าเพราะไม่ต้องมี branch instruction
```

### IT Block ใน Cortex-M Assembler

```assembly
@ สำหรับ Cortex-M (ARM Thumb-2 only processors)
@ ใช้ syntax ที่ต่างกันเล็กน้อย

.thumb
.syntax unified
.cpu cortex-m4

.thumb_func
cortex_m_it_example:
    @ ตรวจสอบ array bounds
    CMP R0, #0          @ R0 = index
    ITE GE
    CMPGE R0, R1        @ if index >= 0: check index < length (R1)
    BLT .lt_zero        @ ถ้า index < 0: error
    
    ITT LT
    LDRLT R2, [R3, R0, LSL #2]  @ if index < length: load array[index]
    BXLT LR             @ return
    
    @ index out of bounds
    MOVS R2, #0
    BX LR

.lt_zero:
    MOVS R2, #0
    BX LR
```

---

## 9. Wide Instructions (W Suffix)

### การใช้ W Suffix

```assembly
.thumb
.syntax unified

@ W suffix บังคับให้ assembler ใช้ 32-bit encoding
@ แม้ว่า 16-bit encoding จะสามารถทำได้

.thumb_func
wide_instruction_demo:
    @ Thumb-2 assembler จะเลือก encoding อัตโนมัติ
    ADD R0, R1, R2      @ assembler เลือก: อาจเป็น 16 หรือ 32-bit
    
    @ W suffix: บังคับ 32-bit encoding
    ADD.W R0, R1, R2    @ 32-bit encoding เสมอ
    
    @ ทำไมต้องใช้ W suffix?
    @ 1. เมื่อ 16-bit encoding ไม่รองรับ operands ที่ต้องการ
    @ 2. สำหรับ alignment (เมื่อต้องการให้ instruction อยู่ที่ 4-byte boundary)
    @ 3. เมื่อ 32-bit instruction มี features มากกว่า
    
    @ ตัวอย่าง: operands ที่ต้องการ 32-bit
    ADD.W R8, R9, R10   @ hi regs: ต้องใช้ 32-bit
    LDR.W R8, [R0, #100] @ offset > 124: ต้องใช้ 32-bit
    
    @ N suffix: บังคับ 16-bit encoding (narrow)
    ADD.N R0, R1, R2    @ 16-bit encoding (ถ้าเป็นไปได้)
    
    BX LR

@ ตัวอย่าง: ความแตกต่างของ encoding
.thumb_func
encoding_comparison:
    @ 16-bit: ADD R0, R0, #7  (immediate 0-7 สำหรับ 3-bit field)
    ADDS R0, R0, #7     @ 2 bytes
    
    @ 32-bit: ADD R0, R0, #255 (immediate ใหญ่กว่า)
    ADD.W R0, R0, #255  @ 4 bytes
    
    @ 16-bit: LDRB R0, [R1, #31]  (offset 0-31 สำหรับ byte)
    LDRB R0, [R1, #31]  @ 2 bytes
    
    @ 32-bit: LDRB R0, [R1, #255]  (offset ใหญ่กว่า)
    LDRB.W R0, [R1, #255] @ 4 bytes
    
    BX LR
```

---

## 10. Thumb-2 Advantages

### ข้อดีของ Thumb-2

```
1. Code Density ดีกว่า ARM
   - 16-bit instructions สำหรับงาน simple
   - ไม่ต้องสลับ state

2. ประสิทธิภาพเทียบเท่า ARM
   - 32-bit instructions สำหรับงาน complex
   - Conditional execution ด้วย IT block

3. ง่ายกว่า Thumb-1
   - ไม่มี state switching overhead
   - รองรับ hi registers มากกว่า

4. เหมาะสำหรับ Cortex-M
   - Cortex-M ใช้ Thumb-2 เพียง ISA เดียว
   - ไม่มี ARM state เลย
```

### การเปรียบเทียบ Performance

```assembly
@ ตัวอย่าง: ฟังก์ชัน strlen ใน ARM, Thumb-1, และ Thumb-2

@ ===== ARM Version (32-bit only) =====
.arm
strlen_arm:
    MOV R1, R0          @ R1 = start address
.loop_arm:
    LDRB R2, [R0], #1   @ โหลด byte, R0++
    CMP R2, #0
    BNE .loop_arm       @ ทำซ้ำจนเจอ null
    SUB R0, R0, R1      @ length = end - start
    SUB R0, R0, #1      @ ปรับ (-1 เพราะผ่าน null ไปแล้ว)
    BX LR

@ ===== Thumb-1 Version (16-bit only) =====
.thumb
strlen_thumb1:
    MOVS R1, R0         @ R1 = start address
.loop_thumb1:
    LDRB R2, [R0]       @ โหลด byte (ไม่มี post-increment ใน Thumb-1)
    ADDS R0, R0, #1     @ R0++
    CMP R2, #0
    BNE .loop_thumb1
    SUBS R0, R0, R1     @ length = end - start
    SUBS R0, R0, #1
    BX LR

@ ===== Thumb-2 Version (16/32-bit mixed) =====
.thumb
.syntax unified
strlen_thumb2:
    MOV R1, R0          @ R1 = start address (32-bit MOV)
.loop_thumb2:
    LDRB R2, [R0], #1   @ 32-bit: post-increment load
    CMP R2, #0
    BNE .loop_thumb2
    SUB R0, R0, R1      @ 32-bit SUB (flexible)
    SUB R0, R0, #1
    BX LR

@ Thumb-2 ได้ทั้ง code size ดี (ใน loop ใช้ instruction น้อย)
@ และ features ครบ (post-increment, full register access)
```

---

## 11. LDREX/STREX: Exclusive Load/Store

### Atomic Operations ด้วย LDREX/STREX

```assembly
@ LDREX/STREX ใช้สำหรับ atomic operations
@ (synchronization primitives, mutexes, semaphores)

@ LDREX: Load Exclusive - marks address for exclusive access
@ STREX: Store Exclusive - stores only if exclusive access ยังใช้ได้
@   - คืน 0 ถ้าสำเร็จ (ไม่มี interrupt/context switch เกิดขึ้น)
@   - คืน 1 ถ้าล้มเหลว (ต้องลองใหม่)

.thumb
.syntax unified

@ ===== Compare-and-Swap (CAS) =====
@ อาร์กิวเมนต์:
@   R0 = pointer ไปยัง value
@   R1 = expected value
@   R2 = new value
@ ผลลัพธ์:
@   R0 = 1 ถ้าสำเร็จ, 0 ถ้าล้มเหลว

.thumb_func
compare_and_swap:
    LDREX R3, [R0]      @ R3 = *ptr (exclusive)
    CMP R3, R1          @ compare กับ expected
    ITE EQ
    STREXEQ R4, R2, [R0] @ ถ้าเท่ากัน: *ptr = new_val, R4 = result
    MOVNE R4, #1        @ ถ้าไม่เท่ากัน: fail (R4 = 1)
    
    CMP R4, #0
    ITE EQ
    MOVEQ R0, #1        @ success
    MOVNE R0, #0        @ failure
    BX LR

@ ===== Spinlock (mutex) =====
@ R0 = pointer ไปยัง lock variable (0 = unlocked, 1 = locked)

.thumb_func
acquire_lock:
    MOVS R1, #1         @ R1 = 1 (locked value)
.spin_loop:
    LDREX R2, [R0]      @ R2 = *lock (exclusive)
    CMP R2, #0
    BNE .spin_loop      @ ถ้า locked แล้ว: รอ
    STREX R3, R1, [R0]  @ ลอง set lock
    CMP R3, #0
    BNE .spin_loop      @ ถ้า STREX ล้มเหลว: ลองใหม่
    DMB                 @ Data Memory Barrier (ป้องกัน reordering)
    BX LR

.thumb_func
release_lock:
    DMB                 @ ป้องกัน reordering ก่อน release
    MOVS R1, #0
    STR R1, [R0]        @ *lock = 0 (unlock, ไม่ต้องใช้ exclusive)
    BX LR

@ ===== Atomic Increment =====
@ R0 = pointer ไปยัง counter

.thumb_func
atomic_increment:
.retry_inc:
    LDREX R1, [R0]      @ R1 = *counter (exclusive)
    ADDS R1, R1, #1     @ R1++
    STREX R2, R1, [R0]  @ ลอง store
    CMP R2, #0
    BNE .retry_inc      @ ถ้าล้มเหลว: ลองใหม่
    BX LR

@ ===== Atomic Decrement =====
@ R0 = pointer ไปยัง counter
@ ผลลัพธ์: R0 = new value (หลัง decrement)

.thumb_func
atomic_decrement:
.retry_dec:
    LDREX R1, [R0]      @ R1 = *counter (exclusive)
    SUBS R1, R1, #1     @ R1--
    STREX R2, R1, [R0]  @ ลอง store
    CMP R2, #0
    BNE .retry_dec      @ retry
    MOV R0, R1          @ return new value
    BX LR
```

### CLREX: Clear Exclusive

```assembly
.thumb
.syntax unified

@ CLREX ใช้เพื่อยกเลิก exclusive reservation
@ จำเป็นเมื่อ context switch หรือ exception เกิดขึ้น

@ Exception handler ต้อง CLREX เพื่อป้องกัน false success
irq_handler:
    PUSH {R0-R3, LR}
    CLREX                @ ยกเลิก exclusive access ใดๆ
    
    @ handle interrupt...
    BL do_irq_work
    
    POP {R0-R3, PC}

do_irq_work:
    BX LR
```

---

## 12. TBB/TBH: Table Branch

### Table Branch Byte/Halfword

```assembly
@ TBB: Table Branch Byte    - สำหรับ jump table ที่ compact
@ TBH: Table Branch Halfword - สำหรับ jump table ขนาดกลาง

@ TBB [Rn, Rm]: PC = PC + 2 * ZeroExtend(memory[Rn + Rm])
@ TBH [Rn, Rm, LSL #1]: PC = PC + 2 * ZeroExtend(HalfWord[Rn + Rm*2])

.thumb
.syntax unified

@ ===== Switch-case ด้วย TBB =====
@ R0 = switch value (0-3)

.thumb_func
switch_case_tbb:
    TBB [PC, R0]        @ jump table ที่อยู่หลัง instruction นี้
.jump_table_byte:
    .byte (.case0 - .jump_table_byte) / 2
    .byte (.case1 - .jump_table_byte) / 2
    .byte (.case2 - .jump_table_byte) / 2
    .byte (.case3 - .jump_table_byte) / 2
    .align 2            @ align ให้ครบ 2 bytes
    
.case0:
    MOVS R0, #0
    B .switch_end
.case1:
    MOVS R0, #10
    B .switch_end
.case2:
    MOVS R0, #20
    B .switch_end
.case3:
    MOVS R0, #30
.switch_end:
    BX LR

@ ===== Switch-case ด้วย TBH (สำหรับ offset ใหญ่กว่า) =====
@ R0 = switch value (0-3)

.thumb_func
switch_case_tbh:
    TBH [PC, R0, LSL #1]   @ 16-bit offsets
.jump_table_half:
    .short (.caseA - .jump_table_half) / 2
    .short (.caseB - .jump_table_half) / 2
    .short (.caseC - .jump_table_half) / 2
    .short (.caseD - .jump_table_half) / 2

.caseA:
    @ case A: code ที่ใช้ bytes มาก อาจต้องใช้ TBH
    MOVS R0, #100
    B .switch_end_h
.caseB:
    MOVS R0, #200
    B .switch_end_h
.caseC:
    MOVS R0, #0
    B .switch_end_h
.caseD:
    MOVS R0, #255
.switch_end_h:
    BX LR

@ ===== ตัวอย่างที่ใช้งานได้จริง: state machine =====
@ R0 = current state (0-4)

.thumb_func
state_machine_step:
    CMP R0, #4
    BHI .invalid_state
    TBB [PC, R0]
.state_table:
    .byte (.state_idle - .state_table) / 2
    .byte (.state_init - .state_table) / 2
    .byte (.state_run  - .state_table) / 2
    .byte (.state_pause - .state_table) / 2
    .byte (.state_stop - .state_table) / 2
    .align 2

.state_idle:
    MOVS R0, #1         @ transition to init
    BX LR
.state_init:
    BL do_init
    MOVS R0, #2         @ transition to run
    BX LR
.state_run:
    BL do_run
    @ stay in run state
    MOVS R0, #2
    BX LR
.state_pause:
    MOVS R0, #2         @ resume: back to run
    BX LR
.state_stop:
    MOVS R0, #0         @ back to idle
    BX LR
.invalid_state:
    MOVS R0, #0         @ reset to idle
    BX LR

do_init:
    BX LR
do_run:
    BX LR
```

---

## 13. WFI/WFE/SEV: Power Management Instructions

### WFI (Wait For Interrupt)

```assembly
.thumb
.syntax unified

@ WFI: Suspend execution จนกว่าจะมี interrupt
@ ใช้ใน idle loop เพื่อประหยัดพลังงาน

@ ===== Idle Loop =====
.thumb_func
idle_loop:
    @ ทำงานปกติ
    BL process_tasks
    
    @ รอ interrupt (CPU เข้าสู่ low-power state)
    WFI                 @ Wait For Interrupt
    
    @ กลับมาทำงานเมื่อ interrupt เกิดขึ้น
    B idle_loop

@ ===== Interrupt-driven processing =====
@ ตัวอย่างจาก embedded system

volatile_flag: .word 0

.thumb_func
main_loop:
    PUSH {LR}
.check_flag:
    LDR R0, =volatile_flag
    LDR R1, [R0]        @ อ่าน flag
    CMP R1, #0
    BNE .process         @ ถ้า flag set: ทำงาน
    WFI                  @ ไม่มีงาน: รอ interrupt
    B .check_flag
.process:
    MOVS R2, #0
    STR R2, [R0]         @ clear flag
    BL do_work
    B .check_flag

do_work:
    BX LR

process_tasks:
    BX LR
```

### WFE (Wait For Event) และ SEV (Send Event)

```assembly
.thumb
.syntax unified

@ WFE: Wait For Event
@ - ถ้า event register set อยู่แล้ว: clear register และ continue ทันที
@ - ถ้า event register clear: suspend จนมี event
@
@ SEV: Send Event to all processors (สำหรับ multi-processor systems)

@ ===== Spin-wait with WFE (ประหยัดพลังงานกว่า busy-wait) =====
@ R0 = pointer ไปยัง lock variable

.thumb_func
efficient_spinlock:
    MOVS R1, #1
.try_lock:
    LDREX R2, [R0]
    CMP R2, #0
    BEQ .try_acquire
    WFE                 @ รอ event แทนที่จะ spin ให้เปล่า
    B .try_lock
    
.try_acquire:
    STREX R3, R1, [R0]
    CMP R3, #0
    BNE .try_lock       @ STREX ล้มเหลว: ลองใหม่
    DMB
    BX LR

.thumb_func
release_lock_with_sev:
    DMB
    MOVS R1, #0
    STR R1, [R0]        @ unlock
    DSB
    SEV                 @ แจ้ง processor อื่นที่กำลัง WFE
    BX LR
```

---

## 14. DSB/DMB/ISB: Memory Barrier Instructions

### ทำไมต้องใช้ Memory Barriers

```
Modern processors ทำ out-of-order execution และ memory reordering
เพื่อประสิทธิภาพ แต่ในบางกรณีต้องการ ordering ที่แน่นอน:

1. Multi-threaded code: ป้องกัน data races
2. Device I/O: ต้องการลำดับการ write ที่ถูกต้อง
3. Cache coherency: ในระบบ multi-processor
4. Interrupt handlers: ป้องกัน reordering ข้าม interrupt
```

### DSB (Data Synchronization Barrier)

```assembly
.thumb
.syntax unified

@ DSB: ทุก memory access ที่ออกไปก่อน DSB ต้องเสร็จสมบูรณ์
@      ก่อนที่ instruction หลัง DSB จะทำงาน

@ ตัวอย่าง: device driver
.thumb_func
write_device_register:
    @ เขียน control register
    STR R0, [R1]        @ device control register
    DSB                 @ รอให้การเขียนเสร็จก่อน
    STR R2, [R3]        @ device data register (หลัง control)
    BX LR

@ ตัวอย่าง: cache maintenance
.thumb_func
flush_cache_line:
    MCR p15, 0, R0, c7, c10, 1  @ Clean cache line
    DSB                          @ รอให้ flush เสร็จ
    BX LR
```

### DMB (Data Memory Barrier)

```assembly
.thumb
.syntax unified

@ DMB: ทุก memory access ก่อน DMB ต้องมองเห็นได้
@      ก่อน memory access หลัง DMB
@ (ต่างจาก DSB ตรงที่ DMB ไม่รอ non-memory instructions)

@ ตัวอย่าง: producer-consumer pattern

data_ready: .word 0
buffer:     .space 100

.thumb_func
producer:
    @ เขียนข้อมูลลง buffer
    LDR R0, =buffer
    MOVS R1, #42
    STR R1, [R0]
    
    DMB                 @ ป้องกัน: data_ready ถูก set ก่อน buffer เขียน
    
    @ บอก consumer ว่าข้อมูลพร้อม
    LDR R0, =data_ready
    MOVS R1, #1
    STR R1, [R0]
    BX LR

.thumb_func
consumer:
.wait_for_data:
    LDR R0, =data_ready
    LDR R1, [R0]
    CMP R1, #0
    BEQ .wait_for_data  @ รอจนข้อมูลพร้อม
    
    DMB                 @ ป้องกัน: buffer อ่านก่อน data_ready อ่าน
    
    LDR R0, =buffer
    LDR R2, [R0]        @ อ่านข้อมูล
    @ ใช้ R2...
    BX LR
```

### ISB (Instruction Synchronization Barrier)

```assembly
.thumb
.syntax unified

@ ISB: flush instruction pipeline
@ ใช้หลังจาก:
@ 1. เปลี่ยน processor configuration (cache, MMU)
@ 2. Self-modifying code
@ 3. Branch predictor flush

@ ตัวอย่าง: เปิด/ปิด MMU

.thumb_func
enable_mmu:
    @ configure MMU
    MCR p15, 0, R0, c2, c0, 0  @ TTBR0
    MCR p15, 0, R1, c3, c0, 0  @ DACR
    
    DSB                          @ รอให้ configuration เสร็จ
    
    @ เปิด MMU
    MRC p15, 0, R2, c1, c0, 0  @ อ่าน SCTLR
    ORR R2, R2, #1              @ set M-bit
    MCR p15, 0, R2, c1, c0, 0  @ เขียน SCTLR
    
    ISB                          @ flush pipeline เพื่อให้ใช้ address translation ใหม่
    BX LR

@ ตัวอย่าง: self-modifying code
.thumb_func
patch_instruction:
    @ แก้ไข instruction ใน memory
    STR R0, [R1]        @ เขียน instruction ใหม่
    
    DSB                 @ รอให้การเขียนเสร็จ
    ISB                 @ flush instruction cache / pipeline
    
    @ ตอนนี้ safe ที่จะ execute ณ R1
    BX R1

@ Memory barriers summary:
@
@ DMB - ป้องกัน memory access reordering
@ DSB - ป้องกัน memory access reordering + รอให้เสร็จ
@ ISB - flush instruction pipeline
@
@ ลำดับความเข้มงวด: ISB > DSB > DMB
```

---

## 15. Cortex-M: Thumb-2 Only Processor

### Cortex-M Architecture

```
Cortex-M processors (embedded MCU):
- Cortex-M0/M0+: Thumb subset เท่านั้น
- Cortex-M3:     Thumb-2 เต็มรูปแบบ
- Cortex-M4:     Thumb-2 + DSP + optional FPU
- Cortex-M7:     Thumb-2 + dual-issue + FPU
- Cortex-M33/55: Thumb-2 + TrustZone + SIMD

ทุก Cortex-M ทำงานใน Thumb state เท่านั้น
ไม่มี ARM state, ไม่มี state switching
```

### Cortex-M Programming

```assembly
@ ===== Cortex-M4 Assembly Example =====
@ ไฟล์: cortex_m4_demo.s

.cpu cortex-m4
.fpu fpv4-sp-d16      @ Floating Point Unit
.thumb
.syntax unified

@ Vector table (ต้องอยู่ที่ 0x00000000)
.section .isr_vector, "a", %progbits
.word _estack           @ Initial SP value
.word Reset_Handler     @ Reset handler
.word NMI_Handler
.word HardFault_Handler
.word MemManage_Handler
.word BusFault_Handler
.word UsageFault_Handler
.word 0, 0, 0, 0        @ Reserved
.word SVC_Handler
.word DebugMon_Handler
.word 0                 @ Reserved
.word PendSV_Handler
.word SysTick_Handler
@ External interrupts...

.section .text

@ ===== Reset Handler =====
.thumb_func
.global Reset_Handler
Reset_Handler:
    @ Initialize stack pointer (ทำโดย hardware แล้ว จาก vector table)
    
    @ Copy .data section from flash to RAM
    LDR R0, =_sdata     @ destination (RAM)
    LDR R1, =_etext     @ source (Flash)
    LDR R2, =_edata     @ end of .data in RAM
    
    CMP R0, R2
    BEQ .data_done
.copy_data:
    LDR R3, [R1], #4    @ load from flash, increment
    STR R3, [R0], #4    @ store to RAM, increment
    CMP R0, R2
    BNE .copy_data
.data_done:

    @ Zero initialize .bss section
    LDR R0, =_sbss
    LDR R1, =_ebss
    MOVS R2, #0
    
    CMP R0, R1
    BEQ .bss_done
.zero_bss:
    STR R2, [R0], #4
    CMP R0, R1
    BNE .zero_bss
.bss_done:

    @ Call main
    BL main
    
    @ ถ้า main return: loop forever
.forever:
    WFI
    B .forever

@ ===== SysTick Handler =====
tick_counter: .word 0

.thumb_func
SysTick_Handler:
    PUSH {R0-R3, LR}
    LDR R0, =tick_counter
    LDR R1, [R0]
    ADDS R1, R1, #1
    STR R1, [R0]
    POP {R0-R3, PC}

@ ===== GPIO Control =====
@ Cortex-M4 STM32F4 example

.equ GPIOA_BASE,  0x40020000
.equ GPIO_MODER,  0x00
.equ GPIO_ODR,    0x14
.equ GPIO_BSRR,   0x18

.thumb_func
gpio_init_output:
    @ R0 = GPIO base address
    @ R1 = pin number (0-15)
    
    LDR R2, [R0, #GPIO_MODER]   @ อ่าน mode register
    
    @ clear 2 bits สำหรับ pin นี้
    MOVS R3, #3
    LSL R3, R3, R1
    LSL R3, R3, R1              @ R3 = 3 << (pin * 2) แต่ต้องทำ 2 ครั้ง
    BIC R2, R2, R3              @ clear mode bits
    
    @ set output mode (01)
    MOVS R3, #1
    LSL R3, R3, R1
    LSL R3, R3, R1
    ORR R2, R2, R3
    
    STR R2, [R0, #GPIO_MODER]   @ เขียนกลับ
    BX LR

.thumb_func
gpio_set_pin:
    @ R0 = GPIO base address
    @ R1 = pin number (0-15)
    
    MOVS R2, #1
    LSL R2, R2, R1
    STR R2, [R0, #GPIO_BSRR]   @ BSRR: bit set/reset register
    BX LR

.thumb_func
gpio_clear_pin:
    @ R0 = GPIO base address
    @ R1 = pin number (0-15)
    
    MOVS R2, #1
    LSL R2, R2, R1
    LSL R2, R2, #16             @ BSRR: upper 16 bits สำหรับ clear
    STR R2, [R0, #GPIO_BSRR]
    BX LR

@ Exception handlers (minimal)
NMI_Handler:
HardFault_Handler:
MemManage_Handler:
BusFault_Handler:
UsageFault_Handler:
SVC_Handler:
DebugMon_Handler:
PendSV_Handler:
    B .                         @ loop forever

@ main function (placeholder)
main:
    B .
```

### Cortex-M Exception Handling

```assembly
.cpu cortex-m3
.thumb
.syntax unified

@ Exception ใน Cortex-M ทำการ push/pop registers อัตโนมัติ
@ Hardware จะ push: xPSR, PC, LR, R12, R3, R2, R1, R0
@ ไม่ต้อง PUSH ใน ISR เองถ้าใช้แค่ R0-R3, R12

@ ===== Simple ISR (ไม่ต้อง PUSH/POP) =====
.thumb_func
TIM2_IRQHandler:
    @ ใช้ R0-R3 ได้เลยโดยไม่ต้อง save
    LDR R0, =TIM2_SR    @ Timer status register
    MOVS R1, #0
    STR R1, [R0]        @ clear interrupt flag
    BX LR               @ return from exception

@ ===== ISR ที่ใช้ R4 ขึ้นไป =====
.thumb_func
USART1_IRQHandler:
    PUSH {R4, R5, LR}   @ save callee-saved registers
    
    LDR R4, =USART1_SR  @ USART status
    LDR R5, =USART1_DR  @ USART data
    
    LDR R0, [R4]        @ อ่าน status
    ANDS R0, R0, #0x20  @ check RXNE bit
    BEQ .no_data
    
    LDR R0, [R5]        @ อ่าน received byte
    BL process_byte
    
.no_data:
    POP {R4, R5, PC}    @ restore และ return

@ Placeholder addresses
TIM2_SR:  .word 0
USART1_SR: .word 0
USART1_DR: .word 0

process_byte:
    BX LR
```

---

## 16. Thumb Code Density Analysis

### การวัด Code Density

```assembly
@ ===== Code Density Benchmark =====
@ เปรียบเทียบ ARM vs Thumb vs Thumb-2

@ ฟังก์ชัน: memcpy
@ void memcpy(void *dst, const void *src, int len)
@ R0 = dst, R1 = src, R2 = len

@ --- ARM Version: 24 bytes ---
.arm
memcpy_arm:
    CMP R2, #0
    BXEQ LR
.loop_arm:
    LDRB R3, [R1], #1
    STRB R3, [R0], #1
    SUBS R2, R2, #1
    BNE .loop_arm
    BX LR

@ --- Thumb-1 Version: ~22 bytes ---
.thumb
memcpy_thumb1:
    CMP R2, #0
    BEQ .done_t1
.loop_t1:
    LDRB R3, [R1]
    ADDS R1, R1, #1
    STRB R3, [R0]
    ADDS R0, R0, #1
    SUBS R2, R2, #1
    BNE .loop_t1
.done_t1:
    BX LR

@ --- Thumb-2 Version: 18 bytes (best!) ---
.thumb
.syntax unified
memcpy_thumb2:
    CBZ R2, .done_t2    @ 2 bytes: Compare and Branch if Zero
.loop_t2:
    LDRB R3, [R1], #1   @ 4 bytes: post-increment (Thumb-2 only)
    STRB R3, [R0], #1   @ 4 bytes: post-increment
    SUBS R2, R2, #1     @ 2 bytes
    BNE .loop_t2        @ 2 bytes
.done_t2:
    BX LR               @ 2 bytes

@ ผลลัพธ์:
@ ARM:     24 bytes, 6 instructions
@ Thumb-1: 22 bytes, 7 instructions (ต้องใช้ instruction มากกว่า)
@ Thumb-2: 18 bytes, 5 instructions (ดีที่สุด!)
```

### CBZ/CBNZ Instructions (Thumb-2)

```assembly
.thumb
.syntax unified

@ CBZ:  Compare and Branch if Zero
@ CBNZ: Compare and Branch if Non-Zero
@ ประหยัดกว่า CMP + B (2 bytes แทน 4 bytes)

.thumb_func
cbz_cbnz_examples:
    @ แทน: CMP R0, #0 + BEQ .null_ptr
    CBZ R0, .null_ptr   @ ถ้า R0 == 0: branch
    
    @ แทน: CMP R1, #0 + BNE .not_zero
    CBNZ R1, .not_zero  @ ถ้า R1 != 0: branch
    
    B .end_example
.null_ptr:
.not_zero:
.end_example:
    BX LR

@ ตัวอย่าง: null check ใน loop
.thumb_func
find_nonzero:
    @ R0 = pointer ไปยัง array
    @ R1 = length
    @ ผลลัพธ์: R0 = pointer ไปยัง first non-zero element, หรือ NULL
    
    CBZ R1, .return_null    @ ถ้า length == 0: return NULL
.search_loop:
    LDRB R2, [R0]
    CBNZ R2, .found         @ ถ้าพบ non-zero: return
    ADDS R0, R0, #1
    SUBS R1, R1, #1
    BNE .search_loop
.return_null:
    MOVS R0, #0
.found:
    BX LR
```

---

## 17. โปรแกรมตัวอย่างที่สมบูรณ์

### ตัวอย่าง 1: Thumb-2 Sorting Algorithm

```assembly
@ ===== Insertion Sort in Thumb-2 =====
@ void insertion_sort(int *arr, int len)
@ R0 = array pointer, R1 = length

.cpu cortex-m3
.thumb
.syntax unified

.section .text

.thumb_func
.global insertion_sort
insertion_sort:
    PUSH {R4, R5, R6, R7, LR}
    
    @ i = 1
    MOVS R2, #1         @ R2 = i
    
.outer_loop:
    CMP R2, R1          @ if i >= len: done
    BGE .sort_done
    
    @ key = arr[i]
    LDR R3, [R0, R2, LSL #2]   @ R3 = arr[i] (key)
    
    @ j = i - 1
    SUBS R4, R2, #1     @ R4 = j
    
.inner_loop:
    @ while j >= 0 && arr[j] > key
    BMI .inner_done     @ j < 0: exit inner
    LDR R5, [R0, R4, LSL #2]   @ R5 = arr[j]
    CMP R5, R3
    BLE .inner_done     @ arr[j] <= key: exit
    
    @ arr[j+1] = arr[j]
    ADD R6, R4, #1
    STR R5, [R0, R6, LSL #2]
    
    SUBS R4, R4, #1     @ j--
    B .inner_loop
    
.inner_done:
    @ arr[j+1] = key
    ADD R6, R4, #1
    STR R3, [R0, R6, LSL #2]
    
    ADDS R2, R2, #1     @ i++
    B .outer_loop
    
.sort_done:
    POP {R4, R5, R6, R7, PC}


@ ===== ทดสอบ insertion_sort =====
.global test_sort
test_sort:
    PUSH {LR}
    
    @ สร้าง array ใน stack
    SUB SP, SP, #20     @ จอง 5 * 4 bytes
    MOVS R0, #50
    STR R0, [SP, #0]
    MOVS R0, #30
    STR R0, [SP, #4]
    MOVS R0, #40
    STR R0, [SP, #8]
    MOVS R0, #10
    STR R0, [SP, #12]
    MOVS R0, #20
    STR R0, [SP, #16]
    
    @ เรียก insertion_sort
    MOV R0, SP          @ R0 = array pointer
    MOVS R1, #5         @ R1 = length
    BL insertion_sort
    
    @ ตรวจสอบผลลัพธ์
    LDR R0, [SP, #0]    @ R0 = arr[0] (ควรเป็น 10)
    
    ADD SP, SP, #20
    POP {PC}
```

### ตัวอย่าง 2: String Processing ใน Thumb-2

```assembly
@ ===== String Library in Thumb-2 =====

.cpu cortex-m4
.thumb
.syntax unified

@ strlen: ความยาวของ string
@ Input:  R0 = string pointer
@ Output: R0 = length

.thumb_func
.global thumb_strlen
thumb_strlen:
    MOV R1, R0          @ save start
.len_loop:
    LDRB R2, [R0], #1   @ โหลด byte, R0++
    CBNZ R2, .len_loop  @ ถ้าไม่ใช่ null: ทำต่อ
    SUB R0, R0, R1      @ length = end - start
    SUB R0, R0, #1      @ ปรับ (ผ่าน null ไปแล้ว)
    BX LR

@ strcpy: copy string
@ Input:  R0 = dst, R1 = src
@ Output: R0 = dst

.thumb_func
.global thumb_strcpy
thumb_strcpy:
    MOV R2, R0          @ save dst
.copy_loop:
    LDRB R3, [R1], #1   @ โหลดจาก src, src++
    STRB R3, [R2], #1   @ เก็บที่ dst, dst++
    CBNZ R3, .copy_loop @ ถ้าไม่ใช่ null: ทำต่อ
    BX LR               @ R0 ยังคงชี้ไปยัง dst เดิม

@ strcmp: เปรียบเทียบ strings
@ Input:  R0 = str1, R1 = str2
@ Output: R0 = 0 (equal), negative (str1 < str2), positive (str1 > str2)

.thumb_func
.global thumb_strcmp
thumb_strcmp:
.cmp_loop:
    LDRB R2, [R0], #1   @ R2 = *str1++
    LDRB R3, [R1], #1   @ R3 = *str2++
    CMP R2, R3
    BNE .cmp_diff       @ ไม่เท่ากัน: คำนวณ difference
    CBZ R2, .cmp_equal  @ ทั้งคู่เป็น null: equal
    B .cmp_loop
.cmp_diff:
    SUB R0, R2, R3      @ return difference
    BX LR
.cmp_equal:
    MOVS R0, #0
    BX LR

@ strcat: ต่อ string
@ Input:  R0 = dst, R1 = src
@ Output: R0 = dst

.thumb_func
.global thumb_strcat
thumb_strcat:
    MOV R2, R0          @ save dst
    
    @ หา end of dst
.find_end:
    LDRB R3, [R2], #1
    CBNZ R3, .find_end
    SUB R2, R2, #1      @ R2 ชี้ไปยัง null terminator
    
    @ copy src ไปต่อ
.cat_loop:
    LDRB R3, [R1], #1
    STRB R3, [R2], #1
    CBNZ R3, .cat_loop
    
    BX LR               @ R0 = dst
```

### ตัวอย่าง 3: Cortex-M RTOS Primitives

```assembly
@ ===== Simple Task Scheduler Primitives for Cortex-M =====

.cpu cortex-m3
.thumb
.syntax unified

@ Task Control Block (TCB) structure:
@ Offset 0: stack pointer (SP ของ task)
@ Offset 4: next TCB pointer
@ Offset 8: task state (0=ready, 1=running, 2=blocked)
@ Offset 12: task ID

@ Current task pointer
current_task: .word 0

@ ===== Context Switch (SysTick or PendSV triggered) =====
.thumb_func
.global PendSV_Handler
PendSV_Handler:
    @ บันทึก context ของ task ปัจจุบัน
    MRS R0, PSP         @ R0 = Process Stack Pointer
    STMDB R0!, {R4-R11} @ บันทึก R4-R11 ลง task stack
    
    @ อัพเดท SP ใน TCB
    LDR R1, =current_task
    LDR R2, [R1]        @ R2 = current TCB pointer
    STR R0, [R2, #0]    @ บันทึก SP ใน TCB offset 0
    
    @ เลือก task ถัดไป
    BL select_next_task @ R0 = next task TCB
    
    @ อัพเดท current_task
    LDR R1, =current_task
    STR R0, [R1]
    
    @ กู้คืน context ของ task ใหม่
    LDR R0, [R0, #0]    @ R0 = next task SP
    LDMIA R0!, {R4-R11} @ กู้คืน R4-R11
    MSR PSP, R0         @ อัพเดท PSP
    
    @ Return to thread mode with PSP
    MOV LR, #0xFFFFFFFD @ EXC_RETURN: return to thread mode, PSP
    BX LR

@ ===== Task Creation =====
@ R0 = TCB pointer
@ R1 = task function pointer
@ R2 = stack top (highest address)

.thumb_func
task_create:
    PUSH {R4, R5, LR}
    MOV R4, R0          @ R4 = TCB
    MOV R5, R2          @ R5 = stack top
    
    @ สร้าง initial stack frame (เหมือน hardware exception frame)
    @ xPSR, PC, LR, R12, R3, R2, R1, R0
    
    MOVS R3, #0x01000000    @ xPSR: Thumb mode
    STR R3, [R5, #-4]!      @ push xPSR
    STR R1, [R5, #-4]!      @ push PC (task function)
    LDR R3, =task_exit      @ default return address
    STR R3, [R5, #-4]!      @ push LR
    MOVS R3, #0
    STR R3, [R5, #-4]!      @ push R12 = 0
    STR R3, [R5, #-4]!      @ push R3 = 0
    STR R3, [R5, #-4]!      @ push R2 = 0
    STR R3, [R5, #-4]!      @ push R1 = 0
    STR R3, [R5, #-4]!      @ push R0 = 0
    
    @ สร้าง software-saved registers (R4-R11)
    STR R3, [R5, #-4]!      @ R11
    STR R3, [R5, #-4]!      @ R10
    STR R3, [R5, #-4]!      @ R9
    STR R3, [R5, #-4]!      @ R8
    STR R3, [R5, #-4]!      @ R7
    STR R3, [R5, #-4]!      @ R6
    STR R3, [R5, #-4]!      @ R5
    STR R3, [R5, #-4]!      @ R4
    
    @ บันทึก SP ใน TCB
    STR R5, [R4, #0]
    
    POP {R4, R5, PC}

task_exit:
    @ task should not return, loop forever
    B .

select_next_task:
    @ Simplified: return same task (real scheduler would round-robin)
    LDR R0, =current_task
    LDR R0, [R0]
    BX LR
```

---

## 18. การ Compile และ Run ด้วย QEMU

### Setup สำหรับ ARM Thumb

```bash
# ติดตั้ง toolchain
sudo apt-get install gcc-arm-linux-gnueabihf
sudo apt-get install qemu-user

# หรือสำหรับ bare-metal Cortex-M
sudo apt-get install gcc-arm-none-eabi
sudo apt-get install qemu-system-arm
```

### Makefile สำหรับ Thumb-2

```makefile
# Makefile สำหรับ ARM Thumb-2 Assembly

AS      = arm-linux-gnueabihf-as
LD      = arm-linux-gnueabihf-ld
QEMU    = qemu-arm

# Thumb-2 flags
ASFLAGS = -mcpu=cortex-a9 -mthumb -mfpu=neon -mfloat-abi=softfp

all: thumb2_demo

thumb2_demo: thumb2_demo.o
	$(LD) -o $@ $^

thumb2_demo.o: thumb2_demo.s
	$(AS) $(ASFLAGS) -o $@ $<

run: thumb2_demo
	$(QEMU) ./thumb2_demo

debug: thumb2_demo
	$(QEMU) -g 1234 ./thumb2_demo &
	arm-linux-gnueabihf-gdb -ex "target remote :1234" ./thumb2_demo

clean:
	rm -f *.o thumb2_demo
```

### โปรแกรมทดสอบที่ Run ได้จริง

```assembly
@ ===== thumb2_test.s: สำหรับ Linux/QEMU =====

.cpu cortex-a9
.thumb
.syntax unified

.section .data
msg_start:  .ascii "=== Thumb-2 Test ===\n"
msg_start_len = . - msg_start

msg_result: .ascii "Result: "
msg_result_len = . - msg_result

newline:    .ascii "\n"

.section .bss
buffer: .space 32

.section .text
.global _start

.thumb_func
_start:
    @ พิมพ์ header
    MOVS R0, #1             @ stdout
    LDR R1, =msg_start
    MOVS R2, #msg_start_len
    MOVS R7, #4             @ write syscall
    SVC #0
    
    @ ทดสอบ arithmetic
    MOVS R0, #10
    MOVS R1, #25
    BL test_arithmetic      @ R0 = result
    
    @ แปลง result เป็น string และแสดง
    BL print_decimal
    
    @ Exit
    MOVS R0, #0
    MOVS R7, #1
    SVC #0

@ R0, R1 = inputs, R0 = output
.thumb_func
test_arithmetic:
    @ (R0 * R1) + (R0 - R1) * 2
    MUL R2, R0, R1      @ R2 = R0 * R1
    SUB R3, R0, R1      @ R3 = R0 - R1
    ADD R3, R3, R3      @ R3 = R3 * 2
    ADD R0, R2, R3      @ result = R2 + R3
    BX LR

@ R0 = number to print
.thumb_func
print_decimal:
    PUSH {R4, R5, R6, LR}
    
    @ พิมพ์ "Result: "
    MOVS R4, #1
    LDR R5, =msg_result
    MOVS R6, #msg_result_len
    MOV R1, R5
    MOV R2, R6
    MOV R0, R4
    MOVS R7, #4
    SVC #0
    
    @ แปลง number เป็น string
    MOV R4, R0          @ save number
    LDR R5, =buffer
    
    @ ตรวจสอบ negative
    CMP R4, #0
    BGE .positive_num
    MOVS R6, #'-'
    STRB R6, [R5], #1
    NEGS R4, R4
    
.positive_num:
    @ แปลงเป็น decimal (reverse order)
    MOV R6, R5          @ save start of digits
    
.convert_loop:
    MOVS R0, R4
    MOVS R1, #10
    BL udiv             @ R0 = quotient, R1 = remainder
    ADDS R1, R1, #'0'
    STRB R1, [R5], #1
    MOV R4, R0
    CBNZ R4, .convert_loop
    
    @ reverse digits
    SUB R7, R5, #1      @ R7 = pointer to last digit
.reverse_loop:
    CMP R6, R7
    BGE .reverse_done
    LDRB R0, [R6]
    LDRB R1, [R7]
    STRB R1, [R6], #1
    STRB R0, [R7, #-1]!
    B .reverse_loop
.reverse_done:
    
    @ เพิ่ม newline
    MOVS R0, #'\n'
    STRB R0, [R5], #1
    
    @ พิมพ์
    MOVS R0, #1
    LDR R1, =buffer
    SUB R2, R5, R1      @ length
    MOVS R7, #4
    SVC #0
    
    POP {R4, R5, R6, PC}

@ Simple unsigned division: R0/R1 -> R0=quotient, R1=remainder
.thumb_func
udiv:
    MOVS R2, #0         @ quotient
.div_loop:
    CMP R0, R1
    BLT .div_done
    SUBS R0, R0, R1
    ADDS R2, R2, #1
    B .div_loop
.div_done:
    MOV R1, R0          @ remainder
    MOV R0, R2          @ quotient
    BX LR
```

```bash
# Compile และ Run
arm-linux-gnueabihf-as -mcpu=cortex-a9 -mthumb -o test.o thumb2_test.s
arm-linux-gnueabihf-ld -o test test.o
qemu-arm ./test
```

---

## 19. แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Thumb Register Constraints

```
โจทย์: แก้ code ต่อไปนี้ให้ compile ได้ใน Thumb-1 mode

ปัญหา:
    ADDS R8, R9, #5     @ ERROR: ใช้ hi reg กับ immediate
    LDRB R10, [R0]      @ ERROR: load to hi reg (Thumb-1)
    MULS R0, R8, R9     @ ERROR: MULS ใช้ได้แค่ lo reg

วิธีแก้:
```

```assembly
@ คำตอบ:
.thumb
fix_thumb_constraints:
    @ แทน: ADDS R8, R9, #5
    MOV R0, R9          @ copy R9 ไป lo reg
    ADDS R0, R0, #5     @ คำนวณใน lo reg
    MOV R8, R0          @ copy กลับ hi reg
    
    @ แทน: LDRB R10, [R0]
    LDRB R1, [R0]       @ load ไป lo reg
    MOV R10, R1         @ copy ไป hi reg
    
    @ แทน: MULS R0, R8, R9
    MOV R1, R8          @ copy R8 ไป lo reg
    MOV R2, R9          @ copy R9 ไป lo reg
    MULS R0, R1, R2     @ คูณใน lo regs
    
    BX LR
```

### แบบฝึกหัดที่ 2: IT Block

```
โจทย์: เขียนฟังก์ชัน clamp ด้วย IT block
int clamp(int val, int min, int max)
- ถ้า val < min: return min
- ถ้า val > max: return max
- ไม่งั้น: return val

ห้ามใช้ branch instructions (ใช้ IT block แทน)
```

```assembly
@ คำตอบ:
.thumb
.syntax unified

.thumb_func
clamp:
    @ R0 = val, R1 = min, R2 = max
    CMP R0, R1
    IT LT
    MOVLT R0, R1        @ if val < min: val = min
    
    CMP R0, R2
    IT GT
    MOVGT R0, R2        @ if val > max: val = max
    
    BX LR               @ return val (possibly clamped)

@ ทดสอบ
test_clamp:
    PUSH {LR}
    
    MOVS R0, #5
    MOVS R1, #1
    MOVS R2, #10
    BL clamp            @ clamp(5, 1, 10) = 5 (ไม่เปลี่ยน)
    @ R0 ควรเป็น 5
    
    MOVS R0, #0
    MOVS R1, #1
    MOVS R2, #10
    BL clamp            @ clamp(0, 1, 10) = 1 (clamp to min)
    @ R0 ควรเป็น 1
    
    MOVS R0, #15
    MOVS R1, #1
    MOVS R2, #10
    BL clamp            @ clamp(15, 1, 10) = 10 (clamp to max)
    @ R0 ควรเป็น 10
    
    POP {PC}
```

### แบบฝึกหัดที่ 3: Atomic Operations

```
โจทย์: เขียนฟังก์ชัน atomic_add ที่ thread-safe
int atomic_add(int *ptr, int value)
- บวก value ไปยัง *ptr
- return ค่าใหม่
- ต้องใช้ LDREX/STREX
```

```assembly
@ คำตอบ:
.thumb
.syntax unified

.thumb_func
atomic_add:
    @ R0 = ptr, R1 = value
    @ R0 = return (new value)
.retry:
    LDREX R2, [R0]      @ R2 = *ptr (exclusive)
    ADD R2, R2, R1      @ R2 = *ptr + value
    STREX R3, R2, [R0]  @ ลอง store
    CBZ R3, .success    @ ถ้า R3 == 0: สำเร็จ
    B .retry            @ ล้มเหลว: ลองใหม่
.success:
    DMB                 @ memory barrier
    MOV R0, R2          @ return new value
    BX LR
```

### แบบฝึกหัดที่ 4: TBB Jump Table

```
โจทย์: เขียน calculator ด้วย TBB
- รับ operation code: 0=add, 1=sub, 2=mul, 3=div
- รับ operands R1 และ R2
- ใช้ TBB สำหรับ dispatch
```

```assembly
@ คำตอบ:
.thumb
.syntax unified

.thumb_func
calculator:
    @ R0 = op (0-3), R1 = a, R2 = b
    @ R0 = result
    CMP R0, #3
    BHI .invalid_op
    TBB [PC, R0]
.op_table:
    .byte (.op_add - .op_table) / 2
    .byte (.op_sub - .op_table) / 2
    .byte (.op_mul - .op_table) / 2
    .byte (.op_div - .op_table) / 2
    .align 2

.op_add:
    ADD R0, R1, R2
    BX LR
.op_sub:
    SUB R0, R1, R2
    BX LR
.op_mul:
    MUL R0, R1, R2
    BX LR
.op_div:
    UDIV R0, R1, R2     @ Thumb-2: hardware division
    BX LR
.invalid_op:
    MOVS R0, #0
    BX LR
```

### แบบฝึกหัดที่ 5: Thumb-2 Optimization

```
โจทย์: Optimize ฟังก์ชันต่อไปนี้ให้ใช้ Thumb-2 features

Original (ARM):
    CMP R0, #0
    BEQ skip
    LDR R1, [R2]
    ADD R1, R1, #1
    STR R1, [R2]
skip:
    BX LR

ใช้ IT block และ CBZ/CBNZ เพื่อลด code size
```

```assembly
@ คำตอบ: Thumb-2 optimized version

.thumb
.syntax unified

.thumb_func
optimized_func:
    @ Version 1: ใช้ CBZ
    CBZ R0, .skip_v1    @ ถ้า R0 == 0: ข้ามไป
    LDR R1, [R2]
    ADDS R1, R1, #1
    STR R1, [R2]
.skip_v1:
    BX LR

.thumb_func
optimized_func_v2:
    @ Version 2: ใช้ IT block (ไม่มี branch เลย!)
    CBNZ R0, .do_work   @ ถ้า != 0: ทำงาน
    BX LR
.do_work:
    LDR R1, [R2]
    ADDS R1, R1, #1
    STR R1, [R2]
    BX LR
    
    @ หรือถ้าต้องการ branch-free ด้วย IT:
    @ แต่ IT block ทำงานกับ conditional execution เท่านั้น
    @ LDR ต้องการ base register จาก R2 ไม่สามารถ conditional skip ได้ง่ายๆ
```

---

## 20. สรุปและ Reference

### Thumb vs Thumb-2 Feature Comparison

```
Feature              | ARM-32 | Thumb-1 | Thumb-2
---------------------|--------|---------|--------
Instruction size     | 32-bit | 16-bit  | 16/32-bit mixed
Code density         | Low    | High    | High
Full register set    | Yes    | No      | Yes (32-bit)
Conditional exec     | All    | None    | IT block
Barrel shift         | Yes    | Limited | Yes (32-bit)
LDREX/STREX          | Yes    | No      | Yes
TBB/TBH              | No     | No      | Yes
CBZ/CBNZ             | No     | No      | Yes
WFI/WFE/SEV          | Yes    | No      | Yes
DSB/DMB/ISB          | Yes    | No      | Yes
LDRD/STRD            | Yes    | No      | Yes
Hardware divide      | No     | No      | Yes (Cortex-M3+)
State switching      | N/A    | BX      | Not needed
```

### Quick Reference: Common Thumb-2 Instructions

```assembly
@ Data Processing (16-bit)
MOVS Rd, #imm8          @ Rd = imm8, flags
ADDS Rd, Rn, #imm3      @ Rd = Rn + imm3, flags
SUBS Rd, Rn, #imm3      @ Rd = Rn - imm3, flags
MULS Rd, Rm, Rd         @ Rd = Rd * Rm, flags

@ Data Processing (32-bit)
MOV.W Rd, #imm16        @ Rd = imm16
MOVW Rd, #imm16         @ same
MOVT Rd, #imm16         @ Rd[31:16] = imm16
MUL Rd, Rn, Rm          @ any registers
UDIV Rd, Rn, Rm         @ unsigned divide
SDIV Rd, Rn, Rm         @ signed divide

@ Load/Store (16-bit)
LDR Rd, [Rn, #imm5*4]   @ Rd = [Rn + imm5*4]
STR Rd, [Rn, #imm5*4]   @ [Rn + imm5*4] = Rd

@ Load/Store (32-bit Thumb-2)
LDR.W Rd, [Rn, #imm12]  @ larger offset
LDRB.W Rd, [Rn, #-imm8] @ negative offset
LDR Rd, [Rn], #imm      @ post-increment
LDR Rd, [Rn, #imm]!     @ pre-increment (writeback)

@ Branches
CBZ Rn, label           @ if Rn == 0: branch
CBNZ Rn, label          @ if Rn != 0: branch
TBB [Rn, Rm]            @ table branch byte
TBH [Rn, Rm, LSL #1]    @ table branch halfword

@ Synchronization
LDREX Rd, [Rn]          @ exclusive load
STREX Rd, Rm, [Rn]      @ exclusive store
CLREX                   @ clear exclusive

@ Barriers
DMB                     @ data memory barrier
DSB                     @ data sync barrier
ISB                     @ instruction sync barrier

@ Power
WFI                     @ wait for interrupt
WFE                     @ wait for event
SEV                     @ send event
```

### Thumb-2 Assembly Directives

```assembly
@ ใช้ directives เหล่านี้สำหรับ Thumb-2

.cpu cortex-m4          @ กำหนด target CPU
.thumb                  @ เปิดใช้ Thumb mode
.syntax unified         @ ใช้ unified assembly syntax (แนะนำ)
.thumb_func             @ บอกว่า function ต่อไปเป็น Thumb
.code 16                @ เหมือน .thumb (สำหรับ ARM assembler)
.code 32                @ เหมือน .arm (สำหรับ ARM assembler)

@ สำหรับ interworking
.arm                    @ เปลี่ยนกลับ ARM mode (GNU assembler)
.align 2                @ align ที่ 4-byte boundary
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **Thumb**: instruction set 16-bit ที่ประหยัด code space ~30-40%
2. **ARM/Thumb State**: สลับด้วย BX/BLX instruction ผ่าน address bit[0]
3. **Thumb Register Constraints**: Lo regs (R0-R7) ใช้ได้กว้าง, Hi regs (R8-R15) จำกัด
4. **Thumb-2**: ผสม 16/32-bit instructions สำหรับ code density + performance
5. **IT Block**: conditional execution ใน Thumb-2 state
6. **W Suffix**: บังคับ 32-bit encoding
7. **LDREX/STREX**: atomic operations สำหรับ synchronization
8. **TBB/TBH**: efficient jump tables
9. **WFI/WFE/SEV**: power management
10. **DMB/DSB/ISB**: memory ordering barriers
11. **Cortex-M**: Thumb-2 only processors สำหรับ embedded systems

Thumb-2 คือ instruction set ที่ทรงพลังที่สุดสำหรับ ARM embedded systems เพราะได้ทั้ง code density และประสิทธิภาพสูง

---

**ต่อไป: Part 038 - ARM NEON SIMD Programming**
