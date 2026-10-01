# Part 041: x87 FPU Programming

## การเขียนโปรแกรม x87 Floating Point Unit

---

## บทนำ (Introduction)

x87 FPU (Floating Point Unit) เป็น coprocessor สำหรับการคำนวณทศนิยมที่ถูกนำมาใช้ตั้งแต่ยุค Intel 8087 (1980)
ถึงแม้ว่าปัจจุบัน SSE/AVX จะมีประสิทธิภาพดีกว่า แต่ x87 ยังคงมีความสำคัญในหลายด้าน:

- **80-bit Extended Precision** — ความแม่นยำสูงสุด 80 bits
- **Legacy Compatibility** — โค้ดเก่าจำนวนมากยังใช้ x87
- **Transcendental Functions** — FSIN, FCOS, FSQRT มีใน hardware
- **Special Values** — รองรับ NaN, Infinity, Denormals ตาม IEEE 754

---

## 1. x87 FPU Register Stack (ST(0) - ST(7))

### โครงสร้าง Register Stack

```
x87 FPU Register Stack:
┌─────────────────────────────────────────┐
│  ST(0)  ← Top of Stack (TOS)           │  80-bit Extended
│  ST(1)                                  │  Precision Format
│  ST(2)                                  │
│  ST(3)                                  │  Sign (1 bit)
│  ST(4)                                  │  Exponent (15 bits)
│  ST(5)                                  │  Mantissa (64 bits)
│  ST(6)                                  │
│  ST(7)                                  │
└─────────────────────────────────────────┘

80-bit Format:
┌──┬──────────────────┬──────────────────────────────────────────────────────────┐
│S │    Exponent      │                    Mantissa                              │
│1 │    15 bits       │                    64 bits                               │
└──┴──────────────────┴──────────────────────────────────────────────────────────┘
```

### Status Word และ Control Word

```nasm
; x87 Status Word (16-bit)
; Bit 15: B   - FPU Busy
; Bit 14: C3  - Condition Code 3
; Bit 11: C2  - Condition Code 2
; Bit 10: C1  - Condition Code 1
; Bit  8: C0  - Condition Code 0
; Bit 7:  ES  - Error Summary
; Bits 11-13: TOP - Stack Top Pointer
; Bit 5:  SF  - Stack Fault
; Bit 4:  P   - Precision Exception
; Bit 3:  U   - Underflow Exception
; Bit 2:  O   - Overflow Exception
; Bit 1:  Z   - Zero Divide Exception
; Bit 0:  I   - Invalid Operation Exception

; x87 Control Word (16-bit)
; Bits 10-11: RC - Rounding Control
;   00 = Round to Nearest (Even)
;   01 = Round toward -Infinity
;   10 = Round toward +Infinity
;   11 = Round toward Zero (Truncate)
; Bits 8-9: PC - Precision Control
;   00 = Single (24-bit)
;   10 = Double (53-bit)
;   11 = Extended (64-bit)
```

---

## 2. FLD/FST/FSTP — Load/Store Operations

### NASM x86 Examples

```nasm
; ============================================================
; ตัวอย่าง: x87 FPU Load/Store Operations
; Part 041 - x87 FPU Programming
; ============================================================

section .data
    ; ตัวแปรทศนิยมชนิดต่างๆ
    float32     dd  3.14159265      ; 32-bit float (single precision)
    float64     dq  2.71828182845   ; 64-bit double precision
    float80     dt  1.41421356237   ; 80-bit extended precision
    
    result32    dd  0.0             ; ผลลัพธ์ 32-bit
    result64    dq  0.0             ; ผลลัพธ์ 64-bit
    result80    dt  0.0             ; ผลลัพธ์ 80-bit
    
    int_val     dd  42              ; ค่า integer สำหรับ load
    int16_val   dw  100             ; ค่า 16-bit integer
    int64_val   dq  1000000         ; ค่า 64-bit integer

section .text
global _start

; ============================================================
; ฟังก์ชัน: fpu_load_store_demo
; แสดงการ load/store ข้อมูลทศนิยม
; ============================================================
fpu_load_store_demo:
    push    ebp
    mov     ebp, esp
    
    ; --- FLD: Load Float ---
    fld     dword [float32]     ; โหลด 32-bit float เข้า ST(0)
    fld     qword [float64]     ; โหลด 64-bit double เข้า ST(0), push stack
    fld     tword [float80]     ; โหลด 80-bit extended เข้า ST(0)
    
    ; ตอนนี้ Stack:
    ; ST(0) = float80 (1.41421...)
    ; ST(1) = float64 (2.71828...)
    ; ST(2) = float32 (3.14159...)
    
    ; --- FILD: Load Integer ---
    fild    dword [int_val]     ; โหลด integer 32-bit แปลงเป็น float
    fild    word  [int16_val]   ; โหลด integer 16-bit แปลงเป็น float
    fild    qword [int64_val]   ; โหลด integer 64-bit แปลงเป็น float
    
    ; Stack ตอนนี้มี 6 ค่า
    
    ; --- FST: Store (ไม่ pop stack) ---
    fst     dword [result32]    ; เก็บ ST(0) เป็น 32-bit float
    fst     qword [result64]    ; เก็บ ST(0) เป็น 64-bit double
    
    ; --- FSTP: Store and Pop (pop stack หลัง store) ---
    fstp    tword [result80]    ; เก็บ ST(0) เป็น 80-bit และ pop
    fstp    qword [result64]    ; เก็บและ pop อีก 5 ครั้ง
    fstp    qword [result64]
    fstp    dword [result32]
    fstp    dword [result32]
    fstp    dword [result32]
    
    ; Stack ว่างแล้ว
    
    pop     ebp
    ret

; ============================================================
; ฟังก์ชัน: fpu_exchange_demo
; แสดงการ exchange registers
; ============================================================
fpu_exchange_demo:
    fld     dword [float32]     ; ST(0) = 3.14159
    fld     qword [float64]     ; ST(0) = 2.71828, ST(1) = 3.14159
    
    ; FXCH: Exchange ST(0) กับ ST(n)
    fxch    st1                 ; สลับ ST(0) และ ST(1)
    ; ST(0) = 3.14159, ST(1) = 2.71828
    
    fxch                        ; สลับ ST(0) และ ST(1) (default st1)
    ; กลับมาเหมือนเดิม
    
    fstp    qword [result64]    ; pop
    fstp    dword [result32]    ; pop
    ret
```

---

## 3. FADD/FSUB/FMUL/FDIV — Arithmetic Operations

```nasm
; ============================================================
; ตัวอย่าง: x87 Arithmetic Operations
; ============================================================

section .data
    val_a       dq  10.5        ; ตัวถูกดำเนินการ A
    val_b       dq  3.2         ; ตัวถูกดำเนินการ B
    val_c       dq  2.0         ; ตัวหาร
    
    pi          dq  3.141592653589793
    e           dq  2.718281828459045
    
    result      dq  0.0

section .text

; ============================================================
; ฟังก์ชัน: arithmetic_demo
; แสดงการคำนวณทางคณิตศาสตร์
; ============================================================
arithmetic_demo:
    ; --- FADD: Addition ---
    fld     qword [val_a]       ; ST(0) = 10.5
    fld     qword [val_b]       ; ST(0) = 3.2, ST(1) = 10.5
    fadd                        ; ST(0) = ST(0) + ST(1) = 13.7, pop
    ; เทียบเท่ากับ: fadd st0, st1
    fstp    qword [result]      ; เก็บผลลัพธ์
    
    ; FADD กับ memory operand
    fld     qword [val_a]       ; ST(0) = 10.5
    fadd    qword [val_b]       ; ST(0) = 10.5 + 3.2 = 13.7
    fstp    qword [result]
    
    ; FADDP: Add and Pop
    fld     qword [val_a]       ; ST(0) = 10.5
    fld     qword [val_b]       ; ST(0) = 3.2, ST(1) = 10.5
    faddp   st1, st0            ; ST(1) = ST(1) + ST(0), pop ST(0)
    ; ST(0) = 13.7
    fstp    qword [result]
    
    ; --- FSUB: Subtraction ---
    fld     qword [val_a]       ; ST(0) = 10.5
    fld     qword [val_b]       ; ST(0) = 3.2, ST(1) = 10.5
    fsub                        ; ST(0) = ST(1) - ST(0) = 7.3
    fstp    qword [result]
    
    ; FSUBR: Reverse Subtraction
    fld     qword [val_a]       ; ST(0) = 10.5
    fld     qword [val_b]       ; ST(0) = 3.2, ST(1) = 10.5
    fsubr                       ; ST(0) = ST(0) - ST(1) = -7.3
    fstp    qword [result]
    
    ; --- FMUL: Multiplication ---
    fld     qword [val_a]       ; ST(0) = 10.5
    fmul    qword [val_b]       ; ST(0) = 10.5 * 3.2 = 33.6
    fstp    qword [result]
    
    ; --- FDIV: Division ---
    fld     qword [val_a]       ; ST(0) = 10.5
    fdiv    qword [val_c]       ; ST(0) = 10.5 / 2.0 = 5.25
    fstp    qword [result]
    
    ; FDIVR: Reverse Division
    fld     qword [val_a]       ; ST(0) = 10.5
    fdivr   qword [val_c]       ; ST(0) = 2.0 / 10.5 ≈ 0.190476
    fstp    qword [result]
    
    ; --- Complex Expression: (a + b) * (a - b) = a² - b² ---
    fld     qword [val_a]       ; ST(0) = a
    fld     qword [val_b]       ; ST(0) = b, ST(1) = a
    fld     st1                 ; ST(0) = a, ST(1) = b, ST(2) = a
    fadd    st0, st1            ; ST(0) = a + b
    fxch    st1                 ; ST(0) = b, ST(1) = a + b, ST(2) = a
    fsubr   st0, st2            ; ST(0) = a - b
    fmulp   st1, st0            ; ST(0) = (a+b)*(a-b)
    fstp    qword [result]      ; เก็บผลลัพธ์
    
    ret

; ============================================================
; ฟังก์ชัน: dot_product_x87
; คำนวณ dot product ของ 2 vectors
; Input: ESI = pointer to vector A
;        EDI = pointer to vector B
;        ECX = number of elements
; Output: ST(0) = dot product
; ============================================================
dot_product_x87:
    push    ebp
    mov     ebp, esp
    push    esi
    push    edi
    
    fldz                        ; ST(0) = 0.0 (accumulator)
    
.loop:
    fld     qword [esi]         ; ST(0) = A[i]
    fmul    qword [edi]         ; ST(0) = A[i] * B[i]
    faddp   st1, st0            ; ST(0) += A[i] * B[i]
    
    add     esi, 8              ; ไปยัง element ถัดไป (double = 8 bytes)
    add     edi, 8
    loop    .loop               ; วนซ้ำ ECX ครั้ง
    
    pop     edi
    pop     esi
    pop     ebp
    ret
```

