# Part 042: MMX Instructions - คำสั่ง MMX สำหรับการประมวลผลแบบขนาน

## บทนำ (Introduction)

MMX (Matrix Math eXtensions หรือ MultiMedia eXtensions) เป็นเทคโนโลยี SIMD (Single Instruction, Multiple Data) รุ่นแรกของ Intel ที่เปิดตัวในปี 1996 พร้อมกับ Pentium MMX โดยช่วยให้สามารถประมวลผลข้อมูลหลายชุดพร้อมกันในคำสั่งเดียว ซึ่งเหมาะอย่างยิ่งสำหรับงาน multimedia เช่น การประมวลผลภาพ เสียง และวิดีโอ

**ประโยชน์หลักของ MMX:**
- ประมวลผล 8 bytes, 4 words, หรือ 2 doublewords พร้อมกัน
- เพิ่มประสิทธิภาพงาน multimedia ได้ 2-8 เท่า
- รองรับ saturating arithmetic (ป้องกัน overflow)
- ใช้ registers เดิม (x87 FPU) ไม่ต้องเพิ่ม hardware ใหม่มาก

---

## 1. MMX Registers (MM0-MM7)

### 1.1 โครงสร้าง MMX Registers

MMX ใช้ registers ที่มีชื่อว่า MM0 ถึง MM7 ซึ่งแต่ละ register มีขนาด **64 bits** และเป็นการ alias (ชี้ไปที่เดิม) กับ x87 FPU stack registers ST(0) ถึง ST(7)

```
MMX Register Layout (64-bit):
┌────────────────────────────────────────────────────────────────┐
│                        MM0 (64 bits)                           │
├────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┤
│ byte 7 │ byte 6 │ byte 5 │ byte 4 │ byte 3 │ byte 2 │ byte 1 │ byte 0 │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
  63:56    55:48    47:40    39:32    31:24    23:16    15:8     7:0

Packed Byte (8×8):   B7|B6|B5|B4|B3|B2|B1|B0   (8 bytes)
Packed Word (4×16):  W3    |W2    |W1    |W0     (4 words)
Packed DWord (2×32): D1          |D0            (2 dwords)
```

### 1.2 การ Alias กับ x87 FPU

```nasm
; MMX registers ถูก map ไปที่ x87 FPU mantissa bits
; MM0 -> ST(0) mantissa (bits 63:0)
; MM1 -> ST(1) mantissa
; ...
; MM7 -> ST(7) mantissa

; ข้อควรระวัง: ไม่ควรใช้ MMX และ x87 FPU พร้อมกัน!
; ถ้าใช้ MMX แล้วต้องการใช้ FPU ต้องเรียก EMMS ก่อน
```

### 1.3 ตัวอย่างการโหลดข้อมูลเข้า MMX Register

```nasm
section .data
    byte_data   db  1, 2, 3, 4, 5, 6, 7, 8    ; 8 bytes
    word_data   dw  100, 200, 300, 400          ; 4 words
    dword_data  dd  1000, 2000                  ; 2 dwords
    qword_data  dq  0x0807060504030201          ; 1 qword (64-bit)

section .text
    global _start

_start:
    ; โหลดข้อมูล 64 บิต (8 bytes) เข้า MM0
    movq    mm0, [byte_data]      ; MM0 = {8,7,6,5,4,3,2,1} (little-endian)
    
    ; โหลด word data เข้า MM1
    movq    mm1, [word_data]      ; MM1 = {400,300,200,100}
    
    ; โหลด dword data เข้า MM2
    movq    mm2, [dword_data]     ; MM2 = {2000,1000}
    
    ; โหลด qword data เข้า MM3
    movq    mm3, [qword_data]     ; MM3 = 0x0807060504030201
    
    ; คัดลอก MM register
    movq    mm4, mm0              ; MM4 = MM0 (คัดลอก)
    
    ; เก็บผลลัพธ์กลับไปที่ memory
    movq    [byte_data], mm0      ; เขียนกลับ
    
    ; สำคัญ: ต้องเรียก EMMS ก่อนใช้ FPU
    emms                          ; รีเซ็ต MMX state
```

---

## 2. EMMS Instruction - รีเซ็ต MMX State

### 2.1 ความสำคัญของ EMMS

```nasm
; EMMS = Empty MMX State
; หลังจากใช้ MMX instructions ต้องเรียก EMMS
; เพื่อบอก CPU ว่า FPU stack ว่างและพร้อมใช้งาน

; ทำไมต้องใช้ EMMS?
; เพราะ MMX กับ FPU ใช้ registers เดิมกัน
; ถ้าไม่เรียก EMMS แล้วพยายามใช้ FPU จะได้ผลลัพธ์ผิด!

section .text
example_emms:
    ; ทำงาน MMX
    movq    mm0, [data1]
    movq    mm1, [data2]
    paddb   mm0, mm1              ; บวก packed bytes
    movq    [result], mm0
    
    emms                          ; รีเซ็ต: ต้องเรียกก่อนใช้ FPU
    
    ; หลังจาก emms แล้วใช้ FPU ได้ปกติ
    fld     dword [float_val]     ; โหลด float value
    fadd    dword [float_val2]    ; บวก float
    fstp    dword [float_result]  ; เก็บผล

; Pattern ที่ถูกต้อง:
;   1. ทำงาน MMX ทั้งหมด
;   2. เรียก EMMS
;   3. ทำงาน FPU
; อย่าสลับไปมาระหว่าง MMX และ FPU โดยไม่เรียก EMMS!
```

### 2.2 Performance Impact ของ EMMS

```nasm
; EMMS มี latency สูง (~20-100 cycles บน CPU เก่า)
; ควรเรียกเมื่อจำเป็นเท่านั้น

; ไม่ดี (เรียก EMMS บ่อยเกินไป):
bad_pattern:
    movq    mm0, [ptr]
    paddb   mm0, mm1
    emms                ; ← penalty สูง
    movq    [ptr], mm0
    
    movq    mm0, [ptr2] ; เริ่ม MMX ใหม่อีก
    paddb   mm0, mm2
    emms                ; ← penalty สูงอีก

; ดี (เรียก EMMS ครั้งเดียวตอนจบ):
good_pattern:
    movq    mm0, [ptr]
    paddb   mm0, mm1
    movq    [ptr], mm0
    
    movq    mm0, [ptr2]
    paddb   mm0, mm2
    movq    [ptr2], mm0
    
    emms                ; ← เรียกครั้งเดียวตอนจบ
```

---

## 3. Packed Integer Types

### 3.1 การตีความข้อมูลในรูปแบบต่างๆ

```nasm
; MMX register 64-bit สามารถตีความเป็น:
; - Packed Byte (PB):    8 × 8-bit  integers
; - Packed Word (PW):    4 × 16-bit integers  
; - Packed DWord (PD):   2 × 32-bit integers
; - QWord (Q):           1 × 64-bit integer

; ตัวอย่างข้อมูล 0x0102030405060708:
; Packed Bytes:  01 | 02 | 03 | 04 | 05 | 06 | 07 | 08
; Packed Words:  0102 | 0304 | 0506 | 0708
; Packed DWords: 01020304 | 05060708

; การ interpret ขึ้นอยู่กับคำสั่งที่ใช้:
; PADDB  → ตีความเป็น 8 bytes และบวกแต่ละ byte
; PADDW  → ตีความเป็น 4 words และบวกแต่ละ word
; PADDD  → ตีความเป็น 2 dwords และบวกแต่ละ dword
```

---

## 4. PADD - Packed Add Instructions

### 4.1 PADDB - Packed Add Bytes

```nasm
; PADDB dst, src
; บวก 8 pairs of bytes พร้อมกัน (wrap-around)
; dst[i] = dst[i] + src[i]  สำหรับ i = 0..7

section .data
    a_bytes     db  10, 20, 30, 40, 50, 60, 70, 80
    b_bytes     db   5, 10, 15, 20, 25, 30, 35, 40
    result_b    db   8 dup(0)

section .text
test_paddb:
    movq    mm0, [a_bytes]        ; MM0 = {80,70,60,50,40,30,20,10}
    movq    mm1, [b_bytes]        ; MM1 = {40,35,30,25,20,15,10,5}
    paddb   mm0, mm1              ; MM0 = {120,105,90,75,60,45,30,15}
    movq    [result_b], mm0       ; เก็บผล
    ; ผล: 10+5=15, 20+10=30, 30+15=45, ...
    
    ; Overflow wrap-around (unsigned):
    ; 200 + 100 = 300 → 300 mod 256 = 44 (ไม่ saturate)
    ret
```

### 4.2 PADDW - Packed Add Words

