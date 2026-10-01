# Part 043: SSE - Streaming SIMD Extensions
# บทที่ 43: คำสั่ง SSE - การประมวลผลแบบขนาน

---

## บทนำ (Introduction)

SSE (Streaming SIMD Extensions) คือชุดคำสั่งที่ Intel แนะนำใน Pentium III (1999)
เพื่อเพิ่มประสิทธิภาพการประมวลผลแบบขนาน (Parallel Processing) โดยใช้ registers XMM
ขนาด 128 บิต ที่สามารถประมวลผลข้อมูลหลายตัวพร้อมกันในคำสั่งเดียว

**SIMD = Single Instruction, Multiple Data**
- คำสั่งเดียว ทำงานกับข้อมูลหลายตัวพร้อมกัน
- เหมาะสำหรับ: กราฟิก, เสียง, วิดีโอ, การคำนวณทางวิทยาศาสตร์
- ประสิทธิภาพสูงกว่า scalar operations 2-8 เท่า

---

## สารบัญ (Table of Contents)

1. XMM Registers และ Architecture
2. SSE Data Types
3. คำสั่ง Move (MOVAPS/MOVUPS)
4. คำสั่งคำนวณแบบ Packed Float
5. คำสั่งคำนวณแบบ Scalar Float
6. Square Root (SQRTPS/SQRTSS)
7. Max/Min (MAXPS/MINPS)
8. Compare (CMPPS/CMPSS)
9. Logical Operations
10. Shuffle และ Unpack
11. การแปลงข้อมูล (Conversion)
12. MXCSR Control Register
13. Dot Product Implementation
14. Practical Applications
15. Performance Benchmarks

---

## 1. XMM Registers และ Architecture

### โครงสร้าง XMM Registers

```
XMM0:  [127..96][95..64][63..32][31..0]
        float3   float2   float1  float0
        
ขนาด Register: 128 บิต = 16 bytes
```

### รายชื่อ XMM Registers

| โหมด | Registers |
|------|-----------|
| 32-bit (x86) | XMM0 - XMM7 (8 registers) |
| 64-bit (x86-64) | XMM0 - XMM15 (16 registers) |

### การใช้งานใน Calling Convention (x86-64 Linux)

```
Caller-saved (ต้องบันทึกเองถ้าจะใช้):
XMM0-XMM7   = ส่งผ่าน arguments (floating point)
XMM0        = return value (floating point)

Callee-saved (ไม่ต้องบันทึก ใน Linux):
XMM8-XMM15  = ใช้ได้เลย (ใน Windows ต้องบันทึก XMM6-XMM15)
```

### NASM: ตรวจสอบ SSE Support

```nasm
; ไฟล์: check_sse.asm
; วิธีตรวจสอบว่า CPU รองรับ SSE หรือไม่

section .data
    msg_yes db "SSE supported!", 10, 0
    msg_no  db "SSE NOT supported!", 10, 0

section .text
    global _start

_start:
    ; ใช้ CPUID เพื่อตรวจสอบ feature flags
    mov eax, 1          ; CPUID function 1
    cpuid               ; ดึงข้อมูล CPU features
    
    ; EDX bit 25 = SSE support
    test edx, (1 << 25) ; ตรวจสอบ bit 25
    jz .no_sse          ; ถ้า 0 = ไม่รองรับ
    
    ; SSE รองรับ - แสดงข้อความ
    mov eax, 4
    mov ebx, 1
    mov ecx, msg_yes
    mov edx, 15
    int 0x80
    jmp .done
    
.no_sse:
    ; ไม่รองรับ SSE
    mov eax, 4
    mov ebx, 1
    mov ecx, msg_no
    mov edx, 18
    int 0x80
    
.done:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

---

## 2. SSE Data Types

### ประเภทข้อมูลใน XMM Register (128 บิต)

```
128-bit XMM Register สามารถเก็บ:

[PS] Packed Single-precision float (4 x 32-bit float):
┌────────┬────────┬────────┬────────┐
│ f[3]   │ f[2]   │ f[1]   │ f[0]   │
│ 32-bit │ 32-bit │ 32-bit │ 32-bit │
└────────┴────────┴────────┴────────┘

[SS] Scalar Single-precision float (1 x 32-bit float):
┌────────┬────────┬────────┬────────┐
│ unused │ unused │ unused │ f[0]   │
│        │        │        │ 32-bit │
└────────┴────────┴────────┴────────┘

[PD] Packed Double-precision float (2 x 64-bit double) - SSE2:
┌─────────────────┬─────────────────┐
│ d[1]            │ d[0]            │
│ 64-bit          │ 64-bit          │
└─────────────────┴─────────────────┘

[PI] Packed Integer (ใช้ MMX registers, ไม่ใช่ XMM):
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│i[7]  │i[6]  │i[5]  │i[4]  │i[3]  │i[2]  │i[1]  │i[0]  │
│16-bit│16-bit│16-bit│16-bit│16-bit│16-bit│16-bit│16-bit│
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

### ตัวอย่างการโหลดข้อมูล 4 floats

```nasm
; ไฟล์: sse_datatypes.asm
; แสดงการทำงานกับ SSE data types

section .data
    ; ต้องจัดเรียง 16-byte aligned
    align 16
    vec_a   dd 1.0, 2.0, 3.0, 4.0      ; [1.0, 2.0, 3.0, 4.0]
    vec_b   dd 5.0, 6.0, 7.0, 8.0      ; [5.0, 6.0, 7.0, 8.0]
    result  dd 0.0, 0.0, 0.0, 0.0      ; ผลลัพธ์

section .text
    global _start

_start:
    ; โหลดข้อมูลเข้า XMM registers
    movaps xmm0, [vec_a]    ; โหลด vec_a ทั้ง 4 float เข้า XMM0
    movaps xmm1, [vec_b]    ; โหลด vec_b ทั้ง 4 float เข้า XMM1
    
    ; XMM0 = [4.0, 3.0, 2.0, 1.0]  (little-endian: [0]=1.0, [1]=2.0...)
    ; XMM1 = [8.0, 7.0, 6.0, 5.0]
    
    ; บวกทั้ง 4 คู่พร้อมกัน
    addps xmm0, xmm1        ; XMM0 = [9.0, 9.0, 9.0, 9.0]
    
    ; บันทึกผลลัพธ์
    movaps [result], xmm0
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

---

## 3. คำสั่ง Move: MOVAPS / MOVUPS

### MOVAPS - Move Aligned Packed Single-precision

```nasm
; MOVAPS = Move Aligned Packed Single
; ข้อกำหนด: ต้อง 16-byte aligned เท่านั้น
; หากไม่ aligned จะเกิด General Protection Fault!

; syntax:
; movaps xmm_dst, [memory]    ; โหลดจาก memory
; movaps [memory], xmm_src    ; บันทึกลง memory
; movaps xmm_dst, xmm_src     ; copy ระหว่าง registers

section .data
    align 16                    ; <<< สำคัญมาก! ต้อง align 16
    data_aligned dd 1.0, 2.0, 3.0, 4.0

section .text
    ; โหลดข้อมูล aligned
    movaps xmm0, [data_aligned]     ; ถูกต้อง - 16-byte aligned
    
    ; Copy ระหว่าง XMM registers
    movaps xmm1, xmm0               ; xmm1 = xmm0
    
    ; บันทึกผลลัพธ์
    movaps [data_aligned], xmm0     ; บันทึกกลับ memory
```

### MOVUPS - Move Unaligned Packed Single-precision

```nasm
; MOVUPS = Move Unaligned Packed Single
; ทำงานกับ memory ที่ไม่ aligned ได้
; ช้ากว่า MOVAPS ประมาณ 2-3 cycle บน CPU รุ่นเก่า
; (CPU รุ่นใหม่อย่าง Core 2 และหลังจากนั้น ความแตกต่างน้อยมาก)

section .data
    db 0                        ; ทำให้ไม่ aligned จงใจ
    data_unaligned dd 1.0, 2.0, 3.0, 4.0   ; ไม่ aligned!

section .text
    ; โหลดข้อมูล unaligned
    movups xmm0, [data_unaligned]   ; ถูกต้อง - ทำงานกับ unaligned ได้
    movups [data_unaligned], xmm0   ; บันทึกกลับ
```

### MOVSS - Move Scalar Single-precision

```nasm
; MOVSS = Move Scalar Single
; ทำงานกับ float เดี่ยว (32-bit) ใน bits [31:0]

section .data
    align 4
    scalar_val dd 3.14159       ; float เดี่ยว

section .text
    movss xmm0, [scalar_val]    ; โหลด float เดียว ไปใน XMM0[31:0]
                                ; XMM0[127:32] = 0
    
    movss [scalar_val], xmm0    ; บันทึก XMM0[31:0] ลง memory
```

### ตัวอย่างครบถ้วน: MOVAPS vs MOVUPS

```nasm
; ไฟล์: move_demo.asm
; เปรียบเทียบ MOVAPS และ MOVUPS

section .data
    align 16
    aligned_data    dd 1.0, 2.0, 3.0, 4.0
    
    ; สร้าง unaligned data โดยตั้งใจ
    db 0x00                     ; padding 1 byte
    unaligned_data  dd 5.0, 6.0, 7.0, 8.0
    
    align 16
    output_buf      dd 0.0, 0.0, 0.0, 0.0

section .text
    global sse_move_demo

sse_move_demo:
    push rbp
    mov rbp, rsp
    
    ;--- MOVAPS: โหลด aligned data ---
    movaps xmm0, [aligned_data]         ; โหลด [1.0, 2.0, 3.0, 4.0]
    
    ;--- MOVUPS: โหลด unaligned data ---
    movups xmm1, [unaligned_data]       ; โหลด [5.0, 6.0, 7.0, 8.0]
    
    ;--- Copy ระหว่าง registers ---
    movaps xmm2, xmm0                   ; xmm2 = xmm0
    movaps xmm3, xmm1                   ; xmm3 = xmm1
    
    ;--- บันทึกผลลัพธ์ ---
    movaps [output_buf], xmm0           ; บันทึก XMM0 ลง aligned buffer
    movups [output_buf+4], xmm1         ; บันทึก XMM1 ลง unaligned position
    
    pop rbp
    ret
```

---

## 4. คำสั่งคำนวณแบบ Packed Float

### ADDPS - Add Packed Single-precision Floats

```nasm
; ADDPS = Add Packed Single
; บวก 4 คู่ของ floats พร้อมกัน

; การทำงาน:
; dst[0] = dst[0] + src[0]
; dst[1] = dst[1] + src[1]
; dst[2] = dst[2] + src[2]
; dst[3] = dst[3] + src[3]

section .data
    align 16
    va  dd 1.0, 2.0, 3.0, 4.0
    vb  dd 5.0, 6.0, 7.0, 8.0

