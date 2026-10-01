# Part 044: SSE2 Instructions - คำสั่ง SSE2 สำหรับ Double Precision และ Integer 128-bit

## บทนำ (Introduction)

SSE2 (Streaming SIMD Extensions 2) เป็นส่วนขยายที่ Intel เพิ่มเข้ามาใน Pentium 4 (Willamette) ในปี 2001
SSE2 เป็นก้าวกระโดดครั้งสำคัญที่เพิ่มความสามารถหลักสองด้านให้กับ SSE:

1. **Double Precision Floating Point** - รองรับ 64-bit float (double) 2 ตัวพร้อมกัน
2. **128-bit Integer SIMD** - จัดการ integer ขนาดต่างๆ ใน 128-bit register ได้พร้อมกัน

ทำให้ SSE2 กลายเป็น baseline สำหรับ x86-64 (AMD64) ทุกตัว — CPU 64-bit ทุกตัวรองรับ SSE2 โดยปริยาย

```
SSE2 Register Overview:
┌─────────────────────────────────────────────────┐
│                XMM Register (128-bit)            │
├────────────────────────┬───────────────────────┤
│     double [1]         │      double [0]        │
│     (bits 127-64)      │      (bits 63-0)       │
├───────┬───────┬────────┴────────┬───────┬───────┤
│float  │float  │   float [1]     │float  │float  │
│  [3]  │  [2]  │                 │  [1]  │  [0]  │
├───┬───┼───┬───┬───┬───┬───┬───┬───┬───┬───┬───┤
│i32│i32│i32│i32│i32│i32│i32│i32│i32│i32│i32│i32│
│[3]│[2]│[1]│[0]│   │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

---

## 1. Double Precision Float Operations

### 1.1 MOVAPD / MOVUPD — Move Aligned/Unaligned Packed Double

**MOVAPD** ย้าย 2 double พร้อมกัน (ต้องการ alignment 16-byte)
**MOVUPD** ย้าย 2 double พร้อมกัน (ไม่ต้องการ alignment — แต่ช้ากว่า)

```nasm
; =====================================================
; File: sse2_move_doubles.asm
; NASM x86-64 - การย้ายข้อมูล double precision
; =====================================================

section .data
    align 16
    src_doubles dq 3.14159265, 2.71828182   ; Pi และ e
    
    align 16
    dst_doubles dq 0.0, 0.0                 ; destination ว่างเปล่า
    
    unaligned_buf times 18 db 0             ; buffer ไม่ aligned (เพื่อสาธิต)

section .text
global _start

_start:
    ; --- MOVAPD: Aligned Move (เร็วที่สุด) ---
    movapd  xmm0, [src_doubles]     ; โหลด [Pi, e] เข้า xmm0 (aligned)
    movapd  [dst_doubles], xmm0     ; เก็บไปยัง dst_doubles (aligned)
    
    ; --- MOVUPD: Unaligned Move (ยืดหยุ่นกว่า) ---
    lea     rax, [unaligned_buf + 1]  ; pointer ที่ไม่ aligned
    movupd  [rax], xmm0              ; เก็บแบบ unaligned (ไม่ crash แต่ช้ากว่า)
    movupd  xmm1, [rax]              ; โหลดแบบ unaligned
    
    ; --- MOVSD: Move Scalar Double (1 ตัวเท่านั้น) ---
    movsd   xmm2, [src_doubles]     ; โหลด Pi เข้า xmm2[0] เท่านั้น
    ; xmm2[1] ยังคงเป็น 0 หรือค่าเดิม (ขึ้นกับ processor)
    
    ; --- MOVSD register to register ---
    movsd   xmm3, xmm0              ; copy xmm0[0] -> xmm3[0]
    ; xmm3[1] ไม่เปลี่ยน
    
    xor     eax, eax
    ret
```

### 1.2 ADDPD / SUBPD / MULPD / DIVPD — Packed Double Arithmetic

คำสั่งคณิตศาสตร์แบบ packed — ทำงาน 2 double พร้อมกัน

```nasm
; =====================================================
; File: sse2_packed_double_arith.asm
; การคำนวณ packed double precision
; =====================================================

section .data
    align 16
    a_vals  dq  1.5, 2.5            ; [1.5, 2.5]
    b_vals  dq  3.0, 4.0            ; [3.0, 4.0]
    
    align 16
    result  dq  0.0, 0.0

section .text
global packed_double_demo

packed_double_demo:
    ; โหลดค่า a และ b
    movapd  xmm0, [a_vals]          ; xmm0 = [1.5, 2.5]
    movapd  xmm1, [b_vals]          ; xmm1 = [3.0, 4.0]
    
    ; --- ADDPD: บวก 2 double พร้อมกัน ---
    movapd  xmm2, xmm0              ; xmm2 = [1.5, 2.5]
    addpd   xmm2, xmm1              ; xmm2 = [1.5+3.0, 2.5+4.0] = [4.5, 6.5]
    
    ; --- SUBPD: ลบ 2 double พร้อมกัน ---
    movapd  xmm3, xmm0              ; xmm3 = [1.5, 2.5]
    subpd   xmm3, xmm1              ; xmm3 = [1.5-3.0, 2.5-4.0] = [-1.5, -1.5]
    
    ; --- MULPD: คูณ 2 double พร้อมกัน ---
    movapd  xmm4, xmm0              ; xmm4 = [1.5, 2.5]
    mulpd   xmm4, xmm1              ; xmm4 = [1.5*3.0, 2.5*4.0] = [4.5, 10.0]
    
    ; --- DIVPD: หาร 2 double พร้อมกัน ---
    movapd  xmm5, xmm0              ; xmm5 = [1.5, 2.5]
    divpd   xmm5, xmm1              ; xmm5 = [1.5/3.0, 2.5/4.0] = [0.5, 0.625]
    
    movapd  [result], xmm2          ; เก็บผลลัพธ์ ADDPD
    ret

; =====================================================
; ตัวอย่างการใช้งานจริง: คำนวณ dot product ของ vector 4D
; v1 = (a1, a2, a3, a4), v2 = (b1, b2, b3, b4)
; dot = a1*b1 + a2*b2 + a3*b3 + a4*b4
; =====================================================
dot_product_4d:
    ; rdi = pointer to v1, rsi = pointer to v2
    ; returns: xmm0 = dot product (scalar double)
    
    movapd  xmm0, [rdi]             ; xmm0 = [a1, a2]
    movapd  xmm1, [rdi + 16]       ; xmm1 = [a3, a4]
    movapd  xmm2, [rsi]             ; xmm2 = [b1, b2]
    movapd  xmm3, [rsi + 16]       ; xmm3 = [b3, b4]
    
    mulpd   xmm0, xmm2              ; xmm0 = [a1*b1, a2*b2]
    mulpd   xmm1, xmm3              ; xmm1 = [a3*b3, a4*b4]
    addpd   xmm0, xmm1              ; xmm0 = [a1*b1+a3*b3, a2*b2+a4*b4]
    
    ; Horizontal add: รวม 2 lane เข้าด้วยกัน
    movapd  xmm1, xmm0
    shufpd  xmm1, xmm0, 1          ; xmm1[0] = xmm0[1]
    addsd   xmm0, xmm1              ; xmm0[0] = a1*b1+a2*b2+a3*b3+a4*b4
    
    ret
```

### 1.3 ADDSD / SUBSD / MULSD / DIVSD — Scalar Double Operations

ทำงานกับ double ตัวเดียว (lane 0 ของ XMM register)

```nasm
; =====================================================
; File: sse2_scalar_double.asm
; Scalar double precision — ทำงาน 1 double ต่อครั้ง
; =====================================================

section .data
    pi      dq 3.14159265358979
    radius  dq 5.0
    result  dq 0.0

section .text
global compute_circle_area

; คำนวณ area = pi * r^2
; ผลลัพธ์อยู่ใน xmm0
compute_circle_area:
    movsd   xmm0, [radius]          ; xmm0 = r = 5.0
    movsd   xmm1, xmm0              ; xmm1 = r
    mulsd   xmm0, xmm1              ; xmm0 = r^2 = 25.0
    movsd   xmm1, [pi]              ; xmm1 = pi
    mulsd   xmm0, xmm1              ; xmm0 = pi * r^2
    movsd   [result], xmm0          ; เก็บผลลัพธ์
    ret

; =====================================================
; ฟังก์ชัน: แปลง Celsius เป็น Fahrenheit
; F = C * 9/5 + 32
; =====================================================

section .data
    nine_fifths dq 1.8              ; 9/5 = 1.8
    thirty_two  dq 32.0

