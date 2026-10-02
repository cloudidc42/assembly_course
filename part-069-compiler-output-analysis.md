# Part 069: Compiler Output Analysis

## การวิเคราะห์ Output จาก Compiler

---

## บทนำ (Introduction)

การทำความเข้าใจว่า compiler สร้าง assembly code อย่างไรนั้นเป็นทักษะที่สำคัญมากสำหรับโปรแกรมเมอร์ระดับ systems ทุกคน เมื่อเราเขียน C/C++ code และ compiler แปลงมันเป็น machine code ผ่านขั้นตอนการ optimize หลายชั้น การอ่าน assembly output ช่วยให้เราเข้าใจว่า code ของเราทำงานได้ดีแค่ไหน และจะปรับปรุงมันได้อย่างไร

ใน Part นี้เราจะเรียนรู้:
- วิธีใช้ `gcc -S` เพื่อ generate assembly
- ความแตกต่างของ optimization levels ตั้งแต่ `-O0` ถึง `-O3` และ `-Ofast`
- การอ่านและวิเคราะห์ compiler output
- Vectorization และ auto-optimization
- Compiler hints และ attributes
- เครื่องมือ Godbolt Compiler Explorer
- LLVM MCA analysis

---

## 1. การ Generate Assembly ด้วย GCC

### 1.1 คำสั่ง gcc -S

```bash
# Generate assembly file จาก C source
gcc -S source.c -o output.s

# ตัวอย่างพื้นฐาน
gcc -S hello.c
# จะสร้าง hello.s ใน directory เดียวกัน
```

flag `-S` บอก compiler ให้หยุดหลังจากขั้นตอน compilation และ output assembly code แทนที่จะสร้าง object file หรือ executable

### 1.2 AT&T Syntax vs Intel Syntax

โดย default GCC ใช้ **AT&T syntax** ซึ่งมีลักษณะดังนี้:

```asm
; AT&T syntax (default GCC)
movl    $5, %eax          # ย้าย immediate value 5 ไปยัง eax
addl    %ebx, %eax        # eax = eax + ebx (destination อยู่หลัง)
```

แต่สำหรับคนที่คุ้นเคยกับ Intel syntax ให้ใช้ flag `-masm=intel`:

```bash
# ใช้ Intel syntax
gcc -S -masm=intel source.c -o output.s
```

```asm
; Intel syntax (-masm=intel)
mov     eax, 5            ; ย้าย immediate value 5 ไปยัง eax
add     eax, ebx          ; eax = eax + ebx (destination อยู่ก่อน)
```

### 1.3 ความแตกต่างระหว่าง AT&T และ Intel Syntax

| ลักษณะ | AT&T | Intel |
|--------|------|-------|
| Operand order | source, dest | dest, source |
| Register prefix | `%eax` | `eax` |
| Immediate prefix | `$5` | `5` |
| Memory operand | `(%rax)` | `[rax]` |
| Size suffix | `movl`, `movw`, `movb` | `mov eax`, `mov ax`, `mov al` |

### 1.4 การ Generate Assembly พร้อม Debug Info

```bash
# Include source code as comments ใน assembly
gcc -S -g -fverbose-asm source.c -o output.s
```

ด้วย `-fverbose-asm` compiler จะใส่ comment ที่แสดง C source line ที่ correspond กับแต่ละ assembly instruction ทำให้อ่านง่ายขึ้นมาก

### 1.5 ตัวอย่าง C Code และ Assembly Output

```c
/* example_basic.c */
int add(int a, int b) {
    return a + b;
}

int main(void) {
    int x = 3;
    int y = 4;
    int z = add(x, y);
    return z;
}
```

AT&T Assembly output ด้วย `-O0`:
```asm
	.file	"example_basic.c"
	.text
	.globl	add
	.type	add, @function
add:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	%edi, -4(%rbp)
	movl	%esi, -8(%rbp)
	movl	-4(%rbp), %edx
	movl	-8(%rbp), %eax
	addl	%edx, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```

---

## 2. Optimization Levels

### 2.1 ภาพรวมของ Optimization Levels

GCC มี optimization levels หลายระดับ:

| Flag | ชื่อ | คำอธิบาย |
|------|------|----------|
| `-O0` | No optimization | ไม่ optimize เลย, ใช้สำหรับ debugging |
| `-O1` | Basic optimization | Optimizations ที่ไม่ใช้เวลา compile นาน |
| `-O2` | Standard optimization | ระดับที่แนะนำสำหรับ production |
| `-O3` | Aggressive optimization | Optimize มากสุด, อาจใช้เวลา compile นาน |
| `-Ofast` | Beyond standard | ละเมิด IEEE floating point rules |
| `-Os` | Size optimization | Optimize เพื่อขนาดที่เล็กที่สุด |
| `-Og` | Debug-friendly | Optimize สำหรับ debugging |

### 2.2 -O0: No Optimization

เมื่อใช้ `-O0` compiler จะ:
- ไม่ reorder instructions
- เก็บตัวแปรทุกตัวไว้บน stack (ไม่ใช้ registers)
- สร้าง full stack frame
- ไม่ inline functions
- ง่ายต่อการ debug เพราะ assembly ตรงกับ source code มาก

```c
/* test_opt.c */
long compute(long a, long b, long c) {
    long temp1 = a * 2;
    long temp2 = b + temp1;
    long result = temp2 * c;
    return result;
}
```

Assembly กับ `-O0`:
```asm
compute:
    pushq   %rbp
    movq    %rsp, %rbp
    movq    %rdi, -24(%rbp)    ; เก็บ a บน stack
    movq    %rsi, -32(%rbp)    ; เก็บ b บน stack
    movq    %rdx, -40(%rbp)    ; เก็บ c บน stack
    movq    -24(%rbp), %rax    ; load a
    addq    %rax, %rax         ; temp1 = a * 2
    movq    %rax, -8(%rbp)     ; เก็บ temp1 บน stack
    movq    -32(%rbp), %rax    ; load b
    addq    -8(%rbp), %rax     ; temp2 = b + temp1
    movq    %rax, -16(%rbp)    ; เก็บ temp2 บน stack
    movq    -16(%rbp), %rax    ; load temp2
    imulq   -40(%rbp), %rax    ; result = temp2 * c
    movq    %rax, -48(%rbp)    ; เก็บ result บน stack
    movq    -48(%rbp), %rax    ; load result สำหรับ return
    popq    %rbp
    ret
```

สังเกตว่ามี memory access ที่ไม่จำเป็นมากมาย เนื่องจาก compiler เก็บทุก intermediate value บน stack

### 2.3 -O1: Basic Optimization

กับ `-O1` compiler เริ่ม optimize ด้วย:
- **Register allocation**: ใช้ registers แทน stack variables
- **Dead code elimination**: ลบ code ที่ไม่ถูกใช้
- **Constant folding**: คำนวณ constant expressions ณ compile time
- **Common subexpression elimination (CSE)**: ลดการคำนวณซ้ำ

```asm
; Assembly กับ -O1
compute:
    imulq   %rsi, %rdi    ; temp1 = a * 2 -> rdi = rdi * 2 (ถูก fold)
    ; ผิด ตัวอย่าง O1 จริงๆ:
    addq    %rdi, %rdi    ; temp1 = a * 2
    addq    %rsi, %rdi    ; temp2 = b + temp1
    imulq   %rdx, %rdi    ; result = temp2 * c
    movq    %rdi, %rax    ; return value
    ret
```

เห็นว่า stack ไม่ถูกใช้เลย ทุก intermediate value อยู่ใน registers

### 2.4 -O2: Standard Optimization

