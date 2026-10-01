# Part 034: ARM Branch และ Control Flow

## บทนำ (Introduction)

ในการเขียนโปรแกรม Assembly สำหรับ ARM หนึ่งในทักษะที่สำคัญที่สุดคือการควบคุมการไหลของโปรแกรม (Control Flow) ซึ่งประกอบด้วยการกระโดด (Branch), การเรียกฟังก์ชัน (Function Call), และการสร้างโครงสร้างควบคุมต่างๆ เช่น if/else, while, for, switch/case

ARM มีชุดคำสั่ง Branch ที่หลากหลายและทรงพลัง รองรับทั้ง ARM mode (32-bit instructions) และ Thumb mode (16/32-bit instructions) ซึ่งในบทนี้เราจะศึกษาทั้งหมดนี้อย่างละเอียด

---

## 1. คำสั่ง B - Branch พื้นฐาน

### 1.1 B (Branch) ใน ARM32

```
ไวยากรณ์: B{cond} label
ระยะทาง: ±32MB (26-bit offset)
```

คำสั่ง `B` เป็นคำสั่งกระโดดพื้นฐานที่สุด ทำงานโดยการเขียนค่าลงใน Program Counter (PC)

```asm
@ ============================================================
@ ไฟล์: branch_basic.s
@ คอมไพล์: as -o branch_basic.o branch_basic.s
@           ld -o branch_basic branch_basic.o
@ วัตถุประสงค์: แสดงการใช้งานคำสั่ง B พื้นฐาน
@ ============================================================

    .section .text
    .global _start

_start:
    @ กำหนดค่าตัวแปร
    MOV r0, #10         @ r0 = 10 (counter)
    MOV r1, #0          @ r1 = 0 (sum)

loop_start:
    @ ตรวจสอบเงื่อนไข: ถ้า r0 <= 0 ให้ออกจาก loop
    CMP r0, #0          @ เปรียบเทียบ r0 กับ 0
    BLE loop_end        @ ถ้า r0 <= 0 กระโดดไป loop_end

    @ ทำงานใน loop
    ADD r1, r1, r0      @ sum += counter
    SUB r0, r0, #1      @ counter--

    B loop_start        @ กระโดดกลับต้น loop

loop_end:
    @ r1 = 55 (1+2+3+...+10)
    @ จบโปรแกรม
    MOV r7, #1          @ syscall: exit
    SWI #0              @ เรียก system call
```

### 1.2 B ใน AArch64

```asm
// ============================================================
// ไฟล์: branch_basic64.s
// คอมไพล์: as -o branch_basic64.o branch_basic64.s
//           ld -o branch_basic64 branch_basic64.o
// วัตถุประสงค์: Branch พื้นฐานใน AArch64
// ============================================================

    .section .text
    .global _start

_start:
    MOV x0, #10         // x0 = 10 (counter)
    MOV x1, #0          // x1 = 0 (sum)

loop_start:
    CMP x0, #0          // เปรียบเทียบ counter กับ 0
    BLE loop_end        // ถ้า counter <= 0 ออกจาก loop

    ADD x1, x1, x0      // sum += counter
    SUB x0, x0, #1      // counter--

    B loop_start        // กระโดดกลับต้น loop

loop_end:
    // ออกจากโปรแกรม: exit(sum)
    MOV x0, x1          // return value = sum (55)
    MOV x8, #93         // syscall: exit
    SVC #0
```

---

## 2. คำสั่ง BL - Branch with Link (Function Call)

### 2.1 หลักการทำงาน BL

`BL` คือคำสั่งเรียกฟังก์ชัน (Function Call) ที่:
1. เก็บที่อยู่ถัดไป (return address) ลงใน Link Register (LR / r14)
2. กระโดดไปยัง label ที่ระบุ

```
BL label  →  LR = PC + 4, PC = label
```

### 2.2 ตัวอย่าง BL ใน ARM32

```asm
@ ============================================================
@ ไฟล์: bl_function.s
@ วัตถุประสงค์: แสดงการเรียกและกลับจากฟังก์ชันด้วย BL/BX LR
@ ============================================================

    .section .text
    .global _start

@ ============================
@ ฟังก์ชัน: add_numbers
@ Input:  r0 = a, r1 = b
@ Output: r0 = a + b
@ ============================
add_numbers:
    ADD r0, r0, r1      @ r0 = a + b
    BX lr               @ กลับไปยัง caller (return)

@ ============================
@ ฟังก์ชัน: multiply
@ Input:  r0 = a, r1 = b
@ Output: r0 = a * b
@ ============================
multiply:
    MUL r0, r0, r1      @ r0 = a * b
    BX lr               @ return

@ ============================
@ ฟังก์ชัน: calculate
@ คำนวณ (a + b) * c
@ Input:  r0 = a, r1 = b, r2 = c
@ Output: r0 = (a + b) * c
@ ============================
calculate:
    @ บันทึก LR และ r2 ลง Stack (เพราะจะเรียก BL ซ้อนกัน)
    PUSH {r2, lr}

    @ เรียก add_numbers(a, b)
    BL add_numbers      @ r0 = a + b
                        @ ค่า r2 (c) ถูกบันทึกใน stack

    @ ดึงค่า c กลับมา
    POP {r1}            @ r1 = c (จาก stack)
    @ แต่ LR ยังอยู่ใน stack

    @ เรียก multiply(r0, c)
    BL multiply         @ r0 = (a + b) * c

    @ กู้คืน LR และ return
    POP {pc}            @ POP pc = return (เทียบเท่า BX lr)

_start:
    @ เรียกใช้งาน calculate(3, 4, 5) = (3+4)*5 = 35
    MOV r0, #3
    MOV r1, #4
    MOV r2, #5
    BL calculate        @ r0 = 35

    @ ออกโปรแกรมด้วยค่า exit code = 35
    MOV r7, #1
    SWI #0
```

### 2.3 ตัวอย่าง BL ใน AArch64

```asm
// ============================================================
// ไฟล์: bl_function64.s
// วัตถุประสงค์: Function call ใน AArch64
// ============================================================

    .section .text
    .global _start

// ============================
// ฟังก์ชัน: add_numbers
// Input:  x0 = a, x1 = b
// Output: x0 = a + b
// ============================
add_numbers:
    ADD x0, x0, x1      // x0 = a + b
    RET                 // กลับไป caller (เทียบเท่า BX lr)

// ============================
// ฟังก์ชัน: multiply
// Input:  x0 = a, x1 = b
// Output: x0 = a * b
// ============================
multiply:
    MUL x0, x0, x1      // x0 = a * b
    RET                 // return

// ============================
// ฟังก์ชัน: calculate
// คำนวณ (a + b) * c
// Input:  x0 = a, x1 = b, x2 = c
// Output: x0 = (a + b) * c
// ============================
calculate:
    // บันทึก x29 (frame pointer), x30 (link register), x2 (c)
    STP x29, x30, [sp, #-32]!   // บันทึก fp และ lr ลง stack
    STR x2, [sp, #16]            // บันทึก c

    MOV x29, sp                  // อัพเดท frame pointer

    // เรียก add_numbers(a, b)
    BL add_numbers               // x0 = a + b

    // ดึงค่า c กลับมา
    LDR x1, [sp, #16]            // x1 = c

    // เรียก multiply(a+b, c)
    BL multiply                  // x0 = (a+b) * c

    // กู้คืนและ return
    LDP x29, x30, [sp], #32     // กู้คืน fp และ lr
    RET

_start:
    // calculate(3, 4, 5) = 35
    MOV x0, #3
    MOV x1, #4
    MOV x2, #5
    BL calculate                 // x0 = 35

    MOV x8, #93                  // syscall: exit
    SVC #0
```