```nasm
; PADDW dst, src
; บวก 4 pairs of 16-bit words พร้อมกัน

section .data
    a_words     dw  1000, 2000, 3000, 4000
    b_words     dw   500,  500,  500,  500
    result_w    dw   4 dup(0)

section .text
test_paddw:
    movq    mm0, [a_words]        ; MM0 = {4000,3000,2000,1000}
    movq    mm1, [b_words]        ; MM1 = {500,500,500,500}
    paddw   mm0, mm1              ; MM0 = {4500,3500,2500,1500}
    movq    [result_w], mm0       ; เก็บผล
    ret
```

### 4.3 PADDD - Packed Add Doublewords

```nasm
; PADDD dst, src
; บวก 2 pairs of 32-bit dwords พร้อมกัน

section .data
    a_dwords    dd  100000, 200000
    b_dwords    dd   50000,  75000
    result_d    dd   2 dup(0)

section .text
test_paddd:
    movq    mm0, [a_dwords]       ; MM0 = {200000, 100000}
    movq    mm1, [b_dwords]       ; MM1 = {75000, 50000}
    paddd   mm0, mm1              ; MM0 = {275000, 150000}
    movq    [result_d], mm0       ; เก็บผล
    ret
```

---

## 5. PADDS - Saturating Add Instructions

### 5.1 ความแตกต่างระหว่าง Wrap-around และ Saturating

```nasm
; Wrap-around (PADDB):
; 200 + 100 = 300 → 44 (overflow, wrap around)
; อาจทำให้สีกลายเป็นสีผิดในภาพ!

; Saturating (PADDSB/PADDSW):
; 200 + 100 = 300 → 255 (clamped ที่ max)
; สีคงที่ที่ขาวสุด ไม่กลับไปเป็นสีอื่น

; Saturating สำหรับ unsigned byte: clamp ที่ 0..255
; Saturating สำหรับ signed byte: clamp ที่ -128..127

section .text
compare_add:
    ; ทดสอบ overflow
    mov     eax, 200
    movd    mm0, eax              ; MM0 = 200 ใน byte 0
    mov     eax, 100
    movd    mm1, eax              ; MM1 = 100 ใน byte 0
    
    movq    mm2, mm0
    paddb   mm2, mm1              ; wrap: 300 mod 256 = 44 ← ผิด!
    
    movq    mm3, mm0
    paddusb mm3, mm1              ; saturate: clamp ที่ 255 ← ถูก สำหรับ pixel
    
    ret
```

### 5.2 PADDSB - Packed Add Signed Bytes with Saturation

```nasm
; PADDSB dst, src
; บวก bytes แบบ signed, saturate ที่ -128..127

section .data
    signed_a    db  100, -100, 50, -50, 70, -70, 120, -120
    signed_b    db   50,  -50, 90, -90, 80, -80,  20,  -20
    result_sb   db   8 dup(0)

section .text
test_paddsb:
    movq    mm0, [signed_a]       ; โหลด signed bytes
    movq    mm1, [signed_b]       ; โหลด signed bytes
    paddsb  mm0, mm1              ; บวกแบบ saturating signed
    ; 100+50=150 → 127 (saturated)
    ; -100+(-50)=-150 → -128 (saturated)
    ; 50+90=140 → 127 (saturated)
    movq    [result_sb], mm0
    ret
```

### 5.3 PADDSW - Packed Add Signed Words with Saturation

```nasm
; PADDSW dst, src
; บวก words แบบ signed, saturate ที่ -32768..32767

section .data
    sw_a    dw  30000, -30000, 10000, -10000
    sw_b    dw  10000, -10000, 25000, -25000
    res_sw  dw  4 dup(0)

section .text
test_paddsw:
    movq    mm0, [sw_a]           ; โหลด signed words
    movq    mm1, [sw_b]           ; โหลด signed words
    paddsw  mm0, mm1              ; บวกแบบ saturating signed word
    ; 30000+10000=40000 → 32767 (saturated)
    ; -30000+(-10000)=-40000 → -32768 (saturated)
    movq    [res_sw], mm0
    ret
```

### 5.4 PADDUSB/PADDUSW - Packed Add Unsigned with Saturation

```nasm
; PADDUSB dst, src  ; unsigned byte saturation: clamp ที่ 0..255
; PADDUSW dst, src  ; unsigned word saturation: clamp ที่ 0..65535

section .data
    ub_a    db  200, 100, 50, 250, 10, 5, 128, 200
    ub_b    db  100,  50, 30,  10, 20, 8, 200, 100

section .text
test_paddusb:
    movq    mm0, [ub_a]           ; โหลด unsigned bytes
    movq    mm1, [ub_b]           ; โหลด unsigned bytes
    paddusb mm0, mm1              ; unsigned saturating add
    ; 200+100=300 → 255 (saturated at 255)
    ; 250+10=260 → 255 (saturated at 255)
    ; 50+30=80 → 80 (no saturation)
    ret
```

---

## 6. PSUB - Packed Subtract Instructions

### 6.1 PSUBB/PSUBW/PSUBD

```nasm
; PSUBB dst, src  ; ลบ 8 bytes
; PSUBW dst, src  ; ลบ 4 words
; PSUBD dst, src  ; ลบ 2 dwords

section .data
    sub_a_b     db  100, 90, 80, 70, 60, 50, 40, 30
    sub_b_b     db   10, 20, 30, 40, 50, 60, 70, 80

section .text
test_psub:
    ; PSUBB - subtract bytes (wrap-around)
    movq    mm0, [sub_a_b]        ; MM0 = {30,40,50,60,70,80,90,100}
    movq    mm1, [sub_b_b]        ; MM1 = {80,70,60,50,40,30,20,10}
    psubb   mm0, mm1              ; MM0 = {-50,-30,-10,10,30,50,70,90}
    ; 100-10=90, 90-20=70, 80-30=50, ...
    ; 30-80=-50 → 206 (wrap-around สำหรับ unsigned)
    
    ; PSUBW - subtract words
    movq    mm2, [word_a]
    movq    mm3, [word_b]
    psubw   mm2, mm3
    
    ; PSUBD - subtract dwords
    movq    mm4, [dword_a]
    movq    mm5, [dword_b]
    psubd   mm4, mm5
    
    emms
    ret
```

### 6.2 PSUBSB/PSUBSW - Saturating Subtract

```nasm
; PSUBSB  dst, src  ; signed byte saturating subtract
; PSUBSW  dst, src  ; signed word saturating subtract
; PSUBUSB dst, src  ; unsigned byte saturating subtract
; PSUBUSW dst, src  ; unsigned word saturating subtract

section .data
    ; Unsigned byte subtract - useful for image processing
    bright_pixel    db  100, 50, 30, 200, 150, 80, 60, 10
    dark_value      db  150, 60, 40,  50, 200, 90, 70, 20

section .text
test_psubusb:
    movq    mm0, [bright_pixel]
    movq    mm1, [dark_value]
    psubusb mm0, mm1              ; unsigned saturating subtract
    ; 100-150=-50 → 0 (saturated, ไม่เป็นลบ)
    ; 200-50=150 → 150 (ปกติ)
    ; 150-200=-50 → 0 (saturated)
    ; เหมาะสำหรับ "darkening" ภาพโดยไม่ wrap
    ret
```

---

## 7. PMUL - Packed Multiply Instructions

### 7.1 PMULLW - Packed Multiply Low Words

```nasm
; PMULLW dst, src
; คูณ 4 pairs of 16-bit words
; เก็บ lower 16 bits ของผลลัพธ์ 32-bit

; ตัวอย่าง: 300 × 400 = 120000
; 120000 ใน hex = 0x1D4C0
; lower 16 bits = 0xD4C0 = 54464

section .data
    mul_a_w     dw  100, 200, 300, 400
    mul_b_w     dw    5,  10,  15,  20
    result_mw   dw  4 dup(0)

section .text
test_pmullw:
    movq    mm0, [mul_a_w]        ; MM0 = {400,300,200,100}
    movq    mm1, [mul_b_w]        ; MM1 = {20,15,10,5}
    pmullw  mm0, mm1              ; MM0 = {8000,4500,2000,500}
    ; 100×5=500, 200×10=2000, 300×15=4500, 400×20=8000
    movq    [result_mw], mm0
    ret
```

### 7.2 PMULHW - Packed Multiply High Words

```nasm
; PMULHW dst, src
; คูณ 4 pairs of signed 16-bit words
; เก็บ upper 16 bits ของผลลัพธ์ 32-bit

; เหมาะสำหรับ fixed-point arithmetic
; เช่น เก็บ 1.0 เป็น 0x7FFF (32767)
; x × y (fixed point) = pmulhw(x, y) << 1

section .data
    fixed_a     dw  16384, 8192, -16384, -8192  ; 0.5, 0.25, -0.5, -0.25 (Q15)
    fixed_b     dw  16384, 16384, 16384, 16384   ; 0.5 (Q15)

section .text
test_pmulhw:
    movq    mm0, [fixed_a]        ; โหลด Q15 values
    movq    mm1, [fixed_b]        ; โหลด Q15 values
    pmulhw  mm0, mm1              ; คูณและเก็บ upper bits
    ; 16384 × 16384 = 268435456 (0x10000000)
    ; upper 16 bits = 0x1000 = 4096
    ; นี่คือ 0.5 × 0.5 = 0.25 ใน Q15 format (4096/16384 ≈ 0.25)
    ret
```