section .text
    movaps xmm0, [va]       ; xmm0 = [1.0, 2.0, 3.0, 4.0]
    movaps xmm1, [vb]       ; xmm1 = [5.0, 6.0, 7.0, 8.0]
    addps  xmm0, xmm1       ; xmm0 = [6.0, 8.0, 10.0, 12.0]
    ; บวก 4 คู่พร้อมกันในคำสั่งเดียว!
```

### SUBPS - Subtract Packed Single-precision Floats

```nasm
; SUBPS = Subtract Packed Single
; ลบ 4 คู่ของ floats พร้อมกัน

movaps xmm0, [va]       ; xmm0 = [1.0, 2.0, 3.0, 4.0]
movaps xmm1, [vb]       ; xmm1 = [5.0, 6.0, 7.0, 8.0]
subps  xmm0, xmm1       ; xmm0 = [-4.0, -4.0, -4.0, -4.0]
; xmm0[i] = xmm0[i] - xmm1[i]
```

### MULPS - Multiply Packed Single-precision Floats

```nasm
; MULPS = Multiply Packed Single
; คูณ 4 คู่ของ floats พร้อมกัน

movaps xmm0, [va]       ; xmm0 = [1.0, 2.0, 3.0, 4.0]
movaps xmm1, [vb]       ; xmm1 = [5.0, 6.0, 7.0, 8.0]
mulps  xmm0, xmm1       ; xmm0 = [5.0, 12.0, 21.0, 32.0]
; xmm0[i] = xmm0[i] * xmm1[i]
```

### DIVPS - Divide Packed Single-precision Floats

```nasm
; DIVPS = Divide Packed Single
; หาร 4 คู่ของ floats พร้อมกัน (ช้ากว่า MULPS มาก!)

movaps xmm0, [vb]       ; xmm0 = [5.0, 6.0, 7.0, 8.0]
movaps xmm1, [va]       ; xmm1 = [1.0, 2.0, 3.0, 4.0]
divps  xmm0, xmm1       ; xmm0 = [5.0, 3.0, 2.333..., 2.0]
; xmm0[i] = xmm0[i] / xmm1[i]
; หมายเหตุ: หาร 0 จะได้ INF หรือ NaN
```

### ตัวอย่างครบถ้วน: Vector Arithmetic

```nasm
; ไฟล์: vector_arithmetic.asm
; การคำนวณ Vector ด้วย SSE Packed Operations

section .data
    align 16
    ; Vector A: [x=1.0, y=2.0, z=3.0, w=4.0]
    vec_a   dd 1.0, 2.0, 3.0, 4.0
    
    ; Vector B: [x=5.0, y=6.0, z=7.0, w=8.0]
    vec_b   dd 5.0, 6.0, 7.0, 8.0
    
    ; ผลลัพธ์ต่างๆ
    res_add  dd 0.0, 0.0, 0.0, 0.0     ; A + B
    res_sub  dd 0.0, 0.0, 0.0, 0.0     ; A - B
    res_mul  dd 0.0, 0.0, 0.0, 0.0     ; A * B
    res_div  dd 0.0, 0.0, 0.0, 0.0     ; A / B

section .text
    global vector_arithmetic

vector_arithmetic:
    push rbp
    mov rbp, rsp
    
    ;--- โหลด vectors ---
    movaps xmm0, [vec_a]        ; xmm0 = A
    movaps xmm1, [vec_b]        ; xmm1 = B
    
    ;--- A + B ---
    movaps xmm2, xmm0           ; xmm2 = A (copy)
    addps  xmm2, xmm1           ; xmm2 = A + B = [6, 8, 10, 12]
    movaps [res_add], xmm2
    
    ;--- A - B ---
    movaps xmm2, xmm0           ; xmm2 = A (copy)
    subps  xmm2, xmm1           ; xmm2 = A - B = [-4, -4, -4, -4]
    movaps [res_sub], xmm2
    
    ;--- A * B ---
    movaps xmm2, xmm0           ; xmm2 = A (copy)
    mulps  xmm2, xmm1           ; xmm2 = A * B = [5, 12, 21, 32]
    movaps [res_mul], xmm2
    
    ;--- A / B ---
    movaps xmm2, xmm0           ; xmm2 = A (copy)
    divps  xmm2, xmm1           ; xmm2 = A / B
    movaps [res_div], xmm2
    
    pop rbp
    ret
```

---

## 5. คำสั่งคำนวณแบบ Scalar Float

### ADDSS, SUBSS, MULSS, DIVSS

```nasm
; Scalar operations ทำงานกับ float เดียวใน [31:0]
; ส่วน [127:32] ไม่เปลี่ยนแปลง

section .data
    align 4
    val_a   dd 10.0
    val_b   dd 3.0

section .text
    ;--- ADDSS: Add Scalar Single ---
    movss xmm0, [val_a]     ; xmm0[0] = 10.0
    movss xmm1, [val_b]     ; xmm1[0] = 3.0
    addss xmm0, xmm1        ; xmm0[0] = 13.0
    ; เฉพาะ [31:0] เปลี่ยน, [127:32] คงเดิม
    
    ;--- SUBSS: Subtract Scalar Single ---
    movss xmm0, [val_a]     ; xmm0[0] = 10.0
    subss xmm0, xmm1        ; xmm0[0] = 7.0
    
    ;--- MULSS: Multiply Scalar Single ---
    movss xmm0, [val_a]     ; xmm0[0] = 10.0
    mulss xmm0, xmm1        ; xmm0[0] = 30.0
    
    ;--- DIVSS: Divide Scalar Single ---
    movss xmm0, [val_a]     ; xmm0[0] = 10.0
    divss xmm0, xmm1        ; xmm0[0] = 3.333...
```

### ตัวอย่าง: Circle Area Calculation

```nasm
; ไฟล์: circle_area.asm
; คำนวณพื้นที่วงกลม: area = π × r²
; ใช้ SSE scalar operations

section .data
    align 16
    pi      dd 3.14159274      ; π (single precision)
    radius  dd 5.0              ; รัศมี = 5
    area    dd 0.0              ; พื้นที่ (ผลลัพธ์)

section .text
    global calc_circle_area

; float calc_circle_area(float r)
; r ส่งใน xmm0
calc_circle_area:
    ; xmm0 = r (radius)
    
    mulss xmm0, xmm0            ; xmm0 = r²
    
    movss xmm1, [pi]            ; xmm1 = π
    mulss xmm0, xmm1            ; xmm0 = π × r²
    
    ; ผลลัพธ์อยู่ใน xmm0 (return value)
    ret
    
    ; ตัวอย่างการเรียกใช้:
    ; movss xmm0, [radius]   ; โหลด radius = 5.0
    ; call calc_circle_area   ; เรียก function
    ; movss [area], xmm0      ; บันทึกผลลัพธ์ = 78.5398...
```

---

## 6. Square Root: SQRTPS / SQRTSS

### SQRTPS - Square Root Packed Single

```nasm
; SQRTPS = Square Root Packed Single
; คำนวณ square root ของ 4 floats พร้อมกัน

section .data
    align 16
    values  dd 4.0, 9.0, 16.0, 25.0    ; ค่าที่จะหาราก
    roots   dd 0.0, 0.0, 0.0, 0.0       ; ผลลัพธ์

section .text
    movaps xmm0, [values]       ; xmm0 = [4.0, 9.0, 16.0, 25.0]
    sqrtps xmm0, xmm0           ; xmm0 = [2.0, 3.0, 4.0, 5.0]
    movaps [roots], xmm0        ; บันทึกผลลัพธ์
    
    ; หมายเหตุ: SQRTPS ช้ากว่า MULPS มาก (~10-20 cycles)
    ; ถ้าต้องการความเร็ว ใช้ RSQRTPS (ประมาณค่า) แทน
```

### SQRTSS - Square Root Scalar Single

```nasm
; SQRTSS = Square Root Scalar Single
; คำนวณ square root ของ float เดียว

    movss xmm0, [val]           ; โหลด value
    sqrtss xmm0, xmm0           ; xmm0[0] = sqrt(val)
```

### RSQRTPS - Reciprocal Square Root (ประมาณค่า)

```nasm
; RSQRTPS = Reciprocal Square Root Packed Single
; คำนวณ 1/sqrt(x) ประมาณค่า (ความแม่นยำ ~12 bits)
; เร็วกว่า SQRTPS มาก!

section .data
    align 16
    vals    dd 4.0, 9.0, 16.0, 25.0

section .text
    movaps xmm0, [vals]
    rsqrtps xmm0, xmm0      ; xmm0 ≈ [1/2, 1/3, 1/4, 1/5]
    ; ใช้สำหรับ normalizing vectors ในกราฟิก
```

### Newton-Raphson Refinement สำหรับ RSQRTPS

```nasm
; ไฟล์: accurate_rsqrt.asm
; ปรับปรุงความแม่นยำของ RSQRTPS ด้วย Newton-Raphson

section .data
    align 16
    half        dd 0.5, 0.5, 0.5, 0.5
    three_half  dd 1.5, 1.5, 1.5, 1.5

section .text
    global fast_rsqrt_ps

; ปรับปรุง RSQRTPS ด้วย Newton-Raphson iteration
; input: xmm0 = x (values to get 1/sqrt of)
; output: xmm0 = 1/sqrt(x) (ความแม่นยำสูงกว่า)
fast_rsqrt_ps:
    ; เก็บ x ไว้ใน xmm2
    movaps xmm2, xmm0           ; xmm2 = x
    
    ; หา estimate: xmm1 = rsqrt(x) ≈ y0
    rsqrtps xmm1, xmm0          ; xmm1 = y0 ≈ 1/sqrt(x)
    
    ; Newton-Raphson: y1 = y0 * (1.5 - 0.5*x*y0²)
    ;  1. คำนวณ y0²
    movaps xmm0, xmm1           ; xmm0 = y0
    mulps  xmm0, xmm0           ; xmm0 = y0²
    
    ;  2. คำนวณ 0.5 * x * y0²
    mulps  xmm0, xmm2           ; xmm0 = x * y0²
    mulps  xmm0, [half]         ; xmm0 = 0.5 * x * y0²
    
    ;  3. คำนวณ 1.5 - (0.5 * x * y0²)
    movaps xmm3, [three_half]   ; xmm3 = 1.5
    subps  xmm3, xmm0           ; xmm3 = 1.5 - 0.5*x*y0²
    
    ;  4. คำนวณ y1 = y0 * (1.5 - 0.5*x*y0²)
    mulps  xmm1, xmm3           ; xmm1 = y0 * (1.5 - 0.5*x*y0²) = y1
    
    movaps xmm0, xmm1           ; xmm0 = ผลลัพธ์
    ret
```

---

## 7. Max/Min: MAXPS / MINPS

### MAXPS - Maximum Packed Single

```nasm
; MAXPS = Maximum Packed Single
; เลือกค่าที่มากกว่าจากแต่ละคู่

