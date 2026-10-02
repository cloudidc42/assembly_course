# Part 067: Memory Bandwidth Optimization

## บทนำ (Introduction)

Memory bandwidth คือหนึ่งใน bottleneck ที่สำคัญที่สุดในระบบประมวลผลสมัยใหม่ แม้ว่า CPU จะมีความเร็วสูงมาก แต่ถ้าไม่สามารถดึงข้อมูลจาก DRAM ได้เร็วพอ ประสิทธิภาพก็จะตกลงอย่างมาก บทนี้จะครอบคลุมเทคนิคต่างๆ ในการเพิ่มประสิทธิภาพ memory bandwidth ตั้งแต่ระดับพื้นฐานไปจนถึงขั้นสูง

---

## 1. Memory Bandwidth Limits และ Memory Wall Problem

### 1.1 DRAM Bandwidth คืออะไร

DRAM bandwidth คือปริมาณข้อมูลสูงสุดที่สามารถถ่ายโอนระหว่าง CPU และ DRAM ในหนึ่งหน่วยเวลา โดยทั่วไปวัดเป็น GB/s (Gigabytes per second)

**ตัวอย่างความสามารถของ memory ชนิดต่างๆ:**

| Memory Type     | Theoretical BW  | Practical BW   |
|-----------------|-----------------|----------------|
| DDR4-3200       | 51.2 GB/s       | ~35-45 GB/s    |
| DDR5-4800       | 76.8 GB/s       | ~55-65 GB/s    |
| DDR5-6400       | 102.4 GB/s      | ~75-85 GB/s    |
| LPDDR5-6400     | 51.2 GB/s       | ~40-45 GB/s    |
| HBM2e           | 460 GB/s        | ~400 GB/s      |
| HBM3            | 819 GB/s        | ~700 GB/s      |

**สูตรคำนวณ Theoretical Bandwidth:**
```
Theoretical BW = Memory Clock × Bus Width × Channels × Multiplier
ตัวอย่าง DDR4-3200 dual channel:
= 1600 MHz × 64 bits × 2 channels × 2 (DDR)
= 1600 × 10^6 × 8 bytes × 2 × 2
= 51.2 GB/s
```

### 1.2 The Memory Wall Problem

"Memory Wall" คือปัญหาที่ความเร็ว CPU เพิ่มขึ้นเร็วกว่า memory bandwidth มาก ทำให้ CPU ต้อง stall รอข้อมูลจาก memory

```
ปี 1995:  CPU ~100 MIPS,  DRAM BW ~100 MB/s   → Ratio 1:1
ปี 2000:  CPU ~1000 MIPS, DRAM BW ~1 GB/s     → Ratio 1:1
ปี 2010:  CPU ~50 GFLOPS, DRAM BW ~20 GB/s    → Ratio 2.5:1
ปี 2020:  CPU ~500 GFLOPS,DRAM BW ~50 GB/s    → Ratio 10:1
ปี 2024:  CPU ~5 TFLOPS,  DRAM BW ~80 GB/s    → Ratio 60:1
```

**ผลกระทบของ Memory Wall:**
- CPU cores นั่ง idle รอข้อมูล
- Cache misses ทำให้ stall หลายร้อย cycles
- Parallelism ถูกจำกัดโดย bandwidth ไม่ใช่ compute

### 1.3 Memory Latency vs Bandwidth

ต้องเข้าใจความแตกต่างระหว่าง latency และ bandwidth:

```
Latency  = เวลาในการเข้าถึงข้อมูล 1 ครั้ง (nanoseconds)
Bandwidth = ปริมาณข้อมูลที่ถ่ายโอนได้ต่อวินาที (GB/s)

DDR4-3200 latency: ~60-80 ns (ไปกลับ)
DDR4-3200 bandwidth: ~35-45 GB/s (streaming)

L1 cache latency:  ~4 cycles  (~1.3 ns at 3 GHz)
L2 cache latency:  ~12 cycles (~4 ns)
L3 cache latency:  ~40 cycles (~13 ns)
DRAM latency:      ~200 cycles (~67 ns)
```

---

## 2. STREAM Benchmark

### 2.1 STREAM Benchmark คืออะไร

STREAM เป็น benchmark มาตรฐานสำหรับวัด sustainable memory bandwidth พัฒนาโดย Dr. John McCalpin ประกอบด้วย 4 kernel:

```c
/* STREAM Kernels */
// 1. Copy:     c[i] = a[i]
// 2. Scale:    b[i] = scalar * c[i]
// 3. Add:      c[i] = a[i] + b[i]
// 4. Triad:    a[i] = b[i] + scalar * c[i]
```

### 2.2 การแปลผล STREAM Benchmark

```
ตัวอย่างผลลัพธ์:
Function    Best Rate MB/s  Avg time     Min time     Max time
Copy:           45234.2     0.070745     0.070745     0.070747
Scale:          44891.1     0.071288     0.071288     0.071291
Add:            46123.4     0.103876     0.103876     0.103876
Triad:          46001.2     0.104162     0.104162     0.104163
```

**การแปลผล:**
- Copy rate = 45.2 GB/s → memory bandwidth ~90% ของ theoretical
- ถ้า Copy rate < 60% ของ theoretical: ปัญหา NUMA หรือ configuration
- Triad rate สูงกว่า Add: แสดงว่า hardware prefetcher ทำงานดี

### 2.3 การทดสอบ STREAM แบบ Assembly

```nasm
; NASM - STREAM Copy kernel
; รับ: rdi = dst, rsi = src, rdx = count (จำนวน 64-byte blocks)
section .text
global stream_copy_avx512

stream_copy_avx512:
    push    rbx
    
    ; ตรวจสอบว่า count > 0
    test    rdx, rdx
    jz      .done
    
    ; ลูปหลัก - copy 64 bytes ต่อครั้ง (1 AVX-512 register)
.loop:
    vmovdqu64   zmm0, [rsi]         ; load 64 bytes
    vmovdqu64   [rdi], zmm0         ; store 64 bytes
    add     rsi, 64
    add     rdi, 64
    dec     rdx
    jnz     .loop
    
    vzeroupper
.done:
    pop     rbx
    ret

; GAS syntax equivalent:
; .global stream_copy_avx512
; stream_copy_avx512:
;     push %rbx
;     test %rdx, %rdx
;     jz .done
; .loop:
;     vmovdqu64 (%rsi), %zmm0
;     vmovdqu64 %zmm0, (%rdi)
;     addq $64, %rsi
;     addq $64, %rdi
;     decq %rdx
;     jnz .loop
;     vzeroupper
; .done:
;     pop %rbx
;     ret
```

---

## 3. Write Bandwidth vs Read Bandwidth

### 3.1 ความแตกต่างระหว่าง Read และ Write

Read และ Write bandwidth ไม่เท่ากันเสมอ:

```
Read Bandwidth:
- CPU request cache line (64 bytes)
- Memory controller ส่งข้อมูลมา
- ข้อมูลเข้า cache และ CPU registers

Write Bandwidth (ปกติ - Write-Back):
- CPU write ไป cache line
- ถ้า cache miss: ต้อง Read-For-Ownership (RFO) ก่อน!
- RFO = อ่านมาก่อน แล้วค่อย modify และ dirty mark
- Effective write bandwidth = ต่ำกว่า read bandwidth
```

### 3.2 Write Bandwidth ในสถานการณ์ต่างๆ

```
Scenario 1: Write to cached line
  → ไม่ต้องไปถึง DRAM, แค่ write ไป cache
  → ใช้ write bandwidth ของ cache ไม่ใช่ DRAM

Scenario 2: Write to uncached line (with RFO)
  → ต้อง read 64 bytes จาก DRAM ก่อน (RFO)
  → แล้ว write กลับ 64 bytes
  → ใช้ bandwidth 2x เพื่อ write 1x
  → Effective bandwidth = DRAM BW / 2

Scenario 3: NT store (Non-Temporal)
  → ไม่ต้องทำ RFO
  → เขียนตรงไป DRAM ผ่าน Write Combining Buffer
  → Effective bandwidth ≈ DRAM BW เต็มๆ
```

---

## 4. RFO (Read-For-Ownership)

### 4.1 RFO คืออะไร

RFO เป็น protocol ใน cache coherency ที่ CPU ต้องทำก่อนที่จะ write ไป cache line ที่ไม่ได้ cache อยู่

```
ขั้นตอน RFO:
1. CPU ต้องการ write ไป address X
2. Check cache: miss!
3. ส่ง RFO request ไป memory controller
4. ถ้ามี CPU อื่นมี copy ใน cache: invalidate ก่อน
5. Memory controller ส่ง cache line มาให้
6. CPU ได้ ownership ของ cache line
7. CPU ทำ write ได้แล้ว

ต้นทุน RFO:
- L3 miss → DRAM latency: ~200 cycles
- Multi-socket: ยิ่งแพงกว่า (NUMA)
- False sharing: RFO แม้ไม่ได้ write data เดียวกัน
```

### 4.2 ตัวอย่าง RFO ใน Assembly

```nasm
; NASM - ตัวอย่างที่ trigger RFO บ่อย
section .text
global bad_scatter_write

; รับ: rdi = array, rsi = indices, rdx = values, rcx = count
; เขียน values[i] ไปที่ array[indices[i]] - pattern นี้ RFO บ่อยมาก!
bad_scatter_write:
    test    rcx, rcx
    jz      .done
    
.loop:
    mov     eax, [rsi]              ; load index
    mov     r8d, [rdx]              ; load value
    
    ; write ไป random location - likely cache miss → RFO!
    mov     [rdi + rax*4], r8d
    
    add     rsi, 4
    add     rdx, 4
    dec     rcx
    jnz     .loop
    
.done:
    ret
```

### 4.3 วิธีลด RFO

```nasm
; NASM - ลด RFO โดยการ prefetch ก่อน write
section .text
global better_scatter_write

better_scatter_write:
    test    rcx, rcx
    jz      .done
    
    ; Prefetch for write (T0 = into L1)
    ; ให้ hardware มีเวลา fetch cache line ก่อน
    cmp     rcx, 8
    jl      .no_prefetch
    
.loop_with_prefetch:
    ; Prefetch future write targets (8 iterations ahead)
    mov     eax, [rsi + 32]         ; load index 8 ahead
    prefetchw [rdi + rax*4]         ; prefetch with write intent
    
    ; Process current
    mov     eax, [rsi]
    mov     r8d, [rdx]
    mov     [rdi + rax*4], r8d
    
    add     rsi, 4
    add     rdx, 4
    dec     rcx
    cmp     rcx, 8
    jge     .loop_with_prefetch
    
.no_prefetch:
    test    rcx, rcx
    jz      .done
    
.loop_tail:
    mov     eax, [rsi]
    mov     r8d, [rdx]
    mov     [rdi + rax*4], r8d
    add     rsi, 4
    add     rdx, 4
    dec     rcx
    jnz     .loop_tail
    
.done:
    ret
```