### 7.3 PMADDWD - Packed Multiply and Add

```nasm
; PMADDWD dst, src
; คูณ 4 pairs of 16-bit words แล้วบวกผล adjacent pairs
; ผล: 2 × 32-bit values

; dst[0:31]  = dst[0:15]×src[0:15]  + dst[16:31]×src[16:31]
; dst[32:63] = dst[32:47]×src[32:47] + dst[48:63]×src[48:63]

; เหมาะสำหรับ dot product และ FIR filter

section .data
    ; คำนวณ dot product: [1,2,3,4] · [5,6,7,8]
    ; = 1×5 + 2×6 + 3×7 + 4×8 = 5+12+21+32 = 70
    vec_a   dw  1, 2, 3, 4
    vec_b   dw  5, 6, 7, 8

section .text
test_pmaddwd:
    movq    mm0, [vec_a]          ; MM0 = {4,3,2,1}
    movq    mm1, [vec_b]          ; MM1 = {8,7,6,5}
    pmaddwd mm0, mm1              ; MM0 = {4×8+3×7, 2×6+1×5}
                                   ;     = {32+21, 12+5}
                                   ;     = {53, 17}
    ; ได้ 2 partial sums: 17 (สำหรับ pair 1,2) และ 53 (สำหรับ pair 3,4)
    ; รวมกันเพื่อ dot product สมบูรณ์:
    movq    mm2, mm0
    psrlq   mm2, 32               ; shift right 32 bits
    paddd   mm0, mm2              ; บวก 2 partial sums
    ; MM0[0:31] = 17 + 53 = 70 (dot product)
    ret
```

---

## 8. PAND/POR/PXOR/PANDN - Bitwise Operations

### 8.1 ภาพรวม Bitwise MMX

```nasm
; PAND  dst, src  ; bitwise AND (64-bit)
; POR   dst, src  ; bitwise OR  (64-bit)
; PXOR  dst, src  ; bitwise XOR (64-bit)
; PANDN dst, src  ; bitwise AND NOT: dst = (~dst) AND src
```

### 8.2 ตัวอย่างการใช้งาน Bitwise

```nasm
section .data
    mask_lower  dq  0x00FF00FF00FF00FF   ; เอาเฉพาะ even bytes
    mask_upper  dq  0xFF00FF00FF00FF00   ; เอาเฉพาะ odd bytes
    pixel_data  db  10, 200, 20, 180, 30, 170, 40, 160

section .text
bitwise_ops:
    movq    mm0, [pixel_data]     ; โหลด pixel data
    
    ; PAND - ใช้ mask เพื่อเลือกบาง channels
    movq    mm1, [mask_lower]
    pand    mm0, mm1              ; เก็บเฉพาะ even bytes (R, B channels)
    
    ; POR - รวม 2 images
    movq    mm2, [image1]
    movq    mm3, [image2]
    por     mm2, mm3              ; OR pixels (brighten)
    
    ; PXOR - XOR สำหรับ invert
    movq    mm4, [pixel_data]
    pcmpeqb mm5, mm5              ; MM5 = all 1s (0xFFFFFFFFFFFFFFFF)
    pxor    mm4, mm5              ; invert all bits (255-pixel)
    
    ; PANDN - NOT dst AND src (useful for masking)
    movq    mm6, [mask]           ; MM6 = mask
    movq    mm7, [data]           ; MM7 = data
    pandn   mm6, mm7              ; MM6 = (~mask) AND data
    ; เลือกบิตจาก data ที่ mask = 0
    
    emms
    ret
```

### 8.3 ตัวอย่างจริง: Alpha Blending ด้วย Bitwise

```nasm
; Alpha blending: result = (fg × alpha) + (bg × (255-alpha))
; Simple version using bitmasks

section .data
    ; สมมติ alpha = 128 (50%)
    fg_pixels   db  200, 100, 50, 255, 0, 0, 0, 0      ; foreground
    bg_pixels   db   50,  50, 50,  50, 0, 0, 0, 0       ; background
    
section .text
simple_blend:
    ; สำหรับ binary masking (alpha = 0 หรือ 255 เท่านั้น)
    movq    mm0, [fg_pixels]
    movq    mm1, [bg_pixels]
    movq    mm2, [alpha_mask]     ; 0xFF = ใช้ fg, 0x00 = ใช้ bg
    
    pand    mm0, mm2              ; fg × mask
    pandn   mm2, mm1              ; (~mask) × bg  (ใช้ mm2 ที่ยังมี mask)
    por     mm0, mm2              ; รวม fg และ bg
    
    movq    [output], mm0
    emms
    ret
```

---

## 9. PCMPEQ - Packed Compare Equal

### 9.1 PCMPEQB/PCMPEQW/PCMPEQD

```nasm
; PCMPEQB dst, src  ; เปรียบเทียบ 8 pairs of bytes
; PCMPEQW dst, src  ; เปรียบเทียบ 4 pairs of words
; PCMPEQD dst, src  ; เปรียบเทียบ 2 pairs of dwords

; ผลลัพธ์: 0xFF (255) ถ้าเท่ากัน, 0x00 ถ้าไม่เท่ากัน
; ใช้เป็น mask สำหรับ conditional operations

section .data
    search_char     db  'A', 'A', 'A', 'A', 'A', 'A', 'A', 'A'  ; ค้นหา 'A'
    text_data       db  'B', 'A', 'C', 'A', 'D', 'A', 'E', 'A'  ; ข้อความ

section .text
test_pcmpeq:
    movq    mm0, [search_char]    ; MM0 = {'A','A','A','A','A','A','A','A'}
    movq    mm1, [text_data]      ; MM1 = text to search
    pcmpeqb mm0, mm1              ; เปรียบเทียบแต่ละ byte
    ; ผล: positions 1,3,5,7 = 0xFF, others = 0x00
    ; MM0 = {0xFF, 0x00, 0xFF, 0x00, 0xFF, 0x00, 0xFF, 0x00}
    ; (little-endian: position 1,3,5,7 match 'A')
    
    ; ใช้ PMOVMSKB เพื่อดู match positions (SSE instruction)
    ; pmovmskb eax, mm0           ; EAX = bitmask ของผล (ต้องการ SSE)
    
    emms
    ret
```

### 9.2 ตัวอย่างจริง: ค้นหา String ด้วย MMX

```nasm
; ค้นหา character ใน string แบบ SIMD
; ประมวลผล 8 bytes ต่อ iteration

section .data
    target_char     db  'x'
    padding         times 7 db 0   ; pad to 8 bytes

section .text
; Input: ESI = string pointer, ECX = length, AL = char to find
; Output: EBX = position found (-1 if not found)
mmx_strchr:
    push    ebp
    mov     ebp, esp
    push    esi
    push    ebx
    
    ; โหลด character เข้า MM2 (broadcast ไป 8 positions)
    movd    mm2, eax              ; MM2[0:7] = char
    ; broadcast: copy byte 0 to all 8 positions
    punpcklbw mm2, mm2            ; MM2 = {c,c,c,c,c,c,c,c} (ใช้ trick)
    punpcklwd mm2, mm2
    punpckldq mm2, mm2
    
    xor     ebx, ebx              ; EBX = current position
    
.search_loop:
    cmp     ecx, 8
    jl      .search_single        ; น้อยกว่า 8 bytes ใช้ loop ธรรมดา
    
    movq    mm0, [esi + ebx]      ; โหลด 8 bytes จาก string
    pcmpeqb mm0, mm2              ; เปรียบเทียบกับ target char
    
    ; ตรวจสอบว่ามี match ไหม
    ; ใน MMX ไม่มี PMOVMSKB จึงใช้วิธีอื่น
    movq    mm1, mm0
    ; ตรวจสอบว่า mm0 != 0
    pxor    mm3, mm3              ; MM3 = 0
    pcmpeqb mm0, mm3              ; mm0 = 0xFF where mm0 was 0, 0x00 where matched
    
    ; ... (simplified - production code ใช้ SSE/PMOVMSKB แทน)
    
    add     ebx, 8
    sub     ecx, 8
    jmp     .search_loop
    
.search_single:
    ; ค้นหา byte ที่เหลือแบบ scalar
    ; ...
    
    emms
    pop     ebx
    pop     esi
    pop     ebp
    ret
```

---

## 10. PCMPGT - Packed Compare Greater Than

### 10.1 PCMPGTB/PCMPGTW/PCMPGTD

