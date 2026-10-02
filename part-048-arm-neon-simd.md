# Part 048: ARM NEON SIMD

## บทนำ (Introduction)

ARM NEON คือ Advanced SIMD (Single Instruction, Multiple Data) extension สำหรับ ARM processors ที่ช่วยให้สามารถประมวลผลข้อมูลหลายชิ้นพร้อมกันได้ในคำสั่งเดียว เหมาะสำหรับงาน multimedia, signal processing, machine learning, และ cryptography

NEON มีให้ใน:
- ARMv7-A/R (ARM32, Thumb-2) — optional แต่พบได้ทั่วไป
- ARMv8-A (AArch64) — standard, เรียกว่า Advanced SIMD หรือ ASIMD

บทนี้จะเน้น **ARM32 (ARMv7) NEON** ด้วย GAS syntax (GNU Assembler) สำหรับ Linux

---

## 1. NEON Register File

### 1.1 โครงสร้าง Register

NEON ใช้ register bank เดียวกับ VFP (Floating-Point Unit) แต่มองต่างกัน:

```
Register Bank (256 bytes = 2048 bits)
┌─────────────────────────────────────────────────────────────────┐
│  Q0  (128-bit) = D1        (64-bit)  | D0        (64-bit)       │
│  Q1  (128-bit) = D3        (64-bit)  | D2        (64-bit)       │
│  Q2  (128-bit) = D5        (64-bit)  | D4        (64-bit)       │
│  Q3  (128-bit) = D7        (64-bit)  | D6        (64-bit)       │
│  ...                                                              │
│  Q15 (128-bit) = D31       (64-bit)  | D30       (64-bit)       │
└─────────────────────────────────────────────────────────────────┘
```

**D registers (64-bit Doubleword):**
- D0 ถึง D31 (32 registers)
- แต่ละ D register = 64 bits
- ใช้คู่กับ VFP (D0-D15) และ NEON (D0-D31)

**Q registers (128-bit Quadword):**
- Q0 ถึง Q15 (16 registers)
- แต่ละ Q register = 128 bits = 2 × D registers
- Q0 = {D1, D0}, Q1 = {D3, D2}, ..., Qn = {D(2n+1), D(2n)}

### 1.2 NEON Data Types

NEON support หลาย data types — ตัวเลขในชื่อคือจำนวน elements:

| Type Suffix | Element Size | D register | Q register | Description              |
|-------------|-------------|------------|------------|--------------------------|
| `.8B`       | 8-bit       | 8 elements | —          | 8 × byte (signed/unsigned)|
| `.16B`      | 8-bit       | —          | 16 elements| 16 × byte                |
| `.4H`       | 16-bit      | 4 elements | —          | 4 × halfword             |
| `.8H`       | 16-bit      | —          | 8 elements | 8 × halfword             |
| `.2S`       | 32-bit      | 2 elements | —          | 2 × word                 |
| `.4S`       | 32-bit      | —          | 4 elements | 4 × word                 |
| `.1D`       | 64-bit      | 1 element  | —          | 1 × doubleword           |
| `.2D`       | 64-bit      | —          | 2 elements | 2 × doubleword           |
| `.2H`       | 16-bit      | 2 elements | —          | (less common)            |
| `.1S`       | 32-bit      | 1 element  | —          | (less common)            |

ตัวอย่างการตีความ Q0 เป็น `.4S` (4 × 32-bit):
```
Q0 = [ S3 | S2 | S1 | S0 ]
       bit127    ...    bit0
```

ตีความเป็น `.8H` (8 × 16-bit):
```
Q0 = [ H7 | H6 | H5 | H4 | H3 | H2 | H1 | H0 ]
       bit127                               bit0
```

### 1.3 VFP vs NEON

| Feature           | VFP (VFPv3/v4)          | NEON                          |
|-------------------|-------------------------|-------------------------------|
| Purpose           | Scalar float arithmetic | SIMD (packed) arithmetic      |
| Register width    | 32-bit (S) / 64-bit (D) | 64-bit (D) / 128-bit (Q)     |
| Float support     | F32, F64                | F32 เท่านั้น (ARM32)          |
| Integer SIMD      | ไม่มี                   | I8, I16, I32, I64             |
| Throughput        | 1 op/cycle              | 2–16 ops/cycle                |
| Shared registers  | ใช่ (D0-D15 ร่วมกัน)   | ใช่ (D0-D31)                  |

> **หมายเหตุ:** เมื่อเปิดใช้ NEON, register S0-S31 ยังใช้ได้สำหรับ VFP scalar แต่ D16-D31 ใช้ได้เฉพาะกับ NEON เท่านั้น

---

## 2. NEON Load/Store

### 2.1 VLD1 / VST1 — Load/Store 1 Register

VLD1 (Vector Load 1) โหลดข้อมูลจาก memory ลง NEON register(s) โดย **ไม่** interleave

```asm
@ Syntax:
@   VLD1.<type> {<list>}, [<Rn>{:<align>}]{!}
@   VST1.<type> {<list>}, [<Rn>{:<align>}]{!}

@ โหลด 8 bytes (8 × I8) ลง D0
VLD1.8  {D0}, [R0]

@ โหลด 16 bytes (16 × I8) ลง Q0 (= D0 + D1)
VLD1.8  {Q0}, [R0]

@ โหลด 2 registers (D0, D1) = 16 bytes
VLD1.8  {D0-D1}, [R0]

@ โหลดพร้อม address update (R0 += 16 หลังโหลด)
VLD1.8  {D0-D1}, [R0]!

@ โหลดพร้อม alignment hint (64-bit aligned)
VLD1.64 {D0-D1}, [R0:64]

@ โหลด 4 registers (32 bytes)
VLD1.8  {D0-D3}, [R0]!

@ เก็บ 8 × I16 จาก D0 ลง memory
VST1.16 {D0}, [R1]!
```

### 2.2 VLD2 / VST2 — Interleaved 2 Registers

VLD2 โหลดข้อมูลแบบ de-interleave: element 0, 2, 4... ไป register 1; element 1, 3, 5... ไป register 2

```
Memory: [ A0 B0 A1 B1 A2 B2 A3 B3 ]
After VLD2.8 {D0, D1}, [R0]:
  D0 = [ A3 A2 A1 A0 ... ]   (even elements)
  D1 = [ B3 B2 B1 B0 ... ]   (odd elements)
```

ตัวอย่างใช้งาน (เช่น stereo audio ที่ L/R interleaved):
```asm
@ stereo PCM: [ L0 R0 L1 R1 L2 R2 L3 R3 ... ]
@ โหลด 8 samples (4 L + 4 R)
VLD2.16 {D0, D1}, [R0]!
@ D0 = { L3, L2, L1, L0 }  (4 × I16)
@ D1 = { R3, R2, R1, R0 }  (4 × I16)

@ process แยก channel...

@ เก็บกลับแบบ interleave
VST2.16 {D0, D1}, [R1]!
```

### 2.3 VLD3 / VST3 — Interleaved 3 Registers (RGB!)

VLD3 เหมาะมากสำหรับ RGB pixel data:
```
Memory: [ R0 G0 B0 R1 G1 B1 R2 G2 B2 R3 G3 B3 ... ]
After VLD3.8 {D0, D1, D2}, [R0]!:
  D0 = { R7..R0 }   (R channel)
  D1 = { G7..G0 }   (G channel)
  D2 = { B7..B0 }   (B channel)
```

```asm
@ โหลด 8 RGB pixels
VLD3.8  {D0, D1, D2}, [R0]!
@ D0 = R channel (8 pixels)
@ D1 = G channel
@ D2 = B channel

@ process channels...
@ เก็บกลับ
VST3.8  {D0, D1, D2}, [R1]!
```

### 2.4 VLD4 / VST4 — Interleaved 4 Registers (RGBA!)

VLD4 สำหรับ RGBA pixel data:
```
Memory: [ R0 G0 B0 A0 R1 G1 B1 A1 ... ]
After VLD4.8 {D0, D1, D2, D3}, [R0]!:
  D0 = { R7..R0 }
  D1 = { G7..G0 }
  D2 = { B7..B0 }
  D3 = { A7..A0 }
```

```asm
@ โหลด 8 RGBA pixels (32 bytes)
VLD4.8  {D0, D1, D2, D3}, [R0]!

@ เช่น ปรับ alpha channel (D3):
VMOV.I8  D4, #200         @ alpha = 200
VMOV     D3, D4           @ set all alpha

@ เก็บกลับ
VST4.8  {D0, D1, D2, D3}, [R1]!
```

### 2.5 VLD1 — Load Single Element

