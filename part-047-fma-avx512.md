# Part 047: FMA (Fused Multiply-Add) และ AVX-512

## บทนำ

บทนี้ครอบคลุมสองเทคโนโลยีสำคัญสำหรับ High-Performance Computing:
- **FMA3** (Fused Multiply-Add) — คำนวณ `a*b+c` ใน instruction เดียว
- **AVX-512** — SIMD registers ขนาด 512-bit พร้อม mask registers

เทคโนโลยีทั้งสองนี้เป็นพื้นฐานของ Machine Learning, Signal Processing, และ Scientific Computing สมัยใหม่

---

## ส่วนที่ 1: FMA3 (Fused Multiply-Add) รายละเอียด

### 1.1 ทำไมต้องมี FMA?

การคำนวณ `a * b + c` แบบปกติใช้ 2 operations:
```
mulss xmm0, xmm1      ; xmm0 = a * b   (rounded)
addss xmm0, xmm2      ; xmm0 += c      (rounded อีกครั้ง)
```

ปัญหา: มี **2 rounding operations** ทำให้เกิด floating-point error สะสม

FMA แก้ปัญหานี้โดยคำนวณ `a * b + c` แบบ **infinite precision** แล้ว round ครั้งเดียว:
```
vfmadd132ss xmm0, xmm2, xmm1   ; xmm0 = (xmm0 * xmm1) + xmm2
```

ข้อดีของ FMA:
- **Accuracy สูงกว่า**: round เพียงครั้งเดียว
- **Performance ดีกว่า**: 1 instruction แทน 2
- **Throughput**: บน Haswell+ สามารถทำ 2 FMA per cycle (port 0 และ port 1)

### 1.2 FMA3 Encoding Schemes: 132 / 213 / 231

FMA3 ใช้ระบบตัวเลข 3 หลักเพื่อบอก operand order:
- **1** = destination register (ตัวแรกใน instruction)
- **2** = source1 register (ตัวที่สอง)
- **3** = source2 register หรือ memory (ตัวที่สาม)

```
VFMADD[132|213|231] dest, src1, src2
```

| Encoding | การคำนวณ                     | ตัวอย่าง                      |
|----------|------------------------------|-------------------------------|
| 132      | dest = (dest × src2) + src1  | xmm0 = (xmm0 × xmm2) + xmm1  |
| 213      | dest = (src1 × dest) + src2  | xmm0 = (xmm1 × xmm0) + xmm2  |
| 231      | dest = (src1 × src2) + dest  | xmm0 = (xmm1 × xmm2) + xmm0  |

**ตัวอย่างเปรียบเทียบ:**

```nasm
; ต้องการคำนวณ: result = (a * b) + c
; สมมติ: xmm0 = a, xmm1 = b, xmm2 = c

; ใช้ VFMADD213: dest = (src1 * dest) + src2
; ตีความ: xmm0 = (xmm1 * xmm0) + xmm2
; = (b * a) + c  ✓
vfmadd213ss xmm0, xmm1, xmm2

; ใช้ VFMADD132: dest = (dest * src2) + src1
; ตีความ: xmm0 = (xmm0 * xmm2) + xmm1
; = (a * c) + b  ✗ ไม่ตรงกับที่ต้องการ
vfmadd132ss xmm0, xmm1, xmm2

; ใช้ VFMADD231: dest = (src1 * src2) + dest
; ตีความ: xmm2 = (xmm0 * xmm1) + xmm2
; = (a * b) + c  ✓ แต่ผลไปอยู่ที่ xmm2
vfmadd231ss xmm2, xmm0, xmm1
```

### 1.3 Instruction Reference: VFMADD

#### VFMADD132PS/SD/SS/PD — Fused Multiply-Add

```nasm
; Scalar Single (SS) — 1 float
vfmadd132ss xmm0, xmm1, xmm2      ; xmm0[0] = (xmm0[0] * xmm2[0]) + xmm1[0]
vfmadd213ss xmm0, xmm1, xmm2      ; xmm0[0] = (xmm1[0] * xmm0[0]) + xmm2[0]
vfmadd231ss xmm0, xmm1, xmm2      ; xmm0[0] = (xmm1[0] * xmm2[0]) + xmm0[0]

; Scalar Double (SD) — 1 double
vfmadd132sd xmm0, xmm1, xmm2
vfmadd213sd xmm0, xmm1, xmm2
vfmadd231sd xmm0, xmm1, xmm2

; Packed Single (PS) — 4 floats (XMM) หรือ 8 floats (YMM)
vfmadd132ps xmm0, xmm1, xmm2      ; xmm0[0:3] = (xmm0 * xmm2) + xmm1
vfmadd132ps ymm0, ymm1, ymm2      ; ymm0[0:7] = (ymm0 * ymm2) + ymm1

; Packed Double (PD) — 2 doubles (XMM) หรือ 4 doubles (YMM)
vfmadd132pd xmm0, xmm1, xmm2
vfmadd132pd ymm0, ymm1, ymm2

; Memory operand (เฉพาะ src2)
vfmadd213ss xmm0, xmm1, [rax]     ; src2 จาก memory
vfmadd213ps ymm0, ymm1, [rsi]     ; 8 floats จาก memory
```

#### Latency และ Throughput (Skylake)

| Instruction  | Latency | Throughput |
|-------------|---------|------------|
| VFMADD*SS   | 4 cycles | 0.5 (2/cycle) |
| VFMADD*SD   | 4 cycles | 0.5 |
| VFMADD*PS XMM | 4 cycles | 0.5 |
| VFMADD*PS YMM | 4 cycles | 0.5 |

### 1.4 VFMSUB — Fused Multiply-Subtract

คล้าย VFMADD แต่เป็น `(a*b) - c`

```nasm
; VFMSUB132: dest = (dest * src2) - src1
vfmsub132ss xmm0, xmm1, xmm2     ; xmm0 = (xmm0 * xmm2) - xmm1
vfmsub213ss xmm0, xmm1, xmm2     ; xmm0 = (xmm1 * xmm0) - xmm2
vfmsub231ss xmm0, xmm1, xmm2     ; xmm0 = (xmm1 * xmm2) - xmm0

; Packed versions
vfmsub132ps ymm0, ymm1, ymm2     ; 8 floats
vfmsub132pd ymm0, ymm1, ymm2     ; 4 doubles
```

### 1.5 VFNMADD — Fused Negate-Multiply-Add

คำนวณ `-(a*b) + c` (negate multiply result แล้ว add)

```nasm
; VFNMADD132: dest = -(dest * src2) + src1
vfnmadd132ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm0 * xmm2) + xmm1
vfnmadd213ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm1 * xmm0) + xmm2
vfnmadd231ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm1 * xmm2) + xmm0

; ใช้กับ packed
vfnmadd132ps ymm0, ymm1, ymm2

; ตัวอย่าง: คำนวณ -(a*b) + c
; xmm0 = a, xmm1 = b, xmm2 = c
; ต้องการ: result = -(a * b) + c
vfnmadd213ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm1 * xmm0) + xmm2
                                  ;      = -(b * a) + c ✓
```

### 1.6 VFNMSUB — Fused Negate-Multiply-Subtract

คำนวณ `-(a*b) - c`

```nasm
vfnmsub132ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm0 * xmm2) - xmm1
vfnmsub213ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm1 * xmm0) - xmm2
vfnmsub231ss xmm0, xmm1, xmm2   ; xmm0 = -(xmm1 * xmm2) - xmm0
```

### 1.7 Horizontal FMA Patterns

Horizontal operations รวมค่าใน vector เดียวกัน:

```nasm
; คำนวณ dot product ของ 2 vectors (4 elements)
; a = [a0, a1, a2, a3], b = [b0, b1, b2, b3]
; result = a0*b0 + a1*b1 + a2*b2 + a3*b3

section .data
    align 16
    vec_a   dd 1.0, 2.0, 3.0, 4.0
    vec_b   dd 5.0, 6.0, 7.0, 8.0

section .text
dot_product_4:
    ; Input: rdi = *a, rsi = *b
    ; Output: xmm0 = dot product
    
    vmovaps     xmm0, [rdi]        ; xmm0 = [a0, a1, a2, a3]
    vmovaps     xmm1, [rsi]        ; xmm1 = [b0, b1, b2, b3]
    
    ; element-wise multiply: xmm0 = a * b
    vmulps      xmm0, xmm0, xmm1  ; [a0*b0, a1*b1, a2*b2, a3*b3]
    
    ; horizontal sum
    vhaddps     xmm0, xmm0, xmm0  ; [a0*b0+a1*b1, a2*b2+a3*b3, ...]
    vhaddps     xmm0, xmm0, xmm0  ; [sum, sum, sum, sum]
    
    ; ผลอยู่ใน xmm0[0]
    ret

; Dot product แบบ FMA สำหรับ 8-element vectors (YMM)
dot_product_8:
    ; Input: rdi = *a (8 floats), rsi = *b (8 floats)
    ; Output: xmm0 = dot product
    
    vmovaps     ymm0, [rdi]        ; ymm0 = [a0..a7]
    vmovaps     ymm1, [rsi]        ; ymm1 = [b0..b7]
    
    ; FMA: ymm2 = (ymm0 * ymm1) + 0
    vxorps      ymm2, ymm2, ymm2   ; ymm2 = 0
    vfmadd231ps ymm2, ymm0, ymm1   ; ymm2 = (ymm0 * ymm1) + ymm2
    
    ; Horizontal sum ของ ymm2
    vextractf128 xmm0, ymm2, 1     ; xmm0 = upper 128 bits
    vaddps       xmm0, xmm0, xmm2  ; sum upper + lower
    vhaddps      xmm0, xmm0, xmm0
    vhaddps      xmm0, xmm0, xmm0
    
    vzeroupper                      ; clean upper YMM bits
    ret
```

