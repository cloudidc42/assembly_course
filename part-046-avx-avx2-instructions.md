# Part 046: AVX (Advanced Vector Extensions) และ AVX2

## บทนำ

AVX (Advanced Vector Extensions) เป็นส่วนขยายของ SSE ที่ Intel แนะนำใน Sandy Bridge (2011)
โดยเพิ่มขนาด register จาก 128-bit (XMM) เป็น 256-bit (YMM) และเปลี่ยนรูปแบบ encoding
ให้รองรับ 3-operand non-destructive operations ผ่าน VEX prefix

AVX2 (Haswell, 2013) เพิ่ม 256-bit integer operations และ gather instructions
FMA3 (Fused Multiply-Add) มักมาพร้อมกับ AVX2 บน Intel

---

## 1. AVX Overview

### 1.1 YMM Registers

YMM registers คือ 256-bit registers ที่ขยายจาก XMM (128-bit):

```
YMM0  [255 .............. 128 | 127 .............. 0]
       Upper 128-bit lane      Lower 128-bit lane (= XMM0)
```

มี YMM0-YMM15 (x86-64) ทั้งหมด 16 registers
- YMM0-YMM7: caller-saved (Windows: YMM6-YMM15 callee-saved ส่วน upper halves)
- แต่ละ YMM register ซ้อนทับกับ XMM register ที่ตรงกัน

**Layout สำหรับ float (YMMO = 8x float32):**
```
YMM0: [float7 | float6 | float5 | float4 | float3 | float2 | float1 | float0]
       bits 255:224  ...                                         bits 31:0
```

**Layout สำหรับ double (YMM0 = 4x float64):**
```
YMM0: [double3 | double2 | double1 | double0]
       bits 255:192  bits 191:128  bits 127:64  bits 63:0
```

### 1.2 VEX Encoding: Non-Destructive 3-Operand

SSE แบบเดิม (2-operand destructive):
```nasm
; SSE: dst = dst OP src (ทำลาย dst เดิม)
addps xmm0, xmm1        ; xmm0 = xmm0 + xmm1  (xmm0 เดิมหาย)
```

AVX ด้วย VEX prefix (3-operand non-destructive):
```nasm
; AVX: dst = src1 OP src2 (dst เป็น register ใหม่, src1 และ src2 ไม่เปลี่ยน)
vaddps ymm0, ymm1, ymm2  ; ymm0 = ymm1 + ymm2
vaddps ymm0, ymm0, ymm1  ; ymm0 = ymm0 + ymm1 (เทียบเท่า SSE แบบเดิม)
```

ประโยชน์:
- ลด false dependencies
- code ที่อ่านง่ายขึ้น
- compiler สามารถ allocate registers ได้อิสระขึ้น

### 1.3 Upper YMM Lanes: Zero vs Non-Zero

นี่คือจุดสำคัญมากที่มือใหม่มักข้ามไป:

**Legacy SSE instruction (ไม่มี VEX prefix):**
- เขียน XMM register → upper 128 bits ของ YMM **ไม่เปลี่ยน**
- อาจทำให้เกิด "dirty upper" state

**AVX instruction ด้วย VEX prefix บน 128-bit operand:**
- เขียน XMM register → upper 128 bits ของ YMM **zero out อัตโนมัติ**
- เรียกว่า "zero-extension behavior"

```nasm
; Legacy SSE: ไม่ zero upper half
movaps xmm0, [rsi]       ; bits 127:0 เปลี่ยน, bits 255:128 ไม่เปลี่ยน (อันตราย!)

; AVX 128-bit: zero upper half อัตโนมัติ
vmovaps xmm0, [rsi]      ; bits 127:0 เปลี่ยน, bits 255:128 = 0 (ปลอดภัย)

; AVX 256-bit: เขียนทั้ง 256 bits
vmovaps ymm0, [rsi]      ; bits 255:0 เปลี่ยนทั้งหมด
```

### 1.4 VZEROUPPER และ VZEROALL

**ปัญหา: SSE/AVX Transition Penalty**

เมื่อ CPU พบ legacy SSE instruction หลังจาก 256-bit AVX instruction
หาก upper YMM bits ไม่เป็น zero จะเกิด penalty ใหญ่มาก (ประมาณ 70-100 cycles
บน Haswell/Broadwell ก่อนจะ pipeline flushed)

เหตุผล: CPU ต้องตรวจสอบและจัดการ "merged" register state

**วิธีแก้ไข: VZEROUPPER**

```nasm
; VZEROUPPER: clear upper 128 bits ของ YMM0-YMM15 ทั้งหมด
; แต่ lower 128 bits (XMM) ไม่เปลี่ยน
vzeroupper

; VZEROALL: clear ทั้ง YMM0-YMM15 ทั้งหมด (รวม lower 128 bits)
; ใช้เวลานานกว่า VZEROUPPER
vzeroall
```

**กฎทั่วไป:**
```nasm
; ก่อน return จาก function ที่ใช้ AVX
my_avx_function:
    push rbp
    mov rbp, rsp
    ; ... ใช้ YMM registers ...
    vzeroupper          ; <-- ต้องทำก่อน return !!!
    pop rbp
    ret

; หรือก่อนเรียก library function ที่ไม่รู้ว่าใช้ AVX หรือเปล่า
call some_library_func  ; อาจเรียก legacy SSE code
; ถ้า YMM dirty อยู่ จะเกิด transition penalty !!!
```

**เมื่อใดต้องใช้ VZEROUPPER:**
1. ก่อน return จาก AVX function ที่ถูกเรียกจาก non-AVX code
2. ก่อนเรียก legacy SSE library functions
3. ก่อน context switch (OS จัดการ)
4. ทั่วไป: เมื่อเปลี่ยนจาก AVX เป็น SSE code

### 1.5 VEX Prefix บน SSE Instructions

AVX encoding ให้ใช้ VEX prefix กับ SSE instructions เดิมได้:

```nasm
; SSE (128-bit, destructive)
addps xmm0, xmm1         ; xmm0 += xmm1

; AVX VEX-encoded SSE (128-bit, non-destructive, zero upper)
vaddps xmm0, xmm1, xmm2  ; xmm0 = xmm1 + xmm2, YMM0 upper = 0

; AVX (256-bit)
vaddps ymm0, ymm1, ymm2  ; ymm0 = ymm1 + ymm2
```

การใช้ VEX prefix กับ 128-bit ป้องกัน dirty upper state โดยอัตโนมัติ

---

## 2. AVX 256-bit Float Instructions

### 2.1 Move Instructions

```nasm
; =====================================================
; VMOVAPS - Move Aligned Packed Single (256-bit)
; Address ต้อง 32-byte aligned
; =====================================================
vmovaps ymm0, [rsi]          ; load 256-bit aligned
vmovaps [rdi], ymm0          ; store 256-bit aligned

; VMOVAPD - Move Aligned Packed Double (256-bit)
vmovapd ymm0, [rsi]          ; load 4 doubles, 32-byte aligned

; VMOVUPS - Move Unaligned Packed Single (256-bit)
; ช้ากว่า aligned เล็กน้อย (modern CPUs แทบไม่ต่าง)
vmovups ymm0, [rsi]          ; load 256-bit unaligned
vmovups [rdi], ymm0          ; store 256-bit unaligned

; VMOVUPD - Move Unaligned Packed Double (256-bit)
vmovupd ymm0, [rsi]          ; load 4 doubles unaligned

; Move with zero upper (128-bit VEX)
vmovaps xmm0, [rsi]          ; load 128-bit, zero upper 128 bits of YMM0
```

### 2.2 Arithmetic: 8x Float (Single Precision)

```nasm
; VADDPS - Add Packed Single (256-bit = 8 floats)
vaddps ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] + ymm2[i], i=0..7

; VSUBPS - Subtract Packed Single
vsubps ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] - ymm2[i]

; VMULPS - Multiply Packed Single
vmulps ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] * ymm2[i]

; VDIVPS - Divide Packed Single (ช้ากว่า mul มาก)
vdivps ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] / ymm2[i]

; Memory operands (ไม่จำเป็นต้อง load ก่อน)
vaddps ymm0, ymm1, [rsi]     ; ymm0 = ymm1 + memory[rsi] (32-byte aligned)
vmulps ymm0, ymm0, [rsi]     ; ymm0 *= memory[rsi]
```

### 2.3 Arithmetic: 4x Double (Double Precision)

```nasm
; VADDPD - Add Packed Double (256-bit = 4 doubles)
vaddpd ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] + ymm2[i], i=0..3

; VSUBPD - Subtract Packed Double
vsubpd ymm0, ymm1, ymm2

; VMULPD - Multiply Packed Double
vmulpd ymm0, ymm1, ymm2

; VDIVPD - Divide Packed Double
vdivpd ymm0, ymm1, ymm2
```

### 2.4 Square Root และ Approximations

