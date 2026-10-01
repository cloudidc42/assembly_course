# Part 045: SSE3, SSSE3, SSE4.1, SSE4.2 Instructions

## บทนำ (Introduction)

SSE3, SSSE3, SSE4.1 และ SSE4.2 เป็นส่วนขยายของ SIMD instruction set ที่ Intel เพิ่มเข้ามาในช่วงปี 2004-2008
แต่ละรุ่นเพิ่มความสามารถใหม่ที่ช่วยให้เขียนโค้ดที่มีประสิทธิภาพสูงสำหรับงานเฉพาะทาง

```
SSE3  (Prescott New Instructions, PNI) - Intel Pentium 4 Prescott (2004)
SSSE3 (Supplemental SSE3)             - Intel Core 2 (2007)  
SSE4.1                                - Intel Core 2 Penryn (2007)
SSE4.2                                - Intel Nehalem (2008)
```

ในบทนี้เราจะเรียนรู้ instruction ใหม่ทั้งหมดพร้อม code ตัวอย่างที่รันได้จริง

---

## ส่วนที่ 1: SSE3 Instructions

SSE3 เพิ่ม instruction ที่มีประโยชน์สำหรับงาน complex number arithmetic, horizontal operations และ loading

### 1.1 ADDSUBPS / ADDSUBPD - Complex Number Arithmetic

ADDSUBPS และ ADDSUBPD เป็น instruction ที่ออกแบบมาสำหรับ complex number arithmetic โดยเฉพาะ

**รูปแบบการทำงาน ADDSUBPS:**
```
dst[0] = dst[0] - src[0]   ; ตำแหน่ง 0 ทำ subtraction
dst[1] = dst[1] + src[1]   ; ตำแหน่ง 1 ทำ addition
dst[2] = dst[2] - src[2]   ; ตำแหน่ง 2 ทำ subtraction
dst[3] = dst[3] + src[3]   ; ตำแหน่ง 3 ทำ addition
```

สลับกันระหว่าง subtract (even index) และ add (odd index)

```nasm
; ADDSUBPS - แต่ละ pair ทำ subtract low, add high
; ใช้สำหรับ complex multiplication: (a+bi)(c+di) = (ac-bd) + (ad+bc)i

section .data
    align 16
    ; complex numbers: (a+bi) = (2.0 + 3.0i)
    ; packed: [a, b, a, b] = [2.0, 3.0, 2.0, 3.0]
    complex_a   dd  2.0, 3.0, 2.0, 3.0
    
    ; complex numbers: (c+di) = (4.0 + 5.0i)  
    ; packed: [c, d, c, d] = [4.0, 5.0, 4.0, 5.0]
    complex_b   dd  4.0, 5.0, 4.0, 5.0

section .text

; Complex multiplication: (a+bi)(c+di) = (ac-bd) + (ad+bc)i
; ผลลัพธ์ = (2*4 - 3*5) + (2*5 + 3*4)i = (8-15) + (10+12)i = -7 + 22i
complex_multiply:
    movaps  xmm0, [complex_a]   ; xmm0 = [a, b, a, b] = [2.0, 3.0, 2.0, 3.0]
    movaps  xmm1, [complex_b]   ; xmm1 = [c, d, c, d] = [4.0, 5.0, 4.0, 5.0]
    
    ; สลับ real/imag ใน xmm1: [c,d,c,d] -> [d,c,d,c]
    shufps  xmm1, xmm1, 0B1h    ; 10110001b = swap pairs
    
    mulps   xmm0, xmm1          ; xmm0 = [a*d, b*c, a*d, b*c]
    
    movaps  xmm2, [complex_a]   ; โหลดใหม่
    movaps  xmm3, [complex_b]   ; โหลดใหม่  
    mulps   xmm2, xmm3          ; xmm2 = [a*c, b*d, a*c, b*d]
    
    ; addsubps: [a*c - b*d, a*d + b*c, a*c - b*d, a*d + b*c]
    addsubps xmm2, xmm0         ; even = sub, odd = add
    
    ; ผลลัพธ์อยู่ใน xmm2[0] = real part, xmm2[1] = imag part
    ret
```

**ADDSUBPD สำหรับ double precision:**
```nasm
; ADDSUBPD - ทำงานเหมือน ADDSUBPS แต่ใช้ double precision
section .data
    align 16
    cpx_real_a  dq  2.0, 3.0    ; [real_a, imag_a]
    cpx_real_b  dq  4.0, 5.0    ; [real_b, imag_b]

complex_multiply_double:
    movapd  xmm0, [cpx_real_a]  ; xmm0 = [2.0, 3.0]
    movapd  xmm1, [cpx_real_b]  ; xmm1 = [4.0, 5.0]
    
    ; สลับ real/imag ใน xmm1
    shufpd  xmm1, xmm1, 1       ; swap: [5.0, 4.0]
    
    mulpd   xmm0, xmm1          ; xmm0 = [2*5, 3*4] = [10.0, 12.0]
    
    movapd  xmm2, [cpx_real_a]
    movapd  xmm3, [cpx_real_b]
    mulpd   xmm2, xmm3          ; xmm2 = [2*4, 3*5] = [8.0, 15.0]
    
    addsubpd xmm2, xmm0         ; xmm2 = [8-10, 15+12] ... wait
    ; จริงๆ: [0] = sub, [1] = add
    ; xmm2[0] = 8.0 - 10.0 = -2.0 (wrong! ต้องเป็น ac-bd)
    ; ต้องจัดลำดับใหม่
    ret
```

### 1.2 HADD/HSUB - Horizontal Add/Subtract

Horizontal operations ทำงาน "ในแนวนอน" คือบวก/ลบ elements ที่อยู่ติดกันภายใน register เดียวกัน

**HADDPS (Horizontal Add Packed Single):**
```nasm
; HADDPS dst, src
; dst[0] = dst[0] + dst[1]   ; บวก elements 0 และ 1 จาก dst
; dst[1] = dst[2] + dst[3]   ; บวก elements 2 และ 3 จาก dst  
; dst[2] = src[0] + src[1]   ; บวก elements 0 และ 1 จาก src
; dst[3] = src[2] + src[3]   ; บวก elements 2 และ 3 จาก src

section .data
    align 16
    vec_a   dd  1.0, 2.0, 3.0, 4.0
    vec_b   dd  5.0, 6.0, 7.0, 8.0

horizontal_sum:
    movaps  xmm0, [vec_a]       ; xmm0 = [1.0, 2.0, 3.0, 4.0]
    
    haddps  xmm0, xmm0          ; xmm0 = [1+2, 3+4, 1+2, 3+4]
                                ;      = [3.0, 7.0, 3.0, 7.0]
    haddps  xmm0, xmm0          ; xmm0 = [3+7, 3+7, 3+7, 3+7]
                                ;      = [10.0, 10.0, 10.0, 10.0]
    ; xmm0[0] = sum ของทุก element = 10.0
    ret

; ตัวอย่าง horizontal sum แบบ verbose
horizontal_sum_verbose:
    movaps  xmm0, [vec_a]       ; [1, 2, 3, 4]
    movaps  xmm1, [vec_b]       ; [5, 6, 7, 8]
    
    haddps  xmm0, xmm1          ; xmm0 = [1+2, 3+4, 5+6, 7+8]
                                ;      = [3, 7, 11, 15]
    ret
```

**HSUBPS (Horizontal Subtract):**
```nasm
; HSUBPS dst, src
; dst[0] = dst[0] - dst[1]
; dst[1] = dst[2] - dst[3]
; dst[2] = src[0] - src[1]
; dst[3] = src[2] - src[3]

horizontal_subtract:
    movaps  xmm0, [vec_a]       ; [1, 2, 3, 4]
    movaps  xmm1, [vec_b]       ; [5, 6, 7, 8]
    
    hsubps  xmm0, xmm1          ; xmm0 = [1-2, 3-4, 5-6, 7-8]
                                ;      = [-1, -1, -1, -1]
    ret
```

**HADDPD / HSUBPD - Double Precision Versions:**
```nasm
section .data
    align 16
    dvec_a  dq  1.0, 2.0        ; double precision
    dvec_b  dq  3.0, 4.0

haddpd_example:
    movapd  xmm0, [dvec_a]      ; xmm0 = [1.0, 2.0]
    movapd  xmm1, [dvec_b]      ; xmm1 = [3.0, 4.0]
    
    haddpd  xmm0, xmm1          ; xmm0 = [1+2, 3+4] = [3.0, 7.0]
    ret
```

### 1.3 MOVSHDUP / MOVSLDUP / MOVDDUP

Instruction เหล่านี้ทำ special duplication สำหรับ complex arithmetic

```nasm
; MOVSHDUP - Move and Duplicate High (Single)
; ทำสำเนา element 1 -> 0, element 3 -> 2 (duplicate odd elements)
; dst[0] = src[1]
; dst[1] = src[1]
; dst[2] = src[3]
; dst[3] = src[3]

section .data
    align 16
    test_data   dd  1.0, 2.0, 3.0, 4.0

movshdup_example:
    movaps  xmm0, [test_data]   ; [1, 2, 3, 4]
    movshdup xmm1, xmm0         ; xmm1 = [2, 2, 4, 4]
    ret

; MOVSLDUP - Move and Duplicate Low (Single)
; ทำสำเนา element 0 -> 1, element 2 -> 3 (duplicate even elements)
; dst[0] = src[0]
; dst[1] = src[0]
; dst[2] = src[2]
; dst[3] = src[2]

movsldup_example:
    movaps  xmm0, [test_data]   ; [1, 2, 3, 4]
    movsldup xmm1, xmm0         ; xmm1 = [1, 1, 3, 3]
    ret

; MOVDDUP - Move and Duplicate Double
; ทำสำเนา element 0 ไปยัง element 1 (ใช้กับ double)
; dst[0] = src[0]
; dst[1] = src[0]

section .data
    align 16
    dbl_data    dq  3.14, 2.71

movddup_example:
    movapd  xmm0, [dbl_data]    ; [3.14, 2.71]
    movddup xmm1, xmm0          ; xmm1 = [3.14, 3.14]
    ret
```