`-O2` เพิ่ม optimizations ที่สำคัญ:
- **Function inlining**: inline functions ขนาดเล็ก
- **Loop optimizations**: unroll loops, strength reduction
- **Advanced CSE**: ขยาย scope ของ CSE
- **Dead store elimination**: ลบการเขียนที่ไม่มีคนอ่าน
- **Branch prediction**: เรียงใหม่ instructions เพื่อ branch prediction

```c
/* inline_test.c */
static inline int square(int x) {
    return x * x;
}

int sum_of_squares(int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        sum += square(i);
    }
    return sum;
}
```

Assembly กับ `-O2`:
```asm
sum_of_squares:
    testl   %edi, %edi
    jle     .L4
    leal    -1(%rdi), %eax      ; n - 1
    leal    -2(%rdi), %ecx      ; n - 2
    movl    %edi, %edx
    ; compiler inlines square() และ optimize loop
    xorl    %eax, %eax          ; sum = 0
    xorl    %ecx, %ecx          ; i = 0
.L3:
    movl    %ecx, %edx
    imull   %ecx, %edx          ; i * i (inlined square)
    addl    %edx, %eax          ; sum += i*i
    addl    $1, %ecx            ; i++
    cmpl    %edi, %ecx
    jl      .L3                 ; loop
.L4:
    ret
```

### 2.5 -O3: Aggressive Optimization

`-O3` เพิ่ม optimizations ที่ aggressive กว่า:
- **Auto-vectorization**: แปลง scalar loops เป็น SIMD instructions
- **Aggressive loop unrolling**: unroll loops มากขึ้น
- **Prefetching**: เพิ่ม prefetch hints
- **Profile-guided prediction**: ใช้ branch statistics

```c
/* vector_test.c */
void array_add(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}
```

Assembly กับ `-O3 -mavx2`:
```asm
array_add:
    testl   %ecx, %ecx
    jle     .L1
    ; Vectorized loop body
    movl    %ecx, %r8d
    shrl    $3, %r8d             ; n / 8 (AVX2 processes 8 floats)
    testl   %r8d, %r8d
    je      .L_scalar             ; fallback สำหรับ n < 8
    ; AVX2 vectorized main loop
.L_vec:
    vmovups (%rdi,%rax), %ymm0   ; load 8 floats จาก array a
    vmovups (%rsi,%rax), %ymm1   ; load 8 floats จาก array b
    vaddps  %ymm1, %ymm0, %ymm0  ; c = a + b (8 floats parallel)
    vmovups %ymm0, (%rdx,%rax)   ; store 8 floats ไปยัง c
    addq    $32, %rax             ; advance pointer by 32 bytes
    decl    %r8d
    jne     .L_vec
    ; Handle remaining elements
.L_scalar:
    ; ... scalar cleanup loop
.L1:
    vzeroupper
    ret
```

### 2.6 -Ofast: Beyond Standards

`-Ofast` ทำทุกอย่างที่ `-O3` ทำบวกกับ:
- ละเมิด IEEE 754 floating-point rules (reorder operations)
- ใช้ `-ffast-math`: อนุมาน `x*0 == 0` แม้ `x` อาจเป็น NaN
- `-fno-trapping-math`: ไม่สนใจ floating-point exceptions
- `-fno-signed-zeros`: อนุมานว่า `-0.0 == 0.0`

**คำเตือน**: อย่าใช้ `-Ofast` กับ code ที่ต้องการ numerical precision หรือ IEEE compliance

```bash
# เปรียบเทียบผลลัพธ์
gcc -O2 math_test.c -o math_O2
gcc -Ofast math_test.c -o math_Ofast

# ผลลัพธ์อาจแตกต่างกันสำหรับ edge cases!
```

---

## 3. ตัวอย่างการวิเคราะห์ Compiler Output อย่างละเอียด

### 3.1 Stack Frame ใน -O0

```c
/* stack_frame.c */
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```

Assembly กับ `-O0`:
```asm
factorial:
    ; Prologue - สร้าง stack frame
    pushq   %rbp                 ; บันทึก old base pointer
    movq    %rsp, %rbp           ; ตั้ง new base pointer
    subq    $16, %rsp            ; จอง space บน stack สำหรับ locals
    
    ; เก็บ parameter n บน stack
    movl    %edi, -4(%rbp)       ; n อยู่ที่ rbp-4
    
    ; if (n <= 1)
    cmpl    $1, -4(%rbp)
    jg      .L2                  ; ถ้า n > 1 ข้ามไป
    
    ; return 1
    movl    $1, %eax
    jmp     .L3
    
.L2:
    ; recursive call: factorial(n - 1)
    movl    -4(%rbp), %eax       ; load n
    subl    $1, %eax             ; n - 1
    movl    %eax, %edi           ; argument for recursive call
    call    factorial
    
    ; n * factorial(n-1)
    imull   -4(%rbp), %eax       ; eax = n * return_value
    
.L3:
    ; Epilogue - ทำลาย stack frame
    leave                        ; movq %rbp, %rsp; popq %rbp
    ret
```

### 3.2 Function Inlining ใน -O2

```c
/* inlining.c */
static inline int double_val(int x) {
    return x * 2;
}

int process(int a, int b) {
    return double_val(a) + double_val(b);
}
```

Assembly กับ `-O2`:
```asm
process:
    ; double_val(a) inlined: a * 2
    leal    (%rdi,%rdi), %eax    ; eax = a + a = a*2
    ; double_val(b) inlined: b * 2
    leal    (%rsi,%rsi), %edx    ; edx = b + b = b*2
    ; sum
    addl    %edx, %eax           ; eax = a*2 + b*2
    ret
    ; ไม่มี call instruction เลย!
```

### 3.3 Dead Code Elimination

```c
/* dead_code.c */
int compute(int x) {
    int unused = x * 100;       // นี่จะถูกลบ
    int result = x + 5;
    return result;
}
```

Assembly กับ `-O1`:
```asm
compute:
    leal    5(%rdi), %eax    ; eax = x + 5 (ตรงๆ ไม่มี unused)
    ret
    ; `unused` ถูก eliminate ไปทั้งหมด
```

### 3.4 Common Subexpression Elimination (CSE)

```c
/* cse.c */
int cse_example(int a, int b) {
    int x = a + b;       // computation 1
    int y = a + b + 5;   // computation 2 (a+b ซ้ำ)
    return x + y;
}
```

Assembly กับ `-O2`:
```asm
cse_example:
    leal    (%rdi,%rsi), %eax    ; eax = a + b  (คำนวณครั้งเดียว)
    leal    5(%rax), %edx        ; edx = (a+b) + 5  (ใช้ cache)
    addl    %edx, %eax           ; eax = x + y
    ret
    ; a+b คำนวณแค่ครั้งเดียว!
```

---

## 4. Auto-Vectorization

### 4.1 การเปิดใช้ Vectorization

```bash
# เปิด vectorization ด้วย -O2 (enabled by default)
gcc -O2 -ftree-vectorize source.c

# ดู vectorization report
gcc -O2 -ftree-vectorize -fopt-info-vec source.c
# หรือ output ไปยัง file
gcc -O2 -ftree-vectorize -fopt-info-vec=vec_report.txt source.c

# ดู missed vectorizations
gcc -O2 -fopt-info-vec-missed source.c
```

### 4.2 ตัวอย่าง Loop ที่ Vectorize ได้

```c
/* vectorizable.c */
void add_arrays(float *restrict a,
                float *restrict b,
                float *restrict c,
                int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}
```

รัน compile:
```bash
gcc -O2 -ftree-vectorize -fopt-info-vec -mavx2 vectorizable.c
# Output: vectorizable.c:4:5: optimized: loop vectorized using 32 byte vectors
```