---

## 4. FIADD/FISUB — Integer Arithmetic

```nasm
; ============================================================
; ตัวอย่าง: x87 Integer Arithmetic Operations
; ============================================================

section .data
    flt_val     dq  100.75      ; ค่า float
    int_op      dd  25          ; integer operand
    int_op16    dw  10          ; 16-bit integer operand
    int_result  dd  0

section .text

; ============================================================
; ฟังก์ชัน: integer_arithmetic_demo
; แสดงการคำนวณกับ integer
; ============================================================
integer_arithmetic_demo:
    ; FIADD: Float + Integer
    fld     qword [flt_val]     ; ST(0) = 100.75
    fiadd   dword [int_op]      ; ST(0) = 100.75 + 25.0 = 125.75
    fstp    qword [flt_val]
    
    ; FISUB: Float - Integer
    fld     qword [flt_val]     ; ST(0) = 125.75
    fisub   dword [int_op]      ; ST(0) = 125.75 - 25.0 = 100.75
    fstp    qword [flt_val]
    
    ; FISUBR: Integer - Float
    fld     qword [flt_val]     ; ST(0) = 100.75
    fisubr  dword [int_op]      ; ST(0) = 25.0 - 100.75 = -75.75
    fstp    qword [flt_val]
    
    ; FIMUL: Float * Integer
    fld     qword [flt_val]     ; ST(0) = -75.75
    fabs                        ; ST(0) = 75.75 (absolute value)
    fimul   dword [int_op]      ; ST(0) = 75.75 * 25 = 1893.75
    fstp    qword [flt_val]
    
    ; FIDIV: Float / Integer
    fld     qword [flt_val]     ; ST(0) = 1893.75
    fidiv   dword [int_op]      ; ST(0) = 1893.75 / 25 = 75.75
    fstp    qword [flt_val]
    
    ; FIST: Store as Integer (convert float to int, no pop)
    fld     qword [flt_val]     ; ST(0) = 75.75
    fist    dword [int_result]  ; int_result = 76 (rounded)
    
    ; FISTP: Store as Integer and Pop
    fistp   dword [int_result]  ; int_result = 76, pop
    
    ; FISTTP: Store as Integer (truncate) and Pop (SSE3)
    fld     qword [flt_val]     ; ST(0) = 75.75
    fisttp  dword [int_result]  ; int_result = 75 (truncated!), pop
    
    ret
```

---

## 5. FCOM/FUCOM — Compare Operations

```nasm
; ============================================================
; ตัวอย่าง: x87 Floating Point Comparison
; ============================================================

section .data
    num1    dq  5.5
    num2    dq  3.7
    num3    dq  5.5             ; เท่ากับ num1
    nan_val dq  0x7FF8000000000000  ; NaN value
    
    msg_gt  db  "num1 > num2", 10, 0
    msg_lt  db  "num1 < num2", 10, 0
    msg_eq  db  "num1 = num2", 10, 0

section .text

; ============================================================
; ฟังก์ชัน: compare_floats
; เปรียบเทียบ float สองค่า
; Input: ST(0) = a, ST(1) = b (b is on stack before a)
; Returns: based on comparison
; ============================================================
compare_floats:
    ; --- FCOM: Compare ST(0) with operand ---
    fld     qword [num1]        ; ST(0) = 5.5
    fcom    qword [num2]        ; เปรียบเทียบ ST(0) กับ num2
    
    ; ผลลัพธ์อยู่ใน Status Word flags C0, C2, C3:
    ; C3=0, C0=0: ST(0) > mem (ST(0) > num2)
    ; C3=0, C0=1: ST(0) < mem (ST(0) < num2)
    ; C3=1, C0=0: ST(0) = mem
    ; C3=1, C0=1: Unordered (NaN)
    
    fnstsw  ax                  ; โหลด Status Word เข้า AX
    sahf                        ; ย้าย AH เข้า FLAGS register
    
    ; หลังจาก SAHF:
    ; CF = C0, ZF = C3, PF = C2
    ; เหมือนกับการเปรียบเทียบ integer
    ja      .greater            ; ST(0) > num2
    jb      .less               ; ST(0) < num2
    jz      .equal              ; ST(0) = num2
    jp      .unordered          ; Unordered (NaN involved)
    
.greater:
    ; num1 > num2
    jmp     .done
.less:
    ; num1 < num2
    jmp     .done
.equal:
    ; num1 = num2
    jmp     .done
.unordered:
    ; NaN comparison
.done:
    fstp    st0                 ; ทิ้งค่าใน stack
    ret

; ============================================================
; ฟังก์ชัน: compare_modern_way
; วิธีใหม่ใช้ FCOMI (Pentium Pro+)
; ============================================================
compare_modern_way:
    fld     qword [num1]        ; ST(0) = 5.5
    fld     qword [num2]        ; ST(0) = 3.7, ST(1) = 5.5
    
    ; FCOMI: เปรียบเทียบและ set EFLAGS โดยตรง (ไม่ต้องใช้ FNSTSW+SAHF)
    fcomi   st0, st1            ; เปรียบเทียบ ST(0) กับ ST(1)
    
    ; ZF=1, CF=0: equal
    ; ZF=0, CF=0: ST(0) > ST(1)
    ; ZF=0, CF=1: ST(0) < ST(1)
    
    jz      .equal_modern
    jc      .st0_less           ; CF=1 หมายถึง ST(0) < ST(1)
    ; ST(0) > ST(1)
    jmp     .compare_done
.equal_modern:
    jmp     .compare_done
.st0_less:
.compare_done:
    fstp    st0
    fstp    st0
    ret

; ============================================================
; ฟังก์ชัน: fucom_demo
; FUCOM - Unordered Compare (สำหรับ NaN)
; ============================================================
fucom_demo:
    fld     qword [nan_val]     ; ST(0) = NaN
    fld     qword [num1]        ; ST(0) = 5.5, ST(1) = NaN
    
    ; FUCOM: เหมือน FCOM แต่ไม่ raise exception สำหรับ NaN
    fucom   st1                 ; เปรียบเทียบกับ NaN - unordered
    
    fnstsw  ax
    sahf
    jp      .is_nan             ; PF=1 หมายถึง unordered (NaN)
    jmp     .not_nan
.is_nan:
    ; มี NaN ใน comparison
.not_nan:
    fstp    st0
    fstp    st0
    ret

; ============================================================
; ฟังก์ชัน: max_float
; หาค่ามากสุดระหว่าง float สองค่า
; Input: เข้าผ่าน stack (a, b)
; Output: ST(0) = max(a, b)
; ============================================================
max_float:
    ; ถือว่า ST(0) = b, ST(1) = a
    fcomi   st0, st1            ; เปรียบเทียบ ST(0) กับ ST(1)
    jnb     .b_is_max           ; ST(0) >= ST(1), b >= a
    ; a > b: ต้องการ a
    fxch    st1                 ; swap, ST(0) = a
    fstp    st1                 ; ลบ b (อยู่ใน ST(1) ตอนนี้ ST(0))
    ; หรือ: fxch, pop top
    ret
.b_is_max:
    ; b >= a
    fstp    st1                 ; ลบ a
    ret
```

---

## 6. FXAM — Examine Register

```nasm
; ============================================================
; ตัวอย่าง: FXAM - Examine FPU Register
; ============================================================

section .data
    test_vals:
        dq  3.14        ; normal number
        dq  0.0         ; zero
        dq  0x7FF0000000000000  ; +Infinity
        dq  0xFFF0000000000000  ; -Infinity
        dq  0x7FF8000000000000  ; NaN (Quiet)
        dq  0x0008000000000001  ; Denormal

section .text

; ============================================================
; ฟังก์ชัน: examine_register
; ตรวจสอบชนิดของค่าใน ST(0)
; ============================================================
examine_register:
    fld     qword [test_vals]   ; โหลดค่าทดสอบ
    
    fxam                        ; ตรวจสอบ ST(0)
    ; ผลลัพธ์ใน C0, C2, C3:
    ; C3 C2 C0
    ;  0  0  0 = Unsupported
    ;  0  0  1 = NaN
    ;  0  1  0 = Normal (finite)
    ;  0  1  1 = Infinity
    ;  1  0  0 = Zero
    ;  1  0  1 = Empty (register empty)
    ;  1  1  0 = Denormal
    ;  1  1  1 = Denormal
    ; C1 = Sign bit
    
    fnstsw  ax                  ; โหลด Status Word
    
    ; แยก C0, C2, C3
    mov     bx, ax
    and     bx, 0x4700          ; mask C0(bit8), C2(bit10), C3(bit14)
    
    ; ตรวจสอบแต่ละ case
    cmp     bx, 0x0200          ; C3=0, C2=1, C0=0 = Normal?
    je      .is_normal
    cmp     bx, 0x0400          ; C3=1, C2=0, C0=0 = Zero?
    je      .is_zero
    cmp     bx, 0x0300          ; C3=0, C2=1, C0=1 = Infinity?
    je      .is_infinity
    cmp     bx, 0x0100          ; C3=0, C2=0, C0=1 = NaN?
    je      .is_nan
    cmp     bx, 0x0600          ; C3=1, C2=1, C0=0 = Denormal?
    je      .is_denormal
    
.is_normal:
    ; ค่าปกติ
    jmp     .examine_done
.is_zero:
    ; เป็นศูนย์
    jmp     .examine_done
.is_infinity:
    ; เป็น infinity
    jmp     .examine_done
.is_nan:
    ; เป็น NaN
    jmp     .examine_done
.is_denormal:
    ; เป็น denormal
.examine_done:
    fstp    st0                 ; ทิ้งค่า
    ret
```

---

## 7. FTST — Test Against Zero