### 1.8 FMA สำหรับ Neural Network (Dot Product Layer)

```nasm
; Fully-Connected Layer: y = W * x + b
; W: weight matrix [out_size x in_size]
; x: input vector [in_size]
; b: bias vector [out_size]
; y: output vector [out_size]

section .text

; fc_layer_fma:
;   rdi = *W (aligned float array)
;   rsi = *x (aligned float array)
;   rdx = *b (aligned float array)
;   rcx = *y (output, aligned float array)
;   r8  = in_size (must be multiple of 8)
;   r9  = out_size

fc_layer_fma:
    push rbx
    push r12
    push r13
    
    mov     r12, r9         ; r12 = out_size
    mov     r13, r8         ; r13 = in_size
    
    xor     rbx, rbx        ; rbx = output index
    
.outer_loop:
    cmp     rbx, r12
    jge     .done
    
    ; Compute dot product of W[rbx] and x
    ; W row = rdi + rbx * in_size * 4
    mov     rax, rbx
    imul    rax, r13
    shl     rax, 2           ; * sizeof(float)
    lea     r10, [rdi + rax] ; r10 = &W[rbx * in_size]
    
    vxorps  ymm0, ymm0, ymm0 ; accumulator = 0
    
    xor     rax, rax
.inner_loop:
    cmp     rax, r13
    jge     .inner_done
    
    vmovaps ymm1, [r10 + rax*4]  ; load 8 weights
    vmovaps ymm2, [rsi + rax*4]  ; load 8 inputs
    
    ; accumulate: ymm0 += W[row][col:col+8] * x[col:col+8]
    vfmadd231ps ymm0, ymm1, ymm2
    
    add     rax, 8
    jmp     .inner_loop
    
.inner_done:
    ; Horizontal sum ymm0 -> xmm0
    vextractf128 xmm1, ymm0, 1
    vaddps       xmm0, xmm0, xmm1
    vhaddps      xmm0, xmm0, xmm0
    vhaddps      xmm0, xmm0, xmm0
    
    ; Add bias: y[rbx] = dot_product + b[rbx]
    vaddss  xmm0, xmm0, [rdx + rbx*4]
    
    ; Store result
    vmovss  [rcx + rbx*4], xmm0
    
    inc     rbx
    jmp     .outer_loop
    
.done:
    vzeroupper
    pop r13
    pop r12
    pop rbx
    ret
```

### 1.9 FMA vs MUL+ADD: Precision Comparison

```nasm
; ทดสอบ precision:
; คำนวณ (1.0000001 * 1.0000001) + (-1.0000002)
; ผลที่ถูกต้องควรเป็น ~ -1e-13

section .data
    align 16
    val_a   dd 1.0000001
    val_b   dd 1.0000001
    val_c   dd -1.0000002

section .text

; แบบ MUL + ADD (2 roundings)
test_mul_add:
    vmovss  xmm0, [val_a]
    vmovss  xmm1, [val_b]
    vmovss  xmm2, [val_c]
    
    vmulss  xmm0, xmm0, xmm1   ; round 1: a * b
    vaddss  xmm0, xmm0, xmm2   ; round 2: (a*b) + c
    ; ผลอาจมี error มากกว่า
    ret

; แบบ FMA (1 rounding)
test_fma:
    vmovss  xmm0, [val_a]
    vmovss  xmm1, [val_b]
    vmovss  xmm2, [val_c]
    
    ; xmm0 = (xmm0 * xmm1) + xmm2
    vfmadd213ss xmm0, xmm1, xmm2  ; round เพียงครั้งเดียว
    ; ผลแม่นยำกว่า
    ret
```

**ข้อสังเกต:** ใน Catastrophic Cancellation situations (เมื่อ a*b ≈ -c), FMA ให้ผลที่แม่นยำกว่าอย่างมีนัยสำคัญ

---

## ส่วนที่ 2: AVX-512 Introduction

### 2.1 ZMM Registers: ZMM0-ZMM31

AVX-512 เพิ่ม registers ใหม่:

```
ZMM0  - ZMM15 : extension จาก XMM0-XMM15 และ YMM0-YMM15
ZMM16 - ZMM31 : ใหม่เลย (ไม่มี XMM/YMM counterpart ใน legacy mode)
```

โครงสร้าง ZMM register:
```
ZMM0 (512-bit):
┌──────────────────────────────────────────────────────────────────┐
│                          ZMM0 [511:0]                            │
├─────────────────────────────────┬────────────────────────────────┤
│         YMM0 [255:0]            │     upper [511:256]            │
├─────────────────┬───────────────┤                                │
│   XMM0 [127:0]  │ [255:128]     │                                │
└─────────────────┴───────────────┴────────────────────────────────┘
```

ความจุ ZMM:
- 16 x float (32-bit)
- 8 x double (64-bit)
- 64 x int8
- 32 x int16
- 16 x int32
- 8 x int64

### 2.2 Mask Registers: K0-K7 (Opmask)

AVX-512 มี 8 mask registers ขนาด 64-bit:
- **K0**: ไม่ใช้เป็น mask (ใช้เป็น "no masking" — all elements active)
- **K1-K7**: ใช้เป็น opmask ได้

```nasm
; การใช้ mask register
kmovw   k1, eax           ; load 16-bit mask จาก eax ไป k1
kmovd   k1, eax           ; load 32-bit mask
kmovq   k1, rax           ; load 64-bit mask

; Mask operations
kandw   k1, k2, k3        ; k1 = k2 AND k3 (16-bit)
korw    k1, k2, k3        ; k1 = k2 OR k3
kxorw   k1, k2, k3        ; k1 = k2 XOR k3
knotw   k1, k2            ; k1 = NOT k2
kandnw  k1, k2, k3        ; k1 = k2 ANDNOT k3

; Compare และสร้าง mask
vcmpps  k1, zmm0, zmm1, 1  ; k1 = (zmm0 < zmm1) ต่อ element
```

### 2.3 EVEX Prefix Encoding

AVX-512 ใช้ EVEX prefix (4 bytes) แทน VEX (2-3 bytes):

```
EVEX Prefix Structure:
Byte 0: 0x62                    ; EVEX marker
Byte 1: R R̃ X B R' 0 0 m m     ; register extensions, map
Byte 2: W v̂ v̂ v̂ v̂ 1 p p       ; operand size, vvvv, PP
Byte 3: z L' L b Ṽ V' a a a   ; masking, rounding, broadcast, opmask

ส่วนสำคัญ:
- aaa  (3 bits): opmask register selector (k0-k7)
- z    (1 bit) : zero-masking (1) vs merge-masking (0)
- b    (1 bit) : broadcast หรือ embedded rounding
- L'L  (2 bits): vector length (00=128, 01=256, 10=512)
- R'   (1 bit) : extends EVEX.R (ZMM16-31)
```

### 2.4 AVX-512 Subsets

| Subset | คำย่อ | ความหมาย | ตัวอย่าง instruction |
|--------|-------|----------|---------------------|
| F      | Foundation | Basic 512-bit ops | VADDPS ZMM |
| CD     | Conflict Detection | gather/scatter conflict | VPCONFLICTD |
| BW     | Byte/Word | 8/16-bit operations | VPADDW ZMM |
| DQ     | Double/Quad | 64-bit operations | VPMULLQ |
| VL     | Vector Length | 128/256-bit masking | VADDPS XMM {k1} |
| VNNI   | Vector Neural Net | INT8 dot product | VPDPBUSD |
| BITALG | Bit Algorithms | bit manipulation | VPSHUFBITQMB |
| VBMI   | Byte Move + Interact | byte permute | VPERMB |
| VBMI2  | Byte Move + Int v2 | more byte ops | VPSHLDW |
| IFMA   | Integer FMA 52-bit | 52-bit integer FMA | VPMADD52LUQ |

