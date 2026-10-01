# Part 033: ARM Load/Store Instructions — คำสั่ง Load และ Store ใน ARM Assembly

## บทนำ (Introduction)

ใน ARM Architecture นั้น การเข้าถึงหน่วยความจำ (Memory Access) ทำผ่านคำสั่ง **Load** และ **Store** เท่านั้น
ต่างจาก x86 ที่สามารถดำเนินการกับ memory operand ได้โดยตรง ARM ใช้ **Load/Store Architecture**
ซึ่งหมายความว่าข้อมูลต้องถูกโหลดเข้า register ก่อน จึงจะประมวลผลได้

```
Load/Store Architecture (RISC Style):
  Memory → Register (Load)
  Register → Memory (Store)
  
ต่างจาก x86:
  ADD [mem], reg   ; x86 ทำได้
  ADD r0, [mem]    ; ARM: ต้องใช้ LDR ก่อน แล้วค่อย ADD
```

---

## หัวข้อที่ครอบคลุม

1. LDR/STR — Word (32-bit) Load/Store
2. LDRB/STRB — Byte (8-bit) Load/Store  
3. LDRH/STRH — Halfword (16-bit) Load/Store
4. LDRSB/LDRSH — Signed Byte/Halfword Load
5. LDM/STM — Multiple Register Load/Store
6. PUSH/POP ใน ARM
7. Addressing Modes: Offset, Pre-indexed, Post-indexed
8. Base Register Update
9. LDM Modes: IA/IB/DA/DB
10. STMDB/LDMIA สำหรับ Stack
11. LDRD/STRD — Doubleword (64-bit)
12. Unaligned Access
13. Memory Barriers: DMB/DSB/ISB
14. Cache Maintenance

---

## 1. LDR/STR — Word Load/Store (32-bit)

### 1.1 คำสั่ง LDR (Load Register)

```asm
@ ARM32 — LDR Syntax
@ LDR{cond} Rd, [Rn {, #offset}]

@ ตัวอย่างพื้นฐาน
.section .data
my_var:   .word 0xDEADBEEF     @ ตัวแปร 32-bit

.section .text
.global _start

_start:
    LDR r0, =my_var        @ โหลด address ของ my_var ลง r0 (pseudo-instruction)
    LDR r1, [r0]           @ โหลด word จาก address ใน r0 → r1
    @ r1 = 0xDEADBEEF

    @ โหลดพร้อม offset
    LDR r2, [r0, #4]       @ โหลดจาก address (r0 + 4)
    LDR r3, [r0, #-4]      @ โหลดจาก address (r0 - 4)
```

### 1.2 คำสั่ง STR (Store Register)

```asm
@ ARM32 — STR Syntax
@ STR{cond} Rd, [Rn {, #offset}]

.section .data
result:   .word 0            @ พื้นที่เก็บผล

.section .text
_start:
    MOV r0, #42             @ ค่าที่ต้องการเก็บ
    LDR r1, =result         @ address ของ result
    STR r0, [r1]            @ เก็บ r0 (42) ไปที่ address ใน r1
    
    @ result ตอนนี้มีค่า 42
```

### 1.3 ตัวอย่างเต็ม: คัดลอกอาร์เรย์

```asm
@ arm32_word_copy.s — คัดลอกอาร์เรย์ word
@ Build: arm-linux-gnueabi-as arm32_word_copy.s -o arm32_word_copy.o
@        arm-linux-gnueabi-ld arm32_word_copy.o -o arm32_word_copy
@ Run:   qemu-arm ./arm32_word_copy

.section .data
src_array:
    .word 10, 20, 30, 40, 50    @ อาร์เรย์ต้นทาง 5 elements

dst_array:
    .word 0, 0, 0, 0, 0         @ อาร์เรย์ปลายทาง

.section .text
.global _start

_start:
    LDR r0, =src_array      @ r0 = address ของ src_array
    LDR r1, =dst_array      @ r1 = address ของ dst_array
    MOV r2, #5              @ จำนวน elements

copy_loop:
    CMP r2, #0              @ ตรวจว่าหมดแล้วหรือยัง
    BEQ done                @ ถ้าหมดแล้ว ไปที่ done
    
    LDR r3, [r0]            @ โหลด element จาก src
    STR r3, [r1]            @ เก็บ element ไปที่ dst
    
    ADD r0, r0, #4          @ เลื่อน pointer src ไปหน้า 4 bytes
    ADD r1, r1, #4          @ เลื่อน pointer dst ไปหน้า 4 bytes
    SUB r2, r2, #1          @ ลดตัวนับ
    B   copy_loop           @ วนซ้ำ

done:
    MOV r7, #1              @ syscall: exit
    MOV r0, #0              @ exit code 0
    SVC #0
```

### 1.4 AArch64 — LDR/STR (64-bit)

```asm
// arm64_ldr_str.s — AArch64 version
// Build: aarch64-linux-gnu-as arm64_ldr_str.s -o arm64_ldr_str.o
//        aarch64-linux-gnu-ld arm64_ldr_str.o -o arm64_ldr_str
// Run:   qemu-aarch64 ./arm64_ldr_str

.section .data
value64:    .quad 0x123456789ABCDEF0   // 64-bit value
value32:    .word 0xDEADBEEF           // 32-bit value

.section .text
.global _start

_start:
    // โหลด 64-bit value ด้วย X register
    adrp    x0, value64             // โหลด page address
    add     x0, x0, :lo12:value64  // เพิ่ม offset ภายใน page
    ldr     x1, [x0]               // โหลด 64-bit → x1

    // โหลด 32-bit value ด้วย W register
    adrp    x2, value32
    add     x2, x2, :lo12:value32
    ldr     w3, [x2]               // โหลด 32-bit → w3 (zero-extend ไปยัง x3)

    // Store
    mov     x4, #0xCAFEBABE
    str     x4, [x0]               // เก็บ 64-bit

    // Exit
    mov     x8, #93                 // syscall: exit
    mov     x0, #0
    svc     #0
```

---

## 2. LDRB/STRB — Byte Load/Store (8-bit)

### 2.1 คำสั่ง LDRB และ STRB

```asm
@ arm32_byte_ops.s — การทำงานกับ Byte
@ LDRB โหลด 1 byte แล้ว zero-extend ไปยัง 32-bit register
@ STRB เก็บ byte ต่ำสุด (bits 7:0) ของ register ลง memory

.section .data
str_data:   .ascii "Hello, ARM!\n"   @ สตริง ASCII
byte_var:   .byte 0                  @ ตัวแปร byte เดียว

.section .text
.global _start

_start:
    LDR r0, =str_data       @ r0 = address ของสตริง

    @ อ่าน bytes ทีละตัว
    LDRB r1, [r0]           @ r1 = 'H' (0x48), zero-extended
    LDRB r2, [r0, #1]       @ r2 = 'e' (0x65)
    LDRB r3, [r0, #2]       @ r3 = 'l' (0x6C)

    @ ตรวจสอบ zero-extension
    @ r1 = 0x00000048 (ไม่ใช่ 0xFF??????)

    @ นับจำนวน bytes ใน string (strlen)
    LDR r0, =str_data
    MOV r4, #0              @ counter

strlen_loop:
    LDRB r1, [r0], #1       @ โหลด byte และ post-increment r0
    CMP r1, #0              @ ตรวจ null terminator
    BEQ strlen_done
    ADD r4, r4, #1          @ เพิ่ม counter
    B strlen_loop

strlen_done:
    @ r4 = ความยาวของสตริง

    @ แก้ไข byte เดียวในสตริง
    LDR r0, =str_data
    MOV r1, #'h'            @ เปลี่ยน 'H' เป็น 'h'
    STRB r1, [r0]           @ เก็บเฉพาะ byte ต่ำสุด

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 2.2 AArch64 — LDRB/STRB

```asm
// arm64_byte_ops.s
.section .data
buffer: .fill 16, 1, 0      // 16 bytes เริ่มต้นเป็น 0

.section .text
.global _start