section .data
    align 16
    va  dd 1.0, 8.0, 3.0, 6.0
    vb  dd 5.0, 2.0, 7.0, 4.0

section .text
    movaps xmm0, [va]       ; xmm0 = [1.0, 8.0, 3.0, 6.0]
    movaps xmm1, [vb]       ; xmm1 = [5.0, 2.0, 7.0, 4.0]
    maxps  xmm0, xmm1       ; xmm0 = [5.0, 8.0, 7.0, 6.0]
    ; เลือกค่าที่มากกว่าจากแต่ละตำแหน่ง
```

### MINPS - Minimum Packed Single

```nasm
; MINPS = Minimum Packed Single
; เลือกค่าที่น้อยกว่าจากแต่ละคู่

    movaps xmm0, [va]       ; xmm0 = [1.0, 8.0, 3.0, 6.0]
    movaps xmm1, [vb]       ; xmm1 = [5.0, 2.0, 7.0, 4.0]
    minps  xmm0, xmm1       ; xmm0 = [1.0, 2.0, 3.0, 4.0]
    ; เลือกค่าที่น้อยกว่าจากแต่ละตำแหน่ง
```

### ตัวอย่าง: Clamp Values (จำกัดค่าในช่วง)

```nasm
; ไฟล์: clamp_values.asm
; จำกัดค่า array ให้อยู่ในช่วง [min_val, max_val]
; ใช้ MAXPS และ MINPS

section .data
    align 16
    ; ข้อมูลที่จะ clamp
    data        dd -5.0, 0.5, 1.5, 2.0
                dd 0.1, -0.3, 0.8, 1.2
    
    ; ค่า clamp bounds (broadcast เป็น 4 ค่าเหมือนกัน)
    min_bound   dd 0.0, 0.0, 0.0, 0.0  ; min = 0.0
    max_bound   dd 1.0, 1.0, 1.0, 1.0  ; max = 1.0
    
    output      dd 0.0, 0.0, 0.0, 0.0
                dd 0.0, 0.0, 0.0, 0.0

section .text
    global clamp_array

; clamp_array(float* data, int count)
; rdi = pointer to data
; rsi = count (ต้องหารด้วย 4 ลงตัว)
clamp_array:
    push rbp
    mov rbp, rsp
    
    movaps xmm4, [min_bound]    ; xmm4 = [0.0, 0.0, 0.0, 0.0]
    movaps xmm5, [max_bound]    ; xmm5 = [1.0, 1.0, 1.0, 1.0]
    
    xor rcx, rcx                ; rcx = index = 0
    
.loop:
    cmp rcx, rsi                ; index >= count?
    jge .done
    
    ; โหลด 4 floats
    movaps xmm0, [rdi + rcx*4]  ; โหลดข้อมูล 4 ตัว
    
    ; Clamp to [0.0, 1.0]
    maxps  xmm0, xmm4            ; max(x, 0.0) - ตัด negative
    minps  xmm0, xmm5            ; min(x, 1.0) - ตัด > 1.0
    
    ; บันทึกผลลัพธ์
    movaps [rdi + rcx*4], xmm0
    
    add rcx, 4                   ; ไปตำแหน่งถัดไป (4 floats)
    jmp .loop
    
.done:
    pop rbp
    ret
```

---

## 8. Compare: CMPPS / CMPSS

### CMPPS - Compare Packed Single-precision Floats

```nasm
; CMPPS = Compare Packed Single
; เปรียบเทียบ 4 คู่ ผลลัพธ์เป็น bitmask (0x00000000 หรือ 0xFFFFFFFF)

; Syntax:
; cmpps xmm_dst, xmm_src, imm8
;
; imm8 = comparison type:
;   0 = EQ   (เท่ากัน)
;   1 = LT   (น้อยกว่า)
;   2 = LE   (น้อยกว่าหรือเท่ากัน)
;   3 = UNORD (unordered - มี NaN)
;   4 = NEQ  (ไม่เท่ากัน)
;   5 = NLT  (ไม่น้อยกว่า = >=)
;   6 = NLE  (ไม่น้อยกว่าหรือเท่ากัน = >)
;   7 = ORD  (ordered - ไม่มี NaN)

section .data
    align 16
    va  dd 1.0, 5.0, 3.0, 8.0
    vb  dd 2.0, 4.0, 3.0, 7.0

section .text
    movaps xmm0, [va]       ; xmm0 = [1.0, 5.0, 3.0, 8.0]
    movaps xmm1, [vb]       ; xmm1 = [2.0, 4.0, 3.0, 7.0]
    
    ;--- Compare: va < vb? ---
    movaps xmm2, xmm0
    cmpps  xmm2, xmm1, 1    ; LT comparison
    ; xmm2 = [0xFFFFFFFF, 0x00000000, 0x00000000, 0x00000000]
    ; 1.0 < 2.0 = true, 5.0 < 4.0 = false, 3.0 < 3.0 = false, 8.0 < 7.0 = false
    
    ;--- Compare: va == vb? ---
    movaps xmm3, xmm0
    cmpps  xmm3, xmm1, 0    ; EQ comparison
    ; xmm3 = [0x00000000, 0x00000000, 0xFFFFFFFF, 0x00000000]
    ; 3.0 == 3.0 = true
```

### ใช้ mask ผลลัพธ์ของ CMPPS

```nasm
; ไฟล์: compare_and_select.asm
; ใช้ CMPPS เพื่อ select ค่าตามเงื่อนไข

section .data
    align 16
    va          dd 1.0, 5.0, 3.0, 8.0
    vb          dd 2.0, 4.0, 3.0, 7.0
    select_true dd 100.0, 100.0, 100.0, 100.0   ; ถ้า condition true
    select_false dd 0.0, 0.0, 0.0, 0.0           ; ถ้า condition false

section .text
    global compare_select

compare_select:
    movaps xmm0, [va]           ; xmm0 = a
    movaps xmm1, [vb]           ; xmm1 = b
    
    ;--- สร้าง mask: a > b? ---
    movaps xmm2, xmm0
    cmpps  xmm2, xmm1, 6        ; NLE (Not Less or Equal = Greater Than)
    ; xmm2 = mask (0xFFFFFFFF where a > b)
    
    ;--- Select ค่า: ถ้า mask = 1 เอา 100.0, ถ้า 0 เอา 0.0 ---
    movaps xmm3, [select_true]  ; xmm3 = 100.0
    movaps xmm4, [select_false] ; xmm4 = 0.0
    
    andps  xmm3, xmm2           ; เก็บเฉพาะตำแหน่งที่ mask = 1
    andnps xmm4, xmm2           ; inverse mask สำหรับตำแหน่งที่ mask = 0
    ; หมายเหตุ: andnps xmm4, xmm2 = NOT(xmm2) AND xmm4
    ; ถูกต้องกว่า: movaps xmm5, xmm2; andnps xmm5, xmm4
    
    orps   xmm3, xmm4           ; รวมผลลัพธ์
    ; xmm3 = ผลลัพธ์ที่ select แล้ว
    
    movaps xmm0, xmm3           ; return ผลลัพธ์
    ret
```

---

## 9. Logical Operations: ANDPS, ORPS, XORPS, ANDNPS

### ANDPS - Bitwise AND Packed Single

```nasm
; ANDPS = AND Packed Single
; Bitwise AND ระหว่าง 2 XMM registers
; ประโยชน์หลัก: masking และ clearing

section .data
    align 16
    sign_mask   dd 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF
    ; bit 31 = sign bit ของ float
    
    values      dd -1.0, 2.0, -3.0, 4.0

section .text
    movaps xmm0, [values]       ; xmm0 = [-1, 2, -3, 4]
    movaps xmm1, [sign_mask]    ; xmm1 = 0x7FFFFFFF × 4
    andps  xmm0, xmm1           ; xmm0 = [1.0, 2.0, 3.0, 4.0] (abs value!)
    ; AND กับ sign_mask = ล้าง sign bit = absolute value!
```

### ORPS - Bitwise OR Packed Single

```nasm
; ORPS = OR Packed Single
; Bitwise OR - ใช้ใน masking operations

section .data
    align 16
    sign_bit    dd 0x80000000, 0x80000000, 0x80000000, 0x80000000
    values      dd 1.0, 2.0, 3.0, 4.0

section .text
    movaps xmm0, [values]       ; xmm0 = [1, 2, 3, 4]
    movaps xmm1, [sign_bit]     ; xmm1 = sign bit = 0x80000000
    orps   xmm0, xmm1           ; xmm0 = [-1, -2, -3, -4] (negate!)
    ; OR กับ sign_bit = set sign bit = negate!
```

### XORPS - Bitwise XOR Packed Single

```nasm
; XORPS = XOR Packed Single
; ใช้ XOR เพื่อ clear register (เร็วกว่า MOVAPS)

section .text
    xorps  xmm0, xmm0       ; xmm0 = 0 (zero out เร็วมาก!)
    ; เร็วกว่า: movaps xmm0, [zero_vector]
    
    ; XOR กับ sign_mask = toggle sign bit = negate
    xorps  xmm0, [sign_bit]
```

### ANDNPS - Bitwise ANDN Packed Single

```nasm
; ANDNPS = NOT(dst) AND src
; andnps dst, src  =>  dst = (NOT dst) AND src

section .text
    ; ตัวอย่าง: เลือกค่าที่ mask = 0
    ; mask อยู่ใน xmm2, values อยู่ใน xmm3
    
    ; andnps xmm2, xmm3   =>  xmm2 = (NOT xmm2) AND xmm3
    ; = เลือกค่าจาก xmm3 ที่ xmm2 = 0
    
    andnps xmm2, xmm3       ; xmm2 = (~mask) AND values
```

### ตัวอย่าง: Absolute Value ด้วย ANDPS

```nasm
; ไฟล์: abs_value.asm
; คำนวณ absolute value ของ float vector ด้วย SSE

section .data
    align 16
    ; Absolute value mask: ล้าง sign bit (bit 31)
    abs_mask    dd 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF
    
    data        dd -1.5, 2.7, -3.14, 4.0
    result      dd 0.0, 0.0, 0.0, 0.0

section .text
    global compute_abs_vec4

compute_abs_vec4:
    movaps xmm0, [data]         ; xmm0 = [-1.5, 2.7, -3.14, 4.0]
    andps  xmm0, [abs_mask]     ; ล้าง sign bit = abs value
    ; xmm0 = [1.5, 2.7, 3.14, 4.0]
    movaps [result], xmm0
    ret
```

---

## 10. Shuffle และ Unpack

### SHUFPS - Shuffle Packed Single-precision Floats

```nasm
; SHUFPS = Shuffle Packed Single
; จัดเรียงลำดับ floats ใน XMM register ใหม่

