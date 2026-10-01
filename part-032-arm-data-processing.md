# Part 032: ARM Data Processing Instructions
## คำสั่งประมวลผลข้อมูลใน ARM Assembly

---

## บทนำ (Introduction)

ARM Data Processing Instructions คือกลุ่มคำสั่งที่ใช้ในการคำนวณและประมวลผลข้อมูลใน registers
ARM มีรูปแบบการเข้ารหัสคำสั่งที่มีประสิทธิภาพสูง โดยเฉพาะ:
- **Conditional Execution**: คำสั่งส่วนใหญ่สามารถทำงานแบบ conditional ได้
- **Barrel Shifter**: การ shift operand ทำได้ใน instruction เดียว
- **S Suffix**: อัพเดต CPSR flags ได้เลือกได้

---

## 1. MOV และ MVN - Move Instructions

### 1.1 MOV (Move)

```asm
@ ARM32 - MOV instruction examples
@ ไฟล์: mov_examples.s

.section .text
.global _start

_start:
    @ MOV ค่าคงที่เข้า register
    MOV r0, #10          @ r0 = 10
    MOV r1, #0xFF        @ r1 = 255
    MOV r2, #0b1010      @ r2 = 10 (binary)
    
    @ MOV register ไปยัง register
    MOV r3, r0           @ r3 = r0
    MOV r4, r1           @ r4 = r1
    
    @ MOV พร้อม shift
    MOV r5, r0, LSL #2   @ r5 = r0 << 2 (คูณ 4)
    MOV r6, r1, LSR #1   @ r6 = r1 >> 1 (หาร 2)
    MOV r7, r2, ASR #1   @ r7 = r2 >> 1 (arithmetic shift)
    
    @ MOVS - MOV แล้วอัพเดต flags
    MOVS r0, #0          @ r0 = 0, Z flag set
    MOVS r1, #-1         @ r1 = -1, N flag set
    
    @ Exit
    MOV r7, #1           @ syscall exit
    MOV r0, #0           @ exit code 0
    SWI 0
```

### 1.2 MVN (Move Not)

```asm
@ ARM32 - MVN instruction examples
@ MVN = bitwise NOT แล้ว move

.section .text
.global _start

_start:
    @ MVN ค่าคงที่
    MVN r0, #0           @ r0 = ~0 = 0xFFFFFFFF = -1
    MVN r1, #1           @ r1 = ~1 = 0xFFFFFFFE
    MVN r2, #0xFF        @ r2 = ~0xFF = 0xFFFFFF00
    
    @ MVN register
    MOV r3, #0x0F0F0F0F
    MVN r4, r3           @ r4 = ~r3 = 0xF0F0F0F0
    
    @ MVNS - MVN แล้วอัพเดต flags
    MVNS r0, #0          @ r0 = -1, N flag set
    
    @ ใช้ MVN โหลดค่า -1 อย่างมีประสิทธิภาพ
    MVN r0, #0           @ ดีกว่า MOV r0, #-1
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 1.3 AArch64 MOV

```asm
// AArch64 - MOV instructions
// ไฟล์: mov64_examples.s

.section .text
.global _start

_start:
    // MOV ค่าคงที่ (ใช้ MOVZ/MOVN/MOVK ภายใน)
    MOV x0, #100         // x0 = 100
    MOV x1, #-1          // x1 = 0xFFFFFFFFFFFFFFFF
    
    // MOV 32-bit (zero-extends)
    MOV w0, #0x1234      // w0 = 0x1234, x0 = 0x0000000000001234
    
    // MOV register
    MOV x2, x0           // x2 = x0
    MOV w3, w1           // w3 = w1 (32-bit)
    
    // MOVZ - move zero (ใส่ค่าใน 16-bit slot)
    MOVZ x4, #0x1234, LSL #0   // x4 = 0x0000000000001234
    MOVZ x4, #0x1234, LSL #16  // x4 = 0x0000000012340000
    MOVZ x4, #0x1234, LSL #32  // x4 = 0x0000123400000000
    MOVZ x4, #0x1234, LSL #48  // x4 = 0x1234000000000000
    
    // MOVN - move not (complement)
    MOVN x5, #0          // x5 = 0xFFFFFFFFFFFFFFFF
    
    // MOVK - keep other bits, insert 16-bit
    MOVZ x6, #0x1234, LSL #0   // x6 = 0x1234
    MOVK x6, #0x5678, LSL #16  // x6 = 0x56781234
    MOVK x6, #0x9ABC, LSL #32  // x6 = 0x9ABC56781234
    
    // Exit
    MOV x8, #93          // syscall exit
    MOV x0, #0
    SVC #0
```

---

## 2. ADD, ADDS, ADC - Addition Instructions

### 2.1 ADD (Add)

```asm
@ ARM32 - ADD instruction examples
@ ไฟล์: add_examples.s

.section .data
result: .word 0

.section .text
.global _start

_start:
    @ ADD ค่าคงที่
    MOV r0, #10
    ADD r0, r0, #5       @ r0 = r0 + 5 = 15
    
    @ ADD registers
    MOV r1, #20
    MOV r2, #30
    ADD r3, r1, r2       @ r3 = r1 + r2 = 50
    
    @ ADD พร้อม shift (barrel shifter)
    MOV r4, #2
    ADD r5, r3, r4, LSL #3  @ r5 = r3 + (r4 << 3) = 50 + 16 = 66
    
    @ ADDS - ADD แล้วอัพเดต flags
    MOV r6, #0xFFFFFFFF  @ ค่าสูงสุด unsigned 32-bit
    MOV r7, #1
    ADDS r8, r6, r7      @ r8 = overflow! C flag set
    
    @ ตรวจสอบ overflow
    BCS overflow_occurred @ Branch if Carry Set
    B no_overflow
    
overflow_occurred:
    MOV r0, #1           @ บอกว่า overflow
    B done
    
no_overflow:
    MOV r0, #0
    
done:
    MOV r7, #1
    SWI 0
```

### 2.2 ADC (Add with Carry)

```asm
@ ARM32 - ADC สำหรับ 64-bit addition
@ ไฟล์: adc_64bit.s

.section .text
.global _start

_start:
    @ เพิ่ม 2 ค่า 64-bit ด้วย r0:r1 (low:high) และ r2:r3
    @ ค่า A = 0x00000001_FFFFFFFF
    MOV r0, #0xFFFFFFFF  @ low word ของ A
    MOV r1, #0x00000001  @ high word ของ A
    
    @ ค่า B = 0x00000000_00000001
    MOV r2, #0x00000001  @ low word ของ B
    MOV r3, #0x00000000  @ high word ของ B
    
    @ บวก low words ก่อน (ADDS เพื่อ set Carry)
    ADDS r4, r0, r2      @ r4 = low result, set C if overflow
    
    @ บวก high words พร้อม carry
    ADC r5, r1, r3       @ r5 = r1 + r3 + C
    
    @ ผลลัพธ์: r5:r4 = 0x00000002_00000000
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

---

## 3. SUB, SUBS, SBC, RSB, RSC - Subtraction Instructions

### 3.1 SUB (Subtract)

```asm
@ ARM32 - SUB instruction examples
@ ไฟล์: sub_examples.s

.section .text
.global _start

_start:
    @ SUB พื้นฐาน
    MOV r0, #100
    SUB r0, r0, #30      @ r0 = 100 - 30 = 70
    
    @ SUB registers
    MOV r1, #50
    MOV r2, #20
    SUB r3, r1, r2       @ r3 = 50 - 20 = 30
    
    @ SUBS - SUB แล้วอัพเดต flags
    MOV r4, #10
    MOV r5, #20
    SUBS r6, r4, r5      @ r6 = 10 - 20 = -10, N flag set
    
    BLT negative_result  @ Branch if Less Than
    B positive_result
    
negative_result:
    MOV r0, #-1
    B done
    
positive_result:
    MOV r0, #1
    
done:
    MOV r7, #1
    SWI 0
```