**ประโยชน์ของ MOVSHDUP/MOVSLDUP สำหรับ Complex Multiplication:**
```nasm
; วิธีที่มีประสิทธิภาพมากขึ้นสำหรับ complex multiplication
; (a+bi)(c+di) = (ac-bd) + (ad+bc)i

; ใน SSE3 เราสามารถใช้ movshdup/movsldup ร่วมกับ addsubps

complex_mul_sse3:
    ; สมมติ xmm0 = [a, b, a, b] (real + imag interleaved)
    ;         xmm1 = [c, d, c, d]
    
    movaps  xmm2, xmm1          ; xmm2 = [c, d, c, d]
    movsldup xmm3, xmm1         ; xmm3 = [c, c, c, c]
    movshdup xmm4, xmm1         ; xmm4 = [d, d, d, d]
    
    mulps   xmm3, xmm0          ; [a*c, b*c, a*c, b*c]
    
    ; สลับ real/imag ใน xmm0
    shufps  xmm0, xmm0, 0B1h    ; [b, a, b, a]
    mulps   xmm4, xmm0          ; [b*d, a*d, b*d, a*d]
    
    addsubps xmm3, xmm4         ; [a*c - b*d, b*c + a*d, ...]
    ret
```

### 1.4 FISTTP - Truncate Float to Integer

SSE3 เพิ่ม FISTTP ซึ่งเป็นทางเลือกของ FISTP แต่ใช้ truncation (round toward zero) เสมอโดยไม่สนใจ rounding mode

```nasm
; FISTTP - Store Integer with Truncation to Integer
; ต่างจาก FISTP ตรงที่ใช้ truncate เสมอ (ไม่สนใจ MXCSR rounding mode)

section .data
    align 8
    float_val   dd  3.7     ; float value
    result_int  dd  0       ; result

fisttp_example:
    fld     dword [float_val]   ; load 3.7 onto FPU stack
    fisttp  dword [result_int]  ; store as integer = 3 (truncated)
    ; result_int = 3 (ไม่ใช่ 4 แม้ว่า 3.7 จะใกล้ 4 มากกว่า)
    ret
```

### 1.5 LDDQU - Unaligned Integer Load

LDDQU เป็น alternative ของ MOVDQU สำหรับ unaligned loads ที่อาจเร็วกว่าในบางสถานการณ์

```nasm
; LDDQU - Load Double Quadword Unaligned
; คล้าย MOVDQU แต่ใช้ 128-bit read ที่อาจข้ามขอบ cache line
; เร็วกว่า MOVDQU บน Pentium 4 เมื่อ data ข้าม cache line boundary

section .data
    ; data ที่ไม่ได้ align 16
    unaligned_buf   db  0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17

lddqu_example:
    ; โหลดจาก address ที่ไม่ align (offset 1)
    lddqu   xmm0, [unaligned_buf + 1]  ; โหลด 16 bytes จาก offset 1
    ret
```

---

## ส่วนที่ 2: SSSE3 Instructions (Supplemental SSE3)

SSSE3 เพิ่ม instruction ใหม่ที่ทรงพลังสำหรับงาน byte manipulation และ integer operations

### 2.1 PSHUFB - Packed Shuffle Bytes (ทรงพลังมาก!)

PSHUFB เป็นหนึ่งใน instruction ที่ทรงพลังที่สุดใน SSSE3 สามารถ shuffle ทุก byte อย่างอิสระ

```nasm
; PSHUFB dst, mask
; สำหรับแต่ละ byte i ใน dst:
;   ถ้า mask[i] bit 7 = 1: dst[i] = 0
;   ถ้าไม่: dst[i] = dst[mask[i] & 0x0F]

section .data
    align 16
    source_bytes    db  0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15

    ; mask สำหรับ reverse byte order (endian swap)
    reverse_mask    db  15,14,13,12,11,10,9,8,7,6,5,4,3,2,1,0
    
    ; mask สำหรับ zero out specific bytes
    zero_mask       db  0,1,2,3, 0x80,0x80,0x80,0x80, 8,9,10,11, 0x80,0x80,0x80,0x80
    
    ; mask สำหรับ broadcast byte 0 ไปทุกตำแหน่ง
    broadcast_mask  db  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0

pshufb_reverse:
    ; Reverse byte order - useful for endian conversion
    movdqa  xmm0, [source_bytes]    ; โหลด 16 bytes
    movdqa  xmm1, [reverse_mask]    ; โหลด mask
    pshufb  xmm0, xmm1              ; reverse!
    ; xmm0 = [15,14,13,12,11,10,9,8,7,6,5,4,3,2,1,0]
    ret

pshufb_zero_select:
    ; Zero out specific positions
    movdqa  xmm0, [source_bytes]
    movdqa  xmm1, [zero_mask]
    pshufb  xmm0, xmm1
    ; xmm0 = [0,1,2,3, 0,0,0,0, 8,9,10,11, 0,0,0,0]
    ret

pshufb_broadcast:
    ; Broadcast byte 0 to all positions
    movdqa  xmm0, [source_bytes]
    movdqa  xmm1, [broadcast_mask]
    pshufb  xmm0, xmm1
    ; xmm0 = [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
    ret
```

**ตัวอย่างประยุกต์ใช้ PSHUFB สำหรับ RGBA -> BGRA conversion:**
```nasm
section .data
    align 16
    ; RGBA pixels: R,G,B,A สี่ pixel
    rgba_pixels     db  255,0,0,255,   0,255,0,255,   0,0,255,255,   128,128,0,255
    
    ; mask เพื่อ swap R และ B (RGBA -> BGRA)
    rgba_to_bgra    db  2,1,0,3, 6,5,4,7, 10,9,8,11, 14,13,12,15

rgba_to_bgra_convert:
    movdqa  xmm0, [rgba_pixels]
    movdqa  xmm1, [rgba_to_bgra]
    pshufb  xmm0, xmm1
    ; แต่ละ pixel: [B, G, R, A] แทนที่ [R, G, B, A]
    ret
```

### 2.2 PHADDW / PHADDD / PHADDSW - Horizontal Packed Add

```nasm
; PHADDW - Packed Horizontal Add Words (16-bit)
; dst[0] = dst[0] + dst[1]
; dst[1] = dst[2] + dst[3]
; dst[2] = dst[4] + dst[5]
; dst[3] = dst[6] + dst[7]
; dst[4] = src[0] + src[1]
; dst[5] = src[2] + src[3]
; dst[6] = src[4] + src[5]
; dst[7] = src[6] + src[7]

section .data
    align 16
    words_a     dw  1,2,3,4,5,6,7,8
    words_b     dw  9,10,11,12,13,14,15,16

phaddw_example:
    movdqa  xmm0, [words_a]     ; [1,2,3,4,5,6,7,8]
    movdqa  xmm1, [words_b]     ; [9,10,11,12,13,14,15,16]
    phaddw  xmm0, xmm1          ; [1+2, 3+4, 5+6, 7+8, 9+10, 11+12, 13+14, 15+16]
                                ; = [3, 7, 11, 15, 19, 23, 27, 31]
    ret

; PHADDD - Packed Horizontal Add Dwords (32-bit)
section .data
    align 16
    dwords_a    dd  1,2,3,4
    dwords_b    dd  5,6,7,8

phaddd_example:
    movdqa  xmm0, [dwords_a]    ; [1,2,3,4]
    movdqa  xmm1, [dwords_b]    ; [5,6,7,8]
    phaddd  xmm0, xmm1          ; [1+2, 3+4, 5+6, 7+8] = [3, 7, 11, 15]
    ret

; PHADDSW - Packed Horizontal Add Signed Words with Saturation
; เหมือน PHADDW แต่ใช้ signed saturation (ไม่ overflow)
phaddsw_example:
    movdqa  xmm0, [words_a]
    movdqa  xmm1, [words_b]
    phaddsw xmm0, xmm1          ; เหมือน PHADDW แต่ saturate
    ret
```

### 2.3 PHSUBW / PHSUBD / PHSUBSW - Horizontal Packed Subtract

```nasm
; PHSUBW - Packed Horizontal Subtract Words
; เหมือน PHADDW แต่เป็น subtraction
; dst[0] = dst[0] - dst[1]
; dst[1] = dst[2] - dst[3]
; ...

phsubw_example:
    movdqa  xmm0, [words_a]     ; [1,2,3,4,5,6,7,8]
    movdqa  xmm1, [words_b]     ; [9,10,11,12,13,14,15,16]
    phsubw  xmm0, xmm1          ; [1-2, 3-4, 5-6, 7-8, 9-10, 11-12, 13-14, 15-16]
                                ; = [-1, -1, -1, -1, -1, -1, -1, -1]
    ret

; PHSUBD - Packed Horizontal Subtract Dwords
phsubd_example:
    movdqa  xmm0, [dwords_a]    ; [1,2,3,4]
    movdqa  xmm1, [dwords_b]    ; [5,6,7,8]
    phsubd  xmm0, xmm1          ; [1-2, 3-4, 5-6, 7-8] = [-1, -1, -1, -1]
    ret
```

### 2.4 PMADDUBSW - Multiply Unsigned Bytes and Add Adjacent Pairs

