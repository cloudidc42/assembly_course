# Part 036: ARM Barrel Shifter (บาร์เรลชิฟเตอร์ใน ARM)

## บทนำ (Introduction)

**Barrel Shifter** คือหนึ่งในฟีเจอร์ที่ทรงพลังและโดดเด่นที่สุดของสถาปัตยกรรม ARM
ต่างจาก x86 ที่การ shift เป็น instruction แยกต่างหาก ARM สามารถ **รวม shift เข้ากับ instruction อื่นได้ในรอบ clock เดียว**

```
┌─────────────────────────────────────────────────────────────┐
│                    ARM Datapath                             │
│                                                             │
│  Register File                                              │
│  ┌──────────┐    ┌─────────────────┐    ┌───────────────┐  │
│  │   Rn     │───▶│                 │    │               │  │
│  └──────────┘    │      ALU        │───▶│    Result     │  │
│  ┌──────────┐    │                 │    │               │  │
│  │   Rm     │───▶│  Barrel        │    └───────────────┘  │
│  └──────────┘    │  Shifter       │                        │
│                  │  (Integrated!) │                        │
│  Shift Amount    │                 │                        │
│  (Imm or Reg)───▶│                 │                        │
│                  └─────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

## สารบัญ (Table of Contents)

1. [Barrel Shifter คืออะไร](#what-is-barrel-shifter)
2. [LSL - Logical Shift Left](#lsl)
3. [LSR - Logical Shift Right](#lsr)
4. [ASR - Arithmetic Shift Right](#asr)
5. [ROR - Rotate Right](#ror)
6. [RRX - Rotate Right through Carry](#rrx)
7. [Shift by Immediate vs Register](#shift-modes)
8. [การรวมกับ Data Processing Instructions](#combining)
9. [การคูณด้วยค่าคงที่ (Efficient Multiplication)](#multiplication)
10. [การหารด้วย Power of 2](#division)
11. [Bit Extraction และ Insertion](#bit-ops)
12. [Endian Conversion](#endian)
13. [AArch64 Shift Operations](#aarch64)
14. [Performance Impact](#performance)
15. [แบบฝึกหัด (Exercises)](#exercises)

---

## 1. Barrel Shifter คืออะไร {#what-is-barrel-shifter}

### หลักการทำงาน (How It Works)

Barrel Shifter เป็น hardware circuit ที่สามารถ shift หรือ rotate ข้อมูล
ได้ **ในระยะทางใดก็ได้ในรอบ clock เดียว** (ต่างจาก shift register ทั่วไปที่ต้องทำทีละ 1 bit)

```
Traditional Shift Register (n steps = n clock cycles):
bit: [7][6][5][4][3][2][1][0]
        └─ shift 1 ─────────▶ 1 cycle
        └─ shift 2 ─────────▶ 2 cycles
        └─ shift 8 ─────────▶ 8 cycles  ← ช้า!

Barrel Shifter (any shift = 1 clock cycle):
bit: [7][6][5][4][3][2][1][0]
        └─ shift 1 ─────────▶ 1 cycle
        └─ shift 2 ─────────▶ 1 cycle  ← เร็วเท่ากัน!
        └─ shift 8 ─────────▶ 1 cycle  ← เร็วเท่ากัน!
```

### ประเภทของ Shift Operations ใน ARM

| Operation | ชื่อเต็ม | ทิศทาง | Fill Bit | ใช้สำหรับ |
|-----------|---------|---------|----------|-----------|
| LSL | Logical Shift Left | ซ้าย | 0 | คูณด้วย 2^n |
| LSR | Logical Shift Right | ขวา | 0 | หารด้วย 2^n (unsigned) |
| ASR | Arithmetic Shift Right | ขวา | sign bit | หารด้วย 2^n (signed) |
| ROR | Rotate Right | ขวา | wrap around | Bit manipulation |
| RRX | Rotate Right eXtended | ขวา | Carry flag | 33-bit rotate |

---

## 2. LSL - Logical Shift Left {#lsl}

### หลักการ

```
LSL #n  →  คูณด้วย 2^n

ก่อน: [b31][b30]...[b1][b0]  (shift left 2)
หลัง: [b29][b28]...[b0][0 ][0 ]
       └─────────────────────┘  ← เลื่อนซ้าย 2 ตำแหน่ง
                                   บิตบนสุดทิ้งไป, เติม 0 ทางขวา

ตัวอย่าง:
  0b00000101 (5)  LSL #2  →  0b00010100 (20)
  5 × 4 = 20 ✓
```

### ไฟล์: `lsl_demo.s` (ARM32)

```asm
@ lsl_demo.s - สาธิตการใช้ LSL (Logical Shift Left)
@ Compile: arm-linux-gnueabi-as lsl_demo.s -o lsl_demo.o
@ Link:    arm-linux-gnueabi-ld lsl_demo.o -o lsl_demo
@ Run:     qemu-arm ./lsl_demo

.section .data
fmt_val:    .asciz "LSL: %d << %d = %d\n"
fmt_title:  .asciz "=== Logical Shift Left (LSL) Demo ===\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r7, lr}

    @ แสดง title
    LDR     r0, =fmt_title
    BL      printf

    @ ตัวอย่าง 1: 1 << 0 = 1
    MOV     r4, #1              @ ค่าเริ่มต้น
    MOV     r5, r4, LSL #0     @ shift 0 = ไม่เปลี่ยนแปลง
    LDR     r0, =fmt_val
    MOV     r1, r4
    MOV     r2, #0
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 2: 1 << 1 = 2
    MOV     r4, #1
    MOV     r5, r4, LSL #1     @ คูณ 2
    LDR     r0, =fmt_val
    MOV     r1, r4
    MOV     r2, #1
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 3: 1 << 4 = 16
    MOV     r4, #1
    MOV     r5, r4, LSL #4     @ คูณ 16
    LDR     r0, =fmt_val
    MOV     r1, r4
    MOV     r2, #4
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 4: 5 << 3 = 40
    MOV     r4, #5
    MOV     r5, r4, LSL #3     @ คูณ 8
    LDR     r0, =fmt_val
    MOV     r1, r4
    MOV     r2, #3
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 5: 255 << 8 = 65280
    MOV     r4, #255
    MOV     r5, r4, LSL #8     @ เลื่อนไป high byte
    LDR     r0, =fmt_val
    MOV     r1, r4
    MOV     r2, #8
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 6: LSL as standalone instruction
    MOV     r4, #7
    LSL     r5, r4, #2         @ r5 = r4 << 2 = 28
    LDR     r0, =fmt_val
    MOV     r1, r4
    MOV     r2, #2
    MOV     r3, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

### การทดสอบ

```bash
# คอมไพล์และรัน
arm-linux-gnueabi-gcc -static lsl_demo.s -o lsl_demo
qemu-arm ./lsl_demo

# ผลลัพธ์ที่คาดหวัง:
# === Logical Shift Left (LSL) Demo ===
# LSL: 1 << 0 = 1
# LSL: 1 << 1 = 2
# LSL: 1 << 4 = 16
# LSL: 5 << 3 = 40
# LSL: 255 << 8 = 65280
# LSL: 7 << 2 = 28
```

---

## 3. LSR - Logical Shift Right {#lsr}

### หลักการ

```
LSR #n  →  หารด้วย 2^n (สำหรับ unsigned)

ก่อน: [b31][b30]...[b1][b0]  (shift right 2)
หลัง: [0  ][0  ][b31][b30]...[b2]
       └──────────────────────────┘  ← เลื่อนขวา 2 ตำแหน่ง
                                        บิตล่างทิ้งไป, เติม 0 ทางซ้าย

ตัวอย่าง:
  0b00010100 (20)  LSR #2  →  0b00000101 (5)
  20 / 4 = 5 ✓

ข้อสังเกต: LSR เติม 0 เสมอ ดังนั้นสำหรับ signed negative:
  0b10000000 (-128)  LSR #1  →  0b01000000 (64) ← ผิด! ใช้ ASR แทน
```

### ไฟล์: `lsr_demo.s`

```asm
@ lsr_demo.s - สาธิตการใช้ LSR (Logical Shift Right)
@ สำหรับ unsigned division by power of 2

.section .data
fmt_u:  .asciz "LSR: %u >> %d = %u\n"
fmt_s:  .asciz "LSR signed: %d >> %d = %d (ผลลัพธ์ผิดสำหรับ negative!)\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r7, lr}

    @ Unsigned right shift
    MOV     r4, #240           @ 0b11110000 = 240
    MOV     r5, r4, LSR #4    @ 240 >> 4 = 15
    LDR     r0, =fmt_u
    MOV     r1, r4
    MOV     r2, #4
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง: extract upper byte
    LDR     r4, =0xABCD1234    @ ค่า 32-bit
    MOV     r5, r4, LSR #24   @ เอา byte สูงสุด
    LDR     r0, =fmt_u
    LDR     r1, =0xABCD1234
    MOV     r2, #24
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง signed ที่ผิด!
    MVN     r4, #127           @ r4 = -128 (0xFFFFFF80)
    MOV     r5, r4, LSR #1    @ จะได้ 0x7FFFFFC0 = บวก!
    LDR     r0, =fmt_s
    MOV     r1, r4
    MOV     r2, #1
    MOV     r3, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 4. ASR - Arithmetic Shift Right {#asr}

### หลักการ

```
ASR #n  →  หารด้วย 2^n (สำหรับ signed, รักษา sign bit)