Assembly ที่ได้กับ `-O3 -mavx2`:
```asm
add_arrays:
    ; Check n > 0
    testl   %ecx, %ecx
    jle     .L_end
    
    movl    %ecx, %eax
    shrl    $3, %eax            ; iterations / 8
    testl   %eax, %eax
    je      .L_cleanup
    
    xorl    %r9d, %r9d
    xorl    %r8d, %r8d
.L_vec_loop:
    ; โหลด 8 floats (256 bits) จาก a และ b
    vmovups (%rdi,%r8), %ymm0   ; ymm0 = a[i..i+7]
    vaddps  (%rsi,%r8), %ymm0, %ymm0  ; ymm0 += b[i..i+7]
    vmovups %ymm0, (%rdx,%r8)   ; c[i..i+7] = ymm0
    addq    $32, %r8             ; next 8 floats
    addl    $1, %r9d
    cmpl    %eax, %r9d
    jl      .L_vec_loop
    
.L_cleanup:
    ; Handle remaining elements (scalar)
    ; ...
.L_end:
    vzeroupper
    ret
```

### 4.3 ตัวอย่าง Loop ที่ Vectorize ไม่ได้

```c
/* non_vectorizable.c */

/* Case 1: Function call ใน loop */
void case1(float *a, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = sqrt(a[i]);    // sqrt() call อาจ block vectorization
    }
}

/* Case 2: Pointer aliasing */
void case2(float *a, float *b, int n) {
    /* ไม่มี restrict ทำให้ compiler ไม่รู้ว่า a กับ b overlap กันหรือเปล่า */
    for (int i = 0; i < n; i++) {
        a[i] = b[i] + 1.0f;
    }
}

/* Case 3: Loop-carried dependency */
void case3(float *a, int n) {
    for (int i = 1; i < n; i++) {
        a[i] = a[i-1] * 2.0f;  /* a[i] depends on a[i-1] */
    }
}

/* Case 4: Non-sequential access */
void case4(float *a, int *idx, float *b, int n) {
    for (int i = 0; i < n; i++) {
        b[i] = a[idx[i]];      /* gather operation ยาก vectorize */
    }
}
```

### 4.4 วิธีแก้ไขให้ Vectorize ได้

```c
/* fixed_vectorizable.c */

/* Fix Case 1: ใช้ sqrtf และ -ffast-math */
void case1_fixed(float *a, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = sqrtf(a[i]);   /* sqrtf จะถูก vectorize เป็น vsqrtps */
    }
}

/* Fix Case 2: เพิ่ม restrict */
void case2_fixed(float *restrict a, float *restrict b, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = b[i] + 1.0f;
    }
}

/* Fix Case 3: ไม่มีทางแก้ loop-carried dependency ได้ตรงๆ
   แต่อาจ restructure algorithm */
void case3_fixed(float *a, int n) {
    float prev = a[0];
    /* ต้องคิด algorithm ใหม่ */
}

/* Fix Case 4: ใช้ gather intrinsic manually */
#include <immintrin.h>
void case4_fixed(float *a, int *idx, float *b, int n) {
    int i;
    for (i = 0; i <= n - 8; i += 8) {
        __m256i vIdx = _mm256_loadu_si256((__m256i*)&idx[i]);
        __m256 vA = _mm256_i32gather_ps(a, vIdx, 4);
        _mm256_storeu_ps(&b[i], vA);
    }
    /* scalar cleanup */
    for (; i < n; i++) {
        b[i] = a[idx[i]];
    }
}
```

### 4.5 Vectorization Diagnostics อย่างละเอียด

```bash
# ดูทุกอย่างที่เกี่ยวกับ vectorization
gcc -O3 -fopt-info-vec-all -mavx2 source.c 2>&1 | head -50

# ดูเฉพาะ missed
gcc -O3 -fopt-info-vec-missed -mavx2 source.c 2>&1

# ตัวอย่าง output ที่อาจเห็น:
# source.c:10:5: optimized: loop vectorized using 32 byte vectors
# source.c:20:5: missed: couldn't vectorize loop
# source.c:20:5: missed: not vectorized: complicated access pattern
```

---

## 5. Target Architecture Flags

### 5.1 -march=native

```bash
# Compile สำหรับ CPU ปัจจุบัน
gcc -O3 -march=native source.c

# ดูว่า march=native เปิด flags อะไรบ้าง
gcc -march=native -Q --help=target | grep enabled
```

`-march=native` ทำให้ compiler รู้จัก CPU features ทั้งหมดของ machine ปัจจุบัน เช่น AVX2, FMA3, BMI2 เป็นต้น

**ข้อระวัง**: Binary ที่ compile ด้วย `-march=native` จะไม่ทำงานบน CPU รุ่นเก่ากว่า

### 5.2 CPU Extension Flags

```bash
# SSE/SSE2/SSE3/SSSE3/SSE4.1/SSE4.2
gcc -msse4.2 source.c

# AVX/AVX2
gcc -mavx2 source.c

# AVX-512 (ถ้า CPU รองรับ)
gcc -mavx512f -mavx512bw -mavx512cd source.c

# FMA (Fused Multiply-Add)
gcc -mfma source.c

# รวมกัน
gcc -O3 -mavx2 -mfma source.c

# ดู flags ทั้งหมด
gcc --target-help
```

### 5.3 ตัวอย่างผลกระทบของ -mavx2

```c
/* avx2_test.c */
void dot_product(float *a, float *b, float *result, int n) {
    float sum = 0.0f;
    for (int i = 0; i < n; i++) {
        sum += a[i] * b[i];
    }
    *result = sum;
}
```

กับ SSE2 (`-msse4.2`):
```asm
; ใช้ xmm registers (128 bits = 4 floats)
.L_loop_sse:
    movups  (%rdi,%rax), %xmm1
    movups  (%rsi,%rax), %xmm2
    mulps   %xmm2, %xmm1         ; 4 multiplications
    addps   %xmm1, %xmm0         ; 4 additions
    addq    $16, %rax
    ; ...
```

กับ AVX2 (`-mavx2 -mfma`):
```asm
; ใช้ ymm registers (256 bits = 8 floats)
.L_loop_avx:
    vmovups (%rdi,%rax), %ymm1
    vfmadd231ps (%rsi,%rax), %ymm1, %ymm0  ; 8 FMAs parallel!
    addq    $32, %rax
    ; ...
```

AVX2 + FMA ประมวลผล 8 floats พร้อมกัน โดย FMA รวม multiply และ add เป็น instruction เดียว

---

## 6. Compiler Hints และ Attributes

### 6.1 __builtin_expect

บอก compiler เกี่ยวกับ branch prediction:

```c
/* builtin_expect.c */
#include <stdio.h>

/* likely/unlikely macros */
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

int process_value(int x) {
    /* บอก compiler ว่า x > 0 เป็นกรณีปกติ */
    if (likely(x > 0)) {
        return x * 2;
    } else {
        /* error case */
        return -1;
    }
}

/* ใช้ใน error checking */
int divide(int a, int b) {
    if (unlikely(b == 0)) {  /* division by zero ไม่ค่อยเกิด */
        fprintf(stderr, "Division by zero!\n");
        return 0;
    }
    return a / b;
}
```

Assembly ที่เห็นความแตกต่าง:
```asm
; กับ likely(x > 0):
process_value:
    testl   %edi, %edi
    jle     .L_unlikely    ; jump to slow path ถ้า <=0
    ; fast path (falls through):
    leal    (%rdi,%rdi), %eax
    ret
.L_unlikely:
    movl    $-1, %eax
    ret
```

```asm
; กับ unlikely(x > 0):
process_value:
    testl   %edi, %edi
    jg      .L_unlikely    ; jump to fast path ถ้า >0
    ; slow path (falls through):
    movl    $-1, %eax
    ret
.L_unlikely:
    leal    (%rdi,%rdi), %eax
    ret
```

