# Part 064: Instruction Latency and Throughput
## ความหน่วงและปริมาณงานของคำสั่ง

---

## บทนำ (Introduction)

เมื่อเราเขียนโค้ด Assembly เราต้องเข้าใจว่า CPU ไม่ได้ทำงานทีละคำสั่งแบบเรียบง่าย  
แต่มี pipeline ที่ซับซ้อน ซึ่งทำให้คำสั่งต่างๆ ใช้เวลาไม่เท่ากัน  
และบางครั้งสามารถทำงานพร้อมกันหลายคำสั่งได้ในเวลาเดียวกัน

ความเข้าใจเรื่อง **Latency** และ **Throughput** เป็นกุญแจสำคัญในการเขียนโค้ด  
ที่มีประสิทธิภาพสูง (high-performance code)

---

## 1. Latency คืออะไร? (What is Latency?)

### 1.1 นิยาม (Definition)

**Latency** คือ จำนวน clock cycles ที่ต้องรอจากเวลาที่ input พร้อม  
จนกว่า output จะพร้อมใช้งาน

```
Latency = cycles from "input ready" to "output ready"
```

กล่าวอีกนัยหนึ่ง: ถ้าคำสั่ง A ต้องการผลลัพธ์จากคำสั่ง B  
คุณต้องรอกี่ cycles ก่อนที่คำสั่ง A จะเริ่มทำงานได้

### 1.2 ตัวอย่างง่ายๆ (Simple Example)

```nasm
; สมมติว่า ADD มี latency = 1 cycle
; คำสั่ง B ต้องรอผลจาก A ก็รอแค่ 1 cycle
add rax, rbx   ; Cycle 1: เริ่มต้น, Cycle 2: ผลพร้อม
add rcx, rax   ; รอ 1 cycle แล้วเริ่มได้เลย
```

```nasm
; สมมติว่า IMUL มี latency = 3 cycles
imul rax, rbx  ; Cycle 1: เริ่มต้น, Cycle 4: ผลพร้อม
add rcx, rax   ; ต้องรอ 3 cycles ก่อนจะทำ ADD ได้
```

### 1.3 Pipeline และ Latency

CPU สมัยใหม่ใช้ **pipeline** ซึ่งแบ่งการทำงานออกเป็นหลาย stage:

```
Fetch → Decode → Execute → Write-back
```

แต่ละ stage ใช้ 1 cycle ดังนั้นคำสั่งหนึ่งๆ อาจผ่านหลาย stage  
ก่อนที่ผลลัพธ์จะพร้อม

```
Cycle:  1    2    3    4    5
ADD:   [F]  [D]  [E]  [W]
               ^-- ผลพร้อมที่ cycle 4 (latency ≈ 1 หลัง execute)
```

---

## 2. Throughput (Reciprocal Throughput) คืออะไร?

### 2.1 นิยาม (Definition)

**Throughput** (หรือ Reciprocal Throughput) คือ ความถี่ที่เราสามารถ "issue"  
คำสั่งเดียวกันได้ต่อ cycle

```
Throughput = how often can we issue (per cycle)
Reciprocal Throughput (RTP) = 1 / Throughput

ถ้า RTP = 1:   issue ได้ทุก cycle
ถ้า RTP = 0.5: issue ได้ 2 ครั้งต่อ cycle (superscalar)
ถ้า RTP = 2:   issue ได้ทุก 2 cycles
```

### 2.2 ความแตกต่างระหว่าง Latency และ Throughput

```
Latency:    ส่งผลต่อโค้ดที่มี dependency (dependent chain)
Throughput: ส่งผลต่อโค้ดที่ independent (parallel execution)
```

ตัวอย่าง:
```nasm
; Independent operations - limited by throughput
add rax, 1    ; ไม่ขึ้นกัน
add rbx, 2    ; ไม่ขึ้นกัน
add rcx, 3    ; ไม่ขึ้นกัน
add rdx, 4    ; ไม่ขึ้นกัน
; CPU สามารถทำทั้ง 4 อย่างพร้อมกันได้ (ถ้า throughput รองรับ)

; Dependent operations - limited by latency
add rax, 1    ; ต้องทำก่อน
add rax, 2    ; รอ rax จาก บรรทัดแรก
add rax, 3    ; รอ rax จาก บรรทัดที่สอง
add rax, 4    ; รอ rax จาก บรรทัดที่สาม
; ต้องทำตามลำดับ ช้ากว่ามาก
```

---

## 3. Key Latencies บน Intel Skylake

### 3.1 Integer Operations

| Instruction | Latency | Throughput (RTP) | หมายเหตุ |
|-------------|---------|------------------|----------|
| MOV r,r     | 0 หรือ 1 | 0.25            | Register rename |
| MOV r,m     | 4-5     | 0.5             | Load latency |
| MOV m,r     | -       | 1               | Store |
| ADD/SUB     | 1       | 0.25            | Fast! |
| AND/OR/XOR  | 1       | 0.25            | Fast! |
| NOT/NEG     | 1       | 0.33            | |
| IMUL r16    | 3       | 1               | |
| IMUL r32    | 3       | 1               | |
| IMUL r64    | 3       | 1               | |
| DIV r32     | 26-90   | 21-74           | Slow! |
| DIV r64     | 35-90   | 21-74           | Very Slow! |
| SHL/SHR     | 1       | 0.5             | |
| SAR         | 1       | 0.5             | |
| LEA (simple)| 1       | 0.25            | |
| LEA (complex)| 3      | 1               | |

### 3.2 Float Point Operations (x87 / SSE scalar)

```
FP ADD (ADDSS/ADDSD):    Latency = 4,  Throughput = 0.5
FP MUL (MULSS/MULSD):    Latency = 4,  Throughput = 0.5
FP DIV single (DIVSS):   Latency = 11, Throughput = 7-14
FP DIV double (DIVSD):   Latency = 13-14, Throughput = 7-14
FP SQRT single (SQRTSS): Latency = 12, Throughput = 7-12
FP SQRT double (SQRTSD): Latency = 14-16, Throughput = 7-14
```

ตัวอย่าง NASM:
```nasm
section .text

float_example:
    ; Scalar float operations
    movss  xmm0, [a]      ; Load float a
    movss  xmm1, [b]      ; Load float b
    
    addss  xmm0, xmm1     ; xmm0 = a + b  (latency: 4)
    mulss  xmm0, xmm1     ; xmm0 *= b     (latency: 4, waits 4 cycles)
    
    ; Total latency: 4 + 4 = 8 cycles (dependent)
    ; แต่ถ้าทำแบบ independent จะเร็วกว่ามาก
    ret
```

### 3.3 SIMD (AVX/AVX2) Operations

| Instruction | Latency | Throughput | Width |
|-------------|---------|------------|-------|
| VADDPS      | 4       | 0.5        | 256-bit |
| VMULPS      | 4       | 0.5        | 256-bit |
| VFMADD231PS | 5       | 0.5        | 256-bit (FMA) |
| VDIVPS      | 11-18   | 5-14       | 256-bit |
| VSQRTPS     | 11-21   | 7-14       | 256-bit |
| VPAND       | 1       | 0.33       | 256-bit |
| VPOR        | 1       | 0.33       | 256-bit |
| VPXOR       | 1       | 0.33       | 256-bit |
| VPCMPEQD    | 1       | 0.5        | 256-bit |

```nasm
; SIMD example (GAS syntax)
.text
simd_add_vectors:
    # rdi = float* a, rsi = float* b, rdx = float* result, rcx = count
    xor     %eax, %eax
.loop:
    vmovaps (%rdi,%rax,4), %ymm0    # Load 8 floats from a
    vmovaps (%rsi,%rax,4), %ymm1    # Load 8 floats from b
    vaddps  %ymm1, %ymm0, %ymm2    # Add (latency: 4)
    vmovaps %ymm2, (%rdx,%rax,4)   # Store result
    add     $8, %eax
    cmp     %ecx, %eax
    jl      .loop
    ret
```

---

## 4. Latency-Bound vs Throughput-Bound Loops

### 4.1 Latency-Bound Loop

Loop ที่มี dependency chain ยาว จะถูกจำกัดด้วย latency:

```nasm
; GAS syntax - Latency-bound loop
; คำนวณผลรวม (sum) แบบมี dependency
.text
sum_latency_bound:
    # rdi = float* array, rsi = count
    vxorps  %ymm0, %ymm0, %ymm0   # accumulator = 0
    xor     %eax, %eax
.loop:
    vmovss  (%rdi,%rax,4), %xmm1   # Load float
    vaddss  %xmm1, %xmm0, %xmm0   # acc += array[i]  <- dependency!
    inc     %eax
    cmp     %esi, %eax
    jl      .loop
    # ผลลัพธ์อยู่ใน xmm0
    ret
```

ปัญหา: `vaddss` มี latency = 4 cycles  
แต่ละ iteration ต้องรอ 4 cycles ก่อนจะทำ iteration ถัดไปได้  
**Performance = N * 4 cycles** (latency-bound)