**CPUID Feature Flags:**
```
CPUID.7.0:EBX[16] = AVX-512F
CPUID.7.0:EBX[17] = AVX-512DQ
CPUID.7.0:EBX[26] = AVX-512PF (Phi only)
CPUID.7.0:EBX[27] = AVX-512ER (Phi only)
CPUID.7.0:EBX[28] = AVX-512CD
CPUID.7.0:EBX[30] = AVX-512BW
CPUID.7.0:EBX[31] = AVX-512VL
CPUID.7.0:ECX[11] = AVX-512VNNI
```

---

## ส่วนที่ 3: AVX-512F Fundamentals

### 3.1 VMOVAPS ZMM: 512-bit Move

```nasm
; Load / Store 512-bit aligned
vmovaps zmm0, [rdi]          ; load 64 bytes จาก aligned address
vmovaps [rsi], zmm1          ; store 64 bytes ไป aligned address

; Unaligned load/store
vmovups zmm0, [rdi]
vmovups [rsi], zmm0

; Move with mask (merge-masking)
vmovaps zmm0 {k1}, [rdi]    ; โหลดเฉพาะ element ที่ k1 bit = 1
                              ; element อื่นเป็น 0 (ตามค่าเดิมใน zmm0)

; Move with zero-mask
vmovaps zmm0 {k1}{z}, [rdi] ; โหลดเฉพาะ element ที่ k1 = 1
                              ; element อื่นถูกตั้งเป็น 0

; Broadcast: 1 value ไปทุก element
vbroadcastss zmm0, xmm1      ; กระจาย xmm1[0] ไปทุก 16 float slots
vbroadcastss zmm0, [rdi]     ; กระจาย scalar จาก memory
vbroadcastsd zmm0, xmm1      ; กระจาย double ไป 8 slots
```

### 3.2 VADDPS ZMM: 16x Float at Once

```nasm
; Basic 512-bit add
vaddps zmm0, zmm1, zmm2      ; zmm0[0:15] = zmm1[0:15] + zmm2[0:15]

; Add with memory operand + broadcast
vaddps zmm0, zmm1, [rdi]{1to16}  ; broadcast 1 float to all 16, then add

; Comparison
vaddpd zmm0, zmm1, zmm2      ; 8 doubles
vaddss xmm0, xmm1, xmm2      ; ยังทำงานได้ใน AVX-512 context
```

### 3.3 Opmask Operations

```nasm
; Zero-masking vs Merge-masking

; Setup mask
mov     eax, 0b1010101010101010   ; ทุก other element
kmovw   k1, eax                    ; k1 = 0xAAAA (16-bit)

; Merge-masking (ค่าเดิมในตำแหน่งที่ mask=0)
vmovaps zmm0, [some_default]       ; ค่า default
vaddps  zmm0 {k1}, zmm1, zmm2     ; เพิ่มเฉพาะ element ที่ k1=1
                                    ; zmm0 elements ที่ k1=0 ไม่เปลี่ยน

; Zero-masking (ตั้งเป็น 0 ที่ตำแหน่ง mask=0)
vaddps  zmm0 {k1}{z}, zmm1, zmm2  ; element ที่ k1=0 จะเป็น 0.0

; Example: conditional negation
; ลบค่าที่ < 0 ออก (absolute value)
vcmpps  k1, zmm0, [zero_vec], 1   ; k1 = (zmm0 < 0.0) per element
vxorps  zmm0 {k1}, zmm0, [sign_bit]  ; flip sign เฉพาะ negative elements
```

### 3.4 Embedded Rounding: {rn-sae}/{rd-sae}/{ru-sae}/{rz-sae}

AVX-512 อนุญาต override rounding mode ต่อ instruction:

```nasm
; Rounding modes:
; {rn-sae}  = Round to Nearest, Suppress All Exceptions
; {rd-sae}  = Round Down (toward -inf), SAE
; {ru-sae}  = Round Up (toward +inf), SAE
; {rz-sae}  = Round toward Zero (truncate), SAE

; ใช้กับ scalar operations
vaddss  xmm0, xmm1, xmm2, {rn-sae}  ; add with nearest rounding
vaddsd  xmm0, xmm1, xmm2, {rd-sae}  ; add rounding down
vmulss  xmm0, xmm1, xmm2, {rz-sae}  ; multiply truncate
vdivss  xmm0, xmm1, xmm2, {ru-sae}  ; divide rounding up

; ใช้กับ packed ZMM operations
vaddps  zmm0, zmm1, zmm2, {rn-sae}
vmulpd  zmm0, zmm1, zmm2, {rd-sae}

; FMA with embedded rounding
vfmadd213ps zmm0, zmm1, zmm2, {rz-sae}
```

### 3.5 Exception Suppression: {sae}

```nasm
; {sae} = Suppress All Exceptions
; ใช้สำหรับ operations ที่อาจเกิด floating-point exceptions
; แต่เราต้องการผลโดยไม่สนใจ exception

vsqrtss xmm0, xmm1, xmm2, {sae}   ; sqrt โดยไม่ raise exception แม้ input < 0
vcvtss2si eax, xmm0, {sae}         ; convert float to int โดยไม่ raise exception
vrsqrt14ss xmm0, xmm1, xmm2        ; approximate reciprocal sqrt
vcvtps2dq zmm0, zmm1, {sae}        ; bulk convert 16 floats to 16 ints
```

### 3.6 VCOMPRESSPS / VEXPANDPS

```nasm
; VCOMPRESSPS: เก็บเฉพาะ element ที่ mask=1 แบบ compact (no gaps)
; Input:  zmm0 = [a, b, c, d, e, f, g, h, ...]  (16 floats)
; Mask:   k1   = 0b0000000001010101   (bits 0,2,4 set)
; Output: zmm1 = [a, c, e, ?, ?, ...]  (3 values, rest undefined)

kmovw       k1, eax                 ; setup mask
vcompressps zmm1 {k1}, zmm0        ; compress to zmm1
; หรือ store ลง memory
vcompressps [rdi] {k1}, zmm0       ; store compressed values ลง memory

; VEXPANDPS: ขยาย compressed values ไปยัง masked positions
; ใส่ค่าจาก source ไปยัง positions ที่ mask=1
; positions ที่ mask=0 คงค่าเดิม (merge-masking)

vexpandps   zmm0 {k1}, [rdi]       ; load และ expand จาก memory
vexpandps   zmm0 {k1}, zmm1        ; expand จาก register

; ตัวอย่าง: filter array เก็บเฉพาะ positive values
filter_positive:
    ; rdi = *src (input array, 16 floats)
    ; rsi = *dst (output array)
    ; rdx = *count (จำนวน elements ที่ filter ผ่าน)
    
    vmovaps     zmm0, [rdi]
    vxorps      zmm1, zmm1, zmm1    ; zmm1 = 0.0
    
    ; สร้าง mask: k1 = (zmm0 > 0.0)
    vcmpps      k1, zmm0, zmm1, 6   ; 6 = GT (greater than)
    
    ; Compress and store
    vcompressps [rsi] {k1}, zmm0
    
    ; นับจำนวน bits ที่ set ใน mask
    kmovw       eax, k1
    popcnt      eax, eax
    mov         [rdx], eax
    
    ret
```

### 3.7 VPCOMPRESSD / VPEXPANDD (Integer Compress/Expand)

```nasm
; VPCOMPRESSD: compress 32-bit integers
vpcompressd zmm1 {k1}, zmm0        ; compress dwords
vpcompressd [rdi] {k1}, zmm0      ; compress to memory

; VPCOMPRESSQ: compress 64-bit integers
vpcompressq zmm1 {k1}, zmm0

; VPEXPANDD: expand 32-bit integers
vpexpandd   zmm0 {k1}, [rdi]
vpexpandd   zmm0 {k1}{z}, zmm1    ; zero-mask: unmasked = 0

; ตัวอย่าง: filter array of int32
filter_nonzero_int32:
    ; rdi = *src (16 int32 values)
    ; rsi = *dst
    ; output: count in eax
    
    vmovdqu32   zmm0, [rdi]
    vpxord      zmm1, zmm1, zmm1    ; zmm1 = 0
    
    ; mask: k1 = (zmm0 != 0)
    vpcmpd      k1, zmm0, zmm1, 4   ; 4 = NEQ
    
    vpcompressd [rsi] {k1}, zmm0
    
    kmovw       eax, k1
    popcnt      eax, eax
    ret
```

### 3.8 VSCATTERDPS / VSCATTERDPD — Scatter to Memory