### 6.2 __builtin_unreachable

บอก compiler ว่า code path นี้ไม่มีทางถึงได้:

```c
/* unreachable.c */
#include <stdlib.h>

typedef enum { RED, GREEN, BLUE } Color;

int color_to_int(Color c) {
    switch (c) {
        case RED:   return 0;
        case GREEN: return 1;
        case BLUE:  return 2;
        default:
            /* บอก compiler ว่า default ไม่มีทางถึง */
            __builtin_unreachable();
    }
}
```

โดยไม่มี `__builtin_unreachable()`:
```asm
; compiler ต้องสร้าง default return value
color_to_int:
    cmpl    $2, %edi
    ja      .L_default
    ; ...
.L_default:
    ; undefined behavior handling
    xorl    %eax, %eax
    ret
```

ด้วย `__builtin_unreachable()`:
```asm
; compiler สามารถ optimize switch table
color_to_int:
    movl    %edi, %eax     ; เพียง load value
    ret
    ; ไม่มี bounds check เพราะ default unreachable
```

### 6.3 __builtin_prefetch

แนะนำ CPU ให้ prefetch data ล่วงหน้า:

```c
/* prefetch.c */
#include <stdlib.h>

#define PREFETCH_DISTANCE 32

void process_array(int *data, int n) {
    for (int i = 0; i < n; i++) {
        /* Prefetch data ที่จะใช้ในอีก 32 iterations */
        __builtin_prefetch(&data[i + PREFETCH_DISTANCE], 
                           0,    /* 0=read, 1=write */
                           3);   /* 0=no locality, 3=high locality */
        
        /* ประมวลผล current element */
        data[i] = data[i] * 2 + 1;
    }
}
```

Assembly ที่ได้:
```asm
process_array:
    testl   %esi, %esi
    jle     .L_end
    xorl    %eax, %eax
.L_loop:
    ; Prefetch hint
    prefetcht0  128(%rdi,%rax,4)    ; prefetch data+32 elements ahead
    
    ; Process current element
    movl    (%rdi,%rax,4), %ecx
    leal    1(%rcx,%rcx), %ecx      ; x*2+1
    movl    %ecx, (%rdi,%rax,4)
    
    incq    %rax
    cmpl    %esi, %eax
    jl      .L_loop
.L_end:
    ret
```

### 6.4 __attribute__((aligned(N)))

บังคับ alignment ของตัวแปรหรือ struct:

```c
/* aligned.c */
#include <immintrin.h>

/* Array ที่ align บน 32-byte boundary (สำหรับ AVX2) */
float __attribute__((aligned(32))) buffer[1024];

/* Struct ที่ align */
struct __attribute__((aligned(64))) CacheLine {
    int data[16];  /* 64 bytes = 1 cache line */
};

/* Function parameter alignment */
void vector_op(float *__attribute__((aligned(32))) a,
               float *__attribute__((aligned(32))) b,
               int n) {
    for (int i = 0; i < n; i += 8) {
        __m256 va = _mm256_load_ps(&a[i]);    /* aligned load */
        __m256 vb = _mm256_load_ps(&b[i]);    /* aligned load */
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_store_ps(&a[i], vc);           /* aligned store */
    }
}
```

Assembly ที่เห็นความแตกต่าง:
```asm
; Aligned operations (vmovaps) vs Unaligned (vmovups)
; aligned load (faster, may fault on misaligned access):
vmovaps     (%rdi), %ymm0

; unaligned load (slower but safe):
vmovups     (%rdi), %ymm0
```

### 6.5 restrict Keyword

บอก compiler ว่า pointers ไม่ overlap กัน:

```c
/* restrict_test.c */

/* ไม่มี restrict: compiler ต้อง assume aliasing */
void copy_slow(int *dst, int *src, int n) {
    for (int i = 0; i < n; i++) {
        dst[i] = src[i];
    }
}

/* มี restrict: compiler รู้ว่า pointers ไม่ overlap */
void copy_fast(int *restrict dst, int *restrict src, int n) {
    for (int i = 0; i < n; i++) {
        dst[i] = src[i];
    }
}
```

Assembly comparison:
```asm
; copy_slow: ต้อง check aliasing ทุก iteration
copy_slow:
    ; compiler อาจ reload src[i] ทุกครั้ง เพราะ dst อาจ alias กับ src
    testl   %edx, %edx
    jle     .L1
    ; scalar loop (ยาก vectorize)
    xorl    %eax, %eax
.L_loop:
    movl    (%rsi,%rax,4), %ecx
    movl    %ecx, (%rdi,%rax,4)
    incq    %rax
    cmpl    %edx, %eax
    jl      .L_loop
.L1:
    ret

; copy_fast: สามารถ vectorize ได้เต็มที่
copy_fast:
    testl   %edx, %edx
    jle     .L1
    ; vectorized loop
    movl    %edx, %eax
    shrl    $3, %eax          ; n/8
    ; ...
    ; vmovdqu, etc.
```

### 6.6 const Correctness

```c
/* const_test.c */

/* ไม่มี const: compiler ไม่รู้ว่า data จะเปลี่ยนไหม */
int sum_array(int *data, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        sum += data[i];
    }
    return sum;
}

/* มี const: compiler รู้ว่า data ไม่เปลี่ยน */
int sum_const(const int *data, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        sum += data[i];
    }
    return sum;
}
```

### 6.7 register Keyword (Modern Context)

```c
/* register_test.c */
/* register keyword เป็น hint เท่านั้น ใน modern C/C++ */
/* compiler สมัยใหม่มักเพิกเฉยต่อ hint นี้ */
int dot_product(int n, register int *a, register int *b) {
    register int sum = 0;
    for (register int i = 0; i < n; i++) {
        sum += a[i] * b[i];
    }
    return sum;
}
```

**Note**: ใน C++17, `register` keyword ถูก deprecated อย่างสมบูรณ์ compiler จะเพิกเฉย

### 6.8 pragma GCC optimize

```c
/* pragma_optimize.c */

/* เพิ่ม optimization สำหรับ function เดียว */
__attribute__((optimize("O3")))
void hot_function(float *a, float *b, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = b[i] * 2.0f;
    }
}

/* หรือใช้ pragma */
#pragma GCC optimize("O3,unroll-loops")
#pragma GCC target("avx2,fma")
void another_hot_function(float *a, float *b, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = b[i] * 3.14f + 2.71f;
    }
}
#pragma GCC reset_options

/* ปิด optimization สำหรับ debugging */
#pragma GCC optimize("O0")
void debug_function(int x) {
    /* code ที่ต้องการ debug */
    int y = x * 2;
    int z = y + 1;
    (void)z;
}
```

---

## 7. OpenMP SIMD

### 7.1 #pragma omp simd

OpenMP SIMD directives บอก compiler ให้ vectorize loop โดยตรง:

```c
/* openmp_simd.c */
#include <omp.h>

/* Basic SIMD loop */
void array_scale(float *a, float scale, int n) {
    #pragma omp simd
    for (int i = 0; i < n; i++) {
        a[i] *= scale;
    }
}

/* SIMD reduction */
float array_sum(float *a, int n) {
    float sum = 0.0f;
    #pragma omp simd reduction(+:sum)
    for (int i = 0; i < n; i++) {
        sum += a[i];
    }
    return sum;
}

/* SIMD with alignment hint */
void aligned_add(__attribute__((aligned(32))) float *a,
                 __attribute__((aligned(32))) float *b,
                 int n) {
    #pragma omp simd aligned(a, b:32)
    for (int i = 0; i < n; i++) {
        a[i] += b[i];
    }
}
```

Compile:
```bash
gcc -O2 -fopenmp-simd -mavx2 openmp_simd.c
# -fopenmp-simd เปิด SIMD directives โดยไม่ต้องใช้ OpenMP runtime
```