```nasm
; ============================================================
; ตัวอย่าง: FTST - Test ST(0) against Zero
; ============================================================

section .data
    positive_val    dq  5.5
    negative_val    dq  -3.7
    zero_val        dq  0.0

section .text

; ============================================================
; ฟังก์ชัน: test_against_zero
; ตรวจสอบว่าค่าเป็น positive, negative, หรือ zero
; ============================================================
test_against_zero:
    ; --- FTST ---
    fld     qword [positive_val]    ; ST(0) = 5.5
    ftst                            ; เปรียบเทียบ ST(0) กับ 0.0
    
    ; ผลลัพธ์ใน C0, C2, C3 (เหมือน FCOM กับ 0.0):
    ; C3=0, C0=0: ST(0) > 0 (positive)
    ; C3=0, C0=1: ST(0) < 0 (negative)
    ; C3=1, C0=0: ST(0) = 0 (zero)
    ; C3=1, C0=1: Unordered (NaN)
    
    fnstsw  ax
    sahf
    
    ja      .positive           ; ST(0) > 0
    jb      .negative           ; ST(0) < 0
    jz      .zero               ; ST(0) = 0
    jp      .nan_detected       ; PF set = NaN
    
.positive:
    ; บวก
    jmp     .test_done
.negative:
    ; ลบ
    jmp     .test_done
.zero:
    ; ศูนย์
    jmp     .test_done
.nan_detected:
    ; NaN
.test_done:
    fstp    st0
    ret

; ============================================================
; ฟังก์ชัน: safe_reciprocal
; คำนวณ 1/x แบบปลอดภัย (ตรวจสอบ zero)
; Input: ST(0) = x
; Output: ST(0) = 1/x หรือ +Infinity ถ้า x=0
; ============================================================
safe_reciprocal:
    ftst                        ; ทดสอบกับ zero
    fnstsw  ax
    sahf
    jz      .handle_zero        ; ถ้า x = 0
    
    ; x ≠ 0: คำนวณ 1/x
    fld1                        ; ST(0) = 1.0, ST(1) = x
    fxch    st1                 ; ST(0) = x, ST(1) = 1.0
    fdivp   st1, st0            ; ST(0) = 1.0 / x
    ret
    
.handle_zero:
    ; x = 0: return +Infinity
    fstp    st0                 ; ลบ 0
    fld1
    fldz
    fdivp   st1, st0            ; 1/0 = +Infinity (ถ้าไม่ mask exception)
    ret
```

---

## 8. Transcendental Functions

### FSIN, FCOS, FPTAN, FPATAN

```nasm
; ============================================================
; ตัวอย่าง: Transcendental Functions
; ============================================================

section .data
    angle_rad   dq  1.0472     ; 60 degrees ใน radians (π/3)
    angle_45    dq  0.7854     ; 45 degrees ใน radians (π/4)
    x_val       dq  3.0
    y_val       dq  4.0
    
    sin_result  dq  0.0
    cos_result  dq  0.0
    tan_result  dq  0.0
    atan_result dq  0.0

section .text

; ============================================================
; ฟังก์ชัน: trig_functions_demo
; แสดง trigonometric functions
; ============================================================
trig_functions_demo:
    ; --- FSIN: Sine ---
    fld     qword [angle_rad]   ; ST(0) = 1.0472 (60°)
    fsin                        ; ST(0) = sin(60°) ≈ 0.8660
    fstp    qword [sin_result]
    
    ; --- FCOS: Cosine ---
    fld     qword [angle_rad]   ; ST(0) = 1.0472 (60°)
    fcos                        ; ST(0) = cos(60°) ≈ 0.5
    fstp    qword [cos_result]
    
    ; --- FSINCOS: Sine and Cosine พร้อมกัน (มีประสิทธิภาพกว่า) ---
    fld     qword [angle_rad]   ; ST(0) = 1.0472
    fsincos                     ; ST(0) = cos(60°), ST(1) = sin(60°)
    fstp    qword [cos_result]  ; เก็บ cos
    fstp    qword [sin_result]  ; เก็บ sin
    
    ; --- FPTAN: Partial Tangent ---
    ; FPTAN ให้ผลลัพธ์เป็น Y/X ไม่ใช่ tan โดยตรง
    fld     qword [angle_45]    ; ST(0) = π/4 (45°)
    fptan                       ; ST(0) = 1.0 (X), ST(1) = 1.0 (Y)
                                ; tan(45°) = Y/X = 1.0/1.0 = 1.0
    fdivp   st1, st0            ; ST(0) = Y/X = tan(angle)
    fstp    qword [tan_result]
    
    ; --- FPATAN: Partial Arctangent ---
    ; FPATAN คำนวณ atan2(ST(1), ST(0)) = atan(y/x)
    fld     qword [x_val]       ; ST(0) = 3.0 (X)
    fld     qword [y_val]       ; ST(0) = 4.0 (Y), ST(1) = 3.0
    fpatan                      ; ST(0) = atan2(4.0, 3.0) ≈ 0.9273 rad
    fstp    qword [atan_result] ; ≈ 53.13°
    
    ret

; ============================================================
; ฟังก์ชัน: fsqrt_fabs_fchs_demo
; FSQRT, FABS, FCHS
; ============================================================
fsqrt_fabs_fchs_demo:
    section .data
    .val    dq  2.0
    .neg    dq  -5.7
    
    section .text
    
    ; --- FSQRT: Square Root ---
    fld     qword [.val]        ; ST(0) = 2.0
    fsqrt                       ; ST(0) = √2 ≈ 1.41421356
    fstp    qword [.val]
    
    ; --- FABS: Absolute Value ---
    fld     qword [.neg]        ; ST(0) = -5.7
    fabs                        ; ST(0) = 5.7 (absolute value)
    fstp    qword [.neg]
    
    ; --- FCHS: Change Sign (Negate) ---
    fld     qword [.neg]        ; ST(0) = 5.7
    fchs                        ; ST(0) = -5.7
    fstp    qword [.neg]
    
    ret

; ============================================================
; ฟังก์ชัน: compute_hypotenuse
; คำนวณ hypotenuse: c = √(a² + b²)
; Input: ST(0) = a, ST(1) = b  (push a ก่อน b)
; หรือส่งผ่าน memory
; ============================================================
compute_hypotenuse:
    ; รับ a, b จาก parameter
    push    ebp
    mov     ebp, esp
    
    ; ถือว่า a อยู่ที่ [ebp+8], b อยู่ที่ [ebp+16] (double)
    fld     qword [ebp+8]       ; ST(0) = a
    fmul    st0, st0            ; ST(0) = a²
    fld     qword [ebp+16]      ; ST(0) = b, ST(1) = a²
    fmul    st0, st0            ; ST(0) = b²
    fadd                        ; ST(0) = a² + b²
    fsqrt                       ; ST(0) = √(a² + b²)
    
    pop     ebp
    ret
```

---

## 9. Logarithmic Functions

```nasm
; ============================================================
; ตัวอย่าง: Logarithmic Functions (ไม่มีใน x87 โดยตรง)
; ต้องสร้างจาก FYL2X และ constants
; ============================================================

section .data
    val_for_log     dq  100.0
    log_result      dq  0.0

section .text

; ============================================================
; ฟังก์ชัน: compute_log2
; คำนวณ log₂(x)
; ใช้ FYL2X: ST(0) = ST(1) * log₂(ST(0))
; Input: ST(0) = x
; Output: ST(0) = log₂(x)
; ============================================================
compute_log2:
    ; FYL2X คำนวณ y * log₂(x)
    ; ต้องการ y=1 สำหรับ log₂(x) ล้วนๆ
    fld1                        ; ST(0) = 1.0 (y)
    fxch    st1                 ; ST(0) = x, ST(1) = 1.0
    fyl2x                       ; ST(0) = 1.0 * log₂(x) = log₂(x)
    ret

; ============================================================
; ฟังก์ชัน: compute_ln
; คำนวณ ln(x) = log₂(x) / log₂(e)
; หรือใช้ ln(x) = log₂(x) * ln(2)
; ============================================================
compute_ln:
    ; ln(x) = log₂(x) * ln(2)
    ; ln(2) = FLDL2E constant ไม่ตรง... ใช้ FYL2X กับ FLDLN2
    ; FLDLN2 โหลด ln(2) ≈ 0.693147
    fldln2                      ; ST(0) = ln(2), ST(1) = x
    fxch    st1                 ; ST(0) = x, ST(1) = ln(2)
    fyl2x                       ; ST(0) = ln(2) * log₂(x) = ln(x)
    ret

; ============================================================
; ฟังก์ชัน: compute_log10
; คำนวณ log₁₀(x)
; log₁₀(x) = log₂(x) * log₁₀(2) = log₂(x) * FLDLG2
; ============================================================
compute_log10:
    ; FLDLG2 โหลด log₁₀(2) ≈ 0.30103
    fldlg2                      ; ST(0) = log10(2), ST(1) = x
    fxch    st1                 ; ST(0) = x, ST(1) = log10(2)
    fyl2x                       ; ST(0) = log10(2) * log2(x) = log10(x)
    ret

; ============================================================
; ฟังก์ชัน: compute_2_to_x
; คำนวณ 2^x ใช้ F2XM1 และ FSCALE
; ============================================================
compute_2_to_x:
    ; 2^x:
    ; 1. แยก x = integer part (n) + fractional part (f)
    ; 2. 2^x = 2^n * 2^f
    ; 3. F2XM1 คำนวณ 2^f - 1 (ใช้ได้กับ -1 ≤ f ≤ 1)
    ; 4. FSCALE: ST(0) = ST(0) * 2^ST(1) (integer scale)
    
    ; สมมติ ST(0) = x
    fld     st0                 ; copy x: ST(0) = x, ST(1) = x
    
    ; round x ลงเป็น integer
    frndint                     ; ST(0) = floor(x) roughly (ขึ้นกับ rounding mode)
    fxch    st1                 ; ST(0) = x, ST(1) = n (integer part)
    fsub    st0, st1            ; ST(0) = f = x - n (fractional)
    
    ; คำนวณ 2^f - 1
    f2xm1                       ; ST(0) = 2^f - 1
    fld1
    faddp   st1, st0            ; ST(0) = 2^f
    
    ; scale โดย 2^n
    fscale                      ; ST(0) = 2^f * 2^n = 2^x
    fstp    st1                 ; pop ST(1) (n)
    ret
```

---

## 10. FLD Constants — Built-in Constants