```nasm
; Scatter: เขียนค่าจาก register ไปยัง memory addresses ที่ระบุในอีก register

; VSCATTERDPS: scatter floats ใช้ dword indices
; syntax: VSCATTERDPS vm32z {k1}, zmm1
;   vm32z = [base + zmm_index * scale + disp]
;   {k1}  = mask (ทุก element ที่ scatter ต้องใช้ mask ≠ k0)

; ตัวอย่าง: scatter 16 floats ไปยัง positions ที่ระบุ
scatter_example:
    ; zmm0 = values to scatter
    ; zmm1 = indices (int32)
    ; rdi  = base address
    ; k1   = all-ones mask (16-bit)
    
    mov     eax, 0xFFFF
    kmovw   k1, eax                     ; mask ทุก element
    
    ; scatter: memory[rdi + zmm1[i]*4] = zmm0[i]
    vscatterdps [rdi + zmm1*4] {k1}, zmm0
    
    ; NOTE: หลัง scatter k1 จะถูก clear (zeroed) โดยอัตโนมัติ
    ret

; VSCATTERDPD: scatter doubles
scatter_doubles:
    mov     eax, 0xFF                   ; 8-bit mask สำหรับ 8 doubles
    kmovb   k1, eax
    vscatterdpd [rdi + ymm1*8] {k1}, zmm0  ; ymm1 = 8 dword indices
    ret
```

### 3.9 VGATHERDPS / VGATHERDPD — Gather from Memory

```nasm
; Gather: อ่านค่าจาก scattered memory locations เข้า register

; VGATHERDPS: gather floats ใช้ dword indices
; syntax: VGATHERDPS zmm1 {k1}, vm32z

; ตัวอย่าง: gather 16 floats จาก non-contiguous locations
gather_example:
    ; zmm1 = indices (int32)
    ; rdi  = base address
    ; output: zmm0 = gathered values
    
    mov     eax, 0xFFFF
    kmovw   k1, eax                     ; mask ทุก element
    
    ; gather: zmm0[i] = memory[rdi + zmm1[i]*4]
    vgatherdps zmm0 {k1}, [rdi + zmm1*4]
    
    ; k1 จะถูก clear หลัง gather
    ret

; Gather แบบ conditional (เฉพาะบาง elements)
gather_masked:
    ; k1 = mask สำหรับ elements ที่ต้องการ
    kmovw   k1, eax                     ; k1 = desired element mask
    vgatherdps zmm0 {k1}, [rdi + zmm1*4]
    ; zmm0 elements ที่ k1=1 ถูกโหลด
    ; zmm0 elements ที่ k1=0 ไม่เปลี่ยน (merge behavior)
    ret

; VGATHERDPD: gather doubles (ymm indices → zmm doubles)
gather_doubles:
    mov     eax, 0xFF
    kmovb   k1, eax
    vgatherdpd zmm0 {k1}, [rdi + ymm1*8]
    ret
```

---

## ส่วนที่ 4: AVX-512BW/DQ

### 4.1 AVX-512BW: Byte/Word Operations

BW extension เพิ่ม 512-bit byte และ word operations:

```nasm
; 512-bit byte operations
vpaddб  zmm0, zmm1, zmm2      ; 64 bytes: add (b = bytes)
vpsubb  zmm0, zmm1, zmm2      ; subtract 64 bytes
vpminub zmm0, zmm1, zmm2      ; unsigned min 64 bytes
vpmaxub zmm0, zmm1, zmm2      ; unsigned max 64 bytes
vpcmpeqb k1, zmm0, zmm1       ; compare 64 bytes → 64-bit mask
vpcmpb   k1, zmm0, zmm1, imm  ; compare with predicate

; 512-bit word operations (16-bit)
vpaddw  zmm0, zmm1, zmm2      ; 32 words: add
vpsubw  zmm0, zmm1, zmm2      ; subtract 32 words
vpmullw zmm0, zmm1, zmm2      ; multiply low 32 words
vpmulhw zmm0, zmm1, zmm2      ; multiply high 32 words (signed)
vpmulhuw zmm0, zmm1, zmm2     ; multiply high 32 words (unsigned)
vpcmpeqw k1, zmm0, zmm1       ; compare 32 words → 32-bit mask

; VPADDW ZMM example: 32 shorts at once
section .data
    align 64
    shorts_a  times 32 dw 100
    shorts_b  times 32 dw 200
    shorts_c  times 32 dw 0

section .text
add_32_shorts:
    vmovdqu16   zmm0, [shorts_a]    ; load 32 int16 values
    vmovdqu16   zmm1, [shorts_b]
    vpaddw      zmm2, zmm0, zmm1    ; zmm2[0:31] = a[i] + b[i]
    vmovdqu16   [shorts_c], zmm2    ; store result
    ret
```

### 4.2 AVX-512DQ: Double/Quadword Operations

```nasm
; 64-bit (quadword) operations
vpmullq zmm0, zmm1, zmm2      ; 8 x 64-bit integer multiply (low 64)
vpmuludq zmm0, zmm1, zmm2     ; unsigned 32x32→64-bit multiply
vpord   zmm0, zmm1, zmm2      ; bitwise OR 512-bit
vpandd  zmm0, zmm1, zmm2      ; bitwise AND 512-bit (dword)
vpandq  zmm0, zmm1, zmm2      ; bitwise AND 512-bit (qword)
vpxorq  zmm0, zmm1, zmm2      ; XOR qword

; Float-to-integer conversions (DQ)
vcvtps2qq  zmm0, ymm1         ; 8 floats → 8 int64
vcvttpd2qq zmm0, zmm1         ; 8 doubles → 8 int64 (truncate)
vcvtqq2ps  ymm0, zmm1         ; 8 int64 → 8 floats
```

### 4.3 64-bit Mask Operations (KMOVQ etc.)

```nasm
; BW extension เพิ่ม 64-bit mask operations
; สำหรับ operations ที่มี 64 elements (เช่น byte operations บน ZMM)

kmovq   k1, rax              ; load 64-bit mask จาก rax
kmovq   rax, k1              ; store 64-bit mask ไป rax
kandq   k1, k2, k3           ; 64-bit AND
korq    k1, k2, k3            ; 64-bit OR
kxorq   k1, k2, k3           ; 64-bit XOR
knotq   k1, k2               ; 64-bit NOT
ktestq  k1, k2               ; test (set flags)
kshiftlq k1, k2, imm8        ; shift left
kshiftrq k1, k2, imm8        ; shift right

; Example: process every other byte in 512-bit vector
process_even_bytes:
    ; สร้าง mask: bits 0,2,4,...,62 = 1 (even positions)
    mov     rax, 0x5555555555555555  ; alternating 1s
    kmovq   k1, rax
    
    ; ทำ operation เฉพาะ even bytes
    vpaddб  zmm0 {k1}, zmm1, zmm2   ; add เฉพาะ even bytes
    ret
```

---

## ส่วนที่ 5: AVX-512VL (Vector Length Extension)

### 5.1 Opmask บน 128/256-bit Registers

VL extension ช่วยให้ใช้ AVX-512 features (masking, broadcast, rounding) กับ XMM/YMM registers:

```nasm
; VL: ใช้ opmask กับ XMM (128-bit)
mov     eax, 0xF             ; 4-bit mask สำหรับ 4 floats
kmovb   k1, eax
vaddps  xmm0 {k1}, xmm1, xmm2   ; add เฉพาะ 4 elements ใน XMM ที่ mask=1

; VL: ใช้ opmask กับ YMM (256-bit)
mov     eax, 0xFF            ; 8-bit mask สำหรับ 8 floats
kmovb   k1, eax
vaddps  ymm0 {k1}, ymm1, ymm2   ; add เฉพาะ 8 elements ใน YMM ที่ mask=1

; VL: zero-masking บน YMM
vaddps  ymm0 {k1}{z}, ymm1, ymm2  ; unmasked elements = 0

; VL: Broadcast บน XMM/YMM (เหมือน AVX2 แต่ใช้ EVEX)
vbroadcastss xmm0, [rdi]     ; broadcast 1 float to 4 (EVEX encoding)
vbroadcastss ymm0, [rdi]     ; broadcast 1 float to 8

; VL: compress/expand บน XMM/YMM
vcompressps xmm0 {k1}, xmm1  ; compress 4 floats
vcompressps ymm0 {k1}, ymm1  ; compress 8 floats

; Masked load/store บน YMM (ไม่ต้องการ AVX-512 VL ถ้า YMM)
vmovaps xmm0 {k1}, [rdi]     ; masked load XMM
vmovaps ymm0 {k1}, [rdi]     ; masked load YMM
```

### 5.2 ประโยชน์ของ VL

```nasm
; ก่อน VL: ต้องใช้ ZMM ทั้งหมดเพื่อใช้ masking
; หลัง VL: ใช้ XMM/YMM ได้เลย ประหยัด registers

; ตัวอย่าง: conditional copy 4 floats
copy_if_positive_128:
    ; rdi = *src, rsi = *dst
    ; เฉพาะ elements > 0.0 จะถูก copy
    
    vmovaps xmm0, [rdi]
    vxorps  xmm1, xmm1, xmm1     ; xmm1 = 0.0
    
    ; k1 = (xmm0 > 0.0)
    vcmpps  k1, xmm0, xmm1, 6    ; 6 = GT
    
    ; store เฉพาะ elements ที่ > 0
    vmovaps [rsi] {k1}, xmm0     ; VL allows XMM mask store
    ret
```