### 3.2 SBC (Subtract with Carry)

```asm
@ ARM32 - SBC สำหรับ 64-bit subtraction
@ ไฟล์: sbc_64bit.s

.section .text
.global _start

_start:
    @ ลบ 64-bit: A - B
    @ A = 0x00000002_00000000
    MOV r0, #0x00000000  @ low word ของ A
    MOV r1, #0x00000002  @ high word ของ A
    
    @ B = 0x00000000_00000001
    MOV r2, #0x00000001  @ low word ของ B
    MOV r3, #0x00000000  @ high word ของ B
    
    @ ลบ low words (SUBS เพื่อ set Carry/Borrow)
    SUBS r4, r0, r2      @ r4 = low result
    
    @ ลบ high words พร้อม borrow
    @ ARM: SBC = dst = op1 - op2 - (1 - Carry)
    SBC r5, r1, r3       @ r5 = r1 - r3 - !C
    
    @ ผลลัพธ์: r5:r4 = 0x00000001_FFFFFFFF
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 3.3 RSB (Reverse Subtract)

```asm
@ ARM32 - RSB instruction
@ RSB: dst = op2 - op1 (กลับลำดับการลบ)

.section .text
.global _start

_start:
    @ RSB: Rd = Operand2 - Rn
    MOV r0, #10
    RSB r1, r0, #100     @ r1 = 100 - r0 = 90
    
    @ ใช้ RSB สำหรับ negate (ค่าลบ)
    MOV r2, #5
    RSB r3, r2, #0       @ r3 = 0 - r2 = -5 (negation)
    
    @ RSB พร้อม shift
    MOV r4, #3
    MOV r5, #1
    RSB r6, r4, r5, LSL #4  @ r6 = (r5 << 4) - r4 = 16 - 3 = 13
    
    @ RSBS - RSB แล้วอัพเดต flags
    MOV r7, #5
    RSBS r0, r7, #3      @ r0 = 3 - 5 = -2, N flag set
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

---

## 4. AND, ORR, EOR, BIC - Logical Instructions

### 4.1 AND (Bitwise AND)

```asm
@ ARM32 - AND instruction examples
@ ไฟล์: logical_examples.s

.section .text
.global _start

_start:
    @ AND พื้นฐาน
    MOV r0, #0b11001100   @ r0 = 0xCC
    MOV r1, #0b10101010   @ r1 = 0xAA
    AND r2, r0, r1        @ r2 = 0b10001000 = 0x88
    
    @ AND สำหรับ mask (เอาเฉพาะ bits ที่ต้องการ)
    MOV r3, #0xFF00FF
    AND r4, r3, #0xFF     @ r4 = ดึงเฉพาะ byte ต่ำสุด = 0xFF
    
    @ AND สำหรับตรวจสอบ even/odd
    MOV r5, #7
    AND r6, r5, #1        @ r6 = 1 ถ้า odd, 0 ถ้า even
    CMP r6, #1
    BEQ is_odd
    B is_even
    
is_odd:
    MOV r0, #1
    B done
    
is_even:
    MOV r0, #0

done:
    @ ANDS - AND แล้วอัพเดต flags
    ANDS r0, r5, #1      @ ตรวจสอบ bit 0
    
    MOV r7, #1
    SWI 0
```

### 4.2 ORR (Bitwise OR)

```asm
@ ARM32 - ORR instruction examples

.section .text
.global _start

_start:
    @ ORR พื้นฐาน
    MOV r0, #0b11001100   @ 0xCC
    MOV r1, #0b10101010   @ 0xAA
    ORR r2, r0, r1        @ r2 = 0b11101110 = 0xEE
    
    @ ORR เพื่อ set bits
    MOV r3, #0b10100000
    ORR r4, r3, #0b00001111  @ ตั้ง 4 bits ต่ำ = 0b10101111
    
    @ ORR เพื่อรวม bytes เป็น word
    MOV r5, #0x12        @ byte 0
    MOV r6, #0x34        @ byte 1
    MOV r7, #0x56        @ byte 2
    MOV r8, #0x78        @ byte 3
    
    ORR r9, r5, r6, LSL #8   @ r9 = 0x3412
    ORR r9, r9, r7, LSL #16  @ r9 = 0x563412
    ORR r9, r9, r8, LSL #24  @ r9 = 0x78563412
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 4.3 EOR (Exclusive OR)

```asm
@ ARM32 - EOR instruction examples

.section .text
.global _start

_start:
    @ EOR พื้นฐาน
    MOV r0, #0b11001100
    MOV r1, #0b10101010
    EOR r2, r0, r1        @ r2 = 0b01100110 = 0x66
    
    @ EOR เพื่อ toggle bits
    MOV r3, #0b10110101
    EOR r4, r3, #0b00001111  @ toggle 4 bits ต่ำ
    
    @ EOR เพื่อ swap values (ไม่ใช้ register กลาง)
    MOV r5, #100
    MOV r6, #200
    EOR r5, r5, r6        @ r5 = r5 XOR r6
    EOR r6, r5, r6        @ r6 = (r5 XOR r6) XOR r6 = r5 original
    EOR r5, r5, r6        @ r5 = (r5 XOR r6) XOR r5 original = r6 original
    @ ตอนนี้ r5 = 200, r6 = 100
    
    @ EOR เพื่อ check ว่าสอง values เท่ากันหรือไม่
    MOV r7, #42
    MOV r8, #42
    EORS r9, r7, r8       @ r9 = 0 ถ้าเท่ากัน, Z flag set
    BEQ values_equal
    B values_different
    
values_equal:
    MOV r0, #1
    B done
    
values_different:
    MOV r0, #0
    
done:
    MOV r7, #1
    SWI 0
```

### 4.4 BIC (Bit Clear)

```asm
@ ARM32 - BIC instruction examples
@ BIC: Rd = Rn AND NOT(operand2)

.section .text
.global _start

_start:
    @ BIC พื้นฐาน
    MOV r0, #0b11111111   @ 0xFF
    BIC r1, r0, #0b00001111  @ r1 = r0 AND NOT(0x0F) = 0xF0
    
    @ BIC เพื่อ clear specific bits
    MOV r2, #0xFF         @ ทุก bits ถูก set
    BIC r3, r2, #0x3      @ clear bit 0 และ bit 1
    
    @ BIC ใช้กับ register
    MOV r4, #0b10101010
    MOV r5, #0b11110000
    BIC r6, r4, r5        @ r6 = r4 AND NOT(r5) = 0b00001010
    
    @ ใช้งานจริง: clear interrupt enable bit ใน control register
    @ สมมติ r7 เป็น control register
    MOV r7, #0xFF
    BIC r7, r7, #(1 << 3)  @ clear bit 3 (interrupt enable)
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

---

## 5. TST และ TEQ - Test Instructions

### 5.1 TST (Test)