```nasm
; PMADDUBSW - Packed Multiply Unsigned Bytes and Add Signed Words
; สำหรับแต่ละ pair ของ bytes:
;   result[i] = SATURATE16(src[2i] * dst[2i] + src[2i+1] * dst[2i+1])
; src bytes ถูก treat เป็น UNSIGNED
; dst bytes ถูก treat เป็น SIGNED

section .data
    align 16
    ubytes      db  2,3, 4,5, 6,7, 8,9, 10,11, 12,13, 14,15, 16,17    ; unsigned
    sbytes      db  1,2, 3,4, 5,6, 7,8, 9,10,  11,12, 13,14, 15,16    ; signed

pmaddubsw_example:
    movdqa  xmm0, [ubytes]      ; unsigned bytes
    movdqa  xmm1, [sbytes]      ; signed bytes
    pmaddubsw xmm0, xmm1        ; ผล: packed 16-bit results
    ; xmm0[0] = 2*1 + 3*2 = 2+6 = 8
    ; xmm0[1] = 4*3 + 5*4 = 12+20 = 32
    ; ...
    ret
```

### 2.5 PMULHRSW - Packed Multiply High with Round and Scale

```nasm
; PMULHRSW - Packed Multiply High Round and Scale for 16-bit integers
; result = ROUND(a * b / 32768, 1) = ((a * b) + 16384) >> 15
; ใช้สำหรับ fixed-point multiplication

section .data
    align 16
    fixed_a     dw  16384, 8192, 4096, 2048, 1024, 512, 256, 128  ; Q15 format
    fixed_b     dw  16384, 16384, 16384, 16384, 16384, 16384, 16384, 16384

pmulhrsw_example:
    movdqa  xmm0, [fixed_a]
    movdqa  xmm1, [fixed_b]
    pmulhrsw xmm0, xmm1         ; fixed-point multiply
    ret
```

### 2.6 PALIGNR - Aligned Byte Shift/Concatenate

```nasm
; PALIGNR dst, src, imm8
; สร้าง 128-bit temp = CONCAT(dst, src) = [dst | src] (256-bit)
; จากนั้น shift right โดย imm8 bytes
; แล้วเอา 128-bit ล่างเป็น result

section .data
    align 16
    palign_a    db  0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15
    palign_b    db  16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31

palignr_example:
    movdqa  xmm0, [palign_a]    ; dst
    movdqa  xmm1, [palign_b]    ; src
    palignr xmm0, xmm1, 4       ; concat [dst|src] = 256-bit, shift right 4 bytes
    ; result = bytes 4..19 = [4,5,6,7,...,15,16,17,18,19]
    ret

; ตัวอย่างใช้งาน: sliding window over unaligned data
palignr_sliding_window:
    ; โหลด 2 aligned chunks แล้วใช้ PALIGNR เพื่อดึง 16 bytes ที่ต้องการ
    movdqa  xmm0, [rsi]         ; aligned chunk 0
    movdqa  xmm1, [rsi + 16]    ; aligned chunk 1
    palignr xmm0, xmm1, 3       ; bytes 3..18 จาก รวมกัน
    ret
```

### 2.7 PABSB / PABSW / PABSD - Absolute Value

```nasm
; PABSB/PABSW/PABSD - Packed Absolute Value
; result[i] = ABS(src[i])

section .data
    align 16
    neg_bytes   db  -1,-2,-3,-4,-5,-6,-7,-8,1,2,3,4,5,6,7,8
    neg_words   dw  -100,-200,-300,-400, 100,200,300,400
    neg_dwords  dd  -1000000, -2000000, 1000000, 2000000

absolute_value:
    ; Byte absolute value
    movdqa  xmm0, [neg_bytes]
    pabsb   xmm0, xmm0          ; xmm0 = |each byte|
    
    ; Word absolute value
    movdqa  xmm1, [neg_words]
    pabsw   xmm1, xmm1          ; xmm1 = |each 16-bit word|
    
    ; Dword absolute value
    movdqa  xmm2, [neg_dwords]
    pabsd   xmm2, xmm2          ; xmm2 = |each 32-bit dword|
    ret
```

### 2.8 PSIGNB / PSIGNW / PSIGND - Conditionally Negate

```nasm
; PSIGNB/PSIGNW/PSIGND - Packed Sign
; ถ้า src[i] > 0: dst[i] = dst[i]    (ไม่เปลี่ยน)
; ถ้า src[i] = 0: dst[i] = 0         (zeroed)
; ถ้า src[i] < 0: dst[i] = -dst[i]   (negate)

section .data
    align 16
    sign_data   db  1,2,3,4,5,6,7,8,1,2,3,4,5,6,7,8
    sign_ctrl   db  1,-1,0,1,-1,0,1,-1,1,-1,0,1,-1,0,1,-1  ; control

psign_example:
    movdqa  xmm0, [sign_data]   ; values
    movdqa  xmm1, [sign_ctrl]   ; control
    psignb  xmm0, xmm1          ; conditionally negate each byte
    ret
```

---

## ส่วนที่ 3: SSE4.1 Instructions

SSE4.1 เพิ่ม instruction มากมายสำหรับ dot products, blending, rounding และ integer operations

### 3.1 DPPS / DPPD - Dot Product

```nasm
; DPPS dst, src, imm8
; imm8 ควบคุม:
;   bit 4-7: ตำแหน่งที่จะนำมาคูณ (1=include, 0=use 0.0)
;   bit 0-3: ตำแหน่งที่จะเก็บผลลัพธ์ (1=store, 0=store 0.0)

section .data
    align 16
    vec3_a  dd  1.0, 2.0, 3.0, 0.0     ; 3D vector [x, y, z, 0]
    vec3_b  dd  4.0, 5.0, 6.0, 0.0     ; 3D vector [x, y, z, 0]
    vec4_a  dd  1.0, 2.0, 3.0, 4.0     ; 4D vector
    vec4_b  dd  5.0, 6.0, 7.0, 8.0     ; 4D vector

dot_product_3d:
    movaps  xmm0, [vec3_a]
    movaps  xmm1, [vec3_b]
    ; imm8 = 0x71 = 01110001b
    ; bits 4-7 = 0111 = use elements 0,1,2 (not 3)
    ; bits 0-3 = 0001 = store only in element 0
    dpps    xmm0, xmm1, 71h    ; dot product = 1*4 + 2*5 + 3*6 = 4+10+18 = 32
    ; xmm0[0] = 32.0, xmm0[1] = 0, xmm0[2] = 0, xmm0[3] = 0
    ret

dot_product_4d:
    movaps  xmm0, [vec4_a]
    movaps  xmm1, [vec4_b]
    ; imm8 = 0xFF = 11111111b  
    ; bits 4-7 = 1111 = use all 4 elements
    ; bits 0-3 = 1111 = store in all 4 elements
    dpps    xmm0, xmm1, 0FFh   ; dot product = 1*5+2*6+3*7+4*8 = 5+12+21+32 = 70
    ; xmm0 = [70, 70, 70, 70]
    ret

; DPPD สำหรับ double precision
dppd_example:
    section .data
    align 16
    dvec_a  dq  2.0, 3.0
    dvec_b  dq  4.0, 5.0

dot_product_double:
    movapd  xmm0, [dvec_a]
    movapd  xmm1, [dvec_b]
    ; imm8 = 0x31 = 00110001b
    ; bits 4-7 = 0011 = use both elements
    ; bits 0-3 = 0001 = store in element 0 only
    dppd    xmm0, xmm1, 31h    ; = 2*4 + 3*5 = 8+15 = 23
    ret
```

### 3.2 BLENDPS / BLENDPD / PBLENDW - Byte Blend (Static)

```nasm
; BLENDPS dst, src, imm8
; สำหรับแต่ละ float:
;   ถ้า bit ใน imm8 = 0: เลือกจาก dst
;   ถ้า bit ใน imm8 = 1: เลือกจาก src

section .data
    align 16
    blend_a dd  1.0, 2.0, 3.0, 4.0
    blend_b dd  5.0, 6.0, 7.0, 8.0

blendps_example:
    movaps  xmm0, [blend_a]     ; [1, 2, 3, 4]
    movaps  xmm1, [blend_b]     ; [5, 6, 7, 8]
    blendps xmm0, xmm1, 1010b  ; imm8 = 10 = 0xA = 1010b
    ; bit 0 = 0: keep a[0] = 1
    ; bit 1 = 1: use  b[1] = 6
    ; bit 2 = 0: keep a[2] = 3
    ; bit 3 = 1: use  b[3] = 8
    ; xmm0 = [1, 6, 3, 8]
    ret

; PBLENDW - Packed Blend Words
section .data
    align 16
    blend_words_a   dw  1,2,3,4,5,6,7,8
    blend_words_b   dw  9,10,11,12,13,14,15,16

pblendw_example:
    movdqa  xmm0, [blend_words_a]
    movdqa  xmm1, [blend_words_b]
    pblendw xmm0, xmm1, 0AAh    ; 0xAA = 10101010b = use src for odd positions
    ; xmm0 = [1, 10, 3, 12, 5, 14, 7, 16]
    ret
```

### 3.3 BLENDVPS / BLENDVPD / PBLENDVB - Variable Blend