```asm
@ โหลด 1 element เข้า lane ที่ระบุ
@ VLD1.<type> {<Dn>[<index>]}, [<Rn>]

VLD1.32 {D0[0]}, [R0]    @ โหลด 32-bit เข้า D0 bits[31:0]
VLD1.32 {D0[1]}, [R0]    @ โหลด 32-bit เข้า D0 bits[63:32]
VLD1.16 {D2[3]}, [R1]    @ โหลด 16-bit เข้า D2 lane 3
VLD1.8  {D4[7]}, [R2]    @ โหลด 8-bit เข้า D4 lane 7
```

### 2.6 VLDM / VSTM — Load/Store Multiple

```asm
@ VLDM: โหลด VFP/NEON registers หลายตัวติดกัน
VLDMIA  R0, {D0-D7}       @ โหลด D0-D7 จาก [R0], R0 ไม่เปลี่ยน
VLDMIA  R0!, {D0-D7}      @ โหลดและ R0 += 64
VLDMDB  R0!, {D0-D7}      @ โหลด decrement before

VSTMIA  R1!, {D0-D7}      @ เก็บ D0-D7, R1 += 64
VSTMDB  R1!, {D0-D7}      @ เก็บ decrement before (stack-like)

@ ใช้บน stack:
PUSH    {LR}
VPUSH   {D8-D15}          @ save callee-saved NEON registers
@ ... code ...
VPOP    {D8-D15}
POP     {PC}
```

### 2.7 Address Update Modes

```asm
@ No update (base unchanged)
VLD1.8  {D0}, [R0]

@ Post-increment (!) — R0 += size_of_transfer
VLD1.8  {D0}, [R0]!       @ R0 += 8

@ Post-increment by register
VLD1.8  {D0}, [R0], R2    @ R0 += R2

@ สำหรับ Q register
VLD1.32 {Q0}, [R0]!       @ R0 += 16

@ สำหรับ 4 registers
VLD1.8  {D0-D3}, [R0]!    @ R0 += 32
```

---

## 3. NEON Arithmetic

### 3.1 VADD — Packed Add

```asm
@ VADD.<type> <Vd>, <Vn>, <Vm>
@ ทำ Vd[i] = Vn[i] + Vm[i] สำหรับทุก lane

@ Integer add:
VADD.I8   D0, D1, D2      @ 8 × 8-bit add  (D register)
VADD.I16  Q0, Q1, Q2      @ 8 × 16-bit add (Q register)
VADD.I32  D0, D1, D2      @ 2 × 32-bit add
VADD.I64  D0, D1, D2      @ 1 × 64-bit add

@ Float add:
VADD.F32  D0, D1, D2      @ 2 × float32 add
VADD.F32  Q0, Q1, Q2      @ 4 × float32 add

@ ตัวอย่าง: บวก 2 arrays ของ int16 (8 elements ต่อ iteration)
@ R0 = src_a, R1 = src_b, R2 = dst, R3 = count/8

add_loop:
    VLD1.16 {Q0}, [R0]!         @ โหลด 8 × I16 จาก a
    VLD1.16 {Q1}, [R1]!         @ โหลด 8 × I16 จาก b
    VADD.I16 Q2, Q0, Q1         @ บวกทีละ 8 elements
    VST1.16 {Q2}, [R2]!         @ เก็บผลลัพธ์
    SUBS    R3, R3, #1
    BNE     add_loop
```

### 3.2 VSUB — Packed Subtract

```asm
VSUB.I8   D0, D1, D2      @ D0[i] = D1[i] - D2[i]
VSUB.I16  Q0, Q1, Q2
VSUB.F32  Q0, Q1, Q2      @ 4 × float subtract
```

### 3.3 VMUL — Packed Multiply

```asm
@ Integer multiply (ผลลัพธ์ truncated):
VMUL.I16  D0, D1, D2      @ 4 × 16-bit * 16-bit -> 16-bit (low)
VMUL.I32  Q0, Q1, Q2      @ 4 × 32-bit multiply

@ Float multiply:
VMUL.F32  D0, D1, D2      @ 2 × float32
VMUL.F32  Q0, Q1, Q2      @ 4 × float32

@ Scalar multiply (ทุก lane คูณด้วย scalar เดียว):
@ VMUL.<type> <Vd>, <Vn>, <Vm[x]>
VMUL.F32  Q0, Q1, D2[0]   @ Q0[i] = Q1[i] * D2[0] (scalar)
VMUL.I16  D0, D1, D4[2]   @ D0[i] = D1[i] * D4[2] (lane 2)
```

### 3.4 VMLA / VMLS — Multiply-Accumulate / Multiply-Subtract

```asm
@ VMLA: Vd[i] = Vd[i] + Vn[i] * Vm[i]  (multiply-accumulate)
VMLA.F32  Q0, Q1, Q2      @ Q0[i] += Q1[i] * Q2[i]
VMLA.I16  D0, D1, D2      @ D0[i] += D1[i] * D2[i]

@ Scalar form:
VMLA.F32  Q0, Q1, D2[0]   @ Q0[i] += Q1[i] * scalar

@ VMLS: Vd[i] = Vd[i] - Vn[i] * Vm[i]  (multiply-subtract)
VMLS.F32  Q0, Q1, Q2      @ Q0[i] -= Q1[i] * Q2[i]

@ ตัวอย่าง: dot product 4 floats
@ สมมติ: Q1 = [a3,a2,a1,a0], Q2 = [b3,b2,b1,b0]
VMUL.F32  Q0, Q1, Q2      @ Q0[i] = a[i] * b[i]
VPADD.F32 D0, D0, D1      @ D0 = {D0[1]+D0[0], D1[1]+D1[0]}
VPADD.F32 D0, D0, D0      @ D0[0] = sum of all 4
@ ผลลัพธ์อยู่ใน D0[0] (S0)
```

### 3.5 VFMA / VFMS — Fused Multiply-Accumulate (VFPv4+NEON)

```asm
@ VFMA: Vd[i] = Vd[i] + Vn[i] * Vm[i]  (fused, no intermediate rounding)
@ ต้องการ VFPv4 / NEON with FMA support
VFMA.F32  Q0, Q1, Q2      @ Q0[i] += Q1[i] * Q2[i] (fused)
VFMS.F32  Q0, Q1, Q2      @ Q0[i] -= Q1[i] * Q2[i] (fused)
```

### 3.6 VABS / VNEG — Absolute Value / Negate

```asm
VABS.S8   D0, D1          @ D0[i] = |D1[i]|  (signed 8-bit)
VABS.S16  Q0, Q1
VABS.S32  D0, D1
VABS.F32  Q0, Q1          @ float absolute value

VNEG.S8   D0, D1          @ D0[i] = -D1[i]
VNEG.F32  Q0, Q1          @ float negate
```

### 3.7 VMAX / VMIN — Maximum / Minimum

```asm
@ Integer (signed and unsigned):
VMAX.S8   D0, D1, D2      @ D0[i] = max(D1[i], D2[i])  signed
VMAX.U8   D0, D1, D2      @ unsigned max
VMAX.S16  Q0, Q1, Q2
VMIN.S32  D0, D1, D2      @ min signed 32-bit

@ Float:
VMAX.F32  Q0, Q1, Q2      @ 4 × float max
VMIN.F32  Q0, Q1, Q2
```

### 3.8 VPADD — Pairwise Add

VPADD บวก pairs ของ adjacent elements ในแต่ละ register:

```
D0 = [a3, a2, a1, a0] (4 × I16)
D1 = [b3, b2, b1, b0]
VPADD.I16 D2, D0, D1
D2 = [b3+b2, b1+b0, a3+a2, a1+a0]
```

```asm
VPADD.I8  D0, D1, D2      @ 8 elements -> 4 elements (pairs summed)
VPADD.I16 D0, D1, D2
VPADD.I32 D0, D1, D2
VPADD.F32 D0, D1, D2

@ ตัวอย่าง: sum 8 × I16 elements
@ D0 = {d7, d6, d5, d4, d3, d2, d1, d0} — 8 × I8
@ สร้าง horizontal sum:
VPADD.I16 D0, D0, D0      @ D0 = {d7+d6, d5+d4, d3+d2, d1+d0}
VPADD.I16 D0, D0, D0      @ D0 = {X, X, (d7+..+d4), (d3+..+d0)}
VPADD.I16 D0, D0, D0      @ D0[0] = total sum
```

### 3.9 VPMAX / VPMIN — Pairwise Max/Min

```asm
VPMAX.S16 D0, D1, D2      @ max ของแต่ละ adjacent pair
VPMIN.U8  D0, D1, D2
VPMAX.F32 D0, D1, D2
```

### 3.10 VADDL / VSUBL — Long Add/Sub (Widening)

VADDL บวกและขยาย element size (result = 2× input size):

```asm
@ VADDL: D register input -> Q register output
VADDL.S8  Q0, D1, D2      @ Q0[i] = (I16)D1[i] + (I16)D2[i]
VADDL.U8  Q0, D1, D2      @ unsigned
VADDL.S16 Q0, D1, D2      @ I16 -> I32
VADDL.S32 Q0, D1, D2      @ I32 -> I64

VSUBL.S16 Q0, D1, D2      @ long subtract
```