```asm
@ ARM32 - TST instruction examples
@ TST = AND แต่ไม่เก็บผลลัพธ์ ใช้แค่ set flags

.section .text
.global _start

_start:
    @ TST ตรวจสอบ bit
    MOV r0, #0b10110100
    
    @ ตรวจสอบ bit 2
    TST r0, #(1 << 2)    @ 0b00000100
    BNE bit2_set          @ Branch if Not Equal (bit2 != 0)
    
    @ ตรวจสอบ bit 4
    TST r0, #(1 << 4)    @ 0b00010000
    BNE bit4_set
    
    @ ตรวจสอบ byte ต่ำสุด
    TST r0, #0xFF
    BNE lower_byte_nonzero
    
bit2_set:
    MOV r1, #2
    B continue_check
    
bit4_set:
    MOV r1, #4
    B continue_check
    
lower_byte_nonzero:
    MOV r1, #0xFF
    
continue_check:
    @ ตรวจสอบว่า word-aligned หรือไม่
    MOV r2, #0x1004       @ address
    TST r2, #3            @ ตรวจสอบ 2 bits ต่ำ
    BEQ word_aligned
    B not_aligned
    
word_aligned:
    MOV r0, #1
    B done
    
not_aligned:
    MOV r0, #0
    
done:
    MOV r7, #1
    SWI 0
```

### 5.2 TEQ (Test Equivalence)

```asm
@ ARM32 - TEQ instruction examples
@ TEQ = EOR แต่ไม่เก็บผลลัพธ์ ใช้แค่ set flags

.section .text
.global _start

_start:
    @ TEQ ตรวจสอบความเท่ากัน
    MOV r0, #42
    MOV r1, #42
    TEQ r0, r1           @ ถ้าเท่ากัน Z flag set
    BEQ values_match
    B values_differ
    
values_match:
    MOV r2, #1
    B check_sign
    
values_differ:
    MOV r2, #0
    
check_sign:
    @ TEQ ตรวจสอบ sign
    MOV r3, #-5
    TEQ r3, #0           @ ตรวจสอบ sign bit
    BMI is_negative       @ Branch if Minus (N flag set)
    B is_positive
    
is_negative:
    MOV r0, #-1
    B done
    
is_positive:
    MOV r0, #1
    
done:
    MOV r7, #1
    SWI 0
```

---

## 6. CMP และ CMN - Compare Instructions

### 6.1 CMP (Compare)

```asm
@ ARM32 - CMP instruction examples
@ CMP = SUB แต่ไม่เก็บผลลัพธ์

.section .text
.global _start

_start:
    @ CMP พื้นฐาน
    MOV r0, #10
    MOV r1, #20
    CMP r0, r1           @ r0 - r1 = -10, N flag set
    
    BLT r0_less          @ r0 < r1
    BGT r0_greater       @ r0 > r1
    BEQ r0_equal         @ r0 == r1
    
r0_less:
    MOV r2, #-1
    B done
    
r0_greater:
    MOV r2, #1
    B done
    
r0_equal:
    MOV r2, #0
    
done:
    @ ตัวอย่าง: หาค่าสูงสุด
    MOV r3, #15
    MOV r4, #23
    CMP r3, r4
    MOVGT r5, r3         @ ถ้า r3 > r4 แล้ว r5 = r3
    MOVLE r5, r4         @ ถ้า r3 <= r4 แล้ว r5 = r4
    @ r5 ควรเป็น 23
    
    @ CMP กับ loop counter
    MOV r6, #0
    MOV r7, #10
    
loop:
    ADD r6, r6, #1       @ r6++
    CMP r6, r7           @ เทียบกับ 10
    BLT loop             @ วนซ้ำถ้า r6 < 10
    
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 6.2 CMN (Compare Negative)

```asm
@ ARM32 - CMN instruction examples
@ CMN = ADD แต่ไม่เก็บผลลัพธ์
@ CMN r0, r1 เหมือน CMP r0, -r1

.section .text
.global _start

_start:
    @ CMN พื้นฐาน
    MOV r0, #5
    CMN r0, #-5          @ เหมือน CMP r0, 5
    BEQ equal_to_neg5
    B not_equal
    
equal_to_neg5:
    MOV r1, #1
    B check_positive
    
not_equal:
    MOV r1, #0
    
check_positive:
    @ CMN ตรวจสอบว่า r0 = -r1
    MOV r2, #10
    MVN r3, #9           @ r3 = -10
    CMN r2, r3           @ r2 + (-(-10)) = r2 + 10, check if = 0
    
    @ ใช้ CMN กับ signed ค่า
    MOV r4, #-5
    CMN r4, #5           @ -5 + 5 = 0, Z flag set
    BEQ sum_is_zero
    
sum_is_zero:
    MOV r0, #42
    
    MOV r7, #1
    SWI 0
```

---

## 7. MUL และ MLA - Multiply Instructions

### 7.1 MUL (Multiply)

```asm
@ ARM32 - MUL instruction examples
@ ไฟล์: mul_examples.s

.section .text
.global _start

_start:
    @ MUL: Rd = Rm * Rs (lower 32 bits)
    MOV r0, #10
    MOV r1, #20
    MUL r2, r0, r1       @ r2 = 10 * 20 = 200
    
    @ หมายเหตุ: Rd ต้องไม่เหมือนกับ Rm ใน ARM
    @ MUL r0, r0, r1   <- ผิด! Rd = Rm ไม่ได้
    MOV r3, #15
    MOV r4, #7
    MUL r5, r3, r4       @ r5 = 105
    
    @ MULS - MUL แล้วอัพเดต flags
    MOV r6, #-3
    MOV r7, #4
    MULS r8, r6, r7      @ r8 = -12, N flag set
    
    @ คูณ powers of 2 ด้วย LSL
    MOV r9, #5
    MOV r10, r9, LSL #3  @ r10 = 5 * 8 = 40 (เร็วกว่า MUL)
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 7.2 MLA (Multiply-Accumulate)

```asm
@ ARM32 - MLA instruction examples
@ MLA: Rd = Rm * Rs + Rn

.section .text
.global _start

_start:
    @ MLA: คูณแล้วบวก accumulator
    MOV r0, #3           @ multiplier
    MOV r1, #4           @ multiplicand
    MOV r2, #10          @ accumulator
    MLA r3, r0, r1, r2   @ r3 = (3 * 4) + 10 = 22
    
    @ ตัวอย่าง: dot product ของ vectors
    @ A = [1, 2, 3], B = [4, 5, 6]
    @ dot = 1*4 + 2*5 + 3*6 = 32
    
    MOV r0, #0           @ accumulator เริ่มต้น
    
    MOV r1, #1           @ A[0]
    MOV r2, #4           @ B[0]
    MLA r0, r1, r2, r0   @ r0 = 1*4 + 0 = 4
    
    MOV r1, #2           @ A[1]
    MOV r2, #5           @ B[1]
    MLA r0, r1, r2, r0   @ r0 = 2*5 + 4 = 14
    
    MOV r1, #3           @ A[2]
    MOV r2, #6           @ B[2]
    MLA r0, r1, r2, r0   @ r0 = 3*6 + 14 = 32
    
    @ Exit
    MOV r7, #1
    SWI 0
```

---

## 8. UMULL, UMLAL - Unsigned Long Multiply

### 8.1 UMULL (Unsigned Multiply Long)

```asm
@ ARM32 - UMULL instruction examples
@ UMULL: RdHi:RdLo = Rm * Rs (64-bit unsigned)

.section .text
.global _start

_start:
    @ UMULL: คูณ unsigned 32-bit ได้ผล 64-bit
    MOV r0, #0xFFFFFFFF  @ max unsigned 32-bit
    MOV r1, #0xFFFFFFFF
    UMULL r2, r3, r0, r1 @ r3:r2 = 0xFFFFFFFF * 0xFFFFFFFF
    @ ผลลัพธ์: 0xFFFFFFFE00000001
    @ r2 = 0x00000001 (low)
    @ r3 = 0xFFFFFFFE (high)
    
    @ ตัวอย่าง: คูณเลขขนาดใหญ่
    LDR r4, =1000000     @ 1,000,000
    LDR r5, =2000000     @ 2,000,000
    UMULL r6, r7, r4, r5 @ r7:r6 = 2,000,000,000,000
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 8.2 UMLAL (Unsigned Multiply-Accumulate Long)

```asm
@ ARM32 - UMLAL instruction examples
@ UMLAL: RdHi:RdLo = Rm * Rs + RdHi:RdLo