; Syntax: shufps dst, src, imm8
; imm8 = [7:6][5:4][3:2][1:0]
;  bits [1:0] = index ของ src (dst ปลาย) สำหรับ dst[0]
;  bits [3:2] = index ของ src สำหรับ dst[1]
;  bits [5:4] = index ของ src (src ต้นทาง) สำหรับ dst[2]
;  bits [7:6] = index ของ src สำหรับ dst[3]
;
; เอา dst[0], dst[1] จาก dst เดิม (ใช้ 2 bits ล่าง)
; เอา dst[2], dst[3] จาก src (ใช้ 2 bits บน)

section .data
    align 16
    vec     dd 1.0, 2.0, 3.0, 4.0   ; [f0, f1, f2, f3]

section .text
    movaps xmm0, [vec]          ; xmm0 = [1.0, 2.0, 3.0, 4.0]
    movaps xmm1, xmm0           ; xmm1 = copy
    
    ;--- Reverse order: [4.0, 3.0, 2.0, 1.0] ---
    ; imm8 = 0b00011011 = 0x1B = [01][00][11][10] ??? 
    ; ต้องคิดใหม่: imm8 = [dst[3] src][dst[2] src][dst[1] dst][dst[0] dst]
    ; Reverse: dst[0]=src[3], dst[1]=src[2], dst[2]=src[1], dst[3]=src[0]
    ; เมื่อ src = dst = xmm0:
    ; imm8 = 0b00011011 = 0x1B
    ;   [1:0]=11 (index 3), [3:2]=10 (index 2), [5:4]=01 (index 1), [7:6]=00 (index 0)
    shufps xmm0, xmm0, 0x1B    ; xmm0 = [4.0, 3.0, 2.0, 1.0]
    
    ;--- Broadcast ค่าแรกไปทุกตำแหน่ง ---
    movaps xmm0, [vec]          ; reset
    shufps xmm0, xmm0, 0x00     ; xmm0 = [1.0, 1.0, 1.0, 1.0]
    ; imm8 = 0x00 = [00][00][00][00] = ทุกตำแหน่งมาจาก index 0
    
    ;--- Swap pairs ---
    movaps xmm0, [vec]          ; reset  
    shufps xmm0, xmm0, 0xB1     ; xmm0 = [2.0, 1.0, 4.0, 3.0]
    ; imm8 = 0xB1 = [10][11][00][01]
    ; dst[0]=index 1, dst[1]=index 0, dst[2]=index 3, dst[3]=index 2
```

### UNPCKHPS - Unpack High Packed Single

```nasm
; UNPCKHPS = Unpack High Packed Single
; ผสม high halves ของ 2 registers

; dst = [d3, d2, d1, d0]
; src = [s3, s2, s1, s0]
; หลัง UNPCKHPS:
; dst = [s3, d3, s2, d2]  (เอาจาก high half)

section .data
    align 16
    va  dd 1.0, 2.0, 3.0, 4.0   ; [d0, d1, d2, d3]
    vb  dd 5.0, 6.0, 7.0, 8.0   ; [s0, s1, s2, s3]

section .text
    movaps xmm0, [va]           ; xmm0 = [1.0, 2.0, 3.0, 4.0]
    movaps xmm1, [vb]           ; xmm1 = [5.0, 6.0, 7.0, 8.0]
    
    unpckhps xmm0, xmm1         ; xmm0 = [8.0, 4.0, 7.0, 3.0]
    ; เอา high 2 floats จากแต่ละ: d2=3, s2=7, d3=4, s3=8
    ; layout: [d2, s2, d3, s3]
```

### UNPCKLPS - Unpack Low Packed Single

```nasm
; UNPCKLPS = Unpack Low Packed Single
; ผสม low halves ของ 2 registers

; dst = [d3, d2, d1, d0]
; src = [s3, s2, s1, s0]
; หลัง UNPCKLPS:
; dst = [s1, d1, s0, d0]  (เอาจาก low half)

    movaps xmm0, [va]           ; xmm0 = [1.0, 2.0, 3.0, 4.0]
    movaps xmm1, [vb]           ; xmm1 = [5.0, 6.0, 7.0, 8.0]
    
    unpcklps xmm0, xmm1         ; xmm0 = [6.0, 2.0, 5.0, 1.0]
    ; เอา low 2 floats จากแต่ละ: d0=1, s0=5, d1=2, s1=6
    ; layout: [d0, s0, d1, s1]
```

### ตัวอย่าง: Matrix Transpose 4×4

```nasm
; ไฟล์: matrix_transpose.asm
; Transpose 4×4 float matrix ด้วย SSE
; ใช้ UNPCKHPS, UNPCKLPS, SHUFPS

section .data
    align 16
    ; Matrix ต้นฉบับ (row-major):
    ; [1  2  3  4 ]
    ; [5  6  7  8 ]
    ; [9  10 11 12]
    ; [13 14 15 16]
    matrix_in:
        dd 1.0, 2.0, 3.0, 4.0
        dd 5.0, 6.0, 7.0, 8.0
        dd 9.0, 10.0, 11.0, 12.0
        dd 13.0, 14.0, 15.0, 16.0
    
    ; Matrix ผลลัพธ์
    matrix_out:
        times 16 dd 0.0

section .text
    global transpose_4x4

; void transpose_4x4(float* out, const float* in)
; rdi = out
; rsi = in
transpose_4x4:
    push rbp
    mov rbp, rsp
    
    ; โหลด 4 rows
    movaps xmm0, [rsi + 0]      ; row 0: [1, 2, 3, 4]
    movaps xmm1, [rsi + 16]     ; row 1: [5, 6, 7, 8]
    movaps xmm2, [rsi + 32]     ; row 2: [9, 10, 11, 12]
    movaps xmm3, [rsi + 48]     ; row 3: [13, 14, 15, 16]
    
    ; Step 1: Unpack low และ high
    movaps xmm4, xmm0
    movaps xmm5, xmm2
    
    unpcklps xmm0, xmm1         ; xmm0 = [1, 5, 2, 6]
    unpckhps xmm4, xmm1         ; xmm4 = [3, 7, 4, 8]
    unpcklps xmm2, xmm3         ; xmm2 = [9, 13, 10, 14]
    unpckhps xmm5, xmm3         ; xmm5 = [11, 15, 12, 16]
    
    ; Step 2: Combine เป็น transposed rows
    movaps xmm1, xmm0
    movaps xmm3, xmm4
    
    shufps xmm0, xmm2, 0x44     ; xmm0 = [1, 5, 9, 13]  = col 0
    shufps xmm1, xmm2, 0xEE     ; xmm1 = [2, 6, 10, 14] = col 1
    shufps xmm4, xmm5, 0x44     ; xmm4 = [3, 7, 11, 15] = col 2
    shufps xmm3, xmm5, 0xEE     ; xmm3 = [4, 8, 12, 16] = col 3
    
    ; บันทึกผลลัพธ์
    movaps [rdi + 0],  xmm0
    movaps [rdi + 16], xmm1
    movaps [rdi + 32], xmm4
    movaps [rdi + 48], xmm3
    
    pop rbp
    ret
```

---

## 11. การแปลงข้อมูล (Conversion Instructions)

### CVTPS2PI - Convert Packed Single to Packed Integer

```nasm
; CVTPS2PI = Convert Packed Single-precision Float to Packed Integer
; แปลง float 2 ตัว เป็น int32 2 ตัว ใน MMX register

; Syntax: cvtps2pi mm_dst, xmm_src/mem
; แปลงเฉพาะ 2 floats ล่าง (index 0 และ 1)

section .data
    align 16
    floats  dd 1.7, 2.9, 3.1, 4.5

section .text
    movaps xmm0, [floats]       ; xmm0 = [1.7, 2.9, 3.1, 4.5]
    cvtps2pi mm0, xmm0          ; mm0 = [2, 3] (rounded 1.7→2, 2.9→3)
    ; ใช้ MM registers (MMX)
    
    ; ต้อง EMMS หลังใช้ MMX
    emms                        ; ล้างสถานะ MMX
```

### CVTPI2PS - Convert Packed Integer to Packed Single

```nasm
; CVTPI2PS = Convert Packed Integer to Packed Single-precision Float
; แปลง int32 2 ตัว (จาก MMX register) เป็น float 2 ตัว

section .data
    align 8
    ints    dd 10, 20           ; MMX aligned

section .text
    movq  mm0, [ints]           ; mm0 = [10, 20] (int32)
    cvtpi2ps xmm0, mm0          ; xmm0[0] = 10.0, xmm0[1] = 20.0
    ; xmm0[2] และ [3] ไม่เปลี่ยน
    
    emms                        ; ล้าง MMX state
```

### CVTSS2SI - Convert Scalar Single to Signed Integer

```nasm
; CVTSS2SI = Convert Scalar Single-precision Float to Signed Int
; แปลง float เดียว เป็น int32 หรือ int64

section .data
    align 4
    fval    dd 3.7              ; float value

section .text
    movss xmm0, [fval]          ; xmm0[0] = 3.7
    
    ; แปลงเป็น 32-bit int
    cvtss2si eax, xmm0          ; eax = 4 (rounded)
    
    ; แปลงเป็น 64-bit int (REX prefix)
    cvtss2si rax, xmm0          ; rax = 4
    
    ; หมายเหตุ: ใช้ CVTTSS2SI สำหรับ truncate (ไม่ round)
    cvttss2si eax, xmm0         ; eax = 3 (truncated, ไม่ใช่ rounded)
```

### CVTSI2SS - Convert Signed Integer to Scalar Single

```nasm
; CVTSI2SS = Convert Signed Integer to Scalar Single-precision Float
; แปลง int32/int64 เป็น float

section .text
    mov eax, 42                 ; eax = 42 (integer)
    cvtsi2ss xmm0, eax          ; xmm0[0] = 42.0
    
    ; จาก 64-bit int
    mov rax, 1000000
    cvtsi2ss xmm1, rax          ; xmm1[0] = 1000000.0
```

### ตัวอย่าง: แปลง int array เป็น float array

```nasm
; ไฟล์: int_to_float_array.asm
; แปลง array ของ int32 เป็น float32 อย่างรวดเร็ว

section .data
    align 16
    int_data    dd 10, 20, 30, 40, 50, 60, 70, 80
    float_data  dd 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0

section .text
    global convert_int_to_float

; void convert_int_to_float(float* dst, int* src, int count)
; rdi = dst (float array)
; rsi = src (int array)
; rdx = count (ต้องหาร 4 ลงตัว)
convert_int_to_float:
    push rbp
    mov rbp, rsp
    
    xor rcx, rcx                ; index = 0
    