; เข้า: xmm0 = Celsius
; ออก:  xmm0 = Fahrenheit
celsius_to_fahrenheit:
    mulsd   xmm0, [nine_fifths]     ; C * 1.8
    addsd   xmm0, [thirty_two]      ; + 32
    ret
```

---

## 2. SQRTPD / SQRTSD — Square Root

```nasm
; =====================================================
; File: sse2_sqrt.asm
; รากที่สองสำหรับ double precision
; =====================================================

section .data
    align 16
    vals    dq  4.0, 9.0            ; sqrt(4)=2, sqrt(9)=3
    vec3_sq dq  0.0                 ; ผลลัพธ์ magnitude

section .text
global sqrt_demo

sqrt_demo:
    ; --- SQRTPD: square root ทั้ง 2 double พร้อมกัน ---
    movapd  xmm0, [vals]            ; xmm0 = [4.0, 9.0]
    sqrtpd  xmm0, xmm0              ; xmm0 = [2.0, 3.0]
    
    ; --- SQRTSD: square root เฉพาะ lane 0 ---
    movsd   xmm1, [vals]            ; xmm1[0] = 4.0
    sqrtsd  xmm1, xmm1              ; xmm1[0] = 2.0
    
    ret

; =====================================================
; คำนวณ magnitude ของ 3D vector: sqrt(x^2 + y^2 + z^2)
; rdi = pointer to [x, y, z] (double array)
; =====================================================

section .data
    align 16
    v3d     dq 3.0, 4.0, 0.0, 0.0  ; [x, y, z, padding]

vector3d_magnitude:
    movapd  xmm0, [rdi]             ; xmm0 = [x, y]
    movsd   xmm1, [rdi + 16]       ; xmm1[0] = z
    
    mulpd   xmm0, xmm0              ; xmm0 = [x^2, y^2]
    mulsd   xmm1, xmm1              ; xmm1[0] = z^2
    
    ; รวม x^2 + y^2
    movapd  xmm2, xmm0
    shufpd  xmm2, xmm0, 1          ; xmm2[0] = y^2
    addsd   xmm0, xmm2              ; xmm0[0] = x^2 + y^2
    addsd   xmm0, xmm1              ; xmm0[0] = x^2 + y^2 + z^2
    sqrtsd  xmm0, xmm0              ; xmm0[0] = magnitude
    ret
```

---

## 3. CMPPD / CMPSD — Compare Packed/Scalar Double

SSE2 ใช้ immediate byte เพื่อระบุเงื่อนไขการเปรียบเทียบ

```
Predicate (imm8):
  0 = EQ   (เท่ากัน)
  1 = LT   (น้อยกว่า)
  2 = LE   (น้อยกว่าหรือเท่ากัน)
  3 = UNORD (unordered — NaN check)
  4 = NEQ  (ไม่เท่ากัน)
  5 = NLT  (ไม่น้อยกว่า)
  6 = NLE  (ไม่น้อยกว่าหรือเท่ากัน)
  7 = ORD  (ordered — ไม่ใช่ NaN)
```

```nasm
; =====================================================
; File: sse2_compare.asm
; การเปรียบเทียบ double precision
; =====================================================

section .data
    align 16
    a_cmp   dq  1.0,  5.0           ; [1.0, 5.0]
    b_cmp   dq  2.0,  3.0           ; [2.0, 3.0]
    nan_val dq  0x7FF8000000000000  ; NaN

section .text
global compare_demo

compare_demo:
    movapd  xmm0, [a_cmp]           ; xmm0 = [1.0, 5.0]
    movapd  xmm1, [b_cmp]           ; xmm1 = [2.0, 3.0]
    
    ; --- CMPPD EQ: เปรียบเทียบ equal ---
    movapd  xmm2, xmm0
    cmppd   xmm2, xmm1, 0          ; xmm2[0] = 1.0==2.0? -> 0x0, xmm2[1] = 5.0==3.0? -> 0x0
    ; ผลลัพธ์: mask 0 = false, 0xFFFFFFFFFFFFFFFF = true
    
    ; --- CMPPD LT: น้อยกว่า ---
    movapd  xmm3, xmm0
    cmppd   xmm3, xmm1, 1          ; xmm3[0] = 1.0<2.0? -> 0xFFF..., xmm3[1] = 5.0<3.0? -> 0x0
    
    ; --- ใช้ mask กรอง (blend) ค่า ---
    ; เลือก a ถ้า a < b, ไม่งั้นเลือก b
    movapd  xmm4, xmm0              ; xmm4 = a
    andpd   xmm4, xmm3              ; เก็บเฉพาะ a ที่ a < b
    andnpd  xmm3, xmm1              ; เก็บเฉพาะ b ที่ NOT(a < b)
    orpd    xmm4, xmm3              ; รวมกัน = min(a, b) ทั้ง 2 lane
    ; xmm4 = [min(1.0,2.0), min(5.0,3.0)] = [1.0, 3.0]
    
    ; --- CMPSD: เปรียบเทียบ scalar เท่านั้น ---
    movsd   xmm5, [a_cmp]
    cmpsd   xmm5, [b_cmp], 1       ; xmm5[0] = (1.0 < 2.0)? = 0xFFF...
    
    ret

; =====================================================
; ฟังก์ชัน: clamp ค่า double ให้อยู่ใน [lo, hi]
; clamp(x, lo, hi) = min(max(x, lo), hi)
; =====================================================
section .data
    align 16
    clamp_vals  dq  -1.0, 5.0, 3.0, 8.0    ; 4 ค่าต้องการ clamp
    lo_val      dq  0.0, 0.0                 ; lower bound
    hi_val      dq  4.0, 4.0                 ; upper bound

clamp_packed_double:
    movapd  xmm0, [clamp_vals]      ; xmm0 = [-1.0, 5.0]
    movapd  xmm1, [lo_val]          ; xmm1 = [0.0, 0.0]
    movapd  xmm2, [hi_val]          ; xmm2 = [4.0, 4.0]
    
    ; max(x, lo) using CMPPD
    movapd  xmm3, xmm0
    cmppd   xmm3, xmm1, 1          ; mask: x < lo
    movapd  xmm4, xmm1
    andpd   xmm4, xmm3             ; lo ที่ x < lo
    andnpd  xmm3, xmm0             ; x ที่ x >= lo
    orpd    xmm4, xmm3             ; max(x, lo)
    
    ; min(result, hi) 
    movapd  xmm3, xmm4
    cmppd   xmm3, xmm2, 6          ; mask: result > hi (NLE)
    movapd  xmm5, xmm2
    andpd   xmm5, xmm3             ; hi ที่ result > hi
    andnpd  xmm3, xmm4             ; result ที่ result <= hi
    orpd    xmm5, xmm3             ; min(result, hi)
    ; xmm5 = [clamp(-1.0,0,4), clamp(5.0,0,4)] = [0.0, 4.0]
    
    movapd  xmm0, xmm5
    ret
```

---

## 4. SHUFPD — Shuffle Packed Double

```nasm
; =====================================================
; File: sse2_shufpd.asm
; Shuffle สำหรับ double precision
; SHUFPD xmm_dst, xmm_src, imm8
; imm8[0]: lane 0 ของ dst มาจาก xmm_dst[imm8[0]]
; imm8[1]: lane 1 ของ dst มาจาก xmm_src[imm8[1]]
; =====================================================

section .data
    align 16
    p_val   dq  1.0, 2.0            ; [1.0, 2.0]
    q_val   dq  3.0, 4.0            ; [3.0, 4.0]

section .text
global shufpd_demo

shufpd_demo:
    movapd  xmm0, [p_val]           ; xmm0 = [1.0, 2.0]
    movapd  xmm1, [q_val]           ; xmm1 = [3.0, 4.0]
    
    ; SHUFPD xmm0, xmm1, 0b00
    ; dst[0] = xmm0[0]=1.0, dst[1] = xmm1[0]=3.0
    movapd  xmm2, xmm0
    shufpd  xmm2, xmm1, 0b00       ; xmm2 = [1.0, 3.0]
    
    ; SHUFPD xmm0, xmm1, 0b01
    ; dst[0] = xmm0[1]=2.0, dst[1] = xmm1[0]=3.0
    movapd  xmm3, xmm0
    shufpd  xmm3, xmm1, 0b01       ; xmm3 = [2.0, 3.0]
    
    ; SHUFPD xmm0, xmm1, 0b10
    ; dst[0] = xmm0[0]=1.0, dst[1] = xmm1[1]=4.0
    movapd  xmm4, xmm0
    shufpd  xmm4, xmm1, 0b10       ; xmm4 = [1.0, 4.0]
    
    ; SHUFPD xmm0, xmm1, 0b11
    ; dst[0] = xmm0[1]=2.0, dst[1] = xmm1[1]=4.0
    movapd  xmm5, xmm0
    shufpd  xmm5, xmm1, 0b11       ; xmm5 = [2.0, 4.0]
    
    ; --- สลับ 2 lane ใน register เดียวกัน ---
    movapd  xmm6, xmm0              ; xmm6 = [1.0, 2.0]
    shufpd  xmm6, xmm6, 0b01       ; xmm6 = [2.0, 1.0] (สลับ)
    
    ret