```nasm
; ============================================================
; ตัวอย่าง: FLD Constants
; ============================================================

section .data
    pi_result   dq  0.0
    e_result    dq  0.0
    log2e_res   dq  0.0

section .text

; ============================================================
; ฟังก์ชัน: load_constants_demo
; แสดงการโหลด constants สำเร็จรูป
; ============================================================
load_constants_demo:
    ; --- FLD1: Load +1.0 ---
    fld1                        ; ST(0) = 1.0
    fstp    qword [e_result]    ; เก็บและ pop
    
    ; --- FLDZ: Load +0.0 ---
    fldz                        ; ST(0) = 0.0
    fstp    qword [e_result]
    
    ; --- FLDPI: Load π ---
    fldpi                       ; ST(0) = π = 3.14159265358979...
    fstp    qword [pi_result]
    
    ; --- FLDL2E: Load log₂(e) ---
    fldl2e                      ; ST(0) = log₂(e) ≈ 1.44269504
    fstp    qword [log2e_res]
    
    ; --- FLDL2T: Load log₂(10) ---
    fldl2t                      ; ST(0) = log₂(10) ≈ 3.32192809
    fstp    qword [log2e_res]
    
    ; --- FLDLG2: Load log₁₀(2) ---
    fldlg2                      ; ST(0) = log₁₀(2) ≈ 0.30103000
    fstp    qword [log2e_res]
    
    ; --- FLDLN2: Load ln(2) ---
    fldln2                      ; ST(0) = ln(2) ≈ 0.69314718
    fstp    qword [log2e_res]
    
    ret

; ============================================================
; ฟังก์ชัน: compute_e_from_scratch
; คำนวณค่า e จาก FPU constants
; e = 2^(log₂(e)) = 2^(FLDL2E)
; ============================================================
compute_e:
    fldl2e                      ; ST(0) = log₂(e)
    ; ใช้ procedure compute_2_to_x ที่เขียนไว้ข้างบน
    call    compute_2_to_x      ; ST(0) = e ≈ 2.71828182845
    ret

; ============================================================
; ฟังก์ชัน: degrees_to_radians
; แปลง degrees เป็น radians: rad = deg * (π/180)
; Input: ST(0) = degrees
; Output: ST(0) = radians
; ============================================================
degrees_to_radians:
    fldpi                       ; ST(0) = π, ST(1) = deg
    fxch    st1                 ; ST(0) = deg, ST(1) = π
    fmulp   st1, st0            ; ST(0) = deg * π
    ; หาร 180
    push    dword 180
    fild    dword [esp]         ; ST(0) = 180.0
    add     esp, 4
    fdivp   st1, st0            ; ST(0) = (deg * π) / 180
    ret
```

---

## 11. FINIT/FNINIT — Initialize FPU

```nasm
; ============================================================
; ตัวอย่าง: FPU Initialization
; ============================================================

section .text

; ============================================================
; ฟังก์ชัน: init_fpu_demo
; แสดงการ initialize FPU
; ============================================================
init_fpu_demo:
    ; --- FINIT: Initialize FPU (wait for pending exceptions) ---
    finit                       ; reset FPU state ทั้งหมด
                                ; ล้าง register stack (set all to empty)
                                ; ล้าง exception flags
                                ; set control word เป็น default (0x037F)
                                ; set status word เป็น 0
                                ; wait สำหรับ pending exceptions ก่อน
    
    ; --- FNINIT: Initialize FPU (no-wait version) ---
    fninit                      ; เหมือน FINIT แต่ไม่ wait exception
                                ; ใช้เมื่อต้องการความรวดเร็ว
    
    ; หลังจาก FINIT:
    ; Control Word = 0x037F (mask all exceptions, extended precision)
    ; Status Word = 0x0000
    ; Tag Word = 0xFFFF (all empty)
    
    ret

; ============================================================
; ฟังก์ชัน: fpu_state_save_restore
; บันทึกและกู้คืนสถานะ FPU
; ============================================================
section .bss
    fpu_state_buf   resb 108    ; FSAVE ใช้ 108 bytes
    fpu_env_buf     resb 28     ; FSTENV ใช้ 28 bytes

section .text
fpu_state_save_restore:
    ; --- FSAVE: Save FPU State (108 bytes) ---
    fsave   [fpu_state_buf]     ; บันทึก FPU state ทั้งหมดรวมถึง register stack
                                ; หลังจาก save จะ reinitialize FPU
    
    ; ทำงานอื่นๆ...
    fld1
    fldpi
    
    ; --- FRSTOR: Restore FPU State ---
    frstor  [fpu_state_buf]     ; กู้คืนสถานะ FPU ที่บันทึกไว้
    
    ; --- FSTENV: Save FPU Environment (ไม่รวม register stack) ---
    fstenv  [fpu_env_buf]       ; บันทึกแค่ control/status/tag words + pointers
    
    ; --- FLDENV: Load FPU Environment ---
    fldenv  [fpu_env_buf]       ; กู้คืน environment
    
    ret
```

---

## 12. FLDCW/FSTCW — Control Word Management

```nasm
; ============================================================
; ตัวอย่าง: Control Word Management
; ============================================================

section .data
    ; Control Word bits:
    ; Bits 0-5: Exception Masks (1=mask, 0=unmask)
    ;   Bit 0: IM - Invalid Operation
    ;   Bit 1: DM - Denormal
    ;   Bit 2: ZM - Zero Divide
    ;   Bit 3: OM - Overflow
    ;   Bit 4: UM - Underflow
    ;   Bit 5: PM - Precision
    ; Bits 8-9: PC - Precision Control
    ;   00 = Single (24-bit mantissa)
    ;   10 = Double (53-bit mantissa)
    ;   11 = Extended (64-bit mantissa)
    ; Bits 10-11: RC - Rounding Control
    ;   00 = Round to nearest
    ;   01 = Round toward -∞
    ;   10 = Round toward +∞
    ;   11 = Round toward 0 (truncate)
    
    ctrl_word_orig  dw  0       ; เก็บค่าเดิม
    ctrl_word_new   dw  0x0F7F  ; Extended precision, round to nearest, all masked

section .text

; ============================================================
; ฟังก์ชัน: set_precision_double
; ตั้งค่า FPU ให้ใช้ double precision (53-bit)
; ============================================================
set_precision_double:
    fstcw   word [ctrl_word_orig]   ; บันทึก control word เดิม
    mov     ax, [ctrl_word_orig]
    and     ax, 0xFCFF              ; clear bits 8-9 (PC)
    or      ax, 0x0200              ; set PC = 10 (double precision)
    mov     [ctrl_word_new], ax
    fldcw   word [ctrl_word_new]    ; โหลด control word ใหม่
    ret

; ============================================================
; ฟังก์ชัน: set_rounding_truncate
; ตั้งค่า rounding เป็น truncate (toward zero)
; ============================================================
set_rounding_truncate:
    fstcw   word [ctrl_word_orig]
    mov     ax, [ctrl_word_orig]
    or      ax, 0x0C00              ; set RC = 11 (truncate)
    mov     [ctrl_word_new], ax
    fldcw   word [ctrl_word_new]
    ret

; ============================================================
; ฟังก์ชัน: fast_float_to_int
; แปลง float เป็น int แบบ truncate (floor toward zero)
; เร็วกว่าการใช้ C library
; ============================================================
section .data
    .saved_cw   dw  0
    .trunc_cw   dw  0

section .text
fast_float_to_int:
    ; บันทึก control word เดิม
    fstcw   word [.saved_cw]
    
    ; ตั้งค่า truncation mode
    mov     ax, [.saved_cw]
    or      ax, 0x0C00
    mov     [.trunc_cw], ax
    fldcw   word [.trunc_cw]
    
    ; แปลงและเก็บ
    ; ST(0) ควรมีค่า float อยู่แล้ว
    sub     esp, 4
    fist    dword [esp]         ; แปลงเป็น int (truncated)
    pop     eax                 ; ผลลัพธ์อยู่ใน EAX
    
    ; กู้คืน control word เดิม
    fldcw   word [.saved_cw]
    ret

; ============================================================
; ฟังก์ชัน: set_extended_precision
; ตั้งค่าให้ใช้ extended precision (80-bit, default)
; ============================================================
set_extended_precision:
    fstcw   word [ctrl_word_orig]
    mov     ax, [ctrl_word_orig]
    and     ax, 0xFCFF              ; clear bits 8-9
    or      ax, 0x0300              ; set PC = 11 (extended precision)
    mov     [ctrl_word_new], ax
    fldcw   word [ctrl_word_new]
    ret
```

---

## 13. Precision and Rounding Modes

```nasm
; ============================================================
; ตัวอย่าง: Precision และ Rounding Modes
; ============================================================

section .data
    val_2_5     dq  2.5
    val_3_5     dq  3.5
    val_neg_2_5 dq  -2.5
    
    int_result1 dd  0
    int_result2 dd  0
    int_result3 dd  0
    int_result4 dd  0
    
    ctrl_orig   dw  0
    ctrl_nearest dw 0x037F  ; Round to nearest (even)
    ctrl_down    dw 0x077F  ; Round toward -∞
    ctrl_up      dw 0x0B7F  ; Round toward +∞
    ctrl_trunc   dw 0x0F7F  ; Truncate toward 0

section .text

; ============================================================
; ฟังก์ชัน: rounding_demo
; แสดงการ rounding แบบต่างๆ
; ============================================================
rounding_demo:
    push    ebp
    mov     ebp, esp
    
    ; ==========================================
    ; Round to Nearest (Banker's Rounding)
    ; ==========================================
    fldcw   word [ctrl_nearest]
    
    fld     qword [val_2_5]     ; 2.5
    frndint                     ; 2.5 → 2 (round to even)
    fistp   dword [int_result1]
    
    fld     qword [val_3_5]     ; 3.5
    frndint                     ; 3.5 → 4 (round to even)
    fistp   dword [int_result2]
    
    ; ==========================================
    ; Round toward -Infinity (Floor)
    ; ==========================================
    fldcw   word [ctrl_down]
    
    fld     qword [val_2_5]     ; 2.5
    frndint                     ; 2.5 → 2
    fistp   dword [int_result1]
    
    fld     qword [val_neg_2_5] ; -2.5
    frndint                     ; -2.5 → -3
    fistp   dword [int_result2]
    
    ; ==========================================
    ; Round toward +Infinity (Ceil)
    ; ==========================================
    fldcw   word [ctrl_up]
    
    fld     qword [val_2_5]     ; 2.5
    frndint                     ; 2.5 → 3
    fistp   dword [int_result1]
    
    fld     qword [val_neg_2_5] ; -2.5
    frndint                     ; -2.5 → -2
    fistp   dword [int_result2]
    
    ; ==========================================
    ; Truncate (Round toward Zero)
    ; ==========================================
    fldcw   word [ctrl_trunc]
    
    fld     qword [val_2_5]     ; 2.5
    frndint                     ; 2.5 → 2
    fistp   dword [int_result1]
    
    fld     qword [val_neg_2_5] ; -2.5
    frndint                     ; -2.5 → -2
    fistp   dword [int_result2]
    
    pop     ebp
    ret

; ============================================================
; ฟังก์ชัน: floor_ceil_round
; Implement floor, ceil, round functions
; ============================================================
section .data
    .saved_cw   dw  0

section .text

; floor(x): ปัดลง
fpu_floor:
    fstcw   [.saved_cw]
    fldcw   word [ctrl_down]    ; round toward -∞
    frndint
    fldcw   [.saved_cw]
    ret

; ceil(x): ปัดขึ้น
fpu_ceil:
    fstcw   [.saved_cw]
    fldcw   word [ctrl_up]      ; round toward +∞
    frndint
    fldcw   [.saved_cw]
    ret

; round(x): ปัดปกติ
fpu_round:
    fstcw   [.saved_cw]
    fldcw   word [ctrl_nearest] ; round to nearest
    frndint
    fldcw   [.saved_cw]
    ret

; trunc(x): ตัดทศนิยมทิ้ง
fpu_trunc:
    fstcw   [.saved_cw]
    fldcw   word [ctrl_trunc]   ; truncate
    frndint
    fldcw   [.saved_cw]
    ret
```