_start:
    adrp    x0, buffer
    add     x0, x0, :lo12:buffer

    // เก็บ bytes ทีละตัว
    mov     w1, #'A'
    strb    w1, [x0]            // เก็บ 'A' ที่ offset 0
    mov     w1, #'R'
    strb    w1, [x0, #1]        // เก็บ 'R' ที่ offset 1
    mov     w1, #'M'
    strb    w1, [x0, #2]        // เก็บ 'M' ที่ offset 2

    // โหลด byte กลับ
    ldrb    w2, [x0]            // w2 = 'A' (0x41), zero-extended

    mov     x8, #93
    mov     x0, #0
    svc     #0
```

---

## 3. LDRH/STRH — Halfword Load/Store (16-bit)

### 3.1 คำสั่ง LDRH และ STRH

```asm
@ arm32_halfword.s — การทำงานกับ Halfword (16-bit)
@ LDRH โหลด 2 bytes แล้ว zero-extend ไปยัง 32-bit
@ STRH เก็บ 2 bytes ต่ำสุด ของ register ลง memory

.section .data
.align 2                     @ align ให้ 2-byte boundary
hw_array:
    .hword 0x1234, 0x5678, 0xABCD, 0xEF01  @ อาร์เรย์ halfword

.section .text
.global _start

_start:
    LDR r0, =hw_array        @ r0 = base address

    LDRH r1, [r0]            @ r1 = 0x00001234
    LDRH r2, [r0, #2]        @ r2 = 0x00005678
    LDRH r3, [r0, #4]        @ r3 = 0x0000ABCD

    @ เปลี่ยนค่า halfword
    MOV r4, #0x9999
    STRH r4, [r0, #2]        @ เขียน 0x9999 ที่ตำแหน่ง [r0+2]

    @ ตรวจสอบ: hw_array ตอนนี้ = {0x1234, 0x9999, 0xABCD, 0xEF01}

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 3.2 ตัวอย่าง: แปลง Endianness ของ Halfword

```asm
@ swap_endian_hword.s — สลับ byte order ของ halfword
@ Big-endian 0xABCD → Little-endian 0xCDAB

.section .text
.global _start

_start:
    MOV r0, #0xABCD          @ ค่า halfword ต้นทาง

    @ วิธีที่ 1: ใช้ ROR
    MOV r1, r0, ROR #8       @ หมุน right 8 bits
    AND r1, r1, #0xFFFF      @ เอาแค่ 16-bit ล่าง
    @ r1 = 0xCDAB

    @ วิธีที่ 2: ใช้ REV16 instruction (ARMv6+)
    REV16 r2, r0             @ สลับ byte ใน each halfword
    AND r2, r2, #0xFFFF
    @ r2 = 0xCDAB

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

---

## 4. LDRSB/LDRSH — Signed Byte/Halfword Load

### 4.1 ความแตกต่างระหว่าง Unsigned และ Signed Load

```asm
@ arm32_signed_load.s — การโหลดค่า Signed

@ ปัญหา: LDRB ทำ zero-extension
@   ค่า -1 (0xFF) → LDRB → 0x000000FF = 255 (ไม่ใช่ -1!)
@ แก้ไข: LDRSB ทำ sign-extension
@   ค่า -1 (0xFF) → LDRSB → 0xFFFFFFFF = -1 (ถูกต้อง!)

.section .data
signed_bytes:
    .byte 127, -1, -128, 0, 100    @ bytes รวมค่า negative

signed_hwords:
    .hword 32767, -1, -32768, 0    @ halfwords รวมค่า negative

.section .text
.global _start

_start:
    LDR r0, =signed_bytes

    @ LDRB — zero extension (unsigned)
    LDRB r1, [r0, #1]       @ r1 = 0x000000FF = 255 (ไม่ใช่ -1)

    @ LDRSB — sign extension (signed)
    LDRSB r2, [r0, #1]      @ r2 = 0xFFFFFFFF = -1 (ถูกต้องสำหรับ signed)

    @ เปรียบเทียบผล
    @ r1 = 255 (unsigned interpretation)
    @ r2 = -1  (signed interpretation)

    LDR r3, =signed_hwords

    @ LDRH — zero extension
    LDRH r4, [r3, #2]       @ r4 = 0x0000FFFF = 65535

    @ LDRSH — sign extension
    LDRSH r5, [r3, #2]      @ r5 = 0xFFFFFFFF = -1

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 4.2 ตัวอย่างจริง: อ่านค่า Temperature Sensor (signed byte)

```asm
@ temperature_sensor.s — อ่านค่าอุณหภูมิที่อาจเป็นค่าลบ
@ สมมติ temperature sensor ส่งค่า signed byte (-40 ถึง +127 องศา)

.section .data
sensor_data:
    .byte -15, 25, -40, 100, 37    @ ค่าจาก sensor
    
.section .text
.global _start

_start:
    LDR r0, =sensor_data
    MOV r1, #5              @ จำนวน readings
    MOV r2, #0              @ sum
    MOV r3, r0              @ pointer

avg_loop:
    CMP r1, #0
    BEQ calc_avg
    
    LDRSB r4, [r3], #1      @ อ่านค่า signed byte + post-increment
    ADD r2, r2, r4          @ บวกเข้า sum
    SUB r1, r1, #1
    B avg_loop

calc_avg:
    @ r2 = sum (-15+25-40+100+37 = 107)
    MOV r1, #5
    @ ใช้ SDIV ถ้า CPU support
    @ หรือใช้ softdiv routine

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

---

## 5. Addressing Modes — รูปแบบการ Address หน่วยความจำ

### 5.1 Overview ของ Addressing Modes

```
ARM32 Addressing Modes:
┌─────────────────────────────────────────────────────────┐
│ 1. Register Offset (Immediate)                          │
│    LDR Rd, [Rn, #imm]      → addr = Rn + imm           │
│                                                         │
│ 2. Register Offset (Register)                           │
│    LDR Rd, [Rn, Rm]        → addr = Rn + Rm             │
│    LDR Rd, [Rn, Rm, LSL#n] → addr = Rn + (Rm << n)     │
│                                                         │
│ 3. Pre-indexed (with writeback)                         │
│    LDR Rd, [Rn, #imm]!     → addr = Rn + imm; Rn += imm│
│                                                         │
│ 4. Post-indexed                                         │
│    LDR Rd, [Rn], #imm      → addr = Rn; Rn += imm      │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Immediate Offset Addressing

```asm
@ immediate_offset.s — Immediate Offset

.section .data
array:  .word 100, 200, 300, 400, 500

.section .text
.global _start

_start:
    LDR r0, =array

    @ Positive immediate offset
    LDR r1, [r0, #0]    @ element[0] = 100
    LDR r2, [r0, #4]    @ element[1] = 200
    LDR r3, [r0, #8]    @ element[2] = 300
    LDR r4, [r0, #12]   @ element[3] = 400
    LDR r5, [r0, #16]   @ element[4] = 500

    @ Negative offset (มองย้อนกลับ)
    ADD r6, r0, #16     @ r6 ชี้ที่ element[4]
    LDR r7, [r6, #-4]   @ โหลด element[3] = 400

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 5.3 Register Offset Addressing

```asm
@ register_offset.s — Register-based Offset

.section .data
matrix:
    .word 1,  2,  3,  4
    .word 5,  6,  7,  8
    .word 9,  10, 11, 12

.section .text
.global _start

_start:
    LDR r0, =matrix      @ base address

    @ เข้าถึง matrix[row][col]
    @ address = base + (row * 4 + col) * 4
    MOV r1, #1           @ row = 1
    MOV r2, #2           @ col = 2
    
    @ คำนวณ index = row * 4 + col
    MOV r3, #4           @ columns per row
    MUL r4, r1, r3       @ r4 = row * 4
    ADD r4, r4, r2       @ r4 = row * 4 + col = index
    
    @ คำนวณ byte offset = index * 4
    LSL r5, r4, #2       @ r5 = index * 4 (shift left 2 = multiply 4)
    
    @ โหลดโดยใช้ register offset
    LDR r6, [r0, r5]     @ r6 = matrix[1][2] = 7
    
    @ ใช้ scaled register offset (ทำในคำสั่งเดียว)
    @ LDR Rd, [Rn, Rm, LSL #2]  → addr = Rn + Rm*4
    LDR r7, [r0, r4, LSL #2]    @ r7 = matrix[1][2] = 7

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 5.4 Pre-indexed Addressing (with Writeback)

```asm
@ pre_indexed.s — Pre-indexed Addressing
@ Syntax: LDR Rd, [Rn, #offset]!
@ Effect: Rn = Rn + offset FIRST, then load from new Rn

.section .data
data: .word 0xAA, 0xBB, 0xCC, 0xDD, 0xEE

.section .text
.global _start

_start:
    LDR r0, =data           @ r0 = base address
    SUB r0, r0, #4          @ r0 ชี้ก่อนหน้า element[0]

    @ Pre-indexed: เลื่อน pointer ก่อน แล้วค่อยอ่าน
    LDR r1, [r0, #4]!       @ r0 += 4 → r0 = &data[0], r1 = data[0] = 0xAA
    LDR r2, [r0, #4]!       @ r0 += 4 → r0 = &data[1], r2 = data[1] = 0xBB
    LDR r3, [r0, #4]!       @ r0 += 4 → r0 = &data[2], r3 = data[2] = 0xCC

    @ ผลลัพธ์:
    @ r0 = &data[2]
    @ r1 = 0xAA, r2 = 0xBB, r3 = 0xCC

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 5.5 Post-indexed Addressing

```asm
@ post_indexed.s — Post-indexed Addressing
@ Syntax: LDR Rd, [Rn], #offset
@ Effect: Load from current Rn FIRST, then Rn = Rn + offset

.section .data
data: .word 0xAA, 0xBB, 0xCC, 0xDD, 0xEE

.section .text
.global _start

_start:
    LDR r0, =data           @ r0 = &data[0]

    @ Post-indexed: อ่านก่อน แล้วค่อยเลื่อน pointer
    LDR r1, [r0], #4        @ r1 = data[0] = 0xAA, r0 += 4 → &data[1]
    LDR r2, [r0], #4        @ r2 = data[1] = 0xBB, r0 += 4 → &data[2]
    LDR r3, [r0], #4        @ r3 = data[2] = 0xCC, r0 += 4 → &data[3]

    @ r0 ตอนนี้ = &data[3]
    @ r1=0xAA, r2=0xBB, r3=0xCC

    @ เปรียบเทียบกับ loop โดยใช้ post-indexed
    LDR r0, =data
    MOV r4, #5              @ counter
    MOV r5, #0              @ sum

sum_loop:
    CMP r4, #0
    BEQ sum_done
    LDR r6, [r0], #4        @ อ่านและเลื่อน pointer พร้อมกัน
    ADD r5, r5, r6          @ บวกเข้า sum
    SUB r4, r4, #1
    B sum_loop

sum_done:
    @ r5 = 0xAA + 0xBB + 0xCC + 0xDD + 0xEE

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

---

## 6. LDM/STM — Multiple Register Load/Store

### 6.1 ทำความเข้าใจ LDM/STM

```
LDM (Load Multiple): โหลดหลาย registers จาก memory พร้อมกัน
STM (Store Multiple): เก็บหลาย registers ลง memory พร้อมกัน

Syntax: LDM{cond}{mode} Rn{!}, reglist
        STM{cond}{mode} Rn{!}, reglist

Modes:
  IA = Increment After  (โหลดแล้วเพิ่ม address)
  IB = Increment Before (เพิ่ม address ก่อนโหลด)
  DA = Decrement After  (โหลดแล้วลด address)
  DB = Decrement Before (ลด address ก่อนโหลด)

! = writeback: อัพเดต base register หลังดำเนินการ
```

### 6.2 LDMIA — Load Multiple, Increment After (ที่ใช้บ่อยที่สุด)

```asm
@ ldmia_example.s — LDMIA: โหลดขึ้นไป

.section .data
vals: .word 10, 20, 30, 40, 50

.section .text
.global _start

_start:
    LDR r0, =vals           @ r0 = address ของ vals[0]

    @ LDMIA: โหลด 5 registers จากที่อยู่ใน r0 (IA = Increment After)
    @ r0 เริ่มต้นชี้ที่ vals[0]
    LDMIA r0, {r1, r2, r3, r4, r5}
    @ r1 = 10, r2 = 20, r3 = 30, r4 = 40, r5 = 50
    @ r0 ไม่เปลี่ยน (ไม่มี !)

    @ LDMIA พร้อม writeback
    LDR r0, =vals
    LDMIA r0!, {r1, r2, r3}
    @ r1 = 10, r2 = 20, r3 = 30
    @ r0 ถูกอัพเดต: r0 = vals + 12 (ชี้ที่ vals[3])

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 6.3 STMIA — Store Multiple, Increment After

```asm
@ stmia_example.s — STMIA

.section .data
output: .fill 20, 1, 0      @ 5 words พื้นที่

.section .text
.global _start

_start:
    @ ตั้งค่า registers
    MOV r1, #100
    MOV r2, #200
    MOV r3, #300
    MOV r4, #400
    MOV r5, #500

    LDR r0, =output         @ r0 = address ของ output

    @ STMIA: เก็บ 5 registers ลง memory (IA = Increment After)
    STMIA r0!, {r1, r2, r3, r4, r5}
    @ output[0]=100, output[1]=200, ..., output[4]=500
    @ r0 ถูกอัพเดต = output + 20

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 6.4 LDM/STM Modes ทั้งหมด

```asm
@ ldm_modes.s — แสดง LDM/STM ทุก modes

.section .data
data: .word 1, 2, 3, 4, 5, 6, 7, 8

.section .text
.global _start

_start:
    LDR r8, =data

    @ --- LDMIA (Increment After) ---
    @ address เริ่มต้นที่ r8, โหลดแล้วเพิ่ม
    LDR r0, =data
    LDMIA r0, {r1, r2, r3}  @ r1=1, r2=2, r3=3 (r0 ไม่เปลี่ยน)

    @ --- LDMIB (Increment Before) ---
    @ เพิ่ม address ก่อน แล้วโหลด
    LDR r0, =data
    LDMIB r0, {r1, r2, r3}  @ r1=2, r2=3, r3=4 (ข้ามตัวแรก)

    @ --- LDMDA (Decrement After) ---
    @ address เริ่มต้นสูง โหลดแล้วลด
    LDR r0, =data
    ADD r0, r0, #8           @ ชี้ที่ data[2]
    LDMDA r0, {r1, r2, r3}  @ r3=data[2], r2=data[1], r1=data[0]

    @ --- LDMDB (Decrement Before) ---
    @ ลด address ก่อน แล้วโหลด (ใช้ใน POP mechanism)
    LDR r0, =data
    ADD r0, r0, #12          @ ชี้ที่ data[3]
    LDMDB r0, {r1, r2, r3}  @ r1=data[0], r2=data[1], r3=data[2]

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

---

## 7. PUSH/POP ใน ARM

### 7.1 PUSH และ POP คือ Alias ของ STM/LDM

```
ARM Stack Convention:
- Stack เติบโตลงล่าง (Descending)
- SP ชี้ที่ top item (Full Descending)

PUSH {regs} = STMDB SP!, {regs}
             (Decrement Before, เพราะต้อง decrement ก่อนเขียน)

POP  {regs} = LDMIA SP!, {regs}
             (Increment After, เพราะอ่านแล้วค่อย increment)
```

### 7.2 การใช้ PUSH/POP พื้นฐาน

```asm
@ push_pop_basic.s — การใช้ PUSH/POP

.section .text
.global _start

_start:
    @ ตั้งค่า stack (ถ้า linker ไม่ตั้งให้)
    LDR sp, =stack_top

    MOV r0, #10
    MOV r1, #20
    MOV r2, #30

    @ บันทึก registers ก่อนเรียก function
    PUSH {r0, r1, r2}       @ SP -= 12; [SP]=r0, [SP+4]=r1, [SP+8]=r2

    @ แก้ไขค่า registers
    MOV r0, #999
    MOV r1, #888

    @ คืนค่า registers
    POP {r0, r1, r2}        @ r0=10, r1=20, r2=30; SP += 12

    @ PUSH/POP กับ LR (Link Register)
    @ สำคัญมากสำหรับ function calls
    BL my_function          @ เรียก function, LR = address ถัดไป

    MOV r7, #1
    MOV r0, #0
    SVC #0

my_function:
    PUSH {r4, r5, lr}       @ บันทึก callee-saved regs + LR
    
    @ body ของ function
    MOV r4, #100
    MOV r5, #200
    ADD r0, r4, r5          @ r0 = 300 (return value)
    
    POP {r4, r5, pc}        @ คืนค่า regs และ jump กลับ (LR → PC)

.section .bss
stack_space: .skip 4096     @ 4KB stack
stack_top:                  @ ชี้ที่ปลาย stack
```

### 7.3 Function Call ที่ถูกต้องด้วย PUSH/POP

```asm
@ function_calls.s — Function Call Convention ใน ARM32
@ ARM PCS (Procedure Call Standard):
@   r0-r3: argument/return, caller-saved
@   r4-r11: callee-saved (function ต้อง save/restore)
@   r12 (ip): intra-procedure-call, caller-saved
@   r13 (sp): stack pointer
@   r14 (lr): link register
@   r15 (pc): program counter

.section .text
.global _start

@ int add(int a, int b, int c, int d, int e)
@ a=r0, b=r1, c=r2, d=r3, e=[sp]
add_five:
    PUSH {r4, lr}           @ บันทึก r4 (callee-saved) และ LR
    
    LDR r4, [sp, #8]        @ อ่าน 5th argument จาก stack (ข้าม r4, lr ที่ push ไป)
    ADD r0, r0, r1
    ADD r0, r0, r2
    ADD r0, r0, r3
    ADD r0, r0, r4          @ รวมทั้ง 5 arguments
    
    POP {r4, pc}            @ คืนค่า r4 และ return

_start:
    LDR sp, =stack_top

    @ เตรียม arguments
    MOV r0, #1              @ arg1
    MOV r1, #2              @ arg2
    MOV r2, #3              @ arg3
    MOV r3, #4              @ arg4
    MOV r4, #5
    PUSH {r4}               @ arg5 บน stack
    
    BL add_five             @ เรียก function
    
    ADD sp, sp, #4          @ ล้าง argument ที่ push ไว้
    @ r0 = 1+2+3+4+5 = 15

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

---

## 8. STMDB/LDMIA — Stack Implementation

### 8.1 Full Descending Stack

```asm
@ full_descending_stack.s — Stack ที่ ARM ใช้จริง

@ ARM ใช้ Full Descending Stack:
@ - "Full" หมายถึง SP ชี้ที่ last item ที่ push ไว้
@ - "Descending" หมายถึง Stack เติบโตลงล่าง (address ลดลง)

@ PUSH = STMDB (Store Multiple, Decrement Before)
@ POP  = LDMIA (Load Multiple, Increment After)

.section .text
.global _start

_start:
    LDR sp, =stack_top

    @ Manual PUSH โดยใช้ STMDB
    MOV r0, #10
    MOV r1, #20
    MOV r2, #30
    
    STMDB sp!, {r0, r1, r2}
    @ เทียบเท่ากับ: PUSH {r0, r1, r2}
    @ SP -= 12
    @ [SP+8] = r2 = 30
    @ [SP+4] = r1 = 20
    @ [SP+0] = r0 = 10  (SP ชี้ที่นี่)

    @ แสดง memory layout:
    @ Address:  Content:
    @ SP+8:    30 (r2)
    @ SP+4:    20 (r1)
    @ SP+0:    10 (r0)  ← SP ชี้ที่นี่

    @ Manual POP โดยใช้ LDMIA
    MOV r0, #0
    MOV r1, #0
    MOV r2, #0
    
    LDMIA sp!, {r0, r1, r2}
    @ เทียบเท่ากับ: POP {r0, r1, r2}
    @ r0 = [SP+0] = 10
    @ r1 = [SP+4] = 20
    @ r2 = [SP+8] = 30
    @ SP += 12

    @ r0=10, r1=20, r2=30 (คืนค่าได้ถูกต้อง)

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

### 8.2 Recursive Function ด้วย PUSH/POP

```asm
@ recursive_factorial.s — Factorial แบบ Recursive

.section .text
.global _start

@ int factorial(int n) — คำนวณ n!
@ input:  r0 = n
@ output: r0 = n!
factorial:
    PUSH {r0, lr}           @ บันทึก n และ return address
    
    CMP r0, #1
    BLE base_case           @ ถ้า n <= 1, ไปที่ base case
    
    SUB r0, r0, #1          @ r0 = n - 1
    BL factorial            @ recursive call: factorial(n-1)
    @ r0 = factorial(n-1)
    
    LDR r1, [sp]            @ โหลด n ที่บันทึกไว้ใน stack
    MUL r0, r0, r1          @ r0 = n * factorial(n-1)
    
    POP {r1, pc}            @ คืน r1 (ทิ้ง n) และ return
                            @ ใช้ r1 เพื่อ discard n ที่ stack

base_case:
    MOV r0, #1              @ factorial(0) = factorial(1) = 1
    POP {r1, pc}            @ return 1

_start:
    LDR sp, =stack_top

    MOV r0, #6              @ คำนวณ 6! = 720
    BL factorial
    @ r0 = 720

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

---

## 9. LDRD/STRD — Doubleword Load/Store (64-bit)

### 9.1 LDRD และ STRD ใน ARM32

```asm
@ arm32_doubleword.s — Doubleword Load/Store

@ LDRD: โหลด 64-bit (2 registers คู่กัน)
@ Syntax: LDRD Rd, Rd+1, [Rn {, #offset}]
@ ข้อกำหนด: Rd ต้องเป็น even register (r0, r2, r4, ...)
@           Rd+1 = Rd + 1 (อัตโนมัติ)

.section .data
.align 8                    @ align 8-byte boundary
dword_val: .quad 0x0123456789ABCDEF    @ 64-bit value

.section .text
.global _start

_start:
    LDR r0, =dword_val       @ address

    @ โหลด 64-bit value
    LDRD r2, r3, [r0]        @ r2 = low 32-bits, r3 = high 32-bits
    @ (little-endian: r2 = 0x89ABCDEF, r3 = 0x01234567)

    @ Store 64-bit value
    MOV r4, #0xAAAA
    MOVT r4, #0x1111         @ r4 = 0x1111AAAA
    MOV r5, #0xBBBB
    MOVT r5, #0x2222         @ r5 = 0x2222BBBB

    LDR r0, =dword_val
    STRD r4, r5, [r0]        @ เก็บ r4,r5 เป็น 64-bit

    MOV r7, #1
    MOV r0, #0
    SVC #0
```

### 9.2 AArch64 — LDP/STP (Load/Store Pair)

```asm
// arm64_ldp_stp.s — AArch64 Load/Store Pair
// LDP/STP มี performance ดีกว่า LDR/STR แยกกัน

.section .data
.align 16
values: .quad 0xAABBCCDD11223344, 0x5566778899AABBCC

.section .text
.global _start

_start:
    adrp    x0, values
    add     x0, x0, :lo12:values

    // LDP — Load Pair
    ldp     x1, x2, [x0]           // x1 = values[0], x2 = values[1]

    // LDP กับ offset
    ldp     x3, x4, [x0, #16]      // โหลดจาก address (x0 + 16)

    // STP — Store Pair
    mov     x5, #0xDEAD
    movk    x5, #0xBEEF, lsl #16
    mov     x6, #0xCAFE
    movk    x6, #0xBABE, lsl #16

    stp     x5, x6, [x0]           // เก็บ x5, x6

    // STP กับ pre-indexed
    sub     sp, sp, #32             // จัดสรร stack space
    stp     x1, x2, [sp, #0]!      // ผิด! ต้องใช้ stp x1,x2,[sp] แล้ว pre-index ต่างหาก
    
    // วิธีถูกต้อง: Push registers to stack
    stp     x1, x2, [sp, #-16]!    // SP -= 16, เก็บ x1, x2
    stp     x3, x4, [sp, #-16]!    // SP -= 16, เก็บ x3, x4

    // Pop registers from stack
    ldp     x3, x4, [sp], #16      // โหลด x3, x4, SP += 16
    ldp     x1, x2, [sp], #16      // โหลด x1, x2, SP += 16

    mov     x8, #93
    mov     x0, #0
    svc     #0
```

---

## 10. Unaligned Access

### 10.1 ปัญหา Alignment ใน ARM

```
ARM Memory Alignment Rules:
┌──────────────────────────────────────────────────────┐
│ Type    │ Natural Alignment │ Note                   │
├──────────────────────────────────────────────────────┤
│ Byte    │ 1 byte            │ ไม่มีปัญหา              │
│ Halfword│ 2 bytes           │ address ต้อง % 2 == 0  │
│ Word    │ 4 bytes           │ address ต้อง % 4 == 0  │
│ Dword   │ 8 bytes           │ address ต้อง % 8 == 0  │
└──────────────────────────────────────────────────────┘

Unaligned access behavior:
- ARMv4/v5: UNPREDICTABLE (อาจ crash)
- ARMv6: สนับสนุน unaligned แต่ช้ากว่า
- ARMv7+: สนับสนุนใน most cases
- AArch64: สนับสนุนโดยทั่วไป
```

### 10.2 จัดการ Unaligned Access อย่างปลอดภัย

```asm
@ unaligned_safe.s — อ่าน word จาก unaligned address อย่างปลอดภัย

.section .data
raw_data: .byte 0x11, 0x22, 0x33, 0x44, 0x55, 0x66

.section .text
.global _start

@ อ่าน 32-bit word จาก unaligned address โดยใช้ LDRB
@ input:  r0 = address (อาจ unaligned)
@ output: r0 = 32-bit word (little-endian)
read_unaligned_word:
    LDRB r1, [r0]            @ byte 0 (ต่ำสุด)
    LDRB r2, [r0, #1]        @ byte 1
    LDRB r3, [r0, #2]        @ byte 2
    LDRB r4, [r0, #3]        @ byte 3 (สูงสุด)
    
    @ รวม bytes เป็น word (little-endian)
    ORR r0, r1, r2, LSL #8
    ORR r0, r0, r3, LSL #16
    ORR r0, r0, r4, LSL #24
    
    BX lr

_start:
    LDR sp, =stack_top

    LDR r0, =raw_data
    ADD r0, r0, #1           @ ทำให้ unaligned (offset 1)
    BL read_unaligned_word   @ อ่านอย่างปลอดภัย
    @ r0 = 0x55443322 (little-endian)

    @ ARMv6+ สามารถใช้ LDR โดยตรง (แต่ช้ากว่า)
    LDR r1, =raw_data
    ADD r1, r1, #1
    LDR r2, [r1]             @ ARMv6+: อ่าน unaligned word
    @ r2 = 0x55443322

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

### 10.3 LDREX/STREX — Exclusive Load/Store

```asm
@ exclusive_access.s — Atomic Operations ใน ARM
@ ใช้สำหรับ synchronization ใน multi-threaded code

.section .data
.align 4
shared_counter: .word 0

.section .text
.global _start

@ Atomic increment (thread-safe)
@ input:  r0 = address ของ counter
@ output: r1 = ค่าใหม่
atomic_increment:
retry:
    LDREX r1, [r0]           @ อ่าน exclusive
    ADD r1, r1, #1           @ เพิ่มค่า
    STREX r2, r1, [r0]       @ พยายาม store exclusive
    CMP r2, #0               @ ตรวจสอบว่าสำเร็จหรือไม่
    BNE retry                @ ถ้าล้มเหลว (r2 != 0) ลองใหม่
    BX lr

_start:
    LDR sp, =stack_top

    LDR r0, =shared_counter
    BL atomic_increment      @ increment อย่างปลอดภัย
    BL atomic_increment      @ increment อีกครั้ง

    @ shared_counter = 2

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

---

## 11. Memory Barriers — DMB/DSB/ISB

### 11.1 ทำไมต้องใช้ Memory Barriers

```
ปัญหา: Modern CPUs ทำ Out-of-Order Execution และ Memory Reordering
       ทำให้ code ที่ดูเหมือน sequential อาจ execute ในลำดับต่างกัน

Memory Barriers บังคับให้ CPU ดำเนินการตามลำดับที่กำหนด:

DMB (Data Memory Barrier):
  - ทุก memory access ก่อน DMB ต้องเสร็จก่อน access หลัง DMB
  - ใช้กับ Normal memory (RAM)

DSB (Data Synchronization Barrier):
  - แรงกว่า DMB: CPU ต้องหยุดรอให้ทุก access เสร็จสิ้น
  - ใช้กับ I/O registers, page tables

ISB (Instruction Synchronization Barrier):
  - Flush pipeline ทั้งหมด
  - ใช้หลัง self-modifying code หรือเปลี่ยน CPU mode
```

### 11.2 ตัวอย่างการใช้ Memory Barriers

```asm
@ memory_barriers.s — การใช้ DMB/DSB/ISB

.section .data
flag:   .word 0
data:   .word 0

.section .text
.global _start

@ Producer: เขียน data แล้ว set flag
@ ต้องใช้ DMB เพื่อให้ data ถูกเขียนก่อน flag
producer:
    LDR r0, =data
    MOV r1, #42
    STR r1, [r0]            @ เขียน data ก่อน
    
    DMB                     @ บังคับให้ STR ข้างบนเสร็จก่อน
    
    LDR r0, =flag
    MOV r1, #1
    STR r1, [r0]            @ set flag หลัง data พร้อมแล้ว
    
    BX lr

@ Consumer: ตรวจ flag แล้วอ่าน data
@ ต้องใช้ DMB หลังอ่าน flag เพื่อให้แน่ใจว่า data พร้อมแล้ว
consumer:
wait_for_flag:
    LDR r0, =flag
    LDR r1, [r0]            @ ตรวจ flag
    CMP r1, #0
    BEQ wait_for_flag       @ รอถ้า flag = 0
    
    DMB                     @ บังคับให้ flag อ่านเสร็จก่อน data
    
    LDR r0, =data
    LDR r1, [r0]            @ อ่าน data อย่างปลอดภัย
    @ r1 = 42 (guaranteed)
    
    BX lr

_start:
    LDR sp, =stack_top

    BL producer
    BL consumer

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

### 11.3 DSB ใน Context Switching

```asm
@ context_switch_barriers.s — DSB ใน OS-level Code

@ DSB ใช้บ่อยใน:
@ 1. Cache maintenance operations
@ 2. TLB invalidation
@ 3. Page table updates
@ 4. Before ISBC after code modification

@ ตัวอย่าง: เปลี่ยน interrupt handler (สมมติ)
change_isr:
    @ เขียน ISR ใหม่ลงที่ vector table
    LDR r0, =vector_table
    LDR r1, =new_handler
    STR r1, [r0]            @ อัพเดต vector

    DSB                     @ ต้องรอให้ store เสร็จก่อน
    ISB                     @ Flush pipeline เพื่อดึง instruction ใหม่

    @ ตอนนี้ safe ที่จะ trigger interrupt
    BX lr

.section .data
vector_table: .word 0
new_handler: .word 0
```

### 11.4 ISB หลัง Self-Modifying Code

```asm
@ self_modifying.s — Self-Modifying Code (ต้องระวัง!)
@ เทคนิคนี้ทำใน runtime code patching, JIT compilers

.section .text
.global _start

_start:
    LDR sp, =stack_top

    @ ตัวอย่าง: แก้ instruction ณ runtime
    LDR r0, =patch_target   @ address ของ instruction ที่ต้องการแก้

    @ เขียน NOP instruction (0xE320F000) แทน MOV r0, #1
    LDR r1, =0xE320F000     @ NOP encoding ใน ARM32
    STR r1, [r0]            @ เขียน NOP ลงที่ patch_target

    DSB                     @ รอให้ store เสร็จ
    ISB                     @ Flush instruction cache/pipeline

    @ ตอนนี้ patch_target จะเป็น NOP แทน MOV r0, #1

patch_target:
    MOV r0, #1              @ instruction นี้อาจถูกแทนที่ด้วย NOP

    ADD r0, r0, #1          @ ถ้า patch สำเร็จ: r0 = 0+1 = 1
                            @ ถ้า patch ล้มเหลว: r0 = 1+1 = 2

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

---

## 12. Cache Maintenance

### 12.1 ทำความเข้าใจ Cache Hierarchy

```
ARM Cache Architecture:
┌─────────────────────────────────────────────────────────┐
│  CPU Core                                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │  L1 I-Cache (Instruction)  L1 D-Cache (Data)    │  │
│  └──────────────────────────────────────────────────┘  │
│              ↓                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              L2 Cache (Unified)                  │  │
│  └──────────────────────────────────────────────────┘  │
│              ↓                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              Main Memory (RAM)                   │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

Cache Coherency ปัญหา:
- D-Cache อาจมีข้อมูลใหม่กว่า RAM
- I-Cache อาจมีข้อมูลเก่ากว่า D-Cache
- จำเป็นต้อง flush/clean เพื่อ sync
```

### 12.2 Cache Operations ใน ARM

```asm
@ cache_ops.s — Cache Maintenance Operations

@ ARM Cache Maintenance Instructions (Privileged):
@ DC CIVAC: Data Cache Clean and Invalidate by VA to PoC
@ DC CVAC:  Data Cache Clean by VA to PoC
@ IC IVAU:  Instruction Cache Invalidate by VA to PoU
@ DCCISW:  Data Cache Clean and Invalidate by Set/Way

@ ใน User Mode: ใช้ syscall (cacheflush) แทน
@ Linux syscall #cacheflush ทำ cache maintenance ให้

.section .data
code_buffer: .fill 64, 1, 0     @ buffer สำหรับ JIT code

.section .text
.global _start

_start:
    LDR sp, =stack_top

    @ สมมติว่าเราเขียน ARM code ลงใน code_buffer
    LDR r0, =code_buffer

    @ เขียน instruction: MOV r0, #42 (0xE3A0002A)
    LDR r1, =0xE3A0002A
    STR r1, [r0]

    @ เขียน instruction: BX lr (0xE12FFF1E)
    LDR r1, =0xE12FFF1E
    STR r1, [r0, #4]

    @ Flush caches (Linux syscall)
    @ syscall #cacheflush: r0=start, r1=end, r2=flags
    LDR r0, =code_buffer        @ start address
    LDR r1, =code_buffer
    ADD r1, r1, #64             @ end address
    MOV r2, #0                  @ flags = 0
    MOV r7, #0xF0002            @ syscall: ARM_cacheflush (0xF0002)
    SVC #0

    @ ตอนนี้ safe ที่จะเรียก code_buffer เหมือน function
    LDR r0, =code_buffer
    BLX r0                      @ เรียก JIT-compiled code
    @ r0 = 42

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

---

## 13. AArch64 — Advanced Load/Store

### 13.1 AArch64 Addressing Modes

```asm
// arm64_addressing.s — AArch64 Addressing Modes

.section .data
.align 8
array: .quad 100, 200, 300, 400, 500

.section .text
.global _start

_start:
    adrp    x0, array
    add     x0, x0, :lo12:array

    // 1. Base Register Only
    ldr     x1, [x0]               // โหลดจาก [x0]

    // 2. Unsigned Offset (Immediate)
    ldr     x2, [x0, #8]           // โหลดจาก [x0 + 8]
    ldr     x3, [x0, #16]          // โหลดจาก [x0 + 16]

    // 3. Pre-indexed
    ldr     x4, [x0, #8]!          // x0 += 8, โหลดจาก [x0]

    // 4. Post-indexed
    ldr     x5, [x0], #8           // โหลดจาก [x0], x0 += 8

    // 5. Register Offset
    mov     x6, #2
    lsl     x6, x6, #3             // x6 = 2 * 8 = 16 (index * sizeof(uint64))
    ldr     x7, [x0, x6]           // โหลดจาก [x0 + x6]

    // 6. Scaled Register Offset
    mov     x6, #3                 // index = 3
    ldr     x8, [x0, x6, lsl #3]  // โหลดจาก [x0 + x6*8]
    // = array[3] = 400

    // 7. SXTW (Sign-extend Word to doubleword)
    mov     w6, #-1                // w6 = -1 (negative index)
    ldr     x9, [x0, w6, sxtw #3] // โหลดจาก [x0 + (int32)w6 * 8]
    // = array[-1] = element ก่อน array

    mov     x8, #93
    mov     x0, #0
    svc     #0
```

### 13.2 AArch64 — Exclusive Access (LDXR/STXR)

```asm
// arm64_exclusive.s — Atomic Operations ใน AArch64

.section .data
.align 8
shared_value: .quad 0

.section .text
.global _start

// Atomic Compare-And-Swap (CAS)
// input:  x0 = address, x1 = expected, x2 = desired
// output: x0 = 0 (success), 1 (failure)
// x3 = old value
cas:
    ldxr    x3, [x0]           // Load exclusive
    cmp     x3, x1             // ตรวจสอบว่าตรงกับที่คาดหวัง
    b.ne    cas_fail            // ถ้าไม่ตรง ล้มเหลว

    stxr    w4, x2, [x0]       // Store exclusive
    cbnz    w4, cas             // ถ้า store ล้มเหลว ลองใหม่

    mov     x0, #0             // success
    ret

cas_fail:
    clrex                      // ล้าง exclusive monitor
    mov     x0, #1             // failure
    ret

// Atomic Increment
// input:  x0 = address
atomic_inc:
retry:
    ldxr    x1, [x0]           // Load exclusive
    add     x1, x1, #1         // เพิ่มค่า
    stxr    w2, x1, [x0]       // Store exclusive
    cbnz    w2, retry           // ลองใหม่ถ้าล้มเหลว
    ret

_start:
    adrp    x0, shared_value
    add     x0, x0, :lo12:shared_value

    // Atomic increment 5 ครั้ง
    bl      atomic_inc
    bl      atomic_inc
    bl      atomic_inc
    bl      atomic_inc
    bl      atomic_inc

    // CAS: เปลี่ยนจาก 5 → 100
    adrp    x0, shared_value
    add     x0, x0, :lo12:shared_value
    mov     x1, #5             // expected = 5
    mov     x2, #100           // desired = 100
    bl      cas
    // x0 = 0 (success)

    mov     x8, #93
    mov     x0, #0
    svc     #0
```

---

## 14. โปรแกรมตัวอย่างสมบูรณ์

### 14.1 โปรแกรม: Memory Copy Optimized

```asm
@ optimized_memcpy.s — memcpy ที่ optimize ด้วย LDM/STM

@ void *memcpy(void *dst, const void *src, size_t n)
@ r0 = dst, r1 = src, r2 = n
@ ส่งคืน r0 = dst

.section .text
.global my_memcpy

my_memcpy:
    PUSH {r4-r11, lr}       @ บันทึก registers
    
    MOV r12, r0             @ เก็บ dst สำหรับ return

    @ ถ้า n < 32 ใช้ byte copy
    CMP r2, #32
    BLT byte_copy_start

    @ Copy ทีละ 32 bytes โดยใช้ LDM/STM
aligned_32_loop:
    CMP r2, #32
    BLT byte_copy_start

    LDMIA r1!, {r3-r10}     @ โหลด 8 words (32 bytes) จาก src
    STMIA r0!, {r3-r10}     @ เก็บ 8 words ลง dst
    SUB r2, r2, #32
    B aligned_32_loop

byte_copy_start:
    @ Copy ที่เหลือ ทีละ byte
byte_copy_loop:
    CMP r2, #0
    BEQ memcpy_done
    LDRB r3, [r1], #1       @ โหลด byte จาก src
    STRB r3, [r0], #1       @ เก็บ byte ลง dst
    SUB r2, r2, #1
    B byte_copy_loop

memcpy_done:
    MOV r0, r12             @ return dst
    POP {r4-r11, pc}

@ ทดสอบ
.section .data
src_buf: .ascii "Hello, ARM Assembly World!\n"
src_len = . - src_buf

.section .bss
dst_buf: .skip 64

.section .text
.global _start

_start:
    LDR sp, =stack_top

    LDR r0, =dst_buf        @ dst
    LDR r1, =src_buf        @ src
    MOV r2, #src_len        @ n

    BL my_memcpy

    @ print dst_buf
    MOV r0, #1              @ stdout
    LDR r1, =dst_buf
    MOV r2, #src_len
    MOV r7, #4              @ write syscall
    SVC #0

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

### 14.2 โปรแกรม: Bubble Sort ด้วย LDR/STR

```asm
@ bubble_sort.s — Bubble Sort ด้วย ARM Assembly

.section .data
numbers: .word 64, 34, 25, 12, 22, 11, 90
count: .word 7

.section .text
.global _start

@ void bubble_sort(int *arr, int n)
@ r0 = array address, r1 = count
bubble_sort:
    PUSH {r4-r9, lr}

    SUB r1, r1, #1          @ n - 1 สำหรับ outer loop

outer_loop:
    CMP r1, #0
    BLE sort_done
    
    MOV r4, #0              @ swapped = false
    MOV r5, r0              @ pointer to array[0]
    MOV r6, r1              @ inner counter

inner_loop:
    CMP r6, #0
    BLE outer_end

    LDR r7, [r5]            @ arr[i]
    LDR r8, [r5, #4]        @ arr[i+1]

    CMP r7, r8
    BLE no_swap             @ ถ้า arr[i] <= arr[i+1] ไม่ต้อง swap

    @ Swap arr[i] and arr[i+1]
    STR r8, [r5]            @ arr[i] = arr[i+1]
    STR r7, [r5, #4]        @ arr[i+1] = arr[i]
    MOV r4, #1              @ swapped = true

no_swap:
    ADD r5, r5, #4          @ เลื่อน pointer
    SUB r6, r6, #1
    B inner_loop

outer_end:
    CMP r4, #0              @ ถ้าไม่มี swap แสดงว่าเรียงแล้ว
    BEQ sort_done
    SUB r1, r1, #1
    B outer_loop

sort_done:
    POP {r4-r9, pc}

_start:
    LDR sp, =stack_top

    LDR r0, =numbers
    LDR r1, =count
    LDR r1, [r1]            @ r1 = 7

    BL bubble_sort
    @ numbers ถูกเรียงแล้ว: 11, 12, 22, 25, 34, 64, 90

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

### 14.3 AArch64 — String Operations

```asm
// arm64_string_ops.s — String Operations ใน AArch64

.section .data
str1:   .asciz "Hello, AArch64!"
str2:   .fill 32, 1, 0

.section .text
.global _start

// strlen: นับความยาวสตริง
// input:  x0 = string address
// output: x0 = length
strlen:
    mov     x1, x0              // สำเนา pointer
strlen_loop:
    ldrb    w2, [x1], #1        // โหลด byte และ post-increment
    cbnz    w2, strlen_loop      // วนถ้าไม่ใช่ null
    sub     x0, x1, x0          // length = end - start
    sub     x0, x0, #1          // ลบ 1 เพราะนับ null
    ret

// strcpy: คัดลอกสตริง
// input:  x0 = dst, x1 = src
strcpy:
    mov     x2, x0              // เก็บ dst
strcpy_loop:
    ldrb    w3, [x1], #1        // โหลด char จาก src
    strb    w3, [x0], #1        // เก็บ char ลง dst
    cbnz    w3, strcpy_loop      // วนถ้าไม่ใช่ null
    mov     x0, x2              // return dst
    ret

// strcmp: เปรียบเทียบสตริง
// input:  x0 = s1, x1 = s2
// output: x0 = 0 (equal), >0 (s1 > s2), <0 (s1 < s2)
strcmp:
strcmp_loop:
    ldrb    w2, [x0], #1        // char จาก s1
    ldrb    w3, [x1], #1        // char จาก s2
    cmp     w2, w3
    b.ne    strcmp_diff
    cbnz    w2, strcmp_loop      // วนถ้าไม่ใช่ null
    mov     x0, #0              // เท่ากัน
    ret
strcmp_diff:
    sub     x0, x2, x3          // return ผลต่าง
    ret

_start:
    // strlen
    adrp    x0, str1
    add     x0, x0, :lo12:str1
    bl      strlen
    // x0 = 15

    // strcpy
    adrp    x0, str2
    add     x0, x0, :lo12:str2
    adrp    x1, str1
    add     x1, x1, :lo12:str1
    bl      strcpy

    // strcmp
    adrp    x0, str1
    add     x0, x0, :lo12:str1
    adrp    x1, str2
    add     x1, x1, :lo12:str2
    bl      strcmp
    // x0 = 0 (เท่ากัน)

    mov     x8, #93
    mov     x0, #0
    svc     #0
```

---

## 15. การ Compile และทดสอบด้วย QEMU

### 15.1 ARM32 Build Script

```bash
#!/bin/bash
# build_arm32.sh — สคริปต์สำหรับ build ARM32

# ติดตั้ง cross-compiler
# Ubuntu/Debian: sudo apt install gcc-arm-linux-gnueabi qemu-user

FILENAME=$1
BASENAME="${FILENAME%.s}"

echo "=== Building ARM32: $FILENAME ==="

# Assemble
arm-linux-gnueabi-as \
    -march=armv7-a \
    "$FILENAME" \
    -o "${BASENAME}.o" \
    && echo "Assembled OK" || { echo "Assembly failed"; exit 1; }

# Link
arm-linux-gnueabi-ld \
    "${BASENAME}.o" \
    -o "${BASENAME}" \
    && echo "Linked OK" || { echo "Linking failed"; exit 1; }

# Run with QEMU
echo "=== Running with QEMU ==="
qemu-arm \
    -L /usr/arm-linux-gnueabi \
    "./${BASENAME}"

echo "Exit code: $?"
```

### 15.2 AArch64 Build Script

```bash
#!/bin/bash
# build_arm64.sh — สคริปต์สำหรับ build AArch64

# ติดตั้ง: sudo apt install gcc-aarch64-linux-gnu qemu-user

FILENAME=$1
BASENAME="${FILENAME%.s}"

echo "=== Building AArch64: $FILENAME ==="

# Assemble
aarch64-linux-gnu-as \
    "$FILENAME" \
    -o "${BASENAME}.o" \
    && echo "Assembled OK" || { echo "Assembly failed"; exit 1; }

# Link
aarch64-linux-gnu-ld \
    "${BASENAME}.o" \
    -o "${BASENAME}" \
    && echo "Linked OK" || { echo "Linking failed"; exit 1; }

# Run with QEMU
echo "=== Running with QEMU ==="
qemu-aarch64 \
    -L /usr/aarch64-linux-gnu \
    "./${BASENAME}"

echo "Exit code: $?"
```

### 15.3 Makefile สำหรับ Project

```makefile
# Makefile — ARM Assembly Project

# Cross-compilers
AS32 = arm-linux-gnueabi-as
LD32 = arm-linux-gnueabi-ld
AS64 = aarch64-linux-gnu-as
LD64 = aarch64-linux-gnu-ld

# QEMU
QEMU32 = qemu-arm
QEMU64 = qemu-aarch64

# Flags
ASFLAGS32 = -march=armv7-a
ASFLAGS64 =

# Targets
ARM32_SRCS = $(wildcard arm32_*.s)
ARM32_BINS = $(ARM32_SRCS:.s=)

ARM64_SRCS = $(wildcard arm64_*.s)
ARM64_BINS = $(ARM64_SRCS:.s=)

.PHONY: all clean test32 test64

all: $(ARM32_BINS) $(ARM64_BINS)

# Rule สำหรับ ARM32
%: %.s
	$(AS32) $(ASFLAGS32) $< -o $@.o
	$(LD32) $@.o -o $@

# Rule สำหรับ AArch64
arm64_%: arm64_%.s
	$(AS64) $(ASFLAGS64) $< -o $@.o
	$(LD64) $@.o -o $@

test32: $(ARM32_BINS)
	@for bin in $(ARM32_BINS); do \
		echo "Testing $$bin..."; \
		$(QEMU32) ./$$bin; \
		echo "Exit: $$?"; \
	done

test64: $(ARM64_BINS)
	@for bin in $(ARM64_BINS); do \
		echo "Testing $$bin..."; \
		$(QEMU64) ./$$bin; \
		echo "Exit: $$?"; \
	done

clean:
	rm -f *.o $(ARM32_BINS) $(ARM64_BINS)
```

### 15.4 Debug ด้วย GDB

```bash
# Debug ARM32 ด้วย QEMU + GDB

# Terminal 1: รัน QEMU ใน debug mode
qemu-arm -g 1234 ./arm32_word_copy

# Terminal 2: เชื่อมต่อ GDB
arm-linux-gnueabi-gdb arm32_word_copy

# GDB commands:
(gdb) target remote :1234
(gdb) break _start
(gdb) continue
(gdb) stepi          # execute 1 instruction
(gdb) info registers # ดู registers ทั้งหมด
(gdb) x/10xw $r0     # ดู memory ที่ r0 (10 words, hex)
(gdb) x/16xb $sp     # ดู 16 bytes ที่ stack
(gdb) layout asm     # แสดง assembly view
(gdb) layout regs    # แสดง register view
```

---

## 16. แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Basic LDR/STR (ระดับ: ง่าย)

```
โจทย์: เขียนโปรแกรม ARM32 ที่:
1. มีอาร์เรย์ integer 10 ตัว
2. หาค่า max, min, และ sum
3. เก็บผลลัพธ์ลงตัวแปรแยกกัน
4. Print ผลลัพธ์ผ่าน write syscall (แปลงเป็น ASCII ก่อน)

Hint:
- ใช้ LDR/STR สำหรับ word access
- ใช้ CMP + BLT/BGT สำหรับ comparison
- ใช้ post-indexed addressing ในการ traverse array
```

### แบบฝึกหัดที่ 2: LDRSB/LDRSH (ระดับ: ปานกลาง)

```
โจทย์: เขียน function ที่รับ array ของ signed bytes
       แล้ว normalize ให้อยู่ในช่วง 0-255:
       normalized = (value - min) * 255 / (max - min)

int8_t input[] = {-128, -64, 0, 64, 127, -10, 50};

ต้องใช้:
- LDRSB สำหรับอ่านค่า signed byte
- STRB สำหรับเขียน normalized value
- ARM arithmetic instructions
```

### แบบฝึกหัดที่ 3: LDM/STM Performance (ระดับ: ปานกลาง)

```
โจทย์: เปรียบเทียบ performance ระหว่าง:
1. copy ทีละ byte (LDRB/STRB)
2. copy ทีละ word (LDR/STR)
3. copy ทีละ 8 words (LDMIA/STMIA)

สำหรับ buffer ขนาด 1MB
วัดเวลาด้วย read_timer หรือนับรอบ

ผลที่คาดหวัง: วิธี 3 เร็วที่สุด (8x หรือมากกว่า)
```

### แบบฝึกหัดที่ 4: Stack-based Expressions (ระดับ: ยาก)

```
โจทย์: เขียน RPN (Reverse Polish Notation) Calculator
โดยใช้ ARM stack (PUSH/POP)

Input: สตริง "3 4 + 2 * 7 -" (= (3+4)*2-7 = 7)
Output: ผลลัพธ์ = 7

Operations: +, -, *, /
Operands: 1-2 digit numbers

ต้องใช้:
- PUSH r0 สำหรับ push operand
- POP r0, POP r1 สำหรับดึง operands
- PUSH ผลลัพธ์กลับ
- STM/LDM สำหรับ state management
```

### แบบฝึกหัดที่ 5: AArch64 SIMD-like with LDP (ระดับ: ยาก)

```
โจทย์: เขียน dot product ของ 2 vectors (64 floats each)
โดยใช้ LDP เพื่อโหลด 2 floats พร้อมกัน

เนื่องจากยังไม่ใช้ NEON/SIMD ให้ใช้ LDP กับ integer operations:
- รับ 2 int64_t arrays
- คำนวณ dot product
- ใช้ LDP x1, x2, [x0] เพื่อโหลด 2 elements พร้อมกัน

Bonus: เปรียบเทียบ performance กับ LDR แบบ sequential
```

### เฉลยแบบฝึกหัดที่ 1

```asm
@ exercise1_solution.s — หา max, min, sum ของ array

.section .data
numbers: .word 42, 17, 93, 5, 78, 31, 66, 9, 55, 24
count:   .word 10

result_max: .word 0
result_min: .word 0
result_sum: .word 0

.section .text
.global _start

_start:
    LDR sp, =stack_top

    LDR r0, =numbers        @ pointer ไปยัง array
    LDR r1, =count
    LDR r1, [r1]            @ r1 = 10

    @ โหลด element แรกเป็นค่าเริ่มต้น
    LDR r2, [r0], #4        @ r2 = numbers[0], r0 → numbers[1]
    MOV r3, r2              @ r3 = max = numbers[0]
    MOV r4, r2              @ r4 = min = numbers[0]
    MOV r5, r2              @ r5 = sum = numbers[0]
    SUB r1, r1, #1          @ เหลือ 9 elements

find_loop:
    CMP r1, #0
    BEQ find_done

    LDR r2, [r0], #4        @ อ่าน element ถัดไป

    @ Update max
    CMP r2, r3
    BLE not_max
    MOV r3, r2              @ r3 = new max
not_max:

    @ Update min
    CMP r2, r4
    BGE not_min
    MOV r4, r2              @ r4 = new min
not_min:

    @ Update sum
    ADD r5, r5, r2

    SUB r1, r1, #1
    B find_loop

find_done:
    LDR r0, =result_max
    STR r3, [r0]            @ เก็บ max
    LDR r0, =result_min
    STR r4, [r0]            @ เก็บ min
    LDR r0, =result_sum
    STR r5, [r0]            @ เก็บ sum

    @ ผลลัพธ์: max=93, min=5, sum=420

    MOV r7, #1
    MOV r0, #0
    SVC #0

.section .bss
.align 12
stack_space: .skip 4096
stack_top:
```

---

## 17. สรุปคำสั่ง Load/Store ทั้งหมด

### 17.1 ตารางสรุป ARM32

```
┌────────────────┬──────────────────────────────────────────────────────┐
│ คำสั่ง         │ ความหมาย                                              │
├────────────────┼──────────────────────────────────────────────────────┤
│ LDR Rd,[Rn]   │ Load 32-bit word                                     │
│ STR Rd,[Rn]   │ Store 32-bit word                                    │
│ LDRB Rd,[Rn]  │ Load byte (zero-extend)                              │
│ STRB Rd,[Rn]  │ Store byte (bits 7:0)                                │
│ LDRH Rd,[Rn]  │ Load halfword (zero-extend)                          │
│ STRH Rd,[Rn]  │ Store halfword (bits 15:0)                           │
│ LDRSB Rd,[Rn] │ Load signed byte (sign-extend)                       │
│ LDRSH Rd,[Rn] │ Load signed halfword (sign-extend)                   │
│ LDRD Rd,[Rn]  │ Load doubleword (Rd even, Rd+1 odd)                  │
│ STRD Rd,[Rn]  │ Store doubleword                                     │
│ LDM{mode}     │ Load multiple registers                               │
│ STM{mode}     │ Store multiple registers                              │
│ PUSH {regs}   │ = STMDB SP!, {regs}                                  │
│ POP  {regs}   │ = LDMIA SP!, {regs}                                  │
│ LDREX Rd,[Rn] │ Load exclusive (for atomic ops)                      │
│ STREX Rm,Rd,[Rn]│ Store exclusive                                    │
├────────────────┼──────────────────────────────────────────────────────┤
│ Memory Barriers│                                                      │
│ DMB            │ Data Memory Barrier                                  │
│ DSB            │ Data Synchronization Barrier                        │
│ ISB            │ Instruction Synchronization Barrier                  │
└────────────────┴──────────────────────────────────────────────────────┘
```

### 17.2 ตารางสรุป AArch64

```
┌────────────────────┬────────────────────────────────────────────────────┐
│ คำสั่ง             │ ความหมาย                                            │
├────────────────────┼────────────────────────────────────────────────────┤
│ LDR Xt,[Xn]       │ Load 64-bit                                        │
│ LDR Wt,[Xn]       │ Load 32-bit (zero-extend to 64)                    │
│ STR Xt,[Xn]       │ Store 64-bit                                       │
│ STR Wt,[Xn]       │ Store 32-bit                                       │
│ LDRB Wt,[Xn]      │ Load byte (zero-extend)                            │
│ STRB Wt,[Xn]      │ Store byte                                         │
│ LDRH Wt,[Xn]      │ Load halfword (zero-extend)                        │
│ STRH Wt,[Xn]      │ Store halfword                                     │
│ LDRSB Xt,[Xn]     │ Load signed byte → 64-bit                          │
│ LDRSB Wt,[Xn]     │ Load signed byte → 32-bit                          │
│ LDRSH Xt/Wt,[Xn]  │ Load signed halfword                               │
│ LDRSW Xt,[Xn]     │ Load signed word → 64-bit                          │
│ LDP Xt1,Xt2,[Xn]  │ Load Pair (2 x 64-bit)                             │
│ STP Xt1,Xt2,[Xn]  │ Store Pair                                         │
│ LDXR Xt,[Xn]      │ Load exclusive                                     │
│ STXR Ws,Xt,[Xn]   │ Store exclusive                                    │
│ LDAXR/STLXR       │ Load/Store Acquire/Release (stronger ordering)     │
└────────────────────┴────────────────────────────────────────────────────┘
```

### 17.3 เปรียบเทียบ Addressing Modes

```
Addressing Mode    │ ARM32                │ AArch64
───────────────────┼──────────────────────┼─────────────────────
Base only          │ [Rn]                 │ [Xn]
Immediate offset   │ [Rn, #imm]           │ [Xn, #imm]
Register offset    │ [Rn, Rm]             │ [Xn, Xm]
Scaled reg offset  │ [Rn, Rm, LSL #n]     │ [Xn, Xm, LSL #n]
Pre-indexed        │ [Rn, #imm]!          │ [Xn, #imm]!
Post-indexed       │ [Rn], #imm           │ [Xn], #imm
```

---

## 18. Tips และ Best Practices

### 18.1 การ Align หน่วยความจำ

```asm
@ เสมอ align ข้อมูลให้เหมาะสม เพื่อ performance
.section .data

.align 2        @ halfword ต้องการ 2-byte align
hw: .hword 0x1234

.align 4        @ word ต้องการ 4-byte align
wd: .word 0xDEADBEEF

.align 8        @ doubleword ต้องการ 8-byte align
dw: .quad 0x123456789ABCDEF0

.align 4        @ ปกติ .text section align อย่างน้อย 4 bytes
.text
```

### 18.2 การ Optimize Loop ด้วย Post-indexed

```asm
@ ไม่ดี (ใช้ index variable แยก)
    MOV r0, #0              @ index
loop_bad:
    LSL r1, r0, #2          @ byte offset = index * 4
    LDR r2, [r3, r1]        @ โหลด array[index]
    ADD r0, r0, #1
    CMP r0, #10
    BLT loop_bad

@ ดีกว่า (ใช้ post-indexed pointer)
    LDR r0, =array
    MOV r1, #10
loop_good:
    LDR r2, [r0], #4        @ โหลดและเลื่อน pointer พร้อมกัน
    SUBS r1, r1, #1
    BNE loop_good
```

### 18.3 การ Save/Restore Registers อย่างถูกต้อง

```asm
@ ARM ABI: r4-r11 ต้องคืนค่าเมื่อออกจาก function
@ r0-r3 สำหรับ arguments/return (caller-saved)

correct_function:
    PUSH {r4, r5, r6, lr}   @ บันทึก callee-saved + LR

    @ ใช้ r4, r5, r6 ใน function body
    MOV r4, #100
    MOV r5, #200
    ADD r6, r4, r5

    @ return value ใน r0
    MOV r0, r6

    POP {r4, r5, r6, pc}    @ คืนค่า regs และ return
```

---

## บทสรุป

ใน Part 033 นี้เราได้เรียนรู้คำสั่ง Load/Store ทั้งหมดของ ARM:

**พื้นฐาน:**
- LDR/STR (32-bit), LDRB/STRB (8-bit), LDRH/STRH (16-bit)
- LDRSB/LDRSH สำหรับ signed values
- LDRD/STRD สำหรับ 64-bit ใน ARM32

**Addressing Modes:**
- Immediate offset: [Rn, #imm]
- Register offset: [Rn, Rm] หรือ [Rn, Rm, LSL #n]
- Pre-indexed: [Rn, #imm]!
- Post-indexed: [Rn], #imm

**Multiple Registers:**
- LDM/STM ทุก modes (IA, IB, DA, DB)
- PUSH/POP เป็น alias ของ STMDB/LDMIA

**Synchronization:**
- LDREX/STREX สำหรับ atomic operations
- DMB/DSB/ISB memory barriers

**AArch64:**
- LDP/STP สำหรับ load/store คู่
- LDXR/STXR exclusive access

**Part ถัดไป:** Part 034 — ARM NEON/SIMD Instructions

---

*Part 033 — ARM Load/Store Instructions*  
*หลักสูตร Assembly Programming จากพื้นฐานสู่ระดับมืออาชีพ*