.section .text
.global _start

_start:
    @ UMLAL: คูณแล้วบวกเข้า 64-bit accumulator
    MOV r0, #0           @ RdLo = 0
    MOV r1, #0           @ RdHi = 0
    
    MOV r2, #100
    MOV r3, #200
    UMLAL r0, r1, r2, r3 @ r1:r0 = 100 * 200 + 0 = 20000
    
    MOV r2, #50
    MOV r3, #300
    UMLAL r0, r1, r2, r3 @ r1:r0 = 50 * 300 + 20000 = 35000
    
    @ Exit
    MOV r7, #1
    SWI 0
```

---

## 9. SMULL, SMLAL - Signed Long Multiply

### 9.1 SMULL (Signed Multiply Long)

```asm
@ ARM32 - SMULL instruction examples
@ SMULL: RdHi:RdLo = Rm * Rs (64-bit signed)

.section .text
.global _start

_start:
    @ SMULL: คูณ signed 32-bit ได้ผล 64-bit
    MOV r0, #-100        @ signed negative
    MOV r1, #200
    SMULL r2, r3, r0, r1 @ r3:r2 = -100 * 200 = -20000
    
    @ คูณค่า negative ทั้งคู่
    MOV r4, #-5
    MOV r5, #-7
    SMULL r6, r7, r4, r5 @ r7:r6 = -5 * -7 = 35 (positive)
    
    @ ตรวจสอบผลลัพธ์ 64-bit
    @ r7 เป็น sign extension ของ r6 ถ้าผลลัพธ์เป็น 32-bit
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 9.2 SMLAL (Signed Multiply-Accumulate Long)

```asm
@ ARM32 - SMLAL instruction examples
@ SMLAL: RdHi:RdLo = Rm * Rs + RdHi:RdLo (signed)

.section .text
.global _start

_start:
    @ SMLAL: คูณ signed แล้วบวกเข้า 64-bit accumulator
    MOV r0, #0           @ RdLo = 0
    MOV r1, #0           @ RdHi = 0
    
    MOV r2, #-10
    MOV r3, #5
    SMLAL r0, r1, r2, r3 @ r1:r0 = (-10 * 5) + 0 = -50
    
    MOV r2, #20
    MOV r3, #3
    SMLAL r0, r1, r2, r3 @ r1:r0 = (20 * 3) + (-50) = 10
    
    @ Exit
    MOV r7, #1
    SWI 0
```

---

## 10. CLZ - Count Leading Zeros

### 10.1 CLZ (Count Leading Zeros)

```asm
@ ARM32 - CLZ instruction examples
@ CLZ: นับจำนวน 0 bits ที่อยู่ข้างหน้า (leading zeros)
@ CLZ ต้องการ ARMv5 หรือสูงกว่า

.section .text
.global _start

_start:
    @ CLZ พื้นฐาน
    MOV r0, #1
    CLZ r1, r0           @ r1 = 31 (มี 31 leading zeros)
    
    MOV r0, #0x80000000  @ bit 31 set
    CLZ r1, r0           @ r1 = 0 (ไม่มี leading zeros)
    
    MOV r0, #0x00010000  @ bit 16 set
    CLZ r1, r0           @ r1 = 15
    
    MOV r0, #0
    CLZ r1, r0           @ r1 = 32 (ทุก bit เป็น 0)
    
    @ ใช้ CLZ หา floor(log2(n))
    MOV r2, #100
    CLZ r3, r2           @ r3 = 25 (leading zeros)
    RSB r4, r3, #31      @ r4 = 31 - 25 = 6
    @ log2(100) ≈ 6 (floor)
    
    @ ใช้ CLZ normalize เลข
    MOV r5, #0x00001234
    CLZ r6, r5           @ นับ leading zeros
    MOV r7, r5, LSL r6   @ shift จนกระทั่ง MSB = 1
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

---

## 11. Conditional Execution Suffixes

### 11.1 ตารางเงื่อนไข Condition Codes

```
Suffix  Flags              ความหมาย
EQ      Z=1                Equal (เท่ากัน)
NE      Z=0                Not Equal (ไม่เท่ากัน)
CS/HS   C=1                Carry Set / Unsigned Higher or Same
CC/LO   C=0                Carry Clear / Unsigned Lower
MI      N=1                Minus (ลบ)
PL      N=0                Plus (บวก หรือ ศูนย์)
VS      V=1                Overflow Set
VC      V=0                Overflow Clear
HI      C=1 AND Z=0        Unsigned Higher
LS      C=0 OR Z=1         Unsigned Lower or Same
GE      N=V                Signed Greater than or Equal
LT      N!=V               Signed Less Than
GT      Z=0 AND N=V        Signed Greater Than
LE      Z=1 OR N!=V        Signed Less than or Equal
AL      (always)           Always (ค่า default)
```

### 11.2 ตัวอย่าง Conditional Execution

```asm
@ ARM32 - Conditional execution examples
@ ไฟล์: conditional_exec.s

.section .text
.global _start

_start:
    @ ตัวอย่าง: if/else ด้วย conditional execution
    MOV r0, #15
    MOV r1, #10
    
    CMP r0, r1           @ เปรียบเทียบ r0 กับ r1
    MOVGT r2, r0         @ ถ้า r0 > r1 แล้ว r2 = r0
    MOVLE r2, r1         @ ถ้า r0 <= r1 แล้ว r2 = r1
    @ r2 = max(r0, r1) = 15
    
    @ ตัวอย่าง: absolute value
    MOV r3, #-7
    CMP r3, #0           @ เทียบกับ 0
    RSBLT r3, r3, #0     @ ถ้า r3 < 0 แล้ว r3 = -r3
    @ r3 = |r3| = 7
    
    @ ตัวอย่าง: clamp value ให้อยู่ใน range [0, 100]
    MOV r4, #150
    CMP r4, #100
    MOVGT r4, #100       @ ถ้า > 100 แล้ว = 100
    CMP r4, #0
    MOVLT r4, #0         @ ถ้า < 0 แล้ว = 0
    @ r4 = clamp(150, 0, 100) = 100
    
    @ ตัวอย่าง: Fibonacci ด้วย conditional
    MOV r5, #0           @ fib(0) = 0
    MOV r6, #1           @ fib(1) = 1
    MOV r7, #10          @ คำนวณ 10 ครั้ง
    
fib_loop:
    ADD r8, r5, r6       @ r8 = r5 + r6
    MOV r5, r6           @ r5 = r6
    MOV r6, r8           @ r6 = r8
    SUBS r7, r7, #1      @ r7--
    BNE fib_loop         @ วนถ้า r7 != 0
    @ r6 = fib(12) = 144
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 11.3 EQ, NE Conditions

```asm
@ ARM32 - EQ/NE condition examples

.section .text
.global _start

_start:
    MOV r0, #5
    MOV r1, #5
    CMP r0, r1
    
    ADDEQ r2, r0, #1     @ r0 == r1: r2 = r0 + 1
    ADDNE r3, r0, #10    @ r0 != r1: r3 = r0 + 10
    
    @ LDREQ/STREQ
    MOV r4, #0
    CMP r4, #0
    MOVEQ r5, #100       @ ถ้า r4 == 0 แล้ว r5 = 100
    
    @ ใช้ multiple conditional
    MOV r6, #3
    CMP r6, #1           @ r6 == 1?
    MOVEQ r7, #10
    CMP r6, #2           @ r6 == 2?
    MOVEQ r7, #20
    CMP r6, #3           @ r6 == 3?
    MOVEQ r7, #30        @ r6 == 3, r7 = 30
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 11.4 LT, LE, GT, GE Conditions (Signed)

```asm
@ ARM32 - Signed comparison conditions