### 7.2 SIMD Directives ที่ซับซ้อน

```c
/* simd_advanced.c */
#include <math.h>

/* หลาย arrays พร้อมกัน */
void fused_op(float *__restrict__ a,
              float *__restrict__ b, 
              float *__restrict__ c,
              float *__restrict__ d,
              int n) {
    #pragma omp simd
    for (int i = 0; i < n; i++) {
        d[i] = a[i] * b[i] + c[i];  /* FMA pattern */
    }
}

/* SIMD loop ที่มี if */
void conditional_op(float *a, float *b, float threshold, int n) {
    #pragma omp simd
    for (int i = 0; i < n; i++) {
        if (a[i] > threshold) {
            b[i] = a[i] * 2.0f;
        } else {
            b[i] = 0.0f;
        }
        /* compiler จะใช้ masked operations */
    }
}
```

Assembly ที่ได้สำหรับ conditional_op:
```asm
; ใช้ vcmpps + vblendvps สำหรับ masking
.L_simd_conditional:
    vmovups     (%rdi,%rax), %ymm0      ; load a[i..i+7]
    vcmpps      $14, %ymm2, %ymm0, %ymm1  ; mask = a > threshold
    vaddps      %ymm0, %ymm0, %ymm3    ; a*2
    vandps      %ymm1, %ymm3, %ymm3    ; mask a*2
    vblendvps   %ymm1, %ymm0, %ymm4, %ymm0  ; blend
    ; ...
```

---

## 8. Godbolt Compiler Explorer

### 8.1 การใช้ Godbolt

**URL**: https://ce.godbolt.org หรือ https://godbolt.org

Godbolt Compiler Explorer เป็นเครื่องมือ web-based ที่ให้เราทดสอบ C/C++/Rust/Go code และดู assembly output แบบ real-time

**วิธีใช้**:
1. เข้าไปที่ https://godbolt.org
2. เลือก language (C, C++, etc.)
3. พิมพ์ code ใน left panel
4. เลือก compiler (GCC x86-64, Clang, MSVC, etc.)
5. ใส่ compiler flags (-O2, -mavx2, etc.)
6. Assembly output จะปรากฏใน right panel

### 8.2 Features ที่มีประโยชน์

```
Godbolt Features:
├── Color highlighting: แต่ละ C line มีสีเดียวกับ assembly ที่ correspond
├── Multiple compilers: เปรียบเทียบ GCC vs Clang vs MSVC
├── Multiple targets: x86-64, ARM, RISC-V, MIPS, etc.
├── Share links: แชร์ code + flags ได้
├── Diff view: เปรียบเทียบ output ของสอง flags
├── Execution: รัน code และดู output
├── Libraries: ใส่ library dependencies
└── Assembly filters: ซ่อน directives, comments, etc.
```

### 8.3 ตัวอย่าง URL สำหรับ Share

```
https://godbolt.org/z/xxxxx (shared link)

# ตัวอย่างการใช้ API
curl "https://godbolt.org/api/compiler/g122/compile" \
  -H "Content-Type: application/json" \
  -d '{"source":"int add(int a,int b){return a+b;}",
       "options":{"userArguments":"-O2"}}'
```

### 8.4 การเปรียบเทียบ Compiler Output

ใน Godbolt สามารถเปิดหลาย compiler พร้อมกัน:

```
Panel 1: GCC 12.2 x86-64 -O2
Panel 2: Clang 15.0 x86-64 -O2  
Panel 3: GCC 12.2 ARM64 -O2

สังเกตว่า:
- GCC และ Clang มักสร้าง assembly ที่คล้ายกันสำหรับ simple code
- สำหรับ complex optimizations อาจแตกต่างกันมาก
- ARM64 ใช้ instruction set ที่ต่างออกไปสิ้นเชิง
```

### 8.5 Godbolt API

```bash
# ใช้ Godbolt API จาก command line
curl -s "https://godbolt.org/api/compilers" | python3 -m json.tool | head -50

# Compile ด้วย API
curl -s -X POST "https://godbolt.org/api/compiler/g1220/compile" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "source": "int add(int a, int b) { return a + b; }",
    "options": {
      "userArguments": "-O2 -mavx2",
      "filters": {
        "intel": true,
        "demangle": true,
        "directives": false
      }
    }
  }' | python3 -m json.tool
```

---

## 9. LLVM MCA Analysis

### 9.1 llvm-mca คืออะไร

LLVM Machine Code Analyzer (llvm-mca) เป็นเครื่องมือที่วิเคราะห์ assembly code และทำนายประสิทธิภาพบน CPU architecture เฉพาะ มันบอก:
- Throughput ของ instruction sequences
- Port utilization
- Resource bottlenecks
- IPC (Instructions Per Cycle) ที่คาดหวัง

### 9.2 การใช้ llvm-mca

```bash
# ติดตั้ง
sudo apt install llvm-15

# Basic usage
llvm-mca -mcpu=skylake source.s

# หรือ pipe จาก clang
clang -O2 -S source.c -o - | llvm-mca -mcpu=skylake

# เปิด timeline view
llvm-mca -mcpu=skylake -timeline source.s

# เปิด bottleneck analysis
llvm-mca -mcpu=skylake -bottleneck-analysis source.s
```

### 9.3 ตัวอย่าง Assembly สำหรับ MCA Analysis

```asm
; dot_product_asm.s
; สำหรับ analysis ด้วย llvm-mca
    .intel_syntax noprefix
    
    .text
    .globl dot_product_kernel

# LLVM-MCA: Analyzing the inner loop
# LLVM-MCA: Begin
dot_product_kernel:
    vpxor       ymm0, ymm0, ymm0
.L_loop:
    vmovups     ymm1, YMMWORD PTR [rdi + rax]
    vfmadd231ps ymm0, ymm1, YMMWORD PTR [rsi + rax]
    add         rax, 32
    cmp         rax, rdx
    jl          .L_loop
    ; Horizontal sum
    vhaddps     ymm0, ymm0, ymm0
    vhaddps     ymm0, ymm0, ymm0
    vperm2f128  ymm1, ymm0, ymm0, 1
    vaddps      ymm0, ymm0, ymm1
    vmovss      DWORD PTR [rcx], xmm0
    ret
# LLVM-MCA: End
```

### 9.4 ตัวอย่าง MCA Output

```
Iterations:        100
Instructions:      800
Total Cycles:      207
Total uOps:        900

Dispatch Width:    6
uOps Per Cycle:    4.35
IPC:               3.86
Block RThroughput: 2.0

Instruction Info:
[1]: #uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 1      4     0.33    *                   vmovups ymm1, ...
 1      5     0.50                        vfmadd231ps ymm0, ymm1, ...
 1      1     0.25                        add rax, 32
 1      1     0.25                        cmp rax, rdx
 1      1     0.50                        jl .L_loop

Resources:
[0]   - SBPort0  
[1]   - SBPort1  
[2]   - SBPort23 
[3]   - SBPort4  
[4]   - SBPort5  

Resource pressure per iteration:
[0]    [1]    [2]    [3]    [4]
0.50   0.50   1.00   0.00   0.50
```

### 9.5 วิธีอ่าน MCA Output

```
IPC (Instructions Per Cycle):
- ค่าสูง = ดี (parallel execution)
- ค่าต่ำ = มี bottleneck

Block RThroughput (Reciprocal Throughput):
- Cycles ต่อ iteration ที่ดีที่สุดที่เป็นไปได้
- ถ้า actual cycles >> RThroughput = มี latency bottleneck

Port Pressure:
- Port ที่มี pressure สูงสุดคือ bottleneck
- ใน Skylake:
  Port 0, 1: Integer/FP ALU
  Port 2, 3: Load units
  Port 4:    Store unit
  Port 5:    Shuffle/Branch
  Port 6:    Integer ALU
  Port 7:    Store address
```