```nasm
; PCMPGTB dst, src  ; signed greater than, byte
; PCMPGTW dst, src  ; signed greater than, word
; PCMPGTD dst, src  ; signed greater than, dword

; ผลลัพธ์: 0xFF ถ้า dst > src (signed), 0x00 ถ้าไม่

section .data
    values_a    db  100, -50, 30, -10, 0, 127, -128, 50
    values_b    db   50,  10, 30, -20, 0, 100, -100, 60

section .text
test_pcmpgt:
    movq    mm0, [values_a]       ; โหลด values A
    movq    mm1, [values_b]       ; โหลด values B
    pcmpgtb mm0, mm1              ; MM0[i] = 0xFF if a[i] > b[i] (signed)
    ; 100>50 → 0xFF
    ; -50>10 → 0x00 (เพราะ -50 < 10)
    ; 30>30 → 0x00 (ไม่ใช่ strictly greater)
    ; -10>-20 → 0xFF (-10 > -20 ในแง่ signed)
    
    ; ใช้ผลลัพธ์เป็น mask สำหรับ conditional select
    movq    mm2, [values_a]
    movq    mm3, [values_b]
    
    ; เลือก max: result = (a > b) ? a : b
    pand    mm2, mm0              ; เก็บ a ที่ a > b
    pandn   mm0, mm3              ; เก็บ b ที่ a <= b
    por     mm2, mm0              ; รวม: max(a, b)
    
    emms
    ret
```

### 10.2 ตัวอย่าง: SIMD Max/Min Operations

```nasm
; หา maximum ของ 2 arrays แบบ SIMD

section .data
    arr1    db  1, 5, 3, 7, 2, 8, 4, 6
    arr2    db  4, 2, 6, 3, 8, 1, 7, 5
    maxArr  db  8 dup(0)
    minArr  db  8 dup(0)

section .text
simd_max_min:
    movq    mm0, [arr1]           ; MM0 = arr1
    movq    mm1, [arr2]           ; MM1 = arr2
    
    ; คำนวณ max(arr1, arr2)
    movq    mm2, mm0              ; MM2 = arr1 copy
    movq    mm3, mm1              ; MM3 = arr2 copy
    pcmpgtb mm2, mm3              ; MM2[i] = 0xFF if arr1[i] > arr2[i]
    movq    mm4, mm2              ; สำรอง mask
    pand    mm0, mm2              ; เก็บ arr1 ที่ arr1 > arr2
    pandn   mm2, mm1              ; เก็บ arr2 ที่ arr1 <= arr2
    por     mm0, mm2              ; max result
    movq    [maxArr], mm0
    
    ; คำนวณ min(arr1, arr2)
    ; MM4 = mask ที่ arr1 > arr2
    movq    mm0, [arr1]
    movq    mm1, [arr2]
    pand    mm1, mm4              ; เก็บ arr2 ที่ arr1 > arr2 (นี่คือ min)
    pandn   mm4, mm0              ; เก็บ arr1 ที่ arr1 <= arr2 (นี่คือ min)
    por     mm1, mm4              ; min result
    movq    [minArr], mm1
    
    emms
    ret
```

---

## 11. PUNPCK - Pack and Unpack Instructions

### 11.1 PUNPCKLBW/PUNPCKHBW - Unpack Bytes to Words

```nasm
; PUNPCKLBW dst, src
; Interleave lower 4 bytes of dst and src into words
; dst[0:63] = {src[3], dst[3], src[2], dst[2], src[1], dst[1], src[0], dst[0]}
;              (as bytes, creating 4 words)

; PUNPCKHBW dst, src
; Interleave upper 4 bytes

section .data
    low_bytes   db  1, 2, 3, 4, 5, 6, 7, 8   ; 8 bytes
    zero_bytes  db  0, 0, 0, 0, 0, 0, 0, 0   ; 8 zeros

section .text
test_punpck:
    movq    mm0, [low_bytes]      ; MM0 = {8,7,6,5,4,3,2,1}
    movq    mm1, [zero_bytes]     ; MM1 = {0,0,0,0,0,0,0,0}
    
    ; Zero-extend bytes to words (unpack with zero)
    movq    mm2, mm0              ; MM2 = copy of mm0
    punpcklbw mm0, mm1            ; MM0 = {0,4, 0,3, 0,2, 0,1} as bytes
                                   ;     = {4, 3, 2, 1} as words (zero-extended)
    punpckhbw mm2, mm1            ; MM2 = {0,8, 0,7, 0,6, 0,5} as bytes
                                   ;     = {8, 7, 6, 5} as words (zero-extended)
    ; ตอนนี้มี 8 words ที่ zero-extended จาก 8 bytes
    ret
```

### 11.2 PUNPCKLWD/PUNPCKHWD - Unpack Words to DWords

```nasm
; PUNPCKLWD dst, src
; Interleave lower 2 words of dst and src into dwords

section .data
    words4  dw  10, 20, 30, 40
    zeros4  dw   0,  0,  0,  0

section .text
test_punpcklwd:
    movq    mm0, [words4]         ; MM0 = {40,30,20,10}
    movq    mm1, [zeros4]         ; MM1 = {0,0,0,0}
    
    punpcklwd mm0, mm1            ; MM0 = {0,20, 0,10} as words
                                   ;     = {20, 10} as dwords (zero-extended)
    ; เหมาะสำหรับ convert 16-bit pixels เป็น 32-bit
    ret
```

### 11.3 PACKUSWB - Pack Words to Unsigned Bytes with Saturation

```nasm
; PACKUSWB dst, src
; Pack 8 signed words into 8 unsigned bytes with saturation
; dst = pack(dst[4 words] || src[4 words])

section .data
    words_to_pack   dw  300, -10, 128, 255, 256, 0, 1000, 100

section .text
test_packus:
    movq    mm0, [words_to_pack]      ; MM0 = {256,1000,0,128}  (words 4-7 first)
    
    ; เพื่อ pack ทั้ง 8 words ต้องใช้ 2 registers
    movq    mm1, [words_to_pack]      ; MM1 = words 0-3
    movq    mm0, [words_to_pack + 8]  ; MM0 = words 4-7
    
    packuswb mm1, mm0             ; pack 8 words เป็น 8 bytes
    ; 300 → 255 (saturated), -10 → 0 (saturated), 128 → 128, 255 → 255
    ; 256 → 255 (saturated), 0 → 0, 1000 → 255, 100 → 100
    
    emms
    ret
```

---

## 12. PSLL/PSRL/PSRA - Shift Instructions

### 12.1 PSLLW/PSLLD/PSLLQ - Shift Left Logical

```nasm
; PSLLW dst, count  ; shift left 4 words by count bits
; PSLLD dst, count  ; shift left 2 dwords by count bits
; PSLLQ dst, count  ; shift left 1 qword by count bits
; PSLLW dst, mm     ; shift by value in mm register

section .data
    words_shift     dw  1, 2, 4, 8

section .text
test_psll:
    movq    mm0, [words_shift]    ; MM0 = {8,4,2,1}
    
    psllw   mm0, 3                ; shift each word left by 3 bits
    ; 1 << 3 = 8, 2 << 3 = 16, 4 << 3 = 32, 8 << 3 = 64
    ; ผล: {64, 32, 16, 8}
    
    ; ใช้ shift สำหรับ multiply by power of 2
    movq    mm1, [values]
    pslld   mm1, 2                ; คูณทุก dword ด้วย 4 (2^2)
    
    emms
    ret
```

### 12.2 PSRLW/PSRLD/PSRLQ - Shift Right Logical

```nasm
; PSRLW dst, count  ; shift right logical (fill with 0)

section .text
test_psrl:
    movq    mm0, [pixel_data]     ; MM0 = pixel bytes
    
    ; ลด brightness ด้วยการ shift right
    movq    mm1, mm0
    punpcklbw mm1, mm3            ; zero-extend bytes to words
    psrlw   mm1, 1                ; หาร 2 (ลด brightness 50%)
    packuswb mm1, mm3             ; pack กลับเป็น bytes
    
    emms
    ret
```

### 12.3 PSRAW/PSRAD - Shift Right Arithmetic

```nasm
; PSRAW dst, count  ; shift right arithmetic (preserve sign)
; PSRAD dst, count  ; shift right arithmetic dwords

; ต่างจาก PSRL: fill ด้วย sign bit (ไม่ใช่ 0)

section .data
    signed_words    dw  -8, -16, 8, 16

section .text
test_psra:
    movq    mm0, [signed_words]   ; MM0 = {16,8,-16,-8}
    
    psraw   mm0, 2                ; หาร 4 (signed)
    ; -8 >> 2 = -2 (preserve sign)
    ; 16 >> 2 = 4
    
    ; เปรียบเทียบ:
    ; PSRLW: -8 (0xFFF8) >> 2 = 0x3FFE = 16382 (ผิดสำหรับ signed)
    ; PSRAW: -8 >> 2 = -2 = 0xFFFE = -2 (ถูก)
    
    emms
    ret
```

---

## 13. Practical Application: Fast Byte Operations

### 13.1 การคัดลอก Memory แบบ MMX-accelerated