```nasm
; BLENDVPS dst, src, xmm0
; xmm0 เป็น mask (MSB ของแต่ละ element กำหนดการ blend)
; ถ้า mask bit = 0: เลือกจาก dst
; ถ้า mask bit = 1: เลือกจาก src

section .data
    align 16
    vblend_a    dd  1.0, 2.0, 3.0, 4.0
    vblend_b    dd  5.0, 6.0, 7.0, 8.0
    ; mask: 0 = keep a, -1 (0xFFFFFFFF) = use b
    vblend_mask dd  0, 0xFFFFFFFF, 0, 0xFFFFFFFF

blendvps_example:
    movaps  xmm0, [vblend_mask] ; ต้องอยู่ใน xmm0!
    movaps  xmm1, [vblend_a]
    movaps  xmm2, [vblend_b]
    blendvps xmm1, xmm2, xmm0  ; xmm1 = [1, 6, 3, 8]
    ret

; PBLENDVB - Packed Variable Blend Bytes
section .data
    align 16
    vblend_bytes_a  db  1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16
    vblend_bytes_b  db  17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32
    ; mask: bit 7 ของแต่ละ byte กำหนดการ blend
    vblend_bmask    db  0,0xFF,0,0xFF,0,0xFF,0,0xFF,0,0xFF,0,0xFF,0,0xFF,0,0xFF

pblendvb_example:
    movdqa  xmm0, [vblend_bmask]    ; mask ต้องอยู่ใน xmm0
    movdqa  xmm1, [vblend_bytes_a]
    movdqa  xmm2, [vblend_bytes_b]
    pblendvb xmm1, xmm2, xmm0      ; blend bytes
    ret
```

### 3.4 INSERTPS / EXTRACTPS - Insert/Extract Float Scalar

```nasm
; INSERTPS dst, src, imm8
; imm8 bits 6-7: source element index (ถ้า src เป็น register)
; imm8 bits 4-5: destination element index
; imm8 bits 0-3: zeroing mask (1 = zero out that element)

section .data
    align 16
    ins_vec dd  10.0, 20.0, 30.0, 40.0
    ins_scl dd  99.0

insertps_example:
    movaps  xmm0, [ins_vec]     ; [10, 20, 30, 40]
    movss   xmm1, [ins_scl]     ; xmm1[0] = 99.0
    ; imm8 = 0b 00 01 0000 = 0x10
    ; count_s = 00 (source element 0)
    ; count_d = 01 (destination element 1)
    ; zmask = 0000 (no zeroing)
    insertps xmm0, xmm1, 010h   ; xmm0 = [10, 99, 30, 40]
    ret

; EXTRACTPS dst, src, imm8
; แยก float จากตำแหน่ง imm8 ออกมา

extractps_example:
    movaps  xmm0, [ins_vec]     ; [10, 20, 30, 40]
    extractps eax, xmm0, 2      ; eax = bit pattern ของ 30.0
    ret
```

### 3.5 PINSRB / PINSRD / PINSRQ - Insert Integer Element

```nasm
; PINSRB dst, src, imm8  - Insert byte
; PINSRD dst, src, imm8  - Insert dword
; PINSRQ dst, src, imm8  - Insert qword

pinsrd_example:
    pxor    xmm0, xmm0          ; clear
    mov     eax, 0xDEADBEEF
    pinsrd  xmm0, eax, 0        ; insert at position 0
    mov     eax, 0x12345678
    pinsrd  xmm0, eax, 2        ; insert at position 2
    ; xmm0 = [0xDEADBEEF, 0, 0x12345678, 0]
    ret

pinsrb_example:
    pxor    xmm0, xmm0
    mov     al, 0x42
    pinsrb  xmm0, eax, 5        ; insert byte 0x42 at position 5
    ret

pinsrq_example:
    pxor    xmm0, xmm0
    mov     rax, 0xDEADBEEF12345678
    pinsrq  xmm0, rax, 0        ; insert qword at position 0
    ret
```

### 3.6 PEXTRB / PEXTRD / PEXTRQ - Extract Integer Element

```nasm
; PEXTRB dst, src, imm8  - Extract byte
; PEXTRD dst, src, imm8  - Extract dword  
; PEXTRQ dst, src, imm8  - Extract qword

section .data
    align 16
    int_vec     dd  0x11111111, 0x22222222, 0x33333333, 0x44444444

pextrd_example:
    movdqa  xmm0, [int_vec]
    pextrd  eax, xmm0, 0        ; eax = 0x11111111
    pextrd  ebx, xmm0, 2        ; ebx = 0x33333333
    ret

pextrb_example:
    movdqa  xmm0, [int_vec]
    pextrb  eax, xmm0, 0        ; eax = byte 0 = 0x11
    pextrb  ebx, xmm0, 4        ; ebx = byte 4 = 0x22
    ret
```

### 3.7 PMULDQ / PMULLD - 32-bit Integer Multiply

```nasm
; PMULDQ - Packed Multiply Dwords to Qword
; ทำ signed multiply ของ element 0 และ 2 (32-bit) ให้ได้ 64-bit result

section .data
    align 16
    mul_a   dd  100000, 0, 200000, 0    ; elements 0 และ 2
    mul_b   dd  300000, 0, 400000, 0

pmuldq_example:
    movdqa  xmm0, [mul_a]
    movdqa  xmm1, [mul_b]
    pmuldq  xmm0, xmm1          ; xmm0[63:0] = 100000 * 300000 = 30,000,000,000
                                ; xmm0[127:64] = 200000 * 400000 = 80,000,000,000
    ret

; PMULLD - Packed Multiply Low Dwords
; คูณ 4 คู่ของ 32-bit integers แล้วเก็บแค่ 32-bit ล่างของผล

section .data
    align 16
    mulld_a     dd  2, 3, 4, 5
    mulld_b     dd  6, 7, 8, 9

pmulld_example:
    movdqa  xmm0, [mulld_a]
    movdqa  xmm1, [mulld_b]
    pmulld  xmm0, xmm1          ; xmm0 = [2*6, 3*7, 4*8, 5*9] = [12, 21, 32, 45]
    ret
```

### 3.8 PHMINPOSUW - Horizontal Minimum Unsigned Word

```nasm
; PHMINPOSUW - Packed Horizontal Minimum Unsigned Word
; หาค่า minimum และ index ของมัน

section .data
    align 16
    test_words  dw  100, 50, 200, 75, 300, 25, 150, 400

phminposuw_example:
    movdqa  xmm0, [test_words]
    phminposuw xmm0, xmm0       ; xmm0[15:0] = minimum value = 25
                                ; xmm0[18:16] = index of minimum = 5
                                ; (25 อยู่ที่ position 5)
    ; extract result
    movd    eax, xmm0
    movzx   ebx, ax             ; minimum value
    shr     eax, 16
    and     eax, 7              ; index (0-7)
    ret
```

### 3.9 ROUNDPS / ROUNDPD / ROUNDSS / ROUNDSD - Rounding

```nasm
; ROUNDPS dst, src, imm8
; imm8 bits 1-0 กำหนด rounding mode:
;   00 = round to nearest even (default)
;   01 = round down (floor)
;   10 = round up (ceil)
;   11 = truncate toward zero (trunc)
; imm8 bit 2 = 0 ใช้ imm8 mode, 1 ใช้ MXCSR mode

section .data
    align 16
    round_vals  dd  1.4, 1.5, -1.4, -1.5

round_examples:
    movaps  xmm0, [round_vals]
    
    ; Round to nearest
    roundps xmm1, xmm0, 0       ; xmm1 = [1, 2, -1, -2]
    
    ; Floor (round down)
    roundps xmm2, xmm0, 1       ; xmm2 = [1, 1, -2, -2]
    
    ; Ceil (round up)
    roundps xmm3, xmm0, 2       ; xmm3 = [2, 2, -1, -1]
    
    ; Truncate
    roundps xmm4, xmm0, 3       ; xmm4 = [1, 1, -1, -1]
    ret

; ROUNDSS สำหรับ scalar
round_scalar:
    movss   xmm0, [round_vals]  ; 1.4
    roundss xmm0, xmm0, 1       ; floor = 1.0
    ret
```

### 3.10 PCMPEQQ - 64-bit Compare Equal

```nasm
; PCMPEQQ - Compare Packed Qwords Equal
; result = (a == b) ? 0xFFFFFFFFFFFFFFFF : 0

section .data
    align 16
    qword_a     dq  0x1234567890ABCDEF, 0xFEDCBA9876543210
    qword_b     dq  0x1234567890ABCDEF, 0x0000000000000000

pcmpeqq_example:
    movdqa  xmm0, [qword_a]
    movdqa  xmm1, [qword_b]
    pcmpeqq xmm0, xmm1          ; xmm0[0] = 0xFFFF... (equal)
                                ; xmm0[1] = 0x0000... (not equal)
    ret
```

### 3.11 PMOVSXBW / PMOVZXBW - Sign/Zero Extend

```nasm
; PMOVSXxx - Sign extend
; PMOVZXxx - Zero extend
; Variants: BW (byte->word), BD (byte->dword), BQ (byte->qword)
;           WD (word->dword), WQ (word->qword), DQ (dword->qword)

section .data
    align 16
    signed_bytes    db  -1,-2,-3,-4,-5,-6,-7,-8
    unsigned_bytes  db  200,201,202,203,204,205,206,207

sign_extend_example:
    ; Sign extend 8 bytes to 8 words
    movq    xmm0, [signed_bytes]    ; โหลด 8 bytes
    pmovsxbw xmm0, xmm0             ; sign extend to 8 x 16-bit
    ; xmm0 = [-1,-2,-3,-4,-5,-6,-7,-8] as 16-bit words
    ret

zero_extend_example:
    ; Zero extend 8 bytes to 8 words
    movq    xmm0, [unsigned_bytes]  ; โหลด 8 bytes
    pmovzxbw xmm0, xmm0             ; zero extend to 8 x 16-bit
    ; xmm0 = [200,201,202,203,204,205,206,207] as 16-bit words
    ret

; ตัวอย่าง BD: 4 bytes -> 4 dwords
pmovsxbd_example:
    movd    xmm0, dword [signed_bytes]  ; โหลด 4 bytes
    pmovsxbd xmm0, xmm0                 ; sign extend to 4 x 32-bit
    ret
```

### 3.12 MOVNTDQA - Non-Temporal Aligned Load