---

## 5. Non-Temporal (NT) Stores

### 5.1 NT Stores คืออะไร

Non-Temporal stores เป็น instruction พิเศษที่ bypass cache และเขียนตรงไปยัง memory ผ่าน Write Combining Buffer (WCB) โดยไม่ต้องทำ RFO

**คำสั่ง NT Store ที่สำคัญ:**

| Instruction    | Data Width | Description                    |
|----------------|------------|--------------------------------|
| `MOVNTPS`      | 128-bit    | NT store 4 floats (SSE)        |
| `MOVNTPD`      | 128-bit    | NT store 2 doubles (SSE2)      |
| `MOVNTDQ`      | 128-bit    | NT store 2 integers (SSE2)     |
| `MOVNTI`       | 32/64-bit  | NT store integer               |
| `VMOVNTPS`     | 256/512-bit| NT store floats (AVX/AVX-512)  |
| `VMOVNTPD`     | 256/512-bit| NT store doubles (AVX/AVX-512) |
| `VMOVNTDQ`     | 256/512-bit| NT store integers (AVX/AVX-512)|

### 5.2 NT Stores Bypass Cache

```
ปกติ (Temporal Store):
CPU → L1 Cache → L2 Cache → L3 Cache → DRAM
      ↑ dirty bit set, eviction later

NT Store:
CPU → Write Combining Buffer → DRAM
      (ไม่แตะ cache เลย!)

ข้อดี NT stores:
1. ไม่ต้องทำ RFO → ประหยัด bandwidth
2. ไม่ pollute cache ด้วยข้อมูลที่ใช้แค่ครั้งเดียว
3. Write Combining รวม writes เล็กๆ เป็น cache-line-sized write
4. Streaming write bandwidth เต็ม~100%

ข้อเสีย NT stores:
1. ข้อมูลไม่อยู่ใน cache (ต้องไป DRAM ถ้าอ่านทันที)
2. ต้องการ SFENCE เพื่อ ensure ordering
3. ต้อง aligned (ส่วนใหญ่)
```

### 5.3 ตัวอย่าง NT Stores ใน NASM

```nasm
; NASM - Streaming write ด้วย NT stores (AVX2)
section .text
global nt_memcpy_avx2

; NT memcpy - เหมาะสำหรับ large buffers (> L3 cache size)
; รับ: rdi = dst, rsi = src, rdx = bytes
nt_memcpy_avx2:
    push    rbx
    push    r12
    push    r13
    
    mov     rbx, rdx                ; save count
    
    ; Process 256-bit (32-byte) chunks
    mov     r12, rdx
    shr     r12, 5                  ; divide by 32
    
.loop256:
    test    r12, r12
    jz      .remainder
    
    vmovdqu ymm0, [rsi]             ; load 32 bytes (temporal)
    vmovntdq [rdi], ymm0            ; NT store 32 bytes
    
    add     rsi, 32
    add     rdi, 32
    dec     r12
    jnz     .loop256
    
.remainder:
    ; Process remaining bytes
    mov     r13, rbx
    and     r13, 31                 ; bytes % 32
    
.loop_byte:
    test    r13, r13
    jz      .done
    
    mov     al, [rsi]
    mov     [rdi], al
    inc     rsi
    inc     rdi
    dec     r13
    jnz     .loop_byte
    
.done:
    ; SFENCE required after NT stores!
    sfence
    
    vzeroupper
    pop     r13
    pop     r12
    pop     rbx
    ret
```

### 5.4 NT Stores แบบ 512-bit (AVX-512)

```nasm
; NASM - Maximum bandwidth NT memcpy ด้วย AVX-512
section .text
global nt_memcpy_avx512

; รับ: rdi = dst, rsi = src, rdx = bytes (ควร 64-byte aligned)
nt_memcpy_avx512:
    push    rbx
    
    mov     rbx, rdx
    shr     rbx, 6                  ; divide by 64 (AVX-512 register size)
    
    test    rbx, rbx
    jz      .remainder
    
    ; Unroll 4x to maximize throughput
    mov     rcx, rbx
    shr     rcx, 2                  ; 4x unroll count
    and     rbx, 3                  ; remainder blocks
    
.loop_unrolled:
    test    rcx, rcx
    jz      .loop_single
    
    ; Load 4 x 64 bytes
    vmovdqu64   zmm0, [rsi]
    vmovdqu64   zmm1, [rsi + 64]
    vmovdqu64   zmm2, [rsi + 128]
    vmovdqu64   zmm3, [rsi + 192]
    
    ; NT store 4 x 64 bytes
    vmovntdq    [rdi],       zmm0
    vmovntdq    [rdi + 64],  zmm1
    vmovntdq    [rdi + 128], zmm2
    vmovntdq    [rdi + 192], zmm3
    
    add     rsi, 256
    add     rdi, 256
    dec     rcx
    jnz     .loop_unrolled
    
.loop_single:
    test    rbx, rbx
    jz      .remainder
    
.loop_64:
    vmovdqu64   zmm0, [rsi]
    vmovntdq    [rdi], zmm0
    add     rsi, 64
    add     rdi, 64
    dec     rbx
    jnz     .loop_64
    
.remainder:
    ; handle remaining < 64 bytes with scalar code
    mov     rcx, rdx
    and     rcx, 63
    
.byte_loop:
    test    rcx, rcx
    jz      .done
    mov     al, [rsi]
    mov     [rdi], al
    inc     rsi
    inc     rdi
    dec     rcx
    jnz     .byte_loop
    
.done:
    sfence                          ; Ensure NT stores are visible
    ret
```

### 5.5 GAS Syntax - NT Stores

```gas
# GAS (AT&T syntax) - NT store examples

.section .text
.global nt_store_example

nt_store_example:
    # Store scalar integer NT
    movnti  %eax, (%rdi)            # 32-bit NT store
    movnti  %rax, (%rdi)            # 64-bit NT store
    
    # Store SSE NT
    movntps %xmm0, (%rdi)           # 128-bit float NT
    movntpd %xmm0, (%rdi)           # 128-bit double NT
    movntdq %xmm0, (%rdi)           # 128-bit integer NT
    
    # Store AVX NT
    vmovntps %ymm0, (%rdi)          # 256-bit float NT
    vmovntpd %ymm0, (%rdi)          # 256-bit double NT
    vmovntdq %ymm0, (%rdi)          # 256-bit integer NT
    
    # Store AVX-512 NT
    vmovntps %zmm0, (%rdi)          # 512-bit float NT
    vmovntdq %zmm0, (%rdi)          # 512-bit integer NT
    
    # MANDATORY fence after NT stores
    sfence
    
    ret
```

---

## 6. Write Combining Buffer (WCB)

### 6.1 WCB คืออะไร

Write Combining Buffer (WCB) เป็น buffer พิเศษใน CPU ที่รวม (combine) NT stores หลายๆ ครั้งให้เป็น full cache line (64 bytes) ก่อนที่จะส่งไป DRAM

```
โครงสร้าง WCB:
+---------+---------+---------+---------+---------+---------+
| WCB [0] | WCB [1] | WCB [2] | WCB [3] | WCB [4] | WCB [5] |  ← 4-6 entries
+---------+---------+---------+---------+---------+---------+
  64 bytes  64 bytes  64 bytes  64 bytes  64 bytes  64 bytes
  
แต่ละ WCB entry มี:
- Address (cache line aligned)
- Validity bits (64 bits, 1 per byte)
- Data (64 bytes)
- State (empty/partial/full)
```

### 6.2 การทำงานของ WCB

```
Timeline of NT writes to same cache line:

t=1: NT write 8 bytes to 0x1000    → WCB[0] partial (8/64 bytes valid)
t=2: NT write 8 bytes to 0x1008    → WCB[0] partial (16/64 bytes valid)
...
t=8: NT write 8 bytes to 0x1038    → WCB[0] full (64/64 bytes valid)
t=9: WCB[0] flushed to DRAM        → 1 full cache line write

vs. ถ้า write ไม่เรียงลำดับ:
t=1: NT write to 0x1000   → WCB[0]
t=2: NT write to 0x2000   → WCB[1]
t=3: NT write to 0x3000   → WCB[2]
t=4: NT write to 0x4000   → WCB[3]
t=5: NT write to 0x5000   → WCB[4]
t=6: NT write to 0x6000   → WCB[5]
t=7: NT write to 0x7000   → WCB full! flush WCB[0] (partial) → waste!
```

### 6.3 Partial Line Fills - ปัญหาหลักของ WCB

```nasm
; NASM - ตัวอย่าง BAD pattern - partial WCB fills

; BAD: สลับระหว่าง cache lines → partial fills
bad_interleaved_nt_write:
    ; กำลัง write ไป dst1 และ dst2 สลับกัน
    ; → 2 WCB entries ถูกใช้พร้อมกัน, partial fills

    mov     rax, rdi                ; dst1
    mov     rbx, rsi                ; dst2
    xor     rcx, rcx
    
.loop:
    cmp     rcx, 8
    jge     .done
    
    ; Interleaved writes - BAD for WCB!
    movnti  [rax + rcx*8], r8      ; write to dst1
    movnti  [rbx + rcx*8], r8      ; write to dst2
    inc     rcx
    jmp     .loop
    
.done:
    sfence
    ret

; -----------------------------------------

; GOOD: เขียน 1 cache line ให้เสร็จก่อน
good_sequential_nt_write:
    ; เขียน cache line ที่ 1 ให้ครบ 64 bytes ก่อน
    mov     rax, rdi                ; dst1
    mov     rbx, rsi                ; dst2
    
    ; Write entire cache line to dst1 first
    vmovdqu     ymm0, [rdx]         ; source data 1
    vmovdqu     ymm1, [rdx + 32]    ; source data 2
    vmovntdq    [rax], ymm0          ; NT store 32 bytes
    vmovntdq    [rax + 32], ymm1     ; NT store 32 bytes → full line!
    
    ; Then write cache line to dst2
    vmovdqu     ymm2, [rdx + 64]
    vmovdqu     ymm3, [rdx + 96]
    vmovntdq    [rbx], ymm2
    vmovntdq    [rbx + 32], ymm3    ; full line!
    
    sfence
    ret
```

### 6.4 WCB Size และการ Manage