```nasm
; mmx_memcpy - คัดลอก memory ด้วย MMX
; Input: EDI = dst, ESI = src, ECX = count (bytes)

section .text
global mmx_memcpy
mmx_memcpy:
    push    esi
    push    edi
    push    ecx
    
    ; ทำ 64-byte blocks ก่อน (8 MMX registers × 8 bytes)
    mov     eax, ecx
    shr     eax, 6                ; eax = count / 64
    test    eax, eax
    jz      .small_copy
    
.copy_64:
    movq    mm0, [esi + 0]        ; โหลด 8 bytes
    movq    mm1, [esi + 8]
    movq    mm2, [esi + 16]
    movq    mm3, [esi + 24]
    movq    mm4, [esi + 32]
    movq    mm5, [esi + 40]
    movq    mm6, [esi + 48]
    movq    mm7, [esi + 56]
    
    movq    [edi + 0], mm0        ; เก็บ 8 bytes
    movq    [edi + 8], mm1
    movq    [edi + 16], mm2
    movq    [edi + 24], mm3
    movq    [edi + 32], mm4
    movq    [edi + 40], mm5
    movq    [edi + 48], mm6
    movq    [edi + 56], mm7
    
    add     esi, 64               ; เลื่อน pointer
    add     edi, 64
    dec     eax
    jnz     .copy_64
    
.small_copy:
    ; ทำ 8-byte blocks ที่เหลือ
    mov     eax, ecx
    and     eax, 63               ; เหลือกี่ bytes
    shr     eax, 3                ; เหลือกี่ 8-byte blocks
    
.copy_8:
    test    eax, eax
    jz      .byte_copy
    movq    mm0, [esi]
    movq    [edi], mm0
    add     esi, 8
    add     edi, 8
    dec     eax
    jmp     .copy_8
    
.byte_copy:
    ; ทำ bytes ที่เหลือ
    mov     eax, ecx
    and     eax, 7
    rep movsb
    
    emms
    pop     ecx
    pop     edi
    pop     esi
    ret
```

### 13.2 Pixel Color Conversion - RGB to Grayscale

```nasm
; แปลง RGB pixels เป็น Grayscale
; Gray = 0.299R + 0.587G + 0.114B
; สำหรับ integer: Gray = (77R + 150G + 29B) >> 8

; รูปแบบ input: RGBA bytes (R,G,B,A, R,G,B,A)
; รูปแบบ output: grayscale bytes

section .data
    ; Coefficients ใน Q8 format (multiply then >> 8)
    coef_r      dw  77,  77,  77,  77    ; 0.299 × 256 ≈ 77
    coef_g      dw  150, 150, 150, 150   ; 0.587 × 256 ≈ 150
    coef_b      dw  29,  29,  29,  29    ; 0.114 × 256 ≈ 29

; ตัวอย่าง: แปลง 2 pixels พร้อมกัน
section .text
rgb_to_gray_mmx:
    ; Input: ESI = RGB pixel pointer (16 bytes = 2 RGBA pixels)
    ; Output: EDI = grayscale output
    
    movq    mm7, [esi]            ; โหลด 8 bytes = 2 pixels (RGBARGBA)
    
    ; แยก R component
    movq    mm0, mm7
    pand    mm0, [mask_r]         ; เก็บเฉพาะ R bytes
    ; ... (simplified, production code ซับซ้อนกว่า)
    
    ; ใช้ PMULLW สำหรับ weighted sum
    movq    mm1, mm0              ; R values
    punpcklbw mm1, mm3            ; zero-extend to words
    pmullw  mm1, [coef_r]         ; R × 77
    
    ; ทำเหมือนกันสำหรับ G และ B
    ; ...
    
    ; รวมและ shift right 8
    paddw   mm1, mm2              ; R×77 + G×150
    paddw   mm1, mm4              ; + B×29
    psrlw   mm1, 8                ; >> 8 (หาร 256)
    
    packuswb mm1, mm3             ; pack กลับเป็น bytes
    
    emms
    ret
```

---

## 14. SIMD Median Filter

### 14.1 ทฤษฎี Median Filter

Median filter เป็น algorithm ที่ใช้ในการประมวลผลภาพเพื่อลด noise โดยแทนที่ค่าแต่ละ pixel ด้วย median ของ neighborhood (เช่น 3×3 หรือ 5×5 block)

```
3×3 Neighborhood:
┌───┬───┬───┐
│ p1│ p2│ p3│
├───┼───┼───┤
│ p4│ p5│ p6│  → median(p1..p9)
├───┼───┼───┤
│ p7│ p8│ p9│
└───┴───┴───┘

Sort network สำหรับ 9 elements:
หา median โดยไม่ต้อง sort เต็มรูปแบบ
```

### 14.2 Comparison Network สำหรับ Median

```nasm
; Sort network approach สำหรับ median of 9
; ใช้ CAS (Compare and Swap) network

; PMAX/PMIN สำหรับ bytes (ใช้ PCMPGT + conditional select)
; สำหรับ MMX ต้องทำ PMIN/PMAX เอง (SSE มี PMAXUB/PMINUB)

%macro pminub 2
    ; compute min(mm%1, mm%2) ใส่ mm%1
    movq    mm6, mm%1             ; save mm%1
    psubusb mm6, mm%2             ; mm6 = max(mm%1-mm%2, 0)
    psubb   mm%1, mm6             ; mm%1 = mm%1 - max(0, mm%1-mm%2) = min
%endmacro

%macro pmaxub 2
    ; compute max(mm%1, mm%2) ใส่ mm%1
    movq    mm6, mm%2             ; save mm%2
    psubusb mm6, mm%1             ; mm6 = max(mm%2-mm%1, 0)
    paddusb mm%1, mm6             ; mm%1 = mm%1 + max(0, mm%2-mm%1) = max
%endmacro
```

### 14.3 ตัวอย่าง Median Filter แบบ Simplified

```nasm
; Simplified 1D median filter ด้วย MMX
; สำหรับ 3-element median ใน 8 positions พร้อมกัน

section .text
; Input: ESI = pixel row pointer, ECX = width
; MM0 = previous row, MM1 = current row, MM2 = next row
median_filter_1d:
    push    esi
    push    ecx
    
.process_8:
    cmp     ecx, 8
    jl      .done
    
    ; โหลด 3 consecutive values
    movq    mm0, [esi - 1]        ; left neighbor
    movq    mm1, [esi]            ; center
    movq    mm2, [esi + 1]        ; right neighbor
    
    ; หา median ของ 3 values ด้วย sort network:
    ; median(a,b,c) = max(min(a,b), min(max(a,b),c))
    
    ; Step 1: min and max ของ mm0, mm1
    movq    mm3, mm0
    movq    mm4, mm0
    ; mm3 = min(mm0, mm1), mm4 = max(mm0, mm1)
    ; ใช้ pminub/pmaxub macros
    
    ; Step 2: min(max(a,b), c)
    ; Step 3: max(min(a,b), result)
    
    ; (Simplified: ใช้ sort network แบบเต็ม)
    movq    mm5, mm1              ; tmp
    
    ; compare-and-swap: sort mm0, mm1
    movq    mm3, mm0
    psubusb mm3, mm1              ; mm3 = max(0, mm0-mm1)
    paddusb mm1, mm3              ; mm1 = max(mm0, mm1)
    psubusb mm0, mm3              ; mm0 = min(mm0, mm1) ... ไม่ถูกต้องสมบูรณ์
    
    ; สำหรับ production code ใช้ SSE2 ที่มี PMAXUB/PMINUB จะสะดวกกว่า
    
    movq    [edi], mm1            ; เขียน median
    
    add     esi, 8
    add     edi, 8
    sub     ecx, 8
    jmp     .process_8
    
.done:
    emms
    pop     ecx
    pop     esi
    ret
```

### 14.4 Complete Working Median Filter Example