```nasm
; VSQRTPS - Square Root Packed Single (precise, ช้า)
vsqrtps ymm0, ymm1           ; ymm0[i] = sqrt(ymm1[i])
vsqrtps ymm0, [rsi]          ; ymm0[i] = sqrt(mem[i])

; VSQRTPD - Square Root Packed Double
vsqrtpd ymm0, ymm1

; VRSQRTPS - Reciprocal Square Root Approximation (เร็วมาก, ~12-bit accurate)
; ผล: ymm0[i] ≈ 1.0 / sqrt(ymm1[i])
; ใช้กับ Newton-Raphson refinement สำหรับความแม่นยำสูงขึ้น
vrsqrtps ymm0, ymm1

; VRCPPS - Reciprocal Approximation (~12-bit accurate)
; ผล: ymm0[i] ≈ 1.0 / ymm1[i]
vrcpps ymm0, ymm1

; Newton-Raphson refinement สำหรับ VRSQRTPS:
; rsqrt_refined(x) = rsqrt(x) * (1.5 - 0.5*x*rsqrt(x)^2)
vrsqrtps ymm2, ymm1          ; ymm2 = approx 1/sqrt(x)
vmulps ymm3, ymm1, [half]    ; ymm3 = 0.5*x
vmulps ymm4, ymm2, ymm2      ; ymm4 = rsqrt^2
vmulps ymm3, ymm3, ymm4      ; ymm3 = 0.5*x*rsqrt^2
vsubps ymm3, [one_five], ymm3 ; ymm3 = 1.5 - 0.5*x*rsqrt^2
vmulps ymm0, ymm2, ymm3      ; result = rsqrt * (1.5 - ...)
```

### 2.5 Compare Instructions

```nasm
; VCMPPS - Compare Packed Single with immediate predicate
; Format: vcmpps dst, src1, src2, imm8
; imm8 กำหนด comparison predicate:
;   0 = EQ (equal)
;   1 = LT (less than)
;   2 = LE (less than or equal)
;   3 = UNORD (unordered)
;   4 = NEQ (not equal)
;   5 = NLT (not less than)
;   6 = NLE (not less or equal)
;   7 = ORD (ordered)

vcmpps ymm0, ymm1, ymm2, 0   ; ymm0[i] = 0xFFFFFFFF if ymm1[i] == ymm2[i], else 0
vcmpps ymm0, ymm1, ymm2, 1   ; ymm0[i] = mask if ymm1[i] < ymm2[i]
vcmpps ymm0, ymm1, ymm2, 4   ; ymm0[i] = mask if ymm1[i] != ymm2[i]

; ใช้ symbolic names (NASM supports):
vcmpps ymm0, ymm1, ymm2, CMP_EQ_OQ   ; equal ordered quiet
vcmpps ymm0, ymm1, ymm2, CMP_LT_OS   ; less than ordered signaling

; VCMPPD - Compare Packed Double
vcmppd ymm0, ymm1, ymm2, 1   ; less than

; ใช้ผลลัพธ์เป็น mask สำหรับ bitwise AND:
vcmpps ymm_mask, ymm_a, ymm_b, 1   ; mask = (a < b)
vandps ymm_result, ymm_a, ymm_mask  ; select a where a < b, else 0
```

### 2.6 Min/Max

```nasm
; VMINPS - Minimum Packed Single (returns min of each pair)
vminps ymm0, ymm1, ymm2      ; ymm0[i] = min(ymm1[i], ymm2[i])
vminps ymm0, ymm0, [rsi]     ; ymm0[i] = min(ymm0[i], mem[i])

; VMAXPS - Maximum Packed Single
vmaxps ymm0, ymm1, ymm2      ; ymm0[i] = max(ymm1[i], ymm2[i])

; VMINPD / VMAXPD - สำหรับ double
vminpd ymm0, ymm1, ymm2
vmaxpd ymm0, ymm1, ymm2
```

### 2.7 Shuffle Instructions

```nasm
; VSHUFPS - Shuffle Packed Single
; Format: vshufps dst, src1, src2, imm8
; imm8 เลือก 4 elements: [bits 7:6 | bits 5:4 | bits 3:2 | bits 1:0]
; แต่ละ 2 bits เลือก index จาก src1 (lower 2 elements) หรือ src2 (upper 2)
; ใน 256-bit mode: ทำ 2 copies ของ 128-bit shuffle

vshufps ymm0, ymm1, ymm2, 0x00  ; dst[0]=src1[0], dst[1]=src1[0], dst[2]=src2[0], dst[3]=src2[0]
                                  ; (repeated in upper 128-bit lane)

; VPERMPS (AVX2) - Permute Packed Single (flexible, any order)
; src2 (ymm) เป็น index vector, แต่ละ element ระบุ index 0-7
; ymm_idx = [7,6,5,4,3,2,1,0] = reverse order
vpermps ymm0, ymm_idx, ymm1     ; ymm0 = permute ymm1 by indices in ymm_idx

; VPERM2F128 - Permute 128-bit Float Lanes
; imm8: bits [3:0] = lower lane source, bits [7:4] = upper lane source
; bits [1:0] หรือ [5:4] = 0-3 เลือก: 0=src1 lower, 1=src1 upper, 2=src2 lower, 3=src2 upper
; bit 3 หรือ 7 = 1 means zero that lane
vperm2f128 ymm0, ymm1, ymm2, 0x01  ; lower lane = ymm1 upper, upper lane = ymm1 lower (swap lanes)
vperm2f128 ymm0, ymm1, ymm2, 0x20  ; lower lane = ymm2 lower, upper lane = ymm1 lower
vperm2f128 ymm0, ymm1, ymm2, 0x31  ; lower lane = ymm1 upper, upper lane = ymm2 upper
```

### 2.8 Insert/Extract 128-bit Lanes

```nasm
; VINSERTF128 - Insert 128-bit Float Lane into 256-bit
; imm8: 0 = insert into lower 128 bits, 1 = insert into upper 128 bits
vinsertf128 ymm0, ymm1, xmm2, 0   ; ymm0 = ymm1, replace lower lane with xmm2
vinsertf128 ymm0, ymm1, xmm2, 1   ; ymm0 = ymm1, replace upper lane with xmm2

; VEXTRACTF128 - Extract 128-bit Float Lane from 256-bit
; imm8: 0 = extract lower lane, 1 = extract upper lane
vextractf128 xmm0, ymm1, 0        ; xmm0 = lower 128 bits of ymm1
vextractf128 xmm0, ymm1, 1        ; xmm0 = upper 128 bits of ymm1

; Example: สลับ lanes
vextractf128 xmm0, ymm1, 0        ; xmm0 = lower
vextractf128 xmm1, ymm1, 1        ; xmm1 = upper  
vinsertf128 ymm2, ymm_tmp, xmm1, 0 ; ใส่ upper เป็น lower
vinsertf128 ymm2, ymm2, xmm0, 1   ; ใส่ lower เป็น upper (swap!)
```

### 2.9 Broadcast Instructions

```nasm
; VBROADCASTSS - Broadcast Single Float to all 8 lanes
; src อาจเป็น memory หรือ XMM register (AVX2 only for reg->reg)
vbroadcastss ymm0, [rsi]     ; ymm0 = {mem32, mem32, mem32, mem32, mem32, mem32, mem32, mem32}
vbroadcastss ymm0, xmm1      ; ymm0 = {xmm1[0], ..., xmm1[0]} (AVX2, reg->reg)

; VBROADCASTSD - Broadcast Double to all 4 lanes
vbroadcastsd ymm0, [rsi]     ; ymm0 = {mem64, mem64, mem64, mem64}
vbroadcastsd ymm0, xmm1      ; ymm0 = {xmm1[0..63], ... x4} (AVX2)

; VBROADCASTF128 - Broadcast 128-bit to both 128-bit lanes
vbroadcastf128 ymm0, [rsi]   ; ymm0 = {mem128, mem128} (upper = lower lane)

; Pattern ที่ใช้บ่อย: broadcast scalar เพื่อ SIMD multiply
movss xmm1, [scale_factor]   ; load scalar float
vbroadcastss ymm1, xmm1      ; broadcast to all 8 lanes
vmulps ymm0, ymm0, ymm1      ; multiply all 8 floats by scalar
```

---

## 3. AVX2 Integer Instructions (256-bit)

AVX2 เพิ่ม 256-bit integer operations ซึ่ง AVX (เวอร์ชันแรก) ไม่มี
ต้องตรวจสอบ CPUID: EBX bit 5 ใน leaf 7 subleaf 0

### 3.1 Packed Add/Subtract

```nasm
; VPADDB - Add Packed Bytes (32 bytes = 32 int8)
vpaddb ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] + ymm2[i], i=0..31 (byte)

; VPADDW - Add Packed Words (16 words = 16 int16)
vpaddw ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] + ymm2[i], i=0..15

; VPADDD - Add Packed Doublewords (8 dwords = 8 int32)
vpaddd ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] + ymm2[i], i=0..7

; VPADDQ - Add Packed Quadwords (4 qwords = 4 int64)
vpaddq ymm0, ymm1, ymm2      ; ymm0[i] = ymm1[i] + ymm2[i], i=0..3

; Saturating Add (ไม่ overflow, clamped to max/min)
vpaddus ymm0, ymm1, ymm2     ; unsigned saturating add bytes/words
vpaddsb ymm0, ymm1, ymm2     ; signed saturating add bytes
vpaddsw ymm0, ymm1, ymm2     ; signed saturating add words

; Subtract variants:
vpsubb ymm0, ymm1, ymm2      ; packed subtract bytes
vpsubw ymm0, ymm1, ymm2      ; packed subtract words
vpsubd ymm0, ymm1, ymm2      ; packed subtract dwords
vpsubq ymm0, ymm1, ymm2      ; packed subtract qwords

; Saturating Subtract:
vpsubusb ymm0, ymm1, ymm2    ; unsigned saturating subtract bytes
vpsubsw ymm0, ymm1, ymm2     ; signed saturating subtract words
```