```
Intel Architecture:
- Haswell+: 12 WCB entries
- Skylake+: 12 WCB entries
- Ice Lake+: 12 WCB entries

AMD Architecture:
- Zen 3: 8 WCB entries
- Zen 4: 12 WCB entries

กฎในการใช้ WCB อย่างมีประสิทธิภาพ:
1. อย่าเปิด WCB entries มากกว่า hardware รองรับ
2. เขียน cache line ให้ครบ 64 bytes ก่อนย้ายไป address อื่น
3. ใช้ instruction ที่กว้างที่สุดเท่าที่ทำได้ (VMOVNTDQ 256/512-bit)
4. SFENCE หลังจาก NT stores เสร็จสิ้น
```

---

## 7. Non-Temporal Loads: MOVNTDQA

### 7.1 NT Loads คืออะไร

`MOVNTDQA` (SSE4.1) เป็น NT load instruction ที่ออกแบบมาสำหรับอ่านจาก WC (Write-Combining) memory type

```nasm
; NASM - MOVNTDQA syntax
; ต้องใช้กับ memory type WC (Write-Combining)

section .text
global nt_load_example

nt_load_example:
    ; MOVNTDQA ใช้ได้กับ WC memory เท่านั้น
    ; สำหรับ regular (WB) memory มันแค่ทำ regular load
    
    ; SSE4.1 version (128-bit)
    movntdqa    xmm0, [rdi]         ; NT load 128 bits
    movntdqa    xmm1, [rdi + 16]
    
    ; AVX2 version (256-bit) - ต้องการ VEX prefix
    vmovntdqa   ymm0, [rdi]         ; NT load 256 bits
    vmovntdqa   ymm1, [rdi + 32]
    
    ; AVX-512 version (512-bit)
    vmovntdqa   zmm0, [rdi]         ; NT load 512 bits
    
    ret
```

### 7.2 WC Memory Type และการใช้งาน

```
Memory Types ใน x86:
- WB (Write-Back):    default, fully cached
- WC (Write-Combining): NT loads/stores, GPU framebuffer
- WT (Write-Through): writes go to cache and memory
- WP (Write-Protected): reads cached, writes not cached
- UC (Uncacheable):   no caching at all

WC memory ใช้สำหรับ:
1. GPU framebuffer access
2. MMIO regions ที่ต้องการ streaming
3. PCIe device memory

วิธีตั้งค่า WC memory:
1. ผ่าน MTRR (Memory Type Range Registers)
2. ผ่าน PAT (Page Attribute Table) ใน page table entries
3. ผ่าน OS API (mmap with specific flags)
```

### 7.3 MOVNTDQA Performance

```
MOVNTDQA บน WC memory:
- ไม่ต้อง wait for WCB entries
- reads จาก WCB ถ้า data อยู่ใน WCB แล้ว (write combining)
- ประสิทธิภาพดีกว่า regular loads สำหรับ GPU/device access

MOVNTDQA บน WB memory (regular DRAM):
- แค่ทำ regular load (ไม่มีผลพิเศษ)
- อาจ hint hardware prefetcher ว่า data นี้ไม่ต้องเก็บใน cache นาน
```

---

## 8. HW Prefetcher กับ NT Instructions

### 8.1 Hardware Prefetcher ทำงานอย่างไร

```
Intel's Hardware Prefetchers (per core):
1. L1 Streamer: detect sequential accesses, prefetch ahead
2. L1 Stride Prefetcher: detect stride patterns
3. L2 Streamer: similar to L1 but for L2
4. L2 Adjacent Cache Line Prefetcher: prefetch neighboring cache lines

การ disable prefetcher:
- MSR 0x1A4: bit 0 = L2 HW Prefetcher
              bit 1 = L2 Adjacent Cache Line Prefetcher
              bit 2 = DCU Prefetcher
              bit 3 = DCU IP Prefetcher
```

### 8.2 NT Instructions และ Prefetcher Interaction

```
NT stores ปิด cache → hardware prefetcher ไม่รู้ว่า pattern คืออะไร
→ HW prefetcher อาจไม่ prefetch สำหรับ NT store regions

สำหรับ NT loads จาก WC memory:
→ HW prefetcher ก็ไม่ prefetch (WC memory bypass cache)

แนวทางปฏิบัติ:
1. สำหรับ streaming write (NT): ไม่ต้องกังวล prefetcher
   ข้อมูลผ่าน WCB โดยตรง

2. สำหรับ streaming read (ปกติ):
   ใช้ SW prefetch (PREFETCHNTA/PREFETCHT0) เพื่อช่วย prefetcher
```

### 8.3 Software Prefetch กับ NT Pattern

```nasm
; NASM - NT store ร่วมกับ SW prefetch สำหรับ read source
section .text
global nt_copy_with_prefetch

; คัดลอก large buffer ด้วย NT stores + SW prefetch
; รับ: rdi = dst, rsi = src, rdx = size in bytes
nt_copy_with_prefetch:
    push    rbx
    push    r12
    
    ; Prefetch distance: 8 cache lines = 512 bytes ahead
    %define PREFETCH_DIST   512
    
    ; Process 128 bytes (2 AVX-512 registers) per iteration
    mov     r12, rdx
    shr     r12, 7                  ; count = size / 128
    
    ; Initial prefetch
    prefetchnta [rsi + PREFETCH_DIST]
    prefetchnta [rsi + PREFETCH_DIST + 64]
    prefetchnta [rsi + PREFETCH_DIST + 128]
    prefetchnta [rsi + PREFETCH_DIST + 192]
    
.main_loop:
    test    r12, r12
    jz      .done
    
    ; Prefetch ahead
    prefetchnta [rsi + PREFETCH_DIST]
    prefetchnta [rsi + PREFETCH_DIST + 64]
    
    ; Load 2 x 64 bytes (temporal - goes to cache L1)
    vmovdqu64   zmm0, [rsi]
    vmovdqu64   zmm1, [rsi + 64]
    
    ; NT store 2 x 64 bytes (bypass cache)
    vmovntdq    [rdi], zmm0
    vmovntdq    [rdi + 64], zmm1
    
    add     rsi, 128
    add     rdi, 128
    dec     r12
    jnz     .main_loop
    
.done:
    sfence
    vzeroupper
    pop     r12
    pop     rbx
    ret
```

### 8.4 PREFETCHNTA vs PREFETCHT0/T1/T2

```nasm
; NASM - Prefetch hints

; PREFETCHNTA = Non-Temporal All levels
;   - hint: fetch into L1 only (no L2/L3 pollution)
;   - ใช้กับ data ที่อ่านครั้งเดียว (streaming)
prefetchnta [rsi + 512]

; PREFETCHT0 = Temporal hint level 0 (L1)
;   - hint: fetch into all cache levels
;   - ใช้กับ data ที่จะ reuse บ่อย
prefetcht0  [rsi + 512]

; PREFETCHT1 = Temporal hint level 1 (L2)
;   - hint: fetch into L2 and above
prefetcht1  [rsi + 1024]

; PREFETCHT2 = Temporal hint level 2 (L3)
;   - hint: fetch into L3
prefetcht2  [rsi + 2048]

; PREFETCHW = Write intent prefetch
;   - hint: fetch with write intent (get exclusive ownership)
;   - ลด RFO ภายหลัง
prefetchw   [rdi + 512]

; PREFETCHWT1 = Write temporal hint level 1
;   - รวม write intent กับ L2 fetch
prefetchwt1 [rdi + 512]
```

---

## 9. Cache Line Write-Back Instructions

### 9.1 CLWB (Cache Line Write-Back)

`CLWB` เป็น instruction (Intel Skylake SP+) ที่ flush cache line ไป memory โดย**ไม่ invalidate** cache line

```nasm
; NASM - CLWB usage
section .text
global clwb_example

; CLWB: flush dirty cache line to memory, keep in cache
; ใช้สำหรับ persistent memory (NVM/PMEM)
clwb_example:
    ; Write data
    mov     qword [rdi], rax
    mov     qword [rdi + 8], rbx
    
    ; Flush to persistent memory but keep in cache
    clwb    [rdi]               ; flush 64-byte cache line at rdi
    
    ; Memory fence to ensure write is durable
    mfence
    
    ret

; เปรียบเทียบ CLWB vs CLFLUSH vs CLFLUSHOPT:
; CLFLUSH:    invalidate + flush → ต้อง re-fetch ถ้าจะอ่านอีก
; CLFLUSHOPT: invalidate + flush (faster, weakly ordered)
; CLWB:       flush only (ไม่ invalidate) → data ยังอยู่ใน cache
```

### 9.2 CLFLUSHOPT

`CLFLUSHOPT` เป็นเวอร์ชัน optimized ของ `CLFLUSH` ที่รวดเร็วกว่าและ weakly ordered

```nasm
; NASM - CLFLUSHOPT vs CLFLUSH
section .text

; CLFLUSH (เก่า):
; - ต้องการ memory fence ก่อนและหลัง
; - Serializing: block pipeline
; - ทำ flush 1 cache line ต่อครั้ง
old_flush_style:
    mfence
    clflush [rdi]
    mfence
    ret

; CLFLUSHOPT (ใหม่ - Broadwell+):
; - Weakly ordered (ไม่ต้องการ fence ระหว่าง clflushopt)
; - เร็วกว่า clflush
; - ยังต้องการ fence ตอนท้าย
new_flush_style:
    clflushopt  [rdi]
    clflushopt  [rdi + 64]          ; สามารถ pipeline ได้
    clflushopt  [rdi + 128]
    clflushopt  [rdi + 192]
    sfence                          ; fence ตอนท้ายเพียงครั้งเดียว
    ret

; ตัวอย่าง flush persistent memory range
flush_pmem_range:
    ; rdi = start address, rsi = size in bytes
    push    rbx
    
    mov     rbx, rdi
    add     rbx, rsi                ; end = start + size
    
    ; Align start to cache line
    and     rdi, ~63                ; round down to 64-byte boundary
    
.flush_loop:
    cmp     rdi, rbx
    jge     .flush_done
    clflushopt  [rdi]
    add     rdi, 64
    jmp     .flush_loop
    
.flush_done:
    sfence                          ; ensure all flushes complete
    pop     rbx
    ret
```

### 9.3 WBNOINVD

`WBNOINVD` (Write-Back No Invalidate) เป็น instruction ที่ flush ทั้ง cache (write back dirty lines) โดยไม่ invalidate

```nasm
; NASM - WBNOINVD
; ต้องการ privilege level (ปกติใช้ใน kernel/hypervisor)
; หรือ CPU ที่รองรับ (CPUID.80000008H:EBX[bit 9])

; INVD: invalidate all caches (ไม่ write back!) - อันตราย!
; WBINVD: write back และ invalidate ทั้งหมด - ช้า
; WBNOINVD: write back ทั้งหมด ไม่ invalidate - Icelake+

; ใช้งาน (kernel mode):
kernel_wbnoinvd:
    wbnoinvd                        ; flush all dirty cache lines
    ; ข้อมูลยังอยู่ใน cache แต่ persistent memory ได้รับข้อมูลแล้ว
    ret
```