### 4.2 Throughput-Bound Loop

Loop ที่ไม่มี dependency ระหว่าง iteration จะถูกจำกัดด้วย throughput:

```nasm
; GAS syntax - Throughput-bound loop
; วนลูปที่ทำงาน independent ต่อกัน
.text
process_independent:
    # rdi = int* array, rsi = count
    xor     %eax, %eax
.loop:
    mov     (%rdi,%rax,4), %ecx    # Load
    imul    %ecx, %ecx             # Compute (independent each iter)
    mov     %ecx, (%rdi,%rax,4)    # Store
    inc     %eax
    cmp     %esi, %eax
    jl      .loop
    ret
```

ในกรณีนี้ แต่ละ iteration ไม่ขึ้นกัน CPU สามารถ overlap ได้  
**Performance ≈ max(latency(loop body), loop_overhead) / iterations_parallel**

### 4.3 การวิเคราะห์ว่า Loop ของเรา Bound อะไร?

```
ถ้า loop_body_latency > loop_body_throughput * num_iterations_in_flight
  -> Latency-bound

ถ้า loop_body_throughput * num_iterations_in_flight > loop_body_latency
  -> Throughput-bound (or memory-bound)
```

---

## 5. Dependency Chain Length (ความยาวของสาย Dependency)

### 5.1 การวิเคราะห์ Dependency Chain

```nasm
; NASM syntax
; Dependency chain analysis

section .text

; Example 1: Chain length = 4
dot_product_v1:
    ; rdi = float* a, rsi = float* b, rcx = count
    xorps   xmm0, xmm0     ; accumulator = 0.0
    xor     eax, eax
.loop:
    movss   xmm1, [rdi + rax*4]   ; a[i]
    mulss   xmm1, [rsi + rax*4]   ; a[i] * b[i]  (latency: 4)
    addss   xmm0, xmm1            ; acc += product (latency: 4)
    ; Chain: addss depends on mulss (4) then on previous addss (4)
    ; Total chain per iteration: 4 cycles (addss latency)
    inc     eax
    cmp     eax, ecx
    jl      .loop
    ret
```

Dependency chain ในตัวอย่างนี้:

```
Iteration 1:  mulss(4) -> addss(4) -> ...
Iteration 2:             รอ addss ของ iter 1 (4 cycles) -> addss(4) -> ...
Iteration 3:                          รอ addss ของ iter 2 -> addss(4) -> ...

Chain: [xmm0] -> add -> add -> add -> ...  (4 cycles ต่อ iteration)
```

### 5.2 Chain Analysis สำหรับ Dot Product

```
สมมติ N = 1000 elements
Latency-bound performance: 1000 * 4 cycles = 4000 cycles

Throughput (ADDSS RTP = 0.5):
  Theoretical best = N * max(MULSS_lat=4, ADDSS_lat=4) / parallelism
  But with dependency: we're stuck at 4 cycles/iteration
  
ดังนั้น dot product version 1 = 4000 cycles minimum
```

---

## 6. Breaking Dependency Chains

### 6.1 Loop Unrolling (การ Unroll Loop)

วิธีแรกคือ unroll loop เพื่อให้มีหลาย independent operations:

```nasm
; NASM syntax - Unrolled loop (2x)
dot_product_v2_2acc:
    ; rdi = float* a, rsi = float* b, rcx = count
    xorps   xmm0, xmm0     ; accumulator 1
    xorps   xmm2, xmm2     ; accumulator 2
    xor     eax, eax
    
    ; Handle pairs
.loop:
    movss   xmm1, [rdi + rax*4]       ; a[i]
    movss   xmm3, [rdi + rax*4 + 4]   ; a[i+1]
    mulss   xmm1, [rsi + rax*4]       ; a[i] * b[i]
    mulss   xmm3, [rsi + rax*4 + 4]   ; a[i+1] * b[i+1]
    addss   xmm0, xmm1                ; acc1 += ...
    addss   xmm2, xmm3                ; acc2 += ... (independent of acc1!)
    add     eax, 2
    cmp     eax, ecx
    jl      .loop
    
    ; Combine accumulators
    addss   xmm0, xmm2
    ret
```

ตอนนี้ xmm0 และ xmm2 เป็น independent chains:
```
Chain 1: xmm0 -> add -> add -> add -> ...  (4 cycles ต่อ 2 iterations)
Chain 2: xmm2 -> add -> add -> add -> ...  (4 cycles ต่อ 2 iterations)

ทั้งสองทำงานพร้อมกัน!
Performance: 4000 cycles / 2 = 2000 cycles  (2x speedup)
```

### 6.2 Multiple Accumulators (หลาย Accumulator)

```nasm
; NASM syntax - 4 accumulators
dot_product_v3_4acc:
    ; rdi = float* a, rsi = float* b, rcx = count
    xorps   xmm0, xmm0     ; acc0
    xorps   xmm2, xmm2     ; acc1
    xorps   xmm4, xmm4     ; acc2
    xorps   xmm6, xmm6     ; acc3
    xor     eax, eax

.loop:
    ; Load and multiply 4 pairs
    movss   xmm1, [rdi + rax*4]
    mulss   xmm1, [rsi + rax*4]
    addss   xmm0, xmm1             ; chain 0
    
    movss   xmm3, [rdi + rax*4 + 4]
    mulss   xmm3, [rsi + rax*4 + 4]
    addss   xmm2, xmm3             ; chain 1
    
    movss   xmm5, [rdi + rax*4 + 8]
    mulss   xmm5, [rsi + rax*4 + 8]
    addss   xmm4, xmm5             ; chain 2
    
    movss   xmm7, [rdi + rax*4 + 12]
    mulss   xmm7, [rsi + rax*4 + 12]
    addss   xmm6, xmm7             ; chain 3
    
    add     eax, 4
    cmp     eax, ecx
    jl      .loop
    
    ; Reduce
    addss   xmm0, xmm2
    addss   xmm1, xmm6   ; reuse xmm1 (free)
    addss   xmm0, xmm1
    ret
```

```
Performance analysis:
4 chains ทำงานพร้อมกัน
Performance: 4000 / 4 = 1000 cycles (4x speedup!)

จริงๆ แล้ว throughput ของ ADDSS = 0.5 cycles/op
ถ้า throughput-bound: 1000 ops * 0.5 = 500 cycles
เราเข้าใกล้ theoretical maximum แล้ว
```

### 6.3 8 Accumulators (SIMD)

ใช้ SIMD เพื่อ break chain และเพิ่ม width พร้อมกัน:

```nasm
; NASM syntax - 8 accumulators with SIMD (VADDPS handles 8 floats)
dot_product_v4_simd:
    ; rdi = float* a, rsi = float* b, rcx = count (must be multiple of 8)
    vxorps  ymm0, ymm0, ymm0   ; acc (8 floats)
    xor     eax, eax

.loop:
    vmovaps ymm1, [rdi + rax*4]    ; Load 8 floats
    vmulps  ymm1, ymm1, [rsi + rax*4]  ; Multiply 8 pairs
    vaddps  ymm0, ymm0, ymm1      ; Accumulate 8 results
    add     eax, 8
    cmp     eax, ecx
    jl      .loop
    
    ; Horizontal reduce ymm0
    vextractf128  xmm1, ymm0, 1   ; Get upper 128 bits
    vaddps        xmm0, xmm0, xmm1  ; Add lower + upper
    vhaddps       xmm0, xmm0, xmm0  ; Horizontal add
    vhaddps       xmm0, xmm0, xmm0  ; Again
    ; xmm0[0] = sum of all 8
    ret
```

```nasm
; ดีกว่านั้น: 8 accumulators + SIMD = 64 parallel chains!
dot_product_v5_best:
    ; 8 ymm accumulators, each handles 8 floats = 64 elements/iteration
    vxorps  ymm0, ymm0, ymm0   ; acc0: elements 0-7
    vxorps  ymm1, ymm1, ymm1   ; acc1: elements 8-15
    vxorps  ymm2, ymm2, ymm2   ; acc2: elements 16-23
    vxorps  ymm3, ymm3, ymm3   ; acc3: elements 24-31
    vxorps  ymm4, ymm4, ymm4   ; acc4
    vxorps  ymm5, ymm5, ymm5   ; acc5
    vxorps  ymm6, ymm6, ymm6   ; acc6
    vxorps  ymm7, ymm7, ymm7   ; acc7
    xor     eax, eax

.loop:
    vmovaps ymm8,  [rdi + rax*4]
    vmovaps ymm9,  [rdi + rax*4 + 32]
    vmovaps ymm10, [rdi + rax*4 + 64]
    vmovaps ymm11, [rdi + rax*4 + 96]
    vmovaps ymm12, [rdi + rax*4 + 128]
    vmovaps ymm13, [rdi + rax*4 + 160]
    vmovaps ymm14, [rdi + rax*4 + 192]
    vmovaps ymm15, [rdi + rax*4 + 224]
    
    vfmadd231ps  ymm0, ymm8,  [rsi + rax*4]
    vfmadd231ps  ymm1, ymm9,  [rsi + rax*4 + 32]
    vfmadd231ps  ymm2, ymm10, [rsi + rax*4 + 64]
    vfmadd231ps  ymm3, ymm11, [rsi + rax*4 + 96]
    vfmadd231ps  ymm4, ymm12, [rsi + rax*4 + 128]
    vfmadd231ps  ymm5, ymm13, [rsi + rax*4 + 160]
    vfmadd231ps  ymm6, ymm14, [rsi + rax*4 + 192]
    vfmadd231ps  ymm7, ymm15, [rsi + rax*4 + 224]
    
    add     eax, 64
    cmp     eax, ecx
    jl      .loop
    
    ; Reduce 8 accumulators
    vaddps  ymm0, ymm0, ymm4
    vaddps  ymm1, ymm1, ymm5
    vaddps  ymm2, ymm2, ymm6
    vaddps  ymm3, ymm3, ymm7
    vaddps  ymm0, ymm0, ymm2
    vaddps  ymm1, ymm1, ymm3
    vaddps  ymm0, ymm0, ymm1
    ; Horizontal reduce ymm0...
    ret
```

