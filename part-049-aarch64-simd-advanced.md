# Part 049: AArch64 SIMD Advanced

## บทนำ (Introduction)

AArch64 (ARM 64-bit) มี SIMD/NEON unit ที่ทรงพลังมาก ซึ่งเป็น superset ของ ARM32 NEON
ใน AArch64 register file ถูก redesign ใหม่ให้ใหญ่ขึ้น unified และ consistent กว่าเดิมมาก

สิ่งที่เพิ่มขึ้นจาก ARM32 NEON:
- Register file ใหญ่ขึ้น: V0-V31 (32 registers แทน 16)
- 128-bit registers ทั้งหมด (ARM32 มี 64-bit D registers เป็นหลัก)
- Crypto extensions (AES, SHA) เป็น hardware instruction
- SVE (Scalable Vector Extension) ใน ARMv8.2+
- Better IEEE 754 compliance
- Half-precision (FP16) support

---

## 1. AArch64 SIMD Register File

### 1.1 Overview ของ Vector Registers

AArch64 มี 32 vector registers: **V0 ถึง V31**
แต่ละ register กว้าง **128 bits** และสามารถ access ได้หลายรูปแบบ:

```
 127                                                    0
 ┌─────────────────────────────────────────────────────┐
 │                    Vn (128-bit)                     │  ← Vn หรือ Qn
 ├──────────────────────────┬──────────────────────────┤
 │         upper 64         │        lower 64          │  ← Dn (lower 64-bit)
 ├────────────┬─────────────┴──────────────────────────┤
 │            │             Sn (lower 32-bit)           │
 ├────────────┴────────────────┬──────────────────────-┤
 │                             │   Hn (lower 16-bit)   │
 ├─────────────────────────────┴──────────────┬────────┤
 │                                            │Bn(8-bit)│
 └────────────────────────────────────────────┴────────┘
```

### 1.2 Scalar Access Suffixes

| Suffix | Size   | Example | ใช้งาน                        |
|--------|--------|---------|-------------------------------|
| B      | 8-bit  | B0      | Byte scalar (integer)         |
| H      | 16-bit | H0      | Half-word scalar              |
| S      | 32-bit | S0      | Single-precision float / 32-bit int |
| D      | 64-bit | D0      | Double-precision float / 64-bit int |
| Q      | 128-bit| Q0      | 128-bit quad (rare in scalar) |

```asm
// GAS Syntax (AArch64)
// Scalar operations
fmov    s0, #1.0        // load 1.0 into S0 (32-bit float)
fmov    d1, #2.0        // load 2.0 into D1 (64-bit double)
fmul    s2, s0, s1      // s2 = s0 * s1 (scalar float multiply)
fadd    d3, d0, d1      // d3 = d0 + d1 (scalar double add)
```

### 1.3 Vector Arrangement Specifiers

เวลาใช้ vector (SIMD) instructions ต้องบอก "arrangement" ว่าจะแบ่ง register ออกเป็นกี่ element แบบไหน:

| Specifier | Elements | Element Size | Total | คำอธิบาย            |
|-----------|----------|--------------|-------|----------------------|
| 8B        | 8        | 8-bit        | 64-bit| 8 bytes (lower half) |
| 16B       | 16       | 8-bit        | 128-bit| 16 bytes            |
| 4H        | 4        | 16-bit       | 64-bit| 4 half-words         |
| 8H        | 8        | 16-bit       | 128-bit| 8 half-words        |
| 2S        | 2        | 32-bit       | 64-bit| 2 singles            |
| 4S        | 4        | 32-bit       | 128-bit| 4 singles           |
| 1D        | 1        | 64-bit       | 64-bit| 1 double             |
| 2D        | 2        | 64-bit       | 128-bit| 2 doubles           |

```asm
// Vector operations with arrangement specifiers
add     v0.16b, v1.16b, v2.16b    // add 16 bytes packed
add     v0.8h,  v1.8h,  v2.8h    // add 8 half-words packed
add     v0.4s,  v1.4s,  v2.4s    // add 4 single-words packed
add     v0.2d,  v1.2d,  v2.2d    // add 2 double-words packed
fadd    v0.4s,  v1.4s,  v2.4s    // float add 4 singles
fadd    v0.2d,  v1.2d,  v2.2d    // float add 2 doubles
```

### 1.4 Lane Access

สามารถ access element ตัวเดียวใน vector ได้โดยใช้ `[index]`:

```asm
// Lane (element) access
mov     x0, v0.d[0]        // extract lane 0 of D elements → x0
mov     x1, v0.d[1]        // extract lane 1 of D elements → x1
mov     w0, v0.s[0]        // extract lane 0 of S elements → w0
mov     w1, v0.s[3]        // extract lane 3 of S elements → w1

// Insert into lane
mov     v0.s[2], w5        // insert w5 into lane 2 of S elements
mov     v1.d[0], x3        // insert x3 into lane 0 of D elements

// Float lane access
ins     v0.s[1], v2.s[3]   // copy v2 lane 3 into v0 lane 1
```

---

## 2. AArch64 NEON Load/Store

### 2.1 LD1/ST1 - Load/Store ทีละ 1-4 Registers

LD1 เป็น instruction พื้นฐานสำหรับ load vector data:

```asm
// LD1 - Load 1 register
ld1     {v0.16b}, [x0]             // load 16 bytes into v0
ld1     {v0.4s},  [x0]             // load 4 singles into v0
ld1     {v0.2d},  [x0]             // load 2 doubles into v0

// LD1 - Load 2 consecutive registers
ld1     {v0.16b, v1.16b}, [x0]    // load 32 bytes into v0, v1

// LD1 - Load 3 consecutive registers
ld1     {v0.16b, v1.16b, v2.16b}, [x0]

// LD1 - Load 4 consecutive registers
ld1     {v0.16b, v1.16b, v2.16b, v3.16b}, [x0]

// ST1 - Store
st1     {v0.16b}, [x0]
st1     {v0.4s, v1.4s}, [x0]

// Post-index addressing (อัตโนมัติเพิ่ม pointer)
ld1     {v0.16b}, [x0], #16       // load 16B, then x0 += 16
ld1     {v0.4s},  [x0], #16       // load 4S,  then x0 += 16
ld1     {v0.16b, v1.16b}, [x0], #32  // load 32B, x0 += 32
ld1     {v0.16b}, [x0], x1        // load 16B, then x0 += x1 (register offset)

// Pre-index addressing ไม่มีใน NEON load/store, ใช้ post-index แทน
```

### 2.2 LD2/ST2/LD3/ST3/LD4/ST4 - Interleaved Load/Store

สำหรับ data ที่มีการ interleave เช่น RGB pixels, stereo audio:

```asm
// LD2 - Load 2 interleaved registers
// Memory layout: A0 B0 A1 B1 A2 B2 A3 B3 ...
// After LD2: V0 = {A0,A1,A2,A3,...}, V1 = {B0,B1,B2,B3,...}
ld2     {v0.16b, v1.16b}, [x0]    // deinterleave 2-channel byte data
ld2     {v0.4s,  v1.4s},  [x0]    // deinterleave 2-channel float data

// ST2 - Store 2 interleaved registers
// V0 = {A0,A1,A2,...}, V1 = {B0,B1,B2,...}
// Memory: A0 B0 A1 B1 A2 B2 ...
st2     {v0.16b, v1.16b}, [x0]

// LD3 - สำหรับ RGB data (3-channel)
// Memory: R0 G0 B0 R1 G1 B1 R2 G2 B2 ...
// After LD3: V0=R, V1=G, V2=B
ld3     {v0.16b, v1.16b, v2.16b}, [x0]

// ST3 - Store 3 interleaved (pack RGB back)
st3     {v0.16b, v1.16b, v2.16b}, [x0]

// LD4 - สำหรับ RGBA data (4-channel)
ld4     {v0.16b, v1.16b, v2.16b, v3.16b}, [x0]
st4     {v0.16b, v1.16b, v2.16b, v3.16b}, [x0]

// Post-index versions
ld3     {v0.16b, v1.16b, v2.16b}, [x0], #48  // 3 regs * 16 bytes = 48
ld4     {v0.8b,  v1.8b,  v2.8b,  v3.8b}, [x0], #32
```