### 3.11 VMULL — Long Multiply

```asm
@ คูณ + widening (input D, output Q)
VMULL.S8  Q0, D1, D2      @ I8 * I8 -> I16
VMULL.S16 Q0, D1, D2      @ I16 * I16 -> I32
VMULL.U16 Q0, D1, D2      @ unsigned
VMULL.S32 Q0, D1, D2      @ I32 * I32 -> I64

@ Polynomial multiply (for GCM, CRC):
VMULL.P8  Q0, D1, D2      @ polynomial over GF(2^8)
```

---

## 4. NEON Bitwise & Compare

### 4.1 Bitwise Operations

```asm
@ AND: Vd[i] = Vn[i] & Vm[i]
VAND    D0, D1, D2        @ 64-bit bitwise AND
VAND    Q0, Q1, Q2        @ 128-bit

@ OR: Vd[i] = Vn[i] | Vm[i]
VORR    D0, D1, D2
VORR    Q0, Q1, Q2

@ EOR (XOR): Vd[i] = Vn[i] ^ Vm[i]
VEOR    D0, D1, D2
VEOR    Q0, Q1, Q2

@ Bit Clear (AND NOT): Vd[i] = Vn[i] & ~Vm[i]
VBIC    D0, D1, D2        @ D0 = D1 & ~D2
VBIC    Q0, Q1, Q2

@ OR NOT: Vd[i] = Vn[i] | ~Vm[i]
VORN    D0, D1, D2
VORN    Q0, Q1, Q2

@ NOT: Vd[i] = ~Vm[i]
VMVN    D0, D1
VMVN    Q0, Q1

@ Immediate forms:
VORR.I16  D0, #0xFF00     @ OR immediate 16-bit
VBIC.I32  Q0, #0x000000FF @ clear low byte of each 32-bit element
VMOV.I8   D0, #0          @ fill with zero
VMOV.I32  Q0, #0xFFFFFFFF @ fill with ones
```

### 4.2 Compare Operations

Compare คืนค่า mask (all 1s = true, all 0s = false):

```asm
@ VCGT: compare greater than
VCGT.S8   D0, D1, D2      @ D0[i] = (D1[i] > D2[i]) ? 0xFF : 0x00
VCGT.S16  Q0, Q1, Q2
VCGT.U32  D0, D1, D2      @ unsigned
VCGT.F32  Q0, Q1, Q2      @ float

@ VCGE: compare >= 
VCGE.S8   D0, D1, D2
VCGE.F32  Q0, Q1, Q2

@ VCEQ: compare ==
VCEQ.I8   D0, D1, D2
VCEQ.I32  Q0, Q1, Q2
VCEQ.F32  Q0, Q1, Q2

@ VCEQ with zero (2-operand form):
VCEQ.I16  D0, D1, #0      @ D0[i] = (D1[i] == 0) ? 0xFFFF : 0

@ VCLE / VCLT: less-than-or-equal / less-than (2-operand only vs. zero, or swap operands)
VCLE.S32  D0, D1, #0      @ D0[i] = (D1[i] <= 0) ? all-1 : 0
VCLT.F32  Q0, Q1, #0      @ float compare < 0
```

### 4.3 VTST — Bitwise Test

```asm
@ VTST: Vd[i] = ((Vn[i] & Vm[i]) != 0) ? all-1 : 0
VTST.8  D0, D1, D2        @ test any bit set
VTST.16 Q0, Q1, Q2
VTST.32 D0, D1, D2
```

### 4.4 VBSL / VBIT / VBIF — Bitwise Select

```asm
@ VBSL: Vd[i] = (Vmask[i]) ? Vn[i] : Vm[i]
@ Vd ทำหน้าที่เป็น mask
VBSL    D0, D1, D2        @ D0 = mask, result มาแทนที่ D0
@ ถ้า bit ใน D0 = 1 -> ใช้ D1, ถ้า 0 -> ใช้ D2

@ VBIT: Vd[i] = Vd[i] | (Vn[i] & Vm[i])
@ (Bit Insert if True)
VBIT    D0, D1, D2        @ ใส่ bits จาก D1 ที่ mask D2 บน = 1

@ VBIF: Vd[i] = Vd[i] | (Vn[i] & ~Vm[i])
@ (Bit Insert if False)
VBIF    D0, D1, D2

@ ตัวอย่าง: clamp int16 ให้อยู่ในช่วง [0, 255]
@ Q0 = ค่าที่ต้องการ clamp
VMOV.I16  Q1, #0          @ zero
VMOV.I16  Q2, #255        @ max
VMAX.S16  Q0, Q0, Q1      @ clamp low
VMIN.S16  Q0, Q0, Q2      @ clamp high
```

### 4.5 VCLS / VCLZ — Count Leading Sign/Zero Bits

```asm
VCLS.S8   D0, D1          @ count leading sign bits (bits after MSB that == MSB)
VCLS.S16  Q0, Q1
VCLS.S32  D0, D1

VCLZ.I8   D0, D1          @ count leading zero bits
VCLZ.I16  Q0, Q1
VCLZ.I32  D0, D1
```

---

## 5. NEON Shift

### 5.1 VSHL / VSHR — Shift by Immediate

```asm
@ VSHL: shift left by immediate (immediate >=0)
VSHL.I8   D0, D1, #3      @ D0[i] = D1[i] << 3
VSHL.I16  Q0, Q1, #4
VSHL.I32  D0, D1, #1
VSHL.I64  Q0, Q1, #8

@ VSHR: shift right by immediate (immediate >0)
VSHR.S8   D0, D1, #2      @ signed shift right (sign-extend)
VSHR.U8   D0, D1, #2      @ unsigned shift right (zero-extend)
VSHR.S16  Q0, Q1, #5
VSHR.U32  D0, D1, #1

@ ตัวอย่าง: scale int16 values by 1/8
VSHR.S16  Q0, Q1, #3      @ divide by 8 (arithmetic right shift)
```

### 5.2 VSHLL — Shift Long (Widening)

```asm
@ VSHLL: shift + widen (D input -> Q output)
VSHLL.S8  Q0, D1, #4      @ I8 -> I16, shifted left 4
VSHLL.U16 Q0, D1, #8      @ U16 -> U32
VSHLL.S32 Q0, D1, #2      @ I32 -> I64

@ เมื่อ shift amount = element size, เทียบเท่ากับ VMOVL:
VSHLL.S16 Q0, D1, #16     @ = VMOVL.S16 Q0, D1
```

### 5.3 VSHRN — Shift Narrow

```asm
@ VSHRN: shift right + narrow (Q input -> D output)
VSHRN.I16 D0, Q1, #4      @ I16 -> I8, shift right 4
VSHRN.I32 D0, Q1, #8      @ I32 -> I16
VSHRN.I64 D0, Q1, #16     @ I64 -> I32

@ VRSHRN: rounding shift narrow
VRSHRN.I16 D0, Q1, #4     @ ปัดเศษ แล้ว narrow
```

### 5.4 VRSHL / VRSHR — Rounding Shift

```asm
@ Rounding shift by register:
VRSHL.S8  D0, D1, D2      @ D0[i] = round_shift(D1[i], D2[i])
@ D2[i] เป็นจำนวน shift: บวก=left, ลบ=right

@ Rounding shift by immediate:
VRSHR.S16 Q0, Q1, #3      @ shift right + round (add 0.5 before shift)
VRSHR.U32 D0, D1, #8

@ Saturating shift:
VQSHL.S8  D0, D1, #4      @ shift + clamp (ไม่ overflow)
VQSHLU.S16 Q0, Q1, #8     @ signed shift to unsigned (with saturation)
```

---

## 6. NEON Permute

### 6.1 VMOV — Move/Copy

```asm
@ Copy register:
VMOV    D0, D1             @ D0 = D1
VMOV    Q0, Q1             @ Q0 = Q1

@ Move ARM core register to/from NEON:
VMOV    D0[0], R0          @ D0 bits[31:0] = R0
VMOV    R0, D0[0]          @ R0 = D0 bits[31:0]
VMOV    D0, R0, R1         @ D0 bits[31:0]=R0, bits[63:32]=R1
VMOV    R0, R1, D0         @ R0=D0 bits[31:0], R1=D0 bits[63:32]

@ Load immediate:
VMOV.I8  D0, #42           @ fill all bytes with 42
VMOV.I16 Q0, #0x1234       @ fill all 16-bit with 0x1234
VMOV.F32 Q0, #1.0          @ fill 4 floats with 1.0
```

### 6.2 VDUP — Duplicate Element to All Lanes