---

## 3. คำสั่ง BX - Branch and Exchange

### 3.1 หลักการทำงาน BX

`BX` (Branch and Exchange) ทำ 2 อย่างพร้อมกัน:
1. กระโดดไปยังที่อยู่ใน register ที่ระบุ
2. สลับระหว่าง ARM mode และ Thumb mode ตาม bit 0 ของที่อยู่

```
ถ้า bit 0 = 0 → ARM mode
ถ้า bit 0 = 1 → Thumb mode
```

```asm
@ ============================================================
@ ไฟล์: bx_exchange.s
@ วัตถุประสงค์: แสดงการใช้งาน BX
@ ============================================================

    .section .text
    .global _start

@ ฟังก์ชัน ARM mode ธรรมดา
arm_function:
    @ ทำงาน...
    MOV r0, #42
    BX lr               @ return ไปยัง caller (ไม่สลับ mode)

@ การใช้ BX สำหรับ indirect branch
indirect_jump:
    @ r0 มีที่อยู่ปลายทาง
    BX r0               @ กระโดดไปยังที่อยู่ใน r0

@ การใช้ BX สำหรับ computed goto (jump table)
jump_table_example:
    @ r0 = index (0, 1, 2, ...)
    ADR r1, jump_table          @ r1 = ที่อยู่ของ jump table
    LDR pc, [r1, r0, LSL #2]   @ pc = jump_table[index]
    @ หรือใช้ BX กับ register:
    @ LDR r2, [r1, r0, LSL #2]
    @ BX r2

jump_table:
    .word case_0
    .word case_1
    .word case_2

case_0:
    MOV r0, #0
    BX lr

case_1:
    MOV r0, #10
    BX lr

case_2:
    MOV r0, #20
    BX lr

_start:
    BL arm_function     @ เรียกฟังก์ชัน

    MOV r7, #1
    SWI #0
```

---

## 4. คำสั่ง BLX - Branch with Link and Exchange

### 4.1 BLX คืออะไร

`BLX` รวมความสามารถของ `BL` และ `BX`:
1. บันทึก return address ลงใน LR
2. กระโดดและอาจสลับ mode

```asm
@ ============================================================
@ ไฟล์: blx_example.s
@ วัตถุประสงค์: แสดงการใช้งาน BLX
@ ============================================================

    .section .text
    .global _start
    .thumb_func                 @ บอกว่า function นี้เป็น Thumb

@ Thumb function
    .thumb
thumb_add:
    ADD r0, r0, r1
    BX lr                       @ return

    .arm                        @ กลับมา ARM mode
_start:
    MOV r0, #5
    MOV r1, #3

    @ เรียก Thumb function จาก ARM mode
    @ ใช้ ADR เพื่อได้ที่อยู่ด้วย bit 0 = 1 (Thumb)
    ADR r2, thumb_add + 1       @ +1 เพื่อ set Thumb bit
    BLX r2                      @ เรียก Thumb function, r0 = 8

    MOV r7, #1
    SWI #0
```

---

## 5. Conditional Branches

### 5.1 รายการ Condition Codes

ARM มี Condition Codes ที่ครอบคลุม:

| Suffix | Meaning | Flags |
|--------|---------|-------|
| EQ | Equal | Z=1 |
| NE | Not Equal | Z=0 |
| CS/HS | Carry Set / Higher or Same | C=1 |
| CC/LO | Carry Clear / Lower | C=0 |
| MI | Minus / Negative | N=1 |
| PL | Plus / Positive | N=0 |
| VS | Overflow | V=1 |
| VC | No Overflow | V=0 |
| HI | Higher (unsigned) | C=1 AND Z=0 |
| LS | Lower or Same (unsigned) | C=0 OR Z=1 |
| GE | Greater or Equal (signed) | N=V |
| LT | Less Than (signed) | N≠V |
| GT | Greater Than (signed) | Z=0 AND N=V |
| LE | Less or Equal (signed) | Z=1 OR N≠V |
| AL | Always (default) | - |

### 5.2 ตัวอย่าง Conditional Branches

```asm
@ ============================================================
@ ไฟล์: conditional_branches.s
@ วัตถุประสงค์: แสดง Conditional Branch ทุกแบบ
@ ============================================================

    .section .text
    .global _start

@ ============================
@ BEQ - Branch if Equal
@ ============================
test_beq:
    MOV r0, #5
    CMP r0, #5          @ 5 == 5 → Z flag = 1
    BEQ equal           @ กระโดดถ้า Z=1
    MOV r1, #0          @ ไม่ถึงบรรทัดนี้
    B done_beq
equal:
    MOV r1, #1          @ r1 = 1 แสดงว่าเท่ากัน
done_beq:
    BX lr

@ ============================
@ BNE - Branch if Not Equal
@ ============================
test_bne:
    MOV r0, #5
    CMP r0, #3          @ 5 != 3 → Z flag = 0
    BNE not_equal       @ กระโดดถ้า Z=0
    MOV r1, #0
    B done_bne
not_equal:
    MOV r1, #1          @ r1 = 1 แสดงว่าไม่เท่ากัน
done_bne:
    BX lr

@ ============================
@ BLT / BLE / BGT / BGE - Signed Comparison
@ ============================
test_signed:
    @ ทดสอบ signed comparison
    MOV r0, #-5         @ r0 = -5
    MOV r1, #3          @ r1 = 3

    CMP r0, r1          @ -5 vs 3
    BLT r0_less         @ -5 < 3 → กระโดด
    B done_signed
r0_less:
    MOV r2, #1          @ r2 = 1 แสดงว่า r0 < r1
done_signed:
    BX lr

@ ============================
@ BGT - Branch if Greater Than (signed)
@ ============================
test_bgt:
    MOV r0, #10
    CMP r0, #5          @ 10 > 5
    BGT r0_greater      @ กระโดดถ้า r0 > 5
    B done_bgt
r0_greater:
    MOV r1, #1          @ r1 = 1
done_bgt:
    BX lr

@ ============================
@ BCS/BCC - Unsigned (Carry) Comparison
@ ============================
test_unsigned:
    @ unsigned comparison: 200 vs 100
    MOV r0, #200
    CMP r0, #100        @ 200 > 100 (unsigned)
    BHI r0_higher       @ Branch if Higher (unsigned >)
    B done_unsigned
r0_higher:
    MOV r1, #1          @ r1 = 1 แสดงว่า r0 > 100 (unsigned)
done_unsigned:
    BX lr

_start:
    BL test_beq
    BL test_bne
    BL test_signed
    BL test_bgt
    BL test_unsigned

    MOV r7, #1
    SWI #0
```

---

## 6. CBZ/CBNZ - Compare and Branch (Thumb-2)

### 6.1 หลักการทำงาน

`CBZ` (Compare and Branch if Zero) และ `CBNZ` (Compare and Branch if Non-Zero) เป็นคำสั่ง Thumb-2 ที่รวม CMP + B เข้าด้วยกันในคำสั่งเดียว ช่วยลดขนาด code