### 3.2 Multiply

```nasm
; VPMULLW - Multiply Low Words (signed/unsigned, lower 16 bits of result)
vpmullw ymm0, ymm1, ymm2     ; ymm0[i] = (ymm1[i] * ymm2[i])[15:0], i=0..15

; VPMULHW - Multiply High Words (signed, upper 16 bits)
vpmulhw ymm0, ymm1, ymm2     ; ymm0[i] = (ymm1[i] * ymm2[i])[31:16]

; VPMULHUW - Multiply High Unsigned Words
vpmulhuw ymm0, ymm1, ymm2

; VPMULLD - Multiply Low Dwords (signed, lower 32 bits)
vpmulld ymm0, ymm1, ymm2     ; ymm0[i] = (ymm1[i] * ymm2[i])[31:0], i=0..7

; VPMULLQ (AVX-512 only, ไม่ใช่ AVX2!)

; VPMULUDQ - Multiply Unsigned Doublewords to Quadwords
; คูณ even-indexed dwords, ผลลัพธ์เป็น 64-bit
vpmuludq ymm0, ymm1, ymm2    ; ymm0[0] = ymm1[0] * ymm2[0] (64-bit)
                               ; ymm0[1] = ymm1[2] * ymm2[2]
                               ; ymm0[2] = ymm1[4] * ymm2[4]
                               ; ymm0[3] = ymm1[6] * ymm2[6]

; VPMADDWD - Multiply and Add Packed Integers (horizontal)
; คูณ 16-bit pairs แล้วบวกผลที่ติดกัน → 32-bit
vpmaddwd ymm0, ymm1, ymm2    ; ymm0[i] = ymm1[2i]*ymm2[2i] + ymm1[2i+1]*ymm2[2i+1]
```

### 3.3 Bitwise Operations

```nasm
; VPAND - Bitwise AND
vpand ymm0, ymm1, ymm2       ; ymm0 = ymm1 & ymm2

; VPANDN - Bitwise AND NOT (ymm1 AND NOT ymm2)
vpandn ymm0, ymm1, ymm2      ; ymm0 = ymm1 & (~ymm2)

; VPOR - Bitwise OR
vpor ymm0, ymm1, ymm2        ; ymm0 = ymm1 | ymm2

; VPXOR - Bitwise XOR
vpxor ymm0, ymm1, ymm2       ; ymm0 = ymm1 ^ ymm2

; ใช้บ่อยมาก: zero a register
vpxor ymm0, ymm0, ymm0       ; ymm0 = 0 (เร็วกว่า vmovdqu ymm0, [zeros])
```

### 3.4 Compare

```nasm
; VPCMPEQB - Compare Equal Packed Bytes (result: all 1s or all 0s per byte)
vpcmpeqb ymm0, ymm1, ymm2    ; ymm0[i] = 0xFF if ymm1[i]==ymm2[i], else 0x00

; VPCMPEQW - Compare Equal Packed Words
vpcmpeqw ymm0, ymm1, ymm2

; VPCMPEQD - Compare Equal Packed Dwords
vpcmpeqd ymm0, ymm1, ymm2

; VPCMPEQQ - Compare Equal Packed Qwords
vpcmpeqq ymm0, ymm1, ymm2

; VPCMPGTB - Compare Greater Than Packed Bytes (signed)
vpcmpgtb ymm0, ymm1, ymm2    ; ymm0[i] = 0xFF if (int8)ymm1[i] > (int8)ymm2[i]

; VPCMPGTW, VPCMPGTD, VPCMPGTQ - same for 16/32/64-bit

; สร้าง mask และ apply:
vpcmpgtd ymm_mask, ymm_a, ymm_zero  ; mask = (a > 0)
vpand ymm_result, ymm_a, ymm_mask   ; result = a where a > 0, else 0 (relu-like)
```

### 3.5 Byte Shuffle (VPSHUFB)

```nasm
; VPSHUFB - Shuffle Bytes (very flexible, 32-byte)
; ymm2 เป็น control mask: แต่ละ byte ระบุ index ของ source byte
; ถ้า high bit ของ control byte = 1 → output = 0
; สำคัญ: shuffle ทำงานภายใน 16-byte lane ไม่ข้าม lane!

; Example: reverse bytes ใน 16-byte groups
section .data
rev_mask: db 15,14,13,12,11,10,9,8,7,6,5,4,3,2,1,0  ; lower lane
          db 15,14,13,12,11,10,9,8,7,6,5,4,3,2,1,0  ; upper lane (same)

section .text
vmovdqu ymm1, [rev_mask]
vpshufb ymm0, ymm0, ymm1         ; reverse bytes in each 16-byte lane

; Example: extract specific channels from RGBA pixels
; Input: RGBA RGBA RGBA RGBA ... (4 pixels per lane, 16 bytes, repeated 2x)
; Extract R channel: indices 0,4,8,12 from lower lane, same from upper lane
section .data
r_mask: db 0,4,8,12, -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1  ; lower lane
        db 0,4,8,12, -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1  ; upper lane

section .text
vmovdqu ymm1, [r_mask]
vpshufb ymm0, ymm_rgba, ymm1     ; R values in positions 0-3 of each lane
```

### 3.6 Permute Instructions (AVX2)

```nasm
; VPERMD - Permute Packed Doublewords (cross-lane, 8 int32)
; ymm_idx: แต่ละ dword = index 0-7 ของ source dword
; ต่างจาก VPSHUFB ตรงที่ข้าม lane ได้!
vpermd ymm0, ymm_idx, ymm_src    ; ymm0[i] = ymm_src[ymm_idx[i]]

; VPERMQ - Permute Packed Quadwords (imm8 controlled)
; imm8: 4 x 2-bit fields กำหนด index 0-3
vpermq ymm0, ymm1, 0x1B          ; 0x1B = 0b00_01_10_11 = reverse qword order
                                   ; [q3,q2,q1,q0] → [q0,q1,q2,q3]

vpermq ymm0, ymm1, 0x4E          ; 0x4E = 0b01_00_11_10 = swap 128-bit lanes

; VPERM2I128 - Permute Integer 128-bit Lanes (same as VPERM2F128 but integer)
vperm2i128 ymm0, ymm1, ymm2, 0x01  ; swap lower/upper of ymm1
vperm2i128 ymm0, ymm1, ymm2, 0x20  ; lower=ymm2 lower, upper=ymm1 lower

; VINSERTI128 / VEXTRACTI128 (integer version of VINSERTF128)
vinserti128 ymm0, ymm1, xmm2, 0   ; insert xmm2 into lower lane of ymm0
vinserti128 ymm0, ymm1, xmm2, 1   ; insert xmm2 into upper lane
vextracti128 xmm0, ymm1, 0        ; extract lower lane
vextracti128 xmm0, ymm1, 1        ; extract upper lane
```

### 3.7 Variable Shift (AVX2)

```nasm
; VPSLLVD - Shift Left Logical Variable Dwords
; แต่ละ element shift ตาม count ของ shift vector
vpsllvd ymm0, ymm1, ymm2     ; ymm0[i] = ymm1[i] << ymm2[i], i=0..7

; VPSRLVD - Shift Right Logical Variable Dwords
vpsrlvd ymm0, ymm1, ymm2     ; ymm0[i] = ymm1[i] >> ymm2[i] (logical, zero fill)

; VPSRAVD - Shift Right Arithmetic Variable Dwords
vpsravd ymm0, ymm1, ymm2     ; ymm0[i] = ymm1[i] >> ymm2[i] (arithmetic, sign fill)

; Qword variants:
vpsllvq ymm0, ymm1, ymm2     ; shift qwords
vpsrlvq ymm0, ymm1, ymm2

; Example: scale different elements by powers of 2
section .data
shifts: dd 0, 1, 2, 3, 4, 5, 6, 7   ; shift amounts per lane

section .text
vmovdqu ymm2, [shifts]
vpsllvd ymm0, ymm1, ymm2     ; ymm0[i] = ymm1[i] * 2^i
```

### 3.8 Broadcast Integer (AVX2)

```nasm
; VPBROADCASTB - Broadcast Byte to all 32 bytes
vpbroadcastb ymm0, xmm1      ; ymm0 = {xmm1[0], ..., xmm1[0]} (32 copies)
vpbroadcastb ymm0, [rsi]     ; ymm0 = {mem8, ..., mem8}

; VPBROADCASTW - Broadcast Word to all 16 words
vpbroadcastw ymm0, xmm1      ; 16 copies of 16-bit value

; VPBROADCASTD - Broadcast Dword to all 8 dwords
vpbroadcastd ymm0, xmm1      ; 8 copies of 32-bit value
vpbroadcastd ymm0, [rsi]

; VPBROADCASTQ - Broadcast Qword to all 4 qwords
vpbroadcastq ymm0, xmm1      ; 4 copies of 64-bit value

; Pattern: broadcast and compare
movzx eax, byte [pattern]
vmovd xmm1, eax
vpbroadcastb ymm1, xmm1      ; fill all 32 bytes with pattern byte
vpcmpeqb ymm0, ymm_data, ymm1 ; find all matches
vpmovmskb eax, xmm0          ; get match mask (32 bits for 256-bit)
; (note: vpmovmskb ใช้ XMM ไม่ใช่ YMM สำหรับ mask extraction)
; สำหรับ 256-bit ต้องใช้ ymm แล้วแยก
vpmovmskb eax, ymm0          ; eax = 32-bit mask (AVX2 รองรับ YMM)
```