```nasm
; median_filter_row: ทำ 3×1 median filter สำหรับ row
; Input: ESI = input pointer, EDI = output pointer
; ECX = length ใน bytes

section .text
global median_filter_row
median_filter_row:
    push    ebp
    mov     ebp, esp
    push    esi
    push    edi
    push    ecx
    push    ebx
    
    ; กรณีพิเศษ: first element = ตัวเอง
    mov     al, [esi]
    mov     [edi], al
    
    inc     esi
    inc     edi
    sub     ecx, 2
    
.loop:
    cmp     ecx, 0
    jle     .done_scalar
    
    ; โหลด left, center, right
    mov     al, [esi - 1]         ; left
    mov     bl, [esi]             ; center
    mov     cl, [esi + 1]         ; right
    
    ; หา median ของ 3 bytes (scalar เพื่อความถูกต้อง)
    ; Sort: swap if out of order
    cmp     al, bl
    jle     .ab_ok
    xchg    al, bl
.ab_ok:
    cmp     bl, cl
    jle     .bc_ok
    xchg    bl, cl
.bc_ok:
    cmp     al, bl
    jle     .ab2_ok
    xchg    al, bl
.ab2_ok:
    ; ตอนนี้ al = min, bl = median, cl = max
    mov     [edi], bl             ; เขียน median
    
    inc     esi
    inc     edi
    dec     ecx
    jmp     .loop
    
.done_scalar:
    ; กรณีพิเศษ: last element = ตัวเอง
    mov     al, [esi]
    mov     [edi], al
    
    pop     ebx
    pop     ecx
    pop     edi
    pop     esi
    pop     ebp
    ret


; MMX-accelerated version (ใช้ technique จาก Knuth sort network)
; ทำ 8 pixels พร้อมกัน
global median_filter_row_mmx
median_filter_row_mmx:
    push    ebp
    mov     ebp, esp
    sub     esp, 32
    push    esi
    push    edi
    push    ecx
    
    ; กรณีพิเศษ: edge pixels
    movzx   eax, byte [esi]
    mov     [edi], al
    
    ; ทำ bulk processing ด้วย MMX
.mmx_loop:
    cmp     ecx, 10               ; ต้องการ buffer space
    jl      .scalar_mode
    
    ; โหลด left (ESI-1), center (ESI), right (ESI+1)
    movq    mm0, [esi - 1]        ; 8 left neighbors
    movq    mm1, [esi]            ; 8 centers
    movq    mm2, [esi + 1]        ; 8 right neighbors
    
    ; Sort network สำหรับ median of 3:
    ; หลังจาก 3 CAS operations จะได้ median ใน mm1
    
    ; CAS(mm0, mm1): mm0=min(a,b), mm1=max(a,b)
    movq    mm3, mm0
    psubusb mm3, mm1              ; mm3 = max(0, a-b) = diff ถ้า a>b
    psubusb mm0, mm3              ; mm0 = a - diff = min(a,b)
    paddusb mm1, mm3              ; mm1 = b + diff = max(a,b)
    
    ; CAS(mm1, mm2): mm1=min(max(a,b), c), mm2=max(max(a,b), c)
    movq    mm4, mm1
    psubusb mm4, mm2
    psubusb mm1, mm4
    paddusb mm2, mm4
    ; ตอนนี้ mm2 = max ของทั้ง 3
    
    ; CAS(mm0, mm1): mm0=min ของทั้ง 3, mm1=median
    movq    mm5, mm0
    psubusb mm5, mm1
    psubusb mm0, mm5
    paddusb mm1, mm5
    ; ตอนนี้ mm1 = median!
    
    movq    [edi], mm1            ; เขียน 8 medians
    
    add     esi, 8
    add     edi, 8
    sub     ecx, 8
    jmp     .mmx_loop
    
.scalar_mode:
    ; ทำ pixels ที่เหลือแบบ scalar
    ; ... (เหมือน scalar version ข้างบน)
    
    emms
    pop     ecx
    pop     edi
    pop     esi
    mov     esp, ebp
    pop     ebp
    ret
```

---

## 15. Benchmark และ Performance Analysis

### 15.1 การวัด Performance

```nasm
; benchmark_mmx.asm - วัดเวลา MMX vs scalar

section .data
    test_size   equ 1024 * 1024      ; 1 MB = 1048576 bytes
    
    msg_scalar  db  "Scalar time: ", 0
    msg_mmx     db  "MMX time: ", 0

section .bss
    src_buf     resb    test_size
    dst_buf     resb    test_size
    start_tick  resq    1
    end_tick    resq    1

section .text
global benchmark_main
benchmark_main:
    ; เตรียม test data
    call    init_test_data
    
    ; Benchmark scalar add
    rdtsc                         ; อ่าน timestamp counter
    mov     [start_tick], eax
    mov     [start_tick + 4], edx
    
    call    scalar_byte_add
    
    rdtsc
    mov     [end_tick], eax
    mov     [end_tick + 4], edx
    ; คำนวณ elapsed cycles ...
    
    ; Benchmark MMX add
    rdtsc
    mov     [start_tick], eax
    mov     [start_tick + 4], edx
    
    call    mmx_byte_add
    
    rdtsc
    ; คำนวณ elapsed cycles ...
    
    ret

; Scalar implementation
scalar_byte_add:
    push    esi
    push    edi
    push    ecx
    mov     esi, src_buf
    mov     edi, dst_buf
    mov     ecx, test_size
.loop:
    mov     al, [esi]
    add     al, 10                ; บวก 10 กับทุก byte
    mov     [edi], al
    inc     esi
    inc     edi
    dec     ecx
    jnz     .loop
    pop     ecx
    pop     edi
    pop     esi
    ret

; MMX implementation
mmx_byte_add:
    push    esi
    push    edi
    push    ecx
    mov     esi, src_buf
    mov     edi, dst_buf
    mov     ecx, test_size
    
    ; สร้าง constant: {10,10,10,10,10,10,10,10}
    mov     eax, 0x0A0A0A0A
    movd    mm7, eax
    punpckldq mm7, mm7            ; MM7 = {10,10,10,10,10,10,10,10}
    
.loop:
    movq    mm0, [esi]            ; โหลด 8 bytes
    paddusb mm0, mm7              ; บวก 10 กับทุก byte (saturating)
    movq    [edi], mm0            ; เก็บ 8 bytes
    add     esi, 8
    add     edi, 8
    sub     ecx, 8
    jnz     .loop
    
    emms
    pop     ecx
    pop     edi
    pop     esi
    ret
```

### 15.2 ผลการทดสอบที่คาดหวัง

```
Performance Comparison (estimated, Intel Pentium MMX era):

Operation: Add constant to 1MB byte array

Method          | Time (approx)  | Speedup
----------------|----------------|--------
Scalar (loop)   | 8,000,000 cy   | 1.0x
MMX (8-wide)    | 1,200,000 cy   | 6.7x
MMX (unrolled)  |   800,000 cy   | 10.0x

Memory throughput limited on modern CPUs:
Modern x86 (Skylake):
Scalar: ~2M bytes/cycle (with prefetch)
MMX: ~8M bytes/cycle (8x width + better pipelining)

Note: On modern CPUs, use SSE2/AVX2 instead of MMX for better performance
MMX useful for: legacy code, 64-bit register operations on old hardware
```

---

## 16. ตัวอย่างโปรแกรมสมบูรณ์: Image Brightness Adjustment

### 16.1 โปรแกรมปรับความสว่างภาพ

```nasm
; image_brightness.asm
; ปรับความสว่างของภาพ grayscale ด้วย MMX
; Compile: nasm -f elf32 image_brightness.asm
; Link: ld -m elf_i386 -o image_brightness image_brightness.o

%include "syscall.inc"  ; syscall definitions

section .data
    ; PPM header สำหรับ test image
    ppm_header  db  "P5", 10          ; P5 = binary PGM (grayscale)
    width_str   db  "256 256", 10     ; 256×256 pixels
    maxval_str  db  "255", 10         ; max value
    
    brightness_add  db  50             ; เพิ่มความสว่าง +50
    brightness_sub  db  50             ; ลด brightness -50

section .bss
    image_buf   resb    256 * 256      ; 64KB image buffer
    output_buf  resb    256 * 256      ; output buffer

section .text
global _start

_start:
    ; เตรียม test image (gradient)
    call    create_test_image
    
    ; เพิ่มความสว่าง
    call    increase_brightness
    
    ; เขียน output
    call    write_output
    
    ; จบโปรแกรม
    mov     eax, 1
    xor     ebx, ebx
    int     0x80

; สร้าง test image: gradient จาก 0-255
create_test_image:
    mov     edi, image_buf
    mov     ecx, 256 * 256
    xor     eax, eax
.fill:
    mov     [edi], al
    inc     edi
    inc     eax
    cmp     al, 255
    jne     .no_wrap
    xor     eax, eax
.no_wrap:
    dec     ecx
    jnz     .fill
    ret

; เพิ่มความสว่างด้วย MMX
increase_brightness:
    push    esi
    push    edi
    push    ecx
    
    mov     esi, image_buf
    mov     edi, output_buf
    mov     ecx, 256 * 256
    
    ; สร้าง constant vector: 8 copies ของ brightness_add
    movzx   eax, byte [brightness_add]  ; EAX = brightness value
    movd    mm7, eax              ; MM7[0:7] = brightness
    ; broadcast ไป 8 positions
    punpcklbw mm7, mm7            ; MM7 = {b,b,b,b,b,b,b,b}
    punpcklwd mm7, mm7
    punpckldq mm7, mm7
    
.brightness_loop:
    cmp     ecx, 8
    jl      .scalar_tail
    
    movq    mm0, [esi]            ; โหลด 8 pixels
    paddusb mm0, mm7              ; บวก brightness (saturate ที่ 255)
    movq    [edi], mm0            ; เก็บ 8 pixels
    
    add     esi, 8
    add     edi, 8
    sub     ecx, 8
    jmp     .brightness_loop
    
.scalar_tail:
    test    ecx, ecx
    jz      .done
.scalar_loop:
    movzx   eax, byte [esi]
    add     eax, [brightness_add]
    cmp     eax, 255
    jle     .no_clamp
    mov     eax, 255
.no_clamp:
    mov     [edi], al
    inc     esi
    inc     edi
    dec     ecx
    jnz     .scalar_loop
    
.done:
    emms
    pop     ecx
    pop     edi
    pop     esi
    ret

write_output:
    ; เขียน output_buf ไปที่ stdout (simplified)
    mov     eax, 4                ; sys_write
    mov     ebx, 1                ; stdout
    mov     ecx, output_buf
    mov     edx, 256 * 256
    int     0x80
    ret
```