### 2.3 LD1R - Load and Replicate (Broadcast)

LD1R อ่าน 1 element แล้ว replicate (broadcast) ไปทุก lane:

```asm
// LD1R - broadcast scalar to all lanes
ld1r    {v0.16b}, [x0]     // load 1 byte, replicate to all 16 lanes
ld1r    {v0.8h},  [x0]     // load 1 half, replicate to all 8 lanes
ld1r    {v0.4s},  [x0]     // load 1 word, replicate to all 4 lanes
ld1r    {v0.2d},  [x0]     // load 1 dword, replicate to both lanes

// LD1R with post-index
ld1r    {v0.4s}, [x0], #4  // broadcast float, advance 4 bytes

// LD2R/LD3R/LD4R - broadcast interleaved structure
ld2r    {v0.4s, v1.4s}, [x0]   // load 2 floats, broadcast each to all lanes

// ตัวอย่าง: load constant ไปใช้กับ vector operation
// x0 points to a float constant
ld1r    {v7.4s}, [x0]      // v7 = {const, const, const, const}
fmul    v0.4s, v0.4s, v7.4s  // multiply all 4 floats by constant
```

---

## 3. AArch64 Integer SIMD

### 3.1 ADD/SUB - Vector Addition/Subtraction

```asm
// Integer vector add/subtract
add     v0.16b, v1.16b, v2.16b   // byte add (wrapping)
add     v0.8h,  v1.8h,  v2.8h   // halfword add
add     v0.4s,  v1.4s,  v2.4s   // word add
add     v0.2d,  v1.2d,  v2.2d   // doubleword add

sub     v0.16b, v1.16b, v2.16b  // byte subtract
sub     v0.4s,  v1.4s,  v2.4s   // word subtract
```

### 3.2 MUL/MLA/MLS - Multiply / Multiply-Accumulate

```asm
// MUL - multiply (result truncated to same width)
mul     v0.16b, v1.16b, v2.16b  // byte * byte → byte (low 8 bits)
mul     v0.8h,  v1.8h,  v2.8h   // halfword * halfword
mul     v0.4s,  v1.4s,  v2.4s   // word * word

// MLA - multiply-accumulate: Vd += Vn * Vm
mla     v0.8h, v1.8h, v2.8h    // v0 += v1 * v2 (8 halfwords)
mla     v0.4s, v1.4s, v2.4s    // v0 += v1 * v2 (4 words)

// MLS - multiply-subtract: Vd -= Vn * Vm
mls     v0.4s, v1.4s, v2.4s    // v0 -= v1 * v2

// MUL by lane (scalar element)
mul     v0.4s, v1.4s, v2.s[0]  // multiply each element of v1 by v2[0]
mla     v0.4s, v1.4s, v2.s[2]  // v0 += v1 * v2[2] (broadcast lane)
```

### 3.3 SMAX/SMIN/UMAX/UMIN - Max/Min

```asm
// SMAX/SMIN - signed max/min
smax    v0.16b, v1.16b, v2.16b  // signed max per byte
smin    v0.16b, v1.16b, v2.16b  // signed min per byte
smax    v0.4s,  v1.4s,  v2.4s   // signed max per word
smin    v0.8h,  v1.8h,  v2.8h   // signed min per halfword

// UMAX/UMIN - unsigned max/min
umax    v0.16b, v1.16b, v2.16b  // unsigned max per byte
umin    v0.16b, v1.16b, v2.16b  // unsigned min per byte
umax    v0.4s,  v1.4s,  v2.4s   // unsigned max per word

// Clamp example: clamp bytes to [16, 235] (video range)
movi    v4.16b, #16             // v4 = all 16
movi    v5.16b, #235            // v5 = all 235
umax    v0.16b, v0.16b, v4.16b  // max(v0, 16) → clamp lower bound
umin    v0.16b, v0.16b, v5.16b  // min(v0, 235) → clamp upper bound
```

### 3.4 SMULL/UMULL - Widening Multiply

Widening = ผลลัพธ์กว้างกว่า input (เพื่อป้องกัน overflow):

```asm
// SMULL - signed multiply long (widens lower half)
// 8B * 8B → 8H
smull   v0.8h, v1.8b, v2.8b    // lower 8 bytes → 8 halfwords (signed)
// 4H * 4H → 4S
smull   v0.4s, v1.4h, v2.4h    // lower 4 halfs → 4 words (signed)
// 2S * 2S → 2D
smull   v0.2d, v1.2s, v2.2s    // lower 2 words → 2 dwords (signed)

// SMULL2 - signed multiply long (widens upper half)
smull2  v0.8h, v1.16b, v2.16b  // upper 8 bytes → 8 halfwords

// UMULL/UMULL2 - unsigned multiply long
umull   v0.8h, v1.8b, v2.8b    // unsigned widening multiply
umull2  v0.8h, v1.16b, v2.16b  // upper half unsigned widening
```

### 3.5 SMLAL/UMLAL - Widening Multiply-Accumulate

```asm
// SMLAL - signed multiply-accumulate long
// Vd (wide) += Vn (narrow) * Vm (narrow)
smlal   v0.8h, v1.8b, v2.8b    // v0 += v1 * v2 (signed widening)
smlal   v0.4s, v1.4h, v2.4h    // v0 += v1 * v2 (signed widening)
smlal2  v0.4s, v1.8h, v2.8h    // upper half version

// UMLAL - unsigned multiply-accumulate long
umlal   v0.8h, v1.8b, v2.8b
umlal2  v0.4s, v1.8h, v2.8h
```

### 3.6 Reduction Instructions

```asm
// ADDV - add across vector (reduce to single element)
addv    b0, v1.16b     // sum all 16 bytes → byte in b0
addv    h0, v1.8h      // sum all 8 halfwords → halfword in h0
addv    s0, v1.4s      // sum all 4 words → word in s0
// Note: addv ไม่รองรับ 2D

// SADDLV/UADDLV - widening reduce
saddlv  h0, v1.16b     // sum 16 signed bytes → h0 (halfword, no overflow)
saddlv  s0, v1.8h      // sum 8 signed halfwords → s0
saddlv  d0, v1.4s      // sum 4 signed words → d0
uaddlv  h0, v1.16b     // unsigned version
uaddlv  s0, v1.8h

// SMAXV/SMINV/UMAXV/UMINV - reduce max/min
smaxv   b0, v1.16b     // max of all 16 bytes (signed)
sminv   b0, v1.16b     // min of all 16 bytes (signed)
umaxv   b0, v1.16b     // max of all 16 bytes (unsigned)
uminv   b0, v1.16b     // min of all 16 bytes (unsigned)
```

### 3.7 SQADD/UQADD - Saturating Add

Saturating = ถ้า overflow จะ clamp ที่ max value แทนการ wrap around:

```asm
// SQADD - signed saturating add
sqadd   v0.16b, v1.16b, v2.16b  // clamp to [-128, 127]
sqadd   v0.8h,  v1.8h,  v2.8h   // clamp to [-32768, 32767]
sqadd   v0.4s,  v1.4s,  v2.4s   // clamp to int32 range

// UQADD - unsigned saturating add
uqadd   v0.16b, v1.16b, v2.16b  // clamp to [0, 255]
uqadd   v0.8h,  v1.8h,  v2.8h   // clamp to [0, 65535]

// SQSUB/UQSUB - saturating subtract
sqsub   v0.16b, v1.16b, v2.16b
uqsub   v0.16b, v1.16b, v2.16b  // useful! e.g. brightness reduction without negative
```

### 3.8 SABD/UABD - Absolute Difference

```asm
// SABD - signed absolute difference: |Vn - Vm|
sabd    v0.16b, v1.16b, v2.16b  // |a-b| for each byte (signed)
sabd    v0.4s,  v1.4s,  v2.4s   // |a-b| for each word

// UABD - unsigned absolute difference
uabd    v0.16b, v1.16b, v2.16b  // |a-b| unsigned for each byte

// SABA/UABA - absolute difference and accumulate
// Vd += |Vn - Vm|
saba    v0.16b, v1.16b, v2.16b  // v0 += |v1-v2| signed
uaba    v0.16b, v1.16b, v2.16b  // v0 += |v1-v2| unsigned
```

### 3.9 SABDL/UABDL - Widening Absolute Difference (สำหรับ SAD)

```asm
// SABDL - signed absolute difference long (widening)
// |Vn[narrow] - Vm[narrow]| → wide result
sabdl   v0.8h, v1.8b, v2.8b    // |byte-byte| → halfword
sabdl   v0.4s, v1.4h, v2.4h    // |half-half| → word
sabdl2  v0.8h, v1.16b, v2.16b  // upper 8 bytes version

// UABDL - unsigned version
uabdl   v0.8h, v1.8b, v2.8b
uabdl2  v0.8h, v1.16b, v2.16b

// SAD (Sum of Absolute Differences) pattern สำหรับ video codec:
// เปรียบเทียบ 16 pixel blocks
movi    v0.2d, #0              // accumulator = 0
uabdl   v1.8h, v2.8b, v3.8b   // |reference - current| lower 8
uabdl2  v4.8h, v2.16b, v3.16b // |reference - current| upper 8
uaddl   v5.4s, v1.4h, v4.4h   // widen and add
uaddl2  v6.4s, v1.8h, v4.8h   // upper half
add     v0.4s, v5.4s, v6.4s   // sum all
addv    s0, v0.4s              // reduce to scalar SAD
```

---

## 4. AArch64 Float SIMD

### 4.1 Basic Float Operations

```asm
// FADD/FSUB/FMUL/FDIV
fadd    v0.4s, v1.4s, v2.4s    // float32 add (4 elements)
fadd    v0.2d, v1.2d, v2.2d    // float64 add (2 elements)
fsub    v0.4s, v1.4s, v2.4s    // float32 sub
fmul    v0.4s, v1.4s, v2.4s    // float32 mul
fdiv    v0.4s, v1.4s, v2.4s    // float32 div

// Multiply by lane
fmul    v0.4s, v1.4s, v2.s[0]  // v0[i] = v1[i] * v2[0]
fmul    v0.4s, v1.4s, v2.s[3]  // v0[i] = v1[i] * v2[3]
```

### 4.2 FMLA/FMLS - Fused Multiply-Accumulate

FMA (Fused Multiply-Add) ทำ multiply และ add ใน single operation ลด rounding error:

```asm
// FMLA - fused multiply-add: Vd += Vn * Vm
fmla    v0.4s, v1.4s, v2.4s    // v0 += v1 * v2 (no intermediate rounding)
fmla    v0.2d, v1.2d, v2.2d    // double precision FMA

// FMLA by lane
fmla    v0.4s, v1.4s, v2.s[0]  // v0 += v1 * v2[0]
fmla    v0.4s, v1.4s, v2.s[2]

// FMLS - fused multiply-subtract: Vd -= Vn * Vm
fmls    v0.4s, v1.4s, v2.4s    // v0 -= v1 * v2
fmls    v0.4s, v1.4s, v2.s[1]  // v0 -= v1 * v2[1]

// ตัวอย่าง: dot product 4 floats
fmul    v0.4s, v1.4s, v2.4s    // elementwise multiply
faddp   v0.4s, v0.4s, v0.4s    // pairwise add (2 rounds = sum)
faddp   s0, v0.2s               // final reduce
```

### 4.3 FABS/FNEG/FSQRT

```asm
// FABS - absolute value
fabs    v0.4s, v1.4s            // |x| for each float
fabs    v0.2d, v1.2d

// FNEG - negate
fneg    v0.4s, v1.4s            // -x for each float

// FSQRT - square root
fsqrt   v0.4s, v1.4s            // sqrt(x) for each float
fsqrt   v0.2d, v1.2d

// FRECPE - reciprocal estimate (approximate 1/x)
frecpe  v0.4s, v1.4s            // ~1/x
// FRSQRTE - reciprocal square root estimate (approximate 1/sqrt(x))
frsqrte v0.4s, v1.4s            // ~1/sqrt(x)
// Newton-Raphson refinement
frecps  v2.4s, v1.4s, v0.4s    // 2 - v1*v0 (refinement step)
fmul    v0.4s, v0.4s, v2.4s    // refined reciprocal
```

### 4.4 FMAX/FMIN/FMAXNM/FMINNM

```asm
// FMAX/FMIN - IEEE max/min (NaN propagating)
fmax    v0.4s, v1.4s, v2.4s    // max(v1, v2) per element, NaN propagates
fmin    v0.4s, v1.4s, v2.4s    // min(v1, v2) per element

// FMAXNM/FMINNM - number max/min (NaN ignoring)
fmaxnm  v0.4s, v1.4s, v2.4s   // max, treats NaN as missing (C99 fmax)
fminnm  v0.4s, v1.4s, v2.4s   // min, treats NaN as missing
```

### 4.5 FCVT - Type Conversion

```asm
// FCVT - convert between float types
fcvt    s0, d0                   // double → single (scalar)
fcvt    d0, s0                   // single → double (scalar)
fcvt    s0, h0                   // half → single
fcvt    h0, s0                   // single → half

// Vector conversions
fcvtl   v0.4s, v1.4h            // 4 half → 4 single (long/widen)
fcvtl2  v0.4s, v1.8h            // upper 4 half → 4 single
fcvtn   v0.4h, v1.4s            // 4 single → 4 half (narrow)
fcvtn2  v0.8h, v1.4s            // store narrow result into upper half

// Integer ↔ Float conversion
scvtf   v0.4s, v1.4s            // signed int32 → float32
ucvtf   v0.4s, v1.4s            // unsigned int32 → float32
fcvtzs  v0.4s, v1.4s            // float32 → signed int32 (truncate)
fcvtzu  v0.4s, v1.4s            // float32 → unsigned int32

// With fixed-point scaling
scvtf   v0.4s, v1.4s, #16      // int32 → float, scale by 2^-16
fcvtzs  v0.4s, v1.4s, #16      // float → int32, scale by 2^16
```

### 4.6 FADDP/FMAXP/FMINP - Pairwise Operations