---

## ส่วนที่ 6: AVX-512VNNI (Vector Neural Network Instructions)

### 6.1 VPDPBUSD: Dot Product uint8 × int8

VNNI instruction ที่สำคัญที่สุดสำหรับ AI/ML inference:

```nasm
; VPDPBUSD: 
; dst[i] += sum_j( src1[4i+j] * src2[4i+j] )
; src1: unsigned 8-bit (uint8)
; src2: signed 8-bit (int8)
; dst: signed 32-bit (int32) accumulator
; ทำ 4 multiply-accumulate ต่อ element, 16 elements per ZMM

; Syntax:
VPDPBUSD zmm0, zmm1, zmm2
; zmm0[i] += (zmm1[4i+0] * zmm2[4i+0]) +
;             (zmm1[4i+1] * zmm2[4i+1]) +
;             (zmm1[4i+2] * zmm2[4i+2]) +
;             (zmm1[4i+3] * zmm2[4i+3])
; สำหรับ i = 0..15 (16 int32 outputs)
; = 64 multiply-accumulates ใน 1 instruction!

; VPDPWSSD: signed int16 × int16 → int32 accumulate
VPDPWSSD zmm0, zmm1, zmm2
; zmm0[i] += (zmm1[2i+0] * zmm2[2i+0]) +
;             (zmm1[2i+1] * zmm2[2i+1])
```

### 6.2 INT8 Inference Acceleration

```nasm
; INT8 Quantized Matrix Multiply
; สำหรับ Neural Network Inference
; 
; ทำ: C[row][col] += A[row][k:k+64] . B[k:k+64][col]
; A: uint8 activations (input)
; B: int8 weights
; C: int32 accumulator

section .text

; vnni_dot_product_64:
;   rdi = *A (64 uint8 values, aligned 64)
;   rsi = *B (64 int8 values, aligned 64)
;   output: rax = dot product (int32)
vnni_dot_product_64:
    ; Load A (uint8) into zmm0
    vmovdqu8  zmm0, [rdi]
    ; Load B (int8) into zmm1
    vmovdqu8  zmm1, [rsi]
    
    ; Initialize accumulator to 0
    vpxord    zmm2, zmm2, zmm2
    
    ; VPDPBUSD: zmm2 += VNNI(zmm0, zmm1)
    ; คำนวณ 16 int32 dot products (แต่ละอัน sum of 4 products)
    ; = 64 multiply-accumulates
    vpdpbusd  zmm2, zmm0, zmm1
    
    ; Horizontal sum of 16 int32 values
    ; zmm2 = [c0, c1, c2, ..., c15]
    ; เราต้องการ c0+c1+...+c15
    
    ; Extract high 256 bits
    vextracti64x4 ymm0, zmm2, 1     ; ymm0 = zmm2[255:128] (top 8 ints)
    vpaddd        ymm0, ymm0, ymm2  ; ymm0 = sum of pairs
    
    ; Horizontal sum of ymm0 (8 int32)
    vextracti128  xmm1, ymm0, 1
    vpaddd        xmm0, xmm0, xmm1  ; xmm0 = sum of 4
    vpshufd       xmm1, xmm0, 0x4E  ; swap pairs
    vpaddd        xmm0, xmm0, xmm1  ; sum of 2
    vpshufd       xmm1, xmm0, 0xB1  ; swap adjacent
    vpaddd        xmm0, xmm0, xmm1  ; final sum
    
    vmovd         eax, xmm0          ; eax = result
    vzeroupper
    ret

; Full INT8 matrix multiply row:
; out[col] += sum_k(A[k] * W[col][k])  สำหรับ col = 0..15
;
; rdi = *activations (64 uint8, aligned)
; rsi = *weights (64x16 int8 matrix, row-major, aligned)
; rdx = *accumulators (16 int32, aligned)

vnni_matmul_row:
    vmovdqu8    zmm0, [rdi]           ; load 64 activations
    vmovdqu32   zmm2, [rdx]           ; load 16 int32 accumulators
    
    ; Process 4 weight rows at a time
    ; (เนื่องจาก VPDPBUSD ทำ 4-way dot product)
    ; สำหรับตัวอย่างนี้เราทำ 16 weight rows
    
    mov         rcx, 16              ; 16 columns to compute
    xor         r8, r8               ; weight row offset
    
.col_loop:
    ; Load 64 weights for this column
    vmovdqu8    zmm1, [rsi + r8]
    
    ; VNNI dot product: accumulate into zmm2
    vpdpbusd    zmm2, zmm0, zmm1
    
    add         r8, 64               ; next weight row (64 bytes)
    dec         rcx
    jnz         .col_loop
    
    ; Store results
    vmovdqu32   [rdx], zmm2
    vzeroupper
    ret
```

---

## ส่วนที่ 7: ตัวอย่าง Code

### 7.1 CPUID Check for AVX-512 Support

```nasm
; ตรวจสอบว่า CPU รองรับ AVX-512 หรือไม่

section .text
global check_avx512_support

; check_avx512_support:
; Return: eax = feature bits
;   bit 0: AVX-512F
;   bit 1: AVX-512DQ
;   bit 2: AVX-512BW
;   bit 3: AVX-512VL
;   bit 4: AVX-512VNNI
check_avx512_support:
    push    rbx
    push    rcx
    push    rdx
    
    ; ตรวจสอบว่า CPUID leaf 7 มีอยู่
    mov     eax, 0
    cpuid
    cmp     eax, 7
    jl      .no_avx512
    
    ; CPUID leaf 7, sub-leaf 0
    mov     eax, 7
    mov     ecx, 0
    cpuid
    
    ; EBX contains AVX-512 flags
    ; bit 16: AVX-512F
    ; bit 17: AVX-512DQ
    ; bit 26: AVX-512PF
    ; bit 27: AVX-512ER
    ; bit 28: AVX-512CD
    ; bit 30: AVX-512BW
    ; bit 31: AVX-512VL
    
    xor     eax, eax                ; result = 0
    
    bt      ebx, 16                 ; test AVX-512F
    jnc     .check_dq
    or      eax, 1
    
.check_dq:
    bt      ebx, 17                 ; test AVX-512DQ
    jnc     .check_bw
    or      eax, 2
    
.check_bw:
    bt      ebx, 30                 ; test AVX-512BW
    jnc     .check_vl
    or      eax, 4
    
.check_vl:
    bt      ebx, 31                 ; test AVX-512VL
    jnc     .check_vnni
    or      eax, 8
    
.check_vnni:
    ; ECX has VNNI at bit 11
    bt      ecx, 11                 ; test AVX-512VNNI
    jnc     .done
    or      eax, 16
    
.done:
    pop     rdx
    pop     rcx
    pop     rbx
    ret
    
.no_avx512:
    xor     eax, eax
    pop     rdx
    pop     rcx
    pop     rbx
    ret

; ตรวจสอบ OS support สำหรับ ZMM (XSAVE)
check_zmm_os_support:
    ; ต้องตรวจ XSAVE และ OSFXSR ก่อน
    ; OS ต้อง set XCR0[7:5] = 111b สำหรับ ZMM support
    
    ; CPUID leaf 1: ECX bit 26 = XSAVE, bit 27 = OSXSAVE
    mov     eax, 1
    cpuid
    
    bt      ecx, 27                 ; OSXSAVE bit
    jnc     .no_zmm
    
    ; ตรวจ XCR0
    xor     ecx, ecx
    xgetbv                          ; eax = XCR0[31:0]
    
    ; Check bits 5,6,7 (opmask, ZMM upper, ZMM hi16)
    and     eax, 0xE0               ; 0b11100000
    cmp     eax, 0xE0
    sete    al
    movzx   eax, al
    ret
    
.no_zmm:
    xor     eax, eax
    ret
```

### 7.2 16x Float Multiply-Add Loop (AVX-512 FMA)