```
CBZ  Rn, label  → ถ้า Rn == 0 กระโดดไป label
CBNZ Rn, label  → ถ้า Rn != 0 กระโดดไป label
```

**ข้อจำกัด**: กระโดดได้เฉพาะ forward (ไปข้างหน้า) เท่านั้น ระยะสูงสุด 126 bytes

```asm
@ ============================================================
@ ไฟล์: cbz_cbnz.s
@ คอมไพล์: as -mthumb -o cbz_cbnz.o cbz_cbnz.s
@ วัตถุประสงค์: แสดงการใช้งาน CBZ/CBNZ
@ ============================================================

    .section .text
    .global _start
    .syntax unified
    .thumb

@ ============================
@ ตัวอย่าง 1: CBZ ใช้ตรวจสอบ NULL pointer
@ ============================
@ ฟังก์ชัน: process_pointer
@ Input:  r0 = pointer (อาจเป็น NULL)
@ Output: r0 = 0 ถ้า NULL, r0 = *pointer ถ้าไม่ NULL
process_pointer:
    CBZ r0, null_pointer    @ ถ้า r0 == 0 (NULL) กระโดดไป null_pointer
    LDR r0, [r0]            @ dereference pointer
    BX lr
null_pointer:
    MOV r0, #0              @ return 0
    BX lr

@ ============================
@ ตัวอย่าง 2: CBNZ ใช้ใน loop
@ ============================
@ ฟังก์ชัน: count_nonzero
@ นับจำนวน elements ที่ไม่ใช่ 0 ใน array
@ Input:  r0 = pointer to array, r1 = size
@ Output: r0 = count ของ non-zero elements
count_nonzero:
    PUSH {r4, lr}
    MOV r4, #0              @ count = 0

count_loop:
    CBZ r1, count_done      @ ถ้า size == 0 หยุด

    LDR r2, [r0], #4        @ r2 = *ptr++
    CBNZ r2, increment      @ ถ้า element != 0 เพิ่ม count
    B next_element
increment:
    ADD r4, r4, #1          @ count++
next_element:
    SUB r1, r1, #1          @ size--
    B count_loop

count_done:
    MOV r0, r4              @ return count
    POP {r4, pc}

@ ============================
@ ตัวอย่าง 3: ใช้ใน string processing
@ ============================
@ ฟังก์ชัน: strlen_arm
@ คำนวณความยาว string
@ Input:  r0 = pointer to string
@ Output: r0 = length
strlen_arm:
    MOV r1, r0              @ r1 = ตำแหน่งเริ่มต้น
strlen_loop:
    LDRB r2, [r1], #1       @ r2 = *r1++
    CBNZ r2, strlen_loop    @ ถ้า r2 != 0 ทำ loop ต่อ
    @ ถ้า r2 == 0 แสดงว่าถึง null terminator
    SUB r1, r1, r0          @ length = current - start
    SUB r0, r1, #1          @ -1 เพราะนับ null terminator ด้วย
    BX lr

.data
test_string:
    .asciz "Hello, ARM!"

_start:
    @ ทดสอบ strlen
    LDR r0, =test_string
    BL strlen_arm           @ r0 = 11

    MOV r7, #1
    SWI #0
```

---

## 7. TBB/TBH - Table Branch (Thumb-2)

### 7.1 หลักการทำงาน

`TBB` (Table Branch Byte) และ `TBH` (Table Branch Halfword) เป็นคำสั่งสำหรับ switch/case ที่มีประสิทธิภาพสูง

```
TBB [Rn, Rm]    → PC += 2 * table[Rm]  (byte table)
TBH [Rn, Rm, LSL #1] → PC += 2 * table[Rm]  (halfword table)
```

```asm
@ ============================================================
@ ไฟล์: tbb_tbh.s
@ คอมไพล์: as -mthumb -march=armv7-a -o tbb_tbh.o tbb_tbh.s
@ วัตถุประสงค์: Table Branch สำหรับ switch/case
@ ============================================================

    .section .text
    .global _start
    .syntax unified
    .thumb

@ ============================
@ Switch/Case ด้วย TBB
@ ============================
@ ฟังก์ชัน: switch_tbb
@ Input:  r0 = value (0-4)
@ Output: r0 = result
switch_tbb:
    CMP r0, #4              @ ตรวจสอบว่า index ไม่เกิน 4
    BHI switch_default      @ ถ้า > 4 ไปที่ default

    TBB [pc, r0]            @ กระโดดด้วย byte table (PC-relative)

branch_table:
    .byte (case0 - branch_table) / 2    @ offset ไป case 0
    .byte (case1 - branch_table) / 2    @ offset ไป case 1
    .byte (case2 - branch_table) / 2    @ offset ไป case 2
    .byte (case3 - branch_table) / 2    @ offset ไป case 3
    .byte (case4 - branch_table) / 2    @ offset ไป case 4
    .align 2                             @ align ให้ตรง 4 bytes

case0:
    MOV r0, #100
    B switch_end
case1:
    MOV r0, #200
    B switch_end
case2:
    MOV r0, #300
    B switch_end
case3:
    MOV r0, #400
    B switch_end
case4:
    MOV r0, #500
    B switch_end
switch_default:
    MOV r0, #-1
switch_end:
    BX lr

@ ============================
@ Switch/Case ด้วย TBH (สำหรับ offset ใหญ่ขึ้น)
@ ============================
switch_tbh:
    CMP r0, #4
    BHI switch_default2

    TBH [pc, r0, LSL #1]   @ halfword table (offset ใหญ่ได้ถึง 64KB)

branch_table_h:
    .hword (case_a - branch_table_h) / 2
    .hword (case_b - branch_table_h) / 2
    .hword (case_c - branch_table_h) / 2
    .hword (case_d - branch_table_h) / 2
    .hword (case_e - branch_table_h) / 2
    .align 2

case_a:
    MOV r0, #1000
    B switch_end2
case_b:
    MOV r0, #2000
    B switch_end2
case_c:
    MOV r0, #3000
    B switch_end2
case_d:
    MOV r0, #4000
    B switch_end2
case_e:
    MOV r0, #5000
    B switch_end2
switch_default2:
    MOV r0, #-1
switch_end2:
    BX lr

_start:
    MOV r0, #2
    BL switch_tbb           @ r0 = 300

    MOV r0, #3
    BL switch_tbh           @ r0 = 4000

    MOV r7, #1
    SWI #0
```

---

## 8. PC-Relative Addressing

### 8.1 หลักการ

ใน ARM, `PC` ชี้ไปที่ instruction ปัจจุบัน + 8 (ARM mode) หรือ +4 (Thumb mode)

```asm
@ ============================================================
@ ไฟล์: pc_relative.s
@ วัตถุประสงค์: PC-relative addressing
@ ============================================================

    .section .text
    .global _start

@ ============================
@ การใช้ ADR - Address of label (PC-relative)
@ ============================
_start:
    @ ADR ใช้สำหรับ address ที่อยู่ใกล้ (±1KB ใน Thumb, ±4KB ใน ARM)
    ADR r0, my_string       @ r0 = ที่อยู่ของ my_string (PC-relative)
    ADR r1, my_data         @ r1 = ที่อยู่ของ my_data

    @ ADRL ใช้สำหรับ address ที่อยู่ไกลกว่า (±256MB ใน ARM)
    ADRL r2, far_label      @ ใช้ 2 instructions

    @ LDR PC-relative (ใช้ .word ใน text section)
    LDR r3, =my_constant    @ r3 = ค่าคงที่ (ผ่าน literal pool)

    MOV r7, #1
    SWI #0

my_string:
    .asciz "Hello, World!"

my_data:
    .word 12345

my_constant = 0xDEADBEEF

far_label:
    .word 0

@ ============================
@ PC-relative ใน AArch64
@ ============================
@ ใน AArch64 ใช้ ADRP (Page) + ADD สำหรับ addresses ไกล

example_aarch64:
    // ADRP โหลด 4KB-aligned page address ของ label
    ADRP x0, my_data_page       // x0 = page address
    ADD  x0, x0, :lo12:my_data_page  // เพิ่ม offset ภายใน page

    RET

    .section .data
my_data_page:
    .word 99999
```