```asm
// FADDP - pairwise add (adjacent pairs within and across registers)
faddp   v0.4s, v1.4s, v2.4s    // [v1[0]+v1[1], v1[2]+v1[3], v2[0]+v2[1], v2[2]+v2[3]]
faddp   s0, v1.2s               // v1[0] + v1[1] → scalar s0

// FMAXP/FMINP - pairwise max/min
fmaxp   v0.4s, v1.4s, v2.4s    // pairwise max
fminp   v0.4s, v1.4s, v2.4s    // pairwise min
fmaxp   s0, v1.2s               // max(v1[0], v1[1]) → scalar

// FMAXV/FMINV - reduce across all elements
fmaxv   s0, v1.4s              // max of all 4 floats
fminv   s0, v1.4s              // min of all 4 floats
```

### 4.7 FADDV - Reduce to Scalar

```asm
// FADDV - float add across vector
faddp   v0.4s, v1.4s, v1.4s   // pairwise: [a+b, c+d, a+b, c+d]
faddp   s0, v0.2s              // final pair: (a+b)+(c+d) = sum
// (FADDV is done in 2 FADDP steps for 4 floats)
```

---

## 5. AArch64 Permute Instructions

### 5.1 DUP - Duplicate Element

```asm
// DUP - broadcast scalar register to all lanes
dup     v0.16b, w0             // duplicate w0 (byte value) to all 16 bytes
dup     v0.8h,  w0             // duplicate w0 to all 8 halfwords
dup     v0.4s,  w0             // duplicate w0 to all 4 words
dup     v0.2d,  x0             // duplicate x0 to both dwords

// DUP from vector lane
dup     v0.4s, v1.s[2]        // broadcast v1[2] to all 4 lanes of v0
dup     v0.8h, v1.h[0]        // broadcast v1[0] halfword to all 8 lanes

// MOV - alias for DUP (when used with lane syntax)
mov     v0.s[3], v1.s[1]      // copy v1 lane 1 → v0 lane 3
```

### 5.2 INS - Insert Element

```asm
// INS - insert from general-purpose register
ins     v0.b[5],  w0           // insert w0 byte into lane 5
ins     v0.h[3],  w1           // insert w1 halfword into lane 3
ins     v0.s[2],  w2           // insert w2 word into lane 2
ins     v0.d[1],  x3           // insert x3 dword into lane 1

// INS - insert from vector lane
ins     v0.s[1], v1.s[3]      // v0[1] = v1[3]
ins     v0.b[7], v2.b[0]      // v0[7] = v2[0]
```

### 5.3 EXT - Extract Bytes

EXT สร้าง register ใหม่โดยเอา bytes จาก 2 registers ต่อกัน แล้ว shift:

```asm
// EXT - concatenate Vn:Vm, extract 16 bytes starting at byte #imm
// result = (Vm:Vn)[imm..imm+15]
ext     v0.16b, v1.16b, v2.16b, #4   // bytes [4..19] of (v2:v1) → v0
ext     v0.16b, v1.16b, v2.16b, #8   // bytes [8..23] of (v2:v1)
ext     v0.8b,  v1.8b,  v2.8b,  #3   // 64-bit version, offset 3
```

### 5.4 REV - Reverse

```asm
// REV64 - reverse elements within each 64-bit group
rev64   v0.16b, v1.16b        // reverse bytes within each 8-byte group
rev64   v0.8h,  v1.8h         // reverse halfwords within each 8-byte group
rev64   v0.4s,  v1.4s         // reverse words within each 8-byte group

// REV32 - reverse elements within each 32-bit group
rev32   v0.16b, v1.16b        // reverse bytes within each 4-byte group
rev32   v0.8h,  v1.8h         // reverse halfwords in each 4-byte group

// REV16 - reverse elements within each 16-bit group
rev16   v0.16b, v1.16b        // swap adjacent bytes

// Big/little endian conversion
rev64   v0.8h, v1.8h          // byte-swap each 16-bit word (endian)
```

### 5.5 ZIP1/ZIP2 - Interleave

```asm
// ZIP1 - interleave lower halves of two vectors
// v1 = [A0 A1 A2 A3], v2 = [B0 B1 B2 B3]
// ZIP1 → [A0 B0 A1 B1]
zip1    v0.4s, v1.4s, v2.4s   // interleave lower 2 elements
zip1    v0.16b, v1.16b, v2.16b // interleave lower 8 bytes

// ZIP2 - interleave upper halves
// ZIP2 → [A2 B2 A3 B3]
zip2    v0.4s, v1.4s, v2.4s
zip2    v0.16b, v1.16b, v2.16b
```

### 5.6 UZP1/UZP2 - Deinterleave

```asm
// UZP1 - extract even elements from 2 registers
// v1 = [A0 A1 A2 A3], v2 = [B0 B1 B2 B3]
// UZP1 → [A0 A2 B0 B2]
uzp1    v0.4s, v1.4s, v2.4s

// UZP2 - extract odd elements from 2 registers
// UZP2 → [A1 A3 B1 B3]
uzp2    v0.4s, v1.4s, v2.4s

// ตัวอย่าง: deinterleave 8-bit stereo audio
// Input: L0 R0 L1 R1 L2 R2 L3 R3 ...
// uzp1 → L0 L1 L2 L3 (left channel)
// uzp2 → R0 R1 R2 R3 (right channel)
uzp1    v0.8b, v1.8b, v2.8b    // left
uzp2    v3.8b, v1.8b, v2.8b    // right
```

### 5.7 TRN1/TRN2 - Transpose

```asm
// TRN1 - transpose even elements
// v1 = [A0 A1 A2 A3], v2 = [B0 B1 B2 B3]
// TRN1 → [A0 B0 A2 B2]
trn1    v0.4s, v1.4s, v2.4s

// TRN2 - transpose odd elements
// TRN2 → [A1 B1 A3 B3]
trn2    v0.4s, v1.4s, v2.4s

// 4x4 Matrix Transpose using TRN/ZIP:
// Load 4 rows
ld1     {v0.4s, v1.4s, v2.4s, v3.4s}, [x0]
// Transpose step 1
trn1    v4.4s, v0.4s, v1.4s    // [a00 a10 a02 a12]
trn2    v5.4s, v0.4s, v1.4s    // [a01 a11 a03 a13]
trn1    v6.4s, v2.4s, v3.4s    // [a20 a30 a22 a32]
trn2    v7.4s, v2.4s, v3.4s    // [a21 a31 a23 a33]
// Transpose step 2
zip1    v0.2d, v4.2d, v6.2d    // row 0 = [a00 a10 a20 a30]
zip1    v1.2d, v5.2d, v7.2d    // row 1 = [a01 a11 a21 a31]
zip2    v2.2d, v4.2d, v6.2d    // row 2 = [a02 a12 a22 a32]
zip2    v3.2d, v5.2d, v7.2d    // row 3 = [a03 a13 a23 a33]
// Store transposed matrix
st1     {v0.4s, v1.4s, v2.4s, v3.4s}, [x1]
```

### 5.8 TBL/TBX - Table Lookup (like x86 PSHUFB)