.section .text
.global _start

_start:
    @ ตัวอย่าง: signed sort (bubble sort step)
    MOV r0, #-5
    MOV r1, #3
    
    CMP r0, r1
    @ ถ้า r0 > r1 แล้ว swap
    MOVGT r2, r0
    MOVGT r0, r1
    MOVGT r1, r2
    @ ตอนนี้ r0 = min = -5, r1 = max = 3
    
    @ ตัวอย่าง: signed range check
    MOV r3, #-50
    CMP r3, #-100        @ r3 >= -100?
    BGE check_upper
    MOV r4, #0           @ ไม่อยู่ใน range
    B done
    
check_upper:
    CMP r3, #100         @ r3 <= 100?
    BLE in_range
    MOV r4, #0           @ ไม่อยู่ใน range
    B done
    
in_range:
    MOV r4, #1           @ อยู่ใน range
    
done:
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 11.5 CS, CC, HI, LS Conditions (Unsigned)

```asm
@ ARM32 - Unsigned comparison conditions

.section .text
.global _start

_start:
    @ ตัวอย่าง: unsigned comparison
    MOV r0, #200         @ unsigned
    MOV r1, #100
    
    CMP r0, r1           @ ตรวจสอบ unsigned
    MOVHI r2, r0         @ ถ้า r0 > r1 (unsigned)
    MOVLS r2, r1         @ ถ้า r0 <= r1 (unsigned)
    
    @ CS/CC สำหรับ carry
    MOV r3, #0xFFFFFFFF
    ADDS r4, r3, #1      @ overflow! C set
    MOVCS r5, #1         @ ถ้า Carry Set
    MOVCC r5, #0         @ ถ้า Carry Clear
    
    @ ตัวอย่าง: unsigned range [0, 255]
    MOV r6, #300         @ ค่าเกิน 255
    CMP r6, #255
    MOVHI r6, #255       @ clamp ให้ไม่เกิน 255
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

---

## 12. S Suffix - Update Flags

### 12.1 การใช้ S Suffix

```asm
@ ARM32 - S suffix examples

.section .text
.global _start

_start:
    @ ADDS vs ADD: ความแตกต่าง
    MOV r0, #10
    ADD r1, r0, #5       @ r1 = 15, flags ไม่เปลี่ยน
    ADDS r2, r0, #5      @ r2 = 15, flags updated
    
    @ SUBS vs SUB
    MOV r3, #0
    SUB r4, r3, #1       @ r4 = -1, flags ไม่เปลี่ยน
    SUBS r5, r3, #1      @ r5 = -1, N flag set, C flag clear
    
    @ MOVS vs MOV
    MOV r6, #0           @ flags ไม่เปลี่ยน
    MOVS r7, #0          @ Z flag set
    
    @ ANDS, ORRS, EORS, BICS
    MOV r8, #0xFF
    ANDS r9, r8, #0x0F   @ r9 = 0x0F, flags updated
    
    @ ตัวอย่าง: loop counter ด้วย SUBS
    MOV r10, #5          @ counter
    
countdown:
    @ ทำงานบางอย่าง
    SUBS r10, r10, #1    @ r10-- และ update flags
    BNE countdown        @ วนถ้า r10 != 0
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

---

## 13. Shift Operations ใน Data Processing

### 13.1 Barrel Shifter

```asm
@ ARM32 - Barrel Shifter in data processing
@ ไฟล์: barrel_shifter.s

.section .text
.global _start

_start:
    @ LSL (Logical Shift Left) - คูณด้วย 2^n
    MOV r0, #5
    MOV r1, r0, LSL #1   @ r1 = 5 * 2 = 10
    MOV r2, r0, LSL #3   @ r2 = 5 * 8 = 40
    MOV r3, r0, LSL #4   @ r3 = 5 * 16 = 80
    
    @ LSR (Logical Shift Right) - หารด้วย 2^n (unsigned)
    MOV r4, #100
    MOV r5, r4, LSR #1   @ r5 = 100 / 2 = 50
    MOV r6, r4, LSR #2   @ r6 = 100 / 4 = 25
    
    @ ASR (Arithmetic Shift Right) - หารด้วย 2^n (signed)
    MOV r7, #-16
    MOV r8, r7, ASR #1   @ r8 = -16 / 2 = -8 (sign preserved)
    MOV r9, r7, ASR #2   @ r9 = -16 / 4 = -4
    
    @ ROR (Rotate Right)
    MOV r10, #0b10110001
    MOV r11, r10, ROR #1  @ rotate right 1
    
    @ RRX (Rotate Right Extended - 1 bit through Carry)
    @ ต้องมี Carry flag set ก่อน
    MOV r12, #1
    ADDS r13, r12, r12    @ C = 0 (no overflow)
    MOV r0, #0b10000000
    MOV r1, r0, RRX       @ rotate right through carry
    
    @ ใช้ register เป็น shift amount
    MOV r2, #0xFF
    MOV r3, #4
    MOV r4, r2, LSL r3    @ r4 = r2 << r3 = 0xFF0
    
    @ Shift ใน ADD/SUB
    MOV r5, #10
    MOV r6, #3
    ADD r7, r5, r5, LSL #1  @ r7 = r5 + r5*2 = r5*3 = 30
    ADD r8, r5, r5, LSL #2  @ r8 = r5 + r5*4 = r5*5 = 50
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 13.2 Shift ใน ARM AArch64

```asm
// AArch64 - Shift operations
// ไฟล์: shift64.s

.section .text
.global _start

_start:
    // LSL
    MOV x0, #1
    LSL x1, x0, #10      // x1 = 1 << 10 = 1024
    
    // LSR
    MOV x2, #1024
    LSR x3, x2, #2       // x3 = 1024 >> 2 = 256
    
    // ASR
    MOV x4, #-64
    ASR x5, x4, #2       // x5 = -64 >> 2 = -16
    
    // ROR
    MOV x6, #0x8000000000000001
    ROR x7, x6, #1       // rotate right 1 bit
    
    // Shift ใน instruction
    MOV x8, #5
    MOV x9, #3
    ADD x10, x8, x9, LSL #2  // x10 = x8 + (x9 << 2) = 5 + 12 = 17
    
    // EXTR - extract register (combine two regs with bit range)
    MOV x11, #0xABCDEF0123456789
    MOV x12, #0x1234567890ABCDEF
    EXTR x13, x11, x12, #32 // extract 64 bits starting at bit 32
    
    // Exit
    MOV x8, #93
    MOV x0, #0
    SVC #0
```

---

## 14. โปรแกรมตัวอย่างสมบูรณ์

### 14.1 Calculator โดยใช้ Data Processing

```asm
@ ARM32 - Simple Calculator
@ ไฟล์: calculator.s
@ รองรับ +, -, *, /, % สำหรับ integers

.section .data
result_msg: .ascii "Result: \0"
newline:    .ascii "\n\0"

.section .text
.global _start

@ ฟังก์ชัน: r0 = add(r0, r1)
add_func:
    ADD r0, r0, r1
    BX lr

@ ฟังก์ชัน: r0 = sub(r0, r1)
sub_func:
    SUB r0, r0, r1
    BX lr

@ ฟังก์ชัน: r0 = mul(r0, r1)
mul_func:
    MUL r0, r0, r1
    BX lr