---

## 7. FP Sum: 1 Accumulator vs 4 Accumulators vs 8 Accumulators

### 7.1 Comparison Analysis

```
Array sum ของ N = 1,000,000 elements (float):

Version 1: 1 accumulator (scalar)
  - Latency bound: 1M * 4 cycles = 4M cycles
  - ≈ 4 ms ที่ 1 GHz

Version 2: 4 accumulators (scalar)
  - Latency bound: 1M * 4 / 4 = 1M cycles
  - ≈ 1 ms ที่ 1 GHz

Version 3: 1 SIMD accumulator (8 elements)
  - 1M/8 = 125K iterations * 4 cycles = 500K cycles
  - ≈ 0.5 ms ที่ 1 GHz

Version 4: 4 SIMD accumulators (32 elements/iter)
  - 1M/32 = 31.25K iterations * 4 cycles = 125K cycles
  - ≈ 0.125 ms ที่ 1 GHz

Version 5: 8 SIMD accumulators (64 elements/iter)
  - 1M/64 = 15.625K iterations * 4 cycles = 62.5K cycles
  - ≈ 0.0625 ms ที่ 1 GHz
  - Speedup: 64x จาก version 1!
```

### 7.2 Code: 1 Accumulator

```nasm
; NASM - Float sum with 1 accumulator
float_sum_1acc:
    ; rdi = float* array, rsi = count
    vxorps  xmm0, xmm0, xmm0
    xor     eax, eax
.loop:
    vaddss  xmm0, xmm0, [rdi + rax*4]  ; ← dependency chain!
    inc     eax
    cmp     eax, esi
    jl      .loop
    ret
    ; Performance: N * 4 cycles
```

### 7.3 Code: 4 Accumulators

```nasm
; NASM - Float sum with 4 accumulators
float_sum_4acc:
    ; rdi = float* array, rsi = count
    vxorps  xmm0, xmm0, xmm0   ; acc0
    vxorps  xmm1, xmm1, xmm1   ; acc1
    vxorps  xmm2, xmm2, xmm2   ; acc2
    vxorps  xmm3, xmm3, xmm3   ; acc3
    xor     eax, eax
    
.loop:
    vaddss  xmm0, xmm0, [rdi + rax*4]       ; chain 0
    vaddss  xmm1, xmm1, [rdi + rax*4 + 4]   ; chain 1
    vaddss  xmm2, xmm2, [rdi + rax*4 + 8]   ; chain 2
    vaddss  xmm3, xmm3, [rdi + rax*4 + 12]  ; chain 3
    add     eax, 4
    cmp     eax, esi
    jl      .loop
    
    vaddss  xmm0, xmm0, xmm1
    vaddss  xmm2, xmm2, xmm3
    vaddss  xmm0, xmm0, xmm2
    ret
    ; Performance: (N/4) * 4 = N cycles = 4x speedup
```

### 7.4 Code: 8 SIMD Accumulators

```nasm
; NASM - Float sum with 8 SIMD accumulators (64 floats/iteration)
float_sum_8simd:
    ; rdi = float* array, rsi = count (multiple of 64)
    vxorps  ymm0, ymm0, ymm0
    vxorps  ymm1, ymm1, ymm1
    vxorps  ymm2, ymm2, ymm2
    vxorps  ymm3, ymm3, ymm3
    vxorps  ymm4, ymm4, ymm4
    vxorps  ymm5, ymm5, ymm5
    vxorps  ymm6, ymm6, ymm6
    vxorps  ymm7, ymm7, ymm7
    xor     eax, eax
    
.loop:
    vaddps  ymm0, ymm0, [rdi + rax*4]
    vaddps  ymm1, ymm1, [rdi + rax*4 + 32]
    vaddps  ymm2, ymm2, [rdi + rax*4 + 64]
    vaddps  ymm3, ymm3, [rdi + rax*4 + 96]
    vaddps  ymm4, ymm4, [rdi + rax*4 + 128]
    vaddps  ymm5, ymm5, [rdi + rax*4 + 160]
    vaddps  ymm6, ymm6, [rdi + rax*4 + 192]
    vaddps  ymm7, ymm7, [rdi + rax*4 + 224]
    add     eax, 64
    cmp     eax, esi
    jl      .loop
    
    ; Tree reduction of 8 YMM registers
    vaddps  ymm0, ymm0, ymm4
    vaddps  ymm1, ymm1, ymm5
    vaddps  ymm2, ymm2, ymm6
    vaddps  ymm3, ymm3, ymm7
    vaddps  ymm0, ymm0, ymm2
    vaddps  ymm1, ymm1, ymm3
    vaddps  ymm0, ymm0, ymm1
    
    ; Horizontal reduce
    vextractf128  xmm1, ymm0, 1
    vaddps        xmm0, xmm0, xmm1
    vhaddps       xmm0, xmm0, xmm0
    vhaddps       xmm0, xmm0, xmm0
    ret
    ; Performance: (N/64) * 4 = N/16 cycles = 64x speedup!
```

---

## 8. Strength Reduction (การลดความซับซ้อน)

### 8.1 คืออะไร?

Strength Reduction คือ การแทนที่ operation ที่ช้ากว่าด้วย operation ที่เร็วกว่า  
โดยให้ผลลัพธ์เหมือนเดิม

### 8.2 Multiplication to Shift: a*8 → a<<3

```nasm
; NASM syntax

; Slow version: IMUL (latency = 3)
imul rax, 8         ; rax = rax * 8  (3 cycles latency)

; Fast version: SHL (latency = 1)
shl  rax, 3         ; rax = rax << 3 = rax * 8  (1 cycle latency)

; อีกตัวอย่าง: a * 16
imul rax, 16        ; slow
shl  rax, 4         ; fast: shift by 4 = multiply by 16

; a * 32
imul rax, 32        ; slow
shl  rax, 5         ; fast
```

```nasm
; Power of 2 table:
; x * 2   = x << 1
; x * 4   = x << 2
; x * 8   = x << 3
; x * 16  = x << 4
; x * 32  = x << 5
; x * 64  = x << 6
; x * 128 = x << 7
; x * 256 = x << 8
```

### 8.3 LEA สำหรับ Multiply ด้วยค่าบางอย่าง

```nasm
; NASM syntax

; x * 3
lea  rax, [rax + rax*2]    ; rax = rax + rax*2 = rax*3 (1 cycle!)

; x * 5
lea  rax, [rax + rax*4]    ; rax = rax + rax*4 = rax*5

; x * 9
lea  rax, [rax + rax*8]    ; rax = rax + rax*8 = rax*9

; x * 6
lea  rax, [rax*2 + rax*4]  ; ไม่ได้นะ - mode addressing มีข้อจำกัด

; ถ้าต้องการ x * 6:
lea  rax, [rax + rax*2]    ; rax = rax*3
shl  rax, 1                ; rax = rax*6

; x * 12
lea  rax, [rax + rax*2]    ; rax = rax*3
shl  rax, 2                ; rax = rax*12
```

### 8.4 GAS syntax version

```gas
# GAS syntax strength reduction
.text
strength_reduce_example:
    # rdi = input value
    mov     %rdi, %rax
    
    # Multiply by 8: use shift
    shl     $3, %rax           # rax = rdi * 8
    
    # Multiply by 3: use LEA
    lea     (%rdi,%rdi,2), %rbx  # rbx = rdi + rdi*2 = rdi*3
    
    # Array access: ptr[i*sizeof(int)]
    # sizeof(int) = 4 = 2^2
    # arr[rdi*4]:
    lea     (%rsi,%rdi,4), %rcx  # rcx = rsi + rdi*4
    
    ret
```

---

## 9. Division to Multiplication: Integer Division Optimization

### 9.1 ปัญหาของ DIV