---

## 14. x87 vs SSE สำหรับ Float

```nasm
; ============================================================
; ตัวอย่าง: เปรียบเทียบ x87 vs SSE
; ============================================================

; === x87 Method ===
section .data
    a_x87   dq  1.5
    b_x87   dq  2.7
    c_x87   dq  3.3
    r_x87   dq  0.0

section .text

; x87: (a + b) * c
x87_calc:
    fld     qword [a_x87]       ; ST(0) = a (โหลด 80-bit internal)
    fadd    qword [b_x87]       ; ST(0) = a + b (80-bit precision)
    fmul    qword [c_x87]       ; ST(0) = (a+b)*c (80-bit precision)
    fstp    qword [r_x87]       ; เก็บเป็น 64-bit (truncate to double)
    ret

; === SSE2 Method ===
section .data
    a_sse   dq  1.5
    b_sse   dq  2.7
    c_sse   dq  3.3
    r_sse   dq  0.0

section .text

; SSE2: (a + b) * c
sse2_calc:
    movsd   xmm0, [a_sse]       ; xmm0 = a (64-bit)
    addsd   xmm0, [b_sse]       ; xmm0 = a + b (64-bit)
    mulsd   xmm0, [c_sse]       ; xmm0 = (a+b)*c (64-bit)
    movsd   [r_sse], xmm0       ; เก็บผลลัพธ์
    ret

; ============================================================
; ตาราง: x87 vs SSE เปรียบเทียบ
; ============================================================
;
; Feature              x87                     SSE/SSE2
; ─────────────────────────────────────────────────────
; Precision           80-bit internal          64-bit (double)
; Register Model      Stack (ST0-ST7)          Flat (XMM0-XMM7/15)
; SIMD Support        ไม่มี                   มี (2 doubles/4 singles)
; Transcendentals     มีใน hardware            ต้องใช้ library
; Speed (modern)      ช้ากว่า SSE             เร็วกว่า (pipelining ดีกว่า)
; Pipelining          ซับซ้อน (stack dep)      ง่ายกว่า (flat regs)
; Exception Model     เดิม (x87)              SSE (MXCSR)
; 64-bit Support      ต้องใช้ legacy REX       native
; Compiler Support    ลด, บาง compiler หยุด   preferred
;
; เมื่อไหร่ควรใช้ x87?
; - ต้องการ 80-bit precision (scientific computing)
; - โค้ดเก่าที่ต้อง maintain
; - transcendental functions โดยไม่ต้องการ math library
;
; เมื่อไหร่ควรใช้ SSE?
; - ต้องการ performance สูง
; - SIMD (parallel computation)
; - Modern applications

; ============================================================
; Benchmark: x87 vs SSE2 สำหรับ loop computation
; ============================================================
section .data
    N_ELEMS     equ 1000000
    array       times N_ELEMS dq 0.0   ; ขนาดใหญ่ - จะอยู่ใน bss
    sum_x87     dq  0.0
    sum_sse2    dq  0.0

section .text

; x87 sum: ช้ากว่าเพราะ stack dependency
sum_array_x87:
    ; ESI = pointer to array, ECX = count
    fldz                        ; ST(0) = 0 (accumulator)
.loop_x87:
    fadd    qword [esi]         ; ST(0) += array[i]  (dependency chain!)
    add     esi, 8
    loop    .loop_x87
    ret                         ; ST(0) = sum

; SSE2 sum: เร็วกว่า สามารถ unroll และ vectorize
sum_array_sse2:
    ; ESI = pointer to array, ECX = count
    xorpd   xmm0, xmm0         ; xmm0 = 0 (accumulator)
.loop_sse2:
    addsd   xmm0, [esi]         ; xmm0 += array[i]
    add     esi, 8
    loop    .loop_sse2
    movsd   [sum_sse2], xmm0
    ret
    
; SSE2 vectorized sum (2 doubles ต่อ iteration):
sum_array_sse2_vec:
    xorpd   xmm0, xmm0         ; accumulator pair
.loop_vec:
    addpd   xmm0, [esi]         ; เพิ่ม 2 doubles พร้อมกัน
    add     esi, 16             ; 2 * 8 bytes
    sub     ecx, 2
    jnz     .loop_vec
    ; รวม 2 lanes
    movhlps xmm1, xmm0         ; xmm1 = high lane
    addsd   xmm0, xmm1         ; xmm0 = sum of both
    movsd   [sum_sse2], xmm0
    ret
```

---

## 15. Implementing Math Functions with x87

```nasm
; ============================================================
; ตัวอย่าง: การ implement math functions ด้วย x87
; ============================================================

section .data
    math_pi     dq  3.141592653589793238
    math_e      dq  2.718281828459045235
    two_pi      dq  6.283185307179586477
    
    ; Lookup table สำหรับ fast approximation
    fast_sin_coeffs:
        dq  -0.10132118          ; coefficient สำหรับ sin approximation
        dq   0.00397137
        dq  -0.00007896224

section .text

; ============================================================
; ฟังก์ชัน: math_pow
; คำนวณ x^y ด้วย x87
; x^y = 2^(y * log₂(x))
; ============================================================
math_pow:
    ; Input: ST(0) = y, ST(1) = x
    ; Output: ST(0) = x^y
    
    ; log₂(x)
    fxch    st1                 ; ST(0) = x, ST(1) = y
    fld1                        ; ST(0) = 1, ST(1) = x, ST(2) = y
    fxch    st1                 ; ST(0) = x, ST(1) = 1, ST(2) = y
    fyl2x                       ; ST(0) = log₂(x), ST(1) = y
    fmulp   st1, st0            ; ST(0) = y * log₂(x)
    
    ; 2^(y * log₂(x))
    call    compute_2_to_x      ; ST(0) = 2^(y * log₂(x)) = x^y
    ret

; ============================================================
; ฟังก์ชัน: math_exp
; คำนวณ e^x
; e^x = 2^(x * log₂(e))
; ============================================================
math_exp:
    ; Input: ST(0) = x
    ; Output: ST(0) = e^x
    
    fldl2e                      ; ST(0) = log₂(e), ST(1) = x
    fmulp   st1, st0            ; ST(0) = x * log₂(e)
    call    compute_2_to_x      ; ST(0) = 2^(x * log₂(e)) = e^x
    ret

; ============================================================
; ฟังก์ชัน: math_sinh
; คำนวณ sinh(x) = (e^x - e^(-x)) / 2
; ============================================================
math_sinh:
    ; Input: ST(0) = x
    ; Output: ST(0) = sinh(x)
    
    fld     st0                 ; copy x: ST(0)=x, ST(1)=x
    call    math_exp            ; ST(0) = e^x
    fxch    st1                 ; ST(0) = x, ST(1) = e^x
    fchs                        ; ST(0) = -x
    call    math_exp            ; ST(0) = e^(-x)
    fsubp   st1, st0            ; ST(0) = e^x - e^(-x)  (WRONG: need to fix order)
    ; ทำใหม่: ST(0) = e^x - e^(-x) = e^x - (1/e^x)
    ; หรือ: sub ST1 - ST0 ก็ได้ขึ้นกับ order
    fxch    st1                 ; แก้ order ถ้าจำเป็น
    fsub    st0, st1            ; ST(0) = e^x - e^(-x)
    fld1
    fadd    st0, st0            ; ST(0) = 2.0
    fdivp   st1, st0            ; ST(0) = (e^x - e^(-x)) / 2
    ret

; ============================================================
; ฟังก์ชัน: math_cosh
; คำนวณ cosh(x) = (e^x + e^(-x)) / 2
; ============================================================
math_cosh:
    fld     st0                 ; copy x
    call    math_exp            ; e^x
    fxch    st1
    fchs
    call    math_exp            ; e^(-x)
    fadd                        ; e^x + e^(-x)
    fld1
    fadd    st0, st0            ; 2.0
    fdivp   st1, st0            ; (e^x + e^(-x)) / 2
    ret

; ============================================================
; ฟังก์ชัน: fast_atan2
; คำนวณ atan2(y, x) ใช้ FPATAN
; ============================================================
fast_atan2:
    ; Input: [esp+4] = y (double), [esp+12] = x (double)
    push    ebp
    mov     ebp, esp
    
    fld     qword [ebp+12]      ; ST(0) = x
    fld     qword [ebp+8]       ; ST(0) = y, ST(1) = x
    fpatan                      ; ST(0) = atan2(y, x)
    
    pop     ebp
    ret 16

; ============================================================
; ฟังก์ชัน: normalize_angle
; ทำให้มุมอยู่ในช่วง [-π, π]
; ============================================================
normalize_angle:
    ; Input: ST(0) = angle in radians
    ; Output: ST(0) = normalized angle in [-π, π]
    
    fldpi                       ; ST(0) = π, ST(1) = angle
    fadd    st0, st0            ; ST(0) = 2π
    fxch    st1                 ; ST(0) = angle, ST(1) = 2π
    
    ; ใช้ FPREM1 สำหรับ IEEE-754 remainder
    fprem1                      ; ST(0) = angle mod 2π (range correct)
    
    fstp    st1                 ; ทิ้ง 2π
    ret
```

---

## 16. Complete Example Program: Scientific Calculator