```asm
@ VDUP: Vd[i] = element จาก scalar หรือ lane ที่ระบุ
VDUP.8  D0, R0             @ D0[i] = R0[7:0] (fill all 8 lanes)
VDUP.16 Q0, R1             @ Q0[i] = R1[15:0]
VDUP.32 D0, R2             @ D0[i] = R2

@ VDUP from lane:
VDUP.32 Q0, D1[0]          @ Q0[i] = D1[0] (replicate lane 0)
VDUP.16 D0, D2[3]          @ D0[i] = D2 lane 3

@ ตัวอย่าง: broadcast scalar ไปทุก lanes
@ สมมติ scale factor ใน R0
VDUP.32 Q1, R0             @ Q1 = {R0, R0, R0, R0}
VMUL.F32 Q0, Q0, Q1        @ scale ทุก element
```

### 6.3 VEXT — Extract (Concatenate and Shift)

VEXT ต่อ 2 registers แล้วดึง N bytes ออกมา:

```
D0 = [d7,d6,d5,d4,d3,d2,d1,d0]
D1 = [e7,e6,e5,e4,e3,e2,e1,e0]
VEXT.8 D2, D0, D1, #3:
D2 = [e2,e1,e0,d7,d6,d5,d4,d3]   (shift 3 bytes)
```

```asm
VEXT.8  D2, D0, D1, #3    @ byte extraction/shift
VEXT.16 D2, D0, D1, #2    @ element extraction (halfword)
VEXT.32 Q2, Q0, Q1, #3    @ 3-element shift (Q registers)
VEXT.8  Q2, Q0, Q1, #5

@ ใช้สำหรับ sliding window / filter operations
```

### 6.4 VREV — Reverse Elements

```asm
@ VREV64: reverse elements within each 64-bit group
VREV64.8  D0, D1           @ [d7,d6,d5,d4,d3,d2,d1,d0] -> [d0,d1,d2,d3,d4,d5,d6,d7]
VREV64.16 Q0, Q1           @ reverse 4 × 16-bit per 64-bit lane
VREV64.32 D0, D1           @ swap 2 × 32-bit

@ VREV32: reverse elements within each 32-bit group
VREV32.8  D0, D1           @ reverse bytes within each 32-bit word
VREV32.16 Q0, Q1

@ VREV16: reverse elements within each 16-bit group
VREV16.8  D0, D1           @ swap bytes within each halfword

@ ตัวอย่าง: big-endian byte swap (I32)
VLD1.32   {Q0}, [R0]!
VREV32.8  Q0, Q0           @ swap bytes in each 32-bit word
VST1.32   {Q0}, [R1]!
```

### 6.5 VZIP / VUZP — Interleave / Deinterleave

```asm
@ VZIP: interleave elements from 2 registers
@ Before: D0 = [a3,a2,a1,a0], D1 = [b3,b2,b1,b0]
@ After VZIP.16 D0, D1:
@   D0 = [b1,a1,b0,a0]
@   D1 = [b3,a3,b2,a2]
VZIP.8  D0, D1
VZIP.16 D0, D1
VZIP.32 Q0, Q1

@ VUZP: deinterleave (separate interleaved data)
@ Before: D0 = [b1,a1,b0,a0], D1 = [b3,a3,b2,a2]
@ After VUZP.16 D0, D1:
@   D0 = [b1,b0,a1,a0] — even, odd from each source
VUZP.8  D0, D1
VUZP.16 Q0, Q1
```

### 6.6 VTRN — Transpose

```asm
@ VTRN: transpose elements (matrix-like)
@ Before: D0 = [a3,a2,a1,a0], D1 = [b3,b2,b1,b0]
@ After VTRN.16 D0, D1:
@   D0 = [b2,a2,b0,a0]
@   D1 = [b3,a3,b1,a1]
VTRN.8  D0, D1
VTRN.16 D0, D1
VTRN.32 Q0, Q1
VTRN.32 D0, D1             @ ใช้สำหรับ 2×2 matrix transpose
```

### 6.7 VTBL / VTBX — Table Lookup

VTBL เป็น per-byte lookup table — มีประโยชน์มากสำหรับ permutation, AES SubBytes, ฯลฯ:

```asm
@ VTBL.8 Dd, {Dn}, Dm
@ Dd[i] = table[Dm[i]] ถ้า Dm[i] < table_size, ไม่งั้น 0
@ table ใน Dn (D register list, 1-4 registers = 8-32 bytes)

@ ตัวอย่าง: lookup table 1 register (8-byte table)
@ D1 = lookup table
@ D2 = indices
VTBL.8  D0, {D1}, D2       @ D0[i] = D1[D2[i]] หรือ 0

@ ตัวอย่าง: lookup table 2 registers (16-byte table)
VTBL.8  D0, {D1, D2}, D3   @ 16-byte table

@ VTBX: เหมือน VTBL แต่ถ้า index out of range -> เก็บค่าเดิมใน Dd
VTBX.8  D0, {D1, D2}, D3   @ D0[i] ไม่เปลี่ยนถ้า D3[i] >= 16
```

---

## 7. NEON Conversion

### 7.1 VCVT — Convert Between Types

```asm
@ Integer <-> Float
VCVT.F32.S32  D0, D1       @ I32 -> F32
VCVT.F32.U32  Q0, Q1       @ U32 -> F32
VCVT.S32.F32  D0, D1       @ F32 -> I32 (truncate toward zero)
VCVT.U32.F32  Q0, Q1       @ F32 -> U32

@ Fixed-point <-> Float
@ VCVT.F32.S32 Vd, Vm, #fbits  (fbits = fractional bits)
VCVT.F32.S32  Q0, Q1, #8  @ I32 Q8 fixed-point -> F32
VCVT.S32.F32  Q0, Q1, #8  @ F32 -> I32 Q8

@ Float rounding:
VCVTM.S32.F32 Q0, Q1       @ floor (round toward -inf)
VCVTP.S32.F32 Q0, Q1       @ ceiling (round toward +inf)
VCVTN.S32.F32 Q0, Q1       @ nearest
VCVTA.S32.F32 Q0, Q1       @ away from zero

@ Float16 <-> Float32 (VFPv4 half-precision):
VCVT.F16.F32  D0, Q1       @ 4 × F32 -> 4 × F16 (D output)
VCVT.F32.F16  Q0, D1       @ 4 × F16 -> 4 × F32
```

### 7.2 VMOVN / VQMOVN — Narrow

```asm
@ VMOVN: narrow (truncate high bits, Q input -> D output)
VMOVN.I16  D0, Q1          @ I16 -> I8 (low 8 bits)
VMOVN.I32  D0, Q1          @ I32 -> I16
VMOVN.I64  D0, Q1          @ I64 -> I32

@ VQMOVN: saturating narrow (clamp แทน truncate)
VQMOVN.S16  D0, Q1         @ I16 -> I8, clamp to [-128, 127]
VQMOVN.U16  D0, Q1         @ U16 -> U8, clamp to [0, 255]
VQMOVN.S32  D0, Q1         @ I32 -> I16
VQMOVUN.S16 D0, Q1         @ signed I16 -> unsigned I8, clamp [0,255]
```

### 7.3 VMOVL — Long (Widen)

```asm
@ VMOVL: widen (D input -> Q output, sign/zero extend)
VMOVL.S8   Q0, D1          @ I8 -> I16 (sign extend)
VMOVL.U8   Q0, D1          @ U8 -> U16 (zero extend)
VMOVL.S16  Q0, D1          @ I16 -> I32
VMOVL.U32  Q0, D1          @ U32 -> U64
```

---

## 8. Code Examples

### 8.1 RGB to Grayscale ด้วย VLD3 + VMULL

สูตร: Gray = 0.299*R + 0.587*G + 0.114*B
ใช้ integer approximation: Gray ≈ (77*R + 150*G + 29*B) >> 8