DIV instruction เป็น instruction ที่ช้าที่สุดอันหนึ่ง:
```
DIV r32:  latency 26-90 cycles
DIV r64:  latency 35-90 cycles
```

เทียบกับ IMUL ที่มี latency 3 cycles เท่านั้น!

### 9.2 Division by Constant: x/d → x * (2^n/d) >> n

ถ้า d เป็นค่าคงที่ เราสามารถแปลงการหารเป็นการคูณได้:

```
x / d  ≈  x * (2^n / d)  >> n

โดยเลือก n ให้ใหญ่พอที่จะได้ precision ที่ต้องการ
ปกติ n = 32 หรือ 64
```

### 9.3 Magic Number Computation

สำหรับ division ด้วยค่าคงที่ d:

```
Algorithm (สำหรับ unsigned division):
1. เลือก n โดยทั่วไป n = 32 + bit_count(d-1)
2. M = ceil(2^n / d)  <- "magic number"
3. x / d  ≈  (x * M) >> n   (unsigned shift right)
```

ตัวอย่าง: x / 7

```python
# Python เพื่อคำนวณ magic number
import math

d = 7
n = 32 + math.ceil(math.log2(d))  # = 32 + 3 = 35

# ไม่ถูกต้องทุกกรณี ต้องใช้ algorithm ที่สมบูรณ์กว่านี้
# Compiler ทำสิ่งนี้โดยอัตโนมัติ

# สำหรับ d=7, n=32:
# M = ceil(2^32 / 7) = ceil(613566756.57...) = 613566757
# แต่ต้องตรวจสอบ exactness
```

### 9.4 ตัวอย่าง: Divide by 7

```nasm
; NASM syntax - Division by 7 using magic number
; x / 7 where x is 32-bit unsigned

divide_by_7:
    ; rdi = dividend (32-bit unsigned value in edi)
    
    ; Magic number สำหรับ /7 (unsigned 32-bit):
    ; M = 0x92492493  (จาก compiler หรือ Hacker's Delight)
    ; algorithm: result = (M * x) >> 32, then adjust
    
    mov     eax, edi          ; eax = x
    mov     ecx, 0x92492493   ; magic number
    mul     ecx               ; edx:eax = x * M  (unsigned multiply)
    
    ; Special adjustment for d=7:
    sub     edi, edx          ; t = x - high_word
    shr     edi, 1            ; t = t >> 1
    add     edx, edi          ; result = high_word + t
    shr     edx, 2            ; result >>= 2
    
    ; edx = x / 7
    mov     eax, edx
    ret
```

```nasm
; ตัวอย่าง: Divide by 10 (unsigned 32-bit)
divide_by_10:
    ; rdi = dividend
    mov     eax, edi
    mov     edx, 0xCCCCCCCD   ; magic number for /10
    mul     edx               ; edx:eax = x * M
    shr     edx, 3            ; edx >>= 3
    ; edx = x / 10
    mov     eax, edx
    ret
```

```nasm
; ตัวอย่าง: Divide by 3 (unsigned 32-bit)
divide_by_3:
    ; rdi = dividend
    mov     eax, edi
    mov     edx, 0xAAAAAAAB   ; magic number for /3
    mul     edx               ; edx:eax = x * M
    shr     edx, 1            ; edx >>= 1
    ; edx = x / 3
    mov     eax, edx
    ret
```

### 9.5 Signed Division by Constant

```nasm
; NASM syntax - Signed division by constant
; x / 3 (signed 32-bit)

signed_divide_by_3:
    ; rdi = dividend (signed)
    mov     eax, edi
    imul    rax, rax, 0x55555556  ; magic number for signed /3
    sar     rax, 32               ; arithmetic shift
    mov     ecx, edi
    sar     ecx, 31               ; sign bit replicate
    sub     eax, ecx              ; adjust for negative
    ; eax = x / 3 (signed)
    ret
```

### 9.6 Compiler Output เปรียบเทียบ

```c
// C code
int div_by_7(int x) { return x / 7; }
```

```nasm
; Output ที่ compiler สร้าง (GCC/Clang -O2):
; div_by_7:
;   movsx  eax, edi
;   imul   rax, rax, 0x6DB6DB6E  ; magic number
;   sar    rax, 33
;   mov    ecx, edi
;   sar    ecx, 31
;   sub    eax, ecx
;   ret
;
; Note: ไม่มี IDIV เลย! เพราะ compiler รู้ว่า IMUL + SAR เร็วกว่ามาก
```

---

## 10. Reciprocal Approximation: RCPPS

### 10.1 RCPPS คืออะไร?

RCPPS = Reciprocal Packed Single-Precision Floating-Point  
คำนวณ approximation ของ 1/x โดยมี precision ประมาณ 11-12 bits

```
RCPPS xmm_dst, xmm_src
xmm_dst[i] ≈ 1 / xmm_src[i]    (สำหรับ i = 0,1,2,3)
```

### 10.2 Latency เปรียบเทียบ

```
DIVPS (exact):  latency 11-18 cycles,  throughput 8-14
RCPPS (approx): latency 4 cycles,      throughput 1

RCPPS เร็วกว่า 3-4x แต่ precision ต่ำกว่า
```

### 10.3 Newton-Raphson Refinement

เพิ่ม precision ด้วย Newton-Raphson iteration:

```
y0 = RCPPS(x)              ; initial approximation, ~12 bits
y1 = y0 * (2 - x * y0)    ; one N-R step, ~24 bits (float precision)
y2 = y1 * (2 - x * y1)    ; two N-R steps, ~48 bits (double precision)
```

```nasm
; NASM syntax - Fast reciprocal with N-R refinement
fast_reciprocal:
    ; xmm0 = x (4 packed floats)
    
    rcpps   xmm1, xmm0          ; xmm1 = ~1/x (12-bit precision)
    
    ; One Newton-Raphson step:
    ; y = y * (2 - x*y)
    movaps  xmm2, xmm0          ; xmm2 = x
    mulps   xmm2, xmm1          ; xmm2 = x * y0
    movaps  xmm3, [const_2]     ; xmm3 = 2.0
    subps   xmm3, xmm2          ; xmm3 = 2 - x*y0
    mulps   xmm1, xmm3          ; xmm1 = y0 * (2 - x*y0) = y1
    
    ; xmm1 ≈ 1/x with ~24-bit precision (สำหรับ float ก็พอแล้ว)
    ret
```

```gas
# GAS syntax - RCPPS with Newton-Raphson
.text
.align 16
.globl fast_reciprocal_4floats
fast_reciprocal_4floats:
    # xmm0 = 4 floats to invert
    rcpps       %xmm0, %xmm1       # xmm1 = approx 1/x
    
    # Newton-Raphson refinement
    mulps       %xmm1, %xmm0       # xmm0 = x * y0
    movaps      .two(%rip), %xmm2  # xmm2 = [2.0, 2.0, 2.0, 2.0]
    subps       %xmm0, %xmm2       # xmm2 = 2.0 - x*y0
    mulps       %xmm2, %xmm1       # xmm1 = y0 * (2.0 - x*y0)
    
    movaps      %xmm1, %xmm0       # return in xmm0
    ret

.section .rodata
.align 16
.two:
    .float 2.0, 2.0, 2.0, 2.0
```

### 10.4 RSQRTPS (Reciprocal Square Root)

คล้ายกัน แต่สำหรับ 1/sqrt(x):

```nasm
; NASM syntax
fast_inv_sqrt:
    ; xmm0 = x
    rsqrtps xmm1, xmm0          ; xmm1 = ~1/sqrt(x) (12-bit precision)
    
    ; Newton-Raphson สำหรับ rsqrt:
    ; y1 = y0 * (1.5 - 0.5 * x * y0^2)
    movaps  xmm2, [const_0_5]
    mulps   xmm2, xmm0          ; xmm2 = 0.5 * x
    movaps  xmm3, xmm1
    mulps   xmm3, xmm1          ; xmm3 = y0^2
    mulps   xmm2, xmm3          ; xmm2 = 0.5 * x * y0^2
    movaps  xmm4, [const_1_5]
    subps   xmm4, xmm2          ; xmm4 = 1.5 - 0.5*x*y0^2
    mulps   xmm1, xmm4          ; xmm1 = y0 * (1.5 - 0.5*x*y0^2) = y1
    
    ; xmm1 ≈ 1/sqrt(x) with float precision
    ret
```

### 10.5 VRCPPS (AVX version)

```nasm
; NASM syntax - AVX version (8 floats at once)
fast_reciprocal_avx:
    ; ymm0 = 8 floats
    vrcpps   ymm1, ymm0          ; ymm1 = ~1/x (8 values at once!)
    
    ; Newton-Raphson:
    vmulps   ymm2, ymm0, ymm1   ; ymm2 = x * y0
    vmovaps  ymm3, [const_2_avx]; ymm3 = [2.0 x8]
    vsubps   ymm3, ymm3, ymm2   ; ymm3 = 2 - x*y0
    vmulps   ymm1, ymm1, ymm3   ; ymm1 = y0 * (2 - x*y0) = y1
    ret
```