---

## 10. เมื่อไหรที่ควรเขียน Assembly ด้วยมือ

### 10.1 กรณีที่ Compiler ทำได้ดีอยู่แล้ว

ส่วนใหญ่แล้ว compiler สมัยใหม่ทำงานได้ดีกว่ามนุษย์ในกรณีเหล่านี้:

```
- General purpose code
- Simple loops ที่ vectorize ได้
- Function calls และ ABI management
- Register allocation สำหรับ complex functions
- Instruction scheduling สำหรับ pipelines
```

### 10.2 กรณีที่ควรเขียน Assembly

```c
/* Case 1: Hardware-specific instructions ที่ compiler ไม่ generate */

/* AES encryption round */
#include <immintrin.h>
__m128i aes_encrypt_block(__m128i data, __m128i key) {
    return _mm_aesenc_si128(data, key);  /* intrinsic */
}

/* หรือ inline assembly */
__m128i aes_encrypt_block_asm(__m128i data, __m128i key) {
    asm volatile (
        "aesenc %1, %0"
        : "+x"(data)
        : "x"(key)
    );
    return data;
}
```

```c
/* Case 2: Context switch / coroutines */
/* Compiler ไม่สามารถ save/restore arbitrary register state */

typedef struct {
    void *rsp;
    void *rbp;
    void *rbx;
    /* ... */
} Context;

/* Assembly function สำหรับ context switch */
extern void switch_context(Context *from, Context *to);
```

```asm
; switch_context.s
switch_context:
    ; Save callee-saved registers ของ 'from'
    mov     [rdi], rsp
    mov     [rdi+8], rbp
    mov     [rdi+16], rbx
    ; ... save other callee-saved registers
    
    ; Restore 'to' context
    mov     rsp, [rsi]
    mov     rbp, [rsi+8]
    mov     rbx, [rsi+16]
    ; ...
    ret
```

```c
/* Case 3: Critical path ที่ต้องการ exact instruction sequence */
/* เช่น CAS (Compare-And-Swap) */
int compare_and_swap(volatile int *addr, int expected, int new_val) {
    int result;
    asm volatile (
        "lock cmpxchg %2, %1"
        : "=a"(result), "+m"(*addr)
        : "r"(new_val), "0"(expected)
        : "memory"
    );
    return result == expected;
}
```

### 10.3 Decision Tree

```
ต้องการ performance optimization?
├── ใช่
│   ├── Profile ก่อน: gperf/perf/vtune
│   │   ├── Hotspot อยู่ที่ไหน?
│   │   ├── เป็น memory bound หรือ compute bound?
│   │   └── Compiler flags ช่วยได้ไหม?
│   │       ├── ได้: -O3 -march=native -mavx2
│   │       └── ไม่ได้: พิจารณา intrinsics ก่อน assembly
│   └── Intrinsics ก่อน assembly (portable กว่า)
└── ไม่ใช่
    └── ปล่อยให้ compiler จัดการ (ง่ายต่อ maintain)
```

---

## 11. ตัวอย่างจริง: Loop ที่ Vectorize vs ไม่ Vectorize

### 11.1 Loop ที่ Vectorize ได้สมบูรณ์

```c
/* fully_vectorizable.c */
#include <stddef.h>

/* Pattern 1: Simple element-wise operation */
void scale_array(float *restrict out, 
                 const float *restrict in,
                 float factor, 
                 size_t n) {
    for (size_t i = 0; i < n; i++) {
        out[i] = in[i] * factor;
    }
}

/* Pattern 2: Two-input element-wise */
void add_arrays(float *restrict c,
                const float *restrict a,
                const float *restrict b,
                size_t n) {
    for (size_t i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

/* Pattern 3: Reduction */
float sum_array(const float *restrict a, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; i++) {
        sum += a[i];
    }
    return sum;
}

/* Pattern 4: Fused operation */
void fma_array(float *restrict out,
               const float *restrict a,
               const float *restrict b,
               const float *restrict c,
               size_t n) {
    for (size_t i = 0; i < n; i++) {
        out[i] = a[i] * b[i] + c[i];  /* FMA */
    }
}
```

Compile และดู vectorization:
```bash
gcc -O3 -mavx2 -mfma -fopt-info-vec fully_vectorizable.c -c
# คาดว่าจะเห็น "optimized: loop vectorized" สำหรับทุก function
```

### 11.2 Loop ที่ Vectorize ไม่ได้และวิธีแก้ไข

```c
/* non_vec_and_fix.c */
#include <math.h>
#include <stdlib.h>

/* Problem 1: Dependency ระหว่าง iterations */
/* BAD: */
void running_sum_bad(float *a, int n) {
    for (int i = 1; i < n; i++) {
        a[i] += a[i-1];  /* a[i] depends on a[i-1] */
    }
}

/* Problem 2: Non-unit stride */
/* BAD: */
void transpose_bad(float *src, float *dst, int rows, int cols) {
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            dst[j * rows + i] = src[i * cols + j];  /* strided write */
        }
    }
}

/* Problem 3: Conditional with side effects */
/* BAD: */
int count_positives_bad(float *a, int n) {
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (a[i] > 0.0f) count++;  /* reduction - ยาก vectorize */
    }
    return count;
}

/* FIXED 3: ใช้ SIMD manually */
#include <immintrin.h>
int count_positives_fast(float *a, int n) {
    __m256 zero = _mm256_setzero_ps();
    __m256i count_vec = _mm256_setzero_si256();
    int i;
    
    for (i = 0; i <= n - 8; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 mask = _mm256_cmp_ps(va, zero, _CMP_GT_OQ);
        __m256i ones = _mm256_set1_epi32(1);
        __m256i masked = _mm256_and_si256(_mm256_castps_si256(mask), ones);
        count_vec = _mm256_add_epi32(count_vec, masked);
    }
    
    /* Horizontal sum of count_vec */
    int temp[8];
    _mm256_storeu_si256((__m256i*)temp, count_vec);
    int count = 0;
    for (int k = 0; k < 8; k++) count += temp[k];
    
    /* Scalar cleanup */
    for (; i < n; i++) {
        if (a[i] > 0.0f) count++;
    }
    return count;
}
```

### 11.3 Memory Access Pattern ที่ดีสำหรับ Vectorization

```c
/* memory_patterns.c */

/* GOOD: Sequential access (cache-friendly + vectorizable) */
float sum_rows(float matrix[][1024], int rows) {
    float total = 0.0f;
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < 1024; j++) {
            total += matrix[i][j];  /* sequential */
        }
    }
    return total;
}

/* BAD: Column-major access สำหรับ row-major storage */
float sum_cols(float matrix[][1024], int rows) {
    float total = 0.0f;
    for (int j = 0; j < 1024; j++) {
        for (int i = 0; i < rows; i++) {
            total += matrix[i][j];  /* strided, cache-unfriendly */
        }
    }
    return total;
}

/* IMPROVEMENT: ใช้ cache blocking */
float sum_blocked(float matrix[][1024], int rows) {
    const int BLOCK = 32;
    float total = 0.0f;
    for (int ii = 0; ii < rows; ii += BLOCK) {
        for (int jj = 0; jj < 1024; jj += BLOCK) {
            /* Process BLOCK x BLOCK tile */
            int iend = ii + BLOCK < rows ? ii + BLOCK : rows;
            for (int i = ii; i < iend; i++) {
                for (int j = jj; j < jj + BLOCK; j++) {
                    total += matrix[i][j];
                }
            }
        }
    }
    return total;
}
```

---

## 12. การ Benchmark และ Verify