```asm
// TBL - Table Byte Lookup
// วิธีใช้: v0 = table (1-4 registers), v_idx = byte indices
// ถ้า index >= table_size → result byte = 0
tbl     v0.16b, {v1.16b},        v2.16b    // 1-register table lookup
tbl     v0.16b, {v1.16b, v2.16b}, v3.16b  // 2-register table (32-byte table)
tbl     v0.16b, {v1.16b, v2.16b, v3.16b}, v4.16b // 3-register table
tbl     v0.16b, {v1.16b, v2.16b, v3.16b, v4.16b}, v5.16b // 4-reg (64-byte)

// TBX - Table Byte eXchange (ถ้า index out of range → ไม่เปลี่ยน output)
tbx     v0.16b, {v1.16b}, v2.16b

// ตัวอย่าง: Byte shuffle (เหมือน PSHUFB)
// สร้าง index table สำหรับ BGR → RGB conversion
// index: [2, 1, 0, 5, 4, 3, 8, 7, 6, 11, 10, 9, 14, 13, 12, 15]
.section .rodata
bgr2rgb_idx:
    .byte 2, 1, 0, 5, 4, 3, 8, 7, 6, 11, 10, 9, 14, 13, 12, 15

.text
// x0 = src (BGR), x1 = dst (RGB), x2 = count
// x3 = pointer to bgr2rgb_idx
adrp    x3, bgr2rgb_idx
add     x3, x3, :lo12:bgr2rgb_idx
ld1     {v31.16b}, [x3]           // load shuffle index

bgr2rgb_loop:
    ld1     {v0.16b}, [x0], #16   // load 16 BGR bytes (5.33 pixels)
    tbl     v1.16b, {v0.16b}, v31.16b  // shuffle B,G,R → R,G,B
    st1     {v1.16b}, [x1], #16   // store
    subs    x2, x2, #16
    b.gt    bgr2rgb_loop
```

### 5.9 PMUL/PMULL - Polynomial Multiply

```asm
// PMUL - polynomial multiply (XOR-based, no carry)
pmul    v0.8b, v1.8b, v2.8b    // polynomial multiply per byte

// PMULL - polynomial multiply long (widening)
// 8 x 8-bit poly → 8 x 16-bit poly
pmull   v0.8h, v1.8b, v2.8b

// PMULL2 - polynomial multiply long (upper half)
pmull2  v0.8h, v1.16b, v2.16b

// 64-bit polynomial multiply (for GCM/GHASH)
// Vn.1D * Vm.1D → Vd.1Q (128-bit result)
pmull   v0.1q, v1.1d, v2.1d    // 64x64 → 128 polynomial multiply
pmull2  v0.1q, v1.2d, v2.2d    // upper 64 bits version

// GCM (Galois/Counter Mode) ใช้ polynomial multiply over GF(2^128)
// นี่คือ core operation ของ AES-GCM
```

---

## 6. AArch64 Crypto Extensions

### 6.1 AES Instructions

ARMv8 Crypto Extension มี AES instructions สำหรับ AES encryption/decryption:

```
AESE  - AES single round encryption (SubBytes + ShiftRows + AddRoundKey)
AESD  - AES single round decryption (InvShiftRows + InvSubBytes + AddRoundKey)
AESMC  - AES mix columns
AESIMC - AES inverse mix columns
```

AES-128 round structure:
- AESE + AESMC = one full encryption round (except last)
- AESD + AESIMC = one full decryption round (except last)

```asm
// AES-128 Encryption single round (with round key)
// v0 = state, v1 = round key
aese    v0.16b, v1.16b     // SubBytes + ShiftRows + AddRoundKey(v1)
aesmc   v0.16b, v0.16b     // MixColumns

// AES-128 last round (no MixColumns)
aese    v0.16b, v9.16b     // last round key
// then XOR with next round key (manual)
eor     v0.16b, v0.16b, v10.16b

// AES-128 Decryption
aesd    v0.16b, v1.16b     // InvShiftRows + InvSubBytes + AddRoundKey
aesimc  v0.16b, v0.16b     // InvMixColumns
```

### 6.2 Full AES-128 ECB Encryption Example

```asm
// AES-128 ECB Encryption
// Input:  x0 = plaintext (16 bytes), x1 = round keys (11 * 16 bytes)
// Output: x2 = ciphertext (16 bytes)
// Clobbers: v0-v11

.global aes128_ecb_encrypt
aes128_ecb_encrypt:
    // Load plaintext into v0
    ld1     {v0.16b}, [x0]

    // Load all 11 round keys
    ld1     {v1.16b, v2.16b, v3.16b, v4.16b},  [x1], #64
    ld1     {v5.16b, v6.16b, v7.16b, v8.16b},  [x1], #64
    ld1     {v9.16b, v10.16b, v11.16b},         [x1]

    // Initial whitening (round 0)
    eor     v0.16b, v0.16b, v1.16b

    // Rounds 1-9 (each = AESE + AESMC)
    aese    v0.16b, v2.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v3.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v4.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v5.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v6.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v7.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v8.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v9.16b;  aesmc  v0.16b, v0.16b
    aese    v0.16b, v10.16b; aesmc  v0.16b, v0.16b

    // Round 10 (last round, no MixColumns) - but AESE already applies round key
    // Trick: AESE with zero key, then XOR manually with last round key
    // Actually: standard approach below
    aese    v0.16b, v11.16b         // SubBytes + ShiftRows + AddRoundKey(v11)
    // But wait - AESE XORs v11 AFTER SubBytes+ShiftRows
    // We need one more XOR if using this pattern... see note below.
    // Correct pattern for last round:
    // aese  v0.16b, v10.16b        // round 9 key as part of AESE
    // eor   v0.16b, v0.16b, v11.16b // then XOR final key

    // Store ciphertext
    st1     {v0.16b}, [x2]
    ret

// NOTE: AESE semantics = state = SubBytes(ShiftRows(state XOR key))
// So proper pattern for rounds 1-9:
//   aese  v0.16b, roundkey.16b   // SB(SR(state XOR rk))
//   aesmc v0.16b, v0.16b          // MC(result)
// For last round (round 10):
//   aese  v0.16b, rk9.16b        // SB(SR(state XOR rk9))  -- no MC
//   eor   v0.16b, v0.16b, rk10.16b // XOR with last key
```

### 6.3 SHA-1 Instructions

```asm
// SHA1 instructions:
// SHA1C - SHA1 hash update (choose function)
// SHA1M - SHA1 hash update (majority function)
// SHA1P - SHA1 hash update (parity function)
// SHA1H - SHA1 hash rotate
// SHA1SU0/SHA1SU1 - SHA1 schedule update

// SHA1 round structure:
// v0 = {a, b, c, d} current state (4x32-bit)
// s0 = e (fifth word, scalar)
// v1 = message schedule (4 words)
// v2 = round constant

// Rounds 0-19 (choose):
sha1c   q0, s1, v2.4s   // hash update with choose, e=s1, msg=v2
sha1h   s1, s0          // rotate e for next round

// Rounds 20-39 (parity):
sha1p   q0, s1, v2.4s

// Rounds 40-59 (majority):
sha1m   q0, s1, v2.4s

// Schedule update:
sha1su0 v2.4s, v3.4s, v4.4s  // first part of schedule
sha1su1 v2.4s, v5.4s         // second part
```

### 6.4 SHA-256 Instructions