---

## 9. ARM IT Block (If-Then)

### 9.1 หลักการทำงาน IT

IT (If-Then) เป็นคำสั่ง Thumb-2 พิเศษที่ทำให้สูงสุด 4 คำสั่งถัดไปทำงานตามเงื่อนไข โดยไม่ต้องกระโดด ช่วยให้ pipeline ทำงานได้ดีขึ้น

```
IT{x{y{z}}} cond
- T = Then (ทำถ้าเงื่อนไขเป็นจริง)
- E = Else (ทำถ้าเงื่อนไขเป็นเท็จ)
```

```asm
@ ============================================================
@ ไฟล์: it_block.s
@ คอมไพล์: as -mthumb -march=armv7-a -o it_block.o it_block.s
@ วัตถุประสงค์: ARM IT Block
@ ============================================================

    .section .text
    .global _start
    .syntax unified
    .thumb

@ ============================
@ ตัวอย่าง 1: IT EQ - คำสั่งเดียว
@ ============================
abs_value:
    @ คำนวณ absolute value ของ r0
    CMP r0, #0
    IT MI                   @ ถ้า Minus (r0 < 0)
    NEGMI r0, r0            @ r0 = -r0 (ทำเฉพาะตอน r0 < 0)
    BX lr

@ ============================
@ ตัวอย่าง 2: ITE (If-Then-Else)
@ ============================
max_value:
    @ หาค่าที่มากกว่า: r0 = max(r0, r1)
    CMP r0, r1
    ITE GT                  @ ถ้า r0 > r1
    MOVGT r0, r0            @   Then: r0 = r0 (ไม่เปลี่ยน)
    MOVLE r0, r1            @   Else: r0 = r1
    BX lr

@ ============================
@ ตัวอย่าง 3: ITEE (If-Then-Else-Else)
@ ============================
classify_number:
    @ แยกประเภทตัวเลข: r0 = sign (-1, 0, +1)
    CMP r0, #0
    ITEE EQ                 @ ถ้า r0 == 0
    MOVEQ r0, #0            @   Then: return 0
    MOVLT r0, #-1           @   Else (r0 < 0): return -1
    MOVGT r0, #1            @   Else (r0 > 0): return +1
    BX lr

@ ============================
@ ตัวอย่าง 4: ITTEE (4 คำสั่ง)
@ ============================
clamp_value:
    @ clamp r0 ให้อยู่ระหว่าง [r1, r2]
    @ ถ้า r0 < r1 → r0 = r1
    @ ถ้า r0 > r2 → r0 = r2
    CMP r0, r1
    ITE LT
    MOVLT r0, r1            @ r0 = min_val ถ้า r0 < min
    BX lr                   @ (ส่วน ITE ที่ 2)
    @ ตัวอย่าง clamp ต้องใช้หลาย IT block

clamp_full:
    @ clamp: min ใน r1, max ใน r2
    CMP r0, r1
    IT LT
    MOVLT r0, r1            @ ถ้า r0 < r1, r0 = r1
    CMP r0, r2
    IT GT
    MOVGT r0, r2            @ ถ้า r0 > r2, r0 = r2
    BX lr

_start:
    @ ทดสอบ abs_value
    MOV r0, #-7
    BL abs_value            @ r0 = 7

    @ ทดสอบ max_value
    MOV r0, #5
    MOV r1, #9
    BL max_value            @ r0 = 9

    @ ทดสอบ classify_number
    MOV r0, #-3
    BL classify_number      @ r0 = -1

    @ ทดสอบ clamp_full
    MOV r0, #15
    MOV r1, #0
    MOV r2, #10
    BL clamp_full           @ r0 = 10

    MOV r7, #1
    SWI #0
```

---

## 10. การ Implement if/else/while/for

### 10.1 if/else

```asm
@ ============================================================
@ ไฟล์: control_structures.s
@ วัตถุประสงค์: โครงสร้าง Control Flow พื้นฐาน
@ ============================================================

    .section .text
    .global _start

@ ============================
@ if (a > b) { result = a; } else { result = b; }
@ max(a, b) → ใส่ใน r0
@ Input: r0 = a, r1 = b
@ ============================
if_else_max:
    CMP r0, r1          @ เปรียบเทียบ a กับ b
    BGT if_true         @ ถ้า a > b ไป if_true
    @ else branch:
    MOV r0, r1          @ result = b
    B if_end
if_true:
    @ ไม่ต้องทำอะไร r0 = a อยู่แล้ว
    @ MOV r0, r0
if_end:
    BX lr

@ ============================
@ while (counter > 0) { sum += counter; counter--; }
@ คำนวณ sum = 1+2+...+n
@ Input: r0 = n
@ Output: r0 = sum
@ ============================
while_sum:
    MOV r1, #0          @ sum = 0

while_condition:
    CMP r0, #0          @ ตรวจสอบ counter > 0
    BLE while_done      @ ถ้า counter <= 0 ออก

    ADD r1, r1, r0      @ sum += counter
    SUB r0, r0, #1      @ counter--

    B while_condition   @ วนซ้ำ

while_done:
    MOV r0, r1          @ return sum
    BX lr

@ ============================
@ for (i = 0; i < n; i++) { sum += i; }
@ Input: r0 = n
@ Output: r0 = sum (0+1+...+(n-1))
@ ============================
for_sum:
    MOV r1, #0          @ i = 0
    MOV r2, #0          @ sum = 0

for_init:
    @ เงื่อนไขอยู่ที่นี่

for_condition:
    CMP r1, r0          @ i < n ?
    BGE for_done        @ ถ้า i >= n ออก

for_body:
    ADD r2, r2, r1      @ sum += i

for_update:
    ADD r1, r1, #1      @ i++
    B for_condition     @ วนซ้ำ

for_done:
    MOV r0, r2          @ return sum
    BX lr

@ ============================
@ do-while loop
@ do { sum += i; i++; } while (i <= n);
@ Input: r0 = n
@ Output: r0 = sum (1+2+...+n)
@ ============================
do_while_sum:
    MOV r1, #1          @ i = 1
    MOV r2, #0          @ sum = 0

do_while_body:
    ADD r2, r2, r1      @ sum += i
    ADD r1, r1, #1      @ i++

do_while_condition:
    CMP r1, r0          @ i <= n ?
    BLE do_while_body   @ ถ้ายัง ทำซ้ำ

do_while_done:
    MOV r0, r2          @ return sum
    BX lr

_start:
    @ ทดสอบ if/else max
    MOV r0, #5
    MOV r1, #9
    BL if_else_max      @ r0 = 9

    @ ทดสอบ while
    MOV r0, #10
    BL while_sum        @ r0 = 55

    @ ทดสอบ for
    MOV r0, #10
    BL for_sum          @ r0 = 45 (0+1+...+9)

    @ ทดสอบ do-while
    MOV r0, #10
    BL do_while_sum     @ r0 = 55 (1+2+...+10)

    MOV r7, #1
    SWI #0
```