---

## 10. Huge Pages

### 10.1 TLB Miss Cost และ Huge Pages

TLB (Translation Lookaside Buffer) cache การ translate virtual → physical address ถ้า TLB miss ต้องทำ page walk (หลาย memory accesses)

```
ขนาด Page ต่างๆ:
- Regular page: 4KB
- Large page:   2MB (PMD level)
- Huge page:    1GB (PUD level)

TLB entries ต่อ core (Skylake):
- L1 ITLB: 128 entries (4KB pages)
- L1 DTLB: 64 entries (4KB pages), 32 entries (2MB pages)
- L2 TLB:  1536 entries (shared)

Coverage:
- 64 x 4KB  = 256 KB (L1 DTLB 4KB)
- 32 x 2MB  = 64 MB  (L1 DTLB 2MB) ← 256x coverage improvement!

TLB Miss Cost:
- L2 TLB hit: ~7-10 cycles
- Page walk:  ~40-100 cycles (4 memory accesses)
- Page walk ใน cold cache: ~200-400 cycles!
```

### 10.2 Transparent Huge Pages (THP)

```bash
# ตรวจสอบ THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never  ← [] บอก current setting

# Enable THP
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# ใช้ madvise กับ specific region
# madvise(ptr, size, MADV_HUGEPAGE)
```

### 10.3 MAP_HUGETLB ใน Assembly

```nasm
; NASM - mmap ด้วย huge pages
; Linux syscall numbers สำหรับ x86-64

section .data
PROT_READ  equ 1
PROT_WRITE equ 2
MAP_PRIVATE equ 2
MAP_ANONYMOUS equ 0x20
MAP_HUGETLB  equ 0x40000          ; request huge pages
MAP_HUGE_2MB equ (21 << 26)       ; 2MB huge pages
MAP_HUGE_1GB equ (30 << 26)       ; 1GB huge pages

section .text
global alloc_huge_pages

; Allocate 2MB huge page using mmap
; รับ: rdi = size (should be multiple of 2MB)
; คืน: rax = pointer หรือ -errno
alloc_huge_pages:
    ; mmap(NULL, size, PROT_READ|PROT_WRITE, 
    ;       MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB|MAP_HUGE_2MB, -1, 0)
    mov     rax, 9                  ; syscall: mmap
    xor     rdi, rdi                ; addr = NULL
    mov     rsi, rdi                ; size (from first arg)
    ; rdi is overwritten - let's use proper calling convention
    
    push    rdi                     ; save size
    
    xor     rdi, rdi                ; addr = NULL
    pop     rsi                     ; size
    mov     rdx, PROT_READ | PROT_WRITE
    mov     r10, MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_HUGE_2MB
    mov     r8, -1                  ; fd = -1
    xor     r9, r9                  ; offset = 0
    
    syscall
    ; rax = allocated pointer or -errno
    ret

; ตัวอย่างใช้งาน huge pages กับ array:
; 1. alloc_huge_pages(2MB) ← get 2MB aligned buffer
; 2. ใช้ buffer ปกติ แต่ TLB miss น้อยกว่ามาก
; 3. munmap เมื่อเสร็จ
```

### 10.4 มาตรวัดผลของ Huge Pages

```nasm
; NASM - เปรียบเทียบ bandwidth ด้วยและไม่มี huge pages
section .text
global measure_bandwidth

; Sequential read ด้วย 4KB pages → TLB misses บ่อย
; Sequential read ด้วย 2MB pages → TLB misses น้อยมาก

; เมื่อ working set > L3 cache:
; 4KB pages: ต้องทำ page walk ทุก 4KB = 6.25% ของ accesses เป็น page walk
; 2MB pages: page walk ทุก 2MB = 0.003% ของ accesses เป็น page walk
; → ประสิทธิภาพเพิ่มขึ้น 10-30% สำหรับ large memory access patterns
```

---

## 11. NUMA: Non-Uniform Memory Access

### 11.1 NUMA Architecture

ในระบบ multi-socket, แต่ละ CPU socket มี local memory ของตัวเอง การเข้าถึง memory ของ socket อื่น (remote memory) ช้ากว่า local memory

```
NUMA Topology ตัวอย่าง 2-socket system:

Socket 0 (NUMA Node 0):        Socket 1 (NUMA Node 1):
+------------------+           +------------------+
| CPU Core 0-7     |           | CPU Core 8-15    |
| L1, L2, L3 Cache |           | L1, L2, L3 Cache |
+------------------+           +------------------+
| Memory: 32GB     |           | Memory: 32GB     |
| (Local)          |           | (Local)          |
+------------------+           +------------------+
           |                              |
           +----------QPI/UPI/IF----------+
                     (interconnect)

Local access:  ~70 ns (NUMA ratio 1.0x)
Remote access: ~120 ns (NUMA ratio 1.7x)
```

### 11.2 Local vs Remote Memory Bandwidth

```
Bandwidth measurements (example Xeon system):
                     Read BW    Write BW
Local memory:        45 GB/s    35 GB/s
Remote memory:       20 GB/s    15 GB/s    ← 2-3x slower!

NUMA ratios vary by platform:
- AMD EPYC: ~1.2x (INFINITY FABRIC ดีมาก)
- Intel Xeon: ~1.5-2x
- ARM Neoverse: ~1.3x
```

### 11.3 NUMA-Aware Memory Access

```bash
# ตรวจสอบ NUMA topology
numactl --hardware
# หรือ
lscpu | grep -i numa

# Run process on specific NUMA node
numactl --cpunodebind=0 --membind=0 ./program

# Interleave memory across nodes (ดีสำหรับ multi-threaded)
numactl --interleave=all ./program
```

```nasm
; NASM - NUMA-aware allocation ผ่าน Linux syscall
; mbind(addr, len, MPOL_BIND, nodemask, maxnode, flags)

section .data
MPOL_DEFAULT    equ 0
MPOL_BIND       equ 2
MPOL_INTERLEAVE equ 3
MPOL_LOCAL      equ 4

section .text
global bind_memory_to_node

; Bind memory region to specific NUMA node
; รับ: rdi = addr, rsi = len, rdx = node_id
bind_memory_to_node:
    push    rbx
    push    r12
    push    r13
    
    mov     r12, rdi                ; save addr
    mov     r13, rsi                ; save len
    
    ; Build nodemask on stack
    push    0
    mov     rax, 1
    shl     rax, rdx                ; nodemask bit = 1 << node_id
    push    rax
    
    ; mbind syscall (235)
    mov     rax, 235                ; sys_mbind
    mov     rdi, r12                ; addr
    mov     rsi, r13                ; len
    mov     rdx, MPOL_BIND          ; mode
    lea     r10, [rsp]              ; nodemask ptr
    mov     r8, 64                  ; maxnode
    xor     r9, r9                  ; flags
    
    syscall
    
    add     rsp, 16                 ; clean stack
    pop     r13
    pop     r12
    pop     rbx
    ret
```

### 11.4 False NUMA Sharing

```nasm
; NASM - ตัวอย่าง false NUMA sharing ปัญหา

; BAD: interleaved struct layout, threads on different NUMA nodes
; Thread 0 (NUMA 0) และ Thread 1 (NUMA 1) share same cache line
; → Remote NUMA access ทุกครั้งที่ access struct!

struc BadStruct
    .val0   resd 1     ; Thread 0 ใช้
    .val1   resd 1     ; Thread 1 ใช้
    ; ทั้งสองอยู่ใน cache line เดียวกัน!
endstruc

; GOOD: pad struct เพื่อ separate NUMA domains
struc GoodStruct
    .val0   resd 1     ; Thread 0 ใช้
    .pad0   resb 60    ; Padding to cache line boundary
    .val1   resd 1     ; Thread 1 ใช้ (separate cache line)
    .pad1   resb 60    ; Padding
endstruc
```

---

## 12. Memory Interleaving Across Channels

### 12.1 Multi-Channel Memory Interleaving

```
Single Channel:
[Channel 0: 8-byte interleave]
0x000-0x007: CH0
0x008-0x00F: CH0
0x010-0x017: CH0
...

Dual Channel (interleaved at cache line level):
[Channel 0: 64 bytes] [Channel 1: 64 bytes] [Channel 0: 64 bytes] ...
0x000-0x03F: CH0
0x040-0x07F: CH1
0x080-0x0BF: CH0
0x0C0-0x0FF: CH1

Quad Channel interleaving เพิ่ม bandwidth 4x:
0x000-0x03F: CH0
0x040-0x07F: CH1
0x080-0x0BF: CH2
0x0C0-0x0FF: CH3
0x100-0x13F: CH0 (repeat)
```

### 12.2 ผลของ Interleaving ต่อ Bandwidth

```
Memory Channel Configuration:
- Single CH DDR4-3200:   25.6 GB/s theoretical
- Dual CH DDR4-3200:     51.2 GB/s theoretical
- Quad CH DDR4-3200:    102.4 GB/s theoretical
- 8-CH DDR4-3200 (server): 204.8 GB/s theoretical

STREAM Triad ผลลัพธ์:
- 1-CH:  22 GB/s
- 2-CH:  44 GB/s
- 4-CH:  85 GB/s (ไม่ scale perfect เพราะ other bottlenecks)
- 8-CH: 160 GB/s
```

### 12.3 การใช้ประโยชน์จาก Channel Interleaving

```nasm
; NASM - Pattern ที่ใช้ประโยชน์จาก channel interleaving

; Sequential access ดีที่สุดสำหรับ channel interleaving
; เพราะ interleaving ทำที่ cache-line (64-byte) boundary
; Sequential access → ใช้ทุก channel อย่างเท่าเทียม

section .text
global optimal_channel_access

; รับ: rdi = data, rsi = count (64-byte blocks)
optimal_channel_access:
    test    rsi, rsi
    jz      .done
    
    ; Sequential access → best channel interleaving
.loop:
    ; ใน loop นี้ hardware เห็น sequential address pattern
    ; → memory controller ใช้ทุก channel พร้อมกัน
    vmovdqu64   zmm0, [rdi]
    vmovdqu64   zmm1, [rdi + 64]
    vmovdqu64   zmm2, [rdi + 128]
    vmovdqu64   zmm3, [rdi + 192]
    
    ; Process data...
    vpaddd  zmm0, zmm0, zmm1
    vpaddd  zmm2, zmm2, zmm3
    
    add     rdi, 256
    sub     rsi, 4                  ; 4 blocks per iteration
    jnz     .loop
    
.done:
    vzeroupper
    ret
```

---

## 13. Bandwidth-Bound vs Compute-Bound Analysis