### 12.1 สร้าง Benchmark ง่ายๆ

```c
/* benchmark.c */
#include <stdio.h>
#include <time.h>
#include <stdlib.h>

#define N (1 << 24)   /* 16M elements */
#define ITERATIONS 10

typedef void (*array_func)(float*, float*, int);

double benchmark(array_func fn, float *a, float *b, int n) {
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    for (int iter = 0; iter < ITERATIONS; iter++) {
        fn(a, b, n);
    }
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) * 1e-9;
    return elapsed / ITERATIONS;
}

/* Function versions */
void add_O0(float *a, float *b, int n) {
    for (int i = 0; i < n; i++) a[i] += b[i];
}

/* ใช้ attribute เพื่อ optimize แยก function */
__attribute__((optimize("O3,tree-vectorize")))
__attribute__((target("avx2,fma")))
void add_O3_avx2(float *a, float *b, int n) {
    for (int i = 0; i < n; i++) a[i] += b[i];
}

int main(void) {
    float *a = aligned_alloc(32, N * sizeof(float));
    float *b = aligned_alloc(32, N * sizeof(float));
    
    /* Initialize */
    for (int i = 0; i < N; i++) {
        a[i] = (float)i;
        b[i] = (float)(N - i);
    }
    
    double t0 = benchmark(add_O0, a, b, N);
    double t3 = benchmark(add_O3_avx2, a, b, N);
    
    printf("O0:       %.3f ms\n", t0 * 1000);
    printf("O3+AVX2:  %.3f ms\n", t3 * 1000);
    printf("Speedup:  %.1fx\n", t0 / t3);
    
    printf("Throughput O0:      %.2f GB/s\n",
           N * sizeof(float) * 2 / t0 / 1e9);
    printf("Throughput O3+AVX2: %.2f GB/s\n",
           N * sizeof(float) * 2 / t3 / 1e9);
    
    free(a);
    free(b);
    return 0;
}
```

Compile และรัน:
```bash
gcc -O0 benchmark.c -o bench_O0
gcc -O3 -mavx2 benchmark.c -o bench_O3
./bench_O3
```

### 12.2 perf stat สำหรับ Hardware Counter

```bash
# วัด cycles, instructions, cache misses
perf stat ./benchmark

# Output ตัวอย่าง:
# Performance counter stats for './benchmark':
#
#          1,234,567      cache-misses
#         56,789,012      cache-references
#        123,456,789      instructions
#         45,678,901      cycles
#
#        0.015 seconds time elapsed
```

---

## 13. เครื่องมืออื่นๆ ที่มีประโยชน์

### 13.1 objdump

```bash
# Disassemble object file
objdump -d -M intel output.o

# Disassemble executable
objdump -d -M intel ./program

# แสดง source lines ด้วย
objdump -d -M intel -l ./program

# แสดงเฉพาะ function
objdump -d -M intel ./program | awk '/^[0-9a-f]+ <add>:/,/^$/'
```

### 13.2 readelf

```bash
# ดู ELF headers
readelf -h ./program

# ดู sections
readelf -S ./program

# ดู symbols
readelf -s ./program
```

### 13.3 size

```bash
# ดูขนาด sections
size ./program_O0
size ./program_O3
# ดูความแตกต่างของ code size
```

### 13.4 nm

```bash
# List symbols
nm ./program

# ดูขนาดของแต่ละ function
nm -S --size-sort ./program | tail -20
```

---

## 14. Workshop: Putting It All Together

### 14.1 ตัวอย่างโปรเจกต์: Image Processing

```c
/* image_process.c */
#include <stdint.h>
#include <string.h>

/* Convert RGB to Grayscale */
/* Y = 0.299*R + 0.587*G + 0.114*B */

/* Version 1: Naive (O0 friendly) */
void rgb_to_gray_v1(
        const uint8_t *rgb,
        uint8_t *gray,
        int width, int height) {
    int n = width * height;
    for (int i = 0; i < n; i++) {
        int r = rgb[3*i + 0];
        int g = rgb[3*i + 1];
        int b = rgb[3*i + 2];
        gray[i] = (uint8_t)((r*299 + g*587 + b*114) / 1000);
    }
}

/* Version 2: Fixed-point arithmetic */
void rgb_to_gray_v2(
        const uint8_t *rgb,
        uint8_t *gray,
        int width, int height) {
    int n = width * height;
    /* Scale ด้วย 1/256 แทน 1/1000 เพื่อใช้ shift */
    /* R: 77/256, G: 150/256, B: 29/256 */
    for (int i = 0; i < n; i++) {
        unsigned r = rgb[3*i + 0];
        unsigned g = rgb[3*i + 1];
        unsigned b = rgb[3*i + 2];
        gray[i] = (uint8_t)((r*77 + g*150 + b*29) >> 8);
    }
}

/* Version 3: SIMD with AVX2 (process 8 pixels at once) */
#ifdef __AVX2__
#include <immintrin.h>

void rgb_to_gray_v3(
        const uint8_t *rgb,
        uint8_t *gray,
        int width, int height) {
    int n = width * height;
    
    /* Coefficients ใน 16-bit fixed point */
    const __m256i coeff_r = _mm256_set1_epi16(77);
    const __m256i coeff_g = _mm256_set1_epi16(150);
    const __m256i coeff_b = _mm256_set1_epi16(29);
    
    int i;
    for (i = 0; i <= n - 8; i += 8) {
        /* Load 8 pixels (24 bytes) */
        /* ต้องการ deinterleave R,G,B */
        /* simplified version - load and process */
        uint16_t r[8], g[8], b[8];
        for (int k = 0; k < 8; k++) {
            r[k] = rgb[3*(i+k) + 0];
            g[k] = rgb[3*(i+k) + 1];
            b[k] = rgb[3*(i+k) + 2];
        }
        
        __m256i vr = _mm256_loadu_si256((__m256i*)r);
        __m256i vg = _mm256_loadu_si256((__m256i*)g);
        __m256i vb = _mm256_loadu_si256((__m256i*)b);
        
        __m256i result = _mm256_add_epi16(
            _mm256_add_epi16(
                _mm256_mullo_epi16(vr, coeff_r),
                _mm256_mullo_epi16(vg, coeff_g)
            ),
            _mm256_mullo_epi16(vb, coeff_b)
        );
        
        /* Shift right by 8 */
        result = _mm256_srli_epi16(result, 8);
        
        /* Pack to 8-bit */
        /* ... */
    }
    
    /* Scalar cleanup */
    for (; i < n; i++) {
        gray[i] = (uint8_t)(
            (rgb[3*i]*77 + rgb[3*i+1]*150 + rgb[3*i+2]*29) >> 8
        );
    }
}
#endif
```

### 14.2 Script สำหรับ Automated Analysis

```bash
#!/bin/bash
# analyze_compiler_output.sh

SOURCE="$1"
BASE="${SOURCE%.c}"

echo "=== Compiler Output Analysis for $SOURCE ==="
echo ""

# Compile ด้วยทุก optimization levels
for opt in O0 O1 O2 O3; do
    echo "--- Compiling with -$opt ---"
    gcc -$opt -S -masm=intel -fverbose-asm "$SOURCE" -o "${BASE}_${opt}.s"
    
    # นับจำนวน instructions
    INST_COUNT=$(grep -E '^\s+[a-z]' "${BASE}_${opt}.s" | wc -l)
    echo "  Assembly instructions: $INST_COUNT"
    
    # Check for vectorization
    if grep -q 'ymm\|xmm' "${BASE}_${opt}.s" 2>/dev/null; then
        echo "  Vectorization: YES (SIMD registers found)"
    else
        echo "  Vectorization: NO"
    fi
    echo ""
done

echo "=== Vectorization Report for -O3 ==="
gcc -O3 -mavx2 -fopt-info-vec -fopt-info-vec-missed "$SOURCE" -c 2>&1

echo ""
echo "=== Code Size Comparison ==="
for opt in O0 O1 O2 O3; do
    gcc -$opt "$SOURCE" -o "${BASE}_${opt}_bin"
    SIZE=$(size "${BASE}_${opt}_bin" | tail -1 | awk '{print $1}')
    echo "  -$opt text size: $SIZE bytes"
done
```