### 3.9 Gather Instructions (AVX2)

Gather ช่วยโหลด data จากหลาย addresses ที่ไม่ต่อเนื่องกัน

```nasm
; VPGATHERDPS - Gather Float32 with Dword Indices
; Format: vpgatherdps dst, [base + vmm_idx * scale], mask
; dst: destination YMM (float32 x 8)
; vmm_idx: YMM ของ int32 indices (0..7)
; mask: YMM mask (gather ทำเมื่อ high bit ของ mask[i] = 1)
;       หลัง gather: mask element เปลี่ยนเป็น 0

section .data
indices: dd 0, 4, 8, 16, 2, 6, 10, 20  ; byte offsets

section .text
vmovdqu ymm2, [indices]       ; load indices
vpcmpeqd ymm1, ymm1, ymm1    ; ymm1 = all ones (mask: gather all)
vpgatherdps ymm0, [rsi + ymm2*4], ymm1  ; gather 8 floats

; VPGATHERQPS - Gather Float32 with Qword Indices (4 floats ใน xmm)
vpgatherqps xmm0, [rsi + ymm2*4], xmm1  ; 4 floats (lower YMM)

; VPGATHERDPD - Gather Float64 with Dword Indices
vpgatherdpd ymm0, [rsi + xmm2*8], ymm1  ; 4 doubles, xmm indices

; VPGATHERQPD - Gather Float64 with Qword Indices
vpgatherqpd ymm0, [rsi + ymm2*8], ymm1  ; 4 doubles

; Integer gather variants:
vpgatherdd ymm0, [rsi + ymm2*4], ymm1   ; gather 8 int32
vpgatherdq ymm0, [rsi + xmm2*8], ymm1   ; gather 4 int64
```

### 3.10 Masked Load/Store

```nasm
; VPMASKMOVD - Masked Move Dwords
; Load: โหลดเฉพาะ elements ที่มี high bit ของ mask = 1, ที่เหลือ = 0
; Store: เขียนเฉพาะ elements ที่ high bit = 1, memory ไม่เปลี่ยนที่เหลือ

; Masked Load:
vpmaskmovd ymm0, ymm_mask, [rsi]    ; load 8 dwords, masked
                                     ; ถ้า mask[i] bit 31 = 0 → ymm0[i] = 0

; Masked Store:
vpmaskmovd [rdi], ymm_mask, ymm0    ; store 8 dwords, masked
                                     ; ถ้า mask[i] bit 31 = 0 → memory ไม่เปลี่ยน

; VPMASKMOVQ - Masked Move Qwords
vpmaskmovq ymm0, ymm_mask, [rsi]    ; load 4 qwords, masked
vpmaskmovq [rdi], ymm_mask, ymm0    ; store 4 qwords, masked

; Float equivalents:
vmaskmovps ymm0, ymm_mask, [rsi]    ; masked load float32
vmaskmovpd ymm0, ymm_mask, [rsi]    ; masked load float64
vmaskmovps [rdi], ymm_mask, ymm0    ; masked store float32
```

---

## 4. FMA (Fused Multiply-Add)

FMA3 (3-operand FMA) มาพร้อม Haswell และ AMD Piledriver
ต้องตรวจสอบ CPUID: ECX bit 12 ใน leaf 1

### 4.1 FMA132/213/231 Convention

ตัวเลข 132, 213, 231 บอกลำดับของ operands ใน multiply-add:
- 1 = dst (ตัวที่กำหนด result)
- 2 = explicit src2 (ใน syntax)  
- 3 = explicit src3 (ใน syntax)

```
VFMADD132PS dst, src2, src3:   dst = dst * src3 + src2
VFMADD213PS dst, src2, src3:   dst = src2 * dst + src3
VFMADD231PS dst, src2, src3:   dst = src2 * src3 + dst
```

### 4.2 FMA Packed Single (8 floats)

```nasm
; VFMADD132PS: dst = dst * src3 + src2
vfmadd132ps ymm0, ymm1, ymm2     ; ymm0[i] = ymm0[i]*ymm2[i] + ymm1[i]
vfmadd132ps ymm0, ymm1, [rsi]    ; ymm0[i] = ymm0[i]*mem[i] + ymm1[i]

; VFMADD213PS: dst = src2 * dst + src3
vfmadd213ps ymm0, ymm1, ymm2     ; ymm0[i] = ymm1[i]*ymm0[i] + ymm2[i]
vfmadd213ps ymm0, ymm1, [rsi]    ; ymm0[i] = ymm1[i]*ymm0[i] + mem[i]

; VFMADD231PS: dst = src2 * src3 + dst  (ใช้บ่อยที่สุดในการ accumulate)
vfmadd231ps ymm0, ymm1, ymm2     ; ymm0[i] += ymm1[i]*ymm2[i]
vfmadd231ps ymm0, ymm1, [rsi]    ; ymm0[i] += ymm1[i]*mem[i]
```

### 4.3 FMA Variants: SUB, NMADD, NMSUB

```nasm
; VFMSUB: Fused Multiply Subtract (a*b - c)
vfmsub132ps ymm0, ymm1, ymm2     ; ymm0 = ymm0*ymm2 - ymm1
vfmsub213ps ymm0, ymm1, ymm2     ; ymm0 = ymm1*ymm0 - ymm2
vfmsub231ps ymm0, ymm1, ymm2     ; ymm0 = ymm1*ymm2 - ymm0

; VFNMADD: Fused Negate Multiply Add (-(a*b) + c)
vfnmadd132ps ymm0, ymm1, ymm2    ; ymm0 = -(ymm0*ymm2) + ymm1
vfnmadd213ps ymm0, ymm1, ymm2    ; ymm0 = -(ymm1*ymm0) + ymm2
vfnmadd231ps ymm0, ymm1, ymm2    ; ymm0 = -(ymm1*ymm2) + ymm0

; VFNMSUB: Fused Negate Multiply Subtract (-(a*b) - c)
vfnmsub132ps ymm0, ymm1, ymm2    ; ymm0 = -(ymm0*ymm2) - ymm1
vfnmsub213ps ymm0, ymm1, ymm2    ; ymm0 = -(ymm1*ymm0) - ymm2
vfnmsub231ps ymm0, ymm1, ymm2    ; ymm0 = -(ymm1*ymm2) - ymm0

; Double precision variants (PD):
vfmadd231pd ymm0, ymm1, ymm2     ; 4 doubles: ymm0 += ymm1*ymm2
vfmsub231pd ymm0, ymm1, ymm2
vfnmadd231pd ymm0, ymm1, ymm2
```

### 4.4 FMA Scalar

```nasm
; FMA Scalar Single (SS):
vfmadd132ss xmm0, xmm1, xmm2    ; xmm0[0] = xmm0[0]*xmm2[0] + xmm1[0]
vfmadd213ss xmm0, xmm1, xmm2    ; xmm0[0] = xmm1[0]*xmm0[0] + xmm2[0]
vfmadd231ss xmm0, xmm1, xmm2    ; xmm0[0] += xmm1[0]*xmm2[0]
vfmadd231ss xmm0, xmm1, [rsi]   ; xmm0[0] += xmm1[0]*mem_float[rsi]

; FMA Scalar Double (SD):
vfmadd132sd xmm0, xmm1, xmm2    ; xmm0[0] = xmm0[0]*xmm2[0] + xmm1[0] (double)
vfmadd231sd xmm0, xmm1, [rsi]   ; xmm0[0] += xmm1[0]*mem_double[rsi]
```

### 4.5 ความแม่นยำของ FMA vs MUL+ADD แยกกัน

```nasm
; กรณีปัญหา: a*b + c ด้วย separate instructions
; มีการ round 2 ครั้ง: หลัง multiply และหลัง add
vmulps ymm_tmp, ymm_a, ymm_b    ; round #1 after multiply
vaddps ymm_result, ymm_tmp, ymm_c  ; round #2 after add

; FMA: round เพียงครั้งเดียว (หลัง fused operation)
; คำนวณ a*b + c ภายใน extended precision แล้วค่อย round ครั้งเดียว
vfmadd231ps ymm_result, ymm_a, ymm_b  ; ymm_result += a*b (1 round only)
; (ที่นี่ ymm_result เป็น c ก่อน)

; ข้อดี FMA:
; 1. ความแม่นยำสูงกว่า (1 rounding error แทน 2)
; 2. เร็วกว่า (1 instruction แทน 2)
; 3. ไม่ต้องใช้ register ชั่วคราว
; 4. Throughput: 2 FMA/cycle บน Haswell = 16 float ops/cycle (256-bit)
```

---

## 5. ตัวอย่าง Code จริง

### 5.1 8x Float Addition