```nasm
; คำนวณ c[i] = a[i] * scalar + b[i] สำหรับ n elements
; เป็น SAXPY (Scalar Alpha X Plus Y) แบบ AVX-512

section .text
global avx512_saxpy

; avx512_saxpy:
;   rdi = *a (float array, 64-byte aligned)
;   rsi = *b (float array, 64-byte aligned)
;   rdx = *c (output float array, 64-byte aligned)
;   xmm0 = alpha (scalar)
;   r8   = n (element count, multiple of 16)

avx512_saxpy:
    vbroadcastss zmm4, xmm0         ; zmm4 = [alpha, alpha, ..., alpha] (16x)
    
    xor         rax, rax             ; rax = index
    
.loop:
    cmp         rax, r8
    jge         .done
    
    ; Load 16 floats from a and b
    vmovaps     zmm0, [rdi + rax*4]  ; zmm0 = a[rax:rax+15]
    vmovaps     zmm1, [rsi + rax*4]  ; zmm1 = b[rax:rax+15]
    
    ; FMA: c = alpha * a + b
    ; vfmadd213ps: zmm0 = (zmm4 * zmm0) + zmm1
    vfmadd213ps zmm0, zmm4, zmm1     ; zmm0 = alpha * a + b
    
    ; Store result
    vmovaps     [rdx + rax*4], zmm0
    
    add         rax, 16
    jmp         .loop
    
.done:
    vzeroupper
    ret

; Unrolled version (4x unroll) for higher throughput
avx512_saxpy_unrolled:
    vbroadcastss zmm15, xmm0         ; zmm15 = alpha (preserved across loop)
    
    xor         rax, rax
    
    ; Process 64 floats per iteration (4x 16)
.loop:
    add         rax, 64
    cmp         rax, r8
    jg          .tail
    sub         rax, 64
    
    vmovaps     zmm0, [rdi + rax*4 + 0]
    vmovaps     zmm1, [rdi + rax*4 + 64]
    vmovaps     zmm2, [rdi + rax*4 + 128]
    vmovaps     zmm3, [rdi + rax*4 + 192]
    
    vfmadd213ps zmm0, zmm15, [rsi + rax*4 + 0]
    vfmadd213ps zmm1, zmm15, [rsi + rax*4 + 64]
    vfmadd213ps zmm2, zmm15, [rsi + rax*4 + 128]
    vfmadd213ps zmm3, zmm15, [rsi + rax*4 + 192]
    
    vmovaps     [rdx + rax*4 + 0],   zmm0
    vmovaps     [rdx + rax*4 + 64],  zmm1
    vmovaps     [rdx + rax*4 + 128], zmm2
    vmovaps     [rdx + rax*4 + 192], zmm3
    
    add         rax, 64
    jmp         .loop
    
.tail:
    ; Handle remaining elements (< 64) ด้วย scalar หรือ mask
    sub         rax, 64
    ; ... (tail handling code)
    
    vzeroupper
    ret
```

### 7.3 Masked Conditional Add

```nasm
; เพิ่มค่า bias เฉพาะ elements ที่มากกว่า threshold

section .data
    align 64
    threshold_val   dd 0.5
    bias_val        dd 1.0

section .text
global conditional_add

; conditional_add:
;   rdi = *data (16 floats, 64-byte aligned)
;   rsi = *result (output, 64-byte aligned)
;   n = 16 (fixed in this example)

conditional_add:
    vmovaps     zmm0, [rdi]          ; load 16 floats
    
    ; Broadcast threshold and bias
    vbroadcastss zmm1, [threshold_val]  ; zmm1 = [0.5 x 16]
    vbroadcastss zmm2, [bias_val]       ; zmm2 = [1.0 x 16]
    
    ; สร้าง mask: k1 = (data[i] > threshold)
    ; predicate 6 = GT (greater than)
    vcmpps      k1, zmm0, zmm1, 6
    
    ; Conditional add: data[i] += bias ถ้า data[i] > threshold
    ; merge-masking: elements ที่ k1=0 จะไม่เปลี่ยน
    vaddps      zmm0 {k1}, zmm0, zmm2
    
    ; Store result
    vmovaps     [rsi], zmm0
    
    vzeroupper
    ret

; ตัวอย่างที่ซับซ้อนกว่า: ReLU activation function
; ReLU(x) = max(x, 0.0)
relu_avx512:
    ; rdi = *input (16 floats)
    ; rsi = *output
    
    vmovaps     zmm0, [rdi]
    vpxord      zmm1, zmm1, zmm1       ; zmm1 = 0.0 (all zeros)
    
    ; max(input, 0.0) ใช้ VMAXPS
    vmaxps      zmm0, zmm0, zmm1
    
    vmovaps     [rsi], zmm0
    vzeroupper
    ret

; Clamp values to [min, max] range
clamp_avx512:
    ; rdi = *data (16 floats)
    ; rsi = *result
    ; xmm0 = min_val
    ; xmm1 = max_val
    
    vbroadcastss zmm2, xmm0           ; zmm2 = min_val (16x)
    vbroadcastss zmm3, xmm1           ; zmm3 = max_val (16x)
    
    vmovaps     zmm0, [rdi]
    vmaxps      zmm0, zmm0, zmm2      ; clamp to min
    vminps      zmm0, zmm0, zmm3      ; clamp to max
    vmovaps     [rsi], zmm0
    
    vzeroupper
    ret
```

### 7.4 AVX-512 Memcpy

```nasm
; High-performance memcpy ใช้ AVX-512
; 64 bytes per store operation

section .text
global avx512_memcpy

; avx512_memcpy:
;   rdi = *dst (64-byte aligned)
;   rsi = *src (64-byte aligned)
;   rdx = n (byte count, multiple of 64)

avx512_memcpy:
    ; ตรวจสอบ n = 0
    test    rdx, rdx
    jz      .done
    
    xor     rax, rax
    
    ; Unrolled 4x: copy 256 bytes per iteration
.loop:
    cmp     rax, rdx
    jge     .done
    
    ; Check if we have at least 256 bytes left
    lea     rcx, [rax + 256]
    cmp     rcx, rdx
    jg      .single_copy
    
    ; 4x unrolled copy (256 bytes)
    vmovaps zmm0, [rsi + rax]
    vmovaps zmm1, [rsi + rax + 64]
    vmovaps zmm2, [rsi + rax + 128]
    vmovaps zmm3, [rsi + rax + 192]
    
    vmovntps [rdi + rax],       zmm0   ; non-temporal store (bypass cache)
    vmovntps [rdi + rax + 64],  zmm1
    vmovntps [rdi + rax + 128], zmm2
    vmovntps [rdi + rax + 192], zmm3
    
    add     rax, 256
    jmp     .loop
    
.single_copy:
    ; Handle remaining 64-byte chunks
    cmp     rax, rdx
    jge     .done
    
    vmovaps zmm0, [rsi + rax]
    vmovntps [rdi + rax], zmm0
    
    add     rax, 64
    jmp     .single_copy
    
.done:
    ; Flush non-temporal stores
    sfence
    vzeroupper
    ret

; Version สำหรับ unaligned buffers
avx512_memcpy_unaligned:
    test    rdx, rdx
    jz      .done
    
    xor     rax, rax
    
.loop:
    cmp     rax, rdx
    jge     .done
    
    vmovups zmm0, [rsi + rax]
    vmovups [rdi + rax], zmm0
    
    add     rax, 64
    jmp     .loop
    
.done:
    vzeroupper
    ret

; Masked memcpy สำหรับ non-multiple-of-64 size
avx512_memcpy_masked:
    ; rdi = *dst
    ; rsi = *src
    ; rdx = n (byte count, any size)
    
    xor     rax, rax
    
.loop:
    ; คำนวณ remaining bytes
    mov     rcx, rdx
    sub     rcx, rax
    jle     .done
    
    cmp     rcx, 64
    jge     .full_copy
    
    ; Partial copy: สร้าง byte mask
    ; rcx = remaining bytes (1-63)
    mov     r8, 1
    shl     r8, cl
    dec     r8                      ; r8 = (1 << rcx) - 1 = bitmask for rcx bytes
    kmovq   k1, r8
    
    vmovdqu8 zmm0 {k1}{z}, [rsi + rax]   ; masked load
    vmovdqu8 [rdi + rax] {k1}, zmm0       ; masked store
    jmp     .done
    
.full_copy:
    vmovups zmm0, [rsi + rax]
    vmovups [rdi + rax], zmm0
    
    add     rax, 64
    jmp     .loop
    
.done:
    vzeroupper
    ret
```

### 7.5 VNNI INT8 Dot Product