@ ฟังก์ชัน: r0 = div(r0, r1) - integer division
@ r0 = numerator, r1 = denominator
@ ผลลัพธ์: r0 = quotient, r1 = remainder
div_func:
    MOV r2, #0           @ quotient = 0
    CMP r1, #0           @ ตรวจสอบหาร 0
    BEQ div_by_zero
    
    @ ตรวจสอบ sign
    MOV r3, #1           @ sign positive
    CMP r0, #0
    RSBLT r0, r0, #0     @ |r0|
    MOVLT r3, #-1
    CMP r1, #0
    RSBLT r1, r1, #0     @ |r1|
    MOVLT r3, r3, EOR #2 @ toggle sign
    
div_loop:
    CMP r0, r1
    BLT div_done
    SUB r0, r0, r1       @ r0 -= r1
    ADD r2, r2, #1       @ quotient++
    B div_loop
    
div_done:
    @ r0 = remainder (ต้องนำ sign กลับ)
    MOV r4, r0           @ save remainder
    MOV r0, r2           @ quotient
    
    @ Apply sign
    CMP r3, #-1
    RSBEQ r0, r0, #0
    
    MOV r1, r4           @ return remainder in r1
    BX lr
    
div_by_zero:
    MOV r0, #0
    MOV r1, #0
    BX lr

@ ฟังก์ชัน: r0 = abs(r0)
abs_func:
    CMP r0, #0
    RSBLT r0, r0, #0
    BX lr

@ ฟังก์ชัน: r0 = max(r0, r1)
max_func:
    CMP r0, r1
    MOVLT r0, r1
    BX lr

@ ฟังก์ชัน: r0 = min(r0, r1)
min_func:
    CMP r0, r1
    MOVGT r0, r1
    BX lr

@ ฟังก์ชัน: r0 = clamp(r0, r1, r2)
@ r0 = value, r1 = min, r2 = max
clamp_func:
    CMP r0, r1
    MOVLT r0, r1         @ if < min then = min
    CMP r0, r2
    MOVGT r0, r2         @ if > max then = max
    BX lr

_start:
    @ ทดสอบ add: 25 + 17 = 42
    MOV r0, #25
    MOV r1, #17
    BL add_func
    @ r0 = 42
    
    @ ทดสอบ sub: 100 - 37 = 63
    MOV r0, #100
    MOV r1, #37
    BL sub_func
    @ r0 = 63
    
    @ ทดสอบ mul: 12 * 11 = 132
    MOV r0, #12
    MOV r1, #11
    BL mul_func
    @ r0 = 132
    
    @ ทดสอบ div: 100 / 7 = 14 remainder 2
    MOV r0, #100
    MOV r1, #7
    BL div_func
    @ r0 = 14, r1 = 2
    
    @ ทดสอบ abs: abs(-42) = 42
    MOV r0, #-42
    BL abs_func
    @ r0 = 42
    
    @ ทดสอบ max: max(15, 23) = 23
    MOV r0, #15
    MOV r1, #23
    BL max_func
    @ r0 = 23
    
    @ ทดสอบ clamp: clamp(150, 0, 100) = 100
    MOV r0, #150
    MOV r1, #0
    MOV r2, #100
    BL clamp_func
    @ r0 = 100
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 14.2 Bit Manipulation Library

```asm
@ ARM32 - Bit Manipulation Library
@ ไฟล์: bitlib.s

.section .text
.global _start

@ === Bit Set/Clear/Toggle/Test ===

@ bit_set(value, bit_pos): ตั้ง bit ตำแหน่งที่กำหนด
@ r0 = value, r1 = bit position
@ return r0 = value with bit set
bit_set:
    MOV r2, #1
    MOV r2, r2, LSL r1   @ r2 = 1 << bit_pos
    ORR r0, r0, r2        @ set the bit
    BX lr

@ bit_clear(value, bit_pos): clear bit ตำแหน่งที่กำหนด
@ r0 = value, r1 = bit position
bit_clear:
    MOV r2, #1
    MOV r2, r2, LSL r1
    BIC r0, r0, r2        @ clear the bit
    BX lr

@ bit_toggle(value, bit_pos): toggle bit ตำแหน่งที่กำหนด
bit_toggle:
    MOV r2, #1
    MOV r2, r2, LSL r1
    EOR r0, r0, r2        @ toggle the bit
    BX lr

@ bit_test(value, bit_pos): ทดสอบ bit ตำแหน่งที่กำหนด
@ return r0 = 0 ถ้า bit = 0, != 0 ถ้า bit = 1
bit_test:
    MOV r2, #1
    MOV r2, r2, LSL r1
    AND r0, r0, r2
    BX lr

@ popcount(value): นับจำนวน bits ที่เป็น 1
@ r0 = value
@ return r0 = count
popcount:
    MOV r1, #0           @ count = 0
    
popcount_loop:
    CMP r0, #0
    BEQ popcount_done
    
    ANDS r2, r0, #1      @ ตรวจสอบ bit ต่ำสุด
    ADDNE r1, r1, #1     @ ถ้า bit = 1 แล้วนับ
    MOV r0, r0, LSR #1   @ shift right
    B popcount_loop
    
popcount_done:
    MOV r0, r1
    BX lr

@ Hamming weight using Brian Kernighan's algorithm
@ (เร็วกว่าการนับทีละ bit)
hamming_weight:
    MOV r1, #0           @ count = 0
    
hw_loop:
    CMP r0, #0
    BEQ hw_done
    SUB r2, r0, #1       @ r2 = r0 - 1
    AND r0, r0, r2        @ r0 = r0 & (r0 - 1), clear lowest set bit
    ADD r1, r1, #1
    B hw_loop
    
hw_done:
    MOV r0, r1
    BX lr

@ reverse_bits(value): กลับ bits ทั้งหมด
@ r0 = value
reverse_bits:
    MOV r1, #0
    MOV r2, #32          @ 32 bits
    
reverse_loop:
    AND r3, r0, #1       @ bit ต่ำสุด
    ORR r1, r3, r1, LSL #1  @ shift r1 left and add bit
    MOV r0, r0, LSR #1   @ next bit
    SUBS r2, r2, #1
    BNE reverse_loop
    
    MOV r0, r1
    BX lr

@ is_power_of_2(value): ตรวจสอบว่าเป็น power of 2
@ r0 = value
@ return r0 = 1 ถ้าใช่, 0 ถ้าไม่ใช่
is_power_of_2:
    CMP r0, #0
    MOVEQ r0, #0         @ 0 ไม่ใช่ power of 2
    BEQ pow2_done
    
    SUB r1, r0, #1       @ r1 = r0 - 1
    TST r0, r1           @ r0 & (r0 - 1) == 0 ถ้าเป็น power of 2
    MOVEQ r0, #1
    MOVNE r0, #0
    
pow2_done:
    BX lr

_start:
    @ ทดสอบ bit_set: set bit 3 ใน 0b10100000
    MOV r0, #0b10100000
    MOV r1, #3
    BL bit_set
    @ r0 = 0b10101000
    
    @ ทดสอบ popcount: count 1s ใน 0b10101010
    MOV r0, #0b10101010
    BL popcount
    @ r0 = 4
    
    @ ทดสอบ hamming_weight: count 1s ใน 0xFF
    MOV r0, #0xFF
    BL hamming_weight
    @ r0 = 8
    
    @ ทดสอบ is_power_of_2
    MOV r0, #64          @ 64 = 2^6
    BL is_power_of_2
    @ r0 = 1
    
    @ Exit
    MOV r7, #1
    MOV r0, #0
    SWI 0
```

### 14.3 AArch64 Data Processing Example