---

## 11. Integer Throughput: x86 สามารถทำ 4 ALU ops/cycle

### 11.1 Superscalar Execution

CPU สมัยใหม่มีหลาย Execution Unit (EU):

```
Intel Skylake:
Port 0: ALU (add, and, or, xor...), MUL, FP
Port 1: ALU, IMUL, FP
Port 2: Load + Address Gen
Port 3: Load + Address Gen
Port 4: Store Data
Port 5: ALU (branches, shifts), SIMD shuffle
Port 6: ALU, branch
Port 7: Store Address

Integer ALU ports: 0, 1, 5, 6 = 4 ALU operations per cycle!
```

### 11.2 Throughput ของ Integer Operations

```
ADD/SUB/AND/OR/XOR:
  Can use port 0, 1, 5, 6
  Throughput = 4 per cycle (RTP = 0.25)

IMUL r32/r64:
  Can use port 1 only
  Throughput = 1 per cycle (RTP = 1)

LEA (simple - one base register):
  Can use port 0, 1, 5, 6
  Throughput = 4 per cycle

LEA (complex - base + index*scale + offset):
  Can use port 1 only
  Throughput = 1 per cycle
```

### 11.3 เพิ่ม Performance ด้วย Instruction Mix

```nasm
; GAS syntax - Good instruction mix (using multiple ports)
.text
good_mix:
    # วนลูปที่ใช้หลาย port
    # Port 0,1,5,6: ADD
    # Port 2,3: LOAD
    # Port 4,7: STORE
.loop:
    add     %rax, %rbx          # Port 0/1/5/6
    mov     (%rdi), %rcx        # Port 2/3
    add     %rdx, %rsi          # Port 0/1/5/6 (different from above)
    mov     (%rdi,%rax,8), %r8  # Port 2/3
    mov     %rcx, (%rsi)        # Port 4/7
    dec     %r9
    jnz     .loop
    ret

; ถ้า instruction ต่างๆ ใช้ port ต่างกัน
; สามารถทำทั้งหมดใน 1 cycle ได้!
```

### 11.4 Throughput Analysis Example

```nasm
; NASM syntax
throughput_analysis:
    ; ลองวิเคราะห์ว่าแต่ละ iteration ต้องใช้กี่ cycles

.loop:
    add  rax, rbx    ; Port 0/1/5/6 - 1 op
    add  rcx, rdx    ; Port 0/1/5/6 - 1 op
    add  r8,  r9     ; Port 0/1/5/6 - 1 op
    add  r10, r11    ; Port 0/1/5/6 - 1 op
    ; ทั้ง 4 ADD สามารถออกพร้อมกันได้ใน 1 cycle!
    ; (assuming no dependencies)
    
    imul rax, rbx    ; Port 1 only - 1 op
    imul rcx, rdx    ; Port 1 only - 1 op
    ; IMUL ทั้ง 2 ต้องรอกัน ต้องใช้ 2 cycles
    
    ; Result: 4 ADD + 2 IMUL
    ; MIN cycles = max(4/4, 2/1) = max(1, 2) = 2 cycles
    dec  r12
    jnz  .loop
    ret
```

---

## 12. Instruction Selection: ตัวเลือกที่ดีกว่า

### 12.1 XOR vs SUB vs MOV สำหรับ Zero a Register

```nasm
; NASM syntax - Different ways to zero a register

; Method 1: MOV
mov  rax, 0        ; 7 bytes, latency 1, but doesn't break dependency

; Method 2: XOR (preferred!)
xor  eax, eax      ; 2 bytes, latency 0 (dependency break), recognized by CPU
                   ; xor eax,eax → zeros rax implicitly (zero-extension)
                   ; CPU can eliminate this in rename stage!

; Method 3: SUB
sub  eax, eax      ; 2 bytes, similar to XOR
                   ; แต่บางทีไม่ recognized เหมือน XOR

; Method 4: AND ด้วย 0 (ไม่แนะนำ)
and  rax, 0        ; ช้ากว่า ใหญ่กว่า

; Best practice:
xor  eax, eax      ; ← ใช้อันนี้เสมอสำหรับ zero register!
```

### 12.2 ทำไม XOR ถึงเร็วกว่า?

```
CPU pipeline: Fetch → Decode → Rename → Execute → Writeback

XOR eax, eax:
  CPU รู้ว่า output ไม่ขึ้นกับ input เดิมของ eax
  สามารถ "break dependency" ได้ที่ Rename stage
  ไม่ต้องรอ execution unit เลย!
  
  เรียกว่า "dependency-breaking idiom"
  
MOV eax, 0:
  CPU ต้องส่งไปยัง ALU แล้วใส่ค่า 0
  อาจยังคง dependency chain เดิม (depends on implementation)
  ปกติ latency = 1 cycle
```

### 12.3 XORPS vs XORPD vs PXOR สำหรับ Clear XMM Register

```nasm
; NASM syntax - Clearing XMM/YMM registers

; สำหรับ float computation:
xorps   xmm0, xmm0    ; 3 bytes, latency 0, clear float registers
                      ; preferred สำหรับ float operations

; สำหรับ double computation:
xorpd   xmm0, xmm0    ; 4 bytes, latency 0, similar

; สำหรับ integer SIMD:
pxor    xmm0, xmm0    ; 4 bytes, latency 0, clear integer SIMD

; AVX versions:
vxorps  ymm0, ymm0, ymm0  ; Clear 256-bit float register
vxorpd  ymm0, ymm0, ymm0  ; Clear 256-bit double register
vpxor   ymm0, ymm0, ymm0  ; Clear 256-bit integer register

; Which to use?
; - ใช้ XORPS/VXORPS สำหรับ float, even if using integer later
;   (มักจะมี better throughput)
; - Avoid mixing SSE (xorps) and AVX (vxorps) - เกิด state transition
; - หลังจาก use AVX, ใช้ VZEROUPPER ก่อน call functions ที่ใช้ SSE
```

```nasm
; ตัวอย่างปัญหา: mixing SSE and AVX
bad_example:
    vxorps  ymm0, ymm0, ymm0    ; AVX operation (256-bit)
    ; ... some AVX code ...
    xorps   xmm1, xmm1           ; SSE operation - STATE TRANSITION!
                                  ; อาจทำให้ช้าลงมาก!

good_example:
    vxorps  ymm0, ymm0, ymm0    ; AVX operation
    ; ... some AVX code ...
    vzeroupper                   ; ล้าง upper bits ก่อนใช้ SSE
    xorps   xmm1, xmm1           ; ตอนนี้ OK
```

```nasm
; Performance comparison บน Skylake:
; XORPS  xmm,xmm  - latency 0, throughput 0.33 (ports 0,1,5)
; XORPD  xmm,xmm  - latency 0, throughput 1    (port 5)
; PXOR   xmm,xmm  - latency 0, throughput 0.33 (ports 0,1,5)

; XORPS/PXOR มี throughput ดีกว่า XORPD สำหรับ clearing!
```

### 12.4 NOP ต่างๆ และ Alignment

```nasm
; NASM syntax - NOP variants
nop           ; 1 byte - do nothing (แต่ยัง issue ไปยัง port)
              
; Multi-byte NOPs (ใช้สำหรับ alignment)
%rep 2
  nop         ; 2 bytes of NOP (2 instructions)
%endrep

; Single multi-byte NOP:
db 0x66, 0x90    ; 2-byte NOP
db 0x0F, 0x1F, 0x00  ; 3-byte NOP
db 0x0F, 0x1F, 0x40, 0x00  ; 4-byte NOP
; ...

; ดีกว่า: ใช้ ALIGN directive
align 16       ; เพิ่ม NOP จนถึง 16-byte boundary
loop_start:
    ; loop body
```

---

## 13. Advanced: FMA (Fused Multiply-Add)

### 13.1 FMA คืออะไร?

FMA ทำ multiply-add ใน single operation:
```
VFMADD231PS ymm_dst, ymm_src1, ymm_src2
  ymm_dst = ymm_dst + ymm_src1 * ymm_src2   (a*b + c)
```

ประโยชน์:
1. **Accuracy**: ทำ rounding ครั้งเดียวแทนสองครั้ง
2. **Throughput**: 2 operations (MUL + ADD) ใน 1 instruction
3. **Latency**: 5 cycles (vs 4+4 = 8 cycles แบบแยก)

### 13.2 FMA Latency และ Throughput

```
VFMADD231PS (256-bit):
  Latency: 5 cycles
  Throughput: 0.5 cycles (2 per cycle)
  
เทียบกับ:
  VMULPS: latency 4, throughput 0.5
  VADDPS: latency 4, throughput 0.5
  Combined: latency 8 (if dependent), throughput 1 (bottleneck)
  
FMA ให้: latency 5 (3 cycles better!), throughput 0.5 (2x better for combined)
```