```asm
@ rgb_to_gray_neon.s
@ ARM32 NEON, GAS syntax
@ void rgb_to_gray(const uint8_t *rgb, uint8_t *gray, int pixels)
@ R0 = rgb source (packed RGB, 3 bytes per pixel)
@ R1 = gray destination
@ R2 = number of pixels (multiple of 8)

.text
.global rgb_to_gray
.type rgb_to_gray, %function
.fpu neon
.arch armv7-a

rgb_to_gray:
    PUSH    {R4, LR}
    
    @ preload coefficients
    VMOV.I8   D4, #77         @ R coefficient (q8 fixed: 0.302)
    VMOV.I8   D5, #150        @ G coefficient (q8 fixed: 0.586)
    VMOV.I8   D6, #29         @ B coefficient (q8 fixed: 0.113)
    
    @ process 8 pixels per iteration (24 bytes RGB -> 8 bytes gray)
.loop8:
    CMP     R2, #8
    BLT     .done
    
    @ โหลด 8 RGB pixels (24 bytes), de-interleave into R, G, B
    VLD3.8  {D0, D1, D2}, [R0]!
    @ D0 = {R7..R0}  (8 red pixels)
    @ D1 = {G7..G0}  (8 green pixels)
    @ D2 = {B7..B0}  (8 blue pixels)
    
    @ คำนวณ 77*R, 150*G, 29*B แบบ widening (U8 * U8 -> U16)
    VMULL.U8  Q4, D0, D4      @ Q4 = 77 * R[i] (U16)
    VMULL.U8  Q5, D1, D5      @ Q5 = 150 * G[i]
    VMULL.U8  Q6, D2, D6      @ Q6 = 29 * B[i]
    
    @ รวมกัน: Q4 = 77R + 150G + 29B (ยังเป็น U16)
    VADD.U16  Q4, Q4, Q5
    VADD.U16  Q4, Q4, Q6
    
    @ >> 8: shift right 8 bits แล้ว narrow กลับเป็น U8
    VSHRN.U16 D0, Q4, #8      @ D0 = gray[i] (U8, 8 values)
    
    @ เก็บผลลัพธ์
    VST1.8  {D0}, [R1]!
    
    SUB     R2, R2, #8
    B       .loop8

.done:
    @ TODO: handle remaining pixels (< 8) with scalar code
    
    POP     {R4, PC}
.size rgb_to_gray, .-rgb_to_gray
```

### 8.2 Audio Mixing — Interleaved Stereo

```asm
@ audio_mix_neon.s
@ ARM32 NEON, GAS syntax
@ void mix_stereo(const int16_t *src1, const int16_t *src2,
@                 int16_t *dst, int samples)
@ Mix 2 stereo streams (L0,R0,L1,R1,...) with saturation
@ R0 = src1, R1 = src2, R2 = dst, R3 = frame count (stereo pairs)

.text
.global mix_stereo
.type mix_stereo, %function
.fpu neon

mix_stereo:
    PUSH    {LR}
    
    @ process 8 stereo frames per iteration (16 × I16 = 32 bytes)
.mix_loop:
    CMP     R3, #8
    BLT     .mix_tail
    
    @ โหลด 8 stereo frames จาก src1 และ src2
    VLD1.16 {Q0, Q1}, [R0]!   @ Q0 = src1[0..7], Q1 = src1[8..15]
    VLD1.16 {Q2, Q3}, [R1]!   @ Q2 = src2[0..7], Q3 = src2[8..15]
    
    @ บวกแบบ saturating (ป้องกัน overflow ของ I16)
    VQADD.S16  Q0, Q0, Q2     @ Q0[i] = sat_add(Q0[i], Q2[i])
    VQADD.S16  Q1, Q1, Q3
    
    @ เก็บผลลัพธ์
    VST1.16 {Q0, Q1}, [R2]!
    
    SUB     R3, R3, #8
    B       .mix_loop

.mix_tail:
    @ 4 frames
    CMP     R3, #4
    BLT     .mix_done
    VLD1.16 {Q0}, [R0]!
    VLD1.16 {Q1}, [R1]!
    VQADD.S16 Q0, Q0, Q1
    VST1.16 {Q0}, [R2]!
    SUB     R3, R3, #4

.mix_done:
    POP     {PC}
.size mix_stereo, .-mix_stereo
```

### 8.3 4×4 Matrix Multiply with NEON

```asm
@ mat4x4_mul_neon.s
@ ARM32 NEON, GAS syntax
@ void mat4_mul(const float *A, const float *B, float *C)
@ C = A * B   (column-major หรือ row-major ขึ้นอยู่กับ convention)
@ R0 = A (16 floats), R1 = B (16 floats), R2 = C (16 floats)
@
@ Strategy: compute one column of C at a time
@   C[:,j] = A * B[:,j]
@   = A[:,0]*B[0,j] + A[:,1]*B[1,j] + A[:,2]*B[2,j] + A[:,3]*B[3,j]

.text
.global mat4_mul
.type mat4_mul, %function
.fpu neon

mat4_mul:
    PUSH    {R4-R6, LR}
    VPUSH   {D8-D15}           @ save callee-saved NEON regs
    
    @ โหลด matrix A: A0-A3 = 4 columns (แต่ละ column = 4 floats = Q register)
    VLD1.32 {Q0, Q1}, [R0]!   @ Q0 = A col0, Q1 = A col1
    VLD1.32 {Q2, Q3}, [R0]!   @ Q2 = A col2, Q3 = A col3
    
    MOV     R4, #4             @ 4 columns ใน B
    MOV     R5, R1             @ pointer to B
    MOV     R6, R2             @ pointer to C

.col_loop:
    @ โหลด 1 column จาก B (4 floats)
    VLD1.32 {D8}, [R5]!        @ D8 = {B[0], B[1]}
    VLD1.32 {D9}, [R5]!        @ D9 = {B[2], B[3]}
    @ Q4 = {B[3], B[2], B[1], B[0]} (1 column of B)
    
    @ C col j = A[:,0]*b0 + A[:,1]*b1 + A[:,2]*b2 + A[:,3]*b3
    VMUL.F32   Q8, Q0, D8[0]  @ Q8 = A col0 * B[0]
    VMLA.F32   Q8, Q1, D8[1]  @ Q8 += A col1 * B[1]
    VMLA.F32   Q8, Q2, D9[0]  @ Q8 += A col2 * B[2]
    VMLA.F32   Q8, Q3, D9[1]  @ Q8 += A col3 * B[3]
    
    @ เก็บ result column
    VST1.32 {Q8}, [R6]!
    
    SUBS    R4, R4, #1
    BNE     .col_loop
    
    VPOP    {D8-D15}
    POP     {R4-R6, PC}
.size mat4_mul, .-mat4_mul
```

### 8.4 Memcpy with NEON

```asm
@ memcpy_neon.s
@ ARM32 NEON, GAS syntax
@ void *neon_memcpy(void *dst, const void *src, size_t n)
@ R0 = dst, R1 = src, R2 = n
@ Returns R0 = dst

.text
.global neon_memcpy
.type neon_memcpy, %function
.fpu neon

neon_memcpy:
    PUSH    {R0, LR}           @ save original dst
    
    @ copy 128 bytes per iteration (4 × Q registers = 4 × 16 bytes)
.copy128:
    CMP     R2, #128
    BLT     .copy64
    
    VLD1.8  {D0-D3}, [R1]!    @ load 32 bytes
    VLD1.8  {D4-D7}, [R1]!    @ load 32 bytes
    VLD1.8  {D8-D11},[R1]!    @ load 32 bytes
    VLD1.8  {D12-D15},[R1]!   @ load 32 bytes
    
    VST1.8  {D0-D3}, [R0]!
    VST1.8  {D4-D7}, [R0]!
    VST1.8  {D8-D11},[R0]!
    VST1.8  {D12-D15},[R0]!
    
    SUB     R2, R2, #128
    B       .copy128

.copy64:
    CMP     R2, #64
    BLT     .copy32
    VLD1.8  {D0-D7}, [R1]!    @ 64 bytes
    VST1.8  {D0-D7}, [R0]!
    SUB     R2, R2, #64
    B       .copy64

.copy32:
    CMP     R2, #32
    BLT     .copy16
    VLD1.8  {D0-D3}, [R1]!    @ 32 bytes
    VST1.8  {D0-D3}, [R0]!
    SUB     R2, R2, #32

.copy16:
    CMP     R2, #16
    BLT     .copy8
    VLD1.8  {D0, D1}, [R1]!   @ 16 bytes
    VST1.8  {D0, D1}, [R0]!
    SUB     R2, R2, #16

.copy8:
    CMP     R2, #8
    BLT     .copy_tail
    VLD1.8  {D0}, [R1]!        @ 8 bytes
    VST1.8  {D0}, [R0]!
    SUB     R2, R2, #8

.copy_tail:
    @ copy remaining bytes (0-7) using ARM scalar
    CMP     R2, #0
    BEQ     .copy_done
.byte_loop:
    LDRB    R3, [R1], #1
    STRB    R3, [R0], #1
    SUBS    R2, R2, #1
    BNE     .byte_loop

.copy_done:
    POP     {R0, PC}           @ return original dst
.size neon_memcpy, .-neon_memcpy
```

### 8.5 String Length with NEON