---

## 11. Switch/Case ด้วย Jump Table

### 11.1 Implementation ด้วย Jump Table ใน ARM32

```asm
@ ============================================================
@ ไฟล์: switch_case.s
@ วัตถุประสงค์: Switch/Case ด้วย Jump Table
@ ============================================================

    .section .text
    .global _start

@ ============================
@ switch (day) { case 0: ... case 6: ... }
@ Input: r0 = day (0-6)
@ Output: r0 = number of work hours
@ ============================
work_hours:
    @ ตรวจสอบ range
    CMP r0, #6
    BHI invalid_day         @ ถ้า day > 6 ไป invalid

    @ คำนวณ jump address
    ADR r1, day_table       @ r1 = ที่อยู่ jump table
    LDR pc, [r1, r0, LSL #2]  @ pc = day_table[day]
                               @ LSL #2 = คูณ 4 (ขนาด word)

day_table:
    .word sunday            @ index 0
    .word monday            @ index 1
    .word tuesday           @ index 2
    .word wednesday         @ index 3
    .word thursday          @ index 4
    .word friday            @ index 5
    .word saturday          @ index 6

sunday:
    MOV r0, #0              @ วันอาทิตย์ = 0 ชั่วโมง
    B work_end

monday:
tuesday:
wednesday:
thursday:
    MOV r0, #8              @ วันจันทร์-พฤหัส = 8 ชั่วโมง
    B work_end

friday:
    MOV r0, #6              @ วันศุกร์ = 6 ชั่วโมง
    B work_end

saturday:
    MOV r0, #4              @ วันเสาร์ = 4 ชั่วโมง
    B work_end

invalid_day:
    MOV r0, #-1
work_end:
    BX lr

@ ============================
@ Switch/Case แบบ String (เปรียบเทียบ month name)
@ ============================
@ ฟังก์ชัน: days_in_month
@ Input: r0 = month (1-12)
@ Output: r0 = days in month (ไม่รวม leap year)
days_in_month:
    CMP r0, #0
    BEQ invalid_month
    CMP r0, #12
    BGT invalid_month

    ADR r1, month_table
    LDR pc, [r1, r0, LSL #2]   @ แต่ต้องลบ 1 เพราะ month เริ่มที่ 1
    @ แก้ไข:
    @ SUB r0, r0, #1
    @ LDR pc, [r1, r0, LSL #2]

month_table:
    .word invalid_month     @ 0 (ไม่ใช้)
    .word jan               @ 1
    .word feb               @ 2
    .word mar               @ 3
    .word apr               @ 4
    .word may               @ 5
    .word jun               @ 6
    .word jul               @ 7
    .word aug               @ 8
    .word sep               @ 9
    .word oct               @ 10
    .word nov               @ 11
    .word dec               @ 12

jan: mar: may: jul: aug: oct: dec:
    MOV r0, #31
    BX lr

apr: jun: sep: nov:
    MOV r0, #30
    BX lr

feb:
    MOV r0, #28             @ ไม่นับ leap year
    BX lr

invalid_month:
    MOV r0, #-1
    BX lr

_start:
    @ ทดสอบ work_hours
    MOV r0, #1              @ Monday
    BL work_hours           @ r0 = 8

    MOV r0, #6              @ Saturday
    BL work_hours           @ r0 = 4

    @ ทดสอบ days_in_month (แก้ไข: ต้องลบ 1 ก่อน)
    MOV r0, #3              @ March
    SUB r0, r0, #1
    ADR r1, month_table
    LDR pc, [r1, r0, LSL #2]
    @ *** ตัวอย่างนี้ต้องแก้ไข logic ในฟังก์ชัน

    MOV r7, #1
    SWI #0
```

---

## 12. Computed GOTO

### 12.1 หลักการ Computed GOTO

```asm
@ ============================================================
@ ไฟล์: computed_goto.s
@ วัตถุประสงค์: Computed GOTO (indirect jump)
@ ============================================================

    .section .text
    .global _start

@ ============================
@ Computed GOTO ใช้ array of function pointers
@ ============================
@ ฟังก์ชัน: dispatch
@ Input: r0 = opcode (0-3), r1 = operand1, r2 = operand2
@ Output: r0 = result
@ ============================
dispatch:
    PUSH {lr}

    @ ตรวจสอบ range
    CMP r0, #3
    BHI dispatch_error

    @ โหลด jump table
    ADR r3, opcode_table
    LDR r3, [r3, r0, LSL #2]   @ r3 = function pointer

    @ บันทึก operands
    MOV r0, r1
    MOV r1, r2

    @ เรียกผ่าน indirect
    BLX r3                      @ เรียก function ที่ r3 ชี้ไป

    POP {pc}

dispatch_error:
    MOV r0, #-1
    POP {pc}

@ Jump table
opcode_table:
    .word op_add
    .word op_sub
    .word op_mul
    .word op_div

@ Operations
op_add:
    ADD r0, r0, r1
    BX lr

op_sub:
    SUB r0, r0, r1
    BX lr

op_mul:
    MUL r0, r0, r1
    BX lr

op_div:
    @ ARM ไม่มีคำสั่ง DIV ใน ARMv7 (มีใน ARMv8)
    @ ใช้ SDIV ถ้า CPU รองรับ
    SDIV r0, r0, r1
    BX lr

@ ============================
@ Virtual Dispatch (C++ vtable style)
@ ============================
@ Object structure:
@ [0] = vtable pointer
@ [4] = data field
@ vtable:
@ [0] = method_a pointer
@ [4] = method_b pointer

virtual_call_method_a:
    @ r0 = object pointer
    LDR r1, [r0]        @ r1 = vtable pointer
    LDR r2, [r1, #0]    @ r2 = method_a pointer (offset 0)
    BLX r2              @ เรียก method_a
    BX lr

virtual_call_method_b:
    LDR r1, [r0]        @ r1 = vtable pointer
    LDR r2, [r1, #4]    @ r2 = method_b pointer (offset 4)
    BLX r2              @ เรียก method_b
    BX lr

_start:
    @ ทดสอบ dispatch: add(10, 5) = 15
    MOV r0, #0          @ ADD opcode
    MOV r1, #10
    MOV r2, #5
    BL dispatch         @ r0 = 15

    @ ทดสอบ dispatch: mul(3, 7) = 21
    MOV r0, #2          @ MUL opcode
    MOV r1, #3
    MOV r2, #7
    BL dispatch         @ r0 = 21

    MOV r7, #1
    SWI #0
```

---

## 13. Recursive Function ใน ARM

### 13.1 Factorial แบบ Recursive