### 13.3 Dot Product ด้วย FMA

```nasm
; NASM syntax - dot product with FMA
dot_product_fma:
    ; rdi = float* a, rsi = float* b, rcx = count (multiple of 8)
    vxorps  ymm0, ymm0, ymm0   ; accumulator
    xor     eax, eax

.loop:
    vmovaps     ymm1, [rdi + rax*4]    ; Load 8 a[i]
    vfmadd231ps ymm0, ymm1, [rsi + rax*4]  ; acc += a[i] * b[i]
    add         eax, 8
    cmp         eax, ecx
    jl          .loop
    
    ; Horizontal reduce ymm0
    vextractf128  xmm1, ymm0, 1
    vaddps        xmm0, xmm0, xmm1
    vhaddps       xmm0, xmm0, xmm0
    vhaddps       xmm0, xmm0, xmm0
    ret
```

```nasm
; ดีกว่า: 4 FMA accumulators
dot_product_fma_4acc:
    vxorps  ymm0, ymm0, ymm0
    vxorps  ymm1, ymm1, ymm1
    vxorps  ymm2, ymm2, ymm2
    vxorps  ymm3, ymm3, ymm3
    xor     eax, eax

.loop:
    vmovaps     ymm4, [rdi + rax*4]
    vmovaps     ymm5, [rdi + rax*4 + 32]
    vmovaps     ymm6, [rdi + rax*4 + 64]
    vmovaps     ymm7, [rdi + rax*4 + 96]
    vfmadd231ps ymm0, ymm4, [rsi + rax*4]
    vfmadd231ps ymm1, ymm5, [rsi + rax*4 + 32]
    vfmadd231ps ymm2, ymm6, [rsi + rax*4 + 64]
    vfmadd231ps ymm3, ymm7, [rsi + rax*4 + 96]
    add         eax, 32
    cmp         eax, ecx
    jl          .loop
    
    ; Reduce 4 accumulators
    vaddps  ymm0, ymm0, ymm2
    vaddps  ymm1, ymm1, ymm3
    vaddps  ymm0, ymm0, ymm1
    ; Horizontal reduce...
    ret
```

---

## 14. Measuring Latency and Throughput

### 14.1 วิธีวัด Latency

```nasm
; NASM - วัด latency ของ instruction X
; (ใช้ dependency chain)
measure_latency:
    ; เตรียม: rdx = start_cycles
    rdtsc
    shl rdx, 32
    or  rax, rdx
    mov r8, rax   ; r8 = start

    ; ทำ 100 iterations ของ instruction ที่ต้องการวัด
    ; โดยสร้าง dependency chain ยาว
    xor eax, eax
%rep 100
    add  eax, ecx  ; ← instruction ที่วัด (dependent!)
%endrep

    ; วัด end
    rdtsc
    shl rdx, 32
    or  rax, rdx
    sub rax, r8   ; rax = elapsed cycles

    ; latency ≈ elapsed / 100
    ret
```

### 14.2 วิธีวัด Throughput

```nasm
; NASM - วัด throughput ของ instruction X
; (ใช้ independent chain)
measure_throughput:
    rdtsc
    ; ... save start ...

    ; ทำ 100 iterations ของ instruction ที่ต้องการวัด
    ; โดยใช้ registers ต่างกัน (no dependency)
%rep 25
    add  eax, ecx   ; independent
    add  ebx, edx   ; independent
    add  esi, edi   ; independent
    add  r8d, r9d   ; independent
%endrep
    ; ทั้งหมด 100 ADD แต่ไม่มี dependency

    ; ... measure end ...
    ; throughput (RTP) ≈ elapsed / 100
    ret
```

### 14.3 RDTSC และ RDTSCP

```nasm
; NASM syntax - using RDTSC for timing
timing_example:
    ; Serialize (prevent out-of-order execution)
    cpuid               ; serializing instruction (expensive!)
    
    ; Read TSC
    rdtsc               ; edx:eax = TSC
    shl  rdx, 32
    or   rax, rdx
    mov  r8, rax        ; r8 = start
    
    ; ... code to measure ...
    xor  ecx, ecx
    mov  edx, 1000
.loop:
    add  rax, rbx       ; instruction being timed
    dec  edx
    jnz  .loop
    
    ; Read end
    rdtscp              ; edx:eax = TSC, ecx = processor ID
    shl  rdx, 32
    or   rax, rdx
    sub  rax, r8        ; rax = elapsed cycles
    
    ; rax / 1000 = cycles per iteration
    ret
```

---

## 15. Tools สำหรับ Analyzing Latency/Throughput

### 15.1 Intel Architecture Code Analyzer (IACA)

```nasm
; NASM - marking code for IACA analysis
; (เฉพาะการ analyze ไม่ใช่ production code)

%define IACA_START db 0x0F, 0x0B; int3; mov ebx, 111; int3; db 0x0F, 0x0B
%define IACA_END   db 0x0F, 0x0B; int3; mov ebx, 222; int3; db 0x0F, 0x0B

section .text
my_loop:
    IACA_START
.loop:
    vmovaps ymm0, [rdi + rax*4]
    vaddps  ymm0, ymm0, ymm1
    vmovaps [rsi + rax*4], ymm0
    add     rax, 8
    cmp     rax, rdx
    jl      .loop
    IACA_END
    ret
```

### 15.2 LLVM-MCA (Machine Code Analyzer)

```bash
# ใช้ LLVM-MCA เพื่อ analyze:
# clang -O2 -S -o code.s code.c
# llvm-mca -mcpu=skylake code.s

# Output จะแสดง:
# - Timeline ของ instructions
# - Resource utilization
# - Throughput estimate
# - Bottleneck analysis
```

### 15.3 Agner Fog's Instruction Tables

Reference ที่สำคัญที่สุด:
```
https://agner.org/optimize/

ตาราง:
- Instruction latencies and throughputs for Intel/AMD
- ครบถ้วนทุก instruction
- อัพเดทสม่ำเสมอ

ตัวอย่างการอ่าน:
Instruction | Lat | TP   | Execution units
ADD         |  1  | 0.25 | p0156
IMUL r64    |  3  |  1   | p1
DIVSS       |  11 |  7   | p0
VADDPS      |  4  | 0.5  | p01
VFMADD231PS |  5  | 0.5  | p01
```

---

## 16. Practical Examples: Real-world Optimization

### 16.1 Matrix Multiply (สาธิตการ optimize)

```nasm
; NASM syntax - Simple matrix multiply inner loop
; C[i][j] += A[i][k] * B[k][j]

; Version 1: Naive (very slow)
matmul_naive_inner:
    ; rdi = &A[i][k], rsi = &B[k][j], rdx = &C[i][j]
    movss   xmm0, [rdi]
    mulss   xmm0, [rsi]
    addss   [rdx], xmm0     ; load-modify-store
    ; ปัญหา: 
    ; 1. Memory dependency chain
    ; 2. Scalar (ทำทีละ element)
    ret

; Version 2: Keep accumulator in register
matmul_better_inner:
    ; rdi = float* A_row, rsi = float* B_col, rdx = count
    ; rcx = step size for B column
    xorps   xmm0, xmm0    ; accumulator
    xor     eax, eax

.loop:
    movss   xmm1, [rdi + rax*4]
    mulss   xmm1, [rsi]     ; B stored column-major
    addss   xmm0, xmm1
    add     rsi, rcx        ; next element in B column
    inc     eax
    cmp     eax, edx
    jl      .loop
    ret
    ; ยังช้า: latency-bound (4 cycles per iteration)

; Version 3: SIMD + multiple accumulators (production quality)
matmul_simd_inner:
    ; Process multiple C[i][j..j+7] at once
    ; rdi = float* A_row, rsi = float* B, rdx = N, rcx = float* C_row
    vxorps  ymm0, ymm0, ymm0   ; C[i][0..7]
    xor     eax, eax

.loop:
    vbroadcastss ymm1, [rdi + rax*4]      ; A[i][k] * 8
    vmovaps      ymm2, [rsi + rax*4*8]   ; B[k][0..7]
    vfmadd231ps  ymm0, ymm1, ymm2        ; C += A * B
    inc     eax
    cmp     eax, edx
    jl      .loop
    
    vaddps  ymm0, ymm0, [rcx]   ; Add existing C values
    vmovaps [rcx], ymm0
    ret
```

### 16.2 String Search Optimization

```nasm
; GAS syntax - Fast string search using SIMD
.text
.globl simd_strchr
simd_strchr:
    # rdi = char* str, rsi = char to find (in lowest byte of esi)
    
    # Broadcast search char to all 16 bytes
    movd        %esi, %xmm1
    vpbroadcastb %xmm1, %xmm1    # xmm1 = [c,c,c,c,...] (16 copies)
    
    xor         %eax, %eax
.loop:
    # Load 16 bytes from string
    movdqu      (%rdi,%rax), %xmm0
    
    # Compare all 16 bytes simultaneously
    pcmpeqb     %xmm1, %xmm0     # xmm0 = 0xFF where match, 0 otherwise
    
    # Get bitmask of matches
    pmovmskb    %xmm0, %ecx      # ecx = bitmask (1 bit per byte)
    
    test        %ecx, %ecx
    jnz         .found            # Any match?
    
    add         $16, %eax
    jmp         .loop
    
.found:
    bsf         %ecx, %ecx        # Find first set bit
    add         %ecx, %eax        # Position = block_start + bit_position
    ret
```