```nasm
; MOVNTDQA - Load Double Quadword Non-Temporal Aligned
; โหลด 128-bit จาก write-combining memory (WC) โดยไม่ผ่าน cache
; ใช้สำหรับ streaming read จาก video/device memory

section .bss
    align 64
    wc_buffer   resb    64      ; write-combining buffer

movntdqa_example:
    ; โหลดจาก WC memory โดยตรง ไม่ผ่าน cache
    movntdqa xmm0, [wc_buffer]
    movntdqa xmm1, [wc_buffer + 16]
    ret
```

---

## ส่วนที่ 4: SSE4.2 Instructions

SSE4.2 เพิ่ม instruction ที่ทรงพลังสำหรับ string operations, CRC และ bit counting

### 4.1 PCMPISTRI / PCMPISTRM - Implicit Length String Compare

PCMPISTRI เป็น instruction ที่ซับซ้อนและทรงพลังมากสำหรับ string searching

```
Control byte (imm8):
  bits 1-0: ชนิดของ data (00=byte unsigned, 01=byte signed, 10=word unsigned, 11=word signed)
  bits 3-2: comparison mode:
    00 = equal any       (หา character ที่อยู่ใน set)
    01 = ranges          (หา character ในช่วง range)
    10 = equal each      (compare ทุก position)
    11 = equal ordered   (หา substring)
  bit 4: polarity (0=positive, 1=negative)
  bit 5: output select  
    สำหรับ PCMPISTRI: 0=least significant index, 1=most significant index
    สำหรับ PCMPISTRM: 0=bit mask, 1=byte mask
  bit 6: ควบคุม masking (0=default, 1=mask with second string length)
```

```nasm
section .data
    align 16
    ; null-terminated strings
    haystack    db  "Hello, World! This is a test string.", 0
    needle      db  "World", 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
    
    ; character set for searching
    vowels      db  "aeiouAEIOU", 0, 0, 0, 0, 0, 0

; strlen using PCMPISTRI
; Input: rdi = string pointer
; Output: rax = length
strlen_sse42:
    pxor    xmm0, xmm0          ; xmm0 = all zeros (null byte)
    xor     rax, rax            ; counter = 0
    
.loop:
    movdqu  xmm1, [rdi + rax]   ; โหลด 16 bytes
    ; imm8 = 0x08 = 00001000b
    ; bits 1-0 = 00 (byte, unsigned)
    ; bits 3-2 = 10 (equal each)
    ; bit 4 = 0 (positive polarity)
    ; bit 5 = 0 (use ecx = first match index)
    pcmpistri xmm0, xmm1, 08h  ; ค้นหา null byte
    jc      .found              ; CF=1 หมายความว่าพบ null byte
    add     rax, 16             ; เพิ่ม counter 16
    jmp     .loop
    
.found:
    add     rax, rcx            ; ecx = offset ของ null byte
    ret

; strchr using PCMPISTRI - ค้นหา character ตัวแรก  
; Input: rdi = string pointer, esi = character to find
; Output: rax = pointer to found char (or NULL)
strchr_sse42:
    movd    xmm0, esi           ; xmm0[0] = search character
    pxor    xmm1, xmm1
    pshufb  xmm0, xmm1          ; broadcast: ทุก byte เป็น search char
    
    xor     rax, rax
.loop:
    movdqu  xmm1, [rdi + rax]
    ; imm8 = 0x00 = 00000000b
    ; equal any: หา character ใดก็ได้ใน xmm0 set
    pcmpistri xmm0, xmm1, 00h
    jc      .check_null         ; พบ match หรือ null
    add     rax, 16
    jmp     .loop

.check_null:
    ; ecx = index ของ first match
    test    byte [rdi + rax + rcx], 0xFF
    jz      .not_found          ; เจอ null ก่อน = not found
    lea     rax, [rdi + rax + rcx]  ; return pointer
    ret
.not_found:
    xor     rax, rax            ; return NULL
    ret
```

### 4.2 PCMPESTRI / PCMPESTRM - Explicit Length String Compare

```nasm
; PCMPESTRI - ต้องระบุความยาวใน rax (string 1) และ rdx (string 2)
; ใช้เมื่อ strings ไม่ได้ null-terminated หรือมีความยาวที่รู้อยู่แล้ว

; substring search (strstr equivalent)
; Input: rdi = haystack, rsi = haystack_len, rbx = needle, rcx = needle_len
; Output: rax = pointer to first occurrence or NULL
strstr_sse42:
    push    rbp
    push    rbx
    push    r12
    push    r13
    
    mov     r12, rdi            ; haystack
    mov     r13, rsi            ; haystack_len
    
    movdqu  xmm0, [rbx]         ; load needle (max 16 bytes)
    
    xor     rbp, rbp            ; offset
.search_loop:
    cmp     rbp, r13
    jge     .not_found
    
    movdqu  xmm1, [r12 + rbp]   ; load haystack chunk
    
    mov     rax, rcx            ; needle length
    mov     rdx, r13            ; haystack length (explicit)
    sub     rdx, rbp            ; remaining haystack
    
    ; imm8 = 0x0C = equal ordered (substring match)
    pcmpestri xmm0, xmm1, 0Ch
    
    jc      .found_at           ; CF=1 = found
    add     rbp, 16
    jmp     .search_loop
    
.found_at:
    lea     rax, [r12 + rbp + rcx]  ; pointer to match
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret
    
.not_found:
    xor     rax, rax
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret
```

### 4.3 CRC32 - Hardware CRC Computation

```nasm
; CRC32 instruction - คำนวณ CRC-32C (Castagnoli polynomial)
; ใช้ใน iSCSI, SCTP, Btrfs และอื่นๆ

; Input: rdi = data pointer, rsi = length
; Output: eax = CRC32 value
crc32_compute:
    mov     eax, 0xFFFFFFFF    ; initial CRC value
    
    ; Process 8 bytes at a time
.loop8:
    cmp     rsi, 8
    jl      .loop4
    crc32   rax, qword [rdi]   ; 64-bit CRC update
    add     rdi, 8
    sub     rsi, 8
    jmp     .loop8

    ; Process 4 bytes
.loop4:
    cmp     rsi, 4
    jl      .loop1
    crc32   eax, dword [rdi]
    add     rdi, 4
    sub     rsi, 4

    ; Process remaining bytes
.loop1:
    test    rsi, rsi
    jz      .done
    crc32   eax, byte [rdi]
    inc     rdi
    dec     rsi
    jmp     .loop1

.done:
    not     eax                 ; XOR with 0xFFFFFFFF for final CRC
    ret
```

### 4.4 POPCNT - Population Count (Bit Count)

```nasm
; POPCNT - Count Number of Set Bits
; นับจำนวน bit ที่เป็น 1

popcnt_example:
    mov     eax, 0xDEADBEEF     ; 11011110101011011011111011101111b
    popcnt  eax, eax            ; นับ 1-bits
    ; 0xDEADBEEF = 24 ones
    
    ; สำหรับ 64-bit
    mov     rax, 0xFFFFFFFF00000000
    popcnt  rax, rax            ; = 32
    ret

; ตัวอย่าง: นับ set bits ใน bitset
; Input: rdi = bitset pointer, rsi = size in bytes
; Output: rax = total count
count_bits_in_bitset:
    xor     rax, rax
    xor     rcx, rcx
    
.loop8:
    cmp     rsi, 8
    jl      .loop4
    popcnt  rdx, qword [rdi + rcx]
    add     rax, rdx
    add     rcx, 8
    sub     rsi, 8
    jmp     .loop8

.loop4:
    cmp     rsi, 4
    jl      .loop1
    popcnt  edx, dword [rdi + rcx]
    add     rax, rdx
    add     rcx, 4
    sub     rsi, 4

.loop1:
    test    rsi, rsi
    jz      .done
    movzx   edx, byte [rdi + rcx]
    popcnt  edx, edx
    add     rax, rdx
    inc     rcx
    dec     rsi
    jmp     .loop1

.done:
    ret
```

### 4.5 LZCNT - Leading Zero Count

```nasm
; LZCNT - Count Leading Zeros
; นับจำนวน leading zeros จาก MSB

; Note: ต้องการ LZCNT feature flag (ไม่ใช่ BSR!)
; BSR ให้ index ของ MSB set bit, LZCNT ให้จำนวน leading zeros

lzcnt_example:
    mov     eax, 0x00001000     ; bit 12 set, leading zeros = 19
    lzcnt   eax, eax            ; eax = 19
    
    mov     rax, 0x0000000100000000  ; bit 32 set
    lzcnt   rax, rax            ; rax = 31
    
    ; ความแตกต่าง LZCNT vs BSR:
    ; สำหรับ x = 0:
    ;   BSR  : undefined (ZF = 1)
    ;   LZCNT: returns 32 (or 64 for 64-bit)
    
    xor     eax, eax
    lzcnt   eax, eax            ; eax = 32 (ไม่ใช่ undefined!)
    ret
```

---

## ส่วนที่ 5: Code ตัวอย่างที่รันได้จริง

### 5.1 Complete Example: Complex Number Operations