---

## 15. Tips และ Best Practices

### 15.1 วิธีอ่าน Compiler Output อย่างมีประสิทธิภาพ

```
1. เริ่มจาก -O2 หรือ -O3 เสมอ
   - O0 output มีมากเกินไปและไม่ representative

2. ใช้ -fverbose-asm เพื่อดู C source mapping
   - ช่วยให้เข้าใจว่า assembly ส่วนไหน correspond กับ C ส่วนไหน

3. ซ่อน noise ใน Godbolt
   - Directives (.cfi_*, .loc, etc.)
   - Comments
   - Library functions

4. มองหา patterns สำคัญ:
   - SIMD registers (ymm, xmm): vectorization
   - call instructions: function calls ที่ไม่ถูก inline
   - stack operations: memory pressure
   - loop structure: unrolling, tiling
```

### 15.2 Red Flags ใน Compiler Output

```asm
; BAD: Memory ถูก access ซ้ำๆ ใน loop (ควรจะอยู่ใน register)
.L_loop:
    mov    eax, DWORD PTR [rbp-4]  ; load from stack ทุก iteration
    add    eax, ecx
    mov    DWORD PTR [rbp-4], eax  ; store to stack ทุก iteration

; GOOD: ใช้ register ตลอด
.L_loop_good:
    add    eax, ecx               ; ทำงานใน register ตลอด
```

```asm
; BAD: Division (ช้ามาก)
    idivq   %rcx              ; division: ~30-90 cycles

; BETTER: Multiplication by reciprocal (เร็วกว่า)
    imulq   %rcx, %rax        ; 3-5 cycles
    ; (compiler มักทำสิ่งนี้โดยอัตโนมัติถ้า divisor เป็น constant)
```

### 15.3 ตารางอ้างอิง Instruction Latency

```
Intel Skylake approximate latencies:
┌────────────────────────────────────────────────────────────────┐
│ Instruction          │ Latency │ Throughput │ Notes           │
├────────────────────────────────────────────────────────────────┤
│ add/sub (int)        │ 1       │ 0.25       │ Very fast       │
│ imul (32-bit)        │ 3       │ 1          │                 │
│ imul (64-bit)        │ 3       │ 1          │                 │
│ idiv (64-bit)        │ 35-90   │ 21-74      │ Avoid in loops  │
│ vaddps (AVX)         │ 4       │ 0.5        │ 8 floats        │
│ vmulps (AVX)         │ 4       │ 0.5        │ 8 floats        │
│ vfmadd (AVX+FMA)     │ 4       │ 0.5        │ Combined mul+add│
│ vsqrtps (AVX)        │ 12-13   │ 6          │                 │
│ vdivps (AVX)         │ 11-14   │ 5-14       │ Expensive       │
│ vmovups (load)       │ 5       │ 0.5        │ L1 cache hit    │
│ vmovups (store)      │ 4-5     │ 1          │ L1 cache hit    │
└────────────────────────────────────────────────────────────────┘
```

---

## 16. สรุปและข้อสรุป

### 16.1 Optimization Levels Summary

```
-O0: ไม่ optimize
  - ใช้สำหรับ debugging
  - Stack frame เต็มรูปแบบ
  - ไม่มี inlining
  - ทุก intermediate value บน stack

-O1: Optimize แบบ conservative
  - Register allocation
  - Dead code elimination
  - Basic CSE
  - เวลา compile เร็ว

-O2: Standard optimization (แนะนำสำหรับ production)
  - ทุกอย่างใน O1 +
  - Function inlining
  - Loop optimizations
  - Advanced scheduling
  - เหมาะสมที่สุดสำหรับส่วนใหญ่

-O3: Aggressive optimization
  - ทุกอย่างใน O2 +
  - Auto-vectorization
  - Loop unrolling เต็มที่
  - Prefetching
  - ใช้เวลา compile นานขึ้น

-Ofast: Beyond standards
  - ทุกอย่างใน O3 +
  - ละเมิด IEEE 754
  - ใช้ได้กับ code ที่ไม่ต้องการ precision สูง
```

### 16.2 สรุป Vectorization Tips

```
ทำให้ loop vectorize ได้:
✓ ใช้ restrict สำหรับ pointers
✓ ใช้ const สำหรับ read-only data
✓ ให้ loop มี known trip count
✓ Access memory แบบ sequential
✓ ไม่มี loop-carried dependency
✓ ใช้ aligned memory (aligned(32))
✓ ใช้ float แทน double ถ้าทำได้ (2x throughput)

ทำให้ loop vectorize ไม่ได้:
✗ Pointer aliasing (ไม่มี restrict)
✗ Indirect memory access (scatter/gather)
✗ Complex control flow ใน loop
✗ Function calls ที่ไม่ใช่ math builtins
✗ Loop-carried dependencies
```

### 16.3 เครื่องมือสรุป

```
gcc -S -masm=intel   : Generate Intel syntax assembly
gcc -fopt-info-vec   : Report vectorization success
gcc -fopt-info-vec-missed : Report missed vectorizations
gcc -march=native    : Target current CPU
Godbolt (godbolt.org): Interactive assembly viewer
llvm-mca             : Machine code analyzer
objdump -d           : Disassemble binaries
perf stat            : Hardware performance counters
```

### 16.4 ลำดับการ Optimize

```
1. Profile ก่อนเสมอ (อย่า guess)
2. ใช้ algorithm ที่ดีกว่า
3. ปรับ data structures (cache-friendly)
4. ใช้ compiler flags (-O3 -march=native)
5. ให้ hints แก่ compiler (restrict, const, aligned)
6. ใช้ intrinsics ถ้าต้องการ
7. เขียน assembly ด้วยมือ (เป็น last resort)
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1
เขียน C function `array_multiply(float *a, float *b, float *c, int n)` แล้ว compile ด้วย `-O0`, `-O2`, `-O3 -mavx2` ดูความแตกต่างของ assembly output

### Exercise 2
ลอง Godbolt: นำ code จาก Exercise 1 ไปทดสอบที่ https://godbolt.org เปรียบเทียบ GCC vs Clang

### Exercise 3
ใช้ `-fopt-info-vec-missed` เพื่อหาว่าทำไม loop หนึ่งไม่ vectorize แล้วแก้ไข

### Exercise 4
เพิ่ม `__builtin_expect` ใน function ที่มี common error checking path แล้วดูว่า assembly เปลี่ยนไปอย่างไร

### Exercise 5
ใช้ `llvm-mca` วิเคราะห์ inner loop ของ dot product function บน Skylake architecture และระบุ bottleneck

---

## อ้างอิง (References)

- GCC Documentation: https://gcc.gnu.org/onlinedocs/
- Intel Intrinsics Guide: https://www.intel.com/content/www/us/en/docs/intrinsics-guide/
- Agner Fog's Optimization Manuals: https://www.agner.org/optimize/
- Godbolt Compiler Explorer: https://godbolt.org
- LLVM MCA Documentation: https://llvm.org/docs/CommandGuide/llvm-mca.html
- Intel Architecture Optimization Reference Manual: https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
- "Computer Systems: A Programmer's Perspective" by Bryant & O'Hallaron

---

*Part 069 สิ้นสุด — ต่อไปใน Part 070: Profile-Guided Optimization (PGO)*