---

## 17. Memory Access Patterns และ Latency

### 17.1 Memory Latency ต่าง Level

```
L1 Cache Hit:   4-5 cycles
L2 Cache Hit:   12 cycles
L3 Cache Hit:   40-50 cycles
RAM Access:     150-300 cycles
SSD (NVMe):     100,000+ cycles
HDD:            10,000,000+ cycles
```

### 17.2 Prefetching

```nasm
; NASM syntax - Software prefetching
; ดึง data ล่วงหน้าก่อนที่จะต้องใช้

section .text
loop_with_prefetch:
    ; rdi = float* data, rcx = count
    xor     eax, eax
    xorps   xmm0, xmm0
    
.loop:
    ; Prefetch data ที่จะใช้ใน ~8 iterations ข้างหน้า
    ; (8 * 4 bytes = 32 bytes ahead for float)
    prefetchnta [rdi + rax*4 + 256]  ; prefetch 64 elements ahead
    
    addss   xmm0, [rdi + rax*4]
    inc     eax
    cmp     eax, ecx
    jl      .loop
    ret
```

### 17.3 Cache Line Alignment

```nasm
; NASM syntax - Proper alignment
section .data
align 64            ; Align to cache line boundary (64 bytes)
my_array: dd 1, 2, 3, 4, 5, 6, 7, 8  ; 32 bytes (8 floats)
          dd 9, 10, 11, 12, 13, 14, 15, 16

; ถ้าไม่ align อาจมี cache line split:
; array ครอม 2 cache lines = 2x memory accesses
```

---

## 18. Branch Prediction และ Performance

### 18.1 Branch Misprediction Cost

```
Misprediction penalty: ~15-20 cycles บน Skylake
```

### 18.2 Branchless Code

```nasm
; NASM syntax - Branchless abs value
; int abs(int x)

; Version 1: With branch (risky if unpredictable)
abs_branch:
    test  edi, edi
    jge   .positive
    neg   edi
.positive:
    mov   eax, edi
    ret

; Version 2: Branchless (always fast, no prediction needed)
abs_branchless:
    mov   eax, edi
    sar   edi, 31     ; edi = 0 หรือ -1 (sign extension)
    xor   eax, edi    ; flip bits if negative
    sub   eax, edi    ; add 1 if negative (two's complement trick)
    ret

; Version 3: CMOV (conditional move - no branch)
abs_cmov:
    mov   eax, edi
    neg   edi
    cmovl eax, edi    ; if was negative, use negated value
    ret
```

```nasm
; NASM syntax - Branchless max
max_branchless:
    ; max(edi, esi)
    cmp   edi, esi
    mov   eax, edi
    cmovl eax, esi    ; if edi < esi, eax = esi
    ret
```

```nasm
; NASM syntax - Branchless min
min_branchless:
    ; min(edi, esi)
    cmp   edi, esi
    mov   eax, edi
    cmovg eax, esi    ; if edi > esi, eax = esi
    ret
```

---

## 19. Out-of-Order Execution และ Instruction Scheduling

### 19.1 Reordering Instructions by Hand

```nasm
; GAS syntax - Manual instruction scheduling
; Goal: avoid pipeline stalls

; Bad: Load then immediately use (load latency = 4-5 cycles)
bad_schedule:
    mov     (%rdi), %rax    # Load A
    add     %rax, %rbx      # USE A immediately! Stall 3-4 cycles
    mov     (%rsi), %rcx    # Load B
    add     %rcx, %rdx      # USE B immediately! Stall 3-4 cycles
    ret

; Good: Interleave loads and uses
good_schedule:
    mov     (%rdi), %rax    # Load A
    mov     (%rsi), %rcx    # Load B  (don't use A yet)
    mov     8(%rdi), %r8    # Load C  (don't use A or B yet)
    add     %rax, %rbx      # Use A (after 3 instructions = ~3 cycles)
    mov     8(%rsi), %r9    # Load D
    add     %rcx, %rdx      # Use B (after 3 instructions)
    add     %r8, %r10       # Use C
    add     %r9, %r11       # Use D
    ret
    ; CPU Out-of-order จะทำแบบนี้เองบ้าง แต่ manual ดีกว่า
```

### 19.2 Register Pressure

```nasm
; NASM syntax - Managing register pressure
; x86-64 มี 16 general-purpose registers: rax, rbx, rcx, rdx,
; rsi, rdi, rbp, rsp, r8-r15
; ถ้าใช้ทั้งหมด Out-of-Order engine มีอิสระมากขึ้น

; Example: จงใช้ registers ให้หลากหลาย
register_pressure_example:
    ; แทนที่จะใช้ rax ตลอด:
    add  rax, rbx      ; chain 1
    add  rax, rcx      ; chain 1 (dependent!)
    add  rax, rdx      ; chain 1 (dependent!)
    
    ; ใช้ registers หลายตัว:
    add  rax, rbx      ; chain 1
    add  r8,  r9       ; chain 2 (independent)
    add  r10, r11      ; chain 3 (independent)
    ; 3 chains ทำงานพร้อมกันได้!
    ret
```

---

## 20. Performance Model สมบูรณ์ (Comprehensive Model)

### 20.1 Bottleneck Analysis Framework

```
Performance bottlenecks:
1. Latency-bound:     Dependency chains เป็น bottleneck
2. Throughput-bound:  Port pressure เป็น bottleneck
3. Memory-bound:      Cache/RAM bandwidth เป็น bottleneck
4. Frontend-bound:    Decode/instruction supply เป็น bottleneck
```

### 20.2 วิธีหา Bottleneck

```python
# Python pseudocode for analysis

def analyze_loop(instructions, cpu):
    # 1. Build dependency graph
    dep_graph = build_dep_graph(instructions)
    
    # 2. Find critical path (latency bound)
    critical_path_len = longest_path(dep_graph, cpu.latencies)
    latency_bound = critical_path_len
    
    # 3. Count port pressure (throughput bound)
    port_counts = {}
    for inst in instructions:
        for port in cpu.get_ports(inst):
            port_counts[port] = port_counts.get(port, 0) + 1 / len(cpu.get_ports(inst))
    throughput_bound = max(port_counts.values())
    
    # 4. Count memory accesses (memory bound)
    loads = sum(1 for i in instructions if i.is_load)
    stores = sum(1 for i in instructions if i.is_store)
    memory_bound = max(loads / cpu.load_ports, stores / cpu.store_ports)
    
    return max(latency_bound, throughput_bound, memory_bound)
```

### 20.3 ตัวอย่างการวิเคราะห์แบบ Manual

```nasm
; NASM - Analyze this loop:
.loop:
    vmovaps     ymm0, [rdi + rax*4]     ; Load 8 floats: port 2/3
    vmulps      ymm0, ymm0, ymm1        ; Multiply: port 0/1, lat=4
    vaddps      ymm2, ymm2, ymm0        ; Add to acc: port 0/1, lat=4
    add         rax, 8                   ; port 0/1/5/6, lat=1
    cmp         rax, rcx                 ; port 0/1/5/6, lat=1
    jl          .loop                    ; port 6, lat=1
```

```
วิเคราะห์:
Port Pressure:
  Port 0/1: vmulps(0.5) + vaddps(0.5) + add(0.25) + cmp(0.25) = ~1.5/cycle
  Port 2/3: vmovaps(0.5 each)
  Port 6:   jl(1)
  
Latency chain:
  vmovaps -> vmulps (4) -> vaddps (4) -> ... next iteration
  Chain: 4 cycles per iteration (if 1 accumulator)
  
Memory:
  1 load per 8 elements = 0.125 loads/element
  
Bottleneck: vaddps chain (4 cycles) if latency-bound
            BUT if we use 4+ accumulators: port pressure (~1.5 cycles)
```

---

## 21. Summary Tables

### 21.1 Quick Reference: Skylake Latencies

```
=== Integer (most important) ===
MOV r,r:     L=0/1,  T=0.25   (register rename!)
ADD/SUB:     L=1,    T=0.25
AND/OR/XOR:  L=1,    T=0.25
IMUL r64:    L=3,    T=1
LEA (simple):L=1,    T=0.25
SHL/SHR:     L=1,    T=0.5
IDIV r64:    L=~60,  T=~40    (VERY SLOW)

=== Float (SSE scalar) ===
ADDSS/SUBSS: L=4,    T=0.5
MULSS:       L=4,    T=0.5
DIVSS:       L=11,   T=7
SQRTSS:      L=12,   T=7
RCPSS:       L=4,    T=1

=== SIMD (AVX/AVX2 256-bit) ===
VADDPS:      L=4,    T=0.5
VMULPS:      L=4,    T=0.5
VFMADD231PS: L=5,    T=0.5
VDIVPS:      L=18,   T=14
VSQRTPS:     L=21,   T=14
VRCPPS:      L=4,    T=1
VPAND/OR/XOR:L=1,    T=0.33
VPADDQ:      L=1,    T=0.5
```