```asm
@ ============================================================
@ ไฟล์: recursive.s
@ วัตถุประสงค์: Recursive Functions ใน ARM
@ ============================================================

    .section .text
    .global _start

@ ============================
@ factorial(n) = n! = n * factorial(n-1)
@ factorial(0) = 1
@ Input:  r0 = n
@ Output: r0 = n!
@ ============================
factorial:
    @ Base case: n == 0 → return 1
    CMP r0, #0
    BEQ fact_base

    @ Recursive case: เก็บ n และ LR ลง stack
    PUSH {r0, lr}           @ บันทึก n และ return address

    @ เรียก factorial(n-1)
    SUB r0, r0, #1          @ r0 = n-1
    BL factorial            @ recursive call, r0 = (n-1)!

    @ r0 = (n-1)!, กู้คืน n จาก stack
    POP {r1, lr}            @ r1 = n (ที่บันทึกไว้), กู้คืน lr

    @ คำนวณ n * (n-1)!
    MUL r0, r0, r1          @ r0 = n * (n-1)!
    BX lr                   @ return

fact_base:
    MOV r0, #1              @ factorial(0) = 1
    BX lr

@ ============================
@ fibonacci(n) แบบ Recursive
@ fib(0) = 0, fib(1) = 1
@ fib(n) = fib(n-1) + fib(n-2)
@ Input:  r0 = n
@ Output: r0 = fibonacci(n)
@ ============================
fibonacci:
    @ Base cases
    CMP r0, #0
    BEQ fib_zero
    CMP r0, #1
    BEQ fib_one

    @ Recursive case: บันทึก n
    PUSH {r0, lr}

    @ เรียก fibonacci(n-1)
    SUB r0, r0, #1
    BL fibonacci
    @ r0 = fib(n-1)

    @ บันทึก fib(n-1) และกู้คืน n
    POP {r1, lr}            @ r1 = n
    PUSH {r0, lr}           @ บันทึก fib(n-1) และ lr อีกครั้ง

    @ เรียก fibonacci(n-2)
    SUB r0, r1, #2          @ r0 = n-2
    BL fibonacci
    @ r0 = fib(n-2)

    POP {r1, lr}            @ r1 = fib(n-1)
    ADD r0, r0, r1          @ r0 = fib(n-1) + fib(n-2)
    BX lr

fib_zero:
    MOV r0, #0
    BX lr

fib_one:
    MOV r0, #1
    BX lr

@ ============================
@ Tower of Hanoi แบบ Recursive
@ hanoi(n, from, to, aux)
@ r0 = n, r1 = from, r2 = to, r3 = aux
@ ============================
.section .data
hanoi_count: .word 0    @ นับจำนวนการเคลื่อน disk

.section .text
tower_of_hanoi:
    @ Base case: n == 0
    CMP r0, #0
    BEQ hanoi_done

    @ บันทึก arguments
    PUSH {r0, r1, r2, r3, lr}

    @ hanoi(n-1, from, aux, to)
    SUB r0, r0, #1          @ n-1
    MOV r2, r3              @ to → aux position
    @ r1 = from, r3 = aux (ไม่เปลี่ยน)
    BL tower_of_hanoi

    @ นับการเคลื่อน disk n
    LDR r4, =hanoi_count
    LDR r5, [r4]
    ADD r5, r5, #1
    STR r5, [r4]

    @ กู้คืน arguments
    POP {r0, r1, r2, r3, lr}

    @ บันทึกอีกครั้ง
    PUSH {r0, r1, r2, r3, lr}

    @ hanoi(n-1, aux, to, from)
    SUB r0, r0, #1          @ n-1
    MOV r1, r3              @ from = aux
    @ r2 = to, r3 = from
    LDR r3, [sp, #4]        @ r3 = original from
    BL tower_of_hanoi

    POP {r0, r1, r2, r3, lr}

hanoi_done:
    BX lr

_start:
    @ ทดสอบ factorial
    MOV r0, #5
    BL factorial            @ r0 = 120

    @ ทดสอบ fibonacci
    MOV r0, #10
    BL fibonacci            @ r0 = 55

    @ ออกด้วย factorial(5) = 120
    MOV r7, #1
    SWI #0
```

---

## 14. AArch64 Control Flow Examples

### 14.1 AArch64 Branch Instructions

```asm
// ============================================================
// ไฟล์: aarch64_control.s
// คอมไพล์: as -o aarch64_ctrl.o aarch64_control.s
//           ld -o aarch64_ctrl aarch64_ctrl.o
// วัตถุประสงค์: Control Flow ใน AArch64
// ============================================================

    .section .text
    .global _start

// ============================
// CBZ/CBNZ ใน AArch64
// ============================
find_first_nonzero:
    // หา index แรกของ element ที่ไม่ใช่ 0 ใน array
    // x0 = pointer to array, x1 = size
    // Return: x0 = index หรือ -1 ถ้าไม่พบ

    MOV x2, #0              // i = 0

ffnz_loop:
    CMP x2, x1
    BGE ffnz_not_found      // ถ้า i >= size ไม่พบ

    LDR x3, [x0, x2, LSL #3]   // x3 = array[i]
    CBNZ x3, ffnz_found         // ถ้า x3 != 0 พบแล้ว

    ADD x2, x2, #1          // i++
    B ffnz_loop

ffnz_found:
    MOV x0, x2              // return i
    RET

ffnz_not_found:
    MOV x0, #-1             // return -1
    RET

// ============================
// TBZ/TBNZ ใน AArch64 (Test and Branch)
// AArch64 มีคำสั่งพิเศษ TBZ/TBNZ
// ============================
check_flags:
    // TBZ = Test Bit and Branch if Zero
    // TBNZ = Test Bit and Branch if Non-Zero

    // ตรวจสอบ bit 3 ของ x0
    TBZ x0, #3, bit3_zero   // ถ้า bit 3 ของ x0 == 0
    MOV x1, #1              // bit 3 = 1
    B check_done
bit3_zero:
    MOV x1, #0              // bit 3 = 0
check_done:
    RET

// ============================
// Conditional Select ใน AArch64
// CSEL (Conditional Select)
// CSINC, CSINV, CSNEG
// ============================
max_aarch64:
    // หาค่าที่มากกว่า: x0 = max(x0, x1)
    CMP x0, x1
    CSEL x0, x0, x1, GT    // x0 = (x0 > x1) ? x0 : x1
    RET

abs_aarch64:
    // absolute value: x0 = abs(x0)
    CMP x0, #0
    CSNEG x0, x0, x0, PL   // x0 = (x0 >= 0) ? x0 : -x0
    RET

// ============================
// Recursive Fibonacci ใน AArch64
// ============================
fibonacci64:
    CMP x0, #1
    BLS fib64_base          // ถ้า x0 <= 1 ไป base case

    STP x29, x30, [sp, #-32]!   // บันทึก frame pointer และ link register
    MOV x29, sp
    STR x0, [sp, #16]           // บันทึก n

    SUB x0, x0, #1              // x0 = n-1
    BL fibonacci64               // x0 = fib(n-1)

    LDR x1, [sp, #16]           // x1 = n
    STR x0, [sp, #24]           // บันทึก fib(n-1)

    SUB x0, x1, #2              // x0 = n-2
    BL fibonacci64               // x0 = fib(n-2)

    LDR x1, [sp, #24]           // x1 = fib(n-1)
    ADD x0, x0, x1              // x0 = fib(n-1) + fib(n-2)

    LDP x29, x30, [sp], #32     // กู้คืน
    RET

fib64_base:
    // fib(0) = 0, fib(1) = 1 → return x0 ตามเดิม
    RET

// ============================
// Switch/Case ใน AArch64
// ============================
switch_case64:
    // x0 = value (0-4)
    CMP x0, #4
    BHI sw64_default

    ADR x1, sw64_table
    LDR x2, [x1, x0, LSL #3]   // 8 bytes per entry (64-bit)
    BR x2                        // indirect branch

sw64_table:
    .quad sw64_case0
    .quad sw64_case1
    .quad sw64_case2
    .quad sw64_case3
    .quad sw64_case4

sw64_case0:
    MOV x0, #100
    RET
sw64_case1:
    MOV x0, #200
    RET
sw64_case2:
    MOV x0, #300
    RET
sw64_case3:
    MOV x0, #400
    RET
sw64_case4:
    MOV x0, #500
    RET
sw64_default:
    MOV x0, #-1
    RET

_start:
    // ทดสอบ fibonacci
    MOV x0, #10
    BL fibonacci64          // x0 = 55

    // ทดสอบ max
    MOV x0, #15
    MOV x1, #7
    BL max_aarch64          // x0 = 15

    // ทดสอบ switch
    MOV x0, #3
    BL switch_case64        // x0 = 400

    MOV x8, #93             // exit
    SVC #0
```