```nasm
; add_8floats: บวก array of floats 8 ตัวต่อ iteration
; Input: rsi = ptr to array A, rdi = ptr to array B, rcx = count (multiple of 8)
;        rdx = ptr to result array
; Uses: ymm0-ymm2

global add_8floats
add_8floats:
    push rbp
    mov rbp, rsp
    
    ; ตรวจสอบว่า count เป็น multiple of 8
    test rcx, 7
    jnz .handle_remainder       ; ถ้าไม่ใช่ ต้องจัดการส่วนที่เหลือ
    
.loop:
    test rcx, rcx
    jz .done
    
    vmovups ymm0, [rsi]         ; load 8 floats from A (unaligned)
    vaddps  ymm0, ymm0, [rdi]   ; add 8 floats from B
    vmovups [rdx], ymm0         ; store result
    
    add rsi, 32                 ; advance by 32 bytes (8 floats * 4 bytes)
    add rdi, 32
    add rdx, 32
    sub rcx, 8                  ; decrement count by 8
    jmp .loop
    
.done:
    vzeroupper                  ; *** สำคัญ: clear upper YMM bits ***
    pop rbp
    ret

.handle_remainder:
    ; จัดการส่วนที่เหลือด้วย scalar หรือ masked operations
    ; (simplified: assume multiple of 8 in this example)
    jmp .done
```

### 5.2 256-bit Dot Product

```nasm
; dot_product_avx: compute dot product of two float arrays
; Input: rsi = array A, rdi = array B, rcx = count (multiple of 8)
; Output: xmm0 = scalar dot product result

global dot_product_avx
dot_product_avx:
    push rbp
    mov rbp, rsp
    
    vpxor ymm0, ymm0, ymm0      ; ymm0 = accumulator = 0.0
    
.loop:
    cmp rcx, 8
    jl .hsum                    ; less than 8 remaining
    
    vmovups ymm1, [rsi]         ; load 8 floats from A
    vfmadd231ps ymm0, ymm1, [rdi] ; acc += A[i] * B[i] (FMA)
    
    add rsi, 32
    add rdi, 32
    sub rcx, 8
    jmp .loop
    
.hsum:
    ; Horizontal sum: reduce 8 floats to 1
    ; ymm0 = [a7,a6,a5,a4,a3,a2,a1,a0]
    vextractf128 xmm1, ymm0, 1  ; xmm1 = [a7,a6,a5,a4]
    vaddps xmm0, xmm0, xmm1    ; xmm0 = [a7+a3, a6+a2, a5+a1, a4+a0]
    vshufps xmm1, xmm0, xmm0, 0xB1 ; xmm1 = [a6+a2, a7+a3, a4+a0, a5+a1]
    vaddps xmm0, xmm0, xmm1    ; xmm0 = [sum76_32, sum76_32, sum54_10, sum54_10]
    vshufps xmm1, xmm0, xmm0, 0x02 ; move sum54_10 to position 0
    vaddss xmm0, xmm0, xmm1    ; xmm0[0] = total sum
    
    vzeroupper
    pop rbp
    ret
```

### 5.3 Matrix 8x8 Multiply ด้วย AVX2

```nasm
; matmul_8x8_avx: multiply two 8x8 float matrices
; C = A * B
; Input: rdi = C (output), rsi = A (input), rdx = B (input)
; All matrices: 8x8 floats = 256 bytes each, 32-byte aligned

global matmul_8x8_avx
matmul_8x8_avx:
    push rbp
    mov rbp, rsp
    sub rsp, 64                 ; local space
    push rbx r12 r13 r14 r15   ; callee-saved
    
    ; Strategy: compute C[row] = sum over k of A[row][k] * B[k]
    ; For each row i of A: load A[i][0..7] as scalar broadcasts
    ; Multiply with each row of B and accumulate
    
    xor r12, r12                ; row index i
    
.row_loop:
    cmp r12, 8
    jge .done
    
    ; Compute C[i][0..7] = sum_k A[i][k] * B[k][0..7]
    vpxor ymm8, ymm8, ymm8     ; accumulator row = 0
    
    xor r13, r13                ; k = 0
    
.k_loop:
    cmp r13, 8
    jge .store_row
    
    ; Load A[i][k] as scalar and broadcast
    mov eax, r12d
    imul eax, 32                ; row * 8 floats * 4 bytes
    mov ebx, r13d
    imul ebx, 4                 ; col * 4 bytes
    add eax, ebx
    
    vbroadcastss ymm0, [rsi + rax]  ; ymm0 = A[i][k] broadcast
    
    ; Load B[k][0..7]
    mov eax, r13d
    imul eax, 32                ; row k of B
    vmovaps ymm1, [rdx + rax]  ; B[k][0..7]
    
    ; Accumulate: C[i] += A[i][k] * B[k]
    vfmadd231ps ymm8, ymm0, ymm1
    
    inc r13
    jmp .k_loop
    
.store_row:
    mov eax, r12d
    imul eax, 32
    vmovaps [rdi + rax], ymm8  ; store C[i][0..7]
    
    inc r12
    jmp .row_loop
    
.done:
    vzeroupper
    pop r15 r14 r13 r12 rbx
    add rsp, 64
    pop rbp
    ret
```

### 5.4 Image RGB Processing (8 Pixels at Once)

```nasm
; process_rgb_8pixels: apply brightness adjustment to 8 RGB pixels
; Input: rsi = input buffer (8 RGB pixels = 24 bytes, packed)
;        rdi = output buffer
;        xmm0 = brightness multiplier (broadcast to all)
; 
; RGB format: R0,G0,B0,R1,G1,B1,...  (uint8_t packed)
; ต้องแยก channels, process แล้ว pack กลับ

global process_rgb_8pixels
process_rgb_8pixels:
    push rbp
    mov rbp, rsp
    
    ; กลยุทธ์: 
    ; 1. Load 24 bytes (8 pixels * 3 channels)
    ; 2. Unpack เป็น float
    ; 3. Multiply by brightness
    ; 4. Pack กลับเป็น uint8

    ; Load 24 bytes of RGB data (แบบ unaligned)
    vmovdqu xmm1, [rsi]         ; load first 16 bytes (pixels 0-5 plus partial 6)
    vmovdqu xmm2, [rsi+8]       ; load bytes 8-23 (pixels 2-7 plus partial)
    
    ; สร้าง float32 จาก uint8 (simplified: ทำ R channel)
    ; Extract R channel (bytes 0,3,6,9,12,15,18,21)
    ; ใช้ VPSHUFB เพื่อ gather R bytes
    
section .data
rgb_r_mask:
    ; Gather R bytes: positions 0,3,6,9 ใน lower 16 bytes
    db 0,3,6,9, -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1
    db -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1
    
rgb_g_mask:
    db 1,4,7,10, -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1
    db -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1, -1,-1,-1,-1

section .text
    ; Convert first 4 R pixels to float
    vmovdqu ymm3, [rgb_r_mask]
    vpshufb xmm4, xmm1, xmm3    ; R values: R0,R1,R2,R3 in bytes 0-3
    
    ; Zero-extend bytes to dwords
    vpmovzxbd xmm4, xmm4         ; extend 4 bytes to 4 dwords
    vcvtdq2ps xmm4, xmm4         ; convert int32 to float32
    
    ; Broadcast brightness and multiply
    vbroadcastss xmm5, xmm0      ; broadcast brightness scalar
    vmulps xmm4, xmm4, xmm5     ; multiply R values
    
    ; Clamp to 0-255
    vminps xmm4, xmm4, [f255]   ; clamp max
    vmaxps xmm4, xmm4, [fzero]  ; clamp min
    
    ; Convert back to int
    vcvtps2dq xmm4, xmm4        ; float to int32
    vpackusdw xmm4, xmm4, xmm4  ; int32 to int16 (unsigned saturate)
    vpackuswb xmm4, xmm4, xmm4  ; int16 to uint8 (unsigned saturate)
    
    ; Store R bytes back (simplified)
    vmovd eax, xmm4
    ; ... (ต้องทำ channel interleaving กลับ)
    
    vzeroupper
    pop rbp
    ret

section .data
align 16
f255: dd 255.0, 255.0, 255.0, 255.0
fzero: dd 0.0, 0.0, 0.0, 0.0
```

### 5.5 Bloom Filter Lookup ด้วย Gather

```nasm
; bloom_filter_lookup: ตรวจสอบ 8 items ใน bloom filter พร้อมกัน
; Input: rsi = bloom filter bitset (byte array)
;        rdi = array of 8 hash values (uint32_t, ระบุ bit positions)
;        rcx = filter size in bytes
; Output: eax = bitmask (bit i = 1 ถ้า item i อาจอยู่ใน filter)

global bloom_filter_lookup
bloom_filter_lookup:
    push rbp
    mov rbp, rsp
    
    ; Load 8 hash values (bit positions)
    vmovdqu ymm0, [rdi]          ; ymm0 = [h7,h6,h5,h4,h3,h2,h1,h0] (int32)
    
    ; Calculate byte index: byte_idx = hash / 8
    vpsrld ymm1, ymm0, 3         ; ymm1 = hash >> 3 (byte index)
    
    ; Calculate bit index within byte: bit_idx = hash & 7
    vpand ymm2, ymm0, [mask_7]   ; ymm2 = hash & 7
    
    ; Gather bytes from bloom filter
    vpxor ymm3, ymm3, ymm3       ; zero out gather dest
    vpcmpeqd ymm4, ymm4, ymm4   ; all-ones mask
    vpgatherdd ymm3, [rsi + ymm1], ymm4  ; gather bytes at byte_idx
    
    ; Create bit masks: 1 << bit_idx
    vpcmpeqd ymm5, ymm5, ymm5   ; ymm5 = all ones
    vpsrld ymm5, ymm5, 31       ; ymm5 = all 1s (as dwords: value 1)
    vpsllvd ymm5, ymm5, ymm2    ; ymm5 = 1 << bit_idx for each element
    
    ; Test if bit is set: (byte & bit_mask) != 0
    vpand ymm6, ymm3, ymm5      ; AND byte with bit mask
    vpcmpeqd ymm6, ymm6, ymm5   ; compare: is bit set?
    
    ; Extract result as 8-bit mask
    vmovmskps eax, ymm6         ; eax = 8-bit mask (1 bit per element)
    
    vzeroupper
    pop rbp
    ret

section .data
align 32
mask_7: dd 7, 7, 7, 7, 7, 7, 7, 7
```