```nasm
; ============================================================
; โปรแกรม: x87 Scientific Calculator
; คำนวณ: Area of sector, arc length, vector magnitude
; Compile: nasm -f elf32 sci_calc.asm -o sci_calc.o
;          ld -m elf_i386 sci_calc.o -o sci_calc
; ============================================================

section .data
    ; ข้อมูล input
    radius      dq  5.0         ; รัศมี 5 หน่วย
    angle_deg   dq  60.0        ; มุม 60 องศา
    
    ; vectors
    vx1         dq  3.0
    vy1         dq  4.0
    vz1         dq  0.0
    
    vx2         dq  1.0
    vy2         dq  2.0
    vz2         dq  2.0
    
    ; ผลลัพธ์
    sector_area dq  0.0
    arc_len     dq  0.0
    v1_mag      dq  0.0
    v2_mag      dq  0.0
    dot_prod    dq  0.0
    cross_x     dq  0.0
    cross_y     dq  0.0
    cross_z     dq  0.0
    angle_bet   dq  0.0         ; มุมระหว่าง vectors
    
    ; control words
    cw_orig     dw  0
    cw_trunc    dw  0x0F7F

section .bss

section .text
global _start

; ============================================================
; Main program
; ============================================================
_start:
    finit                       ; initialize FPU
    
    ; 1. คำนวณ sector area: A = (1/2) * r² * θ (θ in radians)
    fld     qword [angle_deg]   ; ST(0) = 60.0
    call    degrees_to_radians_v2 ; ST(0) = π/3 ≈ 1.0472
    fst     qword [arc_len]     ; บันทึก radian value สำหรับ arc length
    
    fld     qword [radius]      ; ST(0) = 5.0, ST(1) = θ
    fmul    st0, st0            ; ST(0) = 25.0 = r²
    fmulp   st1, st0            ; ST(0) = r² * θ
    fld1
    fadd    st0, st0            ; ST(0) = 2.0
    fdivp   st1, st0            ; ST(0) = r² * θ / 2
    fstp    qword [sector_area] ; sector area
    
    ; 2. คำนวณ arc length: s = r * θ
    fld     qword [radius]      ; ST(0) = 5.0
    fmul    qword [arc_len]     ; ST(0) = 5.0 * π/3
    fstp    qword [arc_len]     ; arc length
    
    ; 3. คำนวณ magnitude ของ v1 = (3, 4, 0)
    fld     qword [vx1]         ; 3.0
    fmul    st0, st0            ; 9.0
    fld     qword [vy1]         ; 4.0
    fmul    st0, st0            ; 16.0
    faddp   st1, st0            ; 25.0
    fld     qword [vz1]
    fmul    st0, st0
    faddp   st1, st0            ; 25.0 + 0
    fsqrt                       ; 5.0
    fstp    qword [v1_mag]
    
    ; 4. คำนวณ magnitude ของ v2 = (1, 2, 2)
    fld     qword [vx2]         ; 1.0
    fmul    st0, st0            ; 1.0
    fld     qword [vy2]         ; 2.0
    fmul    st0, st0            ; 4.0
    faddp   st1, st0            ; 5.0
    fld     qword [vz2]         ; 2.0
    fmul    st0, st0            ; 4.0
    faddp   st1, st0            ; 9.0
    fsqrt                       ; 3.0
    fstp    qword [v2_mag]
    
    ; 5. Dot product: v1·v2 = x1*x2 + y1*y2 + z1*z2
    fld     qword [vx1]
    fmul    qword [vx2]         ; 3*1 = 3
    fld     qword [vy1]
    fmul    qword [vy2]         ; 4*2 = 8
    faddp   st1, st0            ; 11
    fld     qword [vz1]
    fmul    qword [vz2]         ; 0*2 = 0
    faddp   st1, st0            ; 11
    fstp    qword [dot_prod]
    
    ; 6. Cross product: v1 × v2
    ; cx = y1*z2 - z1*y2
    fld     qword [vy1]
    fmul    qword [vz2]
    fld     qword [vz1]
    fmul    qword [vy2]
    fsubp   st1, st0
    fstp    qword [cross_x]     ; 4*2 - 0*2 = 8
    
    ; cy = z1*x2 - x1*z2
    fld     qword [vz1]
    fmul    qword [vx2]
    fld     qword [vx1]
    fmul    qword [vz2]
    fsubp   st1, st0
    fstp    qword [cross_y]     ; 0*1 - 3*2 = -6
    
    ; cz = x1*y2 - y1*x2
    fld     qword [vx1]
    fmul    qword [vy2]
    fld     qword [vy1]
    fmul    qword [vx2]
    fsubp   st1, st0
    fstp    qword [cross_z]     ; 3*2 - 4*1 = 2
    
    ; 7. มุมระหว่าง v1 และ v2:
    ; cos(θ) = (v1·v2) / (|v1| * |v2|)
    ; θ = arccos(v1·v2 / (|v1|*|v2|))
    fld     qword [dot_prod]    ; 11
    fld     qword [v1_mag]      ; 5
    fmul    qword [v2_mag]      ; 5 * 3 = 15
    fdivp   st1, st0            ; 11/15 ≈ 0.7333...
    ; arccos ไม่มีใน x87 โดยตรง ใช้ arctan:
    ; arccos(x) = atan2(√(1-x²), x)
    fld     st0                 ; copy cos_theta
    fmul    st0, st0            ; cos²(θ)
    fld1
    fsubp   st1, st0            ; 1 - cos²(θ) = sin²(θ)
    fsqrt                       ; sin(θ) = √(1-cos²(θ))
    fxch    st1                 ; ST(0) = cos(θ), ST(1) = sin(θ)
    fpatan                      ; atan2(sin(θ), cos(θ)) = θ
    fstp    qword [angle_bet]   ; ≈ 0.7563 rad ≈ 43.33°
    
    ; Exit
    mov     eax, 1
    xor     ebx, ebx
    int     0x80

; ============================================================
; ฟังก์ชัน helper
; ============================================================
degrees_to_radians_v2:
    ; Input: ST(0) = degrees
    ; Output: ST(0) = radians
    fldpi                       ; ST(0) = π, ST(1) = deg
    fxch    st1                 ; ST(0) = deg, ST(1) = π
    fmulp   st1, st0            ; ST(0) = deg * π
    push    dword 180
    fild    dword [esp]
    add     esp, 4
    fdivp   st1, st0            ; ST(0) = (deg * π) / 180
    ret
```

---

## 17. ARM Floating Point (VFP/NEON)

สำหรับ ARM architecture ใช้ VFP (Vector Floating Point) หรือ NEON

```gas
@ ============================================================
@ ตัวอย่าง: ARM VFP Floating Point Operations
@ Assembler: GAS (GNU Assembler)
@ Target: ARM Cortex-A (ARMv7-A with VFP)
@ ============================================================

.section .data
val_a:      .double 3.14159265358979
val_b:      .double 2.71828182845905
result:     .double 0.0
arr_float:  .float 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0

.section .text
.global _start

@ ============================================================
@ ARM VFP Register Layout:
@ S0-S31: 32 single-precision registers (32-bit each)
@ D0-D31: 32 double-precision registers (64-bit each)
@ Q0-Q15: 16 quad-word registers (128-bit, NEON)
@ D0  = S0:S1, D1 = S2:S3, ...
@ Q0  = D0:D1, Q1 = D2:D3, ...
@ ============================================================

@ ============================================================
@ ฟังก์ชัน: arm_vfp_basic
@ การคำนวณพื้นฐานด้วย VFP
@ ============================================================
arm_vfp_basic:
    push    {r4, lr}
    
    @ โหลดค่า double
    vldr    d0, val_a           @ d0 = 3.14159...
    vldr    d1, val_b           @ d1 = 2.71828...
    
    @ VADD: บวก
    vadd.f64 d2, d0, d1        @ d2 = d0 + d1
    
    @ VSUB: ลบ
    vsub.f64 d3, d0, d1        @ d3 = d0 - d1
    
    @ VMUL: คูณ
    vmul.f64 d4, d0, d1        @ d4 = d0 * d1
    
    @ VDIV: หาร
    vdiv.f64 d5, d0, d1        @ d5 = d0 / d1
    
    @ VSQRT: รากที่สอง
    vsqrt.f64 d6, d0           @ d6 = sqrt(d0)
    
    @ VABS: ค่าสมบูรณ์
    vabs.f64 d7, d3            @ d7 = |d3|
    
    @ VNEG: เปลี่ยนเครื่องหมาย
    vneg.f64 d8, d0            @ d8 = -d0
    
    @ VCMP: เปรียบเทียบ
    vcmp.f64 d0, d1            @ เปรียบเทียบ d0 และ d1
    vmrs    APSR_nzcv, FPSCR   @ ย้าย FPSCR flags ไป CPSR
    bgt     .greater_arm        @ branch if d0 > d1
    
.greater_arm:
    @ เก็บผลลัพธ์
    vstr    d2, result
    
    pop     {r4, pc}

@ ============================================================
@ ฟังก์ชัน: arm_vfp_convert
@ การแปลง type ด้วย VFP
@ ============================================================
arm_vfp_convert:
    @ VCVT: Convert between types
    
    @ int32 → float64
    vmov    s0, r0              @ ย้าย int32 จาก r0 ไป s0
    vcvt.f64.s32 d0, s0        @ แปลง signed int32 → float64
    
    @ float64 → int32 (round to nearest)
    vcvt.s32.f64 s1, d0        @ แปลง float64 → signed int32
    vmov    r1, s1              @ ย้ายผลลัพธ์ไป r1
    
    @ float32 → float64
    vcvt.f64.f32 d1, s2        @ float32 s2 → float64 d1
    
    @ float64 → float32
    vcvt.f32.f64 s3, d1        @ float64 d1 → float32 s3
    
    bx      lr

@ ============================================================
@ ฟังก์ชัน: arm_neon_float
@ NEON SIMD สำหรับ floating point
@ ============================================================
arm_neon_float:
    push    {r4-r7, lr}
    
    @ โหลด 4 floats ใน 1 NEON register
    vld1.32 {q0}, [r0]!        @ q0 = 4 floats จาก memory
    vld1.32 {q1}, [r1]!        @ q1 = 4 floats จาก memory
    
    @ VADD.F32: บวก 4 floats พร้อมกัน
    vadd.f32 q2, q0, q1        @ q2[0:3] = q0[0:3] + q1[0:3]
    
    @ VMUL.F32: คูณ 4 floats พร้อมกัน
    vmul.f32 q3, q0, q1        @ q3[0:3] = q0[0:3] * q1[0:3]
    
    @ VSQRT ไม่มีใน NEON, ใช้ VRSQRTE + VRSQRTS แทน
    @ VRSQRTE: reciprocal square root estimate
    vrsqrte.f32 q4, q0         @ q4 ≈ 1/√q0 (approximate)
    
    @ refine ความแม่นยำ (Newton-Raphson iteration)
    vmul.f32 q5, q0, q4        @ q5 = q0 * estimate
    vrsqrts.f32 q6, q5, q4    @ q6 = refinement factor
    vmul.f32 q4, q4, q6        @ q4 = improved estimate
    
    @ สำหรับ √x: x * (1/√x) = √x
    vmul.f32 q7, q0, q4        @ q7 ≈ √q0
    
    @ เก็บผลลัพธ์
    vst1.32 {q2}, [r2]!        @ เก็บ sum
    vst1.32 {q3}, [r2]!        @ เก็บ product
    vst1.32 {q7}, [r2]!        @ เก็บ sqrt approximation
    
    pop     {r4-r7, pc}

@ ============================================================
@ ฟังก์ชัน: arm_dot_product_neon
@ Dot product ของ 2 float arrays ด้วย NEON
@ Input: r0 = array A, r1 = array B, r2 = count
@ Output: d0 = dot product
@ ============================================================
arm_dot_product_neon:
    push    {r4, lr}
    
    veor    q0, q0, q0          @ q0 = 0 (accumulator, 4 floats)
    
.dot_loop:
    cmp     r2, #4
    blt     .dot_remainder
    
    vld1.32 {q1}, [r0]!        @ load 4 floats from A
    vld1.32 {q2}, [r1]!        @ load 4 floats from B
    vmla.f32 q0, q1, q2        @ q0 += q1 * q2 (multiply-accumulate)
    sub     r2, r2, #4
    b       .dot_loop
    
.dot_remainder:
    @ จัดการ element ที่เหลือ (< 4 elements)
    cbz     r2, .dot_done
.dot_single:
    vld1.32 {s4}, [r0]!
    vld1.32 {s5}, [r1]!
    vmla.f32 s0, s4, s5        @ s0 += s4 * s5
    subs    r2, r2, #1
    bne     .dot_single
    
.dot_done:
    @ รวม 4 lanes ของ q0
    vpadd.f32 d0, d0, d1       @ d0[0] = d0[0]+d0[1], d0[1] = d1[0]+d1[1]
    vpadd.f32 d0, d0, d0       @ d0[0] = sum of all 4
    
    pop     {r4, pc}
```