---

## 15. Advanced: Stack Frame และ Function Prologue/Epilogue

### 15.1 Standard Function Structure ใน ARM32

```asm
@ ============================================================
@ ไฟล์: stack_frame.s
@ วัตถุประสงค์: Stack Frame Management
@ ============================================================

    .section .text
    .global _start

@ Standard ARM32 function ที่มีหลาย local variables
@ void complex_function(int a, int b, int c, int d)
complex_function:
    @ --- PROLOGUE ---
    @ บันทึก callee-saved registers และ lr
    @ r4-r11 คือ callee-saved registers
    PUSH {r4, r5, r6, r7, r8, fp, lr}
    @ จัดสรร local variables บน stack: 5 ints = 20 bytes
    SUB sp, sp, #20
    @ ตั้งค่า frame pointer
    ADD fp, sp, #20

    @ --- FUNCTION BODY ---
    @ บันทึก arguments ลงใน callee-saved registers
    MOV r4, r0          @ local_a = a
    MOV r5, r1          @ local_b = b
    MOV r6, r2          @ local_c = c
    MOV r7, r3          @ local_d = d

    @ Local variables ใน stack
    @ [fp - 4]  = local_x
    @ [fp - 8]  = local_y
    @ [fp - 12] = local_z

    ADD r8, r4, r5      @ local_x = a + b
    STR r8, [fp, #-4]

    MUL r8, r6, r7      @ local_y = c * d
    STR r8, [fp, #-8]

    LDR r0, [fp, #-4]
    LDR r1, [fp, #-8]
    ADD r8, r0, r1      @ local_z = local_x + local_y
    STR r8, [fp, #-12]

    @ Return value
    LDR r0, [fp, #-12]

    @ --- EPILOGUE ---
    ADD sp, sp, #20     @ คืน local variable space
    POP {r4, r5, r6, r7, r8, fp, lr}
    BX lr

@ ============================
@ AArch64 Standard Frame
@ ============================
complex_function64:
    @ --- PROLOGUE ---
    STP x29, x30, [sp, #-64]!  @ บันทึก fp, lr และจัดสรร 64 bytes
    MOV x29, sp                  @ x29 = frame pointer
    STP x19, x20, [sp, #16]     @ บันทึก callee-saved registers
    STP x21, x22, [sp, #32]

    @ --- FUNCTION BODY ---
    MOV x19, x0            @ save a
    MOV x20, x1            @ save b
    MOV x21, x2            @ save c
    MOV x22, x3            @ save d

    ADD x0, x19, x20       @ result = a + b + c + d
    ADD x0, x0, x21
    ADD x0, x0, x22

    @ --- EPILOGUE ---
    LDP x21, x22, [sp, #32]    @ กู้คืน callee-saved
    LDP x19, x20, [sp, #16]
    LDP x29, x30, [sp], #64    @ กู้คืน fp, lr และคืน stack
    RET

_start:
    MOV r0, #1
    MOV r1, #2
    MOV r2, #3
    MOV r3, #4
    BL complex_function     @ r0 = (1+2) + (3*4) = 3 + 12 = 15

    MOV r7, #1
    SWI #0
```

---

## 16. การ Compile และ Run ด้วย QEMU

### 16.1 ARM32 บน Linux x86_64

```bash
# ติดตั้ง tools
sudo apt-get install gcc-arm-linux-gnueabihf
sudo apt-get install qemu-user

# Compile ARM32
arm-linux-gnueabihf-as -o program.o program.s
arm-linux-gnueabihf-ld -o program program.o

# Run ด้วย QEMU user mode
qemu-arm ./program
echo "Exit code: $?"

# Debug ด้วย GDB
qemu-arm -g 1234 ./program &
arm-linux-gnueabihf-gdb ./program
(gdb) target remote :1234
(gdb) b _start
(gdb) continue
(gdb) info registers
```

### 16.2 AArch64 บน Linux x86_64

```bash
# ติดตั้ง tools
sudo apt-get install gcc-aarch64-linux-gnu
sudo apt-get install qemu-user

# Compile AArch64
aarch64-linux-gnu-as -o program64.o program64.s
aarch64-linux-gnu-ld -o program64 program64.o

# Run
qemu-aarch64 ./program64

# หรือ compile ด้วย GCC ให้ง่ายขึ้น
aarch64-linux-gnu-gcc -nostdlib -o program64 program64.s
```

### 16.3 Script รวมทดสอบทั้งหมด

```bash
#!/bin/bash
# ไฟล์: test_all.sh
# วัตถุประสงค์: คอมไพล์และทดสอบโปรแกรมทั้งหมด

ARM_CC="arm-linux-gnueabihf-as"
ARM_LD="arm-linux-gnueabihf-ld"
ARM_RUN="qemu-arm"

AARCH64_CC="aarch64-linux-gnu-as"
AARCH64_LD="aarch64-linux-gnu-ld"
AARCH64_RUN="qemu-aarch64"

echo "=== Testing ARM32 Programs ==="

# Test factorial
$ARM_CC -o factorial.o recursive.s
$ARM_LD -o factorial factorial.o
result=$($ARM_RUN ./factorial; echo $?)
echo "Factorial(5) exit code: $result (expected 120)"

# Test fibonacci
echo "=== Testing AArch64 Programs ==="

$AARCH64_CC -o fib64.o aarch64_control.s
$AARCH64_LD -o fib64 fib64.o
$AARCH64_RUN ./fib64
echo "AArch64 program exit code: $?"

echo "=== All tests done ==="
```

### 16.4 Makefile สำหรับ Assembly Projects

```makefile
# Makefile สำหรับ ARM Assembly
CC_ARM    = arm-linux-gnueabihf-as
LD_ARM    = arm-linux-gnueabihf-ld
CC_A64    = aarch64-linux-gnu-as
LD_A64    = aarch64-linux-gnu-ld
QEMU_ARM  = qemu-arm
QEMU_A64  = qemu-aarch64

CFLAGS_ARM  = -march=armv7-a
CFLAGS_A64  =

ARM_SRCS = branch_basic.s bl_function.s recursive.s
A64_SRCS = branch_basic64.s aarch64_control.s

ARM_BINS = $(ARM_SRCS:.s=)
A64_BINS = $(A64_SRCS:.s=)

all: $(ARM_BINS) $(A64_BINS)

%: %.s
	$(CC_ARM) $(CFLAGS_ARM) -o $@.o $<
	$(LD_ARM) -o $@ $@.o

%64: %64.s
	$(CC_A64) $(CFLAGS_A64) -o $@.o $<
	$(LD_A64) -o $@ $@.o

test: all
	@echo "Testing ARM32..."
	@for prog in $(ARM_BINS); do \
		echo -n "  $$prog: "; \
		$(QEMU_ARM) ./$$prog 2>/dev/null; \
		echo "exit=$$?"; \
	done
	@echo "Testing AArch64..."
	@for prog in $(A64_BINS); do \
		echo -n "  $$prog: "; \
		$(QEMU_A64) ./$$prog 2>/dev/null; \
		echo "exit=$$?"; \
	done

clean:
	rm -f *.o $(ARM_BINS) $(A64_BINS)

.PHONY: all test clean
```