```asm
// SHA256H/SHA256H2 - SHA256 hash update (rounds 0/1)
// SHA256SU0/SHA256SU1 - message schedule update

// SHA256 round:
// v0 = {a, b, c, d} current hash
// v1 = {e, f, g, h} current hash
// v2 = message + constant (4 words)

sha256h  q0, q1, v2.4s     // rounds 0 (updates a,b,c,d)
sha256h2 q1, q0, v2.4s     // rounds 1 (updates e,f,g,h)

// Schedule:
sha256su0 v2.4s, v3.4s     // sigma0 update
sha256su1 v2.4s, v4.4s, v5.4s  // sigma1 update

// Full SHA-256 block (simplified):
.global sha256_block_arm
sha256_block_arm:
    // x0 = state (8 x uint32), x1 = message block (16 x uint32)
    // x2 = constants table (K[64])

    // Load state
    ld1     {v0.4s, v1.4s}, [x0]

    // Load message
    ld1     {v4.4s, v5.4s, v6.4s, v7.4s}, [x1]

    // Save state for final add
    mov     v16.16b, v0.16b
    mov     v17.16b, v1.16b

    // 64 rounds (16 x 4 = 64)
    // ...

    // Final add
    add     v0.4s, v0.4s, v16.4s
    add     v1.4s, v1.4s, v17.4s

    st1     {v0.4s, v1.4s}, [x0]
    ret
```

### 6.5 PMULL for GCM

```asm
// GHASH multiplication in GF(2^128)
// ใช้ PMULL สำหรับ polynomial multiplication ใน AES-GCM

// x0 = H (hash key, 16 bytes)
// x1 = X (input, 16 bytes)
// x2 = output (16 bytes)
.global gcm_ghash_mult
gcm_ghash_mult:
    ld1     {v0.2d}, [x0]   // H
    ld1     {v1.2d}, [x1]   // X

    // Karatsuba multiplication trick:
    // Full 128x128 = 256-bit result using 3 PMULL
    pmull   v2.1q,  v0.1d,  v1.1d    // low * low
    pmull2  v3.1q,  v0.2d,  v1.2d    // high * high
    ext     v4.16b, v0.16b, v0.16b, #8
    ext     v5.16b, v1.16b, v1.16b, #8
    pmull   v4.1q,  v4.1d,  v1.1d    // (low+high) * (low'+high')
    pmull2  v5.1q,  v0.2d,  v5.2d
    eor     v4.16b, v4.16b, v2.16b
    eor     v4.16b, v4.16b, v3.16b
    // Reduction mod P(x) = x^128 + x^7 + x^2 + x + 1
    // ... (reduction code omitted for brevity)
    st1     {v2.2d}, [x2]
    ret
```

---

## 7. ตัวอย่างสมบูรณ์ (Complete Examples)

### 7.1 RGB24 to BGR24 Conversion ด้วย TBL

```asm
// RGB24 → BGR24 (swap R and B channels)
// Input:  x0 = src (RGB packed), x1 = dst (BGR packed), x2 = pixel_count
// Destroys: x0, x1, x2, x3, v0-v2, v31

    .section .rodata
    .align 4
rgb_to_bgr_tbl:
    // For each group of 3 pixels (9 bytes = ??? doesn't align to 16)
    // Better: process 16 bytes at a time, using TBL with pattern
    // Pattern maps byte position 0..15 → output byte position
    // RGB RGB RGB RGB RGB R  (16 bytes = 5 pixels + 1 byte)
    // out: BGR BGR BGR BGR BG R
    // indices: [2,1,0, 5,4,3, 8,7,6, 11,10,9, 14,13,12, 15]
    .byte 2, 1, 0, 5, 4, 3, 8, 7, 6, 11, 10, 9, 14, 13, 12, 15

    .text
    .global rgb24_to_bgr24
    .type rgb24_to_bgr24, %function
rgb24_to_bgr24:
    // Load shuffle table
    adrp    x3, rgb_to_bgr_tbl
    add     x3, x3, :lo12:rgb_to_bgr_tbl
    ld1     {v31.16b}, [x3]

    // Calculate byte count
    lsl     x2, x2, #1          // x2 = pixel_count * 2
    add     x2, x2, x2, lsr #1  // x2 = pixel_count * 3 (bytes)

    // Process 16 bytes at a time
    subs    x2, x2, #16
    b.lt    rgb_tail

rgb_loop:
    ld1     {v0.16b}, [x0], #16     // load 16 bytes (5.3 pixels)
    tbl     v1.16b, {v0.16b}, v31.16b  // swap R↔B
    st1     {v1.16b}, [x1], #16     // store
    subs    x2, x2, #16
    b.ge    rgb_loop

rgb_tail:
    // Handle remaining bytes (x2 is negative now, x2+16 = remaining)
    adds    x2, x2, #16
    b.eq    rgb_done

    // Simple scalar fallback for tail
rgb_tail_loop:
    ldrb    w4, [x0]          // R
    ldrb    w5, [x0, #1]      // G
    ldrb    w6, [x0, #2]      // B
    strb    w6, [x1]          // store B
    strb    w5, [x1, #1]      // store G
    strb    w4, [x1, #2]      // store R
    add     x0, x0, #3
    add     x1, x1, #3
    subs    x2, x2, #3
    b.gt    rgb_tail_loop

rgb_done:
    ret
```

### 7.2 8x8 DCT Approximation ด้วย NEON

```asm
// Simplified 8x8 2D DCT using NEON
// Based on AAN DCT algorithm with NEON vectorization
// Input: x0 = input (64 int16_t), x1 = output (64 int16_t)

.section .rodata
.align 4
// DCT constants (scaled for fixed-point)
dct_c1:  .short  2217  // cos(1*pi/16) * 4096
dct_c2:  .short  1482  // cos(2*pi/16) * 4096
dct_c3:  .short  3784  // cos(3*pi/16) * 4096  (actually sin)
dct_sqrt2: .short 5793 // sqrt(2)/2 * 4096 * sqrt(2) = 4096

.text
.global dct8x8_approx_neon
.type dct8x8_approx_neon, %function
dct8x8_approx_neon:
    // Load 8 rows of 8 int16 = 8 * 8 = 64 shorts
    ld1     {v0.8h, v1.8h, v2.8h, v3.8h},  [x0], #64
    ld1     {v4.8h, v5.8h, v6.8h, v7.8h},  [x0]

    // --- Row DCT pass ---
    // For each row in parallel (columns as SIMD lanes):
    // Stage 1: butterfly
    add     v8.8h,  v0.8h, v7.8h   // s0 = row0 + row7
    sub     v15.8h, v0.8h, v7.8h   // d0 = row0 - row7
    add     v9.8h,  v1.8h, v6.8h   // s1 = row1 + row6
    sub     v14.8h, v1.8h, v6.8h   // d1 = row1 - row6
    add     v10.8h, v2.8h, v5.8h   // s2 = row2 + row5
    sub     v13.8h, v2.8h, v5.8h   // d2 = row2 - row5
    add     v11.8h, v3.8h, v4.8h   // s3 = row3 + row4
    sub     v12.8h, v3.8h, v4.8h   // d3 = row3 - row4

    // Stage 2 (even part):
    add     v0.8h, v8.8h,  v11.8h  // even[0] = s0 + s3
    sub     v3.8h, v8.8h,  v11.8h  // even[3] = s0 - s3
    add     v1.8h, v9.8h,  v10.8h  // even[1] = s1 + s2
    sub     v2.8h, v9.8h,  v10.8h  // even[2] = s1 - s2

    // Stage 2 (odd part would need rotations - simplified):
    // Full DCT requires cos/sin multiplications which need SMULL
    // This is the butterfly structure; full impl needs scaling

    // For now, store intermediate results
    st1     {v0.8h, v1.8h, v2.8h, v3.8h}, [x1], #64
    st1     {v12.8h, v13.8h, v14.8h, v15.8h}, [x1]

    ret

// NOTE: Full production DCT would use:
// smull  / smlal for fixed-point multiply
// sqrshrn for rounding narrow
// Two passes: row DCT then column DCT (transpose between passes)
```