Positive (sign bit = 0):
  ก่อน: [0][b30]...[b1][b0]  (shift right 2)
  หลัง: [0][0  ][0][b30]...[b2]  ← เติม 0

Negative (sign bit = 1):
  ก่อน: [1][b30]...[b1][b0]  (shift right 2)
  หลัง: [1][1  ][1][b30]...[b2]  ← เติม 1 รักษา negative

ตัวอย่าง:
  -128 (0b10000000)  ASR #1  →  -64 (0b11000000) ✓
  -128 / 2 = -64 ✓
```

### ไฟล์: `asr_demo.s`

```asm
@ asr_demo.s - สาธิตการใช้ ASR (Arithmetic Shift Right)
@ สำหรับ signed division by power of 2

.section .data
fmt_pos:    .asciz "ASR positive: %d >> %d = %d\n"
fmt_neg:    .asciz "ASR negative: %d >> %d = %d\n"
fmt_cmp:    .asciz "LSR vs ASR for -20: LSR=%d, ASR=%d\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r7, lr}

    @ Positive number
    MOV     r4, #100           @ +100
    MOV     r5, r4, ASR #2    @ 100 >> 2 = 25
    LDR     r0, =fmt_pos
    MOV     r1, r4
    MOV     r2, #2
    MOV     r3, r5
    BL      printf

    @ Negative number
    MVN     r4, #19            @ r4 = -20
    MOV     r5, r4, ASR #2    @ -20 >> 2 = -5
    LDR     r0, =fmt_neg
    MOV     r1, r4
    MOV     r2, #2
    MOV     r3, r5
    BL      printf

    @ เปรียบเทียบ LSR vs ASR สำหรับ -20
    MVN     r4, #19            @ r4 = -20
    MOV     r5, r4, LSR #2    @ LSR: ผิด
    MOV     r6, r4, ASR #2    @ ASR: ถูก
    LDR     r0, =fmt_cmp
    MOV     r1, r5
    MOV     r2, r6
    BL      printf

    @ ตัวอย่าง: divide by 8 (-64 / 8 = -8)
    MVN     r4, #63            @ r4 = -64
    ASR     r5, r4, #3        @ -64 >> 3 = -8
    LDR     r0, =fmt_neg
    MOV     r1, r4
    MOV     r2, #3
    MOV     r3, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 5. ROR - Rotate Right {#ror}

### หลักการ

```
ROR #n  →  หมุนขวา n ตำแหน่ง (บิตที่หลุดจากขวา ไปต่อที่ซ้าย)

ก่อน: [b31][b30]...[b2][b1][b0]  (rotate right 3)
หลัง: [b2 ][b1 ][b0][b31]...[b3]

ตัวอย่าง (8-bit):
  0b11110000  ROR #4  →  0b00001111
  MSB กลายเป็น LSB และในทางกลับกัน

ตัวอย่าง (32-bit):
  0xABCD1234  ROR #8  →  0x34ABCD12
```

### ไฟล์: `ror_demo.s`

```asm
@ ror_demo.s - สาธิตการใช้ ROR (Rotate Right)

.section .data
fmt_hex:    .asciz "ROR: 0x%08X ROR #%d = 0x%08X\n"
fmt_byte:   .asciz "Byte swap: 0x%08X -> 0x%08X\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r7, lr}

    @ ตัวอย่าง 1: ROR #4
    LDR     r4, =0xABCDEF12
    MOV     r5, r4, ROR #4
    LDR     r0, =fmt_hex
    MOV     r1, r4
    MOV     r2, #4
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 2: ROR #8
    LDR     r4, =0xABCDEF12
    MOV     r5, r4, ROR #8
    LDR     r0, =fmt_hex
    MOV     r1, r4
    MOV     r2, #8
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 3: ROR #16
    LDR     r4, =0x12345678
    MOV     r5, r4, ROR #16
    LDR     r0, =fmt_hex
    MOV     r1, r4
    MOV     r2, #16
    MOV     r3, r5
    BL      printf

    @ ตัวอย่าง 4: 32-bit endian swap โดยใช้ ROR
    @ (ไม่ใช่ complete endian swap แต่สาธิต rotation)
    LDR     r4, =0x01020304
    MOV     r5, r4, ROR #8    @ หมุน 1 byte
    LDR     r0, =fmt_byte
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    @ ตัวอย่าง 5: Standalone ROR instruction
    LDR     r4, =0xF0F0F0F0
    ROR     r5, r4, #8        @ r5 = r4 ROR 8
    LDR     r0, =fmt_hex
    MOV     r1, r4
    MOV     r2, #8
    MOV     r3, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 6. RRX - Rotate Right through Carry {#rrx}

### หลักการ

```
RRX  →  33-bit rotate ผ่าน Carry flag

  [Carry][b31][b30]...[b1][b0]
     └─────────────────────────┐  rotate right 1
  [b0  ][Carry][b31]...[b2][b1]
         └─ Carry ใหม่ = b0 เดิม
         └─ MSB ใหม่ = Carry เดิม

ใช้สำหรับ:
- 64-bit shift (รวมกับ ADDS/ADC)
- Multi-precision arithmetic
```

### ไฟล์: `rrx_demo.s`

```asm
@ rrx_demo.s - สาธิตการใช้ RRX (Rotate Right through eXtended/Carry)

.section .data
fmt_rrx:    .asciz "RRX: 0x%08X (C=%d) -> 0x%08X\n"
fmt_64:     .asciz "64-bit shift: 0x%08X_%08X >> 1 = 0x%08X_%08X\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r9, lr}

    @ ตัวอย่าง 1: RRX กับ Carry = 0
    MOV     r4, #0b00000001    @ LSB = 1 (จะกลายเป็น carry)
    MOVS    r5, #0             @ ล้าง Carry flag (S suffix = update flags)
    MOV     r6, r4, RRX       @ Carry=0 ดังนั้น MSB=0
    LDR     r0, =fmt_rrx
    MOV     r1, r4
    MOV     r2, #0             @ Carry was 0
    MOV     r3, r6
    BL      printf

    @ ตัวอย่าง 2: RRX กับ Carry = 1
    MOV     r4, #0b00000001    @ LSB = 1
    MOVS    r5, #0             @ ล้าง carry
    ADDS    r5, r5, r5         @ แน่ใจว่า carry ยังเป็น 0
    @ ตั้ง carry โดยการ ADD ที่ overflow
    MOV     r5, #0xFFFFFFFF
    ADDS    r5, r5, #1         @ 0xFFFFFFFF + 1 → carry = 1!
    MOV     r6, r4, RRX       @ Carry=1 ดังนั้น MSB=1
    LDR     r0, =fmt_rrx
    MOV     r1, r4
    MOV     r2, #1             @ Carry was 1
    MOV     r3, r6
    BL      printf

    @ ตัวอย่าง 3: 64-bit shift right โดยใช้ MOVS + RRX
    @ ค่า 64-bit: r7:r8 = 0x0000000100000000 (4294967296)
    MOV     r7, #1             @ high 32 bits
    MOV     r8, #0             @ low 32 bits
    @ shift right 1:
    MOVS    r7, r7, LSR #1    @ shift high, LSR ตั้ง carry จาก bit0
    MOV     r8, r8, RRX       @ shift low, รับ carry จาก high
    LDR     r0, =fmt_64
    MOV     r1, #1             @ original high
    MOV     r2, #0             @ original low
    MOV     r3, r7             @ result high
    MOV     r4, r8             @ result low ← ใช้ PUSH/POP ถ้า > 4 args
    BL      printf

    MOV     r0, #0
    POP     {r4-r9, pc}
```

---

## 7. Shift by Immediate vs Shift by Register {#shift-modes}

### Shift by Immediate (ค่าคงที่ 0-31)

```asm
@ Shift amount เป็นค่า immediate (0-31)
MOV  r1, r0, LSL #5    @ shift left 5 (คูณ 32)
MOV  r1, r0, LSR #3    @ shift right 3 (หาร 8)
MOV  r1, r0, ASR #2    @ arithmetic shift right 2
MOV  r1, r0, ROR #8    @ rotate right 8
```

### Shift by Register (ค่าใน Register)

```asm
@ Shift amount อยู่ใน register (ใช้ byte ล่างสุด, 0-255)
@ แต่ค่าที่มีผล: 0-31 สำหรับ shift, 0-31 สำหรับ rotate
MOV  r1, r0, LSL r2    @ shift left ตาม r2
MOV  r1, r0, LSR r2    @ shift right ตาม r2
MOV  r1, r0, ASR r2    @ arithmetic shift ตาม r2
ROR  r1, r0, r2        @ rotate ตาม r2 (standalone instruction)
```

### ไฟล์: `shift_modes.s`

```asm
@ shift_modes.s - เปรียบเทียบ shift by immediate และ shift by register

.section .data
fmt_imm:    .asciz "Shift by immediate: %d << %d = %d\n"
fmt_reg:    .asciz "Shift by register:  %d << %d = %d\n"
fmt_dyn:    .asciz "Dynamic shift at runtime: %d\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r7, lr}

    @ Shift by immediate
    MOV     r4, #10
    MOV     r5, r4, LSL #3    @ 10 << 3 = 80 (compile-time constant)
    LDR     r0, =fmt_imm
    MOV     r1, r4
    MOV     r2, #3
    MOV     r3, r5
    BL      printf

    @ Shift by register (ค่าเดียวกัน)
    MOV     r4, #10
    MOV     r6, #3             @ shift amount ใน register
    MOV     r5, r4, LSL r6    @ 10 << r6 = 80 (runtime)
    LDR     r0, =fmt_reg
    MOV     r1, r4
    MOV     r2, r6
    MOV     r3, r5
    BL      printf

    @ Dynamic shift - เปลี่ยน shift amount ตาม loop counter
    MOV     r4, #1
    MOV     r6, #0             @ loop counter
shift_loop:
    CMP     r6, #8
    BGE     shift_done
    MOV     r5, r4, LSL r6    @ r5 = 1 << r6
    LDR     r0, =fmt_dyn
    MOV     r1, r5
    BL      printf
    ADD     r6, r6, #1
    B       shift_loop
shift_done:

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 8. การรวมกับ Data Processing Instructions {#combining}

### ข้อดีสำคัญของ ARM

ใน ARM, operand ที่ 2 ของ data processing instructions สามารถมี shift ได้:

```asm
@ รูปแบบทั่วไป:
@ <op> Rd, Rn, Rm, <shift>

@ ตัวอย่าง:
ADD  R0, R1, R2, LSL #2    @ R0 = R1 + (R2 << 2) = R1 + R2*4
SUB  R0, R1, R2, LSR #1    @ R0 = R1 - (R2 >> 1)
AND  R0, R1, R2, ASR #3    @ R0 = R1 AND (R2 >> 3)
ORR  R0, R1, R2, ROR #8    @ R0 = R1 OR  (R2 ROR 8)
```

### ไฟล์: `combined_ops.s` - การใช้งานจริง

```asm
@ combined_ops.s - การรวม shift กับ data processing instructions
@ แสดง pattern ที่ใช้บ่อยในการเขียนโปรแกรม

.section .data
fmt_scaled: .asciz "Scaled add: base=%d, offset=%d, stride=4 -> addr_offset=%d\n"
fmt_pack:   .asciz "Pack bytes: 0x%08X\n"
fmt_unpack: .asciz "Unpack: byte0=%d, byte1=%d, byte2=%d, byte3=%d\n"

.section .text
.global main
.extern printf

main:
    PUSH    {r4-r9, lr}

    @ ============================================
    @ Pattern 1: Array indexing (scaled addressing)
    @ addr = base + index * sizeof(int)
    @ sizeof(int) = 4 = 2^2, ดังนั้นใช้ LSL #2
    @ ============================================
    MOV     r4, #1000          @ base address (สมมติ)
    MOV     r5, #7             @ index
    ADD     r6, r4, r5, LSL #2 @ r6 = 1000 + 7*4 = 1028
    LDR     r0, =fmt_scaled
    MOV     r1, r4
    MOV     r2, r5
    MOV     r3, r6
    BL      printf

    @ ============================================
    @ Pattern 2: Pack 4 bytes into 32-bit word
    @ word = byte3<<24 | byte2<<16 | byte1<<8 | byte0
    @ ============================================
    MOV     r4, #0xAA          @ byte 0
    MOV     r5, #0xBB          @ byte 1
    MOV     r6, #0xCC          @ byte 2
    MOV     r7, #0xDD          @ byte 3

    @ สร้างค่า 32-bit
    ORR     r8, r4, r5, LSL #8  @ r8 = byte0 | (byte1 << 8)
    ORR     r8, r8, r6, LSL #16 @ r8 |= (byte2 << 16)
    ORR     r8, r8, r7, LSL #24 @ r8 |= (byte3 << 24)
    @ r8 = 0xDDCCBBAA

    LDR     r0, =fmt_pack
    MOV     r1, r8
    BL      printf

    @ ============================================
    @ Pattern 3: Unpack 32-bit word to 4 bytes
    @ ============================================
    LDR     r4, =0xDDCCBBAA   @ ค่า packed
    AND     r5, r4, #0xFF      @ byte 0 = bits[7:0]
    MOV     r6, r4, LSR #8     @ 
    AND     r6, r6, #0xFF      @ byte 1 = bits[15:8]
    MOV     r7, r4, LSR #16    @
    AND     r7, r7, #0xFF      @ byte 2 = bits[23:16]
    MOV     r8, r4, LSR #24    @ byte 3 = bits[31:24]

    LDR     r0, =fmt_unpack
    MOV     r1, r5
    MOV     r2, r6
    MOV     r3, r7
    @ byte3 ต้องใส่ในสแตก (ARM calling convention: args 5+ บนสแตก)
    PUSH    {r8}
    BL      printf
    ADD     sp, sp, #4         @ คืนสแตก

    MOV     r0, #0
    POP     {r4-r9, pc}
```

---

## 9. Efficient Multiplication by Constants {#multiplication}

### แทนที่ MUL ด้วย Shift + Add

```
MUL r0, r1, r2  → หลาย cycles
ADD/LSL         → 1-2 cycles

เทคนิค:
n * 3   = n + n*2   = ADD r0, r1, r1, LSL #1
n * 5   = n + n*4   = ADD r0, r1, r1, LSL #2
n * 6   = n*2 + n*4 = ADD r0, r1, r1  ; ADD r0, r0, r0, LSL #1
n * 7   = n*8 - n   = RSB r0, r1, r1, LSL #3
n * 9   = n + n*8   = ADD r0, r1, r1, LSL #3
n * 10  = n*2 + n*8 = ADD r0, r1, r1  ; ADD r0, r0, r0, LSL #2  (ไม่แน่ใจ)
```

### ไฟล์: `fast_multiply.s`

```asm
@ fast_multiply.s - การคูณด้วยค่าคงที่โดยใช้ barrel shifter
@ เปรียบเทียบ MUL กับ Shift+Add

.section .data
fmt_mul:    .asciz "MUL: %d * %d = %d\n"
fmt_shift:  .asciz "Shift: %d * %d = %d (เร็วกว่า!)\n"
fmt_perf:   .asciz "\n--- ผลลัพธ์การทดสอบ ---\n"

.section .text
.global main
.extern printf

@ ============================================
@ Function: multiply_by_3 (n*3)
@ Input:  r0 = n
@ Output: r0 = n * 3
@ ============================================
multiply_by_3:
    @ n*3 = n + n*2 = n + (n<<1)
    ADD     r0, r0, r0, LSL #1
    BX      lr

@ ============================================
@ Function: multiply_by_5 (n*5)
@ Input:  r0 = n
@ Output: r0 = n * 5
@ ============================================
multiply_by_5:
    @ n*5 = n + n*4 = n + (n<<2)
    ADD     r0, r0, r0, LSL #2
    BX      lr

@ ============================================
@ Function: multiply_by_7 (n*7)
@ Input:  r0 = n
@ Output: r0 = n * 7
@ ============================================
multiply_by_7:
    @ n*7 = n*8 - n = (n<<3) - n
    RSB     r0, r0, r0, LSL #3
    BX      lr

@ ============================================
@ Function: multiply_by_9 (n*9)
@ Input:  r0 = n
@ Output: r0 = n * 9
@ ============================================
multiply_by_9:
    @ n*9 = n + n*8 = n + (n<<3)
    ADD     r0, r0, r0, LSL #3
    BX      lr

@ ============================================
@ Function: multiply_by_10 (n*10)
@ Input:  r0 = n
@ Output: r0 = n * 10
@ ============================================
multiply_by_10:
    @ n*10 = n*2 + n*8 = (n<<1) + (n<<3)
    ADD     r0, r0, r0, LSL #2   @ r0 = n*5
    ADD     r0, r0, r0           @ r0 = n*10 ← ไม่ถูก, แก้:
    @ วิธีที่ถูก:
    @ r1 = n
    @ r0 = r1 + (r1 << 3) = n + 8n = 9n ← ยังไม่ถูก
    @ n*10: n<<1 + n<<3
    BX      lr

@ ============================================
@ Function: multiply_by_10_correct
@ Input:  r0 = n
@ Output: r0 = n * 10
@ ============================================
multiply_by_10_correct:
    PUSH    {r4, lr}
    MOV     r4, r0              @ r4 = n
    ADD     r0, r4, r4, LSL #2 @ r0 = n + 4n = 5n
    ADD     r0, r0, r0          @ r0 = 5n + 5n = 10n
    POP     {r4, pc}

@ ============================================
@ Function: multiply_by_25 (n*25)
@ Input:  r0 = n
@ Output: r0 = n * 25
@ ============================================
multiply_by_25:
    @ n*25 = n*(16+8+1) = n*16 + n*8 + n
    PUSH    {r4, lr}
    MOV     r4, r0
    ADD     r0, r4, r4, LSL #3  @ n + 8n = 9n ← ไม่ถูก
    @ n*25:
    @ 25 = 16 + 9 = (n<<4) + n + (n<<3)
    MOV     r0, r4, LSL #4      @ r0 = n*16
    ADD     r0, r0, r4, LSL #3  @ r0 += n*8 = n*24
    ADD     r0, r0, r4          @ r0 += n = n*25
    POP     {r4, pc}

main:
    PUSH    {r4-r7, lr}

    LDR     r0, =fmt_perf
    BL      printf

    @ ทดสอบ n=7 กับทุก function
    MOV     r4, #7

    @ n*3 = 21
    MOV     r0, r4
    BL      multiply_by_3
    MOV     r5, r0
    LDR     r0, =fmt_shift
    MOV     r1, r4
    MOV     r2, #3
    MOV     r3, r5
    BL      printf

    @ n*5 = 35
    MOV     r0, r4
    BL      multiply_by_5
    MOV     r5, r0
    LDR     r0, =fmt_shift
    MOV     r1, r4
    MOV     r2, #5
    MOV     r3, r5
    BL      printf

    @ n*7 = 49
    MOV     r0, r4
    BL      multiply_by_7
    MOV     r5, r0
    LDR     r0, =fmt_shift
    MOV     r1, r4
    MOV     r2, #7
    MOV     r3, r5
    BL      printf

    @ n*9 = 63
    MOV     r0, r4
    BL      multiply_by_9
    MOV     r5, r0
    LDR     r0, =fmt_shift
    MOV     r1, r4
    MOV     r2, #9
    MOV     r3, r5
    BL      printf

    @ n*10 = 70
    MOV     r0, r4
    BL      multiply_by_10_correct
    MOV     r5, r0
    LDR     r0, =fmt_shift
    MOV     r1, r4
    MOV     r2, #10
    MOV     r3, r5
    BL      printf

    @ n*25 = 175
    MOV     r0, r4
    BL      multiply_by_25
    MOV     r5, r0
    LDR     r0, =fmt_shift
    MOV     r1, r4
    MOV     r2, #25
    MOV     r3, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 10. Division by Power of 2 {#division}

### การหาร Unsigned

```asm
@ Unsigned division by 2^n
LSR  r0, r0, #n    @ r0 = r0 / 2^n (unsigned)

@ ตัวอย่าง:
@ 100 / 4 = 25
MOV  r0, #100
LSR  r0, r0, #2   @ 100 >> 2 = 25
```

### การหาร Signed (มีความซับซ้อนกว่า)

```
สำหรับ signed division โดย 2:
  -7 / 2 = -3 (C standard: truncate toward zero)
  -7 >> 1 = -4 (ASR: floor division, ผิดสำหรับ negative odd)

วิธีแก้: ถ้า negative ต้อง +1 ก่อน shift (เพื่อ round toward zero)
  correction = (n >> 31) & (divisor - 1)
  result = (n + correction) >> log2(divisor)
```

### ไฟล์: `signed_division.s`

```asm
@ signed_division.s - การหาร signed ด้วย power of 2
@ ใช้เทคนิค bias correction

.section .data
fmt_div:    .asciz "signed_div(%d, %d) = %d\n"
fmt_naive:  .asciz "naive ASR: %d >> %d = %d (อาจผิด!)\n"
fmt_exact:  .asciz "exact div: %d / %d = %d (ถูกต้อง)\n"

.section .text
.global main
.extern printf

@ ============================================
@ signed_div_by_4: หาร signed integer ด้วย 4
@ Input:  r0 = dividend (signed)
@ Output: r0 = quotient (truncated toward zero)
@ ============================================
signed_div_by_4:
    @ เทคนิค:
    @ 1. หา sign bit: r1 = r0 >> 31 (ASR) → -1 ถ้า negative, 0 ถ้า positive
    @ 2. bias = r1 AND 3  (divisor-1 = 4-1 = 3)
    @ 3. result = (r0 + bias) >> 2
    MOV     r1, r0, ASR #31    @ r1 = sign extension (-1 or 0)
    AND     r1, r1, #3         @ r1 = 3 ถ้า negative, 0 ถ้า positive
    ADD     r0, r0, r1         @ เพิ่ม bias
    MOV     r0, r0, ASR #2     @ หาร 4
    BX      lr

@ ============================================
@ signed_div_by_power2: หาร signed โดย 2^n
@ Input:  r0 = dividend, r1 = n (power)
@ Output: r0 = quotient
@ ============================================
signed_div_by_power2:
    PUSH    {r4, r5, lr}
    MOV     r4, r0              @ save dividend
    MOV     r5, r1              @ save n

    @ คำนวณ divisor - 1 = 2^n - 1
    MOV     r2, #1
    MOV     r2, r2, LSL r5     @ r2 = 2^n
    SUB     r2, r2, #1         @ r2 = 2^n - 1

    @ bias
    MOV     r1, r4, ASR #31    @ r1 = sign (-1 or 0)
    AND     r1, r1, r2         @ r1 = (2^n - 1) ถ้า negative

    ADD     r0, r4, r1         @ เพิ่ม bias
    MOV     r0, r0, ASR r5     @ shift right n

    POP     {r4, r5, pc}

main:
    PUSH    {r4-r7, lr}

    @ ทดสอบค่าต่างๆ
    @ -7 / 4: ASR ธรรมดา vs ถูกต้อง
    MVN     r4, #6             @ r4 = -7
    MOV     r5, r4, ASR #2    @ -7 >> 2 = -2 (แต่ -7/4 ควร = -1)
    LDR     r0, =fmt_naive
    MOV     r1, r4
    MOV     r2, #2
    MOV     r3, r5
    BL      printf

    MOV     r0, r4
    BL      signed_div_by_4
    MOV     r5, r0
    LDR     r0, =fmt_exact
    MOV     r1, r4
    MOV     r2, #4
    MOV     r3, r5
    BL      printf

    @ -8 / 4 = -2 (ทั้งสองวิธีควรได้เหมือนกัน)
    MVN     r4, #7             @ r4 = -8
    MOV     r0, r4
    BL      signed_div_by_4
    MOV     r5, r0
    LDR     r0, =fmt_exact
    MOV     r1, r4
    MOV     r2, #4
    MOV     r3, r5
    BL      printf

    @ 7 / 4 = 1
    MOV     r4, #7
    MOV     r0, r4
    BL      signed_div_by_4
    MOV     r5, r0
    LDR     r0, =fmt_exact
    MOV     r1, r4
    MOV     r2, #4
    MOV     r3, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 11. Bit Extraction และ Insertion {#bit-ops}

### Bit Field Extraction

```asm
@ ดึงบิตตำแหน่ง start จำนวน width บิต
@ result = (value >> start) & ((1 << width) - 1)

@ ตัวอย่าง: ดึง bits[11:8] จาก r0
MOV  r1, r0, LSR #8    @ เลื่อนบิตที่ต้องการมาที่ LSB
AND  r1, r1, #0xF      @ mask 4 บิต

@ ARM Thumb2/ARMv7 มี UBFX instruction:
UBFX r1, r0, #8, #4   @ r1 = r0[11:8] (unsigned)
SBFX r1, r0, #8, #4   @ r1 = r0[11:8] (signed, sign-extended)
```

### Bit Field Insertion

```asm
@ แทนที่บิต bits[11:8] ด้วยค่าใหม่
@ BFI instruction (ARMv7+):
BFI  r0, r1, #8, #4   @ r0[11:8] = r1[3:0]

@ วิธี manual:
@ 1. ล้างบิตที่ต้องการใน destination
@ 2. mask และ shift source
@ 3. OR เข้าด้วยกัน
BIC  r0, r0, #(0xF << 8)   @ ล้าง bits[11:8]
AND  r1, r1, #0xF           @ mask source
ORR  r0, r0, r1, LSL #8    @ ใส่ค่าใหม่
```

### ไฟล์: `bit_ops.s`

```asm
@ bit_ops.s - การ extract และ insert บิตด้วย barrel shifter

.section .data
fmt_extract: .asciz "Extract bits[%d:%d] from 0x%08X = 0x%X\n"
fmt_insert:  .asciz "Insert 0x%X at bits[%d:%d] of 0x%08X = 0x%08X\n"
fmt_count:   .asciz "Bit count of 0x%08X = %d\n"

.section .text
.global main
.extern printf

@ ============================================
@ extract_bits: ดึง bit field
@ r0 = value
@ r1 = start bit
@ r2 = width (จำนวนบิต)
@ return r0 = extracted value
@ ============================================
extract_bits:
    PUSH    {r4, r5, lr}
    MOV     r4, r0              @ save value
    MOV     r5, r2              @ save width

    @ shift right to align
    MOV     r0, r4, LSR r1     @ r0 = value >> start

    @ create mask = (1 << width) - 1
    MOV     r3, #1
    MOV     r3, r3, LSL r5     @ r3 = 1 << width
    SUB     r3, r3, #1         @ r3 = mask

    @ apply mask
    AND     r0, r0, r3

    POP     {r4, r5, pc}

@ ============================================
@ insert_bits: แทรก bit field
@ r0 = destination
@ r1 = value to insert
@ r2 = start bit
@ r3 = width
@ return r0 = modified destination
@ ============================================
insert_bits:
    PUSH    {r4-r7, lr}
    MOV     r4, r0              @ dest
    MOV     r5, r1              @ value
    MOV     r6, r2              @ start
    MOV     r7, r3              @ width

    @ create mask
    MOV     r0, #1
    MOV     r0, r0, LSL r7     @ 1 << width
    SUB     r0, r0, #1         @ mask = (1<<width)-1

    @ clear bits in destination
    MOV     r1, r0, LSL r6     @ shifted mask
    BIC     r4, r4, r1         @ clear field

    @ mask and shift source value
    AND     r5, r5, r0         @ mask source
    ORR     r4, r4, r5, LSL r6 @ insert

    MOV     r0, r4
    POP     {r4-r7, pc}

@ ============================================
@ count_bits: นับจำนวนบิตที่เป็น 1 (Hamming weight)
@ r0 = value
@ return r0 = bit count
@ ============================================
count_bits:
    MOV     r1, #0             @ count = 0
count_loop:
    CMP     r0, #0
    BEQ     count_done
    ANDS    r2, r0, #1         @ ดู LSB
    ADDNE   r1, r1, #1         @ ถ้า LSB=1, count++
    MOV     r0, r0, LSR #1    @ shift right
    B       count_loop
count_done:
    MOV     r0, r1
    BX      lr

main:
    PUSH    {r4-r9, lr}

    @ ดึง bits[7:4] จาก 0xABCD
    LDR     r4, =0x0000ABCD
    MOV     r0, r4
    MOV     r1, #4             @ start = 4
    MOV     r2, #4             @ width = 4
    BL      extract_bits
    MOV     r5, r0
    LDR     r0, =fmt_extract
    MOV     r1, #7             @ high bit
    MOV     r2, #4             @ low bit
    MOV     r3, r4
    PUSH    {r5}
    BL      printf
    ADD     sp, sp, #4

    @ ใส่ค่า 0xF ที่ bits[11:8] ของ 0x0000ABCD
    LDR     r4, =0x0000ABCD
    MOV     r0, r4
    MOV     r1, #0xF
    MOV     r2, #8
    MOV     r3, #4
    BL      insert_bits
    MOV     r5, r0
    LDR     r0, =fmt_insert
    MOV     r1, #0xF
    MOV     r2, #11
    MOV     r3, #8
    PUSH    {r4, r5}
    BL      printf
    ADD     sp, sp, #8

    @ นับบิตใน 0xFF00FF00
    LDR     r4, =0xFF00FF00
    MOV     r0, r4
    BL      count_bits
    MOV     r5, r0
    LDR     r0, =fmt_count
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r9, pc}
```

---

## 12. Endian Conversion with Shifts {#endian}

### Little-Endian vs Big-Endian

```
Little-Endian (ARM default):
  0x12345678 ในหน่วยความจำ:
  Address+0: 0x78 (LSB ก่อน)
  Address+1: 0x56
  Address+2: 0x34
  Address+3: 0x12 (MSB หลัง)

Big-Endian (Network byte order):
  Address+0: 0x12 (MSB ก่อน)
  Address+1: 0x34
  Address+2: 0x56
  Address+3: 0x78 (LSB หลัง)

Byte Swap (32-bit):
  0x12345678 → 0x78563412
```

### ไฟล์: `endian_convert.s`

```asm
@ endian_convert.s - การแปลง endianness โดยใช้ shifts

.section .data
fmt_swap:   .asciz "Byte swap 32: 0x%08X -> 0x%08X\n"
fmt_swap16: .asciz "Byte swap 16: 0x%04X -> 0x%04X\n"
fmt_htons:  .asciz "htons(0x%04X) = 0x%04X\n"
fmt_htonl:  .asciz "htonl(0x%08X) = 0x%08X\n"

.section .text
.global main
.extern printf

@ ============================================
@ byte_swap32: สลับ byte order ของ 32-bit value
@ r0 = input
@ return r0 = byte-swapped output
@ ============================================
byte_swap32:
    @ วิธีที่ 1: ใช้ shift และ OR
    @ input:  0xAABBCCDD
    @ output: 0xDDCCBBAA

    @ ดึงแต่ละ byte
    AND     r1, r0, #0xFF          @ r1 = byte0 (0xDD)
    AND     r2, r0, #0xFF00        @ r2 = byte1 shifted (0xCC00)
    AND     r3, r0, #0xFF0000      @ r3 = byte2 shifted (0xBB0000)

    @ สร้าง output
    MOV     r1, r1, LSL #24        @ byte0 → bits[31:24]
    MOV     r2, r2, LSL #8         @ byte1 → bits[23:16]
    MOV     r3, r3, LSR #8         @ byte2 → bits[15:8]
    MOV     r4, r0, LSR #24        @ byte3 → bits[7:0]

    @ รวมกัน
    ORR     r0, r1, r2
    ORR     r0, r0, r3
    ORR     r0, r0, r4
    BX      lr

@ ============================================
@ byte_swap32_rev: ใช้ REV instruction (ARMv6+)
@ เร็วกว่าวิธีด้านบน!
@ r0 = input
@ return r0 = byte-swapped output
@ ============================================
byte_swap32_rev:
    REV     r0, r0              @ hardware byte swap!
    BX      lr

@ ============================================
@ byte_swap16: สลับ byte order ของ 16-bit value
@ r0 = input (ใช้ bits[15:0])
@ return r0 = byte-swapped output
@ ============================================
byte_swap16:
    @ 0xAABB → 0xBBAA
    AND     r1, r0, #0xFF       @ r1 = low byte
    MOV     r2, r0, LSR #8      @ r2 = high byte
    AND     r2, r2, #0xFF
    ORR     r0, r2, r1, LSL #8  @ r0 = (low<<8) | high
    BX      lr

@ ============================================
@ htons: host to network short (เหมือน byte_swap16)
@ htonl: host to network long (เหมือน byte_swap32)
@ ============================================

main:
    PUSH    {r4-r7, lr}

    @ byte_swap32 ด้วย manual shifts
    LDR     r4, =0x12345678
    MOV     r0, r4
    BL      byte_swap32
    MOV     r5, r0
    LDR     r0, =fmt_swap
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    @ byte_swap32 ด้วย REV (เร็วกว่า)
    LDR     r4, =0xAABBCCDD
    MOV     r0, r4
    BL      byte_swap32_rev
    MOV     r5, r0
    LDR     r0, =fmt_swap
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    @ byte_swap16
    MOV     r4, #0x1234
    MOV     r0, r4
    BL      byte_swap16
    MOV     r5, r0
    LDR     r0, =fmt_swap16
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    @ Network byte order (Big-Endian)
    MOV     r4, #0x0050        @ port 80 in host byte order
    MOV     r0, r4
    BL      byte_swap16
    MOV     r5, r0
    LDR     r0, =fmt_htons
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    LDR     r4, =0xC0A80001    @ 192.168.0.1 in host byte order
    MOV     r0, r4
    BL      byte_swap32_rev
    MOV     r5, r0
    LDR     r0, =fmt_htonl
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 13. AArch64 Shift Operations {#aarch64}

### ความแตกต่างจาก ARM32

ใน AArch64 (64-bit ARM):
- Shift ยังคงใช้ได้กับ data processing instructions
- Register ขนาด 64-bit (X registers) หรือ 32-bit (W registers)
- Shift amount สำหรับ 64-bit: 0-63
- Shift amount สำหรับ 32-bit: 0-31

```asm
// AArch64 Shift Syntax
LSL   X0, X1, #n       // Logical shift left by immediate
LSR   X0, X1, #n       // Logical shift right
ASR   X0, X1, #n       // Arithmetic shift right
ROR   X0, X1, #n       // Rotate right

// Shift by register
LSL   X0, X1, X2       // Shift left by register value
LSR   X0, X1, X2
ASR   X0, X1, X2
ROR   X0, X1, X2

// Combined with data processing
ADD   X0, X1, X2, LSL #3    // X0 = X1 + (X2 << 3)
SUB   X0, X1, X2, ASR #2    // X0 = X1 - (X2 >> 2)

// 32-bit operations (W registers, same syntax)
ADD   W0, W1, W2, LSL #3
```

### ไฟล์: `aarch64_shifts.s`

```asm
// aarch64_shifts.s - Shift operations ใน AArch64
// Compile: aarch64-linux-gnu-as aarch64_shifts.s -o aarch64_shifts.o
// Link:    aarch64-linux-gnu-ld aarch64_shifts.o -o aarch64_shifts
// Run:     qemu-aarch64 ./aarch64_shifts

.section .data
fmt_64:     .asciz "64-bit: 0x%016llX << %d = 0x%016llX\n"
fmt_32:     .asciz "32-bit: %u << %d = %u\n"
fmt_array:  .asciz "Array[%lld] access: base=%lld, offset=%lld, addr=%lld\n"
fmt_title:  .asciz "=== AArch64 Barrel Shifter Demo ===\n"

.section .text
.global main
.extern printf

main:
    STP     X29, X30, [SP, #-16]!    // Save frame pointer and link register
    MOV     X29, SP

    // แสดง title
    ADRP    X0, fmt_title
    ADD     X0, X0, :lo12:fmt_title
    BL      printf

    // ============================================
    // 64-bit LSL
    // ============================================
    MOV     X19, #1
    LSL     X20, X19, #32           // 1 << 32 = 4294967296
    ADRP    X0, fmt_64
    ADD     X0, X0, :lo12:fmt_64
    MOV     X1, X19
    MOV     X2, #32
    MOV     X3, X20
    BL      printf

    // ============================================
    // 64-bit ASR
    // ============================================
    MOV     X19, #-8                 // negative number
    ASR     X20, X19, #2            // -8 >> 2 = -2 (arithmetic)
    ADRP    X0, fmt_64
    ADD     X0, X0, :lo12:fmt_64
    MOV     X1, X19
    MOV     X2, #2
    MOV     X3, X20
    BL      printf

    // ============================================
    // Array indexing (64-bit)
    // int64_t array[]; addr = base + index * 8
    // 8 = 2^3, ดังนั้น LSL #3
    // ============================================
    MOV     X19, #0x10000           // base address (example)
    MOV     X20, #5                  // index
    ADD     X21, X19, X20, LSL #3  // addr = base + index * 8
    ADRP    X0, fmt_array
    ADD     X0, X0, :lo12:fmt_array
    MOV     X1, X20                  // index
    MOV     X2, X19                  // base
    MOV     X3, X20, LSL #3         // offset
    MOV     X4, X21                  // final addr
    BL      printf

    // ============================================
    // 32-bit operations with W registers
    // ============================================
    MOV     W19, #7
    LSL     W20, W19, #3            // 7 << 3 = 56
    ADRP    X0, fmt_32
    ADD     X0, X0, :lo12:fmt_32
    MOV     W1, W19
    MOV     W2, #3
    MOV     W3, W20
    BL      printf

    // ============================================
    // UBFX/SBFX - Unsigned/Signed Bit Field Extract
    // ============================================
    // ดึง bits[15:8] (8 บิต) จาก X19
    LDR     X19, =0xABCDEF123456789A
    UBFX    X20, X19, #8, #8        // X20 = X19[15:8]
    ADRP    X0, fmt_64
    ADD     X0, X0, :lo12:fmt_64
    MOV     X1, X19
    MOV     X2, #8
    MOV     X3, X20
    BL      printf

    LDP     X29, X30, [SP], #16
    MOV     X0, #0
    RET
```

### การทดสอบ AArch64

```bash
# คอมไพล์ด้วย GCC (ง่ายกว่า)
aarch64-linux-gnu-gcc -static aarch64_shifts.s -o aarch64_shifts

# รันด้วย QEMU
qemu-aarch64 ./aarch64_shifts

# หรือ cross-compile และรันบน ARM hardware
```

---

## 14. Performance Impact {#performance}

### การเปรียบเทียบ: ARM vs x86

```
ARM (ด้วย Barrel Shifter):
  ADD  r0, r1, r2, LSL #2    @ 1 instruction, 1 cycle
  
x86 (ไม่มี Barrel Shifter รวมใน ALU):
  MOV  eax, [r2]             @ ต้องแยก instruction
  SHL  eax, 2
  ADD  [r0], [r1]            @ 3 instructions, 3+ cycles
```

### Case Study: Dot Product Calculation

```asm
@ dot_product.s - สาธิต performance ของ barrel shifter

.section .data
array_a:    .word 1, 2, 3, 4, 5, 6, 7, 8
array_b:    .word 10, 20, 30, 40, 50, 60, 70, 80
n_elements: .word 8
fmt_result: .asciz "Dot product = %d\n"
fmt_time:   .asciz "Computed in efficient ARM style!\n"

.section .text
.global main
.extern printf

@ ============================================
@ dot_product_naive: วิธีธรรมดา
@ ============================================
dot_product_naive:
    PUSH    {r4-r8, lr}
    MOV     r4, #0              @ sum = 0
    MOV     r5, #0              @ i = 0
    LDR     r6, =array_a
    LDR     r7, =array_b
    LDR     r8, =n_elements
    LDR     r8, [r8]
naive_loop:
    CMP     r5, r8
    BGE     naive_done
    LDR     r0, [r6, r5, LSL #2]  @ a[i] ← ใช้ barrel shifter สำหรับ address!
    LDR     r1, [r7, r5, LSL #2]  @ b[i]
    MUL     r2, r0, r1
    ADD     r4, r4, r2
    ADD     r5, r5, #1
    B       naive_loop
naive_done:
    MOV     r0, r4
    POP     {r4-r8, pc}

main:
    PUSH    {r4-r7, lr}

    BL      dot_product_naive
    MOV     r4, r0

    LDR     r0, =fmt_result
    MOV     r1, r4
    BL      printf

    LDR     r0, =fmt_time
    BL      printf

    @ คำนวณ: 1*10 + 2*20 + 3*30 + 4*40 + 5*50 + 6*60 + 7*70 + 8*80
    @ = 10 + 40 + 90 + 160 + 250 + 360 + 490 + 640 = 2040

    MOV     r0, #0
    POP     {r4-r7, pc}
```

### ตาราง Performance

| Operation | ARM (cycles) | x86 (cycles) | ARM Advantage |
|-----------|-------------|--------------|---------------|
| shift alone | 0 (free!) | 1 | ดีกว่า |
| ADD + shift | 1 | 2 | 2x เร็วกว่า |
| MUL by 2^n | 1 (shift) | 3 (imul) | 3x เร็วกว่า |
| Array index | 1 (combined) | 2-3 | 2-3x เร็วกว่า |
| Byte swap | 1 (REV) | 3 (bswap) | 3x เร็วกว่า |

### Code Generation Example

```c
// C code:
int scale_add(int base, int offset) {
    return base + offset * 4;
}

// ARM Compiler output (OPTIMIZED):
// ADD r0, r0, r1, LSL #2    ← 1 instruction!

// x86 Compiler output:
// lea eax, [r0 + r1*4]      ← 1 instruction (x86 has LEA)
// (คล้ายกัน แต่ ARM มี barrel shifter กับทุก ALU op, x86 แค่ LEA)
```

---

## 15. โปรแกรมประยุกต์: Image Processing {#image-processing}

### การ Process Pixel Data ด้วย Barrel Shifter

```asm
@ image_process.s - ใช้ barrel shifter สำหรับ image processing
@ แสดงการ pack/unpack ARGB pixel และการ brightness adjustment

.section .data
@ pixel format: 0xAARRGGBB
pixel1:     .word 0xFF804020    @ semi-transparent orange
pixel2:     .word 0xFF1040A0    @ dark blue
fmt_pixel:  .asciz "Pixel 0x%08X: A=%d R=%d G=%d B=%d\n"
fmt_bright: .asciz "Brightness +50%%: 0x%08X -> 0x%08X\n"
fmt_blend:  .asciz "50%% blend of 0x%08X and 0x%08X = 0x%08X\n"

.section .text
.global main
.extern printf

@ ============================================
@ unpack_pixel: แยก ARGB components
@ r0 = pixel (0xAARRGGBB)
@ output: r0=A, r1=R, r2=G, r3=B
@ ============================================
unpack_pixel:
    PUSH    {r4, lr}
    MOV     r4, r0

    MOV     r3, r4, AND #0xFF       @ B = bits[7:0]  ← ผิดต้องแก้
    @ ที่ถูก:
    AND     r3, r4, #0xFF           @ B = bits[7:0]
    MOV     r2, r4, LSR #8          @
    AND     r2, r2, #0xFF           @ G = bits[15:8]
    MOV     r1, r4, LSR #16         @
    AND     r1, r1, #0xFF           @ R = bits[23:16]
    MOV     r0, r4, LSR #24         @ A = bits[31:24]

    POP     {r4, pc}

@ ============================================
@ pack_pixel: รวม ARGB components
@ r0=A, r1=R, r2=G, r3=B
@ return r0 = packed pixel
@ ============================================
pack_pixel:
    AND     r0, r0, #0xFF
    AND     r1, r1, #0xFF
    AND     r2, r2, #0xFF
    AND     r3, r3, #0xFF

    ORR     r0, r3, r2, LSL #8     @ G|B
    ORR     r0, r0, r1, LSL #16    @ R|G|B
    ORR     r0, r0, r0, LSL #24    @ ผิด! ต้องเป็น A
    @ แก้: ต้องเก็บ A ไว้ก่อน
    BX      lr

@ ============================================
@ adjust_brightness: เพิ่ม/ลด ความสว่าง
@ r0 = pixel, r1 = adjustment (-255 to +255)
@ return r0 = adjusted pixel
@ ============================================
adjust_brightness:
    PUSH    {r4-r9, lr}
    MOV     r4, r0              @ original pixel
    MOV     r5, r1              @ adjustment

    @ แยก RGB (ไม่รวม Alpha)
    AND     r6, r4, #0xFF       @ B
    MOV     r7, r4, LSR #8
    AND     r7, r7, #0xFF       @ G
    MOV     r8, r4, LSR #16
    AND     r8, r8, #0xFF       @ R
    MOV     r9, r4, LSR #24     @ A (ไม่เปลี่ยน)

    @ เพิ่มค่า และ clamp ให้อยู่ใน 0-255
    ADD     r6, r6, r5          @ B + adj
    ADD     r7, r7, r5          @ G + adj
    ADD     r8, r8, r5          @ R + adj

    @ Clamp: ถ้า < 0 → 0, ถ้า > 255 → 255
    MOV     r0, #0
    MOV     r1, #255

    @ Clamp B
    CMP     r6, #0
    MOVLT   r6, #0
    CMP     r6, #255
    MOVGT   r6, #255

    @ Clamp G
    CMP     r7, #0
    MOVLT   r7, #0
    CMP     r7, #255
    MOVGT   r7, #255

    @ Clamp R
    CMP     r8, #0
    MOVLT   r8, #0
    CMP     r8, #255
    MOVGT   r8, #255

    @ Pack กลับ
    ORR     r0, r6, r7, LSL #8
    ORR     r0, r0, r8, LSL #16
    ORR     r0, r0, r9, LSL #24

    POP     {r4-r9, pc}

main:
    PUSH    {r4-r7, lr}

    @ แสดง pixel components
    LDR     r4, =0xFF804020        @ test pixel
    PUSH    {r4}                    @ เก็บค่าไว้
    @ (ละเว้นการเรียก unpack_pixel เพราะ clobbers r0-r3)

    @ Brightness adjustment
    LDR     r4, =0xFF804020
    MOV     r0, r4
    MOV     r1, #50                 @ +50 brightness
    BL      adjust_brightness
    MOV     r5, r0
    LDR     r0, =fmt_bright
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    LDR     r4, =0xFF804020
    MOV     r0, r4
    MOV     r1, #-50                @ -50 brightness (darker)
    BL      adjust_brightness
    MOV     r5, r0
    LDR     r0, =fmt_bright
    MOV     r1, r4
    MOV     r2, r5
    BL      printf

    ADD     sp, sp, #4              @ ล้าง stack จาก PUSH ด้านบน
    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## 16. การทดสอบและ Debugging {#testing}

### ไฟล์: `test_barrel_shifter.s`

```asm
@ test_barrel_shifter.s - Unit tests สำหรับ barrel shifter operations
@ ทดสอบทุก shift type กับค่าต่างๆ

.section .data
pass_msg:   .asciz "PASS: %s\n"
fail_msg:   .asciz "FAIL: %s (expected %d, got %d)\n"
total_msg:  .asciz "\n=== Results: %d/%d tests passed ===\n"

test_lsl:   .asciz "LSL #2 of 5 = 20"
test_lsr:   .asciz "LSR #2 of 20 = 5"
test_asr:   .asciz "ASR #1 of -8 = -4"
test_ror:   .asciz "ROR #8 of 0x12345678 = 0x78123456"
test_mul3:  .asciz "multiply_by_3(7) = 21"
test_mul5:  .asciz "multiply_by_5(7) = 35"
test_mul7:  .asciz "multiply_by_7(7) = 49"

.section .bss
pass_count: .word 0
fail_count: .word 0

.section .text
.global main
.extern printf

@ ============================================
@ check_equal: ตรวจสอบผลลัพธ์
@ r0 = actual, r1 = expected, r2 = test name ptr
@ ============================================
check_equal:
    PUSH    {r4-r6, lr}
    MOV     r4, r0              @ actual
    MOV     r5, r1              @ expected
    MOV     r6, r2              @ name

    CMP     r4, r5
    BNE     check_fail

check_pass:
    LDR     r0, =pass_msg
    MOV     r1, r6
    BL      printf
    @ pass_count++
    LDR     r0, =pass_count
    LDR     r1, [r0]
    ADD     r1, r1, #1
    STR     r1, [r0]
    B       check_done

check_fail:
    LDR     r0, =fail_msg
    MOV     r1, r6
    MOV     r2, r5              @ expected
    MOV     r3, r4              @ actual
    BL      printf
    @ fail_count++
    LDR     r0, =fail_count
    LDR     r1, [r0]
    ADD     r1, r1, #1
    STR     r1, [r0]

check_done:
    POP     {r4-r6, pc}

main:
    PUSH    {r4-r7, lr}

    @ Test 1: LSL
    MOV     r4, #5
    MOV     r0, r4, LSL #2     @ 5 << 2 = 20
    MOV     r1, #20
    LDR     r2, =test_lsl
    BL      check_equal

    @ Test 2: LSR
    MOV     r4, #20
    MOV     r0, r4, LSR #2    @ 20 >> 2 = 5
    MOV     r1, #5
    LDR     r2, =test_lsr
    BL      check_equal

    @ Test 3: ASR
    MVN     r4, #7             @ r4 = -8
    MOV     r0, r4, ASR #1    @ -8 >> 1 = -4
    MVN     r1, #3             @ expected = -4
    LDR     r2, =test_asr
    BL      check_equal

    @ Test 4: ROR
    LDR     r4, =0x12345678
    MOV     r0, r4, ROR #8
    LDR     r1, =0x78123456    @ expected
    LDR     r2, =test_ror
    BL      check_equal

    @ Test 5: multiply_by_3
    MOV     r4, #7
    ADD     r0, r4, r4, LSL #1 @ 7*3 = 21
    MOV     r1, #21
    LDR     r2, =test_mul3
    BL      check_equal

    @ Test 6: multiply_by_5
    MOV     r4, #7
    ADD     r0, r4, r4, LSL #2 @ 7*5 = 35
    MOV     r1, #35
    LDR     r2, =test_mul5
    BL      check_equal

    @ Test 7: multiply_by_7
    MOV     r4, #7
    RSB     r0, r4, r4, LSL #3 @ 7*7 = 49
    MOV     r1, #49
    LDR     r2, =test_mul7
    BL      check_equal

    @ แสดงผลสรุป
    LDR     r4, =pass_count
    LDR     r4, [r4]
    LDR     r5, =fail_count
    LDR     r5, [r5]
    ADD     r6, r4, r5         @ total tests

    LDR     r0, =total_msg
    MOV     r1, r4             @ passed
    MOV     r2, r6             @ total
    BL      printf

    MOV     r0, #0
    POP     {r4-r7, pc}
```

---

## การ Compile และ Run ทั้งหมด {#compilation}

### Script: `build_and_test.sh`

```bash
#!/bin/bash
# build_and_test.sh - สคริปต์คอมไพล์และทดสอบทุกโปรแกรมใน Part 036

set -e  # หยุดถ้าเกิด error

ARM32_GCC="arm-linux-gnueabi-gcc"
AARCH64_GCC="aarch64-linux-gnu-gcc"
QEMU_ARM="qemu-arm"
QEMU_AARCH64="qemu-aarch64"

echo "=========================================="
echo "Building Part 036: ARM Barrel Shifter"
echo "=========================================="

# ตรวจสอบว่ามี cross-compiler
check_tool() {
    if ! command -v $1 &> /dev/null; then
        echo "WARNING: $1 not found, skipping..."
        return 1
    fi
    return 0
}

# ARM32 programs
ARM32_PROGS=(
    "lsl_demo"
    "lsr_demo"
    "asr_demo"
    "ror_demo"
    "rrx_demo"
    "shift_modes"
    "combined_ops"
    "fast_multiply"
    "signed_division"
    "bit_ops"
    "endian_convert"
    "dot_product"
    "image_process"
    "test_barrel_shifter"
)

if check_tool $ARM32_GCC; then
    echo ""
    echo "--- Building ARM32 programs ---"
    for prog in "${ARM32_PROGS[@]}"; do
        if [ -f "${prog}.s" ]; then
            echo -n "  Building ${prog}... "
            $ARM32_GCC -static "${prog}.s" -o "${prog}" 2>/dev/null && \
                echo "OK" || echo "FAILED"
        fi
    done

    echo ""
    echo "--- Running ARM32 tests ---"
    if check_tool $QEMU_ARM; then
        for prog in "${ARM32_PROGS[@]}"; do
            if [ -f "${prog}" ]; then
                echo ""
                echo ">>> Running ${prog}:"
                $QEMU_ARM "./${prog}" || true
            fi
        done
    fi
fi

# AArch64 programs
if check_tool $AARCH64_GCC; then
    echo ""
    echo "--- Building AArch64 programs ---"
    if [ -f "aarch64_shifts.s" ]; then
        echo -n "  Building aarch64_shifts... "
        $AARCH64_GCC -static "aarch64_shifts.s" -o "aarch64_shifts" 2>/dev/null && \
            echo "OK" || echo "FAILED"
    fi

    if check_tool $QEMU_AARCH64 && [ -f "aarch64_shifts" ]; then
        echo ""
        echo ">>> Running aarch64_shifts:"
        $QEMU_AARCH64 "./aarch64_shifts" || true
    fi
fi

echo ""
echo "=========================================="
echo "Build complete!"
echo "=========================================="
```

### การติดตั้ง Tools ที่ต้องการ

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    gcc-arm-linux-gnueabi \
    gcc-aarch64-linux-gnu \
    qemu-user \
    qemu-user-static \
    binutils-arm-linux-gnueabi \
    binutils-aarch64-linux-gnu

# Fedora/RHEL
sudo dnf install -y \
    gcc-arm-linux-gnu \
    gcc-aarch64-linux-gnu \
    qemu-user

# macOS (Homebrew)
brew install arm-linux-gnueabi \
    aarch64-linux-gnu-gcc \
    qemu
```

---

## แบบฝึกหัด (Exercises) {#exercises}

### ระดับพื้นฐาน (Basic)

**แบบฝึกหัด 1: Shift Calculator**

เขียนโปรแกรมที่รับ:
- ค่าตัวเลข 32-bit
- ประเภทของ shift (LSL/LSR/ASR/ROR)
- จำนวนตำแหน่ง

แล้วแสดงผลลัพธ์พร้อม carry flag

```asm
@ template สำหรับแบบฝึกหัด 1
.section .text
.global main

main:
    @ TODO: implement shift calculator
    @ hint: ใช้ MOVS เพื่อ update flags
    @ hint: ใช้ MRS r0, CPSR เพื่ออ่าน flags
    MOV r0, #0
    BX  lr
```

**แบบฝึกหัด 2: Power of 2 Checker**

เขียน function `is_power_of_2(n)` ที่คืนค่า 1 ถ้า n เป็น power of 2
โดยใช้ shift operations (hint: n & (n-1) == 0)

```asm
@ template สำหรับแบบฝึกหัด 2
is_power_of_2:
    @ Input:  r0 = n
    @ Output: r0 = 1 ถ้าเป็น power of 2, 0 ถ้าไม่ใช่
    @ TODO: implement using bitwise operations
    BX  lr
```

### ระดับกลาง (Intermediate)

**แบบฝึกหัด 3: Fast Integer Square Root**

เขียน `isqrt(n)` โดยใช้ bit-by-bit method ที่ใช้ shifts

```asm
@ isqrt.s - Integer square root ด้วย binary search + shifts
@ Algorithm: Newton-Raphson หรือ bit-by-bit
@ hint: ใช้ CLZ (Count Leading Zeros) ใน ARMv5+

isqrt:
    @ Input:  r0 = n (non-negative)
    @ Output: r0 = floor(sqrt(n))
    @ TODO: implement
    BX  lr
```

**แบบฝึกหัด 4: CRC-32 Calculation**

เขียน CRC-32 checksum calculator โดยใช้ shift operations
(CRC ต้องการ shift และ XOR ซ้ำๆ)

```asm
@ crc32.s - CRC-32 checksum calculation
@ Polynomial: 0xEDB88320 (reversed)

crc32_byte:
    @ r0 = current CRC
    @ r1 = byte to process
    @ return r0 = new CRC
    PUSH    {r4-r5, lr}
    EOR     r0, r0, r1          @ XOR with input byte
    MOV     r4, #8              @ process 8 bits
crc_loop:
    MOVS    r0, r0, LSR #1     @ shift right, LSB → Carry
    LDRCS   r2, =0xEDB88320    @ if Carry (LSB was 1)
    EORCS   r0, r0, r2         @ XOR with polynomial
    SUBS    r4, r4, #1
    BNE     crc_loop
    POP     {r4-r5, pc}
```

### ระดับสูง (Advanced)

**แบบฝึกหัด 5: SIMD-style Parallel Operations**

ใน ARM32 ไม่มี SIMD แต่เราสามารถใช้ barrel shifter เพื่อทำ
"parallel" operations บน bytes ใน word เดียว:

```asm
@ parallel_add.s - เพิ่มค่าใน 4 bytes พร้อมกัน (SWAR technique)
@ SWAR = SIMD Within A Register

parallel_add_bytes:
    @ r0 = 4 bytes packed: [B3][B2][B1][B0]
    @ r1 = value to add to each byte (0-127)
    @ return r0 = [B3+r1][B2+r1][B1+r1][B0+r1] (with saturation)
    
    @ TODO: ใช้เทคนิค bit manipulation เพื่อป้องกัน carry overflow
    @ hint: แยก odd/even bytes แล้วบวก จากนั้นรวมกลับ
    BX  lr
```

**แบบฝึกหัด 6: Fast Base-2 Logarithm**

```asm
@ log2_floor.s - หา floor(log2(n))
@ = ตำแหน่งของ Most Significant Bit (MSB)

log2_floor:
    @ Input:  r0 = n (n > 0)
    @ Output: r0 = floor(log2(n))
    @
    @ วิธีที่ 1: ใช้ CLZ (Count Leading Zeros) ใน ARMv5T+
    @   log2(n) = 31 - CLZ(n)
    @
    @ วิธีที่ 2: Loop shift ทีละ 1 (สำหรับ ARMv4)
    @   count = 0
    @   while (n > 1) { n >>= 1; count++ }
    @
    @ TODO: implement ทั้ง 2 วิธีและเปรียบเทียบ performance
    BX  lr
```

---

## สรุป (Summary)

### จุดสำคัญที่ต้องจำ

```
┌──────────────────────────────────────────────────────────────┐
│              ARM Barrel Shifter Summary                      │
│                                                              │
│  Shift Types:                                                │
│  ┌─────┬──────────────────────┬──────────────────────────┐  │
│  │ LSL │ Logical Shift Left   │ Multiply by 2^n          │  │
│  │ LSR │ Logical Shift Right  │ Divide by 2^n (unsigned) │  │
│  │ ASR │ Arithmetic SR        │ Divide by 2^n (signed)   │  │
│  │ ROR │ Rotate Right         │ Bit manipulation         │  │
│  │ RRX │ Rotate via Carry     │ 64-bit shift             │  │
│  └─────┴──────────────────────┴──────────────────────────┘  │
│                                                              │
│  Key Patterns:                                               │
│  • Array index: ADD r0, base, idx, LSL #log2(sizeof)        │
│  • Multiply 3:  ADD r0, r0, r0, LSL #1                      │
│  • Multiply 5:  ADD r0, r0, r0, LSL #2                      │
│  • Multiply 7:  RSB r0, r0, r0, LSL #3                      │
│  • Multiply 9:  ADD r0, r0, r0, LSL #3                      │
│  • Pack bytes:  ORR r0, r0, byte, LSL #(8*n)                │
│  • Extract:     AND r0, (val LSR #pos), mask                 │
│                                                              │
│  AArch64: Same syntax, bigger ranges (0-63 for 64-bit)      │
│                                                              │
│  Performance: Barrel shifter adds 0 cycles to ALU ops!      │
└──────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบ Shift Types

```
Value: 0b10110100 (180)

LSL #2: 0b11010000 (208)   ← เลื่อนซ้าย, เติม 0 ทางขวา
         ← ← ← ←

LSR #2: 0b00101101 (45)    ← เลื่อนขวา, เติม 0 ทางซ้าย
         → → → →

ASR #2: 0b11101101 (237/-19) ← เลื่อนขวา, เติม sign bit
         → → → →

ROR #2: 0b00101101 (45)   ← หมุนขวา
         bit วนรอบ

ถ้า sign bit=1 (negative):
Value: 0b10110100 (-76)

ASR #2: 0b11101101 (-19)  ← เติม 1 (รักษา sign)
LSR #2: 0b00101101 (45)   ← เติม 0 (ทำให้กลายเป็น positive!)
```

### เมื่อไหรควรใช้อะไร

| สถานการณ์ | Shift ที่ใช้ | เหตุผล |
|-----------|------------|--------|
| คูณ/หารด้วย 2^n (unsigned) | LSL/LSR | เร็วกว่า MUL/DIV |
| หารด้วย 2^n (signed) | ASR + bias | ระวัง round direction |
| สร้าง bitmask | LSL | สร้าง 1 << n |
| ดึงบิต | LSR + AND | เลื่อนมาที่ LSB แล้ว mask |
| Array indexing | LSL ใน addressing | address = base + idx << log2(size) |
| Endian conversion | REV หรือ shift+OR | ใช้ REV ถ้า ARMv6+ |
| Bit rotation | ROR | symmetric rotation |
| 64-bit shift | LSL/LSR ร่วมกับ RRX | chain ผ่าน carry |

---

## การทดสอบบน Hardware จริง

### Raspberry Pi (ARM32/AArch64)

```bash
# บน Raspberry Pi 4 (AArch64)
# คอมไพล์โดยตรง (ไม่ต้อง cross-compile)
gcc -march=armv8-a -o aarch64_shifts aarch64_shifts.s
./aarch64_shifts

# ดู assembly output
gcc -S -march=armv8-a -O2 test.c -o test.s
objdump -d aarch64_shifts | grep -A5 "main"

# Profile การทำงาน
perf stat ./aarch64_shifts
```

### BeagleBone Black (ARM Cortex-A8)

```bash
# ARM32 บน BeagleBone
gcc -march=armv7-a -mfpu=neon -o lsl_demo lsl_demo.s
./lsl_demo

# ดู cache performance
valgrind --tool=cachegrind ./dot_product
cg_annotate cachegrind.out.*
```

---

## เนื้อหาในตอนต่อไป (Next Part)

**Part 037: ARM NEON SIMD Instructions**
- Introduction to NEON (Advanced SIMD)
- Vector operations on multiple data
- Float and integer NEON
- Practical optimizations

---

*จบ Part 036: ARM Barrel Shifter*

*Barrel Shifter คือหัวใจของ ARM ISA ที่ทำให้ ARM มีประสิทธิภาพสูงด้วย instruction จำนวนน้อย*
*การเข้าใจและใช้งาน Barrel Shifter อย่างถูกต้องเป็นกุญแจสำคัญในการเขียน ARM Assembly ที่มีประสิทธิภาพสูง*