---

## 18. Advanced x87 Techniques

```nasm
; ============================================================
; เทคนิคขั้นสูง: x87 FPU
; ============================================================

; ============================================================
; เทคนิค 1: Fast Modular Arithmetic
; ============================================================
section .data
    mod_divisor     dq  360.0   ; สำหรับมุม

section .text

; fmod(x, y): คำนวณ x mod y ด้วย FPREM
fpu_fmod:
    ; Input: ST(0) = x (dividend), ST(1) = y (divisor)
    ; หรือโหลดเอง:
    fld     qword [mod_divisor] ; ST(0) = 360, ST(1) = angle
    fxch                        ; ST(0) = angle, ST(1) = 360
.fprem_loop:
    fprem                       ; ST(0) = partial remainder
    fstsw   ax                  ; เช็ค C2 flag
    test    ah, 0x04            ; C2 = 1 หมายถึง reduction ยังไม่เสร็จ
    jnz     .fprem_loop         ; ทำซ้ำจนกว่าจะเสร็จ
    fstp    st1                 ; ลบ divisor
    ret

; ============================================================
; เทคนิค 2: Polynomial Evaluation (Horner's Method)
; ============================================================
; คำนวณ p(x) = a₀ + a₁x + a₂x² + ... + aₙxⁿ
; Horner: p(x) = a₀ + x(a₁ + x(a₂ + ... + x(aₙ₋₁ + xaₙ)...))

section .data
    ; Polynomial coefficients สำหรับ sin(x) ≈ x - x³/6 + x⁵/120 - x⁷/5040
    sin_coeff:
        dq  -1.984090e-4    ; a7 (highest degree first)
        dq   8.333333e-3    ; a5
        dq  -1.666667e-1    ; a3
        dq   1.0            ; a1 (lowest degree)
    sin_degree  dd  3       ; ลดรูปให้เห็นชัด (เฉพาะกำลังคี่)

section .text

; ============================================================
; ฟังก์ชัน: horner_polynomial
; ประเมิน polynomial ด้วย Horner's method
; Input: ST(0) = x
;        ESI = pointer to coefficients (highest first)
;        ECX = degree (number of terms - 1)
; Output: ST(0) = p(x)
; ============================================================
horner_polynomial:
    push    ebp
    mov     ebp, esp
    push    esi
    push    ecx
    
    ; โหลด coefficient แรก (highest degree)
    fld     qword [esi]         ; ST(0) = aₙ
    add     esi, 8
    dec     ecx
    
.horner_loop:
    cmp     ecx, 0
    jl      .horner_done
    
    fmul    st0, st1            ; ST(0) = current * x
    fadd    qword [esi]         ; ST(0) += next coefficient
    add     esi, 8
    dec     ecx
    jmp     .horner_loop
    
.horner_done:
    fstp    st1                 ; ลบ x จาก stack
    pop     ecx
    pop     esi
    pop     ebp
    ret

; ============================================================
; เทคนิค 3: Reciprocal Approximation
; ============================================================
; ใช้ Newton-Raphson สำหรับ 1/x:
; x_{n+1} = x_n * (2 - a * x_n)
; โดย x_0 เริ่มจาก approximation

section .data
    two     dq  2.0
    approx_x    dq  0.0

section .text

; fast_reciprocal: ประมาณค่า 1/x
; Input: ST(0) = a
; Output: ST(0) ≈ 1/a
fast_reciprocal:
    ; Initial approximation: ใช้ FRCP ถ้ามี (SSE) หรือ estimate เอง
    ; สำหรับ x87: ใช้ fdiv โดยตรง (ไม่มี approximation instruction)
    ; แต่สามารถใช้เป็น illustration ของ algorithm:
    
    fld1                        ; ST(0) = 1.0, ST(1) = a
    fxch    st1                 ; ST(0) = a, ST(1) = 1.0
    fdivp   st1, st0            ; ST(0) = 1.0/a (direct)
    ; x87 ทำ direct division ซึ่งแม่นยำ 80-bit
    ; Newton-Raphson ใช้ประโยชน์มากกับ SSE RCPPS instruction
    ret

; ============================================================
; เทคนิค 4: สูตร Quadratic Discriminant
; ============================================================
; แก้ ax² + bx + c = 0
; x = (-b ± √(b² - 4ac)) / (2a)

section .data
    coeff_a     dq  1.0
    coeff_b     dq  -5.0
    coeff_c     dq  6.0
    root1       dq  0.0
    root2       dq  0.0
    discriminant dq 0.0

section .text

; ============================================================
; ฟังก์ชัน: solve_quadratic
; แก้สมการกำลังสอง ax² + bx + c = 0
; ============================================================
solve_quadratic:
    ; คำนวณ discriminant: b² - 4ac
    fld     qword [coeff_b]     ; ST(0) = b
    fmul    st0, st0            ; ST(0) = b²
    
    fld     qword [coeff_a]     ; ST(0) = a, ST(1) = b²
    fmul    qword [coeff_c]     ; ST(0) = ac
    fld1
    fadd    st0, st0            ; ST(0) = 2.0
    fadd    st0, st0            ; ST(0) = 4.0
    fmulp   st1, st0            ; ST(0) = 4ac
    
    fsubp   st1, st0            ; ST(0) = b² - 4ac
    fstp    qword [discriminant]
    
    ; ตรวจสอบ discriminant
    fld     qword [discriminant]
    ftst
    fnstsw  ax
    sahf
    jb      .complex_roots      ; discriminant < 0
    
    fsqrt                       ; ST(0) = √discriminant
    
    ; root1 = (-b + √D) / (2a)
    fld     qword [coeff_b]     ; ST(0) = b, ST(1) = √D
    fchs                        ; ST(0) = -b
    fadd    st0, st1            ; ST(0) = -b + √D
    fld     qword [coeff_a]     ; ST(0) = a
    fld1
    fadd    st0, st0            ; ST(0) = 2.0
    fmulp   st1, st0            ; ST(0) = 2a
    fdivp   st1, st0            ; ST(0) = (-b + √D) / 2a
    fstp    qword [root1]
    
    ; root2 = (-b - √D) / (2a)
    fld     qword [coeff_b]     ; ST(0) = b, ST(1) = √D
    fchs                        ; ST(0) = -b
    fsub    st0, st1            ; ST(0) = -b - √D
    fld     qword [coeff_a]
    fld1
    fadd    st0, st0
    fmulp   st1, st0
    fdivp   st1, st0
    fstp    qword [root2]
    
    fstp    st0                 ; ลบ √D
    jmp     .quad_done
    
.complex_roots:
    ; ไม่มี real roots (กรณี complex)
    ; สามารถคำนวณ real และ imaginary parts ได้
    fstp    st0
    
.quad_done:
    ret
```

---

## 19. Performance Benchmarks

```nasm
; ============================================================
; Benchmark: x87 Performance Tests
; วัดประสิทธิภาพการทำงานของ x87 FPU
; ============================================================

section .data
    bench_iterations    equ 10000000    ; 10 ล้านครั้ง
    bench_input         dq  1.5
    bench_result        dq  0.0
    
    ; RDTSC time stamps
    time_start_lo   dd  0
    time_start_hi   dd  0
    time_end_lo     dd  0
    time_end_hi     dd  0

section .text

; ============================================================
; ฟังก์ชัน: benchmark_x87_fadd
; วัดเวลา FADD ล้วนๆ
; ============================================================
benchmark_x87_fadd:
    ; เริ่มจับเวลา
    rdtsc                           ; Read Time-Stamp Counter
    mov     [time_start_lo], eax
    mov     [time_start_hi], edx
    
    fld     qword [bench_input]     ; โหลดค่าเริ่มต้น
    fldz                            ; accumulator = 0
    
    mov     ecx, bench_iterations
.bench_fadd_loop:
    fadd    st0, st1                ; acc += input
    loop    .bench_fadd_loop
    
    fstp    qword [bench_result]    ; เก็บผลลัพธ์
    fstp    st0                     ; ล้าง stack
    
    ; สิ้นสุดจับเวลา
    rdtsc
    mov     [time_end_lo], eax
    mov     [time_end_hi], edx
    
    ; คำนวณเวลา (cycles)
    ; elapsed = end - start
    mov     eax, [time_end_lo]
    sub     eax, [time_start_lo]
    ; ผล: cycles สำหรับ 10M iterations ≈ latency * count
    
    ret

; ============================================================
; ฟังก์ชัน: benchmark_x87_transcendental
; วัดเวลา transcendental functions
; ============================================================
benchmark_x87_transcendental:
    rdtsc
    mov     [time_start_lo], eax
    mov     [time_start_hi], edx
    
    mov     ecx, 1000               ; transcendentals ช้ากว่ามาก
.bench_trig_loop:
    fld     qword [bench_input]
    fsin                            ; latency ≈ 20-100 cycles
    fstp    qword [bench_result]
    loop    .bench_trig_loop
    
    rdtsc
    mov     [time_end_lo], eax
    mov     [time_end_hi], edx
    
    ret

; ============================================================
; ผลการทดสอบโดยประมาณ (Intel Core i7)
; ============================================================
;
; Operation        x87 Latency    SSE2 Latency
; ─────────────────────────────────────────────
; FADD/ADDSD       3-5 cycles     4 cycles
; FMUL/MULSD       3-5 cycles     5 cycles
; FDIV/DIVSD       9-36 cycles    10-20 cycles
; FSQRT/SQRTSD     16-58 cycles   17-20 cycles
; FSIN             20-100 cycles  N/A (software)
; FCOS             20-100 cycles  N/A (software)
; FPTAN            30-150 cycles  N/A (software)
;
; ข้อสรุป:
; - x87 มี overhead จาก stack model และ 80-bit conversion
; - SSE2 เร็วกว่าสำหรับ basic operations
; - x87 transcendentals ยังคงมีประโยชน์สำหรับ precision สูง
; - Modern compilers มักเลือก SSE2 โดย default
```