```asm
// AArch64 - Comprehensive data processing example
// ไฟล์: aarch64_dp.s

.section .text
.global _start

// ฟังก์ชัน: 64-bit addition
// x0, x1 = operands
// return x0 = sum
add64:
    ADD x0, x0, x1
    RET

// ฟังก์ชัน: 128-bit addition using pairs
// x0:x1 = A (high:low), x2:x3 = B (high:low)
// return x0:x1 = result
add128:
    ADDS x1, x1, x3      // low + low, set carry
    ADC x0, x0, x2       // high + high + carry
    RET

// ฟังก์ชัน: mul64 with 128-bit result
// x0, x1 = operands
// return x0:x1 = result (high:low)
mul128:
    MUL x2, x0, x1       // low 64 bits
    UMULH x3, x0, x1     // high 64 bits
    MOV x0, x3
    MOV x1, x2
    RET

// ฟังก์ชัน: count leading zeros
// x0 = value
// return x0 = count
clz64:
    CLZ x0, x0
    RET

// ฟังก์ชัน: bit reversal
// x0 = value
// return x0 = reversed
rbit64:
    RBIT x0, x0
    RET

// ฟังก์ชัน: byte reversal
// x0 = value
// return x0 = byte reversed
rev64:
    REV x0, x0
    RET

// ฟังก์ชัน: extract bits [hi:lo]
// x0 = value, x1 = lsb, x2 = width
// return x0 = extracted bits
extract_bits:
    LSR x0, x0, x1       // shift right to lsb position
    MOV x3, #1
    LSL x3, x3, x2       // 1 << width
    SUB x3, x3, #1       // mask = (1 << width) - 1
    AND x0, x0, x3       // apply mask
    RET

_start:
    // ทดสอบ add64
    MOV x0, #0x7FFFFFFFFFFFFFFF
    MOV x1, #1
    BL add64
    // x0 = 0x8000000000000000 (overflow!)
    
    // ทดสอบ add128
    MOV x0, #0           // high A
    MOV x1, #0xFFFFFFFFFFFFFFFF  // low A
    MOV x2, #0           // high B
    MOV x3, #1           // low B
    BL add128
    // x0:x1 = 1:0 = 0x10000000000000000
    
    // ทดสอบ mul128
    MOV x0, #0xFFFFFFFF
    MOV x1, #0xFFFFFFFF
    BL mul128
    // x0:x1 = 0:0xFFFFFFFE00000001
    
    // ทดสอบ clz64
    MOV x0, #1
    BL clz64
    // x0 = 63
    
    // ทดสอบ rbit64
    MOV x0, #0x8000000000000001
    BL rbit64
    // x0 = 0x8000000000000001 (palindrome!)
    
    // ทดสอบ extract_bits: bits [7:4] ของ 0xAB
    MOV x0, #0xAB
    MOV x1, #4           // lsb = 4
    MOV x2, #4           // width = 4
    BL extract_bits
    // x0 = 0xA
    
    // Exit
    MOV x8, #93
    MOV x0, #0
    SVC #0
```

---

## 15. การ Compile และ Execute ด้วย QEMU

### 15.1 ติดตั้ง Tools

```bash
# Ubuntu/Debian
sudo apt-get install gcc-arm-linux-gnueabihf
sudo apt-get install qemu-user
sudo apt-get install binutils-arm-linux-gnueabihf

# สำหรับ AArch64
sudo apt-get install gcc-aarch64-linux-gnu
sudo apt-get install qemu-user

# ตรวจสอบการติดตั้ง
arm-linux-gnueabihf-as --version
qemu-arm --version
```

### 15.2 Compile ARM32

```bash
# Assemble
arm-linux-gnueabihf-as -o output.o source.s

# Link (standalone executable)
arm-linux-gnueabihf-ld -o program output.o

# หรือ link กับ C library
arm-linux-gnueabihf-gcc -static -o program source.s

# Run ด้วย QEMU
qemu-arm ./program
qemu-arm -g 1234 ./program  # Debug mode (port 1234)

# ดู disassembly
arm-linux-gnueabihf-objdump -d program
```

### 15.3 Compile AArch64

```bash
# Assemble
aarch64-linux-gnu-as -o output.o source.s

# Link
aarch64-linux-gnu-ld -o program output.o

# Run ด้วย QEMU
qemu-aarch64 ./program

# Debug
qemu-aarch64 -g 1234 ./program
```

### 15.4 Makefile สำหรับโปรเจกต์

```makefile
# Makefile สำหรับ ARM Assembly

# Toolchain
ARM32_AS = arm-linux-gnueabihf-as
ARM32_LD = arm-linux-gnueabihf-ld
ARM32_OBJDUMP = arm-linux-gnueabihf-objdump
ARM32_RUN = qemu-arm

ARM64_AS = aarch64-linux-gnu-as
ARM64_LD = aarch64-linux-gnu-ld
ARM64_RUN = qemu-aarch64

# ตัวแปร
ASFLAGS = -g

.PHONY: all clean run32 run64

all: program32 program64

# ARM32
program32: main32.o
	$(ARM32_LD) -o $@ $^

main32.o: main32.s
	$(ARM32_AS) $(ASFLAGS) -o $@ $<

# ARM64
program64: main64.o
	$(ARM64_LD) -o $@ $^

main64.o: main64.s
	$(ARM64_AS) $(ASFLAGS) -o $@ $<

run32: program32
	$(ARM32_RUN) ./program32

run64: program64
	$(ARM64_RUN) ./program64

disasm32: program32
	$(ARM32_OBJDUMP) -d $<

disasm64: program64
	$(ARM64_OBJDUMP) -d $<

clean:
	rm -f *.o program32 program64
```

### 15.5 ตัวอย่าง Debug Session

```bash
# Terminal 1: Start program in debug mode
qemu-arm -g 1234 ./calculator

# Terminal 2: Connect GDB
arm-linux-gnueabihf-gdb calculator

# ใน GDB:
(gdb) target remote :1234
(gdb) break _start
(gdb) continue
(gdb) info registers    # ดู registers ทั้งหมด
(gdb) print $r0         # ดูค่า r0
(gdb) stepi             # execute 1 instruction
(gdb) nexti             # execute 1 instruction (skip call)
(gdb) x/10i $pc         # ดู 10 instructions จาก PC
```

---

## 16. แบบฝึกหัด (Exercises)

### Exercise 1: Implement Factorial

```asm
@ แบบฝึกหัด 1: คำนวณ n! ด้วย ARM32
@ ใช้คำสั่ง MUL และ SUBS
@ Input: r0 = n
@ Output: r0 = n!

.section .text
.global factorial

factorial:
    @ TODO: implement factorial
    @ Hint: use loop with MUL
    @ Special case: 0! = 1
    
    CMP r0, #0
    MOVEQ r0, #1
    MOVLE pc, lr
    
    MOV r1, r0           @ save n
    MOV r0, #1           @ result = 1
    
fact_loop:
    @ TODO: r0 = r0 * r1
    @       r1--
    @       if r1 > 0 goto fact_loop
    
    BX lr
```

**เฉลย Exercise 1:**

```asm
factorial:
    CMP r0, #0
    MOVEQ r0, #1
    MOVLE pc, lr
    
    MOV r1, r0           @ n
    MOV r0, #1           @ result = 1
    
fact_loop:
    MUL r0, r0, r1       @ result *= n
    SUBS r1, r1, #1      @ n--
    BGT fact_loop        @ ถ้า n > 0 ให้วนต่อ
    
    BX lr
```

### Exercise 2: Implement Bitwise Operations

```asm
@ แบบฝึกหัด 2: ฟังก์ชัน encode/decode ง่ายๆ
@ Encode: XOR ทุก byte ด้วย key
@ Decode: XOR อีกครั้งด้วย key เดิม

@ ฟังก์ชัน xor_encode(buffer, length, key)
@ r0 = buffer address, r1 = length, r2 = key
xor_encode:
    @ TODO: วนลูปผ่าน buffer
    @       LDRB byte จาก address
    @       EOR byte กับ key
    @       STRB byte กลับ
    BX lr
```