### 5.6 AVX2 Memcpy (256-bit version)

```nasm
; avx2_memcpy: copy memory using 256-bit AVX2 operations
; Input: rdi = dst, rsi = src, rdx = size (bytes)
; Fast path: aligned copy using YMM registers

global avx2_memcpy
avx2_memcpy:
    push rbp
    mov rbp, rsp
    
    mov rcx, rdx                ; size
    
    ; Handle less than 32 bytes first
    cmp rcx, 32
    jl .small_copy
    
    ; Align destination to 32 bytes
    mov rax, rdi
    and rax, 31                 ; check alignment
    jz .aligned_loop
    
    ; Copy unaligned head (up to 32 bytes)
    mov r8, 32
    sub r8, rax                 ; bytes to alignment
    cmp r8, rcx
    cmova r8, rcx               ; min(r8, rcx)
    
.head_loop:
    test r8, r8
    jz .aligned_loop
    mov al, [rsi]
    mov [rdi], al
    inc rsi
    inc rdi
    dec rcx
    dec r8
    jmp .head_loop
    
.aligned_loop:
    ; Main loop: copy 128 bytes (4 x 32-byte) per iteration
    cmp rcx, 128
    jl .tail_32
    
    vmovdqu ymm0, [rsi]         ; load 32 bytes (unaligned src ok)
    vmovdqu ymm1, [rsi+32]
    vmovdqu ymm2, [rsi+64]
    vmovdqu ymm3, [rsi+96]
    vmovdqa [rdi], ymm0         ; store 32 bytes (aligned dst)
    vmovdqa [rdi+32], ymm1
    vmovdqa [rdi+64], ymm2
    vmovdqa [rdi+96], ymm3
    
    add rsi, 128
    add rdi, 128
    sub rcx, 128
    jmp .aligned_loop
    
.tail_32:
    ; Handle remaining 32-byte blocks
    cmp rcx, 32
    jl .small_copy
    vmovdqu ymm0, [rsi]
    vmovdqa [rdi], ymm0
    add rsi, 32
    add rdi, 32
    sub rcx, 32
    jmp .tail_32
    
.small_copy:
    ; Copy remaining bytes one by one
    test rcx, rcx
    jz .done
    mov al, [rsi]
    mov [rdi], al
    inc rsi
    inc rdi
    dec rcx
    jmp .small_copy
    
.done:
    vzeroupper
    pop rbp
    ret
```

### 5.7 VZEROUPPER ที่ถูกต้อง: ตัวอย่างสมบูรณ์

```nasm
; ตัวอย่าง: AVX function ที่มี VZEROUPPER อย่างถูกต้อง

; ===================================================
; WRONG: ลืม VZEROUPPER → transition penalty
; ===================================================
bad_avx_function:
    push rbp
    mov rbp, rsp
    vmovaps ymm0, [rsi]
    vmulps ymm0, ymm0, ymm1
    vmovaps [rdi], ymm0
    pop rbp
    ret                          ; *** WRONG: upper YMM bits dirty! ***

; ===================================================
; CORRECT: VZEROUPPER ก่อน return
; ===================================================
good_avx_function:
    push rbp
    mov rbp, rsp
    vmovaps ymm0, [rsi]
    vmulps ymm0, ymm0, ymm1
    vmovaps [rdi], ymm0
    vzeroupper                   ; *** CORRECT: zero upper bits ***
    pop rbp
    ret

; ===================================================
; CORRECT: VZEROUPPER ก่อนเรียก legacy function
; ===================================================
call_legacy_function:
    push rbp
    mov rbp, rsp
    
    ; ... use AVX ...
    vmovaps ymm0, [data1]
    vmulps ymm0, ymm0, ymm1
    
    vzeroupper                   ; *** CORRECT: before calling SSE code ***
    call some_sse_library        ; safe: no penalty
    
    ; ถ้าต้องการใช้ AVX อีก ก็ทำได้ทันที (upper bits = 0 แล้ว)
    vmovaps ymm0, [data2]
    vmulps ymm0, ymm0, ymm2
    
    vzeroupper                   ; *** CORRECT: before return ***
    pop rbp
    ret

; ===================================================
; Pattern: Check if in "dirty" state (for debug)
; ===================================================
; ใน production code ไม่ต้องตรวจสอบ แค่เรียก VZEROUPPER เสมอ
; VZEROUPPER ใช้เวลาเพียงไม่กี่ cycles เมื่อ upper bits = 0 อยู่แล้ว
```

---

## 6. Performance Comparison: SSE vs AVX vs AVX2

### 6.1 Theoretical Throughput

| Operation           | SSE (128-bit)   | AVX (256-bit)    | AVX2 (256-bit int) |
|---------------------|-----------------|------------------|---------------------|
| Float add/mul       | 4 float/cycle   | 8 float/cycle    | 8 float/cycle       |
| Double add/mul      | 2 double/cycle  | 4 double/cycle   | 4 double/cycle      |
| FMA (w/ FMA3)       | N/A             | 16 float/cycle   | 16 float/cycle      |
| Int32 add           | 4 int32/cycle   | N/A (AVX only float) | 8 int32/cycle  |
| Int8 compare        | 16 byte/cycle   | N/A              | 32 byte/cycle       |

*ค่าบน Haswell (dual issue FMA units)*

### 6.2 Latency ตัวอย่าง (Haswell)

| Instruction     | Latency | Throughput (recip) |
|-----------------|---------|--------------------|
| VADDPS ymm      | 3 cycles | 1 cycle           |
| VMULPS ymm      | 5 cycles | 0.5 cycles        |
| VDIVPS ymm      | 21 cycles | 14 cycles        |
| VSQRTPS ymm     | 14 cycles | 14 cycles        |
| VRSQRTPS ymm    | 5 cycles  | 1 cycle          |
| VFMADD231PS ymm | 5 cycles  | 0.5 cycles       |
| VPGATHERDD ymm  | varies    | varies (slow!)   |

### 6.3 Benchmark: Scalar vs SSE vs AVX vs AVX2 FMA

```nasm
; ตัวอย่าง: Sum of products ของ array 1024 floats

; === Scalar (C equivalent) ===
; float sum = 0;
; for (int i = 0; i < 1024; i++) sum += a[i] * b[i];
; ประมาณ: 1024 cycles (1 MUL + 1 ADD per iter)

; === SSE (128-bit, no FMA) ===
; ประมาณ: 1024/4 * (1 MULPS + 1 ADDPS) = 256 * 2 = 512 cycles
; + horizontal sum

; === AVX (256-bit, no FMA) ===
; ประมาณ: 1024/8 * (1 VMULPS + 1 VADDPS) = 128 * 2 = 256 cycles
; + horizontal sum

; === AVX2 + FMA (256-bit) ===
; ประมาณ: 1024/8 * 1 VFMADD = 128 cycles
; + horizontal sum
; ~8x speedup จาก scalar!

; === ตัวอย่าง loop จริง ===
; float dot_product_fma(float* a, float* b, int n)

dot_product_fma_bench:
    push rbp
    mov rbp, rsp
    
    ; Unroll 4 times: process 32 floats per iteration
    vpxor ymm0, ymm0, ymm0    ; acc0 = 0
    vpxor ymm1, ymm1, ymm1    ; acc1 = 0
    vpxor ymm2, ymm2, ymm2    ; acc2 = 0
    vpxor ymm3, ymm3, ymm3    ; acc3 = 0
    
    xor eax, eax
.loop4x:
    cmp eax, ecx              ; eax = i, ecx = n
    jge .hsum
    
    ; Process 4 x 8 = 32 floats at once
    vfmadd231ps ymm0, ymm4, [rsi + rax*4]     ; acc0 += a[i+0..7] * b[i+0..7]
    vmovups ymm4, [rdi + rax*4]
    vfmadd231ps ymm1, ymm4, [rsi + rax*4 + 32] ; acc1 += a[i+8..15] * b[i+8..15]
    vmovups ymm4, [rdi + rax*4 + 64]
    vfmadd231ps ymm2, ymm4, [rsi + rax*4 + 64]
    vmovups ymm4, [rdi + rax*4 + 96]
    vfmadd231ps ymm3, ymm4, [rsi + rax*4 + 96]
    
    add eax, 32               ; advance by 32 elements
    jmp .loop4x
    
.hsum:
    vaddps ymm0, ymm0, ymm1   ; combine accumulators
    vaddps ymm2, ymm2, ymm3
    vaddps ymm0, ymm0, ymm2
    ; horizontal sum ของ ymm0 (8 floats → 1 scalar)
    vextractf128 xmm1, ymm0, 1
    vaddps xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0  ; xmm0[0] = total sum
    
    vzeroupper
    pop rbp
    ret
```

### 6.4 Memory Bandwidth Considerations