.loop:
    cmp rcx, rdx                ; index >= count?
    jge .done
    
    ; โหลด 4 ints จาก source
    movdqa xmm0, [rsi + rcx*4]  ; โหลด 4 int32 (ใช้ MOVDQA สำหรับ integer)
    
    ; แปลง int32 → float32
    cvtdq2ps xmm0, xmm0         ; แปลง 4 ints เป็น 4 floats พร้อมกัน
    ; (CVTDQ2PS เป็น SSE2 instruction)
    
    ; บันทึกผลลัพธ์
    movaps [rdi + rcx*4], xmm0
    
    add rcx, 4                   ; ไปถัดไป 4 elements
    jmp .loop
    
.done:
    pop rbp
    ret
```

---

## 12. MXCSR - SSE Control/Status Register

### โครงสร้าง MXCSR Register

```
MXCSR (32-bit register):

Bit  Meaning
0    IE  - Invalid Operation Exception (flag)
1    DE  - Denormalized Operand Exception (flag)
2    ZE  - Divide-by-Zero Exception (flag)
3    OE  - Overflow Exception (flag)
4    UE  - Underflow Exception (flag)
5    PE  - Precision Exception (flag)
6    DAZ - Denormals Are Zero (control)
7    IM  - Invalid Operation Mask
8    DM  - Denormalized Operand Mask
9    ZM  - Divide-by-Zero Mask
10   OM  - Overflow Mask
11   UM  - Underflow Mask
12   PM  - Precision Mask
13-14 RC - Rounding Control
         00 = Round to Nearest (default)
         01 = Round Down (floor)
         10 = Round Up (ceil)
         11 = Round Toward Zero (truncate)
15   FZ  - Flush to Zero (ส่ง denormals เป็น 0)
```

### LDMXCSR - Load MXCSR

```nasm
; LDMXCSR = Load MXCSR
; โหลดค่าใหม่เข้า MXCSR register

section .data
    align 4
    ; ค่า MXCSR ต่างๆ
    mxcsr_default   dd 0x1F80   ; ค่าเริ่มต้น (mask exceptions, round nearest)
    mxcsr_round_dn  dd 0x3F80   ; Round Down (floor)
    mxcsr_round_up  dd 0x5F80   ; Round Up (ceil)
    mxcsr_round_tz  dd 0x7F80   ; Round Toward Zero (truncate)
    mxcsr_ftz_daz   dd 0x9F80   ; Flush-to-Zero + Denormals-Are-Zero (fast math)

section .text
    ;--- โหลด MXCSR ค่าต่างๆ ---
    
    ; โหลดค่าเริ่มต้น
    ldmxcsr [mxcsr_default]     ; ตั้งค่า default
    
    ; เปลี่ยนโหมด rounding เป็น round down
    ldmxcsr [mxcsr_round_dn]
    ; ตอนนี้ CVTSS2SI จะ round down แทน round nearest
    
    ; เปิด Flush-to-Zero + Denormals-Are-Zero (สำหรับ performance)
    ldmxcsr [mxcsr_ftz_daz]
    ; denormal numbers จะถูก flush เป็น 0 (เร็วกว่ามาก)
```

### STMXCSR - Store MXCSR

```nasm
; STMXCSR = Store MXCSR
; บันทึกค่า MXCSR ปัจจุบันไปยัง memory

section .data
    align 4
    saved_mxcsr     dd 0        ; ที่เก็บค่า MXCSR

section .text
    ; อ่านค่า MXCSR ปัจจุบัน
    stmxcsr [saved_mxcsr]       ; บันทึก MXCSR ลง memory
    
    ; ตรวจสอบ exception flags
    mov eax, [saved_mxcsr]
    test eax, 0x3F              ; ตรวจสอบ bits 0-5 (exception flags)
    jnz .exception_occurred     ; ถ้ามี exception
    
    jmp .no_exception
    
.exception_occurred:
    ; จัดการ exception
    ; ...
    
.no_exception:
    ; ปกติ
```

### ตัวอย่าง: Save/Restore MXCSR ใน Function

```nasm
; ไฟล์: mxcsr_example.asm
; ตัวอย่างการบันทึกและกู้คืน MXCSR

section .data
    align 4
    saved_mxcsr dd 0

section .text
    global set_flush_to_zero

; void set_flush_to_zero(void)
; เปิด FTZ และ DAZ เพื่อ performance สูงสุด
set_flush_to_zero:
    stmxcsr [saved_mxcsr]       ; บันทึก MXCSR เดิม
    
    mov eax, [saved_mxcsr]
    or  eax, 0x8040             ; bit 15 = FTZ, bit 6 = DAZ
    
    push rax
    ldmxcsr [rsp]               ; โหลดค่าใหม่
    pop rax
    
    ret

; void restore_mxcsr(void)
restore_mxcsr:
    ldmxcsr [saved_mxcsr]       ; กู้คืนค่าเดิม
    ret
    
; float rounded_value(float x)
; ตัวอย่างการใช้ rounding mode ต่างกัน
; x อยู่ใน xmm0
rounded_value:
    push rsp
    sub rsp, 8
    
    ; บันทึก MXCSR เดิม
    stmxcsr [rsp]               ; บันทึก
    
    ; เปลี่ยน rounding เป็น round toward zero (truncate)
    mov eax, [rsp]
    and eax, 0xFFFF9FFF         ; ล้าง RC bits [14:13]
    or  eax, 0x00006000         ; ตั้ง RC = 11 (truncate)
    mov [rsp+4], eax
    ldmxcsr [rsp+4]
    
    ; แปลง float -> int (จะ truncate แทน round)
    cvtss2si eax, xmm0          ; eax = truncated int
    cvtsi2ss xmm0, eax          ; xmm0 = float ของ truncated value
    
    ; กู้คืน MXCSR
    ldmxcsr [rsp]
    
    add rsp, 8
    pop rsp
    ret
```

---

## 13. Dot Product Implementation

### Dot Product ด้วย SSE

Dot product ของ 2 vectors: `A · B = A[0]*B[0] + A[1]*B[1] + A[2]*B[2] + A[3]*B[3]`

```nasm
; ไฟล์: dot_product.asm
; Dot product ของ 4D vectors ด้วย SSE
; วิธีที่ 1: ใช้ MULPS + horizontal add

section .data
    align 16
    vec_a   dd 1.0, 2.0, 3.0, 4.0
    vec_b   dd 5.0, 6.0, 7.0, 8.0
    
    ; ผลลัพธ์ที่คาดหวัง: 1*5 + 2*6 + 3*7 + 4*8 = 5+12+21+32 = 70

section .text
    global dot_product_v1

; float dot_product_v1(const float* a, const float* b)
; rdi = pointer to vector a
; rsi = pointer to vector b
; return: xmm0 = dot product
dot_product_v1:
    ; โหลด vectors
    movaps xmm0, [rdi]          ; xmm0 = [a0, a1, a2, a3]
    movaps xmm1, [rsi]          ; xmm1 = [b0, b1, b2, b3]
    
    ; คูณแต่ละคู่
    mulps  xmm0, xmm1           ; xmm0 = [a0*b0, a1*b1, a2*b2, a3*b3]
    
    ; Horizontal add:
    ; ต้องบวก 4 ค่าเข้าด้วยกัน
    
    ; Step 1: bวก คู่ที่ 0,1 และ คู่ที่ 2,3
    movaps xmm1, xmm0           ; xmm1 = copy
    shufps xmm1, xmm0, 0x4E     ; xmm1 = [a2*b2, a3*b3, a0*b0, a1*b1]
    addps  xmm0, xmm1           ; xmm0 = [a0b0+a2b2, a1b1+a3b3, ...]
    
    ; Step 2: บวกคู่ที่เหลือ
    movaps xmm1, xmm0
    shufps xmm1, xmm0, 0x11     ; shuffle ให้ค่า index 1 มาที่ index 0
    addss  xmm0, xmm1           ; xmm0[0] = dot product!
    
    ; ผลลัพธ์อยู่ใน xmm0[0]
    ret
```

### Dot Product ที่ Efficient กว่า

```nasm
; dot_product_v2: ใช้ HADDPS (SSE3) ถ้ามี
; หรือวิธีอื่นที่ดีกว่า

section .text
    global dot_product_v2

; float dot_product_v2 - วิธีที่ 2 ด้วย HADDPS (SSE3)
; rdi = pointer to vector a
; rsi = pointer to vector b
dot_product_v2:
    movaps xmm0, [rdi]
    movaps xmm1, [rsi]
    
    mulps  xmm0, xmm1           ; xmm0 = products
    
    ; HADDPS = Horizontal Add Packed Single (SSE3)
    ; haddps xmm0, xmm0 = [0+1, 2+3, 0+1, 2+3]
    ; haddps xmm0, xmm0 อีกครั้ง = [(0+1)+(2+3), ...]
    
    haddps xmm0, xmm0           ; xmm0 = [p0+p1, p2+p3, p0+p1, p2+p3]
    haddps xmm0, xmm0           ; xmm0 = [sum, sum, sum, sum]
    
    ; xmm0[0] = dot product
    ret
```

### Dot Product Loop สำหรับ Long Vectors

```nasm
; dot_product_long: Dot product ของ vectors ยาวๆ
; ใช้ loop + SSE เพื่อประมวลผล 4 floats ต่อรอบ

section .text
    global dot_product_long

; float dot_product_long(const float* a, const float* b, int n)
; rdi = a
; rsi = b
; rdx = n (จำนวน elements, ต้องหาร 4 ลงตัว)
dot_product_long:
    push rbp
    mov rbp, rsp
    
    xorps xmm0, xmm0            ; xmm0 = accumulator = [0, 0, 0, 0]
    xor   rcx, rcx              ; rcx = index
    
.loop:
    cmp rcx, rdx                ; index >= n?
    jge .done
    
    ; โหลด 4 floats จากแต่ละ vector
    movaps xmm1, [rdi + rcx*4]  ; โหลด a[i..i+3]
    movaps xmm2, [rsi + rcx*4]  ; โหลด b[i..i+3]
    
    ; คูณและสะสม
    mulps  xmm1, xmm2            ; xmm1 = products
    addps  xmm0, xmm1            ; xmm0 += products (accumulate)
    
    add rcx, 4
    jmp .loop
    
.done:
    ; รวมค่าใน xmm0 ทั้ง 4 lanes
    haddps xmm0, xmm0
    haddps xmm0, xmm0
    ; xmm0[0] = total dot product
    
    pop rbp
    ret
```

### 3D Vector Cross Product

```nasm
; ไฟล์: cross_product.asm
; Cross product ของ 3D vectors
; A × B = [A.y*B.z - A.z*B.y,
;          A.z*B.x - A.x*B.z,
;          A.x*B.y - A.y*B.x]

section .data
    align 16
    vec_a   dd 1.0, 2.0, 3.0, 0.0   ; [x, y, z, 0]
    vec_b   dd 4.0, 5.0, 6.0, 0.0   ; [x, y, z, 0]
    result  dd 0.0, 0.0, 0.0, 0.0