### Exercise 3: 64-bit Operations

```asm
@ แบบฝึกหัด 3: คำนวณ 64-bit square
@ Input: r0 = value (32-bit)
@ Output: r1:r0 = value^2 (64-bit)
@ ใช้ UMULL

square64:
    @ TODO: use UMULL
    BX lr
```

### Exercise 4: Conditional Sorting

```asm
@ แบบฝึกหัด 4: เรียงลำดับ array 3 ตัว
@ Input: r0, r1, r2 = 3 ค่า
@ Output: r0 <= r1 <= r2

sort3:
    @ TODO: ใช้ CMP และ conditional MOV เรียง r0, r1, r2
    @ ไม่ต้องใช้ branch
    BX lr
```

**เฉลย Exercise 4:**

```asm
sort3:
    @ Sort r0, r1
    CMP r0, r1
    MOVGT r3, r0
    MOVGT r0, r1
    MOVGT r1, r3
    
    @ Sort r1, r2
    CMP r1, r2
    MOVGT r3, r1
    MOVGT r1, r2
    MOVGT r2, r3
    
    @ Sort r0, r1 again (bubble sort pass)
    CMP r0, r1
    MOVGT r3, r0
    MOVGT r0, r1
    MOVGT r1, r3
    
    BX lr
```

### Exercise 5: CPSR Flags Practice

```asm
@ แบบฝึกหัด 5: ทำนาย flags
@ สำหรับแต่ละ instruction ทำนาย N, Z, C, V flags

@ Test 1: 
MOV r0, #0xFF
MOV r1, #0xFF
ADDS r2, r0, r1
@ N=? Z=? C=? V=?
@ เฉลย: r2 = 0x1FE
@       N=1 (bit 31 of result = 1 ถ้า sign extend)
@       หมายเหตุ: 0x1FE = 0b111111110
@       ใน 32-bit: r2 = 510, N=0, Z=0, C=0, V=0

@ Test 2:
MOV r3, #0x7FFFFFFF  @ INT_MAX signed
MOV r4, #1
ADDS r5, r3, r4
@ N=? Z=? C=? V=?
@ เฉลย: N=1, Z=0, C=0, V=1 (signed overflow!)
```

---

## 17. Reference Sheet

### 17.1 Data Processing Instruction Summary

```
ARM32 Data Processing Instructions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Instruction  Format                    Description
─────────────────────────────────────────────────
MOV          MOV{cond}{S} Rd, Op2      Rd = Op2
MVN          MVN{cond}{S} Rd, Op2      Rd = ~Op2
ADD          ADD{cond}{S} Rd, Rn, Op2  Rd = Rn + Op2
ADC          ADC{cond}{S} Rd, Rn, Op2  Rd = Rn + Op2 + C
SUB          SUB{cond}{S} Rd, Rn, Op2  Rd = Rn - Op2
SBC          SBC{cond}{S} Rd, Rn, Op2  Rd = Rn - Op2 - !C
RSB          RSB{cond}{S} Rd, Rn, Op2  Rd = Op2 - Rn
RSC          RSC{cond}{S} Rd, Rn, Op2  Rd = Op2 - Rn - !C
AND          AND{cond}{S} Rd, Rn, Op2  Rd = Rn AND Op2
ORR          ORR{cond}{S} Rd, Rn, Op2  Rd = Rn OR Op2
EOR          EOR{cond}{S} Rd, Rn, Op2  Rd = Rn XOR Op2
BIC          BIC{cond}{S} Rd, Rn, Op2  Rd = Rn AND ~Op2
TST          TST{cond} Rn, Op2         Rn AND Op2, set flags
TEQ          TEQ{cond} Rn, Op2         Rn XOR Op2, set flags
CMP          CMP{cond} Rn, Op2         Rn - Op2, set flags
CMN          CMN{cond} Rn, Op2         Rn + Op2, set flags
MUL          MUL{cond}{S} Rd, Rm, Rs   Rd = Rm * Rs
MLA          MLA{cond}{S} Rd, Rm, Rs, Rn Rd = Rm * Rs + Rn
UMULL        UMULL{cond}{S} RdL, RdH, Rm, Rs RdH:RdL = Rm * Rs (u)
UMLAL        UMLAL{cond}{S} RdL, RdH, Rm, Rs RdH:RdL += Rm * Rs (u)
SMULL        SMULL{cond}{S} RdL, RdH, Rm, Rs RdH:RdL = Rm * Rs (s)
SMLAL        SMLAL{cond}{S} RdL, RdH, Rm, Rs RdH:RdL += Rm * Rs (s)
CLZ          CLZ{cond} Rd, Rm           Rd = count_leading_zeros(Rm)
```

### 17.2 Condition Codes Table

```
Code  Flags             Meaning (Signed)    Meaning (Unsigned)
EQ    Z=1               Equal               Equal
NE    Z=0               Not Equal           Not Equal
CS    C=1               -                   Carry/Unsigned >=
CC    C=0               -                   No Carry/Unsigned <
MI    N=1               Negative            -
PL    N=0               Positive/Zero       -
VS    V=1               Overflow            -
VC    V=0               No Overflow         -
HI    C=1, Z=0          -                   Unsigned >
LS    C=0 or Z=1        -                   Unsigned <=
GE    N==V              Signed >=           -
LT    N!=V              Signed <            -
GT    Z=0, N==V         Signed >            -
LE    Z=1 or N!=V       Signed <=           -
AL    (any)             Always              Always
```

### 17.3 Shift Types

```
Type  Assembly    Description
LSL   LSL #n      Logical Shift Left: value << n
LSR   LSR #n      Logical Shift Right: value >> n (zero fill)
ASR   ASR #n      Arithmetic Shift Right: value >> n (sign fill)
ROR   ROR #n      Rotate Right: rotate n bits right
RRX   RRX         Rotate Right Extended: rotate 1 bit through Carry
```

---

## 18. สรุป (Summary)

บทนี้ครอบคลุม ARM Data Processing Instructions ที่สำคัญ:

1. **MOV/MVN**: เคลื่อนย้ายข้อมูลและ bitwise NOT
2. **ADD/ADDS/ADC**: การบวก รวมถึง 64-bit addition
3. **SUB/SUBS/SBC/RSB/RSC**: การลบรูปแบบต่างๆ
4. **AND/ORR/EOR/BIC**: Logical operations สำหรับ bit manipulation
5. **TST/TEQ**: ทดสอบ bits โดยไม่เปลี่ยนค่า
6. **CMP/CMN**: เปรียบเทียบและ set flags
7. **MUL/MLA**: การคูณและ multiply-accumulate
8. **UMULL/UMLAL/SMULL/SMLAL**: การคูณ 64-bit
9. **CLZ**: นับ leading zeros
10. **Conditional Execution**: ทำให้ code มีประสิทธิภาพสูง
11. **S Suffix**: ควบคุมการอัพเดต CPSR flags
12. **Barrel Shifter**: การ shift ใน instruction

**จุดเด่นของ ARM:**
- เกือบทุก instruction รองรับ conditional execution
- Barrel shifter ทำให้ shift ได้ใน instruction เดียว
- S suffix ให้ควบคุม flag update ได้อย่างละเอียด

---

## References

- ARM Architecture Reference Manual ARMv7-A and ARMv7-R edition
- ARM Cortex-A Series Programmer's Guide
- ARM Architecture Reference Manual for A-profile architecture (AArch64)
- GNU Assembler (GAS) ARM documentation

---

*Part 032 - ARM Data Processing Instructions | Assembly Programming Course*