### 13.1 Roofline Model

Roofline Model เป็นวิธีวิเคราะห์ว่า algorithm ถูก limit โดย bandwidth หรือ compute

```
Roofline Model:
                    Peak FLOPS (compute bound)
Performance ───────────────────────────────┐
  (GFLOPS/s) │              /              |
             │            /                |
             │          /    (bandwidth    |
             │        /       bound)       |
             │      /                      |
             └────────────────────────────────────
                         Arithmetic Intensity
                         (FLOPS / byte)

Arithmetic Intensity (AI) = FLOPS / Bytes_transferred

เส้น bandwidth limit = Peak_BW × AI
เส้น compute limit  = Peak_FLOPS (horizontal line)

ถ้า AI ต่ำ (เช่น 0.5 FLOPS/byte): bandwidth-bound
ถ้า AI สูง (เช่น 10 FLOPS/byte): compute-bound
```

### 13.2 การคำนวณ Arithmetic Intensity

```
ตัวอย่าง DAXPY: y[i] = a*x[i] + y[i]
- FLOPS: 2 (1 multiply + 1 add)
- Bytes: read x[i] (8 bytes) + read y[i] (8 bytes) + write y[i] (8 bytes) = 24 bytes
- AI = 2/24 ≈ 0.083 FLOPS/byte → Bandwidth-bound!

ตัวอย่าง Matrix multiply C = A*B (N×N matrices):
- FLOPS: 2N³
- Bytes: N² × 3 × 8 bytes (read A, B, write C)
- AI = 2N³ / (24N²) = N/12 FLOPS/byte
- สำหรับ N=1000: AI ≈ 83 FLOPS/byte → Compute-bound!

ตัวอย่าง SpMV (Sparse Matrix-Vector Multiply):
- FLOPS: 2 × nnz
- Bytes: 12 × nnz (row indices, col indices, values, x, y)  
- AI ≈ 0.167 FLOPS/byte → Bandwidth-bound!
```

### 13.3 ตัวอย่างในรูปแบบ Assembly

```nasm
; NASM - DAXPY: Bandwidth-bound operation
; y[i] = a * x[i] + y[i]

section .text
global daxpy_bandwidth_bound

; รับ: rdi = y (double*), rsi = x (double*), xmm0 = a, rdx = n
daxpy_bandwidth_bound:
    push    rbx
    
    ; Broadcast scalar 'a' to all lanes
    vbroadcastsd    ymm15, xmm0
    
    ; Process 4 doubles per iteration (256-bit)
    mov     rbx, rdx
    shr     rbx, 2                  ; count/4
    
.loop_avx:
    test    rbx, rbx
    jz      .remainder
    
    ; Load x[i..i+3] and y[i..i+3]
    vmovupd ymm0, [rsi]             ; x[i:i+4]
    vmovupd ymm1, [rdi]             ; y[i:i+4]
    
    ; y[i] = a * x[i] + y[i]  (FMA)
    vfmadd231pd ymm1, ymm15, ymm0
    
    ; Store y back
    vmovupd [rdi], ymm1
    
    add     rsi, 32
    add     rdi, 32
    dec     rbx
    jnz     .loop_avx
    
.remainder:
    ; Handle remaining elements
    mov     rcx, rdx
    and     rcx, 3
    
.loop_scalar:
    test    rcx, rcx
    jz      .done
    
    vmovsd  xmm1, [rsi]             ; x[i]
    vmovsd  xmm2, [rdi]             ; y[i]
    vfmadd231sd xmm2, xmm15, xmm1  ; y = a*x + y
    vmovsd  [rdi], xmm2
    
    add     rsi, 8
    add     rdi, 8
    dec     rcx
    jnz     .loop_scalar
    
.done:
    vzeroupper
    pop     rbx
    ret
```

---

## 14. DAXPY Benchmark

### 14.1 DAXPY เป็น Memory Bandwidth Benchmark

DAXPY (Double-precision A*X + Y) เป็น BLAS Level 1 operation ที่ใช้วัด memory bandwidth เพราะมี arithmetic intensity ต่ำมาก

```nasm
; NASM - Complete DAXPY benchmark
section .data
    align 8
a_scalar    dq  2.5                 ; scalar multiplier
N           equ 33554432            ; 32M elements = 256 MB

section .bss
    align 64
x_array     resq N
y_array     resq N

section .text
global daxpy_benchmark

; High-performance DAXPY ด้วย AVX-512 + NT stores
; รับ: rdi = y, rsi = x, xmm0 = a, rdx = n
daxpy_avx512_nt:
    push    rbx
    push    r12
    
    ; Broadcast scalar
    vbroadcastsd    zmm31, xmm0
    
    ; Main loop: 8 doubles per iteration (512-bit)
    mov     r12, rdx
    shr     r12, 3                  ; n/8 iterations
    
.main_loop:
    test    r12, r12
    jz      .done
    
    ; Load x and y (prefetch ahead)
    prefetchnta [rsi + 512]
    prefetchnta [rdi + 512]
    
    vmovupd     zmm0, [rsi]         ; load x[i:i+8]
    vmovupd     zmm1, [rdi]         ; load y[i:i+8]
    
    ; FMA: y = a*x + y
    vfmadd231pd zmm1, zmm31, zmm0
    
    ; NT store result (streaming write)
    vmovntpd    [rdi], zmm1
    
    add     rsi, 64
    add     rdi, 64
    dec     r12
    jnz     .main_loop
    
.done:
    sfence                          ; ensure NT stores complete
    vzeroupper
    pop     r12
    pop     rbx
    ret
```

### 14.2 Performance Analysis ของ DAXPY

```
DAXPY Analysis:
  Elements: 32M doubles = 256 MB per array
  Arrays: x (read), y (read+write)
  
  Bytes transferred per iteration:
  - Read x:  8 bytes
  - Read y:  8 bytes  
  - Write y: 8 bytes
  Total: 24 bytes per element
  
  Total data movement:
  32M × 24 bytes = 768 MB per call
  
  สมมติ DRAM bandwidth = 40 GB/s:
  Minimum time = 768 MB / 40 GB/s ≈ 19.2 ms
  
  ถ้าใช้ NT stores (ไม่ต้อง read y ก่อน write):
  Data movement = 32M × (8 + 8) = 512 MB  ← save 256 MB!
  Minimum time = 512 MB / 40 GB/s ≈ 12.8 ms
  
  Speedup จาก NT stores: 19.2 / 12.8 = 1.5x !!!
```

### 14.3 DAXPY Bandwidth Measurement

```nasm
; NASM - Measure DAXPY bandwidth ด้วย RDTSC
section .text
global measure_daxpy_bandwidth

; ตัวอย่างการ measure (pseudo-code in comments)
; 1. warm up cache ก่อน (optional - depends on what you want to measure)
; 2. RDTSC start
; 3. run DAXPY
; 4. RDTSC end
; 5. calculate bandwidth

measure_daxpy_bandwidth:
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; Get start time
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    mov     r14, rax                ; start_tsc
    
    ; Run DAXPY (assume rdi=y, rsi=x, xmm0=a, rcx=n)
    ; (call daxpy here)
    
    ; Get end time
    rdtscp
    shl     rdx, 32
    or      rax, rdx
    mov     r15, rax                ; end_tsc
    
    ; cycles = end - start
    sub     r15, r14
    mov     rax, r15                ; return cycles
    
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    ret
```

---

## 15. Memory Access Patterns และ Bandwidth Efficiency

### 15.1 Sequential vs Strided vs Random Access

```
Bandwidth Efficiency (approximate, vs peak):

Sequential  100%   ████████████████████
Stride-2     95%   ███████████████████
Stride-4     80%   ████████████████
Stride-8     60%   ████████████
Stride-16    40%   ████████
Stride-32    25%   █████
Random        5%   █

เหตุผล:
- Sequential: hardware prefetcher ทำงานได้ดี, full cache line utilization
- Strided:    prefetcher อาจ miss, cache lines ถูก evict ก่อนใช้หมด
- Random:     ทุก access เป็น cache miss, พลาดทั้ง cache line (use 8/64 bytes)
```

### 15.2 การปรับปรุง Access Pattern

```nasm
; NASM - เปลี่ยน strided access เป็น sequential

; BAD: Strided access (AoS - Array of Structures)
struc Particle
    .x  resd 1      ; float x
    .y  resd 1      ; float y
    .z  resd 1      ; float z
    .w  resd 1      ; float mass (padding)
endstruc
; Access pattern: p[0].x, p[1].x, p[2].x, ... → stride 16!

; GOOD: Sequential access (SoA - Structure of Arrays)
; float x[N], y[N], z[N], w[N]
; Access pattern: x[0], x[1], x[2], ... → sequential!

; ตัวอย่าง update_positions แบบ SoA
; รับ: rdi=x, rsi=y, rdx=z, rcx=vx, r8=vy, r9=vz, dt in xmm0, n in stack
update_positions_soa:
    push    rbx
    push    r12
    
    mov     rbx, [rsp + 24]         ; n from stack (after pushes)
    
    ; Broadcast dt to all lanes
    vbroadcastss ymm15, xmm0
    
    mov     r12, rbx
    shr     r12, 3                  ; n/8 (AVX2 = 8 floats)
    
.loop:
    test    r12, r12
    jz      .done
    
    ; Load positions and velocities (all sequential!)
    vmovups ymm0, [rdi]             ; x[i:i+8]
    vmovups ymm1, [rsi]             ; y[i:i+8]
    vmovups ymm2, [rdx]             ; z[i:i+8]
    vmovups ymm3, [rcx]             ; vx[i:i+8]
    vmovups ymm4, [r8]              ; vy[i:i+8]
    vmovups ymm5, [r9]              ; vz[i:i+8]
    
    ; x += vx * dt
    vfmadd231ps ymm0, ymm3, ymm15
    vfmadd231ps ymm1, ymm4, ymm15
    vfmadd231ps ymm2, ymm5, ymm15
    
    ; Store updated positions
    vmovups [rdi], ymm0
    vmovups [rsi], ymm1
    vmovups [rdx], ymm2
    
    add     rdi, 32
    add     rsi, 32
    add     rdx, 32
    add     rcx, 32
    add     r8,  32
    add     r9,  32
    dec     r12
    jnz     .loop
    
.done:
    vzeroupper
    pop     r12
    pop     rbx
    ret
```

---

## 16. Advanced Memory Optimization Techniques

### 16.1 Loop Tiling (Blocking) สำหรับ Cache