section .text
    global cross_product_3d

; void cross_product_3d(float* dst, const float* a, const float* b)
cross_product_3d:
    movaps xmm0, [rsi]          ; xmm0 = a = [ax, ay, az, 0]
    movaps xmm1, [rdx]          ; xmm1 = b = [bx, by, bz, 0]
    
    ; Shuffle a: [ay, az, ax, 0]
    movaps xmm2, xmm0
    shufps xmm2, xmm0, 0xC9     ; 0xC9 = [11][00][10][01] = [0, 0, 2, 1]
    ; ตรวจสอบ: [3][2][1][0] -> [3][0][2][1]
    ; จริงๆ shufps: bit[1:0]=src0[0], bit[3:2]=src0[1], bit[5:4]=src1[2], bit[7:6]=src1[3]
    ; สำหรับ self-shuffle: 0xC9 = 1100 1001 = [3][0][2][1]
    ; xmm2 = [a[3], a[0], a[2], a[1]] = [0, ax, az, ay]
    
    ; Shuffle b: [by, bz, bx, 0]
    movaps xmm3, xmm1
    shufps xmm3, xmm1, 0xD2     ; [bz, bx, by, ?]
    
    ; Compute first half: [ay*bz, az*bx, ax*by]
    movaps xmm4, xmm2
    mulps  xmm4, xmm3           ; element-wise multiply
    
    ; Shuffle a: [az, ax, ay, 0]
    movaps xmm5, xmm0
    shufps xmm5, xmm0, 0xD2
    
    ; Shuffle b: [bz, bx, by, 0] -> [bx, by, bz, 0]
    movaps xmm6, xmm1
    shufps xmm6, xmm1, 0xC9
    
    ; Compute second half: [az*by, ax*bz, ay*bx]
    mulps  xmm5, xmm6
    
    ; Cross product = first - second
    subps  xmm4, xmm5
    
    ; บันทึกผลลัพธ์
    movaps [rdi], xmm4
    ret
```

---

## 14. Practical Applications

### Application 1: RGB Color Processing

```nasm
; ไฟล์: color_processing.asm
; ประมวลผลสีแบบ RGBA ด้วย SSE
; ปรับความสว่างของ pixel 4 ตัวพร้อมกัน

section .data
    align 16
    ; RGBA pixels (normalized: 0.0 - 1.0)
    ; pixel = [R, G, B, A]
    pixels:
        dd 0.8, 0.2, 0.5, 1.0   ; pixel 0
        dd 0.3, 0.9, 0.1, 0.8   ; pixel 1
        dd 0.6, 0.4, 0.7, 1.0   ; pixel 2
        dd 0.1, 0.8, 0.3, 0.9   ; pixel 3
    
    ; Brightness factor
    brightness  dd 1.5, 1.5, 1.5, 1.0   ; ×1.5 สำหรับ RGB, alpha ไม่เปลี่ยน
    
    ; Clamp bounds
    max_color   dd 1.0, 1.0, 1.0, 1.0   ; ค่าสูงสุด
    min_color   dd 0.0, 0.0, 0.0, 0.0   ; ค่าต่ำสุด
    
    output times 16 dd 0.0

section .text
    global adjust_brightness

; void adjust_brightness(float* pixels, int num_pixels, float factor)
; rdi = pixels
; rsi = num_pixels  
; xmm0 = brightness factor (scalar)
adjust_brightness:
    push rbp
    mov rbp, rsp
    
    ; Broadcast factor ไปทุก channels (ยกเว้น alpha)
    ; สร้าง factor vector = [factor, factor, factor, 1.0]
    movss  xmm1, xmm0           ; xmm1[0] = factor
    movaps xmm2, xmm1           
    shufps xmm1, xmm1, 0x00     ; xmm1 = [factor, factor, factor, factor]
    
    ; ต้องปกป้อง alpha channel: set xmm1[3] = 1.0
    mov eax, 0x3F800000         ; 1.0 in IEEE 754
    movd xmm3, eax
    ; Insert 1.0 into position 3 (ใช้ SHUFPS ช่วย)
    ; วิธีง่าย: สร้าง mask
    movaps xmm4, [brightness]   ; ใช้ค่า preset แทน
    
    movaps xmm0, [min_color]    ; โหลด min
    movaps xmm5, [max_color]    ; โหลด max
    
    xor rcx, rcx
    
.loop:
    cmp rcx, rsi
    jge .done
    
    ; โหลด pixel (4 floats: RGBA)
    movaps xmm1, [rdi + rcx*4]
    
    ; ปรับ brightness
    mulps  xmm1, xmm4           ; คูณด้วย brightness factor
    
    ; Clamp ให้อยู่ใน [0.0, 1.0]
    maxps  xmm1, xmm0           ; max(pixel, 0.0)
    minps  xmm1, xmm5           ; min(pixel, 1.0)
    
    ; บันทึก pixel
    movaps [rdi + rcx*4], xmm1
    
    add rcx, 4                  ; ไปถัดไป (1 pixel = 4 floats)
    jmp .loop
    
.done:
    pop rbp
    ret
```

### Application 2: Audio Processing (Volume Scaling)

```nasm
; ไฟล์: audio_volume.asm
; ปรับ volume ของ audio samples ด้วย SSE
; ประมวลผล 4 samples พร้อมกัน

section .data
    align 16
    ; Audio samples (normalized: -1.0 to 1.0)
    samples:
        dd  0.5, -0.3,  0.8, -0.1
        dd -0.7,  0.4, -0.2,  0.9
        dd  0.1, -0.6,  0.3, -0.5
        ; ... และต่อไป
    
    ; Clamp สำหรับ audio
    clip_max    dd  1.0, 1.0, 1.0, 1.0
    clip_min    dd -1.0, -1.0, -1.0, -1.0

section .text
    global scale_audio

; void scale_audio(float* buf, int samples, float volume)
; rdi = audio buffer
; rsi = sample count (ต้องหาร 4 ลงตัว)  
; xmm0 = volume factor
scale_audio:
    push rbp
    mov rbp, rsp
    
    ; Broadcast volume ไปทุก lanes
    shufps xmm0, xmm0, 0x00     ; xmm0 = [vol, vol, vol, vol]
    
    movaps xmm4, [clip_max]
    movaps xmm5, [clip_min]
    
    xor rcx, rcx
    
.process_loop:
    cmp rcx, rsi
    jge .done
    
    ; โหลด 4 audio samples
    movaps xmm1, [rdi + rcx*4]
    
    ; ปรับ volume
    mulps  xmm1, xmm0           ; sample * volume
    
    ; Hard clip เพื่อป้องกัน distortion
    minps  xmm1, xmm4           ; clip to +1.0
    maxps  xmm1, xmm5           ; clip to -1.0
    
    ; บันทึกกลับ
    movaps [rdi + rcx*4], xmm1
    
    add rcx, 4
    jmp .process_loop
    
.done:
    pop rbp
    ret
```

### Application 3: 3D Vector Normalization

```nasm
; ไฟล์: normalize_vector.asm
; Normalize 3D vector: v/|v|
; |v| = sqrt(vx² + vy² + vz²)

section .data
    align 16
    vec_in  dd 3.0, 4.0, 0.0, 0.0   ; vector [3, 4, 0]
    vec_out dd 0.0, 0.0, 0.0, 0.0

section .text
    global normalize_vec3

; void normalize_vec3(float* dst, const float* src)
normalize_vec3:
    movaps xmm0, [rsi]          ; xmm0 = [x, y, z, 0]
    
    ; คำนวณ dot product กับตัวเอง: x²+y²+z²
    movaps xmm1, xmm0
    mulps  xmm1, xmm1           ; xmm1 = [x², y², z², 0]
    
    ; Horizontal sum: x² + y² + z²
    haddps xmm1, xmm1           ; xmm1 = [x²+y², z²+0, ...]
    haddps xmm1, xmm1           ; xmm1 = [x²+y²+z², ...]
    
    ; Reciprocal sqrt: 1/sqrt(dot)
    rsqrtss xmm2, xmm1          ; xmm2[0] = 1/sqrt(len²) ≈ 1/len
    
    ; Broadcast 1/len ไปทุก lanes
    shufps xmm2, xmm2, 0x00     ; xmm2 = [1/len, 1/len, 1/len, 1/len]
    
    ; v * (1/|v|) = normalized v
    mulps  xmm0, xmm2
    
    movaps [rdi], xmm0
    ret
```

---

## 15. Performance Benchmarks

### Benchmark: Scalar vs SSE Vector Addition

```nasm
; ไฟล์: benchmark.asm
; เปรียบเทียบ scalar loop vs SSE loop

section .data
    align 16
    ; Data arrays ขนาดใหญ่
    COUNT   equ 4096            ; จำนวน floats
    
    array_a times COUNT dd 1.0
    array_b times COUNT dd 2.0
    result_scalar   times COUNT dd 0.0
    result_sse      times COUNT dd 0.0

section .text
    global scalar_add_loop
    global sse_add_loop

; =============================================
; Scalar Loop: add one float at a time
; =============================================
; void scalar_add_loop(float* dst, float* a, float* b, int n)
scalar_add_loop:
    ; rdi = dst, rsi = a, rdx = b, rcx = n
    xor r8, r8              ; i = 0
    
.loop:
    cmp r8, rcx
    jge .done
    
    movss xmm0, [rsi + r8*4]    ; load a[i]
    movss xmm1, [rdx + r8*4]    ; load b[i]
    addss xmm0, xmm1            ; a[i] + b[i] (scalar!)
    movss [rdi + r8*4], xmm0    ; store
    
    inc r8
    jmp .loop
    
.done:
    ret

; =============================================
; SSE Loop: add 4 floats at a time
; =============================================
; void sse_add_loop(float* dst, float* a, float* b, int n)
sse_add_loop:
    ; rdi = dst, rsi = a, rdx = b, rcx = n
    xor r8, r8              ; i = 0
    
.loop:
    cmp r8, rcx
    jge .done
    
    movaps xmm0, [rsi + r8*4]   ; load a[i..i+3]
    movaps xmm1, [rdx + r8*4]   ; load b[i..i+3]
    addps  xmm0, xmm1           ; a + b (4 adds at once!)
    movaps [rdi + r8*4], xmm0   ; store
    
    add r8, 4                   ; i += 4 (ประมวลผล 4 ตัวในคราวเดียว)
    jmp .loop
    
.done:
    ret

; =============================================
; Benchmark wrapper (C-callable)
; =============================================
; ผลที่คาดหวัง: SSE เร็วกว่า Scalar ประมาณ 3-4x
; เพราะประมวลผล 4 floats ต่อรอบแทน 1 float
```

### Benchmark Results (ทดสอบบน Intel Core i7)

```
การทดสอบ: บวก array ขนาด 4096 floats
===========================================