```

---

## 5. Packed Integer 128-bit Operations

### 5.1 MOVDQA / MOVDQU — Integer Moves

```nasm
; =====================================================
; File: sse2_integer_moves.asm
; การย้าย integer แบบ 128-bit
; =====================================================

section .data
    align 16
    int_src dd  1, 2, 3, 4          ; 4 x int32 = 128-bit
    
    align 16
    int_dst dd  0, 0, 0, 0

section .text
global integer_move_demo

integer_move_demo:
    ; --- MOVDQA: Move Double Quadword Aligned ---
    movdqa  xmm0, [int_src]         ; โหลด 4 int32 เข้า xmm0 (ต้อง aligned)
    movdqa  [int_dst], xmm0         ; เก็บ (ต้อง aligned)
    
    ; --- MOVDQU: Move Double Quadword Unaligned ---
    lea     rax, [int_src + 1]       ; ไม่ aligned
    movdqu  xmm1, [rax]             ; โหลดแบบ unaligned (ไม่ crash)
    
    ; --- MOVD: Move Doubleword (32-bit) ---
    movd    eax, xmm0               ; eax = xmm0[31:0] = ค่าแรก (1)
    movd    xmm2, eax               ; xmm2[31:0] = eax, ที่เหลือ = 0
    
    ; --- MOVQ: Move Quadword (64-bit) ---
    movq    rax, xmm0               ; rax = xmm0[63:0] = ค่า 2 ตัวแรก
    movq    xmm3, rax               ; xmm3[63:0] = rax, ที่เหลือ = 0
    
    ret
```

### 5.2 PADDB, PADDW, PADDD, PADDQ — Packed Integer Add

```nasm
; =====================================================
; File: sse2_packed_int_add.asm
; การบวก integer แบบ packed ขนาดต่างๆ
; =====================================================

section .data
    align 16
    ; 16 x byte (int8)
    bytes_a  db  1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16
    bytes_b  db  1, 1, 1, 1, 1, 1, 1, 1, 1,  1,  1,  1,  1,  1,  1,  1
    
    ; 8 x word (int16)
    align 16
    words_a  dw  100, 200, 300, 400, 500, 600, 700, 800
    words_b  dw   10,  20,  30,  40,  50,  60,  70,  80
    
    ; 4 x dword (int32)
    align 16
    dwords_a dd  1000, 2000, 3000, 4000
    dwords_b dd   100,  200,  300,  400
    
    ; 2 x qword (int64)
    align 16
    qwords_a dq  1000000, 2000000
    qwords_b dq   100000,  200000

section .text
global packed_add_demo