```nasm
; NASM - Loop tiling for matrix transpose
; Untiled: poor cache usage, lots of cache misses
; Tiled: keeps data in cache, much better bandwidth utilization

section .text
global matrix_transpose_tiled

; Tiled matrix transpose
; รับ: rdi = dst (N×N), rsi = src (N×N), rdx = N, rcx = tile_size
matrix_transpose_tiled:
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    push    rbp
    
    mov     rbp, rdx                ; N
    
    ; for (int ii = 0; ii < N; ii += tile)
    xor     r12, r12                ; ii = 0
    
.outer_loop_ii:
    cmp     r12, rbp
    jge     .done
    
    ; for (int jj = 0; jj < N; jj += tile)
    xor     r13, r13                ; jj = 0
    
.outer_loop_jj:
    cmp     r13, rbp
    jge     .next_ii
    
    ; tile boundaries
    mov     r14, r12
    add     r14, rcx
    cmp     r14, rbp
    cmovg   r14, rbp                ; min(ii+tile, N)
    
    mov     r15, r13
    add     r15, rcx
    cmp     r15, rbp
    cmovg   r15, rbp                ; min(jj+tile, N)
    
    ; Inner loops for this tile
    mov     rbx, r12                ; i = ii
    
.inner_loop_i:
    cmp     rbx, r14
    jge     .next_jj
    
    mov     rax, r13                ; j = jj
    
.inner_loop_j:
    cmp     rax, r15
    jge     .next_i
    
    ; dst[j][i] = src[i][j]
    ; src offset: (i * N + j) * 4 bytes
    ; dst offset: (j * N + i) * 4 bytes
    
    push    rax
    push    rbx
    
    imul    rbx, rbp                ; i * N
    add     rbx, rax                ; i * N + j
    mov     edx, [rsi + rbx*4]     ; src[i][j]
    
    imul    rax, rbp                ; j * N
    add     rax, [rsp]              ; j * N + i (original rbx)
    mov     [rdi + rax*4], edx     ; dst[j][i]
    
    pop     rbx
    pop     rax
    
    inc     rax                     ; j++
    jmp     .inner_loop_j
    
.next_i:
    inc     rbx                     ; i++
    jmp     .inner_loop_i
    
.next_jj:
    add     r13, rcx               ; jj += tile
    jmp     .outer_loop_jj
    
.next_ii:
    add     r12, rcx               ; ii += tile
    jmp     .outer_loop_ii
    
.done:
    pop     rbp
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    ret
```

### 16.2 Memory Bandwidth Saturation Test

```nasm
; NASM - ทดสอบ memory bandwidth saturation ด้วย parallel threads
; (conceptual - ใช้กับ OpenMP หรือ pthreads ในภาษาสูงกว่า)

section .text
global parallel_stream_triad

; STREAM Triad: a[i] = b[i] + scalar * c[i]
; รับ: rdi=a, rsi=b, rdx=c, xmm0=scalar, rcx=n
parallel_stream_triad:
    push    rbx
    
    vbroadcastsd    zmm31, xmm0     ; broadcast scalar
    
    mov     rbx, rcx
    shr     rbx, 3                  ; n/8 (8 doubles per AVX-512)
    
.loop:
    test    rbx, rbx
    jz      .done
    
    ; Load b and c
    vmovupd zmm0, [rsi]             ; b[i:i+8]
    vmovupd zmm1, [rdx]             ; c[i:i+8]
    
    ; a = b + scalar * c
    vfmadd213pd zmm0, zmm31, zmm1  ; b * scalar + c... wait, wrong
    ; correct: a = b + scalar*c
    vmulpd      zmm2, zmm31, zmm1  ; scalar * c
    vaddpd      zmm0, zmm0, zmm2   ; b + scalar*c
    
    ; NT store result
    vmovntpd [rdi], zmm0
    
    add     rdi, 64
    add     rsi, 64
    add     rdx, 64
    dec     rbx
    jnz     .loop
    
.done:
    sfence
    vzeroupper
    pop     rbx
    ret
```

---

## 17. Performance Counters สำหรับ Memory Bandwidth

### 17.1 PMU Events ที่สำคัญ

```
Intel Performance Counter Events สำหรับ memory:

Event Name                    Counter     Description
MEM_LOAD_RETIRED.L1_MISS     0x01D1      L1 cache miss
MEM_LOAD_RETIRED.L2_MISS     0x02D1      L2 cache miss
MEM_LOAD_RETIRED.L3_MISS     0x20D1      L3 cache miss (→DRAM)
L2_LINES_IN.ALL              0xF724      L2 cache line fills
LLC_MISSES                   0x412E      LLC misses
OFFCORE_REQUESTS.ALL_DATA_RD 0x80B0      All data reads
MEM_BANDWIDTH_READS          uncore      Read bandwidth
MEM_BANDWIDTH_WRITES         uncore      Write bandwidth
```

### 17.2 ใช้ perf สำหรับ Measurement

```bash
# วัด cache misses และ memory bandwidth
perf stat -e \
  cache-references,cache-misses,\
  mem-loads,mem-stores,\
  r04D1,r08D1,r20D1 \
  ./program

# วัด uncore memory bandwidth (Intel)
perf stat -e \
  uncore_imc/data_reads/,\
  uncore_imc/data_writes/ \
  ./program

# ใช้ perf record + report สำหรับ hotspot analysis
perf record -e mem-loads:u,mem-stores:u -c 1000 ./program
perf report --sort=sym,dso

# NUMA statistics
numastat -p ./program
```

### 17.3 การวิเคราะห์ผลลัพธ์

```
ตัวอย่างผลจาก perf stat:

Performance counter stats for './daxpy_benchmark':

     768,000,000      mem-loads:u               #  memory loads
     256,000,000      mem-stores:u              #  memory stores
      15,360,000      cache-misses              #    2.00% cache miss rate
       4,096,000      r20D1 (L3 misses)         #    0.53% L3 miss rate

       0.019234 seconds time elapsed

Bandwidth calculation:
  L3 misses × 64 bytes = 4M × 64 = 256 MB read from DRAM
  Stores (NT) = 256M × 8 bytes = 2 GB written to DRAM
  
  Wait... something is off. Let me recalculate:
  
  Total DRAM bytes = (reads_from_DRAM + writes_to_DRAM)
                   = (256M doubles × 8 + 128M doubles × 8)
                   = 3072 MB
  
  Bandwidth = 3072 MB / 0.019 s ≈ 161 GB/s ← using all channels!
```

---

## 18. Case Study: Optimizing a Memory-Bound Kernel

### 18.1 Problem: Image Convolution

```nasm
; NASM - Optimizing a 2D convolution (memory-bound for large kernels)

section .text

; SLOW version: poor memory access pattern
convolve_naive:
    ; for each output pixel (y, x):
    ;   sum = 0
    ;   for each kernel element (ky, kx):
    ;     sum += image[y+ky][x+kx] * kernel[ky][kx]
    ;   output[y][x] = sum
    ; 
    ; Problem: image access pattern = stride W for each ky increment
    ; → nhiều cache misses khi kernel lớn!
    ret

; FAST version: tiled for cache efficiency
convolve_tiled:
    ; Tile the computation so that image data stays in L1/L2 cache
    ; Process OUTPUT_TILE × OUTPUT_TILE output pixels at once
    ; → image data accessed multiple times while in cache
    ret

; FASTEST version: separate kernel into 1D filters (if separable)
convolve_separable:
    ; For Gaussian blur: G(x,y) = G(x) × G(y)
    ; Apply horizontal filter first, then vertical
    ; → each pass is sequential → maximum bandwidth utilization
    ret
```

### 18.2 Optimization Checklist สรุป

```
Memory Bandwidth Optimization Checklist:

1. Measure first:
   □ ใช้ STREAM benchmark วัด achievable bandwidth
   □ ใช้ perf stat วัด cache miss rates
   □ คำนวณ arithmetic intensity ของ algorithm

2. Access pattern:
   □ Sequential > Strided > Random
   □ ใช้ SoA แทน AoS ถ้าต้องการ sequential access
   □ Loop tiling เพื่อ cache reuse

3. NUMA:
   □ Pin threads ไว้ที่ NUMA node เดียวกับ memory
   □ ใช้ numactl --interleave สำหรับ multi-threaded
   □ หลีกเลี่ยง false sharing ระหว่าง NUMA nodes

4. NT stores:
   □ ใช้ MOVNTDQ/VMOVNTDQ สำหรับ streaming writes
   □ เพิ่ม SFENCE หลัง NT stores
   □ เขียน full cache lines เพื่อ efficient WCB use

5. Huge pages:
   □ ใช้ 2MB THP สำหรับ large arrays
   □ MAP_HUGETLB สำหรับ explicit huge pages
   □ madvise(MADV_HUGEPAGE) สำหรับ specific regions

6. Prefetching:
   □ ใช้ PREFETCHNTA สำหรับ streaming (no cache pollution)
   □ ใช้ PREFETCHT0 สำหรับ data ที่จะ reuse
   □ Prefetch distance = latency × bandwidth / data_per_iteration

7. Write patterns:
   □ ลด RFO: ใช้ NT stores หรือ prefetchw ล่วงหน้า
   □ Write full cache lines เมื่อทำได้
   □ Avoid partial writes ไป random locations
```

---

## 19. ตัวอย่างโปรแกรมสมบูรณ์: Memory Bandwidth Tester

### 19.1 Complete Bandwidth Test Program (NASM)