### 21.2 Optimization Rules of Thumb

```
1. ใช้ XOR reg,reg แทน MOV reg,0 (dependency break)
2. ใช้ XORPS/VXORPS แทน XOR สำหรับ float registers
3. หลีกเลี่ยง DIV - ใช้ magic number multiply แทน
4. ใช้ multiple accumulators เพื่อ break dependency chains
5. ใช้ FMA แทน FADD+FMUL
6. ใช้ SIMD (8x width) + multiple accumulators (4-8x)
7. Align data ตาม cache line (64 bytes)
8. Prefetch data ล่วงหน้า
9. Mix instruction types เพื่อ use multiple ports
10. หลีก mixing SSE/AVX (ใช้ VZEROUPPER เมื่อจำเป็น)
```

---

## 22. Exercises (แบบฝึกหัด)

### Exercise 1: Identify Bottleneck

```nasm
; วิเคราะห์ว่า loop นี้ bound อะไร?
.loop:
    movss   xmm0, [rdi + rax*4]
    addss   xmm1, xmm0              ; ← มี dependency chain!
    inc     rax
    cmp     rax, rsi
    jl      .loop
```

คำตอบ: Latency-bound (addss dependency chain = 4 cycles/iter)

### Exercise 2: Optimize the Loop

```nasm
; Optimize นี้ให้เร็วขึ้น 4x:
.loop:
    movss   xmm0, [rdi + rax*4]
    mulss   xmm0, xmm0
    addss   xmm1, xmm0
    inc     rax
    cmp     rax, rsi
    jl      .loop
```

```nasm
; Solution: 4 accumulators
    xorps   xmm1, xmm1   ; acc0
    xorps   xmm3, xmm3   ; acc1
    xorps   xmm5, xmm5   ; acc2
    xorps   xmm7, xmm7   ; acc3
.loop:
    movss   xmm0, [rdi + rax*4]
    movss   xmm2, [rdi + rax*4 + 4]
    movss   xmm4, [rdi + rax*4 + 8]
    movss   xmm6, [rdi + rax*4 + 12]
    mulss   xmm0, xmm0
    mulss   xmm2, xmm2
    mulss   xmm4, xmm4
    mulss   xmm6, xmm6
    addss   xmm1, xmm0
    addss   xmm3, xmm2
    addss   xmm5, xmm4
    addss   xmm7, xmm6
    add     rax, 4
    cmp     rax, rsi
    jl      .loop
    addss   xmm1, xmm3
    addss   xmm5, xmm7
    addss   xmm1, xmm5
```

### Exercise 3: Strength Reduction

```nasm
; แปลงนี้:
imul rax, 12      ; rax = rax * 12
```

```nasm
; Solution:
lea  rax, [rax + rax*2]  ; rax = rax * 3
shl  rax, 2              ; rax = rax * 4  (total: rax * 12)
```

### Exercise 4: Reciprocal

```nasm
; ใช้ RCPPS แทน DIVPS สำหรับ a[i] / b[i]:
; แต่ต้องการ precision ระดับ float

; Solution:
vrcpps   ymm2, ymm1         ; ymm2 ≈ 1/b[i]
; Newton-Raphson: y1 = y0*(2 - b*y0)
vmulps   ymm3, ymm1, ymm2   ; ymm3 = b * y0
movaps   ymm4, [two_avx]    ; ymm4 = 2.0
vsubps   ymm4, ymm4, ymm3   ; ymm4 = 2 - b*y0
vmulps   ymm2, ymm2, ymm4   ; ymm2 = y0*(2-b*y0) = 1/b (full precision)
vmulps   ymm0, ymm0, ymm2   ; ymm0 = a * (1/b) = a/b
```

---

## 23. Cheat Sheet: Common Patterns

### 23.1 Zero Registers

```nasm
xor  eax, eax          ; zero rax (best!)
xorps xmm0, xmm0       ; zero xmm0 (float)
vxorps ymm0, ymm0, ymm0; zero ymm0 (AVX float)
pxor  xmm0, xmm0       ; zero xmm0 (integer SIMD)
vpxor ymm0, ymm0, ymm0 ; zero ymm0 (AVX integer)
```

### 23.2 Multiply by Constants

```nasm
; x * 2   = shl x, 1   or  lea [x + x]
; x * 3   = lea [x + x*2]
; x * 4   = shl x, 2   or  lea [x*4]
; x * 5   = lea [x + x*4]
; x * 6   = lea [x + x*2]; shl x, 1
; x * 7   = lea [x*8]; sub x, original
; x * 8   = shl x, 3   or  lea [x*8]
; x * 9   = lea [x + x*8]
; x * 10  = lea [x + x*4]; shl x, 1
; x * 16  = shl x, 4
; x * 32  = shl x, 5
; x * 64  = shl x, 6
```

### 23.3 Dependency Breaking Patterns

```nasm
; Pattern 1: XOR to break dep
xor  eax, eax     ; ← breaks any dep on old eax

; Pattern 2: Multiple accumulators
xorps xmm0, xmm0  ; acc0
xorps xmm1, xmm1  ; acc1
xorps xmm2, xmm2  ; acc2
xorps xmm3, xmm3  ; acc3
; ... use all 4, combine at end

; Pattern 3: SIMD for 8x
vxorps ymm0, ymm0, ymm0  ; 8 accumulators in one register
```

### 23.4 Fast Division by Common Constants

```nasm
; Division by 2 (signed):
sar  eax, 1       ; or: cdq; idiv 2 (much slower!)

; Division by 4 (signed):
sar  eax, 2

; Division by 8 (signed):
sar  eax, 3

; Division by 2 (unsigned):
shr  eax, 1

; Division by constant N (general, use compiler output):
; gcc -O2 -S -masm=intel to see magic numbers
```

---

## 24. สรุป (Conclusion)

### หลักการสำคัญ (Key Principles)

1. **Latency** กำหนดประสิทธิภาพของ dependent code  
   **Throughput** กำหนดประสิทธิภาพของ independent code

2. **Dependency chains** ทำให้โค้ดช้าลง  
   แก้ไขด้วย multiple accumulators, loop unrolling

3. **SIMD** เพิ่ม parallelism 8x (AVX float) หรือมากกว่า

4. **FMA** ลด latency และเพิ่ม throughput สำหรับ fused operations

5. **Division** ช้ามาก ควรใช้ magic number multiplication แทน

6. **Strength reduction** แปลง heavy ops เป็น light ops  
   (IMUL → SHL, IDIV → SAR/multiply)

7. **Instruction selection** มีความสำคัญ:  
   XOR vs MOV, XORPS vs XORPD vs PXOR

8. **Out-of-order execution** ช่วยได้แต่ไม่ใช่ magic  
   เราต้องช่วย CPU โดย scheduling instructions ด้วยตัวเอง

9. **วัดจริงเสมอ** - อย่า assume ว่าอะไรเร็วกว่า  
   ใช้ RDTSC, LLVM-MCA, Agner Fog's tables

10. **Profile-guided optimization** ให้แก้ hotspot ก่อน  
    อย่า optimize ทุกอย่างโดยไม่รู้ว่า bottleneck อยู่ที่ไหน

---

## 25. Additional Resources

### เอกสารอ้างอิงที่แนะนำ:

```
1. Agner Fog's Optimization Manuals (เล่มที่ดีที่สุด):
   https://agner.org/optimize/
   
2. Intel® 64 and IA-32 Architectures Optimization Reference Manual:
   https://intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
   
3. Hacker's Delight (สำหรับ magic numbers):
   Henry S. Warren Jr.
   
4. What Every Programmer Should Know About Memory:
   Ulrich Drepper (ฟรีบน lwn.net)
   
5. Computer Architecture: A Quantitative Approach:
   Hennessy & Patterson
   
6. uops.info - ข้อมูล latency/throughput ที่วัดจริง:
   https://uops.info/
```

### Tools ที่แนะนำ:

```
- LLVM-MCA:    llvm.org/docs/CommandGuide/llvm-mca.html
- IACA:        Intel Architecture Code Analyzer (deprecated แต่ยังใช้ได้)
- Perf:        Linux perf tool (perf stat, perf record, perf report)
- VTune:       Intel VTune Profiler
- AMD μProf:   AMD's equivalent of VTune
- Cachegrind:  Valgrind tool for cache simulation
```

---

*End of Part 064: Instruction Latency and Throughput*

*จบ Part 064 แล้ว ใน Part ถัดไปเราจะพูดถึง Vectorization และ Auto-vectorization*