```asm
@ strlen_neon.s
@ ARM32 NEON, GAS syntax
@ size_t neon_strlen(const char *s)
@ R0 = string pointer
@ Returns R0 = length

.text
.global neon_strlen
.type neon_strlen, %function
.fpu neon

neon_strlen:
    MOV     R1, R0             @ R1 = pointer (scan)
    VMOV.I8 Q0, #0             @ Q0 = zero vector (null byte pattern)
    
    @ align pointer to 16-byte boundary
    ANDS    R2, R1, #15
    BEQ     .aligned_scan
    
    @ handle unaligned prefix scalar
.prefix_loop:
    LDRB    R3, [R1], #1
    CMP     R3, #0
    BEQ     .found_null
    ANDS    R2, R1, #15
    BNE     .prefix_loop

.aligned_scan:
    @ scan 16 bytes at a time
.scan16:
    VLD1.8  {Q1}, [R1]!        @ โหลด 16 bytes
    VCEQ.I8 Q2, Q1, Q0         @ Q2[i] = (Q1[i] == 0) ? 0xFF : 0x00
    
    @ check if any null found
    VMOV    R2, R3, D4         @ D4 = low 64 bits of Q2
    ORRS    R2, R2, R3
    BNE     .found_in_block
    VMOV    R2, R3, D5         @ D5 = high 64 bits of Q2
    ORRS    R2, R2, R3
    BNE     .found_in_block_hi
    
    B       .scan16

.found_in_block:
    @ null อยู่ใน 8 bytes แรก ย้ายกลับ
    SUB     R1, R1, #16
    B       .scalar_scan

.found_in_block_hi:
    @ null อยู่ใน 8 bytes หลัง
    SUB     R1, R1, #8
    B       .scalar_scan

.scalar_scan:
    LDRB    R3, [R1], #1
    CMP     R3, #0
    BNE     .scalar_scan

.found_null:
    @ R1 ชี้หลัง null byte
    SUB     R0, R1, R0
    SUB     R0, R0, #1         @ ลบ 1 (ไม่นับ null terminator)
    BX      LR
.size neon_strlen, .-neon_strlen
```

### 8.6 AES SubBytes with VTBL

SubBytes ใน AES ใช้ S-box lookup ซึ่ง VTBL ทำได้ดีมาก:

```asm
@ aes_subbytes_neon.s
@ ARM32 NEON, GAS syntax
@ void aes_subbytes_neon(uint8_t state[16])
@ R0 = pointer to 16-byte AES state

@ AES S-box (256 bytes, แต่ VTBL รองรับ table ได้สูงสุด 32 bytes)
@ ต้องแบ่งเป็น 2 ส่วน: high nibble และ low nibble
@ หรือใช้ 32-byte table (4 × D registers) สำหรับ VTBL

@ วิธีที่ง่ายกว่า: ใช้ VTBL สำหรับ 16 elements ทีละครั้ง
@ ต้องการ nibble decomposition และ recombination
@ แต่ approach นี้ซับซ้อน ดังนั้นขอแสดง VTBL พื้นฐานก่อน

@ ตัวอย่างที่ง่าย: S-box 16 bytes แรก (0x00-0x0F)
@ ใน AES จริงจะต้อง loop หรือใช้ technique ที่ซับซ้อนกว่า

.section .rodata
.align 4
sbox_low:
    @ AES S-box สำหรับ input 0x00-0x0F:
    .byte 0x63, 0x7C, 0x77, 0x7B, 0xF2, 0x6B, 0x6F, 0xC5
    .byte 0x30, 0x01, 0x67, 0x2B, 0xFE, 0xD7, 0xAB, 0x76

sbox_high:
    @ AES S-box สำหรับ input 0x10-0x1F:
    .byte 0xCA, 0x82, 0xC9, 0x7D, 0xFA, 0x59, 0x47, 0xF0
    .byte 0xAD, 0xD4, 0xA2, 0xAF, 0x9C, 0xA4, 0x72, 0xC0

.text
.global aes_subbytes_demo
.type aes_subbytes_demo, %function
.fpu neon

@ Demo: SubBytes สำหรับ bytes ที่มีค่า 0x00-0x1F เท่านั้น
@ สำหรับ full AES ต้องการ full 256-byte S-box
aes_subbytes_demo:
    PUSH    {LR}
    
    @ โหลด S-box tables
    ADR     R1, sbox_low
    ADR     R2, sbox_high
    VLD1.8  {D0, D1}, [R1]    @ D0,D1 = S-box[0x00..0x0F]
    VLD1.8  {D2, D3}, [R2]    @ D2,D3 = S-box[0x10..0x1F]
    
    @ โหลด AES state (16 bytes)
    VLD1.8  {Q8}, [R0]         @ Q8 = state bytes
    
    @ VTBL lookup: แบ่ง state เป็น 2 ส่วน ตาม high nibble
    @ bytes ที่ high nibble = 0 (0x00-0x0F): ใช้ {D0,D1}
    VTBL.8  D16, {D0, D1}, D16  @ lookup low table
    VTBL.8  D17, {D0, D1}, D17  @ (D16,D17 = Q8)
    
    @ เก็บผลลัพธ์
    VST1.8  {Q8}, [R0]
    
    POP     {PC}
.size aes_subbytes_demo, .-aes_subbytes_demo
```

**Full AES SubBytes ด้วย VTBL (ใช้ nibble technique):**

```asm
@ Full AES SubBytes using NEON VTBL (nibble-based approach)
@ Adapted from a common NEON AES implementation technique
@
@ Concept:
@   - แบ่ง S-box เป็น 16 subtables แต่ละอัน 16 bytes
@   - ใช้ high nibble เพื่อเลือก subtable
@   - ใช้ low nibble เพื่อ index ใน subtable
@   - iterate 16 ครั้ง (หนึ่งครั้งต่อ subtable group)

.section .rodata
.align 4

@ Full AES S-box (256 bytes)
aes_sbox:
    .byte 0x63,0x7C,0x77,0x7B,0xF2,0x6B,0x6F,0xC5
    .byte 0x30,0x01,0x67,0x2B,0xFE,0xD7,0xAB,0x76
    .byte 0xCA,0x82,0xC9,0x7D,0xFA,0x59,0x47,0xF0
    .byte 0xAD,0xD4,0xA2,0xAF,0x9C,0xA4,0x72,0xC0
    .byte 0xB7,0xFD,0x93,0x26,0x36,0x3F,0xF7,0xCC
    .byte 0x34,0xA5,0xE5,0xF1,0x71,0xD8,0x31,0x15
    .byte 0x04,0xC7,0x23,0xC3,0x18,0x96,0x05,0x9A
    .byte 0x07,0x12,0x80,0xE2,0xEB,0x27,0xB2,0x75
    .byte 0x09,0x83,0x2C,0x1A,0x1B,0x6E,0x5A,0xA0
    .byte 0x52,0x3B,0xD6,0xB3,0x29,0xE3,0x2F,0x84
    .byte 0x53,0xD1,0x00,0xED,0x20,0xFC,0xB1,0x5B
    .byte 0x6A,0xCB,0xBE,0x39,0x4A,0x4C,0x58,0xCF
    .byte 0xD0,0xEF,0xAA,0xFB,0x43,0x4D,0x33,0x85
    .byte 0x45,0xF9,0x02,0x7F,0x50,0x3C,0x9F,0xA8
    .byte 0x51,0xA3,0x40,0x8F,0x92,0x9D,0x38,0xF5
    .byte 0xBC,0xB6,0xDA,0x21,0x10,0xFF,0xF3,0xD2
    @ ... (rows 0x80-0xFF ตัดออกเพื่อความกระชับ)

.text
.global aes_subbytes_full
.type aes_subbytes_full, %function
.fpu neon

@ void aes_subbytes_full(uint8_t state[16])
@ ใช้ scalar lookups (fast แต่ไม่ใช่ SIMD แบบ pure)
aes_subbytes_full:
    ADR     R1, aes_sbox
    
    @ โหลด state ทีละ byte และ lookup
    MOV     R2, #16
.sbox_loop:
    LDRB    R3, [R0]
    LDRB    R3, [R1, R3]
    STRB    R3, [R0], #1
    SUBS    R2, R2, #1
    BNE     .sbox_loop
    
    BX      LR
.size aes_subbytes_full, .-aes_subbytes_full
```

---

## 9. การ Compile และใช้งาน

### 9.1 เปิดใช้ NEON ใน GCC

```bash
# Compile สำหรับ ARMv7 พร้อม NEON
arm-linux-gnueabihf-gcc -O2 \
    -march=armv7-a -mfpu=neon -mfloat-abi=hard \
    -o program program.c neon_asm.s

# Compile ด้วย VFPv4 + NEON
arm-linux-gnueabihf-gcc -O2 \
    -march=armv7-a+vfpv4 -mfpu=neon-vfpv4 -mfloat-abi=hard \
    -o program program.c

# Cross-compile บน x86 Linux
sudo apt-get install gcc-arm-linux-gnueabihf

# Assemble NEON assembly โดยตรง
arm-linux-gnueabihf-as -march=armv7-a -mfpu=neon neon_asm.s -o neon_asm.o
```

### 9.2 NEON Intrinsics ใน C (สำหรับ reference)