---

## 20. Full Working Example: Statistics Calculator

```nasm
; ============================================================
; โปรแกรมสมบูรณ์: Statistics Calculator ด้วย x87 FPU
; คำนวณ mean, variance, standard deviation
; ============================================================
; Compile: nasm -f elf32 stats.asm && ld -m elf_i386 stats.o -o stats
; Run: ./stats

section .data
    ; ข้อมูลสถิติ
    data_set:
        dq  12.5, 15.3, 11.8, 14.2, 13.7, 16.1, 10.9, 12.8, 15.5, 13.3
    data_count  dd  10
    
    ; ผลลัพธ์
    stat_mean   dq  0.0
    stat_var    dq  0.0
    stat_stddev dq  0.0
    stat_min    dq  0.0
    stat_max    dq  0.0
    
    ; Messages
    msg_mean    db  "Mean: ", 0
    msg_var     db  "Variance: ", 0
    msg_std     db  "StdDev: ", 0
    newline     db  10, 0

section .bss
    print_buf   resb 32

section .text
global _start

; ============================================================
; _start: Entry point
; ============================================================
_start:
    finit                       ; initialize FPU (สำคัญมาก!)
    
    ; 1. คำนวณ Mean
    call    compute_mean
    
    ; 2. คำนวณ Variance
    call    compute_variance
    
    ; 3. คำนวณ Standard Deviation
    fld     qword [stat_var]    ; ST(0) = variance
    fsqrt                       ; ST(0) = std dev
    fstp    qword [stat_stddev]
    
    ; 4. หา Min/Max
    call    find_min_max
    
    ; Exit
    mov     eax, 1
    xor     ebx, ebx
    int     0x80

; ============================================================
; ฟังก์ชัน: compute_mean
; คำนวณ arithmetic mean
; ============================================================
compute_mean:
    push    ebp
    mov     ebp, esp
    
    fldz                        ; ST(0) = 0 (sum)
    mov     ecx, [data_count]
    mov     esi, data_set
    
.mean_loop:
    fadd    qword [esi]         ; sum += data[i]
    add     esi, 8
    loop    .mean_loop
    
    ; หาร count เพื่อได้ mean
    fild    dword [data_count]  ; ST(0) = n, ST(1) = sum
    fdivp   st1, st0            ; ST(0) = sum / n
    fstp    qword [stat_mean]
    
    pop     ebp
    ret

; ============================================================
; ฟังก์ชัน: compute_variance
; คำนวณ variance: σ² = Σ(xᵢ - μ)² / n
; ============================================================
compute_variance:
    push    ebp
    mov     ebp, esp
    
    fldz                        ; ST(0) = 0 (sum of squares)
    mov     ecx, [data_count]
    mov     esi, data_set
    
.var_loop:
    fld     qword [esi]         ; ST(0) = x_i
    fsub    qword [stat_mean]   ; ST(0) = x_i - mean
    fmul    st0, st0            ; ST(0) = (x_i - mean)²
    faddp   st1, st0            ; sum += (x_i - mean)²
    add     esi, 8
    loop    .var_loop
    
    fild    dword [data_count]
    fdivp   st1, st0            ; ST(0) = sum / n
    fstp    qword [stat_var]
    
    pop     ebp
    ret

; ============================================================
; ฟังก์ชัน: find_min_max
; หาค่า min และ max ในชุดข้อมูล
; ============================================================
find_min_max:
    push    ebp
    mov     ebp, esp
    
    mov     esi, data_set
    fld     qword [esi]         ; โหลดค่าแรก
    fld     st0                 ; copy: ST(0) = max candidate, ST(1) = min candidate
    add     esi, 8
    mov     ecx, [data_count]
    dec     ecx
    
.minmax_loop:
    fld     qword [esi]         ; ST(0) = x_i, ST(1) = max, ST(2) = min
    
    ; เปรียบเทียบกับ max
    fcomi   st0, st1            ; x_i vs max
    jbe     .check_min          ; x_i <= max, ไม่ต้องอัปเดต max
    fxch    st1                 ; ST(0) = old max, ST(1) = x_i
    fstp    st0                 ; ลบ old max
    fld     st0                 ; copy x_i เพื่อใช้เป็น min check
    fxch    st1                 ; ปรับ stack
    jmp     .next_elem
    
.check_min:
    ; เปรียบเทียบกับ min
    fcomi   st0, st2            ; x_i vs min
    jae     .no_update_min      ; x_i >= min, ไม่ต้องอัปเดต min
    fxch    st2                 ; swap x_i กับ min
    fstp    st0                 ; ลบ old min
    jmp     .next_elem
    
.no_update_min:
    fstp    st0                 ; ลบ x_i (ไม่ใช้)
    
.next_elem:
    add     esi, 8
    loop    .minmax_loop
    
    ; บันทึก max และ min
    fstp    qword [stat_max]    ; เก็บ max (ST(0))
    fstp    qword [stat_min]    ; เก็บ min (ST(0))
    
    pop     ebp
    ret
```

---

## 21. สรุปคำสั่ง x87 FPU (Quick Reference)

```
╔══════════════════════════════════════════════════════════════╗
║              x87 FPU Instruction Quick Reference             ║
╠══════════════════════╦═══════════════════════════════════════╣
║ Category             ║ Instructions                          ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Load                 ║ FLD, FILD, FBLD, FLD1, FLDZ, FLDPI,  ║
║                      ║ FLDL2E, FLDL2T, FLDLG2, FLDLN2       ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Store                ║ FST, FSTP, FIST, FISTP, FISTTP, FBSTP ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Arithmetic           ║ FADD, FADDP, FIADD                    ║
║                      ║ FSUB, FSUBP, FSUBR, FSUBRP, FISUB    ║
║                      ║ FMUL, FMULP, FIMUL                    ║
║                      ║ FDIV, FDIVP, FDIVR, FDIVRP, FIDIV    ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Comparison           ║ FCOM, FCOMP, FCOMPP, FCOMI, FCOMIP   ║
║                      ║ FUCOM, FUCOMP, FUCOMPP, FUCOMI        ║
║                      ║ FTST, FXAM                            ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Transcendental       ║ FSIN, FCOS, FSINCOS, FPTAN, FPATAN    ║
║                      ║ FYL2X, FYL2XP1, F2XM1, FSCALE        ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Misc Math            ║ FSQRT, FABS, FCHS, FRNDINT, FPREM     ║
║                      ║ FPREM1, FXTRACT, FXCH                 ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Control              ║ FINIT, FNINIT, FLDCW, FSTCW, FNCLEX  ║
║                      ║ FSAVE, FNSAVE, FRSTOR                 ║
║                      ║ FSTENV, FLDENV, FSTSW, FNSTSW         ║
╠══════════════════════╬═══════════════════════════════════════╣
║ Stack                ║ FXCH (exchange), FFREE (free register) ║
╚══════════════════════╩═══════════════════════════════════════╝
```

---

## 22. แบบฝึกหัด (Exercises)

### ระดับเบื้องต้น
1. เขียนโปรแกรมคำนวณพื้นที่วงกลม πr² ด้วย x87 FPU
2. แปลงอุณหภูมิจาก Celsius เป็น Fahrenheit: F = C × 9/5 + 32
3. คำนวณ hypotenuse ของสามเหลี่ยมมุมฉาก

### ระดับกลาง
4. Implement factorial function โดยใช้ floating-point multiplication
5. คำนวณ series: sin(x) ≈ x - x³/3! + x⁵/5! - x⁷/7! + ...
6. เขียน function แปลง float เป็น string (ใช้ FIST และ loop)

### ระดับสูง
7. Implement ตัวแก้สมการ Newton-Raphson: xₙ₊₁ = xₙ - f(xₙ)/f'(xₙ)
8. คำนวณ Fast Fourier Transform (FFT) โดยใช้ x87
9. เขียน matrix multiplication 4x4 ด้วย x87 เปรียบเทียบกับ SSE

---

## สรุป (Summary)

ใน Part 041 นี้เราได้เรียนรู้:

1. **x87 Register Stack** — โครงสร้าง ST(0)-ST(7) และ 80-bit extended precision
2. **Load/Store Operations** — FLD, FST, FSTP, FILD, FIST สำหรับ float/integer
3. **Arithmetic** — FADD, FSUB, FMUL, FDIV พร้อม variants
4. **Integer Arithmetic** — FIADD, FISUB, FIMUL, FIDIV
5. **Comparison** — FCOM, FCOMI, FUCOM สำหรับ comparison operations
6. **FXAM/FTST** — ตรวจสอบชนิดของค่าและเปรียบเทียบกับ zero
7. **Transcendentals** — FSIN, FCOS, FPTAN, FPATAN, FSQRT, FABS, FCHS
8. **Constants** — FLD1, FLDZ, FLDPI, FLDL2E, FLDLG2, FLDLN2
9. **FPU Control** — FINIT, FLDCW, FSTCW สำหรับ precision/rounding
10. **x87 vs SSE** — เมื่อไหร่ควรใช้อะไร
11. **Math Functions** — Implementation ของ exp, ln, pow, atan2
12. **ARM VFP/NEON** — Floating point บน ARM architecture

---

**Part ถัดไป:** Part 042 — SSE/SSE2 Programming (Streaming SIMD Extensions)

---

*Assembly Programming Course — Part 041*
*x87 FPU Programming*
*เนื้อหา: Thai/English Mixed | ระดับ: Intermediate to Advanced*