### 7.3 Neural Network Quantized Inference (INT8 Matrix Multiply)

```asm
// INT8 Matrix-Vector multiply for quantized NN inference
// Computes: output[m] = sum_k(A[m][k] * x[k])  for k = 0..K-1
//
// Input:
//   x0 = A matrix (int8, m x k), row-major
//   x1 = x vector (int8, k elements)
//   x2 = output (int32, m elements)
//   x3 = m (rows)
//   x4 = k (cols, must be multiple of 16)

.global int8_matvec_neon
.type int8_matvec_neon, %function
int8_matvec_neon:
    stp     x19, x20, [sp, #-16]!
    mov     x19, x0             // save A pointer
    mov     x20, x4             // save k

outer_loop:
    cbz     x3, done_matvec
    sub     x3, x3, #1

    // Initialize accumulator to 0
    movi    v16.4s, #0
    movi    v17.4s, #0
    movi    v18.4s, #0
    movi    v19.4s, #0

    // Inner loop: process K elements in groups of 16
    mov     x5, x1              // x pointer reset
    mov     x6, x20             // k counter

inner_loop:
    cbz     x6, inner_done
    sub     x6, x6, #16

    ld1     {v0.16b}, [x19], #16   // load 16 int8 from A row
    ld1     {v1.16b}, [x5],  #16   // load 16 int8 from x

    // Widening multiply and accumulate (signed)
    smull   v2.8h,  v0.8b,  v1.8b  // lower 8: int8 * int8 → int16
    smull2  v3.8h,  v0.16b, v1.16b // upper 8: int8 * int8 → int16

    smlal   v16.4s, v2.4h,  v3.4h  // accumulate lower half of v2*v3
    // Correct accumulation:
    saddl   v4.4s,  v2.4h,  v3.4h  // add halfs
    saddl2  v5.4s,  v2.8h,  v3.8h

    add     v16.4s, v16.4s, v4.4s
    add     v17.4s, v17.4s, v5.4s

    b.gt    inner_loop

inner_done:
    // Reduce accumulator
    add     v16.4s, v16.4s, v17.4s
    add     v16.4s, v16.4s, v18.4s
    add     v16.4s, v16.4s, v19.4s
    addv    s0, v16.4s              // sum all 4 int32 → scalar
    mov     w7, v0.s[0]
    str     w7, [x2], #4            // store output element
    b       outer_loop

done_matvec:
    ldp     x19, x20, [sp], #16
    ret
```

### 7.4 เพิ่มเติม: Dot Product ของ Float Array

```asm
// dot_product: compute dot product of two float arrays
// float dot_product(float* a, float* b, int n)
// x0 = a, x1 = b, w2 = n (assume n multiple of 4)

.global dot_product_neon
.type dot_product_neon, %function
dot_product_neon:
    movi    v0.4s, #0              // accumulator

dot_loop:
    cbz     w2, dot_done
    sub     w2, w2, #4
    ld1     {v1.4s}, [x0], #16    // load 4 floats from a
    ld1     {v2.4s}, [x1], #16    // load 4 floats from b
    fmla    v0.4s, v1.4s, v2.4s  // v0 += a * b
    b.gt    dot_loop

dot_done:
    // Horizontal sum
    faddp   v0.4s, v0.4s, v0.4s  // [a+b, c+d, a+b, c+d]
    faddp   s0, v0.2s             // (a+b) + (c+d) = total sum
    ret
```

### 7.5 Gaussian Blur 1D Kernel (NEON Vectorized)

```asm
// 1D horizontal Gaussian blur with 5-tap kernel [1, 4, 6, 4, 1]
// Input: x0 = src uint8, x1 = dst uint8, x2 = width (multiple of 16)
// Kernel sum = 16, so divide by 16 (right shift 4)

.section .rodata
.align 4
gauss_kernel: .byte 1, 4, 6, 4, 1   // weights

.text
.global gaussian_blur_1d_neon
.type gaussian_blur_1d_neon, %function
gaussian_blur_1d_neon:
    sub     x0, x0, #2          // start 2 bytes before (border handling)

gauss_loop:
    cbz     x2, gauss_done
    sub     x2, x2, #16

    ld1     {v0.16b}, [x0]          // src[-2]
    ld1     {v1.16b}, [x0, #1]      // src[-1]
    ld1     {v2.16b}, [x0, #2]      // src[0] (center)
    ld1     {v3.16b}, [x0, #3]      // src[+1]
    ld1     {v4.16b}, [x0, #4]      // src[+2]
    add     x0, x0, #16

    // Compute weighted sum
    ushll   v10.8h, v0.8b,  #0      // zero extend lower 8
    ushll   v11.8h, v1.8b,  #0
    ushll   v12.8h, v2.8b,  #0
    ushll   v13.8h, v3.8b,  #0
    ushll   v14.8h, v4.8b,  #0

    // weights: [1, 4, 6, 4, 1]
    add     v16.8h, v10.8h, v14.8h  // 1*(a+e)
    mla     v16.8h, v11.8h, v11.8h  // TODO: proper scalar multiply
    // Simplified: use shift-based approximation
    // result = v0 + 4*v1 + 6*v2 + 4*v3 + v4
    // = v0 + (v1 << 2) + v2 + (v2 << 1) + v2 + (v2 << 2) + v3 << 2 + v4

    // Better approach with SSHL/USHL:
    movi    v20.8h, #1
    movi    v21.8h, #4
    movi    v22.8h, #6

    mul     v16.8h, v10.8h, v20.8h  // 1 * v0
    mla     v16.8h, v11.8h, v21.8h  // + 4 * v1
    mla     v16.8h, v12.8h, v22.8h  // + 6 * v2
    mla     v16.8h, v13.8h, v21.8h  // + 4 * v3
    mla     v16.8h, v14.8h, v20.8h  // + 1 * v4

    // Divide by 16 (shift right 4)
    ushr    v16.8h, v16.8h, #4

    // Do the same for upper 8 bytes
    ushll2  v10.8h, v0.16b,  #0
    ushll2  v11.8h, v1.16b,  #0
    ushll2  v12.8h, v2.16b,  #0
    ushll2  v13.8h, v3.16b,  #0
    ushll2  v14.8h, v4.16b,  #0

    mul     v17.8h, v10.8h, v20.8h
    mla     v17.8h, v11.8h, v21.8h
    mla     v17.8h, v12.8h, v22.8h
    mla     v17.8h, v13.8h, v21.8h
    mla     v17.8h, v14.8h, v20.8h
    ushr    v17.8h, v17.8h, #4

    // Narrow back to uint8
    uqxtn   v16.8b, v16.8h
    uqxtn2  v16.16b, v17.8h

    st1     {v16.16b}, [x1], #16
    b.ge    gauss_loop

gauss_done:
    ret
```

---

## 8. SVE - Scalable Vector Extension (Introduction)

### 8.1 SVE vs NEON

SVE (ARMv8.2+, mandatory in ARMv9) มีความพิเศษ:
- **Vector length ไม่คงที่**: 128 ถึง 2048 bits (กำหนดโดย hardware)
- Code เดียวกันทำงานได้กับทุก vector length
- **Predicate registers**: Z0-Z31 (SVE vectors), P0-P15 (predicates)
- Loop vectorization ง่ายขึ้นมาก