```c
// <arm_neon.h>
#include <arm_neon.h>

// ตัวอย่าง: บวก 4 float32 ใน parallel
float32x4_t a = vld1q_f32(src_a);    // โหลด 4 floats
float32x4_t b = vld1q_f32(src_b);
float32x4_t c = vaddq_f32(a, b);     // บวก
vst1q_f32(dst, c);                   // เก็บ

// NEON intrinsic naming convention:
// v<op>{q}_{type}
//   q = Q register (128-bit), ไม่มี = D register (64-bit)
//   type = f32, s8, u16, etc.

// ตัวอย่าง multiply-accumulate
float32x4_t result = vmulq_f32(a, b);
result = vmlaq_f32(result, c, d);     // result += c * d
```

### 9.3 NEON Function Calling Convention

ARM AAPCS (Procedure Call Standard) สำหรับ NEON:

```
Argument/Return registers:
  D0-D7, Q0-Q3 — ใช้ส่ง float/NEON arguments

Caller-saved (volatile):
  D0-D7 (= Q0-Q3) — ไม่ต้อง save ก่อน call

Callee-saved:
  D8-D15 (= Q4-Q7) — ต้อง save/restore ถ้าใช้

D16-D31 (= Q8-Q15):
  ไม่มีใน callee-saved list แต่ ARM AAPCS v7 ถือว่า preserved
  ปกติ treat เหมือน callee-saved เพื่อความปลอดภัย
```

```asm
@ Template function with proper register saving
my_neon_func:
    PUSH    {R4-R11, LR}      @ save ARM registers
    VPUSH   {D8-D15}          @ save NEON callee-saved registers (64 bytes)
    
    @ ... use Q0-Q15 freely ...
    
    VPOP    {D8-D15}
    POP     {R4-R11, PC}
```

---

## 10. Performance Tips

### 10.1 Pipeline และ Latency

```
Cortex-A9 NEON approximate latencies:
  VADD.I8/I16/I32  : 1 cycle throughput, 1-2 cycle latency
  VMUL.I16/I32     : 1 cycle throughput, 3 cycle latency
  VLD1 (cache hit) : 1-2 cycle throughput, 5-6 cycle latency
  VST1             : 1-2 cycle throughput

Best practices:
  - Interleave load -> compute -> store ให้ pipeline เต็ม
  - ใช้ software prefetch (PLD) สำหรับ large data
  - หลีกเลี่ยง reading result ทันทีหลัง VMUL (latency)
```

### 10.2 Alignment

```asm
@ Aligned access เร็วกว่า (และบางครั้ง required)
@ ระบุ alignment hint ใน VLD/VST:
VLD1.8  {D0}, [R0:64]     @ 64-bit aligned
VLD1.32 {Q0}, [R0:128]    @ 128-bit aligned
VLD1.8  {D0-D3}, [R0:256] @ 256-bit aligned (Cortex-A9+)
```

### 10.3 Loop Unrolling

```asm
@ เพิ่ม throughput ด้วย unrolling
.loop_unrolled:
    CMP     R2, #32
    BLT     .loop_single
    
    VLD1.8  {D0-D3}, [R0]!    @ load 32 bytes (4 D regs)
    VABS.S8   D0, D0
    VABS.S8   D1, D1
    VABS.S8   D2, D2
    VABS.S8   D3, D3
    VST1.8  {D0-D3}, [R1]!
    SUB     R2, R2, #32
    B       .loop_unrolled
```

### 10.4 Prefetch

```asm
@ PLD: prefetch data ล่วงหน้า
.loop_prefetch:
    PLD     [R0, #64]          @ prefetch 64 bytes ahead
    VLD1.8  {Q0, Q1}, [R0]!   @ load current 32 bytes
    @ ... process ...
    VST1.8  {Q2, Q3}, [R1]!
    SUBS    R2, R2, #32
    BNE     .loop_prefetch
```

---

## 11. Debugging NEON Code

### 11.1 GDB Commands

```bash
# ดู NEON registers ใน GDB
(gdb) info registers vfp
(gdb) p $q0           # แสดง Q0 register
(gdb) p/x $d0         # แสดง D0 เป็น hex

# cast เพื่อดู data types:
(gdb) p ((uint8_t[16])$q0)    # ดูเป็น 16 × uint8
(gdb) p ((float[4])$q0)       # ดูเป็น 4 × float
```

### 11.2 Common Mistakes

1. **Data type mismatch** — ลืม suffix เช่น `VADD` แทน `VADD.I16`
2. **Register overlap** — ใช้ D register ที่เป็นส่วนหนึ่งของ Q register
   ```asm
   @ BUG: D0 และ D1 เป็นส่วนของ Q0
   VMUL.F32 Q0, Q0, Q0    @ ปกติ OK
   VMUL.F32 D0, Q0, Q0    @ ERROR! ขนาดไม่ match
   ```
3. **Unaligned access** — อาจทำให้ fault บน ARM strict mode
4. **Missing VPUSH/VPOP** — ลืม save callee-saved registers

---

## 12. สรุป NEON Instruction Quick Reference

### Load/Store
| Instruction | Description |
|-------------|-------------|
| `VLD1.t {Dn-Dm}, [Rn]!` | Load consecutive elements |
| `VLD2.t {Dn,Dm}, [Rn]!` | Load interleaved 2 channels |
| `VLD3.t {Dn,Dm,Dl}, [Rn]!` | Load interleaved 3 channels (RGB) |
| `VLD4.t {Dn-Dm+3}, [Rn]!` | Load interleaved 4 channels (RGBA) |
| `VST1/2/3/4` | Corresponding stores |
| `VPUSH/VPOP {D8-D15}` | Save/restore callee-saved regs |

### Arithmetic
| Instruction | Description |
|-------------|-------------|
| `VADD.t Vd,Vn,Vm` | Packed add |
| `VSUB.t Vd,Vn,Vm` | Packed subtract |
| `VMUL.t Vd,Vn,Vm` | Packed multiply |
| `VMLA.t Vd,Vn,Vm` | Vd += Vn*Vm |
| `VFMA.F32 Vd,Vn,Vm` | Fused multiply-add |
| `VMAX/VMIN.t Vd,Vn,Vm` | Packed max/min |
| `VPADD.t Dd,Dn,Dm` | Pairwise add |
| `VMULL.t Qd,Dn,Dm` | Widening multiply |
| `VADDL.t Qd,Dn,Dm` | Widening add |

### Permute
| Instruction | Description |
|-------------|-------------|
| `VDUP.t Vd, Rn` | Broadcast scalar to all lanes |
| `VEXT.t Vd,Vn,Vm,#n` | Extract / shift elements |
| `VREV64/32/16.t Vd,Vm` | Reverse elements |
| `VZIP/VUZP.t Vd,Vm` | Interleave / deinterleave |
| `VTRN.t Vd,Vm` | Transpose |
| `VTBL.8 Dd,{Dn},Dm` | Table lookup (zeroing) |
| `VTBX.8 Dd,{Dn},Dm` | Table lookup (preserving) |

### Convert & Shift
| Instruction | Description |
|-------------|-------------|
| `VCVT.F32.S32 Vd,Vm` | Int to float |
| `VCVT.S32.F32 Vd,Vm` | Float to int |
| `VMOVL.t Qd,Dn` | Widen (extend) |
| `VMOVN.t Dd,Qn` | Narrow (truncate) |
| `VQMOVN.t Dd,Qn` | Saturating narrow |
| `VSHL.t Vd,Vm,#n` | Shift left immediate |
| `VSHR.t Vd,Vm,#n` | Shift right immediate |
| `VSHLL.t Qd,Dn,#n` | Shift left long (widening) |

---

## 13. ตัวอย่างประยุกต์ — Horizontal Sum of 16 uint8 values

```asm
@ hsum_u8_neon.s
@ uint32_t horizontal_sum_u8(const uint8_t *v)
@ คำนวณ sum ของ 16 bytes ใน v[0..15]
@ R0 = pointer

.text
.global horizontal_sum_u8
.type horizontal_sum_u8, %function
.fpu neon

horizontal_sum_u8:
    VLD1.8  {Q0}, [R0]         @ โหลด 16 bytes
    
    @ widen to U16 แล้วบวกคู่
    VPADDL.U8  Q0, Q0          @ Q0 = 8 × U16 (pairwise sum)
    VPADDL.U16 Q0, Q0          @ Q0 = 4 × U32
    VPADDL.U32 D0, D0          @ D0 = 2 × U64 (sum of 4)
    
    @ รวม 2 U64 ใน D0
    @ (ใช้ VADD เพราะผลลัพธ์ fit ใน U32 อยู่แล้ว)
    VADD.U64  D0, D0, D1       @ D0[0] = total sum (จาก Q0)
    
    @ ดึงค่าออกมาใน R0
    VMOV    R0, R1, D0         @ R0 = D0 bits[31:0] = result
    
    BX      LR
.size horizontal_sum_u8, .-horizontal_sum_u8
```

---

## 14. ตัวอย่างประยุกต์ — Alpha Blending