---

## 17. แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: ง่าย

**ข้อ 1.1**: เขียนฟังก์ชัน `is_even(n)` ที่คืนค่า 1 ถ้า n เป็นเลขคู่ 0 ถ้าเลขคี่ ใช้ TST แทน CMP

```asm
@ TODO: เขียนฟังก์ชัน is_even ที่นี่
@ Input:  r0 = n
@ Output: r0 = 1 ถ้า n เป็นเลขคู่, 0 ถ้าเลขคี่
is_even:
    @ ???
```

**ข้อ 1.2**: เขียน loop นับ 1-100 และหาผลบวกด้วย BL และ BX

**ข้อ 1.3**: ใช้ CBZ/CBNZ เขียนฟังก์ชัน `count_zeros(array, size)` ที่นับจำนวน 0 ในอาร์เรย์

### แบบฝึกหัดที่ 2: ปานกลาง

**ข้อ 2.1**: เขียน switch/case ด้วย jump table สำหรับ calculator (ADD, SUB, MUL, DIV)

```asm
@ TODO: Calculator ด้วย jump table
@ Input:  r0 = opcode (0=ADD, 1=SUB, 2=MUL, 3=DIV)
@         r1 = a, r2 = b
@ Output: r0 = result
calculator:
    @ ???
```

**ข้อ 2.2**: เขียน Quicksort แบบ recursive ใน ARM32

**ข้อ 2.3**: ใช้ IT block เขียนฟังก์ชัน `clamp(value, min, max)` ให้กระชับที่สุด

### แบบฝึกหัดที่ 3: ยาก

**ข้อ 3.1**: เขียน Binary Search แบบ iterative และ recursive ใน AArch64

```asm
// TODO: Binary Search
// x0 = pointer to sorted array
// x1 = size
// x2 = target
// Return: x0 = index หรือ -1 ถ้าไม่พบ
binary_search:
    // ???
```

**ข้อ 3.2**: Implement Bubble Sort ใน ARM32 ด้วย nested loops

**ข้อ 3.3**: เขียน TBB-based jump table สำหรับ simple virtual machine ที่มี 8 opcodes

### แบบฝึกหัดที่ 4: ท้าทาย

**ข้อ 4.1**: เขียน Merge Sort ใน AArch64 แบบ recursive ที่ถูกต้องตาม AAPCS64 calling convention

**ข้อ 4.2**: Implement Depth-First Search (DFS) บน graph ที่ represent ด้วย adjacency list ใน ARM32

**ข้อ 4.3**: เขียน Coroutine framework อย่างง่ายๆ ใน ARM32 โดยใช้ manual stack switching

---

## 18. เฉลยแบบฝึกหัดบางส่วน

### เฉลยข้อ 1.1

```asm
@ เฉลย: is_even
is_even:
    TST r0, #1          @ ตรวจสอบ bit 0
    MOVEQ r0, #1        @ ถ้า bit 0 = 0 (เลขคู่) return 1
    MOVNE r0, #0        @ ถ้า bit 0 = 1 (เลขคี่) return 0
    BX lr
```

### เฉลยข้อ 2.1

```asm
@ เฉลย: calculator ด้วย jump table
    .section .text
calculator:
    CMP r0, #3
    BHI calc_error

    ADR r3, calc_table
    LDR pc, [r3, r0, LSL #2]

calc_table:
    .word calc_add
    .word calc_sub
    .word calc_mul
    .word calc_div

calc_add:
    ADD r0, r1, r2
    BX lr

calc_sub:
    SUB r0, r1, r2
    BX lr

calc_mul:
    MUL r0, r1, r2
    BX lr

calc_div:
    @ ต้องตรวจสอบ division by zero ก่อน
    CMP r2, #0
    BEQ calc_error
    SDIV r0, r1, r2
    BX lr

calc_error:
    MOV r0, #-1
    BX lr
```

### เฉลยข้อ 3.1 (AArch64 Binary Search)

```asm
// เฉลย: binary_search ใน AArch64
binary_search:
    MOV x3, #0              // left = 0
    SUB x4, x1, #1          // right = size - 1

bs_loop:
    CMP x3, x4
    BGT bs_not_found        // ถ้า left > right ไม่พบ

    ADD x5, x3, x4          // mid = (left + right)
    LSR x5, x5, #1          // mid /= 2

    LDR x6, [x0, x5, LSL #3]   // x6 = array[mid]

    CMP x6, x2
    BEQ bs_found
    BLT bs_right
    B bs_left

bs_right:
    ADD x3, x5, #1          // left = mid + 1
    B bs_loop

bs_left:
    SUB x4, x5, #1          // right = mid - 1
    B bs_loop

bs_found:
    MOV x0, x5              // return mid
    RET

bs_not_found:
    MOV x0, #-1
    RET
```

---

## 19. สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

| คำสั่ง | การใช้งาน | หมายเหตุ |
|--------|----------|---------|
| `B label` | กระโดดแบบไม่มีเงื่อนไข | ±32MB range |
| `BL label` | เรียกฟังก์ชัน (บันทึก LR) | ±32MB range |
| `BX Rn` | กระโดดผ่าน register, อาจสลับ mode | สำหรับ return (BX lr) |
| `BLX Rn/label` | เรียกฟังก์ชัน + อาจสลับ mode | ARM↔Thumb |
| `Bcond label` | กระโดดตามเงื่อนไข (BEQ, BNE, ...) | ขึ้นกับ CPSR flags |
| `CBZ/CBNZ Rn, label` | Compare & Branch (Thumb-2) | forward only |
| `TBB/TBH [Rn, Rm]` | Table Branch | switch/case |
| `IT{x} cond` | If-Then block (Thumb-2) | 1-4 conditional instructions |

**หลักการสำคัญ**:
1. ใช้ `PUSH {lr}` / `POP {pc}` สำหรับฟังก์ชันที่เรียกฟังก์ชันอื่น
2. ใช้ `BX lr` สำหรับ leaf function (ไม่เรียกฟังก์ชันอื่น)
3. Jump table ช่วยให้ switch/case มีประสิทธิภาพ O(1)
4. IT block ช่วยลด branch penalty ใน CPU pipeline
5. CBZ/CBNZ ช่วยลดขนาด code ใน Thumb mode

---

## อ้างอิง (References)

- ARM Architecture Reference Manual (ARMv7-A/R)
- ARM Cortex-A Series Programmer's Guide
- AArch64 Instruction Set Architecture Reference
- AAPCS (ARM Architecture Procedure Call Standard)
- GAS (GNU Assembler) documentation

---

*Part 034 - ARM Branch และ Control Flow | Assembly Programming Course*
*ระดับ: ปานกลาง-สูง | ภาษา: Thai/English*