```nasm
; INT8 Dot Product สำหรับ Deep Learning Inference
; คำนวณ score = sum(activation[i] * weight[i]) สำหรับ i = 0..255

section .data
    align 64
    ; Test data
    activations times 256 db 1      ; uint8 activations (all 1)
    weights     times 256 db 2      ; int8 weights (all 2)
    expected    dd 512               ; expected: 256 * 1 * 2 = 512

section .text
global vnni_int8_dot_256

; vnni_int8_dot_256:
;   rdi = *activations (256 uint8, 64-byte aligned)
;   rsi = *weights (256 int8, 64-byte aligned)
;   return: eax = dot product (int32)

vnni_int8_dot_256:
    ; Initialize 4 accumulators (4 x ZMM)
    vpxord  zmm8,  zmm8,  zmm8    ; acc0 = 0
    vpxord  zmm9,  zmm9,  zmm9    ; acc1 = 0
    vpxord  zmm10, zmm10, zmm10   ; acc2 = 0
    vpxord  zmm11, zmm11, zmm11   ; acc3 = 0
    
    ; Process 256 bytes in 4 chunks of 64 bytes each
    ; Chunk 0: bytes 0-63
    vmovdqu8    zmm0, [rdi + 0]
    vmovdqu8    zmm1, [rsi + 0]
    vpdpbusd    zmm8, zmm0, zmm1
    
    ; Chunk 1: bytes 64-127
    vmovdqu8    zmm2, [rdi + 64]
    vmovdqu8    zmm3, [rsi + 64]
    vpdpbusd    zmm9, zmm2, zmm3
    
    ; Chunk 2: bytes 128-191
    vmovdqu8    zmm4, [rdi + 128]
    vmovdqu8    zmm5, [rsi + 128]
    vpdpbusd    zmm10, zmm4, zmm5
    
    ; Chunk 3: bytes 192-255
    vmovdqu8    zmm6, [rdi + 192]
    vmovdqu8    zmm7, [rsi + 192]
    vpdpbusd    zmm11, zmm6, zmm7
    
    ; Sum all accumulators
    vpaddd      zmm8, zmm8, zmm9    ; acc = acc0 + acc1
    vpaddd      zmm10, zmm10, zmm11  ; acc = acc2 + acc3
    vpaddd      zmm8, zmm8, zmm10   ; final acc = all 4
    
    ; Horizontal sum of 16 int32 values in zmm8
    vextracti64x4 ymm0, zmm8, 1    ; ymm0 = upper 8 ints
    vpaddd        ymm0, ymm0, ymm8  ; ymm0 = sum of pairs (8 ints)
    
    vextracti128  xmm1, ymm0, 1    ; xmm1 = upper 4 ints
    vpaddd        xmm0, xmm0, xmm1  ; xmm0 = sum of 4 ints
    
    vpshufd       xmm1, xmm0, 0x4E ; xmm1 = [xmm0[2],xmm0[3],xmm0[0],xmm0[1]]
    vpaddd        xmm0, xmm0, xmm1  ; xmm0[0] = sum of 4 ints: [0+2, 1+3, ...]
    
    vpshufd       xmm1, xmm0, 0xB1 ; xmm1 = [xmm0[1],xmm0[0],...]
    vpaddd        xmm0, xmm0, xmm1  ; xmm0[0] = final sum
    
    vmovd         eax, xmm0         ; eax = result
    vzeroupper
    ret

; Generalized VNNI matmul kernel
; Matrix A: M x K (uint8 activations)
; Matrix B: K x N (int8 weights, transposed: N x K)
; Matrix C: M x N (int32 output, accumulated)
; This computes C += A * B

vnni_matmul_16x16:
    ; rdi = *A (M=16 x K=64 uint8 matrix, row-major)
    ; rsi = *B (N=16 x K=64 int8 matrix, row-major, already transposed)
    ; rdx = *C (M=16 x N=16 int32 matrix, row-major)
    
    ; Compute one row of C at a time
    xor     r8, r8                  ; row index = 0
    
.row_loop:
    cmp     r8, 16
    jge     .done_rows
    
    ; Load row r8 of A (64 uint8 bytes)
    vmovdqu8 zmm0, [rdi + r8*64]
    
    ; Compute 16 outputs for this row
    ; C[r8][0:15] += dot(A[r8], B[col]) for col = 0:15
    
    ; Load 16 accumulators for this row
    vmovdqu32 zmm8, [rdx + r8*64]   ; 16 x int32
    
    ; Compute against each of the 16 weight rows
    ; (B is K x N = 64 x 16, stored as 16 rows of 64 int8)
    ; Process all 16 B rows simultaneously using VPDPBUSD
    
    xor     r9, r9                   ; B row = 0
.col_loop:
    cmp     r9, 16
    jge     .col_done
    
    vmovdqu8 zmm1, [rsi + r9*64]    ; load one B row (64 int8)
    vpdpbusd zmm8, zmm0, zmm1       ; accumulate
    
    inc     r9
    jmp     .col_loop
    
.col_done:
    ; Store updated row of C
    vmovdqu32 [rdx + r8*64], zmm8
    
    inc     r8
    jmp     .row_loop
    
.done_rows:
    vzeroupper
    ret
```

---

## ส่วนที่ 8: Performance Tips และ Pitfalls

### 8.1 ZMM vs YMM: เมื่อไหรควรใช้อะไร

```nasm
; ZMM ดีกว่า เมื่อ:
; - ประมวลผลข้อมูลจำนวนมาก (> 16 floats ต่อ iteration)
; - ต้องการ masking features
; - ใช้ gather/scatter

; YMM ดีกว่า เมื่อ:
; - ข้อมูลน้อย (< 8 floats)
; - ต้องการหลีกเลี่ยง ZMM frequency throttling
; - Code ต้องทำงานบน CPUs ที่ไม่มี AVX-512

; IMPORTANT: ZMM operations อาจ throttle CPU frequency!
; Intel Skylake/Cascade Lake: ใช้ "heavy" ZMM → reduced frequency
; Solution: ใช้ vzeroupper หลัง ZMM code ก่อน transition ไป non-AVX code
; หรือใช้ vzeroall ถ้าต้องการ clear ทั้งหมด

avx512_function:
    ; ... ZMM operations ...
    vzeroupper          ; ต้องมีก่อน ret เพื่อ clear upper YMM/ZMM
    ret
```

### 8.2 Alignment Requirements

```nasm
; AVX-512 alignment:
; VMOVAPS  ZMM = ต้องการ 64-byte alignment
; VMOVUPS  ZMM = ไม่ต้องการ alignment (ช้ากว่าเล็กน้อย)
; VMOVDQU8 ZMM = ไม่ต้องการ alignment

; ใน data section:
section .data
    align 64
    my_array  dd 1.0, 2.0, 3.0, 4.0    ; 64-byte aligned

; Dynamic allocation (C):
; float* arr = aligned_alloc(64, n * sizeof(float));
; หรือ
; posix_memalign(&arr, 64, n * sizeof(float));

; ใน assembly, ตรวจสอบ alignment:
check_aligned:
    test    rdi, 63             ; test ว่า lower 6 bits = 0
    jnz     .not_aligned
    ; aligned!
.not_aligned:
    ; handle unaligned case
```

### 8.3 Scatter/Gather Performance Notes

```nasm
; Gather/Scatter มี latency สูง:
; - VGATHERDPS (16 elements): ~20-30 cycles ขึ้นอยู่กับ cache hits
; - ถ้า data อยู่ใน L1 cache: ~7 cycles per element
; - ถ้า data อยู่ใน DRAM: ~70-200 cycles per element

; Best practices:
; 1. ใช้ gather เมื่อ access pattern ไม่ regular จริงๆ
; 2. ถ้า access pattern regular → ใช้ vmovaps ดีกว่า
; 3. Prefetch data ก่อน gather ถ้าเป็นไปได้

; Prefetch example:
gather_with_prefetch:
    ; zmm_idx = indices
    ; Prefetch first
    ; (VPSCATTERQD และ VSCATTERPREFETCH ไม่ใช่ standard instruction)
    ; ใช้ scalar prefetch แทน:
    vpextrd eax, xmm_idx, 0
    prefetcht0 [rdi + rax*4]
    vpextrd eax, xmm_idx, 1
    prefetcht0 [rdi + rax*4]
    ; ... etc
    
    ; จากนั้น gather
    mov     eax, 0xFFFF
    kmovw   k1, eax
    vgatherdps zmm0 {k1}, [rdi + zmm_idx*4]
    ret
```

### 8.4 FMA Dependency Chain

```nasm
; Bad: dependency chain ทำให้ pipeline stall
; latency 4 cycles ต่อ iteration
fma_serial:
    vxorps  ymm0, ymm0, ymm0    ; acc = 0
.loop:
    vfmadd231ps ymm0, ymm1, [rsi]  ; ymm0 depends on previous ymm0
    add     rsi, 32
    dec     rcx
    jnz     .loop

; Good: multiple accumulators → หลีกเลี่ยง dependency
; throughput 2 FMA/cycle ได้จริง
fma_parallel:
    vxorps  ymm0, ymm0, ymm0    ; acc0 = 0
    vxorps  ymm1, ymm1, ymm1    ; acc1 = 0
    vxorps  ymm2, ymm2, ymm2    ; acc2 = 0
    vxorps  ymm3, ymm3, ymm3    ; acc3 = 0
.loop:
    vfmadd231ps ymm0, ymm4, [rsi]      ; independent accumulators
    vfmadd231ps ymm1, ymm5, [rsi+32]
    vfmadd231ps ymm2, ymm6, [rsi+64]
    vfmadd231ps ymm3, ymm7, [rsi+96]
    add     rsi, 128
    sub     rcx, 4
    jg      .loop
    
    ; Combine accumulators
    vaddps  ymm0, ymm0, ymm1
    vaddps  ymm2, ymm2, ymm3
    vaddps  ymm0, ymm0, ymm2
    vzeroupper
    ret
```