```asm
@ alpha_blend_neon.s
@ ARM32 NEON, GAS syntax
@ Alpha blending: dst = src*alpha + bg*(255-alpha) >> 8
@ void alpha_blend(const uint8_t *src, const uint8_t *bg,
@                  const uint8_t *alpha, uint8_t *dst, int pixels)
@ R0=src R1=bg R2=alpha R3=dst, pixels on stack

.text
.global alpha_blend
.type alpha_blend, %function
.fpu neon

alpha_blend:
    PUSH    {R4, LR}
    LDR     R4, [SP, #8]       @ pixels count
    
    VMOV.I16  Q15, #255        @ constant 255

.blend_loop:
    CMP     R4, #8
    BLT     .blend_done
    
    @ โหลด 8 pixels ของแต่ละ channel
    VLD1.8  {D0}, [R0]!        @ src (8 bytes)
    VLD1.8  {D1}, [R1]!        @ bg  (8 bytes)
    VLD1.8  {D2}, [R2]!        @ alpha (8 bytes)
    
    @ widen เป็น U16
    VMOVL.U8  Q0, D0           @ Q0 = src (U16)
    VMOVL.U8  Q1, D1           @ Q1 = bg  (U16)
    VMOVL.U8  Q2, D2           @ Q2 = alpha (U16)
    
    @ inv_alpha = 255 - alpha
    VSUB.U16  Q3, Q15, Q2      @ Q3 = 255 - alpha
    
    @ src * alpha
    VMUL.U16  Q4, Q0, Q2       @ Q4 = src * alpha
    
    @ bg * (255 - alpha)
    VMUL.U16  Q5, Q1, Q3       @ Q5 = bg * inv_alpha
    
    @ รวม
    VADD.U16  Q4, Q4, Q5       @ Q4 = src*alpha + bg*(255-alpha)
    
    @ >> 8 แล้ว narrow กลับเป็น U8
    VSHRN.U16 D0, Q4, #8       @ D0 = result U8
    
    VST1.8  {D0}, [R3]!
    SUB     R4, R4, #8
    B       .blend_loop

.blend_done:
    POP     {R4, PC}
.size alpha_blend, .-alpha_blend
```

---

## 15. ตัวอย่างประยุกต์ — Fast Transpose 4×4 Float Matrix

```asm
@ transpose4x4_neon.s
@ ARM32 NEON, GAS syntax
@ void transpose4x4_f32(const float *in, float *out)
@ R0 = input (16 floats), R1 = output
@ การ transpose 4×4 matrix ด้วย VTRN

.text
.global transpose4x4_f32
.type transpose4x4_f32, %function
.fpu neon

transpose4x4_f32:
    @ โหลด 4 rows ของ matrix
    VLD1.32 {Q0}, [R0]!        @ row 0: [a00,a01,a02,a03]
    VLD1.32 {Q1}, [R0]!        @ row 1: [a10,a11,a12,a13]
    VLD1.32 {Q2}, [R0]!        @ row 2: [a20,a21,a22,a23]
    VLD1.32 {Q3}, [R0]!        @ row 3: [a30,a31,a32,a33]
    
    @ Step 1: VTRN.32 pairs (swap adj elements between pairs of rows)
    VTRN.32 Q0, Q1             @ interleave pairs: Q0=[a10,a00,a12,a02], Q1=[a11,a01,a13,a03]
    VTRN.32 Q2, Q3
    
    @ Step 2: VSWP ข้าม D registers เพื่อรวม 4×4
    @ แต่ละ Q register ตอนนี้มี 2 elements จาก 2 rows
    @ ต้องการ swap D1 กับ D4 และ D3 กับ D6
    VSWP    D1, D4             @ swap high halves: Q0 row0, Q2 row2
    VSWP    D3, D6             @ swap high halves: Q1 row1, Q3 row3
    
    @ ตอนนี้:
    @ Q0 = [a30,a20,a10,a00] = column 0 transposed
    @ Q1 = [a31,a21,a11,a01] = column 1 transposed
    @ Q2 = [a32,a22,a12,a02] = column 2 transposed
    @ Q3 = [a33,a23,a13,a03] = column 3 transposed
    
    @ เก็บผลลัพธ์
    VST1.32 {Q0}, [R1]!
    VST1.32 {Q1}, [R1]!
    VST1.32 {Q2}, [R1]!
    VST1.32 {Q3}, [R1]!
    
    BX      LR
.size transpose4x4_f32, .-transpose4x4_f32
```

---

## 16. Differences: ARM32 NEON vs AArch64 ASIMD

| Feature          | ARM32 NEON              | AArch64 ASIMD                    |
|------------------|-------------------------|----------------------------------|
| Registers        | Q0-Q15 (D0-D31)         | V0-V31 (128-bit each)            |
| Register names   | Q/D notation            | V notation (Vn.type)             |
| Syntax           | VADD.F32 Q0, Q1, Q2     | FADD v0.4s, v1.4s, v2.4s        |
| 64-bit float     | ผ่าน VFP เท่านั้น        | NEON รองรับ F64 เต็ม             |
| Scalar ops       | ใช้ lane notation [n]    | ใช้ lane notation .s[n]          |
| Assembler dirs   | .fpu neon               | ไม่จำเป็น (built-in)             |
| Instruction count| ~200+                   | มากกว่า (crypto extensions, etc.)|

ตัวอย่าง AArch64 equivalent:
```asm
@ ARM32 NEON:
VADD.F32  Q0, Q1, Q2
VLD1.8    {Q0}, [R0]!
VMULL.S16 Q0, D2, D3

@ AArch64 ASIMD equivalent:
FADD      v0.4s, v1.4s, v2.4s
LD1       {v0.16b}, [x0], #16
SMULL     v0.4s, v2.4h, v3.4h
```

---

## 17. Conditional Execution กับ NEON

NEON instructions ใน ARM32 ไม่รองรับ condition codes (ไม่มี `VADDNE`, `VMULEQ` ฯลฯ) แต่สามารถใช้ compare + select แทน:

```asm
@ ตัวอย่าง: conditional abs (ถ้า a > b, result = a, ไม่งั้น result = b)
@ ซึ่งก็คือ VMAX

@ ตัวอย่างที่ซับซ้อนกว่า: if (a[i] > threshold) a[i] = threshold else a[i] = a[i]
@ = clamp to max = VMIN

VMOV.I16  Q1, #100         @ threshold = 100
VMIN.S16  Q0, Q0, Q1       @ clamp

@ ตัวอย่าง: ถ้า a[i] < 0: a[i] = 0 (ReLU activation)
VMOV.I32  Q1, #0
VMAX.S32  Q0, Q0, Q1       @ max(a[i], 0) = ReLU
```

---

## 18. NEON และ Cache Considerations

```asm
@ NEON operations มักจะ memory bound ดังนั้น cache management สำคัญมาก

@ Prefetch data:
PLD     [R0, #128]         @ prefetch 128 bytes ahead
PLDW    [R1, #128]         @ prefetch for write

@ Non-temporal store (hint: ไม่ต้องการ cache)
@ ARM32 ไม่มี non-temporal store เหมือน x86's MOVNTQ
@ แต่สามารถใช้ cache eviction hints ใน newer cores

@ ตัวอย่าง loop พร้อม prefetch:
.prefetch_loop:
    PLD     [R0, #128]
    VLD1.8  {Q0, Q1}, [R0]!
    @ ... process ...
    VST1.8  {Q2, Q3}, [R1]!
    SUBS    R2, R2, #32
    BNE     .prefetch_loop
```

---

## สรุป (Summary)

ARM NEON SIMD เป็นเครื่องมือที่ทรงพลังมากสำหรับงาน:
- **Image/Video Processing**: RGB/RGBA manipulation, color space conversion, filtering
- **Audio Processing**: mixing, DSP filters, resampling
- **Linear Algebra**: matrix operations, dot products, transforms
- **Cryptography**: AES, hash functions (ด้วย lookup tables)
- **String Operations**: search, compare, copy
- **Machine Learning**: inference (dot products, activations)

Key concepts ที่ต้องจำ:
1. **Register hierarchy**: S (32) -> D (64) -> Q (128), ใช้ register bank เดียวกัน
2. **Type suffixes**: ระบุ element size และจำนวน (เช่น `.4S` = 4 × 32-bit)
3. **Load patterns**: VLD1 (sequential), VLD2/3/4 (interleaved)
4. **Widening/Narrowing**: VMULL, VADDL (widen); VMOVN, VQMOVN (narrow)
5. **Permutation**: VDUP, VEXT, VZIP, VTRN, VTBL สำหรับ rearranging data
6. **ABI**: VPUSH/VPOP D8-D15 ใน callee-saved functions

การ optimize ด้วย NEON อาจได้ speedup 4-16× เทียบกับ scalar code ขึ้นอยู่กับ algorithm และ data patterns.