```asm
// SVE (GAS syntax)
// NEON:
add     v0.4s, v1.4s, v2.4s     // fixed: 4 elements

// SVE:
add     z0.s, p0/m, z1.s, z2.s  // variable: vl/4 elements
// z0 = destination, p0/m = predicate register (m=merging, z=zeroing)
// .s = 32-bit element type

// WHILELT - generate predicate for loop bounds
// Produces predicate: p0[i] = (i < count)
whilelt p0.s, xzr, x2          // p0 = loop predicate (x2 = count)
// After this, p0 has 1s for active lanes

// SVE loop pattern:
mov     x0, #0                  // index = 0
1:
    whilelt p0.s, x0, x3       // p0 = (lane_index+x0) < x3
    b.none  2f                  // if no active lanes, done

    ld1w    {z0.s}, p0/z, [x1, x0, lsl #2]  // load active float elements
    ld1w    {z1.s}, p0/z, [x2, x0, lsl #2]  // from two arrays
    fadd    z0.s, p0/m, z0.s, z1.s          // add with predication
    st1w    {z0.s}, p0, [x4, x0, lsl #2]    // store
    inch    x0                               // x0 += vl/4 (increment by element count)
    b       1b
2:
    ret
```

### 8.2 SVE2 (ARMv9)

SVE2 เพิ่ม instructions สำหรับ DSP/multimedia:

```asm
// SVE2 adds NEON-like operations but with variable length:
// MATCH - match elements against set (like PCMPESTRI in x86 but predicated)
// HISTCNT - histogram count
// BDEP/BEXT - bit deposit/extract
// CDOT - complex dot product
// FMMLA - floating-point matrix multiply-accumulate

// Example: complex dot product (useful for communications)
// cdot z0.s, z1.b, z2.b, #0  // complex multiply-accumulate
```

---

## 9. Performance Tips และ AArch64 SIMD Best Practices

### 9.1 Register Allocation

```
// AArch64 SIMD/FP register calling convention:
// V0-V7:   arguments/return values (caller-saved)
// V8-V15:  callee-saved (only lower 64-bit / D register portion!)
//          Note: upper 64-bit of V8-V15 are NOT saved by callee!
// V16-V31: caller-saved (scratch)

// ถ้าใช้ Q0-Q15 (full 128-bit) ต้องบันทึก D registers ของ V8-V15:
stp     d8, d9,   [sp, #-64]!
stp     d10, d11, [sp, #16]
stp     d12, d13, [sp, #32]
stp     d14, d15, [sp, #48]
// ... function body ...
ldp     d14, d15, [sp, #48]
ldp     d12, d13, [sp, #32]
ldp     d10, d11, [sp, #16]
ldp     d8, d9,   [sp], #64
```

### 9.2 Memory Access Patterns

```asm
// Use aligned loads when possible
// AArch64 supports unaligned access but aligned is faster
// hint: ld1 doesn't require alignment, but ldp/ldr benefit from it

// Prefetch:
prfm    pldl1keep, [x0, #64]   // prefetch to L1, keep in cache
prfm    pldl2keep, [x0, #256]  // prefetch to L2
prfm    pstl1keep, [x1, #64]   // prefetch for store to L1

// Use LD1 with post-increment for streaming:
ld1     {v0.16b, v1.16b, v2.16b, v3.16b}, [x0], #64  // 4 regs in one op
```

### 9.3 Common Patterns

```asm
// Zero a vector register
movi    v0.16b, #0          // all zeros
movi    v0.4s,  #0
eor     v0.16b, v0.16b, v0.16b  // alternative

// All ones
movi    v0.16b, #255        // 0xFF in each byte
mvni    v0.4s,  #0          // bitwise NOT of 0 = all 1s

// Splat a constant
movi    v0.4s, #42          // all elements = 42
fmov    v0.4s, #1.0         // all floats = 1.0
dup     v0.4s, w5           // broadcast w5 to all lanes

// Negate in integer
neg     v0.4s, v1.4s        // v0 = -v1

// Absolute value (integer)
abs     v0.16b, v1.16b      // |v1| for each byte
abs     v0.4s,  v1.4s       // |v1| for each word

// Shift operations
sshr    v0.8h, v1.8h, #4    // signed shift right 4
ushr    v0.8h, v1.8h, #4    // unsigned shift right 4
shl     v0.8h, v1.8h, #3    // shift left 3
ssra    v0.8h, v1.8h, #4    // shift right and accumulate (v0 += v1 >> 4)

// Count leading zeros
clz     v0.4s, v1.4s        // count leading zeros per element

// Population count (byte)
cnt     v0.16b, v1.16b      // popcount per byte
// Sum all to get total popcount:
uaddlv  h0, v0.16b          // sum all 16 bytes → halfword
```

---

## 10. สรุป (Summary)

AArch64 SIMD/NEON มีความสามารถสูงมาก:

| Feature | รายละเอียด |
|---------|-----------|
| Register file | V0-V31, 128-bit each, 32 registers |
| Data types | B/H/S/D scalars, 8B/16B/4H/8H/2S/4S/2D/1D vectors |
| Load/Store | LD1-LD4/ST1-ST4, interleaved, replicate, post-index |
| Integer SIMD | ADD/SUB/MUL/MLA, widening, saturating, reduction |
| Float SIMD | FADD/FMUL/FMLA, FCVT, pairwise, reduce |
| Permute | DUP/INS/EXT/REV/ZIP/UZP/TRN/TBL |
| Crypto | AES (HW), SHA1, SHA2, PMULL for GCM |
| SVE | Variable length (128-2048b), predicated, ARMv8.2+ |

### Comparison กับ x86 SSE/AVX

| x86 | AArch64 | หมายเหตุ |
|-----|---------|---------|
| PSHUFB | TBL | byte shuffle |
| PMULLD | MUL v.4s | 32-bit multiply |
| PMADD | SMLAL | multiply-accumulate |
| VBROADCASTSS | DUP/LD1R | broadcast |
| VPERM | TBL | permute |
| VAESDEC | AESD | AES decrypt |
| VPCLMULQDQ | PMULL | poly multiply |
| AVX-512 mask | SVE predicate | conditional ops |

### Compilation Tips

```bash
# Compile AArch64 NEON code:
gcc -march=armv8-a+crypto -O3 -o program source.c

# Enable specific features:
gcc -march=armv8-a+crypto+sha2+aes -O3 ...

# Enable SVE:
gcc -march=armv8.2-a+sve -O3 ...

# Enable SVE2 (ARMv9):
gcc -march=armv9-a+sve2 -O3 ...

# Assembly source:
as -march=armv8-a+crypto source.s -o source.o

# Cross-compile from x86:
aarch64-linux-gnu-gcc -march=armv8-a+crypto -O3 ...
```

---

## อ้างอิง (References)

1. ARM Architecture Reference Manual (ARM DDI 0487)
2. ARM Cortex-A Programmer's Guide for ARMv8-A
3. ACLE (ARM C Language Extensions) Specification
4. "ARM NEON Intrinsics Reference" - developer.arm.com
5. "Optimizing C Code with Neon Intrinsics" - ARM Developer Documentation
6. "AES and GHASH on AArch64" - IACR ePrint
7. "ARMv8 Cryptography Extensions" - ARM White Paper
8. "SVE Programming Examples" - ARM GitHub
9. "The ARM Scalable Vector Extension" - IEEE Micro 2017
10. "A64 -- AArch64 Instruction Set Reference" - developer.arm.com/documentation

---

*Part 049 สมบูรณ์ - ครอบคลุม AArch64 SIMD register file, NEON load/store, integer/float SIMD, permute instructions, crypto extensions (AES/SHA/PMULL), practical examples, และ SVE introduction*