---

## ส่วนที่ 9: Real-World Application Examples

### 9.1 Fast Sigmoid Function (Neural Network Activation)

```nasm
; Sigmoid(x) = 1 / (1 + exp(-x))
; ใช้ polynomial approximation:
; sig(x) ≈ 0.5 + 0.25*x - 0.0208*x^3 + ... (for small x)
; หรือ clamp + scale approach

section .data
    align 64
    sig_half    times 16 dd 0.5
    sig_quarter times 16 dd 0.25
    sig_one     times 16 dd 1.0
    sig_neg_one times 16 dd -1.0

section .text

; fast_sigmoid_avx512:
;   zmm0 = input (16 floats)
;   return zmm0 = sigmoid(input)
fast_sigmoid_approx:
    ; Clamp input to [-4, 4] for approximation accuracy
    vmovaps     zmm1, [sig_neg_one + rip]   ; -1.0 (clamp -4 scaled)
    vmovaps     zmm2, [sig_one + rip]        ; 1.0
    ; (Note: ใช้ actual clamp values)
    
    ; Simple tanh-based: sigmoid(x) = (tanh(x/2) + 1) / 2
    ; approximate: sigmoid(x) ≈ 0.5 + 0.25*x สำหรับ |x| < 1
    
    vmovaps     zmm1, [sig_quarter + rip]
    vmovaps     zmm2, [sig_half + rip]
    
    ; result = x * 0.25 + 0.5
    vfmadd213ps zmm0, zmm1, zmm2            ; zmm0 = (zmm1 * zmm0) + zmm2
                                             ;      = (0.25 * x) + 0.5
    
    ; Clamp to [0, 1]
    vxorps      zmm3, zmm3, zmm3           ; 0.0
    vmovaps     zmm4, [sig_one + rip]
    vmaxps      zmm0, zmm0, zmm3           ; max(x, 0.0)
    vminps      zmm0, zmm0, zmm4           ; min(x, 1.0)
    
    ret
```

### 9.2 Softmax Function

```nasm
; Softmax(x[i]) = exp(x[i]) / sum(exp(x[j]))
; ใช้ AVX-512 สำหรับ 16 values พร้อมกัน

; Note: exp() ต้องใช้ library function หรือ approximation
; ตัวอย่างนี้แสดง structure โดยใช้ placeholder

; softmax_avx512:
;   rdi = *input (16 floats)
;   rsi = *output (16 floats)

softmax_avx512_structure:
    ; Step 1: Find max for numerical stability
    vmovaps     zmm0, [rdi]                 ; load inputs
    vreduceps   zmm1, zmm0, 0              ; (ไม่ใช่ reduce, แสดง structure)
    
    ; ใน reality ต้องทำ horizontal max:
    vextractf32x8 ymm1, zmm0, 1            ; upper 8
    vmaxps      ymm0, ymm0, ymm1           ; max pairwise
    vextractf128 xmm1, ymm0, 1
    vmaxps      xmm0, xmm0, xmm1
    ; ... reduce to scalar max
    
    ; Step 2: Subtract max (for stability)
    vbroadcastss zmm_max, xmm0             ; broadcast max
    vsubps      zmm_input, zmm_input, zmm_max
    
    ; Step 3: Compute exp() for each element
    ; (ใช้ Intel SVML หรือ polynomial approximation)
    ; call exp_avx512  ; สมมติว่ามี function นี้
    
    ; Step 4: Compute sum
    ; horizontal sum ของ 16 exp values
    
    ; Step 5: Divide each exp by sum
    vbroadcastss zmm_sum, xmm_sum
    vdivps      zmm_output, zmm_exp, zmm_sum
    
    vmovaps     [rsi], zmm_output
    vzeroupper
    ret
```

---

## ส่วนที่ 10: Debugging และ Testing

### 10.1 Print ZMM Register Values

```nasm
; Helper function สำหรับ debug: print ZMM register
; (ใช้กับ C printf)

section .data
    zmm_fmt db "ZMM: [%f, %f, %f, %f, %f, %f, %f, %f, %f, %f, %f, %f, %f, %f, %f, %f]", 10, 0

section .text
extern printf

; print_zmm:
;   zmm0 = register to print
print_zmm:
    sub     rsp, 136        ; 16 floats * 4 bytes + alignment
    
    ; Store ZMM to stack
    vmovaps [rsp + 8], zmm0
    
    ; Load 16 floats as double สำหรับ printf (float args promoted to double)
    lea     rdi, [zmm_fmt]
    vcvtss2sd xmm0, xmm0, [rsp + 8 + 0]    ; float[0] → double
    vcvtss2sd xmm1, xmm1, [rsp + 8 + 4]    ; float[1] → double
    ; ... load xmm2-xmm15 similarly
    
    mov     al, 16          ; 16 float args (for varargs)
    call    printf
    
    add     rsp, 136
    ret
```

### 10.2 Unit Test Structure

```nasm
; Simple unit test framework สำหรับ AVX-512 code

section .data
    test_pass_msg db "PASS", 10, 0
    test_fail_msg db "FAIL: expected %f, got %f", 10, 0

section .text
extern printf, fabsf

; assert_float_equal:
;   xmm0 = actual
;   xmm1 = expected
;   xmm2 = tolerance
assert_float_equal:
    vsubss  xmm0, xmm0, xmm1   ; diff = actual - expected
    vandps  xmm0, xmm0, [abs_mask]  ; abs(diff)
    vucomiss xmm0, xmm2          ; compare with tolerance
    jbe     .pass
    
    ; Fail: print message
    lea     rdi, [test_fail_msg]
    vmovss  xmm0, xmm1          ; expected
    vmovss  xmm1, [rsp + 8]     ; actual (saved)
    call    printf
    jmp     .done
    
.pass:
    lea     rdi, [test_pass_msg]
    call    printf
.done:
    ret
```

---

## สรุป

| Feature | Instruction | Performance |
|---------|-------------|-------------|
| FMA3 | VFMADD213PS YMM | 2/cycle (8 float/cycle) |
| AVX-512F | VADDPS ZMM | 2/cycle (32 float/cycle) |
| AVX-512 FMA | VFMADD213PS ZMM | 2/cycle (32 float/cycle) |
| AVX-512 Masked | VADDPS ZMM {k1} | 2/cycle (16-32 float/cycle) |
| VNNI | VPDPBUSD ZMM | 2/cycle (128 int8 mul/cycle) |
| Gather | VGATHERDPS ZMM | variable (~10-30 cycle) |

### Key Takeaways

1. **FMA3** ให้ทั้ง accuracy ดีขึ้น (1 rounding) และ performance ดีขึ้น (1 instruction)

2. **Encoding 132/213/231** ต่างกันตรงตำแหน่งของ multiply operands และ addend

3. **AVX-512 ZMM** = 16 floats พร้อมกัน, มี 32 registers (ZMM0-ZMM31)

4. **Opmask registers K0-K7** ช่วย conditional operations โดยไม่ต้องทำ branch

5. **Zero-masking** ({k}{z}) ตั้ง unmasked elements เป็น 0 vs **merge-masking** ที่เก็บค่าเดิม

6. **Embedded rounding** ช่วย control rounding mode ต่อ instruction

7. **VCOMPRESSPS/VEXPANDPS** ช่วย pack/unpack data ตาม mask อย่างมีประสิทธิภาพ

8. **Scatter/Gather** ให้ random memory access ใน 1 instruction

9. **VNNI (VPDPBUSD)** = 64 multiply-accumulates ต่อ instruction เหมาะสำหรับ INT8 inference

10. ต้องมี **vzeroupper** ก่อน return จาก function ที่ใช้ YMM/ZMM เพื่อหลีกเลี่ยง performance penalty

---

## References

- Intel 64 and IA-32 Architectures Software Developer's Manual, Vol. 2 (Instruction Set Reference)
- Intel Intrinsics Guide: https://www.intel.com/content/www/us/en/docs/intrinsics-guide/
- Agner Fog's Instruction Tables: https://agner.org/optimize/
- "Hacker's Delight" by Henry S. Warren Jr.
- Intel AVX-512 White Paper: "Intel AVX-512 Instructions"
- "Intel 64 and IA-32 Architectures Optimization Reference Manual"

---

*Part 047 — FMA และ AVX-512 | Assembly Language Course*
*เนื้อหาถัดไป: Part 048 — AVX-512 Advanced: BITALG, VBMI2, และ Crypto Extensions*