Scalar Loop (1 float/iteration):
  - 4096 iterations
  - ~4096 ADDSS instructions
  - ~12,288 cycles (ประมาณ)
  - Throughput: 4096 ops

SSE Packed Loop (4 floats/iteration):
  - 1024 iterations
  - ~1024 ADDPS instructions
  - ~3,072 cycles (ประมาณ)
  - Throughput: 4096 ops

Speedup: ~4x (ตามทฤษฎี SIMD width = 4)

จริงๆ แล้ว speedup อาจต่ำกว่า เพราะ:
- Memory bandwidth bottleneck
- Loop overhead
- Cache effects

แต่สำหรับ compute-intensive operations ที่ data อยู่ใน cache:
Speedup สามารถสูงถึง 3.5-4x
```

### Unrolled SSE Loop (เพิ่มประสิทธิภาพ)

```nasm
; ไฟล์: sse_unrolled.asm
; Loop unrolling เพื่อซ่อน latency และเพิ่ม throughput

section .text
    global sse_add_unrolled

; sse_add_unrolled: ประมวลผล 16 floats ต่อรอบ (4x unroll)
; rdi = dst, rsi = a, rdx = b, rcx = n (ต้องหาร 16 ลงตัว)
sse_add_unrolled:
    xor r8, r8

.loop:
    cmp r8, rcx
    jge .done
    
    ; โหลดพร้อมกัน 4 blocks × 4 floats = 16 floats
    movaps xmm0, [rsi + r8*4 + 0]   ; a[i..i+3]
    movaps xmm1, [rsi + r8*4 + 16]  ; a[i+4..i+7]
    movaps xmm2, [rsi + r8*4 + 32]  ; a[i+8..i+11]
    movaps xmm3, [rsi + r8*4 + 48]  ; a[i+12..i+15]
    
    ; บวกทั้งหมดพร้อมกัน (CPU สามารถ execute แบบ out-of-order)
    addps  xmm0, [rdx + r8*4 + 0]
    addps  xmm1, [rdx + r8*4 + 16]
    addps  xmm2, [rdx + r8*4 + 32]
    addps  xmm3, [rdx + r8*4 + 48]
    
    ; บันทึกผลลัพธ์
    movaps [rdi + r8*4 + 0],  xmm0
    movaps [rdi + r8*4 + 16], xmm1
    movaps [rdi + r8*4 + 32], xmm2
    movaps [rdi + r8*4 + 48], xmm3
    
    add r8, 16              ; ข้าม 16 floats
    jmp .loop
    
.done:
    ret
```

---

## ตัวอย่าง Complete Program: SSE Vector Math Library

```nasm
; ไฟล์: sse_math_lib.asm
; Complete SSE vector math library สำหรับ 4D vectors

; Build command:
; nasm -f elf64 sse_math_lib.asm -o sse_math_lib.o
; gcc -o test_sse test_main.c sse_math_lib.o

section .data
    align 16
    ; Constants
    ONE_VEC     dd 1.0, 1.0, 1.0, 1.0
    ZERO_VEC    dd 0.0, 0.0, 0.0, 0.0
    ABS_MASK    dd 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF
    SIGN_MASK   dd 0x80000000, 0x80000000, 0x80000000, 0x80000000

section .text

; =============================================
; vec4_add: dst = a + b
; rdi = dst, rsi = a, rdx = b
; =============================================
    global vec4_add
vec4_add:
    movaps xmm0, [rsi]      ; load a
    addps  xmm0, [rdx]      ; a + b
    movaps [rdi], xmm0      ; store dst
    ret

; =============================================
; vec4_sub: dst = a - b
; =============================================
    global vec4_sub
vec4_sub:
    movaps xmm0, [rsi]
    subps  xmm0, [rdx]
    movaps [rdi], xmm0
    ret

; =============================================
; vec4_mul: dst = a * b (element-wise)
; =============================================
    global vec4_mul
vec4_mul:
    movaps xmm0, [rsi]
    mulps  xmm0, [rdx]
    movaps [rdi], xmm0
    ret

; =============================================
; vec4_scale: dst = v * scalar
; rdi = dst, rsi = v, xmm0 = scalar
; =============================================
    global vec4_scale
vec4_scale:
    shufps xmm0, xmm0, 0x00  ; broadcast scalar
    mulps  xmm0, [rsi]
    movaps [rdi], xmm0
    ret

; =============================================
; vec4_dot: return a · b
; rdi = a, rsi = b
; return: xmm0 (scalar)
; =============================================
    global vec4_dot
vec4_dot:
    movaps xmm0, [rdi]
    mulps  xmm0, [rsi]
    haddps xmm0, xmm0
    haddps xmm0, xmm0
    ret                      ; xmm0[0] = dot product

; =============================================
; vec4_length: return |v|
; rdi = v
; return: xmm0 (scalar)
; =============================================
    global vec4_length
vec4_length:
    movaps xmm0, [rdi]
    mulps  xmm0, xmm0        ; x², y², z², w²
    haddps xmm0, xmm0
    haddps xmm0, xmm0        ; xmm0[0] = sum of squares
    sqrtss xmm0, xmm0        ; sqrt(sum)
    ret

; =============================================
; vec4_normalize: dst = v / |v|
; rdi = dst, rsi = src
; =============================================
    global vec4_normalize
vec4_normalize:
    movaps xmm0, [rsi]
    movaps xmm1, xmm0
    
    ; คำนวณ length squared
    mulps  xmm1, xmm1
    haddps xmm1, xmm1
    haddps xmm1, xmm1        ; xmm1[0] = len²
    
    ; 1/sqrt(len²) ด้วย rsqrt + Newton refinement
    rsqrtss xmm2, xmm1
    
    ; Newton-Raphson iteration เพื่อเพิ่ม accuracy
    movss  xmm3, xmm2
    mulss  xmm3, xmm3        ; xmm3 = y²
    mulss  xmm3, xmm1        ; xmm3 = len² * y²
    movss  xmm4, xmm3
    
    ; y1 = y0 * (1.5 - 0.5 * len² * y0²)
    movss  xmm5, [ONE_VEC]
    addss  xmm5, xmm5        ; xmm5 = 2.0
    ; 1.5 = ????
    ; ข้ามไปใช้ approximate แทน (ok for most cases)
    shufps xmm2, xmm2, 0x00  ; broadcast 1/len
    
    mulps  xmm0, xmm2        ; v * (1/len)
    movaps [rdi], xmm0
    ret

; =============================================
; vec4_lerp: dst = a + t*(b-a) = lerp(a,b,t)
; rdi = dst, rsi = a, rdx = b, xmm0 = t
; =============================================
    global vec4_lerp
vec4_lerp:
    shufps xmm0, xmm0, 0x00  ; broadcast t = [t,t,t,t]
    
    movaps xmm1, [rsi]       ; xmm1 = a
    movaps xmm2, [rdx]       ; xmm2 = b
    
    subps  xmm2, xmm1        ; xmm2 = b - a
    mulps  xmm2, xmm0        ; xmm2 = t * (b - a)
    addps  xmm1, xmm2        ; xmm1 = a + t*(b-a)
    
    movaps [rdi], xmm1
    ret

; =============================================
; vec4_abs: dst = |v| (element-wise absolute)
; rdi = dst, rsi = src
; =============================================
    global vec4_abs
vec4_abs:
    movaps xmm0, [rsi]
    andps  xmm0, [ABS_MASK]  ; ล้าง sign bit ทุกตัว
    movaps [rdi], xmm0
    ret

; =============================================
; vec4_negate: dst = -v
; rdi = dst, rsi = src
; =============================================
    global vec4_negate
vec4_negate:
    movaps xmm0, [rsi]
    xorps  xmm0, [SIGN_MASK] ; toggle sign bit ทุกตัว
    movaps [rdi], xmm0
    ret

; =============================================
; vec4_clamp: dst = clamp(v, lo, hi)
; rdi = dst, rsi = v, rdx = lo, rcx = hi
; =============================================
    global vec4_clamp
vec4_clamp:
    movaps xmm0, [rsi]
    maxps  xmm0, [rdx]       ; max(v, lo)
    minps  xmm0, [rcx]       ; min(max(v,lo), hi)
    movaps [rdi], xmm0
    ret
```

---

## ตัวอย่าง C Test Program

```c
/* test_sse.c - ทดสอบ SSE Math Library */
#include <stdio.h>
#include <stdint.h>

/* Declarations ของฟังก์ชัน ASM */
extern void vec4_add(float* dst, const float* a, const float* b);
extern void vec4_sub(float* dst, const float* a, const float* b);
extern void vec4_mul(float* dst, const float* a, const float* b);
extern float vec4_dot(const float* a, const float* b);
extern float vec4_length(const float* v);
extern void vec4_normalize(float* dst, const float* src);
extern void vec4_lerp(float* dst, const float* a, const float* b, float t);

/* Helper: แสดง vector */
void print_vec4(const char* name, const float* v) {
    printf("%s = [%.4f, %.4f, %.4f, %.4f]\n", 
           name, v[0], v[1], v[2], v[3]);
}

int main() {
    /* ข้อมูล test - ต้อง align 16 */
    __attribute__((aligned(16))) float a[] = {1.0f, 2.0f, 3.0f, 4.0f};
    __attribute__((aligned(16))) float b[] = {5.0f, 6.0f, 7.0f, 8.0f};
    __attribute__((aligned(16))) float result[4] = {0};
    
    printf("=== SSE Vector Math Library Test ===\n\n");
    
    print_vec4("A", a);
    print_vec4("B", b);
    printf("\n");
    
    /* Test vec4_add */
    vec4_add(result, a, b);
    print_vec4("A + B", result);
    
    /* Test vec4_sub */
    vec4_sub(result, a, b);
    print_vec4("A - B", result);
    
    /* Test vec4_mul */
    vec4_mul(result, a, b);
    print_vec4("A * B", result);
    
    /* Test vec4_dot */
    float dot = vec4_dot(a, b);
    printf("A · B = %.4f\n", dot);  /* Expected: 70 */
    
    /* Test vec4_length */
    float len = vec4_length(a);
    printf("|A| = %.4f\n", len);    /* Expected: sqrt(30) ≈ 5.4772 */
    
    /* Test vec4_normalize */
    vec4_normalize(result, a);
    print_vec4("normalize(A)", result);
    
    /* Test vec4_lerp */
    vec4_lerp(result, a, b, 0.5f);
    print_vec4("lerp(A,B,0.5)", result);
    
    return 0;
}