```nasm
; NASM - Complete memory bandwidth test program
; Compile: nasm -f elf64 bw_test.asm -o bw_test.o
; Link:    gcc bw_test.o -o bw_test -lm

section .data
    align 8
    msg_header      db  "Memory Bandwidth Test", 10, 0
    msg_read        db  "Read BW:  %8.2f GB/s", 10, 0
    msg_write       db  "Write BW: %8.2f GB/s", 10, 0
    msg_copy        db  "Copy BW:  %8.2f GB/s", 10, 0
    fmt_float       db  "%.2f", 10, 0
    
    ; Test parameters
    BUFFER_SIZE     equ (256 * 1024 * 1024)    ; 256 MB
    ELEMENT_SIZE    equ 8                        ; double (8 bytes)
    N_ELEMENTS      equ (BUFFER_SIZE / ELEMENT_SIZE) ; 32M elements

section .bss
    align 64
    buf_a   resb    BUFFER_SIZE
    buf_b   resb    BUFFER_SIZE

section .text
    extern printf
    global main

; ────────────────────────────────────────────────
; Read bandwidth test: read entire buffer
; รับ: rdi = buffer, rsi = n_elements
; คืน: rax = cycles taken
; ────────────────────────────────────────────────
test_read_bandwidth:
    push    rbx
    push    r12
    
    mov     rbx, rdi                ; buffer
    mov     r12, rsi                ; n_elements
    shr     r12, 3                  ; /8 for AVX-512
    
    ; Warmup + serialization
    cpuid
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    push    rax                     ; save start TSC
    
.read_loop:
    test    r12, r12
    jz      .read_done
    
    ; Non-temporal read (streaming)
    vmovntdqa   zmm0, [rbx]
    vmovntdqa   zmm1, [rbx + 64]
    vmovntdqa   zmm2, [rbx + 128]
    vmovntdqa   zmm3, [rbx + 192]
    
    ; Use data to prevent optimization
    vpxord  zmm4, zmm0, zmm1
    vpxord  zmm5, zmm2, zmm3
    vpxord  zmm4, zmm4, zmm5
    
    add     rbx, 256
    sub     r12, 4
    jnz     .read_loop
    
.read_done:
    rdtscp
    shl     rdx, 32
    or      rax, rdx
    pop     rbx                     ; restore start TSC
    sub     rax, rbx                ; cycles = end - start
    
    pop     r12
    pop     rbx
    ret

; ────────────────────────────────────────────────
; Write bandwidth test: write entire buffer (NT)
; รับ: rdi = buffer, rsi = n_elements
; คืน: rax = cycles taken
; ────────────────────────────────────────────────
test_write_bandwidth:
    push    rbx
    push    r12
    
    mov     rbx, rdi
    mov     r12, rsi
    shr     r12, 8                  ; /8 for 64-byte writes
    
    vpxord  zmm0, zmm0, zmm0        ; zero register for writing
    
    cpuid
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    push    rax
    
.write_loop:
    test    r12, r12
    jz      .write_done
    
    ; NT store - bypass cache, write directly to DRAM
    vmovntdq    [rbx],       zmm0
    vmovntdq    [rbx + 64],  zmm0
    vmovntdq    [rbx + 128], zmm0
    vmovntdq    [rbx + 192], zmm0
    
    add     rbx, 256
    dec     r12
    jnz     .write_loop
    
.write_done:
    sfence                          ; ensure all NT stores complete
    
    rdtscp
    shl     rdx, 32
    or      rax, rdx
    pop     rbx
    sub     rax, rbx
    
    pop     r12
    pop     rbx
    ret

; ────────────────────────────────────────────────
; Copy bandwidth test: NT copy (read src, NT write dst)
; รับ: rdi = dst, rsi = src, rdx = n_elements  
; คืน: rax = cycles taken
; ────────────────────────────────────────────────
test_copy_bandwidth:
    push    rbx
    push    r12
    push    r13
    
    mov     rbx, rdi                ; dst
    mov     r12, rsi                ; src
    mov     r13, rdx
    shr     r13, 3                  ; /8 for 64-byte ops
    
    cpuid
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    push    rax
    
.copy_loop:
    test    r13, r13
    jz      .copy_done
    
    ; Load source (with prefetch)
    prefetchnta [r12 + 512]
    
    vmovdqu64   zmm0, [r12]         ; temporal load
    vmovntdq    [rbx], zmm0         ; NT store
    
    add     r12, 64
    add     rbx, 64
    dec     r13
    jnz     .copy_loop
    
.copy_done:
    sfence
    
    rdtscp
    shl     rdx, 32
    or      rax, rdx
    pop     rbx
    sub     rax, rbx
    
    pop     r13
    pop     r12
    pop     rbx
    ret

; ────────────────────────────────────────────────
; Main function
; ────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32
    push    rbx
    push    r12
    push    r13
    
    ; Print header
    lea     rdi, [rel msg_header]
    xor     eax, eax
    call    printf
    
    ; Test read bandwidth
    lea     rdi, [rel buf_a]
    mov     rsi, N_ELEMENTS
    call    test_read_bandwidth
    ; rax = cycles
    
    ; Calculate GB/s: (BUFFER_SIZE / 1e9) / (cycles / clock_freq)
    ; GB/s = BUFFER_SIZE * clock_freq / (cycles * 1e9)
    ; Simplified: assume 3.0 GHz clock
    ; GB/s ≈ (256 * 3000) / cycles [MB/s → GB/s with factor]
    
    ; Print result
    mov     rbx, rax                ; save cycles
    lea     rdi, [rel msg_read]
    ; (calculate and print bandwidth - simplified)
    xor     eax, eax
    call    printf
    
    ; Test write bandwidth
    lea     rdi, [rel buf_a]
    mov     rsi, N_ELEMENTS
    call    test_write_bandwidth
    
    lea     rdi, [rel msg_write]
    xor     eax, eax
    call    printf
    
    ; Test copy bandwidth
    lea     rdi, [rel buf_b]
    lea     rsi, [rel buf_a]
    mov     rdx, N_ELEMENTS
    call    test_copy_bandwidth
    
    lea     rdi, [rel msg_copy]
    xor     eax, eax
    call    printf
    
    xor     eax, eax
    pop     r13
    pop     r12
    pop     rbx
    leave
    ret
```

---

## 20. สรุปและแนวทางปฏิบัติ

### 20.1 Key Takeaways

```
1. DRAM Bandwidth:
   - เป็น bottleneck หลักสำหรับ memory-intensive algorithms
   - ใช้ STREAM benchmark วัด achievable bandwidth
   - Practical BW ≈ 70-85% ของ theoretical

2. RFO (Read-For-Ownership):
   - Happens เมื่อ write ไป non-cached line
   - ใช้ bandwidth 2x: read + write
   - แก้ด้วย NT stores หรือ prefetchw

3. NT Stores (MOVNTDQ etc.):
   - Bypass cache, write โดยตรงไป DRAM
   - ประหยัด RFO bandwidth
   - ต้องการ SFENCE ตอนท้าย
   - เหมาะสำหรับ streaming write (ข้อมูลไม่ถูกอ่านทันที)

4. Write Combining Buffer:
   - Hardware buffer รวม NT writes
   - 4-12 entries per core
   - ต้อง write full 64-byte cache lines เพื่อประสิทธิภาพสูงสุด
   - อย่าเปิด entries มากกว่า hardware รองรับพร้อมกัน

5. Huge Pages (2MB/1GB):
   - ลด TLB misses อย่างมีนัยสำคัญ
   - สำคัญมากสำหรับ large dataset
   - ใช้ THP หรือ MAP_HUGETLB

6. NUMA:
   - Remote access ช้ากว่า local 1.5-3x
   - Pin threads กับ memory ไว้ที่ node เดียวกัน
   - numactl ช่วยจัดการ NUMA policy

7. Bandwidth Analysis:
   - คำนวณ Arithmetic Intensity ก่อน
   - ถ้า AI < ridge point: bandwidth-bound
   - ถ้า AI > ridge point: compute-bound
   - DAXPY เป็น classic bandwidth-bound example (AI ≈ 0.083)
```

### 20.2 Decision Tree สำหรับ Optimization

```
เริ่มต้น: algorithm ช้ากว่าที่คาด
        |
        v
วัด bandwidth (STREAM) → ถึง peak? → YES → ปัญหาอื่น (compute?)
        |
        NO
        |
        v
วัด cache miss rate (perf) → miss rate สูง? → YES → ปัญหา access pattern
        |
        NO
        |
        v
ตรวจสอบ NUMA → remote access? → YES → ใช้ numactl
        |
        NO
        |
        v
ตรวจสอบ TLB misses → สูง? → YES → ใช้ huge pages
        |
        NO  
        |
        v
ตรวจสอบ write pattern → RFO สูง? → YES → ใช้ NT stores
        |
        NO
        |
        v
ตรวจสอบ WCB utilization → partial fills? → YES → ปรับ write order
```

---

## 21. ตัวอย่างเพิ่มเติม: GAS Syntax

### 21.1 NT Copy ใน GAS

```gas
# GAS (AT&T syntax) - NT copy ด้วย AVX2
.section .text
.global nt_memcpy_gas

# rdi = dst, rsi = src, rdx = count (bytes)
nt_memcpy_gas:
    pushq   %rbx
    
    movq    %rdx, %rbx
    shrq    $5, %rbx            # count / 32 (AVX2 = 256-bit = 32 bytes)
    
    testq   %rbx, %rbx
    jz      .nt_remainder
    
.nt_loop:
    vmovdqu     (%rsi), %ymm0   # load 32 bytes
    vmovntdq    %ymm0, (%rdi)   # NT store 32 bytes
    
    addq    $32, %rsi
    addq    $32, %rdi
    decq    %rbx
    jnz     .nt_loop
    
.nt_remainder:
    movq    %rdx, %rcx
    andq    $31, %rcx           # remaining bytes
    
.byte_loop:
    testq   %rcx, %rcx
    jz      .nt_done
    
    movb    (%rsi), %al
    movb    %al, (%rdi)
    incq    %rsi
    incq    %rdi
    decq    %rcx
    jnz     .byte_loop
    
.nt_done:
    sfence
    vzeroupper
    popq    %rbx
    retq
```

### 21.2 NUMA-Aware Allocation Helper ใน GAS

```gas
# GAS - mmap กับ huge pages
.section .rodata
.align 8
.Lprot_rw:   .quad 3                  # PROT_READ | PROT_WRITE = 3
.Lmap_flags: .quad 0x400022           # MAP_PRIVATE|MAP_ANON|MAP_HUGETLB

.section .text
.global alloc_2mb_hugepage

# rdi = size (multiple of 2MB)
# returns: rax = pointer or negative errno
alloc_2mb_hugepage:
    movq    %rdi, %rsi              # size
    xorq    %rdi, %rdi              # addr = NULL
    movq    $3, %rdx                # PROT_READ | PROT_WRITE
    movq    $0x400022, %r10         # MAP_PRIVATE | MAP_ANON | MAP_HUGETLB
    orq     $(21 << 26), %r10       # | MAP_HUGE_2MB
    movq    $-1, %r8                # fd = -1
    xorq    %r9, %r9                # offset = 0
    movq    $9, %rax                # SYS_mmap
    syscall
    retq

.global free_hugepage

# rdi = ptr, rsi = size
free_hugepage:
    movq    $11, %rax               # SYS_munmap
    syscall
    retq
```

---

## 22. อ้างอิงและแหล่งข้อมูลเพิ่มเติม

### 22.1 Intel Architecture References

```
- Intel 64 and IA-32 Architectures Optimization Reference Manual
  Chapter 7: Memory Access Optimization
  Section 7.3: Non-Temporal Stores

- Intel 64 and IA-32 Architectures Software Developer's Manual
  Volume 2: Instruction Set Reference
  MOVNTDQ, MOVNTPS, MOVNTPD, MOVNTI, MOVNTDQA
  CLWB, CLFLUSHOPT, WBNOINVD
  PREFETCHNTA, PREFETCHT0, PREFETCHT1, PREFETCHT2

- Intel Memory Latency Checker (MLC)
  Tool สำหรับวัด memory bandwidth และ latency

- STREAM Benchmark: http://www.cs.virginia.edu/stream/
```