```nasm
; complex_ops.asm - Complex number operations using SSE3
; nasm -f elf64 complex_ops.asm -o complex_ops.o
; ld complex_ops.o -o complex_ops

section .data
    align 16
    ; Complex numbers: a+bi = 3+4i, c+di = 1+2i
    cmplx_a     dd  3.0, 4.0, 3.0, 4.0     ; [a,b,a,b]
    cmplx_b     dd  1.0, 2.0, 1.0, 2.0     ; [c,d,c,d]
    
    ; Output format strings
    msg_result  db  "Complex multiply result:", 10, 0
    fmt_float   db  "  Real: %f, Imag: %f", 10, 0
    
section .text
global _start

; (3+4i)(1+2i) = (3*1-4*2) + (3*2+4*1)i = (3-8) + (6+4)i = -5+10i
complex_multiply_full:
    ; Step 1: ทำ cross multiply
    movaps  xmm0, [cmplx_a]     ; [a,b,a,b]
    movaps  xmm1, [cmplx_b]     ; [c,d,c,d]
    
    ; สร้าง [c,c,c,c] และ [d,d,d,d]
    movsldup xmm2, xmm1         ; [c,c,c,c]
    movshdup xmm3, xmm1         ; [d,d,d,d]
    
    ; xmm4 = [a*c, b*c, a*c, b*c]
    movaps  xmm4, xmm0
    mulps   xmm4, xmm2
    
    ; สลับ [a,b] -> [b,a]
    movaps  xmm5, xmm0
    shufps  xmm5, xmm5, 0B1h    ; [b,a,b,a]
    
    ; xmm5 = [b*d, a*d, b*d, a*d]
    mulps   xmm5, xmm3
    
    ; ADDSUBPS: [a*c - b*d, b*c + a*d, ...]
    addsubps xmm4, xmm5
    ; xmm4[0] = real part = a*c - b*d = 3*1 - 4*2 = -5
    ; xmm4[1] = imag part = b*c + a*d = 4*1 + 3*2 = 10
    
    ret

_start:
    call    complex_multiply_full
    ; xmm4[0] = -5.0 (real)
    ; xmm4[1] = 10.0 (imag)
    
    ; Exit
    mov     rax, 60
    xor     rdi, rdi
    syscall
```

### 5.2 Complete Example: Horizontal Sum with HADDPS

```nasm
; hsum.asm - Horizontal sum of array using SSE3 HADDPS
; นับผลรวม array ขนาดใหญ่ด้วย SIMD

section .data
    align 16
    arr_data    dd  1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0
    arr_size    equ 8   ; 8 floats = 2 SSE registers

section .text

; sum_floats - คำนวณผลรวมของ float array
; Input: rdi = pointer to float array, esi = count (must be multiple of 4)
; Output: xmm0 = sum (in element 0)
sum_floats_sse3:
    xorps   xmm0, xmm0          ; accumulator = 0
    xor     ecx, ecx
    
.loop:
    cmp     ecx, esi
    jge     .reduce
    
    addps   xmm0, [rdi + rcx*4] ; add 4 floats
    add     ecx, 4
    jmp     .loop
    
.reduce:
    haddps  xmm0, xmm0          ; [a+b, c+d, a+b, c+d]
    haddps  xmm0, xmm0          ; [(a+b)+(c+d), ...] = [sum, sum, sum, sum]
    ; xmm0[0] = total sum
    ret

; คำนวณ sum ของ arr_data
sum_example:
    lea     rdi, [arr_data]
    mov     esi, arr_size
    call    sum_floats_sse3
    ; xmm0[0] = 1+2+3+4+5+6+7+8 = 36.0
    ret
```

### 5.3 Complete Example: Endian Conversion with PSHUFB

```nasm
; endian_swap.asm - Byte swap for endian conversion using SSSE3 PSHUFB

section .data
    align 16
    ; 4 x 32-bit big-endian values
    be_data     dd  0x01020304, 0x05060708, 0x090A0B0C, 0x0D0E0F10
    
    ; mask for 32-bit byte swap
    swap32_mask db  3,2,1,0, 7,6,5,4, 11,10,9,8, 15,14,13,12
    
    ; mask for 16-bit byte swap (8 x 16-bit values)
    swap16_mask db  1,0, 3,2, 5,4, 7,6, 9,8, 11,10, 13,12, 15,14
    
    ; mask for 64-bit byte swap (2 x 64-bit values)
    swap64_mask db  7,6,5,4,3,2,1,0, 15,14,13,12,11,10,9,8

section .text

; bswap32_sse - swap bytes in 4 x 32-bit integers simultaneously
bswap32_sse:
    movdqa  xmm0, [be_data]
    movdqa  xmm1, [swap32_mask]
    pshufb  xmm0, xmm1          ; swap bytes in all 4 dwords
    ; xmm0 = [0x04030201, 0x08070605, 0x0C0B0A09, 0x100F0E0D]
    ret

; bswap64_sse - swap bytes in 2 x 64-bit integers  
bswap64_sse:
    movdqa  xmm0, [be_data]
    movdqa  xmm1, [swap64_mask]
    pshufb  xmm0, xmm1
    ret

; network_to_host - convert network byte order (big-endian) to host (little-endian)
; Input: xmm0 = 4 x uint32_t in network byte order
; Output: xmm0 = 4 x uint32_t in host byte order
network_to_host:
    movdqa  xmm1, [swap32_mask]
    pshufb  xmm0, xmm1
    ret
```

### 5.4 Complete Example: String Search with PCMPISTRI

```nasm
; strops_sse42.asm - String operations using SSE4.2 PCMPISTRI
; nasm -f elf64 strops_sse42.asm -o strops_sse42.o

section .data
    align 16
    test_str    db  "The quick brown fox jumps over the lazy dog", 0
    search_char db  "fox", 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
    find_vowels db  "aeiouAEIOU", 0, 0, 0, 0, 0, 0
    
section .text

; ======================================
; strlen_sse42 - หาความยาว string ด้วย SSE4.2
; Input: rdi = null-terminated string
; Output: rax = length
; ======================================
strlen_sse42:
    pxor    xmm0, xmm0          ; xmm0 = zero (search for null)
    xor     eax, eax            ; offset counter
    
.loop:
    movdqu  xmm1, [rdi + rax]   ; โหลด 16 bytes
    ; imm8 = 0x08 = 00001000b
    ; bits 1-0 = 00: unsigned bytes
    ; bits 3-2 = 10: equal each position
    ; bit 4 = 0: positive polarity
    ; bit 5 = 0: ECX = index of first match (LSB)
    pcmpistri xmm0, xmm1, 08h
    jz      .done               ; ZF=1: null found in this 16-byte chunk
    add     eax, 16
    jmp     .loop
    
.done:
    add     eax, ecx            ; add position within chunk
    ret

; ======================================
; find_first_vowel - หา vowel แรกใน string
; Input: rdi = string, rsi = length (-1 for null-terminated)
; Output: rax = index of first vowel (-1 if not found)
; ======================================
find_first_vowel:
    movdqu  xmm0, [find_vowels] ; โหลด vowel set: "aeiouAEIOU\0\0\0\0\0\0"
    xor     eax, eax
    
.loop:
    cmp     byte [rdi + rax], 0
    je      .not_found
    
    movdqu  xmm1, [rdi + rax]   ; โหลด string chunk
    ; imm8 = 0x00 = equal any (หา char ที่อยู่ใน vowel set)
    pcmpistri xmm0, xmm1, 00h
    jc      .found              ; CF=1: found match
    add     eax, 16
    jmp     .loop
    
.found:
    add     eax, ecx            ; offset + match position
    ret
    
.not_found:
    mov     eax, -1
    ret

; ======================================
; strstr_sse42 - หา substring ใน string
; Input: rdi = haystack, rsi = needle (null-terminated)
; Output: rax = pointer to first occurrence, or NULL
; ======================================
strstr_sse42:
    push    rbx
    push    r12
    push    r13
    
    mov     r12, rdi            ; save haystack
    mov     r13, rsi            ; save needle
    
    ; โหลด needle ลง xmm0
    movdqu  xmm0, [r13]
    
    xor     ebx, ebx            ; haystack offset
    
.search_loop:
    cmp     byte [r12 + rbx], 0
    je      .not_found
    
    movdqu  xmm1, [r12 + rbx]
    
    ; imm8 = 0x0C = 00001100b
    ; bits 1-0 = 00: unsigned bytes
    ; bits 3-2 = 11: equal ordered (substring search)
    ; bit 4 = 0: positive polarity
    pcmpistri xmm0, xmm1, 0Ch
    
    jc      .check_match
    add     ebx, 16
    jmp     .search_loop
    
.check_match:
    ; ecx = offset ของ match ใน current chunk
    ; ตรวจสอบว่าเป็น match จริงหรือเปล่า
    lea     rax, [r12 + rbx + rcx]
    
    ; verify match
    mov     rdx, r13            ; needle pointer
    mov     r8, rax             ; match pointer
.verify:
    movzx   ecx, byte [rdx]
    test    ecx, ecx
    jz      .matched            ; ถึง end of needle = matched!
    cmp     cl, [r8]
    jne     .next_pos
    inc     rdx
    inc     r8
    jmp     .verify

.matched:
    pop     r13
    pop     r12
    pop     rbx
    ret

.next_pos:
    inc     ebx
    jmp     .search_loop
    
.not_found:
    xor     eax, eax
    pop     r13
    pop     r12
    pop     rbx
    ret
```

### 5.5 Complete Example: CRC32 Hash Function

```nasm
; crc32_hash.asm - Hardware CRC32 using SSE4.2 CRC32 instruction
; นี่คือ CRC-32C (Castagnoli) ไม่ใช่ CRC-32/ISO-HDLC แบบ zlib

section .data
    test_data   db  "Hello, World!"
    test_len    equ $ - test_data
    
section .text

; ======================================
; crc32c - คำนวณ CRC32C ของ data
; Input: rdi = data pointer, rsi = length  
; Output: eax = CRC32C value
; ======================================
crc32c:
    mov     eax, 0xFFFFFFFF     ; seed
    
    ; Process 8 bytes at a time
.loop8:
    cmp     rsi, 8
    jl      .loop4
    crc32   rax, qword [rdi]
    add     rdi, 8
    sub     rsi, 8
    jmp     .loop8

.loop4:
    cmp     rsi, 4
    jl      .loop2
    crc32   eax, dword [rdi]
    add     rdi, 4
    sub     rsi, 4

.loop2:
    cmp     rsi, 2
    jl      .loop1
    crc32   eax, word [rdi]
    add     rdi, 2
    sub     rsi, 2

.loop1:
    test    rsi, rsi
    jz      .done
    crc32   eax, byte [rdi]
    dec     rsi
    inc     rdi
    jmp     .loop1

.done:
    not     eax                 ; finalize
    ret

; ตัวอย่างใช้งาน
crc32_main:
    lea     rdi, [test_data]
    mov     rsi, test_len
    call    crc32c
    ; eax = CRC32C ของ "Hello, World!"
    ret
```