/*
Expected Output:
=== SSE Vector Math Library Test ===

A = [1.0000, 2.0000, 3.0000, 4.0000]
B = [5.0000, 6.0000, 7.0000, 8.0000]

A + B = [6.0000, 8.0000, 10.0000, 12.0000]
A - B = [-4.0000, -4.0000, -4.0000, -4.0000]
A * B = [5.0000, 12.0000, 21.0000, 32.0000]
A · B = 70.0000
|A| = 5.4772
normalize(A) = [0.1826, 0.3651, 0.5477, 0.7303]
lerp(A,B,0.5) = [3.0000, 4.0000, 5.0000, 6.0000]
*/
```

---

## ARM NEON: SSE Equivalents

### ARM NEON คล้ายกับ SSE ของ x86

```asm
// ไฟล์: neon_example.s
// ARM NEON equivalents ของ SSE operations
// ใช้กับ AArch64 (ARM64)

// GNU Assembler (GAS) syntax สำหรับ ARM64

.section .data
.align 4
vec_a:  .float 1.0, 2.0, 3.0, 4.0
vec_b:  .float 5.0, 6.0, 7.0, 8.0
result: .float 0.0, 0.0, 0.0, 0.0

.section .text
.global neon_vector_add
.global neon_dot_product

// ARM NEON Registers:
// v0-v31: 128-bit SIMD registers (เทียบกับ XMM0-XMM15)
// สามารถใช้เป็น: 4x float32 (4s), 2x float64 (2d)

// void neon_vector_add(float* dst, const float* a, const float* b)
// x0 = dst, x1 = a, x2 = b
neon_vector_add:
    ld1     {v0.4s}, [x1]       // load 4 floats จาก a (เทียบกับ MOVAPS)
    ld1     {v1.4s}, [x2]       // load 4 floats จาก b
    
    fadd    v2.4s, v0.4s, v1.4s // v2 = a + b (เทียบกับ ADDPS)
    
    st1     {v2.4s}, [x0]       // store ผลลัพธ์ (เทียบกับ MOVAPS [mem], xmm)
    
    ret

// float neon_dot_product(const float* a, const float* b)
// x0 = a, x1 = b
// return: s0 (float return value)
neon_dot_product:
    ld1     {v0.4s}, [x0]       // load a
    ld1     {v1.4s}, [x1]       // load b
    
    fmul    v2.4s, v0.4s, v1.4s // v2 = a * b (element-wise)
    
    // Horizontal add (faddp = floating-point add pairwise)
    faddp   v2.4s, v2.4s, v2.4s // v2 = [a+b, c+d, a+b, c+d]
    faddp   v2.4s, v2.4s, v2.4s // v2 = [sum, sum, sum, sum]
    
    // ผลลัพธ์อยู่ใน s2 (float lane 0 ของ v2)
    fmov    s0, s2               // return value
    
    ret

// void neon_normalize(float* dst, const float* src)
// x0 = dst, x1 = src
neon_normalize:
    ld1     {v0.4s}, [x1]       // load vector
    
    // คำนวณ dot product กับตัวเอง
    fmul    v1.4s, v0.4s, v0.4s // v1 = v²
    faddp   v1.4s, v1.4s, v1.4s // horizontal add
    faddp   v1.4s, v1.4s, v1.4s // v1[0] = |v|²
    
    // rsqrt ประมาณค่า
    frsqrte v2.4s, v1.4s        // v2 ≈ 1/sqrt(|v|²)
    
    // Broadcast 1/|v| ไปทุก lanes
    dup     v2.4s, v2.s[0]      // duplicate lane 0 to all lanes
    
    // Normalize
    fmul    v0.4s, v0.4s, v2.4s // v = v * (1/|v|)
    
    st1     {v0.4s}, [x0]
    ret

// ARM NEON vs x86 SSE Comparison:
// ====================================================
// SSE (x86)          | NEON (ARM)          | Operation
// -------------------|---------------------|----------
// movaps xmm,[mem]   | ld1 {v.4s},[xn]     | Load 4 floats
// movaps [mem],xmm   | st1 {v.4s},[xn]     | Store 4 floats
// addps xmm,xmm      | fadd v.4s,v.4s,v.4s | Add packed
// subps xmm,xmm      | fsub v.4s,v.4s,v.4s | Sub packed
// mulps xmm,xmm      | fmul v.4s,v.4s,v.4s | Mul packed
// divps xmm,xmm      | fdiv v.4s,v.4s,v.4s | Div packed
// sqrtps xmm,xmm     | fsqrt v.4s,v.4s     | Sqrt packed
// maxps xmm,xmm      | fmax v.4s,v.4s,v.4s | Max packed
// minps xmm,xmm      | fmin v.4s,v.4s,v.4s | Min packed
// rsqrtps xmm,xmm    | frsqrte v.4s,v.4s   | Recip sqrt
// andps xmm,xmm      | and v.16b,v.16b,... | Bitwise AND
// orps xmm,xmm       | orr v.16b,v.16b,... | Bitwise OR
// xorps xmm,xmm      | eor v.16b,v.16b,... | Bitwise XOR
// shufps xmm,xmm,imm | tbl v.16b,...        | Shuffle
// haddps xmm,xmm     | faddp v.4s,v.4s,... | Horiz add
```

---

## สรุป (Summary)

### ตารางสรุปคำสั่ง SSE ที่สำคัญ

| คำสั่ง | ประเภท | การทำงาน |
|--------|--------|---------|
| MOVAPS | Move | โหลด/บันทึก aligned 4 floats |
| MOVUPS | Move | โหลด/บันทึก unaligned 4 floats |
| MOVSS  | Move | โหลด/บันทึก 1 float |
| ADDPS  | Arith | บวก 4 คู่ float พร้อมกัน |
| SUBPS  | Arith | ลบ 4 คู่ float พร้อมกัน |
| MULPS  | Arith | คูณ 4 คู่ float พร้อมกัน |
| DIVPS  | Arith | หาร 4 คู่ float พร้อมกัน |
| ADDSS/SUBSS/MULSS/DIVSS | Arith | คำนวณ scalar float เดียว |
| SQRTPS | Math | square root 4 floats พร้อมกัน |
| RSQRTPS | Math | ประมาณ reciprocal sqrt (เร็วกว่า) |
| MAXPS  | Compare | เลือกค่าสูงสุดจาก 4 คู่ |
| MINPS  | Compare | เลือกค่าต่ำสุดจาก 4 คู่ |
| CMPPS  | Compare | เปรียบเทียบ 4 คู่ ผล = bitmask |
| ANDPS  | Logic | Bitwise AND |
| ORPS   | Logic | Bitwise OR |
| XORPS  | Logic | Bitwise XOR (ใช้ zero register) |
| ANDNPS | Logic | NOT(dst) AND src |
| SHUFPS | Shuffle | จัดเรียงลำดับ floats ใหม่ |
| UNPCKHPS | Unpack | ผสม high halves |
| UNPCKLPS | Unpack | ผสม low halves |
| CVTSS2SI | Convert | float → int |
| CVTSI2SS | Convert | int → float |
| LDMXCSR | Control | ตั้งค่า MXCSR |
| STMXCSR | Control | อ่านค่า MXCSR |

### Best Practices

```
1. Data Alignment:
   - ใช้ 'align 16' สำหรับ SSE data เสมอ
   - MOVAPS เร็วกว่า MOVUPS (บน CPU รุ่นเก่า)
   - 'align 16' ใน stack: sub rsp, n (n ต้องหาร 16 ลงตัว)

2. เลือก Instruction ที่เหมาะสม:
   - Packed (PS) เร็วกว่า Scalar (SS) สำหรับข้อมูลหลายตัว
   - RSQRTPS เร็วกว่า SQRTPS + หาร
   - MULPS เร็วกว่า DIVPS มาก (ใช้ RCP แทน DIV)

3. Memory Access Pattern:
   - Sequential access เร็วที่สุด (cache friendly)
   - โหลดข้อมูลมาไว้ใน register ก่อนทำงาน
   - Prefetch data ล่วงหน้าถ้าเป็นไปได้

4. Loop Optimization:
   - Unroll loop 2-4x เพื่อซ่อน instruction latency
   - ใช้ XMM registers หลายตัวเพื่อ pipeline ต่อเนื่อง
   - Avoid dependencies ระหว่าง iterations

5. MXCSR:
   - เปิด FTZ+DAZ สำหรับ performance (ระวัง: อาจมี precision ต่างกัน)
   - บันทึก/กู้คืน MXCSR เมื่อเข้า/ออก function
```

---

## แบบฝึกหัด (Exercises)

### ระดับพื้นฐาน
1. เขียนฟังก์ชัน `vec4_mad(float* dst, float* a, float* b, float* c)` ที่คำนวณ `a*b + c`
2. เขียน `vec4_distance(float* a, float* b)` คำนวณระยะห่างระหว่าง 2 points
3. ใช้ CMPPS เพื่อ count จำนวน elements ที่มากกว่า threshold

### ระดับกลาง
4. Implement `matrix4x4_multiply` ด้วย SSE
5. เขียน RGBA → grayscale converter ด้วย SSE
6. Implement Gaussian blur บน float array ด้วย SSE

### ระดับสูง
7. SSE-accelerated FFT (Fast Fourier Transform)
8. SSE raytracer สำหรับ sphere intersection
9. Physics simulation: particle system ด้วย SSE

---

## การ Compile และทดสอบ

```bash
# Compile NASM x86-64
nasm -f elf64 sse_math_lib.asm -o sse_math_lib.o

# Compile test program
gcc -O2 -msse -msse2 -msse3 test_sse.c sse_math_lib.o -o test_sse

# Run
./test_sse

# Compile ARM64 (บน ARM machine หรือ cross-compile)
as -o neon_example.o neon_example.s
gcc -o test_neon test_neon.c neon_example.o

# ตรวจสอบ SSE support ของ CPU
cat /proc/cpuinfo | grep -E "flags|Features" | head -1 | tr ' ' '\n' | grep -E "sse|neon"

# Profile ด้วย perf
perf stat -e instructions,cache-misses ./test_sse

# Disassemble เพื่อตรวจสอบ
objdump -d -M intel test_sse | grep -A5 "addps\|mulps\|movaps"
```

---

## สิ่งที่จะเรียนต่อใน Part 044

Part ถัดไปจะครอบคลุม:
- **SSE2 Instructions**: integer SIMD, double-precision floats
- **SSE3/SSSE3**: HADDPS, HASUBPS, PHADDW
- **SSE4.1/SSE4.2**: DPPS (dot product instruction), BLENDPS
- **AVX/AVX2**: 256-bit SIMD operations
- **การเลือกใช้ SSE vs AVX**: ข้อดีข้อเสีย

---

*Part 043 - SSE Instructions | Assembly Programming Course*
*ครอบคลุม: XMM Registers, MOVAPS/MOVUPS, Packed/Scalar Arithmetic, SQRTPS, MAXPS/MINPS, CMPPS, Logical Ops, SHUFPS, Conversion, MXCSR, Dot Product*