---

## 17. ตัวอย่างโปรแกรม: Histogram Calculation

### 17.1 การคำนวณ Histogram ด้วย MMX

```nasm
; histogram.asm
; คำนวณ histogram ของ grayscale image

section .bss
    histogram   resd    256    ; histogram: 256 dword counters

section .text
; Input: ESI = image data, ECX = pixel count
; Output: histogram array populated
compute_histogram:
    push    esi
    push    ecx
    push    eax
    
    ; Initialize histogram to zero
    mov     edi, histogram
    xor     eax, eax
    mov     ecx, 256
    rep stosd
    
    ; Count pixel values
    mov     ecx, [pixel_count]
    mov     esi, image_buf
    
.count_loop:
    ; Process 8 pixels per iteration โดยใช้ MMX ตรวจสอบ
    ; (ปกติ histogram accumulation เป็น scalar เพราะ data dependency)
    
    movzx   eax, byte [esi]       ; อ่าน pixel value
    inc     dword [histogram + eax * 4]  ; เพิ่ม count
    inc     esi
    dec     ecx
    jnz     .count_loop
    
    pop     eax
    pop     ecx
    pop     esi
    ret

; MMX-accelerated: ตรวจสอบ distribution ด้วย threshold
; ผล: นับ pixels ที่ >= threshold
count_bright_pixels_mmx:
    ; Input: ESI = image, ECX = count, DL = threshold
    ; Output: EAX = count of bright pixels
    
    push    esi
    push    ecx
    
    ; broadcast threshold ไป 8 positions
    movd    mm7, edx
    punpcklbw mm7, mm7
    punpcklwd mm7, mm7
    punpckldq mm7, mm7
    
    xor     eax, eax              ; EAX = accumulator
    pxor    mm6, mm6              ; MM6 = 0 (for compare)
    pxor    mm5, mm5              ; MM5 = running count
    
.bright_loop:
    cmp     ecx, 8
    jl      .scalar_bright
    
    movq    mm0, [esi]            ; โหลด 8 pixels
    movq    mm1, mm0
    psubusb mm1, mm7              ; mm1 = max(0, pixel - threshold)
    ; ถ้า pixel >= threshold: mm1 != 0
    ; ถ้า pixel < threshold: mm1 = 0
    
    pcmpeqb mm1, mm6              ; mm1 = 0xFF ที่ pixel < threshold
    ; invert: 0xFF ที่ pixel >= threshold
    pcmpeqb mm6, mm6              ; mm6 = all 1s
    pxor    mm1, mm6              ; ← invert
    pcmpeqb mm6, mm6              ; reset mm6
    
    ; นับ bits ที่ set (แต่ละ byte = 0xFF = 255 หมายความว่า 1 pixel)
    ; วิธีง่าย: แปลง 0xFF เป็น 1 แล้วบวก
    ; (production code ใช้ POPCNT หรือ PMOVMSKB+POPCNT)
    
    ; Simplified: บวก 0xFF (= -1) → ต้องการ abs/special handling
    
    add     esi, 8
    sub     ecx, 8
    jmp     .bright_loop
    
.scalar_bright:
    test    ecx, ecx
    jz      .bright_done
.scalar_b_loop:
    movzx   ebx, byte [esi]
    cmp     bl, dl
    jl      .not_bright
    inc     eax
.not_bright:
    inc     esi
    dec     ecx
    jnz     .scalar_b_loop
    
.bright_done:
    emms
    pop     ecx
    pop     esi
    ret
```

---

## 18. ARM Assembly - SIMD ที่เทียบเท่ากับ MMX

### 18.1 ARM NEON vs MMX

ARM ไม่มี MMX แต่มี **NEON** ซึ่งเป็น SIMD instruction set ที่ทรงพลังกว่า MMX มาก

```gas
// arm_simd_intro.s - ARM NEON SIMD
// GAS syntax (GNU Assembler)
// สำหรับ ARMv7 + NEON extension

.syntax unified
.arch armv7-a
.fpu neon
.text

// NEON Registers:
// Q0-Q15 (128-bit, quadword)
// D0-D31 (64-bit, doubleword) - D0/D1 = Q0, D2/D3 = Q1, etc.

// เทียบเท่า MMX บน ARM NEON:
// MMX MM0 (64-bit) ≈ ARM D0 (64-bit NEON register)

// NEON packed byte add (เทียบกับ PADDB)
.global neon_paddb_equiv
neon_paddb_equiv:
    // r0 = dest pointer, r1 = src1 pointer, r2 = src2 pointer
    vld1.8  {d0}, [r1]!      // โหลด 8 bytes จาก src1 เข้า d0
    vld1.8  {d1}, [r2]!      // โหลด 8 bytes จาก src2 เข้า d1
    vadd.i8 d0, d0, d1       // d0 = d0 + d1 (8 byte additions พร้อมกัน)
    vst1.8  {d0}, [r0]!      // เก็บ 8 bytes ไปที่ dest
    bx      lr               // return

// Saturating add (เทียบกับ PADDUSB)
.global neon_paddusb_equiv
neon_paddusb_equiv:
    vld1.8  {d0}, [r1]
    vld1.8  {d1}, [r2]
    vqadd.u8 d0, d0, d1      // unsigned saturating add
    vst1.8  {d0}, [r0]
    bx      lr

// Packed word multiply (เทียบกับ PMULLW)
.global neon_pmullw_equiv
neon_pmullw_equiv:
    vld1.16 {d0}, [r1]       // โหลด 4 words
    vld1.16 {d1}, [r2]       // โหลด 4 words
    vmul.i16 d0, d0, d1      // คูณ 4 words
    vst1.16 {d0}, [r0]
    bx      lr

// Byte compare equal (เทียบกับ PCMPEQB)
.global neon_pcmpeqb_equiv
neon_pcmpeqb_equiv:
    vld1.8  {d0}, [r1]
    vld1.8  {d1}, [r2]
    vceq.i8 d0, d0, d1       // d0[i] = 0xFF if equal, 0x00 if not
    vst1.8  {d0}, [r0]
    bx      lr
```

### 18.2 ARM NEON Packed Operations

```gas
// arm_neon_ops.s - ARM NEON packed operations
.syntax unified
.arch armv7-a
.fpu neon
.text

// Unpack (เทียบกับ PUNPCKLBW)
.global neon_unpack_bytes
neon_unpack_bytes:
    // r0 = output pointer, r1 = input pointer
    vld1.8  {d0}, [r1]       // โหลด 8 bytes
    
    vmovl.u8 q0, d0          // zero-extend bytes to 16-bit words
    // q0 = {0:byte7, 0:byte6, 0:byte5, 0:byte4, 0:byte3, 0:byte2, 0:byte1, 0:byte0}
    // เทียบกับ PUNPCKLBW mm0, zero
    
    vst1.16 {q0}, [r0]       // เก็บ 8 words (16 bytes)
    bx      lr

// Shift operations (เทียบกับ PSLLW/PSRLW)
.global neon_shifts
neon_shifts:
    vld1.16 {d0}, [r1]       // โหลด 4 words
    
    vshl.i16 d0, d0, #3      // shift left 3 bits (เทียบกับ PSLLW mm0, 3)
    vshr.u16 d1, d0, #2      // shift right 2 bits unsigned (เทียบกับ PSRLW)
    vshr.s16 d2, d0, #2      // shift right 2 bits signed (เทียบกับ PSRAW)
    
    bx      lr

// Absolute difference (bonus: ไม่มีใน MMX แต่มีใน NEON)
.global neon_abs_diff
neon_abs_diff:
    // r0 = output, r1 = src1, r2 = src2
    vld1.8  {d0}, [r1]
    vld1.8  {d1}, [r2]
    vabd.u8 d0, d0, d1       // d0[i] = |d0[i] - d1[i]|
    vst1.8  {d0}, [r0]
    bx      lr
```

---

## 19. MMX ใน Context ของ Real Programs

### 19.1 การ Check CPU Support สำหรับ MMX

```nasm
; ตรวจสอบว่า CPU รองรับ MMX หรือไม่

section .text
check_mmx_support:
    ; ใช้ CPUID instruction
    mov     eax, 1
    cpuid
    
    ; bit 23 ของ EDX = MMX support
    test    edx, (1 << 23)
    jz      .no_mmx
    
    ; MMX supported
    mov     eax, 1
    ret
    
.no_mmx:
    xor     eax, eax
    ret
```