```nasm
; AVX2 แม้ประมวลผลได้มากขึ้น แต่ต้องการ bandwidth มากขึ้นด้วย
; Modern CPU: ~50 GB/s memory bandwidth
; AVX2 loop ที่ 1 GHz: 32 bytes/cycle * 1e9 = 32 GB/s (ต่อ core)

; Optimization strategies:
; 1. Reuse data (maximize cache utilization)
; 2. Use streaming stores (VMOVNTPS) สำหรับ write-only data
; 3. Prefetch (PREFETCHT0/T1/T2/NTA) สำหรับ sequential access

; VMOVNTPS: Non-temporal store (bypass cache, write directly to memory)
vmovntps [rdi], ymm0          ; 256-bit non-temporal store
vmovntpd [rdi], ymm0          ; double version
vmovntdq [rdi], ymm0          ; integer version

; Memory fence หลัง non-temporal stores:
sfence                        ; ensure non-temporal stores are complete
```

---

## 7. CPUID Checks สำหรับ AVX/AVX2/FMA

```nasm
; ตรวจสอบก่อนใช้ AVX/AVX2/FMA instructions

check_avx_support:
    push rbp
    mov rbp, rsp
    push rbx r12 r13
    
    ; Check OSXSAVE (OS ต้อง enable YMM registers ผ่าน CR4.OSXSAVE)
    mov eax, 1
    cpuid
    
    ; Check AVX support: ECX bit 28
    test ecx, (1 << 28)
    jz .no_avx
    
    ; Check OSXSAVE: ECX bit 27  
    test ecx, (1 << 27)
    jz .no_avx
    
    ; Check OS actually enables YMM (via XGETBV)
    xor ecx, ecx                ; XCR0
    xgetbv                      ; result in EDX:EAX
    and eax, 0x6                ; bits 1,2 = SSE state, YMM state
    cmp eax, 0x6
    jne .no_avx
    
    ; Check FMA3: ECX bit 12 (leaf 1)
    mov eax, 1
    cpuid
    test ecx, (1 << 12)
    setnz r12b                  ; r12b = 1 if FMA supported
    
    ; Check AVX2: EBX bit 5 (leaf 7, subleaf 0)
    mov eax, 7
    xor ecx, ecx                ; subleaf 0
    cpuid
    test ebx, (1 << 5)
    setnz r13b                  ; r13b = 1 if AVX2 supported
    
    ; Return: eax = flags: bit0=AVX, bit1=AVX2, bit2=FMA
    mov eax, 1                  ; AVX supported
    test r12b, r12b
    jz .check_avx2
    or eax, 4                   ; FMA supported
.check_avx2:
    test r13b, r13b
    jz .done
    or eax, 2                   ; AVX2 supported
    jmp .done
    
.no_avx:
    xor eax, eax                ; no AVX
    
.done:
    pop r13 r12 rbx
    pop rbp
    ret
```

---

## 8. Common Patterns และ Best Practices

### 8.1 Lane Crossing vs In-Lane Operations

```nasm
; *** สำคัญมาก: AVX ส่วนใหญ่ทำงาน "in-lane" (128-bit boundary) ***

; IN-LANE: shuffle ภายใน 128-bit lane เท่านั้น
vshufps ymm0, ymm1, ymm2, 0xB1  ; shuffle ภายใน lower 128, และภายใน upper 128 แยกกัน

; CROSS-LANE: ข้าม 128-bit boundary ได้ (มักช้ากว่า)
vperm2f128 ymm0, ymm1, ymm2, 0x01  ; สลับ 128-bit lanes
vpermd ymm0, ymm_idx, ymm_src       ; arbitrary permute ข้าม lane ได้

; AVX2 horizontal operations ยังคง in-lane:
vphaddd ymm0, ymm1, ymm2  ; horizontal add ทำใน lower lane และ upper lane แยกกัน
```

### 8.2 Alignment Best Practices

```nasm
section .data
align 32                         ; *** 32-byte alignment สำหรับ 256-bit data ***
my_float_array: times 8 dd 0.0  ; 8 floats = 32 bytes

; ใน C/C++: __attribute__((aligned(32))) หรือ alignas(32)

; Runtime alignment check:
test rsi, 31                     ; check 32-byte alignment
jnz .unaligned                   ; handle unaligned case
; aligned path: use VMOVAPS (faster)
vmovaps ymm0, [rsi]
jmp .continue
.unaligned:
; unaligned path: use VMOVUPS (slightly slower on old CPUs)
vmovups ymm0, [rsi]
.continue:
```

### 8.3 Mixing AVX กับ x87/SSE

```nasm
; *** อย่า mix SSE และ AVX โดยไม่มี VZEROUPPER ***

; Pattern 1: Pure AVX code block
avx_block:
    vmovaps ymm0, [data1]      ; VEX-encoded: zero upper on XMM use
    vaddps ymm0, ymm0, ymm1
    vmovaps [result], ymm0
    vzeroupper                  ; *** ก่อนออกจาก AVX block ***

; Pattern 2: Inline AVX in mixed code
mixed_code:
    ; SSE code here
    movaps xmm0, [data1]        ; regular SSE
    addps xmm0, xmm1
    
    vzeroupper                  ; *** ก่อน switch ไป AVX ใน old code (เผื่อ dirty) ***
    vmovaps ymm0, [data2]       ; AVX
    vaddps ymm0, ymm0, ymm1
    vzeroupper                  ; *** ก่อน switch กลับ ***
    
    movaps xmm0, [data3]        ; back to SSE: safe
```

### 8.4 Register Allocation Tips

```nasm
; AVX มี 16 YMM registers (x86-64)
; ใช้ให้คุ้ม: minimize spills

; Pattern: compute a*b + c*d + e*f ด้วย 4 registers
vmovaps ymm0, [a]               ; ymm0 = a
vfmadd231ps ymm4, ymm0, [b]    ; ymm4 += a*b  (ymm4 = acc)
vmovaps ymm0, [c]               ; reuse ymm0
vfmadd231ps ymm4, ymm0, [d]    ; ymm4 += c*d
vmovaps ymm0, [e]               ; reuse ymm0
vfmadd231ps ymm4, ymm0, [f]    ; ymm4 += e*f
; Result ใน ymm4 ใช้แค่ 2 registers

; Pattern: pipelining multiple FMA (hide latency=5)
; Haswell: 2 FMA units, latency 5, throughput 0.5
; ต้อง unroll เพื่อ fill pipeline
vfmadd231ps ymm0, ymm8, [a+0]   ; iteration 1
vfmadd231ps ymm1, ymm9, [a+32]  ; iteration 2 (parallel)
vfmadd231ps ymm2, ymm10, [a+64] ; iteration 3 (parallel)
vfmadd231ps ymm3, ymm11, [a+96] ; iteration 4 (parallel)
; ... (multiple independent accumulators = better IPC)
```

---

## 9. ตัวอย่างเพิ่มเติม: Neural Network Inference Kernel

```nasm
; nn_layer_avx2: compute y = ReLU(W*x + b)
; Input: rdi = output y (float*, n elements)
;        rsi = weight matrix W (float*, n*m row-major)
;        rdx = input x (float*, m elements)
;        rcx = b bias (float*, n elements)
;        r8d = n (output size, multiple of 8)
;        r9d = m (input size, multiple of 8)

global nn_layer_avx2
nn_layer_avx2:
    push rbp
    mov rbp, rsp
    push r12 r13 r14 r15
    sub rsp, 32
    
    xor r12, r12                ; i = 0 (output row)
    
.output_loop:
    cmp r12d, r8d               ; i >= n?
    jge .done
    
    ; Compute dot product: W[i][0..m-1] . x[0..m-1]
    vpxor ymm0, ymm0, ymm0    ; acc = 0
    
    xor r13, r13                ; j = 0
    lea r14, [rsi + r12*4]      ; W[i] (assuming m floats per row = m*4 bytes)
    ; Note: r14 should be rsi + r12 * (m * 4), simplifying here
    
.inner_loop:
    cmp r13d, r9d               ; j >= m?
    jge .add_bias
    
    vmovups ymm1, [r14 + r13*4] ; W[i][j..j+7]
    vfmadd231ps ymm0, ymm1, [rdx + r13*4] ; acc += W[i][j..j+7] * x[j..j+7]
    
    add r13d, 8
    jmp .inner_loop
    
.add_bias:
    ; Horizontal sum of ymm0 (8 floats → 1 float)
    vextractf128 xmm1, ymm0, 1
    vaddps xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0    ; xmm0[0] = dot product
    
    ; Add bias b[i]
    vaddss xmm0, xmm0, [rcx + r12*4]
    
    ; ReLU: max(0, x)
    vxorps xmm1, xmm1, xmm1    ; xmm1 = 0.0
    vmaxss xmm0, xmm0, xmm1    ; xmm0 = max(0, xmm0)
    
    ; Store y[i]
    vmovss [rdi + r12*4], xmm0
    
    inc r12d
    jmp .output_loop
    
.done:
    vzeroupper
    add rsp, 32
    pop r15 r14 r13 r12
    pop rbp
    ret
```

---

## 10. Horizontal Operations และ Reduction