### 22.2 Performance Analysis Tools

```
Tools ที่ควรรู้จัก:
1. perf          - Linux performance counter tool
2. Intel VTune   - Detailed CPU/memory profiling
3. AMD uProf     - AMD's performance profiler  
4. numactl       - NUMA control utility
5. numastat      - NUMA statistics
6. hwloc         - Hardware locality tool
7. likwid        - LIKwid performance tools
8. PAPI          - Performance API library
```

### 22.3 Benchmark Programs

```bash
# STREAM benchmark
wget https://www.cs.virginia.edu/stream/FTP/Code/stream.c
gcc -O3 -march=native -fopenmp -DSTREAM_ARRAY_SIZE=100000000 \
    stream.c -o stream
./stream

# Intel MLC
# (ต้อง download จาก Intel website)
mlc --bandwidth_matrix
mlc --latency_matrix

# สร้าง simple bandwidth test
cat > bw_test.c << 'EOF'
#include <stdlib.h>
#include <time.h>

#define N (1<<27)  // 128M elements
double a[N], b[N], c[N];

int main() {
    struct timespec t1, t2;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    
    double scalar = 3.0;
    for (long i = 0; i < N; i++)
        a[i] = b[i] + scalar * c[i];    // STREAM Triad
    
    clock_gettime(CLOCK_MONOTONIC, &t2);
    
    double dt = (t2.tv_sec - t1.tv_sec) + 
                (t2.tv_nsec - t1.tv_nsec) * 1e-9;
    double bytes = 3.0 * N * sizeof(double);
    printf("Bandwidth: %.2f GB/s\n", bytes / dt / 1e9);
    return 0;
}
EOF
gcc -O3 -march=native bw_test.c -o bw_test
./bw_test
```

---

## 23. ตัวอย่างโค้ดสมบูรณ์: Optimized Matrix-Vector Product

### 23.1 SpMV หรือ Dense MV Product (Bandwidth-Bound)

```nasm
; NASM - Dense matrix-vector product (bandwidth-bound for large matrices)
; y = A * x, A is M×N matrix
; รับ: rdi=y(M), rsi=A(M×N), rdx=x(N), rcx=M, r8=N

section .text
global matvec_bandwidth_optimized

matvec_bandwidth_optimized:
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    push    rbp
    sub     rsp, 64
    
    mov     r12, rcx                ; M
    mov     r13, r8                 ; N
    mov     r14, rdi                ; y
    mov     r15, rsi                ; A
    mov     rbp, rdx                ; x
    
    ; Outer loop: for each row i of A
    xor     rbx, rbx                ; i = 0
    
.row_loop:
    cmp     rbx, r12
    jge     .done
    
    ; Inner product: y[i] = dot(A[i], x)
    vxorpd  ymm0, ymm0, ymm0        ; accumulator
    
    xor     rcx, rcx                ; j = 0
    
.col_loop:
    ; Process 4 doubles per iteration (AVX2)
    lea     rax, [rcx + 4]
    cmp     rax, r13
    jg      .col_tail
    
    ; Prefetch next row's data
    lea     rax, [r15 + (r13 + 64)*8]
    prefetcht1 [rax]
    
    ; Load 4 elements from A row and x
    vmovupd     ymm1, [r15 + rcx*8] ; A[i][j:j+4]
    vmovupd     ymm2, [rbp + rcx*8] ; x[j:j+4]
    
    ; Accumulate: y[i] += A[i][j] * x[j]
    vfmadd231pd ymm0, ymm1, ymm2
    
    add     rcx, 4
    jmp     .col_loop
    
.col_tail:
    ; Handle remaining columns
    cmp     rcx, r13
    jge     .col_done
    
    vmovsd  xmm1, [r15 + rcx*8]
    vmovsd  xmm2, [rbp + rcx*8]
    vfmadd231sd xmm0, xmm1, xmm2
    inc     rcx
    jmp     .col_tail
    
.col_done:
    ; Horizontal sum of ymm0
    vextractf128    xmm1, ymm0, 1
    vaddpd          xmm0, xmm0, xmm1
    vhaddpd         xmm0, xmm0, xmm0
    
    ; Store y[i]
    vmovsd  [r14 + rbx*8], xmm0
    
    ; Advance to next row
    add     r15, r13                ; A += N (next row)
    inc     rbx
    jmp     .row_loop
    
.done:
    vzeroupper
    add     rsp, 64
    pop     rbp
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    ret
```

---

## 24. ทบทวนสูตรและตาราง Reference

### 24.1 Quick Reference: NT Instructions

```
╔══════════════╦═══════════╦══════════════════════════════════════╗
║ Instruction  ║   Width   ║ Notes                                ║
╠══════════════╬═══════════╬══════════════════════════════════════╣
║ MOVNTI       ║ 32/64-bit ║ NT store integer, requires alignment ║
║ MOVNTPS      ║ 128-bit   ║ NT store 4×float (SSE)               ║
║ MOVNTPD      ║ 128-bit   ║ NT store 2×double (SSE2)             ║
║ MOVNTDQ      ║ 128-bit   ║ NT store 128-bit int (SSE2)          ║
║ MOVNTDQA     ║ 128-bit   ║ NT load from WC memory (SSE4.1)      ║
╠══════════════╬═══════════╬══════════════════════════════════════╣
║ VMOVNTPS     ║ 256-bit   ║ NT store 8×float (AVX)               ║
║ VMOVNTPD     ║ 256-bit   ║ NT store 4×double (AVX)              ║
║ VMOVNTDQ     ║ 256-bit   ║ NT store 256-bit int (AVX2)          ║
║ VMOVNTDQA    ║ 256-bit   ║ NT load from WC (AVX2)               ║
╠══════════════╬═══════════╬══════════════════════════════════════╣
║ VMOVNTPS     ║ 512-bit   ║ NT store 16×float (AVX-512)          ║
║ VMOVNTPD     ║ 512-bit   ║ NT store 8×double (AVX-512)          ║
║ VMOVNTDQ     ║ 512-bit   ║ NT store 512-bit int (AVX-512)       ║
║ VMOVNTDQA    ║ 512-bit   ║ NT load from WC (AVX-512)            ║
╠══════════════╬═══════════╬══════════════════════════════════════╣
║ CLFLUSH      ║ 64-byte   ║ Flush+invalidate cache line          ║
║ CLFLUSHOPT   ║ 64-byte   ║ Flush+invalidate (optimized)         ║
║ CLWB         ║ 64-byte   ║ Flush without invalidate             ║
║ WBINVD       ║ all       ║ Write back and invalidate all        ║
║ WBNOINVD     ║ all       ║ Write back, no invalidate (ICL+)     ║
╠══════════════╬═══════════╬══════════════════════════════════════╣
║ SFENCE       ║ -         ║ Store fence (after NT stores)        ║
║ LFENCE       ║ -         ║ Load fence                           ║
║ MFENCE       ║ -         ║ Full memory fence                    ║
╚══════════════╩═══════════╩══════════════════════════════════════╝
```

### 24.2 Quick Reference: Prefetch Instructions

```
╔═══════════════╦═══════════════════════════════════════════════╗
║ Instruction   ║ Description                                   ║
╠═══════════════╬═══════════════════════════════════════════════╣
║ PREFETCHNTA   ║ Non-temporal: fetch to L1 only (no pollution) ║
║ PREFETCHT0    ║ Temporal L0: fetch to all levels (L1/L2/L3)   ║
║ PREFETCHT1    ║ Temporal L1: fetch to L2 and above            ║
║ PREFETCHT2    ║ Temporal L2: fetch to L3                      ║
║ PREFETCHW     ║ Write: fetch with write intent (exclusive)    ║
║ PREFETCHWT1   ║ Write T1: fetch write intent to L2+           ║
╚═══════════════╩═══════════════════════════════════════════════╝

การเลือกใช้:
- Streaming read (once): PREFETCHNTA
- Reused data:           PREFETCHT0
- Write-then-read:       PREFETCHW
- Large working set:     PREFETCHT1 / PREFETCHT2
```

### 24.3 Memory Bandwidth Rules of Thumb

```
กฎเบื้องต้น:
1. NT stores ≈ 1.5-2x เร็วกว่า temporal stores สำหรับ large streaming write
2. Huge pages ≈ 10-30% เร็วกว่าสำหรับ large dataset access
3. NUMA local ≈ 1.5-3x เร็วกว่า NUMA remote
4. Sequential ≈ 10-100x เร็วกว่า random access (ขึ้นกับ dataset size)
5. 4-channel memory ≈ 3.5x เร็วกว่า single channel (ไม่ scale perfect)
6. AVX-512 NT store ≈ ดีกว่า AVX2 NT store ≈ 30-50% สำหรับ large streams
7. Optimal prefetch distance = memory_latency × loop_bandwidth / data_per_iter
```

---

## บทสรุป (Conclusion)

Memory bandwidth optimization เป็นศิลปะที่ต้องเข้าใจทั้ง hardware architecture และ software patterns บทนี้ครอบคลุมตั้งแต่:

1. **DRAM Bandwidth fundamentals** - ทำความเข้าใจ theoretical vs practical bandwidth
2. **Memory Wall Problem** - ทำไม CPU ถึง wait รอ memory
3. **STREAM Benchmark** - วิธีวัด sustainable bandwidth
4. **RFO** - ต้นทุนของ write to uncached lines
5. **NT Stores** - วิธี bypass cache สำหรับ streaming write
6. **Write Combining Buffer** - hardware mechanism สำหรับ batch writes
7. **NT Loads (MOVNTDQA)** - สำหรับ WC memory access
8. **Cache flush instructions** - CLWB, CLFLUSHOPT, WBNOINVD
9. **Huge Pages** - ลด TLB miss overhead
10. **NUMA** - manage non-uniform memory topology
11. **Bandwidth analysis** - Roofline model, arithmetic intensity
12. **DAXPY** - classic bandwidth-bound benchmark

การเลือกใช้เทคนิคที่เหมาะสมขึ้นอยู่กับ:
- ขนาดของ dataset (เล็กกว่า L3 ↔ ใหญ่กว่า L3)
- Pattern การ access (sequential, strided, random)
- Write vs read ratio
- Multi-socket vs single-socket
- ความสำคัญของ cache temporal locality

ด้วยความเข้าใจเหล่านี้ สามารถเพิ่มประสิทธิภาพ memory-intensive code ได้ตั้งแต่ 2x ไปจนถึง 10x+ เมื่อเปรียบเทียบกับ naive implementation

---

*Part 067 - Memory Bandwidth Optimization*
*Assembly Language Course - Thai/English Mixed*
*NASM and GAS (AT&T) Syntax Examples*