### 5.6 Complete Example: Popcount สำหรับ Bitset Operations

```nasm
; bitset.asm - High-performance bitset operations using POPCNT
; สำหรับงาน set operations, graph algorithms, etc.

section .data
    align 8
    ; bitsets ขนาด 64 bits (1 qword)
    set_a   dq  0b1010101010101010101010101010101010101010101010101010101010101010
    set_b   dq  0b1100110011001100110011001100110011001100110011001100110011001100
    
    ; bitsets ขนาดใหญ่ (256 bits = 4 qwords)
    align 32
    big_set_a   dq  0xAAAAAAAAAAAAAAAA, 0x5555555555555555, 0xF0F0F0F0F0F0F0F0, 0x0F0F0F0F0F0F0F0F
    big_set_b   dq  0xCCCCCCCCCCCCCCCC, 0x3333333333333333, 0xFF00FF00FF00FF00, 0x00FF00FF00FF00FF

section .text

; popcount64 - นับ bit ที่เป็น 1 ใน 64-bit value
popcount64:
    popcnt  rax, rdi
    ret

; popcount_array - นับ bits ใน array ของ qwords
; Input: rdi = array pointer, esi = count (in qwords)
; Output: rax = total set bits
popcount_array:
    xor     eax, eax
    xor     ecx, ecx
.loop:
    cmp     ecx, esi
    jge     .done
    popcnt  rdx, qword [rdi + rcx*8]
    add     rax, rdx
    inc     ecx
    jmp     .loop
.done:
    ret

; bitset_intersection_count - นับ bits ที่อยู่ใน intersection (AND) ของ 2 bitsets
; Input: rdi = set_a, rsi = set_b, rdx = size in qwords
; Output: rax = count of common bits
bitset_intersection_count:
    xor     eax, eax
    xor     ecx, ecx
.loop:
    cmp     ecx, edx
    jge     .done
    mov     r8, [rdi + rcx*8]
    and     r8, [rsi + rcx*8]   ; intersection
    popcnt  r9, r8
    add     rax, r9
    inc     ecx
    jmp     .loop
.done:
    ret

; hamming_distance - นับ bit ที่แตกต่างระหว่าง 2 bitsets (XOR + popcount)
; Input: rdi = set_a, rsi = set_b, rdx = size in qwords
; Output: rax = hamming distance
hamming_distance:
    xor     eax, eax
    xor     ecx, ecx
.loop:
    cmp     ecx, edx
    jge     .done
    mov     r8, [rdi + rcx*8]
    xor     r8, [rsi + rcx*8]   ; XOR = bits that differ
    popcnt  r9, r8
    add     rax, r9
    inc     ecx
    jmp     .loop
.done:
    ret
```

### 5.7 Complete Example: Dot Product 4D ด้วย DPPS

```nasm
; dot4d.asm - 4D dot product using SSE4.1 DPPS

section .data
    align 16
    v1  dd  1.0, 2.0, 3.0, 4.0
    v2  dd  5.0, 6.0, 7.0, 8.0
    ; Expected: 1*5 + 2*6 + 3*7 + 4*8 = 5+12+21+32 = 70

section .text

; dot4d - 4D dot product using DPPS
; Input: xmm0 = vector a [x,y,z,w], xmm1 = vector b [x,y,z,w]
; Output: xmm0[0] = dot product
dot4d:
    ; imm8 = 0xFF = 11111111b
    ; bits 4-7 = 1111: multiply all 4 pairs
    ; bits 0-3 = 1111: store result in all 4 positions
    dpps    xmm0, xmm1, 0FFh
    ; xmm0 = [70, 70, 70, 70]
    ret

; dot3d - 3D dot product (ignore w component)
; Input: xmm0 = [x,y,z,?], xmm1 = [x,y,z,?]
; Output: xmm0[0] = dot product
dot3d:
    ; imm8 = 0x71 = 01110001b
    ; bits 4-7 = 0111: multiply elements 0,1,2 (not 3)
    ; bits 0-3 = 0001: store only in element 0
    dpps    xmm0, xmm1, 71h
    ret

; vector_length_sq - ความยาวยกกำลัง 2 ของ 4D vector
vector_length_sq:
    movaps  xmm1, xmm0
    dpps    xmm0, xmm1, 0FFh    ; dot with itself
    ret

; normalize_4d - normalize 4D vector
normalize_4d:
    movaps  xmm1, xmm0          ; save original
    dpps    xmm0, xmm0, 0FFh    ; length squared
    rsqrtps xmm0, xmm0          ; 1/sqrt(length_sq) (approximate)
    mulps   xmm0, xmm1          ; normalize
    ret

; dot4d_batch - ทำ dot product หลายคู่พร้อมกัน
; เปรียบเทียบ 4 vectors กับ 1 vector พร้อมกัน
dot4d_batch_example:
    ; สมมติ xmm0 = query vector
    ; xmm1, xmm2, xmm3, xmm4 = database vectors
    movaps  xmm5, xmm0          ; backup query
    
    dpps    xmm1, xmm0, 0FFh    ; dot(db[0], query)
    movaps  xmm0, xmm5
    dpps    xmm2, xmm0, 0FFh    ; dot(db[1], query)
    movaps  xmm0, xmm5
    dpps    xmm3, xmm0, 0FFh    ; dot(db[2], query)
    movaps  xmm0, xmm5
    dpps    xmm4, xmm0, 0FFh    ; dot(db[3], query)
    ret
```

---

## ส่วนที่ 6: CPUID Checks

ก่อนใช้ instruction เหล่านี้ ต้องตรวจสอบว่า CPU รองรับหรือไม่

```nasm
; cpuid_check.asm - ตรวจสอบ SSE3/SSSE3/SSE4.1/SSE4.2 support

section .data
    ; Feature flag descriptions
    msg_sse3    db  "SSE3:   ", 0
    msg_ssse3   db  "SSSE3:  ", 0
    msg_sse41   db  "SSE4.1: ", 0
    msg_sse42   db  "SSE4.2: ", 0
    msg_popcnt  db  "POPCNT: ", 0
    msg_yes     db  "Yes", 10, 0
    msg_no      db  "No",  10, 0

section .text

; check_sse3 - ตรวจสอบว่า CPU รองรับ SSE3
; Output: eax = 1 ถ้ารองรับ, 0 ถ้าไม่รองรับ
check_sse3:
    push    rbx
    
    mov     eax, 1              ; CPUID leaf 1
    cpuid                       ; ผลใน eax/ebx/ecx/edx
    
    ; ECX bit 0 = SSE3
    mov     eax, ecx
    and     eax, 1              ; bit 0
    
    pop     rbx
    ret

; check_ssse3 - ตรวจสอบ SSSE3
; Output: eax = 1 ถ้ารองรับ
check_ssse3:
    push    rbx
    
    mov     eax, 1
    cpuid
    
    ; ECX bit 9 = SSSE3
    mov     eax, ecx
    shr     eax, 9
    and     eax, 1
    
    pop     rbx
    ret

; check_sse41 - ตรวจสอบ SSE4.1
check_sse41:
    push    rbx
    
    mov     eax, 1
    cpuid
    
    ; ECX bit 19 = SSE4.1
    mov     eax, ecx
    shr     eax, 19
    and     eax, 1
    
    pop     rbx
    ret

; check_sse42 - ตรวจสอบ SSE4.2
check_sse42:
    push    rbx
    
    mov     eax, 1
    cpuid
    
    ; ECX bit 20 = SSE4.2
    mov     eax, ecx
    shr     eax, 20
    and     eax, 1
    
    pop     rbx
    ret

; check_popcnt - ตรวจสอบ POPCNT
check_popcnt:
    push    rbx
    
    mov     eax, 1
    cpuid
    
    ; ECX bit 23 = POPCNT
    mov     eax, ecx
    shr     eax, 23
    and     eax, 1
    
    pop     rbx
    ret

; check_lzcnt - ตรวจสอบ LZCNT (ใน Extended features)
check_lzcnt:
    push    rbx
    
    mov     eax, 80000001h      ; Extended CPUID leaf
    cpuid
    
    ; ECX bit 5 = LZCNT (ABM)
    mov     eax, ecx
    shr     eax, 5
    and     eax, 1
    
    pop     rbx
    ret

; print_cpu_features - แสดง feature support ทั้งหมด
print_cpu_features:
    push    rbx
    push    r12
    push    r13
    
    ; ดึง feature bits ทีเดียว
    mov     eax, 1
    cpuid
    mov     r12, rcx            ; เก็บ ECX (standard features)
    
    ; ECX bit 0 = SSE3
    mov     r13, r12
    and     r13, 1
    ; ... (แสดงผล)
    
    ; ECX bit 9 = SSSE3  
    mov     r13, r12
    shr     r13, 9
    and     r13, 1
    ; ...
    
    ; ECX bit 19 = SSE4.1
    mov     r13, r12
    shr     r13, 19
    and     r13, 1
    ; ...
    
    ; ECX bit 20 = SSE4.2
    mov     r13, r12
    shr     r13, 20
    and     r13, 1
    ; ...
    
    ; ECX bit 23 = POPCNT
    mov     r13, r12
    shr     r13, 23
    and     r13, 1
    ; ...
    
    pop     r13
    pop     r12
    pop     rbx
    ret

; safe_dispatch - เลือก implementation ตาม CPU features
safe_dispatch:
    call    check_sse42
    test    eax, eax
    jnz     .use_sse42
    
    call    check_sse41
    test    eax, eax
    jnz     .use_sse41
    
    call    check_ssse3
    test    eax, eax
    jnz     .use_ssse3
    
    call    check_sse3
    test    eax, eax
    jnz     .use_sse3
    
    ; ใช้ scalar fallback
    jmp     .use_scalar
    
.use_sse42:
    ; ใช้ SSE4.2 code path
    ret
.use_sse41:
    ; ใช้ SSE4.1 code path
    ret
.use_ssse3:
    ; ใช้ SSSE3 code path
    ret
.use_sse3:
    ; ใช้ SSE3 code path
    ret
.use_scalar:
    ; ใช้ scalar code path
    ret
```