```nasm
; =====================================================
; Horizontal Sum ของ 256-bit Float (8 floats → 1)
; =====================================================
hsum_avx_float8:
    ; Input: ymm0 = [f7,f6,f5,f4,f3,f2,f1,f0]
    
    ; Step 1: fold upper lane into lower
    vextractf128 xmm1, ymm0, 1  ; xmm1 = [f7,f6,f5,f4]
    vaddps xmm0, xmm0, xmm1    ; xmm0 = [f7+f3, f6+f2, f5+f1, f4+f0]
    
    ; Step 2: horizontal add within xmm
    vhaddps xmm0, xmm0, xmm0   ; xmm0 = [f7+f3+f6+f2, ..., f5+f1+f4+f0, ...]
    vhaddps xmm0, xmm0, xmm0   ; xmm0[0] = total sum
    ; Result in xmm0[0]
    ret

; =====================================================
; Horizontal Max ของ 256-bit Float (8 floats → 1)
; =====================================================
hmax_avx_float8:
    ; Input: ymm0 = [f7,f6,f5,f4,f3,f2,f1,f0]
    
    vextractf128 xmm1, ymm0, 1  ; xmm1 = upper lane
    vmaxps xmm0, xmm0, xmm1    ; xmm0 = max of corresponding pairs
    
    vshufps xmm1, xmm0, xmm0, 0xB1  ; rotate pairs
    vmaxps xmm0, xmm0, xmm1
    
    vshufps xmm1, xmm0, xmm0, 0x02
    vmaxss xmm0, xmm0, xmm1    ; xmm0[0] = global max
    ret

; =====================================================
; Prefix Sum (scan) ด้วย AVX2
; =====================================================
; Input: rsi = array float, rcx = count
; Output: [rdi] = prefix sums
prefix_sum_avx2:
    push rbp
    mov rbp, rsp
    
    vpxor ymm_carry, ymm_carry, ymm_carry  ; running sum = 0
    
.loop:
    cmp rcx, 8
    jl .tail
    
    vmovups ymm0, [rsi]         ; load 8 floats
    
    ; Compute prefix sum within ymm0:
    ; [a0, a1, a2, a3, a4, a5, a6, a7] → [a0, a0+a1, ..., sum(a0..a7)]
    
    ; shift left by 1 float and add
    vperm2f128 ymm1, ymm_zero, ymm0, 0x20   ; ymm1 = [0, 0, 0, 0, a0, a1, a2, a3]
    ; hmm, this needs VPERMPS with proper indices
    ; simplified approach using vshufps:
    vpxor ymm1, ymm1, ymm1
    vshufps ymm1, ymm1, ymm0, 0x90  ; complex shuffle
    vaddps ymm0, ymm0, ymm1
    
    ; Add carry from previous block
    vbroadcastss ymm_tmp, xmm_carry_scalar
    vaddps ymm0, ymm0, ymm_tmp
    
    vmovups [rdi], ymm0
    
    ; Update carry (last element)
    vextractf128 xmm_carry_scalar, ymm0, 1
    vshufps xmm_carry_scalar, xmm_carry_scalar, xmm_carry_scalar, 0xFF
    
    add rsi, 32
    add rdi, 32
    sub rcx, 8
    jmp .loop
    
.tail:
    ; Handle remaining elements
    vzeroupper
    pop rbp
    ret
```

---

## 11. Debugging AVX Code

```nasm
; เทคนิค debugging สำหรับ AVX

; 1. ตรวจสอบ alignment ก่อน VMOVAPS
check_alignment:
    test rsi, 31                ; 32-byte alignment check
    jnz .misaligned             ; จะเกิด General Protection Fault ถ้าไม่ align!
    vmovaps ymm0, [rsi]         ; safe
    jmp .ok
.misaligned:
    ; log error หรือ use VMOVUPS
    vmovups ymm0, [rsi]         ; fallback
.ok:

; 2. Print YMM register (debug helper)
; ใช้ stack เพื่อ dump:
debug_print_ymm0:
    sub rsp, 32
    vmovdqu [rsp], ymm0
    ; อ่าน [rsp+0], [rsp+4], ..., [rsp+28] เป็น 8 floats
    add rsp, 32
    ret

; 3. ตรวจสอบ MXCSR สำหรับ exceptions
check_mxcsr:
    stmxcsr [mxcsr_val]
    mov eax, [mxcsr_val]
    test eax, 0x3F              ; check exception bits 0-5
    jnz .has_exception          ; มี floating point exception!
    ret
.has_exception:
    ; bit 0: Invalid Operation (#I)
    ; bit 1: Denormal (#D)
    ; bit 2: Divide by Zero (#Z)
    ; bit 3: Overflow (#O)
    ; bit 4: Underflow (#U)
    ; bit 5: Precision (#P)
    ret

section .bss
mxcsr_val: resd 1
```

---

## 12. Cross-Platform Notes

### x86-64 (Intel/AMD)

- AVX: Intel Sandy Bridge (2011), AMD Bulldozer (2011)
- AVX2: Intel Haswell (2013), AMD Excavator (2015)
- FMA3: Intel Haswell, AMD Piledriver (2012)
- FMA4: AMD only (Bulldozer/Piledriver), deprecated

### ARM NEON vs x86 AVX

ARM NEON เป็นเทคโนโลยีคล้ายกันบน ARM:
```gas
// ARM: NEON (128-bit, 4x float32)
// ไม่มี 256-bit equivalent ใน standard NASM; ต้องใช้ SVE สำหรับ wider SIMD

fadd v0.4s, v1.4s, v2.4s    // ARM NEON: v0 = v1 + v2 (4 floats)
fmla v0.4s, v1.4s, v2.4s    // ARM NASM: v0 += v1*v2 (FMA)

// ARM SVE (Scalable Vector Extension): variable width
fadd z0.s, z1.s, z2.s       // SVE: add, width depends on hardware
```

### GAS Syntax สำหรับ AVX (AT&T)

```gas
# GAS (GNU Assembler) syntax: operands reversed!
# Source before destination
vmovaps (%rsi), %ymm0          # load: dst=ymm0, src=[rsi]
vaddps %ymm2, %ymm1, %ymm0    # ymm0 = ymm1 + ymm2 (AT&T: src1, src2, dst)
vmulps %ymm1, %ymm0, %ymm0    # ymm0 *= ymm1
vfmadd231ps %ymm1, %ymm2, %ymm0  # ymm0 += ymm1*ymm2

# Broadcast
vbroadcastss (%rsi), %ymm0     # broadcast float from memory

# Immediate
vshufps $0xB1, %ymm1, %ymm1, %ymm0  # shuffle with imm8

# VZEROUPPER
vzeroupper                      # no operands (same in both syntaxes)
```

---

## 13. สรุปและ Reference

### Instruction Summary Table

| Category          | Instructions                                        | Width   |
|-------------------|-----------------------------------------------------|---------|
| Float Move        | VMOVAPS/D, VMOVUPS/D                                | 256-bit |
| Float Arith       | VADDPS/D, VSUBPS/D, VMULPS/D, VDIVPS/D             | 256-bit |
| Float FMA         | VFMADD132/213/231PS/PD                             | 256-bit |
| Float Approx      | VRSQRTPS, VRCPPS                                   | 256-bit |
| Float Compare     | VCMPPS/PD (imm8)                                   | 256-bit |
| Float Min/Max     | VMINPS/D, VMAXPS/D                                 | 256-bit |
| Float Shuffle     | VSHUFPS, VPERMPS, VPERM2F128                       | 256-bit |
| Float Lane        | VINSERTF128, VEXTRACTF128                          | 256-bit |
| Float Broadcast   | VBROADCASTSS/SD/F128                               | 256-bit |
| Int Add/Sub       | VPADDB/W/D/Q, VPSUBB/W/D/Q (AVX2)                 | 256-bit |
| Int Mul           | VPMULLW/D, VPMULUDQ (AVX2)                        | 256-bit |
| Int Bitwise       | VPAND, VPOR, VPXOR, VPANDN (AVX2)                 | 256-bit |
| Int Compare       | VPCMPEQB/W/D/Q, VPCMPGTB/W/D (AVX2)              | 256-bit |
| Int Shuffle       | VPSHUFB, VPERMD, VPERMQ (AVX2)                    | 256-bit |
| Int Shift         | VPSLLVD/Q, VPSRLVD/Q, VPSRAVD (AVX2)             | 256-bit |
| Int Broadcast     | VPBROADCASTB/W/D/Q (AVX2)                         | 256-bit |
| Gather            | VPGATHERDPS, VPGATHERQPS, etc. (AVX2)             | 256-bit |
| Masked Move       | VMASKMOVPS/PD, VPMASKMOVD/Q (AVX2)               | 256-bit |
| State Mgmt        | VZEROUPPER, VZEROALL                               | N/A     |

### Quick Reference: เมื่อใดใช้อะไร

1. **ต้องการ float arithmetic ธรรมดา**: `VADDPS/VMULPS/VDIVPS`
2. **ต้องการ a*b + c**: `VFMADD231PS` (accumulate form)
3. **ต้องการ a*b + c แบบ scalar**: `VFMADD231SS`
4. **ต้องการ integer operations**: ใช้ AVX2 `VPADDD/VPSUBD/VPMULLD`
5. **ต้องการ compare และ mask**: `VCMPPS`/`VPCMPEQD` + bitwise AND
6. **ต้องการ gather data**: `VPGATHERDPS` (ช้า, แต่สะดวก)
7. **ต้องการ masked load/store**: `VMASKMOVPS`/`VPMASKMOVD`
8. **ก่อน return/call legacy**: ใช้ `VZEROUPPER` เสมอ!

---

*Part 046 จบ - ดูต่อที่ Part 047: AVX-512 Instructions*