### 19.2 MMX กับ Context Switch

```nasm
; เมื่อ OS ทำ context switch ต้อง save/restore MMX registers
; OS kernel ดูแลเรื่องนี้อัตโนมัติ (FXSAVE/FXRSTOR)

; แต่ถ้าเขียน kernel module ต้องทำเอง:
; FXSAVE [mem]   ; save x87 FPU + MMX state (512 bytes)
; FXRSTOR [mem]  ; restore x87 FPU + MMX state

; Note: FXSAVE พื้นที่ต้องการ alignment 16 bytes
section .bss
    fpu_state   resb 512
    align 16

section .text
save_mmx_state:
    fxsave  [fpu_state]
    ret

restore_mmx_state:
    fxrstor [fpu_state]
    ret
```

---

## 20. สรุปและบทเรียนต่อไป

### 20.1 MMX Instruction Reference Table

| Instruction | ประเภท | คำอธิบาย |
|-------------|--------|-----------|
| MOVQ | Data Transfer | ย้ายข้อมูล 64-bit |
| MOVD | Data Transfer | ย้ายข้อมูล 32-bit |
| EMMS | State | รีเซ็ต MMX/FPU state |
| PADDB/W/D | Arithmetic | Packed add (wrap) |
| PADDSB/SW | Arithmetic | Packed signed saturating add |
| PADDUSB/USW | Arithmetic | Packed unsigned saturating add |
| PSUBB/W/D | Arithmetic | Packed subtract |
| PSUBSB/SW | Arithmetic | Packed signed saturating sub |
| PSUBUSB/USW | Arithmetic | Packed unsigned saturating sub |
| PMULLW | Arithmetic | Packed multiply (low bits) |
| PMULHW | Arithmetic | Packed multiply (high bits) |
| PMADDWD | Arithmetic | Packed multiply-add |
| PAND | Logical | Packed bitwise AND |
| POR | Logical | Packed bitwise OR |
| PXOR | Logical | Packed bitwise XOR |
| PANDN | Logical | Packed bitwise AND-NOT |
| PCMPEQB/W/D | Compare | Packed compare equal |
| PCMPGTB/W/D | Compare | Packed compare greater-than |
| PUNPCKLBW | Pack/Unpack | Unpack low bytes to words |
| PUNPCKHBW | Pack/Unpack | Unpack high bytes to words |
| PUNPCKLWD | Pack/Unpack | Unpack low words to dwords |
| PUNPCKHWD | Pack/Unpack | Unpack high words to dwords |
| PUNPCKLDQ | Pack/Unpack | Unpack low dwords to qword |
| PUNPCKHDQ | Pack/Unpack | Unpack high dwords to qword |
| PACKSSWB | Pack/Unpack | Pack signed words to bytes |
| PACKUSWB | Pack/Unpack | Pack unsigned words to bytes |
| PACKSSDW | Pack/Unpack | Pack signed dwords to words |
| PSLLW/D/Q | Shift | Shift left logical |
| PSRLW/D/Q | Shift | Shift right logical |
| PSRAW/D | Shift | Shift right arithmetic |

### 20.2 เมื่อควรใช้ MMX

```
ควรใช้ MMX เมื่อ:
✓ ต้องรองรับ CPU เก่า (Pentium MMX era)
✓ ทำงานกับ 64-bit integers
✓ ต้องการ packed byte/word operations
✓ ไม่ต้องการ floating point

ควรใช้ SSE2 แทน เมื่อ:
✓ CPU รองรับ SSE2 (Pentium 4 ขึ้นไป)
✓ ต้องการ 128-bit registers (2x ความกว้าง)
✓ ต้องการ PMAXUB/PMINUB (มีใน SSE2 ไม่มีใน MMX)
✓ ต้องใช้ FPU ร่วมด้วย (ไม่ต้อง EMMS)

ควรใช้ AVX2 เมื่อ:
✓ CPU รองรับ AVX2 (Haswell ขึ้นไป)
✓ ต้องการ 256-bit registers (4x ความกว้าง)
✓ Maximum performance บน modern CPU
```

### 20.3 โค้ดสรุปการใช้งาน MMX

```nasm
; สรุป pattern การใช้ MMX ที่ถูกต้อง

; Pattern 1: Basic MMX processing
mmx_basic_pattern:
    ; 1. Check MMX support (ทำครั้งเดียวตอน startup)
    call    check_mmx_support
    test    eax, eax
    jz      .use_scalar_fallback
    
    ; 2. Load data
    movq    mm0, [src1]
    movq    mm1, [src2]
    
    ; 3. Process
    paddb   mm0, mm1
    
    ; 4. Store
    movq    [dst], mm0
    
    ; 5. Reset state (ต้องทำก่อนใช้ FPU!)
    emms
    
    ret

.use_scalar_fallback:
    ; ทำแบบ scalar ถ้าไม่มี MMX
    ; ...
    ret

; Pattern 2: Loop with MMX
mmx_loop_pattern:
    ; Setup constant
    mov     eax, 0x0A0A0A0A
    movd    mm7, eax
    punpckldq mm7, mm7
    
    ; Main loop (process 8 bytes per iteration)
.loop:
    cmp     ecx, 8
    jl      .done
    movq    mm0, [esi]
    paddusb mm0, mm7
    movq    [edi], mm0
    add     esi, 8
    add     edi, 8
    sub     ecx, 8
    jmp     .loop
    
.done:
    ; Handle remaining bytes (<8)
    ; ...
    emms
    ret
```

### 20.4 การเปรียบเทียบ MMX กับ Scalar

```
Benchmark Results (Conceptual, 1MB data):

Operation           Scalar    MMX      Speedup
-----------------   ------    -----    -------
Byte add            100%      13%      7.7x
Word add            100%      25%      4.0x
Byte compare        100%      15%      6.7x
Memory copy         100%      20%      5.0x
Color conversion    100%      12%      8.3x

Note: Actual speedup depends on:
- Memory bandwidth
- Cache utilization
- CPU pipeline effects
- Loop overhead
- Data alignment
```

---

## 21. แบบฝึกหัด (Exercises)

### แบบฝึกหัดระดับ 1: พื้นฐาน

```
1. เขียน MMX function เพื่อ invert ทุก byte ใน array
   (ทำ 255-x สำหรับทุก byte โดยใช้ PXOR กับ 0xFF mask)

2. เขียน function นับจำนวน bytes ที่เท่ากับ 0 ใน buffer
   โดยใช้ PCMPEQB

3. เขียน function หา absolute difference ระหว่าง 2 byte arrays
   (|a[i] - b[i]|) โดยใช้ PSUBUSB
```

### แบบฝึกหัดระดับ 2: กลาง

```
4. เขียน RGB to grayscale converter สำหรับ 8 pixels พร้อมกัน
   Gray = (R + G + B) / 3 (approximate)
   
5. เขียน threshold function: pixel[i] = pixel[i] >= T ? 255 : 0
   โดยใช้ PCMPGTB และ bitwise ops
   
6. เขียน running sum ของ 4 words โดยใช้ PADDD
```

### แบบฝึกหัดระดับ 3: ยาก

```
7. เขียน FIR filter ด้วย PMADDWD
   output[i] = coef[0]*input[i] + coef[1]*input[i-1] + coef[2]*input[i-2]
   
8. เขียน 3×3 median filter สำหรับ grayscale image
   
9. Implement ตัวเข้ารหัสแบบ XOR cipher บน byte arrays ด้วย MMX
   
10. เขียน histogram equalization ด้วย MMX
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **MMX Registers** (MM0-MM7) และการ alias กับ x87 FPU stack
2. **EMMS** instruction สำคัญสำหรับรีเซ็ต state ก่อนใช้ FPU
3. **Packed integer types**: byte (8×8), word (4×16), dword (2×32)
4. **Arithmetic instructions**: PADDB/W/D, PADDS, PADDUS, PSUB variants
5. **Multiply instructions**: PMULLW, PMULHW, PMADDWD
6. **Bitwise operations**: PAND, POR, PXOR, PANDN
7. **Compare instructions**: PCMPEQB/W/D, PCMPGTB/W/D
8. **Pack/Unpack**: PUNPCK variants, PACKUS
9. **Shift instructions**: PSLL, PSRL, PSRA
10. **Applications**: memory copy, pixel operations, median filter

**บทต่อไป**: SSE (Streaming SIMD Extensions) ซึ่งเป็น evolution ของ MMX ที่เพิ่ม 128-bit registers (XMM0-XMM7) และ floating-point SIMD support ที่ MMX ไม่มี

---

*Part 042 - MMX Instructions | Assembly Programming Course*  
*ระดับ: Intermediate-Advanced | ความยาว: ~1,000+ บรรทัด*