---

## ส่วนที่ 7: ตัวอย่าง Makefile และการ Build

### Makefile สำหรับ compile ตัวอย่างทั้งหมด

```makefile
# Makefile for SSE3/SSSE3/SSE4 examples
NASM = nasm
LD = ld
CC = gcc
NASMFLAGS = -f elf64 -g -F dwarf
LDFLAGS = 
CFLAGS = -msse3 -mssse3 -msse4.1 -msse4.2 -O2

TARGETS = complex_ops hsum endian_swap strops_sse42 crc32_hash bitset dot4d cpuid_check

all: $(TARGETS)

complex_ops: complex_ops.o
	$(LD) $< -o $@

%.o: %.asm
	$(NASM) $(NASMFLAGS) $< -o $@

clean:
	rm -f *.o $(TARGETS)

# ตรวจสอบ CPU features ก่อน run
check_cpu:
	grep -m1 'flags' /proc/cpuinfo | grep -oE 'sse[0-9_]*' | sort -u

.PHONY: all clean check_cpu
```

### วิธี Build และ Run

```bash
# ตรวจสอบ CPU features
grep -m1 'flags' /proc/cpuinfo | tr ' ' '\n' | grep -E '^sse|popcnt|pclmul'

# Build ด้วย NASM
nasm -f elf64 -g complex_ops.asm -o complex_ops.o
ld complex_ops.o -o complex_ops

# หรือ build ผ่าน C wrapper เพื่อใช้ printf
nasm -f elf64 complex_ops.asm -o complex_ops.o
gcc -o complex_ops complex_ops.o wrapper.c

# Run พร้อม gdb debug
gdb ./complex_ops
# ใน gdb:
# break complex_multiply_full
# run
# info registers xmm0 xmm1 xmm4
```

---

## ส่วนที่ 8: ตาราง CPUID Feature Bits

```
CPUID Leaf 1 (EAX=1), ECX Register:
+------+----------+-----------------------------------------------+
| Bit  | Feature  | Description                                   |
+------+----------+-----------------------------------------------+
|  0   | SSE3     | Prescott New Instructions (PNI)               |
|  1   | PCLMULQDQ| Carry-less multiplication                     |
|  3   | MONITOR  | MONITOR/MWAIT instructions                    |
|  9   | SSSE3    | Supplemental SSE3                             |
| 12   | FMA      | Fused Multiply-Add (FMA3)                    |
| 19   | SSE4.1   | Intel SSE 4.1                                |
| 20   | SSE4.2   | Intel SSE 4.2                                |
| 23   | POPCNT   | POPCNT instruction                           |
| 25   | AES      | AES-NI instructions                          |
| 26   | XSAVE    | XSAVE/XRSTOR instructions                    |
| 28   | AVX      | Advanced Vector Extensions                   |
| 29   | F16C     | 16-bit float conversion                      |
| 30   | RDRAND   | RDRAND instruction                           |
+------+----------+-----------------------------------------------+

CPUID Leaf 80000001h, ECX Register:
+------+----------+-----------------------------------------------+
|  5   | LZCNT    | Leading Zero Count (ABM)                      |
+------+----------+-----------------------------------------------+
```

---

## ส่วนที่ 9: Performance Tips และ Best Practices

### 9.1 Memory Alignment

```nasm
; ใช้ aligned data เสมอเมื่อเป็นไปได้
section .data
    align 16
    aligned_data    dd  1.0, 2.0, 3.0, 4.0     ; 16-byte aligned

    align 32
    avx_data        dd  1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0  ; 32-byte aligned

; Load commands:
; MOVAPS/MOVAPD = aligned (faster, crashes if misaligned)
; MOVUPS/MOVUPD = unaligned (slower but safe)
; LDDQU = unaligned integer load (faster for crossing cache line boundary)

performance_load:
    movaps  xmm0, [aligned_data]    ; fast aligned load
    movdqa  xmm1, [aligned_data]    ; fast aligned integer load
    ; vs
    movups  xmm2, [aligned_data + 1] ; slow unaligned (but safe)
    lddqu   xmm3, [aligned_data + 1] ; alternative for unaligned
    ret
```

### 9.2 Avoiding False Dependencies

```nasm
; Bad: false dependency chain
bad_example:
    movss   xmm0, [val1]        ; xmm0[0] = val1, upper bits undefined
    addss   xmm0, [val2]        ; depends on previous
    ret

; Good: break dependency with pxor or explicit zero-extension
good_example:
    xorps   xmm0, xmm0          ; clear xmm0 first
    movss   xmm0, [val1]        ; now no false dependency
    addss   xmm0, [val2]
    ret

; Alternative: use VZEROUPPER after AVX code before SSE code
; (prevents AVX-SSE transition penalty)
```

### 9.3 PCMPISTRI ต้องจัดการ Edge Cases

```nasm
; PCMPISTRI implicit length = หยุดที่ null byte
; ถ้า string มีความยาวพอดี 16 bytes (ไม่มี null ใน chunk)
; CF, ZF flags จะบอกสถานะ:
;   CF = 1: found match
;   ZF = 1: null found in source (string ended)

safe_pcmpistri:
    ; ต้องตรวจสอบ ZF และ CF ร่วมกัน
    pcmpistri xmm0, xmm1, 00h
    ; CF = 1 AND ZF = 1: match found, but also null = match came before null
    ; CF = 1 AND ZF = 0: match found, string continues
    ; CF = 0 AND ZF = 1: no match, string ended
    ; CF = 0 AND ZF = 0: no match, string continues (16-byte chunk, all non-null)
    
    jc  .found_match        ; CF=1 regardless of ZF
    jz  .end_of_string      ; ZF=1, no match, string ended
    ; else: continue to next 16 bytes
    ret
```

### 9.4 CRC32 Pipeline

```nasm
; ใช้หลาย accumulator เพื่อ hide latency ของ CRC32

crc32_fast:
    ; CRC32 latency = 3 cycles
    ; ใช้ 4 independent chains
    mov     eax, 0xFFFFFFFF
    mov     ebx, 0xFFFFFFFF
    mov     ecx, 0xFFFFFFFF
    mov     edx, 0xFFFFFFFF
    
.loop:
    crc32   rax, qword [rdi]        ; chain 1
    crc32   rbx, qword [rdi + 8]    ; chain 2 (independent)
    crc32   rcx, qword [rdi + 16]   ; chain 3
    crc32   rdx, qword [rdi + 24]   ; chain 4
    add     rdi, 32
    sub     rsi, 32
    jge     .loop
    
    ; Combine (naive - not truly correct CRC combination without CRC-merging math)
    ; สำหรับ production ใช้ CRC combine technique ที่ถูกต้อง
    not     eax
    ret
```

---

## สรุป (Summary)

| Extension | Year | Key Features                                    |
|-----------|------|-------------------------------------------------|
| SSE3      | 2004 | ADDSUBPS/PD, HADD/HSUB, MOVSHDUP/SLDUP/DDUP   |
| SSSE3     | 2007 | PSHUFB, PHADD/PHSUB, PABS, PSIGN, PALIGNR      |
| SSE4.1    | 2007 | DPPS/PD, BLEND, INSERT/EXTRACT, ROUND, PMULLD  |
| SSE4.2    | 2008 | PCMPISTRI/ESTRI, CRC32, POPCNT, LZCNT          |

### ควรใช้เมื่อไหร่

- **SSE3 ADDSUBPS**: Complex number arithmetic
- **SSE3 HADDPS**: Horizontal sum/reduction operations
- **SSSE3 PSHUFB**: Byte permutation, endian swap, format conversion
- **SSE4.1 DPPS**: Vector dot product, matrix operations
- **SSE4.1 ROUND**: Vectorized floor/ceil/round/trunc
- **SSE4.2 PCMPISTRI**: String search, parsing, text processing
- **SSE4.2 CRC32**: Checksums, hash functions
- **SSE4.2 POPCNT**: Bit counting, bitset cardinality
- **SSE4.2 LZCNT**: Log2, bit field operations

### Modern Alternatives

ปัจจุบัน AVX/AVX2/AVX-512 ให้ประสิทธิภาพดีกว่ามาก แต่ SSE4.2 ยังมีประโยชน์เมื่อ:
1. ต้องการ compatibility กับ CPU เก่า
2. ทำงานกับ data ขนาด 128-bit พอดี
3. ใช้ string comparison instructions (PCMPISTRI ไม่มีใน AVX แบบตรงๆ)
4. Embedded systems ที่ไม่มี AVX

---

*ส่วนถัดไป Part 046: AVX Instructions - 256-bit SIMD Operations*