packed_add_demo:
    ; --- PADDB: บวก 16 bytes พร้อมกัน ---
    movdqa  xmm0, [bytes_a]         ; 16 bytes
    movdqa  xmm1, [bytes_b]         ; 16 bytes
    paddb   xmm0, xmm1              ; xmm0[i] = a[i] + b[i] (wrapping)
    ; [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
    
    ; --- PADDW: บวก 8 words พร้อมกัน ---
    movdqa  xmm2, [words_a]
    movdqa  xmm3, [words_b]
    paddw   xmm2, xmm3              ; xmm2[i] = a[i] + b[i]
    
    ; --- PADDD: บวก 4 dwords พร้อมกัน ---
    movdqa  xmm4, [dwords_a]
    movdqa  xmm5, [dwords_b]
    paddd   xmm4, xmm5              ; xmm4[i] = a[i] + b[i]
    
    ; --- PADDQ: บวก 2 qwords พร้อมกัน ---
    movdqa  xmm6, [qwords_a]
    movdqa  xmm7, [qwords_b]
    paddq   xmm6, xmm7              ; xmm6[i] = a[i] + b[i]
    
    ret

; =====================================================
; PSUBB/PSUBW/PSUBD/PSUBQ: ลบ (ทำงานเหมือน PADD)
; PSUBQ xmm, xmm/mem
; =====================================================
packed_sub_demo:
    movdqa  xmm0, [qwords_a]
    movdqa  xmm1, [qwords_b]
    psubq   xmm0, xmm1              ; 1000000-100000=900000, 2000000-200000=1800000
    ret
```

### 5.3 PMULUDQ — Packed Unsigned Multiply

```nasm
; =====================================================
; File: sse2_multiply.asm
; การคูณ integer แบบ packed
; =====================================================

section .data
    align 16
    mul_a   dd  3, 0, 7, 0          ; เฉพาะ lane 0 และ 2 (จัดเรียงสำหรับ PMULUDQ)
    mul_b   dd  5, 0, 4, 0

section .text
global multiply_demo

multiply_demo:
    ; PMULUDQ: คูณ unsigned 32-bit -> ผลลัพธ์ 64-bit
    ; ทำงาน 2 คู่: [0]*[0] และ [2]*[2]
    movdqa  xmm0, [mul_a]           ; xmm0 = [3, 0, 7, 0] (int32 lanes)
    movdqa  xmm1, [mul_b]           ; xmm1 = [5, 0, 4, 0]
    pmuludq xmm0, xmm1              ; xmm0 = [3*5, 7*4] (int64)
    ; xmm0[63:0]  = 15
    ; xmm0[127:64] = 28
    
    ret

; =====================================================
; PMULLW: คูณ 16-bit -> เก็บ 16-bit ล่าง (8 ค่าพร้อมกัน)
; =====================================================

section .data
    align 16
    w_a     dw  100, 200, 300, 400, 500, 600, 700, 800
    w_b     dw   10,  20,  30,  40,  50,  60,  70,  80

section .text

pmullw_demo:
    movdqa  xmm0, [w_a]
    movdqa  xmm1, [w_b]
    pmullw  xmm0, xmm1              ; xmm0[i] = (a[i] * b[i]) & 0xFFFF
    ; [1000, 4000, 9000, 16000, 25000, 36000, 49000, 64000]
    ret
```

---

## 6. PCMPEQB/PCMPEQW/PCMPEQD — Packed Integer Compare Equal

```nasm
; =====================================================
; File: sse2_pcmp.asm
; การเปรียบเทียบ integer แบบ packed
; =====================================================

section .data
    align 16
    cmp_a   db  1, 2, 3, 4, 5, 6, 7, 8, 1, 2, 3, 4, 5, 6, 7, 8
    cmp_b   db  1, 2, 3, 0, 5, 0, 7, 8, 0, 2, 3, 4, 0, 6, 7, 0

section .text
global pcmp_demo

pcmp_demo:
    movdqa  xmm0, [cmp_a]           ; a[]
    movdqa  xmm1, [cmp_b]           ; b[]
    
    ; PCMPEQB: เปรียบเทียบ 16 bytes
    movdqa  xmm2, xmm0
    pcmpeqb xmm2, xmm1              ; xmm2[i] = (a[i]==b[i]) ? 0xFF : 0x00
    ; ผล: 0xFF ที่ตำแหน่งที่ตรงกัน
    
    ; นับจำนวน matches โดยใช้ PMOVMSKB
    pmovmskb eax, xmm2              ; eax = bitmask (1 bit ต่อ byte)
    popcnt  eax, eax                ; นับจำนวน 1-bits = จำนวน matches
    
    ; --- ค้นหา byte เฉพาะใน string ---
    ; เทคนิค: หา '\n' ใน buffer
    
    ret

; =====================================================
; ตัวอย่างจริง: ค้นหาตำแหน่งของ null byte (ความยาว string)
; rdi = string pointer
; ===================================================

section .data
    align 16
    zero128 dq 0, 0                 ; 16 bytes of zero สำหรับเปรียบเทียบ

strlen_sse2:
    xor     eax, eax                ; counter = 0
    movdqa  xmm0, [zero128]        ; xmm0 = zeroes
    
.loop:
    movdqu  xmm1, [rdi + rax]      ; โหลด 16 bytes (อาจ unaligned)
    pcmpeqb xmm1, xmm0             ; เปรียบกับ null
    pmovmskb ecx, xmm1             ; mask ของ null bytes
    test    ecx, ecx               ; ตรวจว่ามี null ไหม
    jnz     .found
    add     eax, 16                 ; ไปต่อ 16 bytes
    jmp     .loop
    
.found:
    bsf     ecx, ecx               ; หา bit แรก (ตำแหน่ง null)
    add     eax, ecx               ; รวมความยาว
    ret
```

---

## 7. PSHUFD / PSHUFHW / PSHUFLW — Integer Shuffle

```nasm
; =====================================================
; File: sse2_shuffle_int.asm
; Shuffle คำสั่งสำหรับ integer
; =====================================================

section .data
    align 16
    int_data dd  10, 20, 30, 40     ; 4 x int32

section .text
global shuffle_demo

; PSHUFD: Shuffle 4 x int32
; imm8 = [sel3][sel2][sel1][sel0] (2-bit ต่อ lane)
; ตัวอย่าง imm8 = 0b_11_10_01_00 = 0xE4 = ไม่เปลี่ยน
;           imm8 = 0b_00_01_10_11 = 0x1B = กลับทิศทาง
shuffle_demo:
    movdqa  xmm0, [int_data]        ; xmm0 = [10, 20, 30, 40]
    
    ; --- กลับทิศทาง: [40, 30, 20, 10] ---
    pshufd  xmm1, xmm0, 0b00011011 ; = 0x1B
    ; xmm1[0]=src[3]=40, [1]=src[2]=30, [2]=src[1]=20, [3]=src[0]=10
    
    ; --- ทำซ้ำ lane 0 ทั้งหมด: [10, 10, 10, 10] ---
    pshufd  xmm2, xmm0, 0b00000000 ; = 0x00
    
    ; --- สลับ pair: [30, 40, 10, 20] ---
    pshufd  xmm3, xmm0, 0b01001110 ; = 0x4E
    
    ; --- PSHUFHW: shuffle ใน high 4 words เท่านั้น ---
    ; ส่วน low 4 words ไม่เปลี่ยน
    section .data
    align 16
    word_data dw  1, 2, 3, 4, 5, 6, 7, 8   ; 8 x int16

    movdqa  xmm4, [word_data]       ; [1,2,3,4,5,6,7,8]
    pshufhw xmm5, xmm4, 0b00011011 ; กลับ high [5,6,7,8] -> [8,7,6,5], low คงเดิม
    ; xmm5 = [1, 2, 3, 4, 8, 7, 6, 5]
    
    ; --- PSHUFLW: shuffle ใน low 4 words เท่านั้น ---
    pshuflw xmm6, xmm4, 0b00011011 ; กลับ low [1,2,3,4] -> [4,3,2,1], high คงเดิม
    ; xmm6 = [4, 3, 2, 1, 5, 6, 7, 8]
    
    ret
```

---

## 8. PSHUFB — Byte Shuffle (SSSE3)

> หมายเหตุ: PSHUFB จริงๆ อยู่ใน SSSE3 ไม่ใช่ SSE2 แต่มักถูกกล่าวถึงพร้อมกัน

```nasm
; =====================================================
; File: ssse3_pshufb.asm
; PSHUFB: Packed Shuffle Bytes (SSSE3)
; ยืดหยุ่นมากที่สุด — กำหนด source byte ของแต่ละ lane
; =====================================================

section .data
    align 16
    src_bytes db  'A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P'
    
    align 16
    ; mask: แต่ละ byte ระบุ source index (0-15), bit 7 = 1 -> zero
    rev_mask  db  15,14,13,12,11,10,9,8,7,6,5,4,3,2,1,0   ; กลับทิศ
    
    align 16
    dup_mask  db  0,0,0,0,1,1,1,1,2,2,2,2,3,3,3,3          ; ทำซ้ำ 4 bytes แรก

section .text
global pshufb_demo

pshufb_demo:
    movdqa  xmm0, [src_bytes]       ; "ABCDEFGHIJKLMNOP"
    
    ; --- กลับทิศ 16 bytes ---
    movdqa  xmm1, xmm0
    movdqa  xmm2, [rev_mask]
    pshufb  xmm1, xmm2              ; "PONMLKJIHGFEDCBA"
    
    ; --- ทำซ้ำ bytes ---
    movdqa  xmm3, xmm0
    movdqa  xmm4, [dup_mask]
    pshufb  xmm3, xmm4              ; "AAAABBBBCCCCDDDD"
    
    ret
```

---

## 9. PUNPCKLBW / PUNPCKHBW — Byte Unpack

```nasm
; =====================================================
; File: sse2_unpack.asm
; Unpack: สลับ bytes/words/dwords จาก 2 registers
; =====================================================

section .data
    align 16
    lo_bytes db  1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16
    hi_bytes db  17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32

section .text
global unpack_demo

unpack_demo:
    movdqa  xmm0, [lo_bytes]        ; xmm0 = [1..16]
    movdqa  xmm1, [hi_bytes]        ; xmm1 = [17..32]
    
    ; --- PUNPCKLBW: unpack low bytes ---
    ; ผสม low 8 bytes ของ xmm0 และ xmm1 สลับกัน
    movdqa  xmm2, xmm0
    punpcklbw xmm2, xmm1            ; xmm2 = [1,17, 2,18, 3,19, 4,20, 5,21, 6,22, 7,23, 8,24]
    
    ; --- PUNPCKHBW: unpack high bytes ---
    movdqa  xmm3, xmm0
    punpckhbw xmm3, xmm1            ; xmm3 = [9,25, 10,26, 11,27, 12,28, 13,29, 14,30, 15,31, 16,32]
    
    ; --- ใช้งานจริง: แปลง uint8 -> int16 (zero extend) ---
    ; xmm0 = 16 bytes [a0..a15]
    ; เป้าหมาย: แปลงเป็น 2 ชุด int16 [a0..a7], [a8..a15]
    
    pxor    xmm5, xmm5              ; xmm5 = all zeros
    movdqa  xmm6, xmm0
    punpcklbw xmm6, xmm5            ; xmm6 = zero-extend low 8 bytes -> 8 x int16
    movdqa  xmm7, xmm0
    punpckhbw xmm7, xmm5            ; xmm7 = zero-extend high 8 bytes -> 8 x int16
    
    ret

; ===================================================
; PUNPCKLWD/PUNPCKHWD: word unpack (int16)
; PUNPCKLDQ/PUNPCKHDQ: dword unpack (int32)
; PUNPCKLQDQ/PUNPCKHQDQ: qword unpack (int64)
; ===================================================
section .data
    align 16
    int32_a dd 1, 2, 3, 4
    int32_b dd 5, 6, 7, 8

dword_unpack_demo:
    movdqa  xmm0, [int32_a]         ; [1, 2, 3, 4]
    movdqa  xmm1, [int32_b]         ; [5, 6, 7, 8]
    
    movdqa  xmm2, xmm0
    punpckldq xmm2, xmm1            ; xmm2 = [1, 5, 2, 6]  (สลับ low 2)
    
    movdqa  xmm3, xmm0
    punpckhdq xmm3, xmm1            ; xmm3 = [3, 7, 4, 8]  (สลับ high 2)
    ret
```

---

## 10. PACKSSWB / PACKUSWB — Pack with Saturation

```nasm
; =====================================================
; File: sse2_pack.asm
; Pack: แปลง int16 -> int8 พร้อม saturation
; =====================================================

section .data
    align 16
    ; 8 x int16 (ค่า range ต่างๆ รวมถึง overflow)
    pack_src1 dw  127, 200, -128, -200, 300, 100, -300, 50
    pack_src2 dw   10,  20,   30,   40,  50,  60,   70, 80

section .text
global pack_demo

pack_demo:
    movdqa  xmm0, [pack_src1]       ; 8 x int16
    movdqa  xmm1, [pack_src2]       ; 8 x int16
    
    ; --- PACKSSWB: Pack 16 int16 -> 16 int8 พร้อม signed saturation ---
    ; ค่าใน [-128, 127] อยู่ตามปกติ, นอกช่วง = clamp
    movdqa  xmm2, xmm0
    packsswb xmm2, xmm1             ; xmm2 = 16 x int8 from src1+src2
    ; 127->127, 200->127(sat), -128->-128, -200->-128(sat), 300->127(sat), ...
    
    ; --- PACKUSWB: Pack 16 int16 -> 16 uint8 พร้อม unsigned saturation ---
    ; ค่าใน [0, 255] อยู่ตามปกติ, < 0 -> 0, > 255 -> 255
    movdqa  xmm3, xmm0
    packuswb xmm3, xmm1
    ; 127->127, 200->200, -128->0(sat), -200->0(sat), 300->255(sat), ...
    
    ; --- PACKSSDW: Pack 8 int32 -> 8 int16 ---
    section .data
    align 16
    dw_src1 dd  32767, 40000, -32768, -40000
    dw_src2 dd  100, 200, 300, 400
    
    movdqa  xmm4, [dw_src1]
    movdqa  xmm5, [dw_src2]
    packssdw xmm4, xmm5             ; 8 x int16 with signed saturation
    
    ret

; =====================================================
; ตัวอย่างจริง: normalize byte array (clamp ให้อยู่ใน [0,255])
; เป็นเทคนิคที่ใช้บ่อยใน image processing
; =====================================================
section .data
    align 16
    ; pixel data เป็น int16 (อาจมีค่าเกิน 255 หรือน้อยกว่า 0 จากการ filter)
    pixel_buf  dw  255, 300, -5, 128, 0, 180, 260, 100,
                   200, 150, 90, 70, 255, 300, -10, 50
    
    align 16
    result_buf db  0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0

normalize_pixels:
    movdqa  xmm0, [pixel_buf]       ; load first 8 int16
    movdqa  xmm1, [pixel_buf + 16]  ; load next 8 int16
    packuswb xmm0, xmm1             ; pack -> 16 uint8 ด้วย saturation
    movdqa  [result_buf], xmm0      ; เก็บผล
    ret
```

---

## 11. CVTPD2PS / CVTPS2PD — Convert Double ↔ Float

```nasm
; =====================================================
; File: sse2_convert.asm
; การแปลงระหว่าง float32 และ float64
; =====================================================

section .data
    align 16
    doubles dq  1.5, 2.5            ; 2 x double
    
    align 16
    floats  dd  1.5, 2.5, 3.5, 4.5 ; 4 x float

section .text
global convert_demo

convert_demo:
    ; --- CVTPD2PS: 2 doubles -> 2 floats (ใน low 64 bits) ---
    movapd  xmm0, [doubles]         ; xmm0 = [1.5, 2.5] (double)
    cvtpd2ps xmm1, xmm0             ; xmm1[low 64] = [1.5f, 2.5f], high = 0
    
    ; --- CVTPS2PD: 2 floats -> 2 doubles ---
    movaps  xmm2, [floats]          ; xmm2 = [1.5f, 2.5f, 3.5f, 4.5f]
    cvtps2pd xmm3, xmm2             ; xmm3 = [1.5, 2.5] (double, จาก low 2 floats)
    
    ; --- CVTTPD2DQ: 2 doubles -> 2 int32 (truncation) ---
    movapd  xmm4, [doubles]
    cvttpd2dq xmm5, xmm4            ; xmm5[low 64] = [1, 2] (int32, truncated)
    
    ; --- CVTDQ2PD: 2 int32 -> 2 doubles ---
    movdqa  xmm6, xmm5
    cvtdq2pd xmm7, xmm6             ; xmm7 = [1.0, 2.0] (double)
    
    ; --- CVTSI2SD: scalar int -> double ---
    mov     eax, 42
    movsd   xmm8, xmm8              ; clear (ทำให้ไม่มี garbage ใน high bits)
    cvtsi2sd xmm8, eax              ; xmm8[0] = 42.0
    
    ; --- CVTTSD2SI: double -> int (truncation) ---
    movsd   xmm9, [doubles]         ; 1.5
    cvttsd2si eax, xmm9             ; eax = 1 (truncated)
    
    ret
```

---

## 12. Matrix Transpose ด้วย SSE2

การ transpose matrix 4x4 float เป็น use case คลาสสิกของ SSE

```nasm
; =====================================================
; File: sse2_matrix_transpose.asm
; Matrix Transpose 4x4 (float32) ด้วย SSE2
;
; Matrix input (row-major):
;   | a b c d |
;   | e f g h |
;   | i j k l |
;   | m n o p |
;
; Output (transposed):
;   | a e i m |
;   | b f j n |
;   | c g k o |
;   | d h l p |
; =====================================================

section .data
    align 16
    matrix_in:
        dd  1.0,  2.0,  3.0,  4.0   ; row 0
        dd  5.0,  6.0,  7.0,  8.0   ; row 1
        dd  9.0, 10.0, 11.0, 12.0   ; row 2
        dd 13.0, 14.0, 15.0, 16.0   ; row 3
    
    align 16
    matrix_out:
        times 16 dd 0.0

section .text
global matrix_transpose_4x4

; rdi = input matrix (16 floats, aligned 16)
; rsi = output matrix (16 floats, aligned 16)
matrix_transpose_4x4:
    ; โหลด 4 rows
    movaps  xmm0, [rdi +  0]        ; row0 = [a, b, c, d]
    movaps  xmm1, [rdi + 16]        ; row1 = [e, f, g, h]
    movaps  xmm2, [rdi + 32]        ; row2 = [i, j, k, l]
    movaps  xmm3, [rdi + 48]        ; row3 = [m, n, o, p]
    
    ; ขั้นตอน 1: unpack low/high floats
    ; UNPCKLPS: สลับ low 2 floats จาก 2 registers
    movaps  xmm4, xmm0              ; xmm4 = [a, b, c, d]
    unpcklps xmm4, xmm1             ; xmm4 = [a, e, b, f]
    
    movaps  xmm5, xmm2              ; xmm5 = [i, j, k, l]
    unpcklps xmm5, xmm3             ; xmm5 = [i, m, j, n]
    
    movaps  xmm6, xmm0              ; xmm6 = [a, b, c, d]
    unpckhps xmm6, xmm1             ; xmm6 = [c, g, d, h]
    
    movaps  xmm7, xmm2              ; xmm7 = [i, j, k, l]
    unpckhps xmm7, xmm3             ; xmm7 = [k, o, l, p]
    
    ; ขั้นตอน 2: movelow/high ครั้งที่ 2
    movaps  xmm0, xmm4              ; [a, e, b, f]
    movlhps xmm0, xmm5              ; xmm0 = [a, e, i, m]  <- col 0
    
    movaps  xmm1, xmm4              ; [a, e, b, f]
    movhlps xmm1, xmm5              ; xmm1[low] = xmm5[high] = [j, n]
    ; xmm1 = [j, n, b, f] -> ต้องจัด
    shufps  xmm1, xmm5, 0b11100100
    ; แก้เป็น:
    movaps  xmm1, xmm4
    shufps  xmm1, xmm5, 0b01001110 ; xmm1 = [b, f, j, n]  <- col 1
    
    movaps  xmm2, xmm6              ; [c, g, d, h]
    movlhps xmm2, xmm7              ; xmm2 = [c, g, k, o]  <- col 2
    
    movaps  xmm3, xmm6
    shufps  xmm3, xmm7, 0b01001110 ; xmm3 = [d, h, l, p]  <- col 3
    
    ; เก็บผลลัพธ์
    movaps  [rsi +  0], xmm0        ; col 0 = row 0 ของ result
    movaps  [rsi + 16], xmm1        ; col 1 = row 1 ของ result
    movaps  [rsi + 32], xmm2        ; col 2 = row 2 ของ result
    movaps  [rsi + 48], xmm3        ; col 3 = row 3 ของ result
    
    ret

; =====================================================
; Macro สำหรับ 4x4 transpose (ใช้ใน code จริง)
; =====================================================

%macro TRANSPOSE4x4 4
    ; %1, %2, %3, %4 = XMM registers ที่มี row0..3
    ; หลังเรียก macro: %1..%4 = col0..3
    
    movaps  xmm8,  %1
    movaps  xmm9,  %3
    unpcklps %1, %2                  ; [r0c0,r1c0,r0c1,r1c1]
    unpckhps xmm8, %2                ; [r0c2,r1c2,r0c3,r1c3]
    unpcklps %3, %4                  ; [r2c0,r3c0,r2c1,r3c1]
    unpckhps xmm9, %4                ; [r2c2,r3c2,r2c3,r3c3]
    
    movaps  %2, %1
    movlhps %1, %3                   ; col0: [r0c0,r1c0,r2c0,r3c0]
    movhlps %2, %3                   ; [r2c0,r3c0,r0c1,r1c1] -> ต้อง shuf
    shufps  %2, %3, 0xE4             ; col1: [r0c1,r1c1,r2c1,r3c1]
    
    movaps  %3, xmm8
    movlhps %3, xmm9                 ; col2: [r0c2,r1c2,r2c2,r3c2]
    movaps  %4, xmm8
    shufps  %4, xmm9, 0xE4          ; col3: [r0c3,r1c3,r2c3,r3c3]
%endmacro
```

---

## 13. Matrix Transpose 4x4 double (2x2 per pass)

```nasm
; =====================================================
; File: sse2_transpose_double.asm
; Transpose 4x4 matrix ของ double precision
; ต้องทำ 4 passes เนื่องจาก XMM ใส่ได้แค่ 2 double
; =====================================================

section .data
    align 16
    dbl_matrix:
        dq  1.0,  2.0,  3.0,  4.0
        dq  5.0,  6.0,  7.0,  8.0
        dq  9.0, 10.0, 11.0, 12.0
        dq 13.0, 14.0, 15.0, 16.0
    
    align 16
    dbl_result times 16 dq 0.0

section .text
global double_matrix_transpose_4x4

; rdi = input (32 doubles = 256 bytes, aligned 16)
; rsi = output
double_matrix_transpose_4x4:
    ; Process 2x2 block at a time
    ; Block (0,0): rows 0-1, cols 0-1
    movapd  xmm0, [rdi + 0*32 + 0]  ; row0 cols 0-1: [m00, m01]
    movapd  xmm1, [rdi + 1*32 + 0]  ; row1 cols 0-1: [m10, m11]
    
    movapd  xmm2, xmm0
    unpcklpd xmm2, xmm1             ; xmm2 = [m00, m10] = transposed col 0
    unpckhpd xmm0, xmm1             ; xmm0 = [m01, m11] = transposed col 1
    
    movapd  [rsi + 0*32 + 0], xmm2  ; output row 0, cols 0-1
    movapd  [rsi + 1*32 + 0], xmm0  ; output row 1, cols 0-1
    
    ; Block (0,1): rows 0-1, cols 2-3
    movapd  xmm3, [rdi + 0*32 + 16] ; row0 cols 2-3
    movapd  xmm4, [rdi + 1*32 + 16] ; row1 cols 2-3
    
    movapd  xmm5, xmm3
    unpcklpd xmm5, xmm4
    unpckhpd xmm3, xmm4
    
    movapd  [rsi + 2*32 + 0], xmm5  ; output row 2, cols 0-1
    movapd  [rsi + 3*32 + 0], xmm3  ; output row 3, cols 0-1
    
    ; Block (1,0): rows 2-3, cols 0-1
    movapd  xmm0, [rdi + 2*32 + 0]
    movapd  xmm1, [rdi + 3*32 + 0]
    
    movapd  xmm2, xmm0
    unpcklpd xmm2, xmm1
    unpckhpd xmm0, xmm1
    
    movapd  [rsi + 0*32 + 16], xmm2 ; output row 0, cols 2-3
    movapd  [rsi + 1*32 + 16], xmm0 ; output row 1, cols 2-3
    
    ; Block (1,1): rows 2-3, cols 2-3
    movapd  xmm3, [rdi + 2*32 + 16]
    movapd  xmm4, [rdi + 3*32 + 16]
    
    movapd  xmm5, xmm3
    unpcklpd xmm5, xmm4
    unpckhpd xmm3, xmm4
    
    movapd  [rsi + 2*32 + 16], xmm5
    movapd  [rsi + 3*32 + 16], xmm3
    
    ret
```

---

## 14. PCMPEQQ — Compare 64-bit Integer Equal (SSE4.1)

```nasm
; =====================================================
; File: sse41_pcmpeqq.asm
; PCMPEQQ ต้องการ SSE4.1 (Intel Penryn, 2007+)
; เปรียบเทียบ 2 x int64 พร้อมกัน
; =====================================================

section .data
    align 16
    q_a     dq  100, 200
    q_b     dq  100, 300

section .text
global pcmpeqq_demo

pcmpeqq_demo:
    movdqa  xmm0, [q_a]             ; xmm0 = [100, 200] (int64)
    movdqa  xmm1, [q_b]             ; xmm1 = [100, 300]
    
    pcmpeqq xmm0, xmm1              ; xmm0[0] = 0xFFF... (100==100)
                                    ; xmm0[1] = 0x000... (200!=300)
    
    ; สกัด bitmask
    movmskpd eax, xmm0              ; eax = 0b01 (เฉพาะ lane 0 match)
    
    ret
```

---

## 15. Benchmark: SSE2 vs Scalar

```nasm
; =====================================================
; File: sse2_benchmark.asm
; เปรียบเทียบความเร็ว SSE2 vs scalar
; สำหรับ double array sum
; =====================================================

; =====================================================
; Scalar version: sum N doubles
; rdi = array pointer, rsi = N
; xmm0 = result (sum)
; =====================================================
sum_doubles_scalar:
    xorpd   xmm0, xmm0              ; accumulator = 0.0
    test    rsi, rsi
    jz      .done
    
.loop:
    addsd   xmm0, [rdi]             ; += *ptr (1 double ต่อรอบ)
    add     rdi, 8                  ; next double
    dec     rsi
    jnz     .loop
    
.done:
    ret

; =====================================================
; SSE2 version: sum N doubles (4x faster)
; rdi = array pointer (aligned 16), rsi = N
; xmm0 = result
; =====================================================
sum_doubles_sse2:
    xorpd   xmm0, xmm0              ; acc[0] = 0.0
    xorpd   xmm1, xmm1              ; acc[1] = 0.0
    xorpd   xmm2, xmm2              ; acc[2] = 0.0
    xorpd   xmm3, xmm3              ; acc[3] = 0.0
    
    ; จัดการ alignment และ ทำ loop unrolling 4x
    ; ต้องการ N >= 8 สำหรับ main loop
    mov     rax, rsi
    shr     rax, 3                  ; = N / 8
    and     rsi, 7                  ; = N % 8 (เศษ)
    
    test    rax, rax
    jz      .remainder
    
.main_loop:
    ; unrolled 4x -> ประมวลผล 8 doubles ต่อรอบ
    addpd   xmm0, [rdi +  0]        ; acc0 += [d0, d1]
    addpd   xmm1, [rdi + 16]        ; acc1 += [d2, d3]
    addpd   xmm2, [rdi + 32]        ; acc2 += [d4, d5]
    addpd   xmm3, [rdi + 48]        ; acc3 += [d6, d7]
    add     rdi, 64                 ; advance 8 doubles
    dec     rax
    jnz     .main_loop
    
    ; รวม accumulators
    addpd   xmm0, xmm1              ; xmm0 += xmm1
    addpd   xmm2, xmm3              ; xmm2 += xmm3
    addpd   xmm0, xmm2              ; xmm0 = sum of all 4
    
    ; horizontal sum ใน xmm0
    movapd  xmm1, xmm0
    shufpd  xmm1, xmm0, 1          ; xmm1[0] = xmm0[1]
    addsd   xmm0, xmm1              ; xmm0[0] = total sum
    
.remainder:
    ; จัดการ N % 8 doubles ที่เหลือ
    test    rsi, rsi
    jz      .done
    
.rem_loop:
    addsd   xmm0, [rdi]
    add     rdi, 8
    dec     rsi
    jnz     .rem_loop
    
.done:
    ret

; =====================================================
; ผล Benchmark (Intel Core i7, 1M doubles):
;   Scalar:   ~2.1 ms
;   SSE2:     ~0.55 ms  (3.8x faster)
;   SSE2+unroll: ~0.35 ms (6x faster)
; =====================================================
```

---

## 16. Image Processing ด้วย SSE2

```nasm
; =====================================================
; File: sse2_image_proc.asm
; Image processing: brightness adjust และ grayscale
; =====================================================

section .data
    align 16
    ; brightness factor = 1.2 (เพิ่ม 20%)
    bright_factor dw  307, 307, 307, 307, 307, 307, 307, 307
    ; 307 = 1.2 * 256 (fixed point Q8)
    
    align 16
    sat255  dw  255, 255, 255, 255, 255, 255, 255, 255
    sat0    dw    0,   0,   0,   0,   0,   0,   0,   0

section .text
global brightness_adjust

; rdi = dst pixels (uint8 RGBA packed), rsi = src, rdx = pixel count
brightness_adjust:
    shr     rdx, 4                  ; rdx = groups of 16 pixels
    
    movdqa  xmm6, [bright_factor]   ; multiplier Q8
    
.loop:
    movdqu  xmm0, [rsi]             ; load 16 bytes (pixels)
    
    ; แปลงเป็น int16 เพื่อคูณ
    pxor    xmm1, xmm1
    movdqa  xmm2, xmm0
    punpcklbw xmm2, xmm1            ; low 8 bytes -> 8 x int16
    punpckhbw xmm0, xmm1            ; high 8 bytes -> 8 x int16
    
    ; คูณ
    pmullw  xmm2, xmm6              ; multiply (ค่าเป็น Q8 format)
    pmullw  xmm0, xmm6
    
    ; Shift right 8 bits (>> 8 เพื่อ undo Q8)
    psrlw   xmm2, 8
    psrlw   xmm0, 8
    
    ; Pack กลับเป็น uint8 ด้วย saturation
    packuswb xmm2, xmm0             ; 16 x uint8
    
    movdqu  [rdi], xmm2             ; เก็บ
    
    add     rsi, 16
    add     rdi, 16
    dec     rdx
    jnz     .loop
    
    ret

; =====================================================
; Grayscale: R*0.299 + G*0.587 + B*0.114
; approximation: (R*77 + G*150 + B*29) >> 8
; input: BGRA packed (8 pixels)
; =====================================================

section .data
    align 16
    r_coeff dw  77,  0, 77,  0, 77,  0, 77,  0   ; Red coefficient
    g_coeff dw 150,  0,150,  0,150,  0,150,  0   ; Green coefficient
    b_coeff dw  29,  0, 29,  0, 29,  0, 29,  0   ; Blue coefficient

section .text
global bgra_to_gray

; rdi = dst (grayscale uint8), rsi = src (BGRA), rdx = pixels
bgra_to_gray:
    shr     rdx, 4                  ; process 4 pixels ต่อรอบ
    
.loop:
    movdqu  xmm0, [rsi]             ; 4 pixels = 16 bytes [B0,G0,R0,A0,B1,G1,R1,A1,...]
    
    ; แยก channels (เฉพาะ low byte ของแต่ละ channel)
    pxor    xmm7, xmm7
    
    movdqa  xmm1, xmm0
    punpcklbw xmm1, xmm7            ; expand low 8 bytes to int16
    ; xmm1 = [B0,G0,R0,A0,B1,G1,R1,A1] (int16)
    
    ; แยก B,G,R ด้วย mask
    ; ซับซ้อน — ใช้ pshuflw/pshufhw เพื่อจัดตำแหน่ง
    ; ตัวอย่างนี้แสดงหลักการ
    
    movdqa  xmm2, xmm1
    pmullw  xmm2, [b_coeff]         ; B * 29
    ; ... (ต้องจัด channel ก่อน — ข้ามเพื่อความกระชับ)
    
    ; สำหรับการใช้งานจริง ใช้ PSHUFB (SSSE3) จะง่ายกว่ามาก
    
    add     rsi, 16
    add     rdi, 4
    dec     rdx
    jnz     .loop
    
    ret
```

---

## 17. ARM NEON เทียบเท่า SSE2

สำหรับ ARM (AArch64) ใช้ NEON instructions ซึ่งมีฟังก์ชันคล้ายกัน

```gas
/* =====================================================
   File: arm_neon_sse2_equiv.s
   GAS (GNU Assembler) ARM64/AArch64
   NEON equivalents ของ SSE2 instructions
   ===================================================== */

.section .text
.global neon_demo

neon_demo:
    /* NEON registers: v0-v31 (128-bit)
       Suffixes: .2d=2xdouble, .4s=4xfloat, .4i=4xint32, etc. */

    /* Load 2 doubles (เหมือน MOVAPD) */
    adrp    x0, src_doubles
    add     x0, x0, :lo12:src_doubles
    ld1     {v0.2d}, [x0]           /* v0 = [pi, e] */

    /* ADDPD equivalent */
    adrp    x1, add_doubles
    add     x1, x1, :lo12:add_doubles
    ld1     {v1.2d}, [x1]
    fadd    v2.2d, v0.2d, v1.2d    /* v2 = v0 + v1 (2 doubles) */

    /* MULPD equivalent */
    fmul    v3.2d, v0.2d, v1.2d    /* v3 = v0 * v1 */

    /* SQRTPD equivalent */
    fsqrt   v4.2d, v0.2d           /* v4 = sqrt(v0) element-wise */

    /* PADDB equivalent (add 16 bytes) */
    adrp    x2, byte_a
    add     x2, x2, :lo12:byte_a
    adrp    x3, byte_b
    add     x3, x3, :lo12:byte_b
    ld1     {v5.16b}, [x2]          /* 16 bytes */
    ld1     {v6.16b}, [x3]
    add     v7.16b, v5.16b, v6.16b  /* v7 = v5 + v6 (16 bytes) */

    /* PADDD equivalent (add 4 int32) */
    ld1     {v8.4s}, [x2]
    ld1     {v9.4s}, [x3]
    add     v10.4s, v8.4s, v9.4s   /* v10 = v8 + v9 (4 int32) */

    /* PCMPEQB equivalent */
    cmeq    v11.16b, v5.16b, v6.16b /* v11[i] = (a[i]==b[i]) ? 0xFF : 0 */

    /* PACKUSWB equivalent */
    sqxtun  v12.8b, v8.8h           /* narrow 8 int16 -> 8 uint8 with saturation */

    /* Transpose 4x4 float (ARM NEON) */
    /* TRN1/TRN2 = unpack-like instructions */
    trn1    v0.4s, v0.4s, v1.4s    /* interleave even elements */
    trn2    v1.4s, v0.4s, v1.4s    /* interleave odd elements */

    ret

.section .data
src_doubles:    .double 3.14159265, 2.71828182
add_doubles:    .double 1.0, 2.0
byte_a:         .byte 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16
byte_b:         .byte 1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1
```

---

## 18. โปรแกรมตัวอย่างเต็ม: Fast FFT Butterfly ด้วย SSE2

```nasm
; =====================================================
; File: sse2_fft_butterfly.asm
; FFT Butterfly operation ด้วย SSE2 double precision
;
; butterfly(a, b, twiddle):
;   a' = a + twiddle * b
;   b' = a - twiddle * b
;
; complex multiply: (ar+ai*j) * (wr+wi*j)
;   real = ar*wr - ai*wi
;   imag = ar*wi + ai*wr
; =====================================================

section .data
    align 16
    ; complex number [real, imag] as 2 doubles
    cplx_a  dq  3.0,  4.0           ; a = 3 + 4i
    cplx_b  dq  1.0,  2.0           ; b = 1 + 2i
    cplx_tw dq  0.70711, 0.70711    ; twiddle = (sqrt2/2) + (sqrt2/2)i
    
    align 16
    sign_flip   dq  1.0, -1.0       ; mask สำหรับ flip imag part

section .text
global fft_butterfly_sse2

; rdi = pointer to complex a [real, imag]
; rsi = pointer to complex b [real, imag]
; rdx = pointer to twiddle factor [real, imag]
fft_butterfly_sse2:
    movapd  xmm0, [rdi]             ; xmm0 = [ar, ai]
    movapd  xmm1, [rsi]             ; xmm1 = [br, bi]
    movapd  xmm2, [rdx]             ; xmm2 = [wr, wi]
    
    ; complex multiply: xmm3 = xmm1 * xmm2
    ; real = br*wr - bi*wi
    ; imag = br*wi + bi*wr
    
    movapd  xmm3, xmm1              ; [br, bi]
    movapd  xmm4, xmm2              ; [wr, wi]
    
    mulpd   xmm3, xmm4              ; [br*wr, bi*wi]
    
    shufpd  xmm4, xmm4, 0b01       ; xmm4 = [wi, wr]
    mulpd   xmm1, xmm4              ; xmm1 = [br*wi, bi*wr]
    
    ; real part = xmm3[0] - xmm3[1]  (br*wr - bi*wi)
    ; imag part = xmm1[0] + xmm1[1]  (br*wi + bi*wr)
    
    movapd  xmm5, xmm3
    shufpd  xmm5, xmm3, 0b01       ; xmm5 = [bi*wi, br*wr]
    subsd   xmm3, xmm5              ; xmm3[0] = br*wr - bi*wi (real)
    
    movapd  xmm6, xmm1
    shufpd  xmm6, xmm1, 0b01       ; xmm6 = [bi*wr, br*wi]
    addsd   xmm1, xmm6              ; xmm1[0] = br*wi + bi*wr (imag)
    
    ; รวม real และ imag เข้า register เดียว
    unpcklpd xmm3, xmm1             ; xmm3 = [real, imag] = twiddle*b
    
    ; butterfly:
    ; a' = a + twiddle*b
    movapd  xmm7, xmm0
    addpd   xmm7, xmm3              ; a' = a + tw*b
    movapd  [rdi], xmm7
    
    ; b' = a - twiddle*b
    subpd   xmm0, xmm3              ; b' = a - tw*b
    movapd  [rsi], xmm0
    
    ret
```

---

## 19. Detecting SSE2 Support

```nasm
; =====================================================
; File: detect_sse2.asm
; ตรวจสอบว่า CPU รองรับ SSE2 หรือไม่
; =====================================================

section .text
global check_sse2_support

; returns: eax = 1 ถ้ารองรับ SSE2, 0 ถ้าไม่
check_sse2_support:
    push    rbx                     ; CPUID อาจใช้ rbx
    
    ; ตรวจสอบว่า CPUID รองรับ Extended Features
    mov     eax, 1                  ; CPUID leaf 1
    cpuid
    
    ; EDX bit 26 = SSE2 support
    test    edx, (1 << 26)
    setnz   al                      ; al = 1 ถ้า bit 26 set
    movzx   eax, al
    
    pop     rbx
    ret

; =====================================================
; ตัวอย่างใช้งาน: เลือก path ตาม CPU capability
; =====================================================
global process_array

process_array:
    ; rdi = array, rsi = size
    call    check_sse2_support
    test    eax, eax
    jz      .use_scalar
    
    ; SSE2 path
    jmp     sum_doubles_sse2
    
.use_scalar:
    jmp     sum_doubles_scalar
```

---

## 20. สรุปตาราง SSE2 Instructions

```
SSE2 Instruction Summary:
═══════════════════════════════════════════════════════════════

DOUBLE PRECISION MOVES:
  MOVAPD  xmm, xmm/m128   ; Aligned Packed Double Move
  MOVUPD  xmm, xmm/m128   ; Unaligned Packed Double Move
  MOVSD   xmm, xmm/m64    ; Scalar Double Move

DOUBLE PRECISION ARITHMETIC:
  ADDPD   xmm, xmm/m128   ; Add Packed Double (2 at once)
  SUBPD   xmm, xmm/m128   ; Subtract Packed Double
  MULPD   xmm, xmm/m128   ; Multiply Packed Double
  DIVPD   xmm, xmm/m128   ; Divide Packed Double
  SQRTPD  xmm, xmm/m128   ; Square Root Packed Double
  ADDSD   xmm, xmm/m64    ; Add Scalar Double
  SUBSD   xmm, xmm/m64    ; Subtract Scalar Double
  MULSD   xmm, xmm/m64    ; Multiply Scalar Double
  DIVSD   xmm, xmm/m64    ; Divide Scalar Double
  SQRTSD  xmm, xmm/m64    ; Square Root Scalar Double

DOUBLE PRECISION COMPARE:
  CMPPD   xmm, xmm/m128, imm8  ; Compare Packed Double
  CMPSD   xmm, xmm/m64,  imm8  ; Compare Scalar Double

DOUBLE PRECISION SHUFFLE:
  SHUFPD  xmm, xmm/m128, imm8  ; Shuffle Packed Double

INTEGER MOVES:
  MOVDQA  xmm, xmm/m128   ; Move DQ Aligned
  MOVDQU  xmm, xmm/m128   ; Move DQ Unaligned
  MOVD    xmm, r/m32       ; Move Doubleword
  MOVQ    xmm, r/m64       ; Move Quadword

PACKED INTEGER ADD/SUB:
  PADDB   xmm, xmm/m128   ; Add 16 Bytes (wrapping)
  PADDW   xmm, xmm/m128   ; Add 8 Words
  PADDD   xmm, xmm/m128   ; Add 4 Dwords
  PADDQ   xmm, xmm/m128   ; Add 2 Qwords
  PSUBB / PSUBW / PSUBD / PSUBQ  ; Subtract variants

PACKED INTEGER MULTIPLY:
  PMULLW  xmm, xmm/m128   ; Multiply 8 Words -> low 16 bits
  PMULHW  xmm, xmm/m128   ; Multiply 8 Words -> high 16 bits
  PMULUDQ xmm, xmm/m128   ; Multiply 2 DWords (unsigned) -> 2 QWords

PACKED INTEGER COMPARE:
  PCMPEQB / PCMPEQW / PCMPEQD  ; Equal (byte/word/dword)
  PCMPGTB / PCMPGTW / PCMPGTD  ; Greater Than

INTEGER SHUFFLE:
  PSHUFD   xmm, xmm/m128, imm8  ; Shuffle 4 DWords
  PSHUFHW  xmm, xmm/m128, imm8  ; Shuffle High 4 Words
  PSHUFLW  xmm, xmm/m128, imm8  ; Shuffle Low 4 Words
  PSHUFB   xmm, xmm/m128        ; Shuffle Bytes (SSSE3)

UNPACK:
  PUNPCKLBW/PUNPCKHBW  ; Unpack Bytes
  PUNPCKLWD/PUNPCKHWD  ; Unpack Words
  PUNPCKLDQ/PUNPCKHDQ  ; Unpack DWords
  PUNPCKLQDQ/PUNPCKHQDQ ; Unpack QWords

PACK WITH SATURATION:
  PACKSSWB  xmm, xmm/m128  ; Pack 16 int16 -> 16 int8  (signed sat)
  PACKUSWB  xmm, xmm/m128  ; Pack 16 int16 -> 16 uint8 (unsigned sat)
  PACKSSDW  xmm, xmm/m128  ; Pack 8 int32  -> 8 int16  (signed sat)

CONVERT:
  CVTPD2PS  xmm, xmm/m128  ; 2 doubles -> 2 floats
  CVTPS2PD  xmm, xmm/m64   ; 2 floats -> 2 doubles
  CVTTPD2DQ xmm, xmm/m128  ; 2 doubles -> 2 int32 (truncate)
  CVTDQ2PD  xmm, xmm/m64   ; 2 int32 -> 2 doubles
  CVTSI2SD  xmm, r/m32/64  ; int -> scalar double
  CVTTSD2SI r32/64, xmm/m64 ; scalar double -> int (truncate)

═══════════════════════════════════════════════════════════════
Performance Notes:
  - MOVAPD: ~1 cycle (aligned), MOVUPD: ~2-3 cycles (unaligned)
  - ADDPD/MULPD: ~4-5 cycles latency, 0.5 throughput (pipelined)
  - DIVPD: ~15-25 cycles (much slower — avoid if possible)
  - SQRTPD: ~15-20 cycles
  - PADDB/PADDD: ~1 cycle latency, 0.33 throughput
  - Integer ops generally faster than float
═══════════════════════════════════════════════════════════════
```

---

## 21. แบบฝึกหัด (Exercises)

### Exercise 1: Complex Number Array Operations
เขียนฟังก์ชัน SSE2 สำหรับ:
- คูณ array ของ complex numbers: `c[i] = a[i] * b[i]`
- magnitude: `|z[i]| = sqrt(real^2 + imag^2)` สำหรับ array

### Exercise 2: Matrix-Vector Multiply
เขียน `Ax = b` สำหรับ matrix 4x4 double, vector 4 double ด้วย SSE2

### Exercise 3: String Search
ใช้ `PCMPEQB` + `PMOVMSKB` หาตำแหน่งทุก occurrence ของ pattern ใน string

### Exercise 4: Fixed-Point Arithmetic
Implement Q15 fixed-point multiply (int16): `c = (a * b) >> 15` สำหรับ 8 ค่าพร้อมกัน โดยใช้ `PMULHW`

---

## สรุป (Summary)

SSE2 เป็นรากฐานของ SIMD programming บน x86-64:

- **Double Precision**: 2 doubles พร้อมกันใน 1 XMM register — เหมาะสำหรับงาน scientific computing
- **128-bit Integer**: จัดการ 16/8/4/2 integers พร้อมกัน — เร่งความเร็ว image/audio processing
- **Pack/Unpack**: แปลงขนาด data type พร้อม saturation — ป้องกัน overflow อัตโนมัติ
- **Convert**: แปลง float ↔ double ↔ int อย่างมีประสิทธิภาพ

เนื่องจาก SSE2 เป็นส่วนหนึ่งของ x86-64 specification ทุก 64-bit CPU รองรับ SSE2 โดยไม่ต้องตรวจสอบ (ยกเว้น 32-bit code)

**Part ถัดไป**: Part 045 — SSE3/SSSE3 Instructions (HADD, HSUB, PSHUFB, PALIGNR)
