# Part 068: Profiling and Performance Analysis
# การ Profiling และการวิเคราะห์ประสิทธิภาพ

---

## บทนำ (Introduction)

การเขียน Assembly ที่เร็วนั้นไม่ใช่แค่การรู้จัก instruction ต่างๆ แต่ต้องรู้ว่าโปรแกรมช้าตรงไหน
และทำไมถึงช้า Profiling คือกระบวนการวัดและวิเคราะห์ประสิทธิภาพของโปรแกรม

```
Performance Engineering Pipeline:
┌─────────────────────────────────────────────────────────┐
│  Code → Build → Profile → Analyze → Optimize → Repeat   │
└─────────────────────────────────────────────────────────┘

ไม่มีการ profile = ไม่รู้ว่าควร optimize ตรงไหน
"Premature optimization is the root of all evil" - Donald Knuth
```

---

## 1. RDTSC Instruction — อ่าน Timestamp Counter

### 1.1 พื้นฐาน RDTSC

`RDTSC` (Read Time-Stamp Counter) อ่านค่า 64-bit counter จาก CPU
ที่นับ clock cycle ตั้งแต่ processor reset

```nasm
; NASM syntax - RDTSC พื้นฐาน
section .text
global _start

_start:
    ; RDTSC ใส่ผลใน EDX:EAX (EDX = high 32 bits, EAX = low 32 bits)
    rdtsc
    
    ; รวม EDX:EAX เป็น 64-bit value
    shl     rdx, 32         ; shift high part ขึ้น
    or      rax, rdx        ; รวมกัน → RAX = TSC value
    
    ; RAX ตอนนี้มีค่า TSC 64-bit
    ret
```

```gas
# GAS syntax - RDTSC พื้นฐาน
.section .text
.globl read_tsc

read_tsc:
    rdtsc
    shlq    $32, %rdx
    orq     %rdx, %rax      # rax = full 64-bit TSC
    ret
```

### 1.2 ปัญหาของ RDTSC — Out-of-Order Execution

RDTSC ไม่ serializing หมายความว่า CPU อาจ reorder คำสั่งนี้กับ instruction อื่น
ทำให้ผลการวัดไม่แม่นยำ

```
ปัญหา Out-of-Order:
┌────────────────────────────────────────────────────┐
│  เราเขียน:          CPU อาจทำ:                      │
│  rdtsc             work_to_measure  ← เร็วขึ้น     │
│  work_to_measure   rdtsc            ← วัดไม่ถูก    │
│  rdtsc             rdtsc                            │
└────────────────────────────────────────────────────┘
```

---

## 2. RDTSCP — Serializing Version

`RDTSCP` เพิ่ม serialization บางส่วน: รอให้ instructions ก่อนหน้า retire ก่อน
แต่ยังไม่รอ instructions หลังจากนั้น

```nasm
; NASM - RDTSCP
; ผลใน EDX:EAX เหมือน RDTSC
; ECX = TSC_AUX register (processor ID / NUMA node)
section .text
global read_tsc_serialized

read_tsc_serialized:
    rdtscp
    ; EDX:EAX = TSC, ECX = processor ID
    shl     rdx, 32
    or      rax, rdx
    ; สามารถเช็ค ECX ว่า migrate ไป core อื่นหรือเปล่า
    ret
```

```gas
# GAS - RDTSCP
.globl bench_with_rdtscp
bench_with_rdtscp:
    # วัด start
    rdtscp
    shlq    $32, %rdx
    orq     %rdx, %rax
    movq    %rax, %r8           # r8 = start_tsc
    movl    %ecx, %r9d          # r9d = start_cpu_id

    # --- code to measure ---
    call    my_function
    # --- end of measured code ---

    # วัด end
    rdtscp
    shlq    $32, %rdx
    orq     %rdx, %rax
    movq    %rax, %r10          # r10 = end_tsc
    movl    %ecx, %r11d         # r11d = end_cpu_id

    # ตรวจสอบว่า CPU migrate หรือเปล่า
    cmpl    %r9d, %r11d
    jne     .cpu_migrated       # ถ้า migrate ผลไม่น่าเชื่อถือ

    # คำนวณ elapsed
    subq    %r8, %r10           # r10 = elapsed cycles
    movq    %r10, %rax
    ret

.cpu_migrated:
    movq    $-1, %rax           # return error
    ret
```

---

## 3. CPUID ก่อน RDTSC เพื่อ Serialization เต็มรูปแบบ

วิธีที่ถูกต้องที่สุดคือใช้ `CPUID` เป็น serializing instruction ก่อน `RDTSC`
เพราะ CPUID บังคับให้ pipeline ทำ retire ทุก pending instruction ก่อน

```nasm
; NASM - Pattern ที่ถูกต้องสำหรับ RDTSC
; Pattern: CPUID → RDTSC → work → RDTSC → CPUID

section .data
    tsc_start   dq 0
    tsc_end     dq 0

section .text
global measure_cycles

measure_cycles:
    push    rbx                 ; CPUID เปลี่ยน rbx
    push    r12
    
    ; === START MEASUREMENT ===
    ; Serialize pipeline ด้วย CPUID
    xor     eax, eax            ; CPUID leaf 0
    cpuid                       ; serialize ทุก pending instructions
    
    ; อ่าน TSC
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    mov     [tsc_start], rax    ; บันทึก start time
    
    ; === CODE TO MEASURE ===
    call    function_to_measure
    ; === END CODE ===
    
    ; === END MEASUREMENT ===
    ; อ่าน TSC ก่อน (ก่อน CPUID)
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    mov     r12, rax            ; บันทึก end time ชั่วคราว
    
    ; Serialize pipeline หลัง measurement
    xor     eax, eax
    cpuid
    
    ; คำนวณ elapsed
    mov     rax, r12
    sub     rax, [tsc_start]    ; elapsed cycles
    
    pop     r12
    pop     rbx
    ret
```

### 3.1 ทำไมต้องใช้ Pattern นี้?

```
Pattern อธิบาย:

CPUID (serialize) ──→ pipeline ว่าง ─┐
RDTSC ──────────────────────────────→ อ่าน TSC แม่นยำ (start)
                                      │
[code to measure] ──────────────────→ ทำงาน
                                      │
RDTSC ──────────────────────────────→ อ่าน TSC ทันที (end)
CPUID (serialize) ──→ ทำให้แน่ใจว่า code ทำเสร็จแล้วจริงๆ

หมายเหตุ: RDTSC ก่อน CPUID ตัวที่สอง
เพราะ CPUID ทำให้ code ที่ต้องวัด "drain" ออกจาก pipeline
```

---

## 4. วัดด้วย RDTSC: ใช้ Minimum ไม่ใช่ Average

### 4.1 ทำไม Minimum?

```
การกระจายของ Timing Measurements:
                    │
 frequency          │ ████
                    │ ████
                    │ ████ ██
                    │ ████ ██ █
                    │ ████ ██ █  █       █
                    └─────────────────────── cycles
                      min  │        outliers
                           avg (เกิดจาก noise)

min = "ideal case" — เกิดขึ้นจริง ไม่มี noise
avg = ผลรวมของ noise ทั้งหมด (OS interrupts, cache misses, etc.)
```

### 4.2 Benchmark Framework ใช้ Minimum

```c
/* C code สำหรับวัด minimum cycles */
#include <stdint.h>
#include <stdio.h>

static inline uint64_t rdtsc_start(void) {
    uint32_t lo, hi;
    __asm__ __volatile__ (
        "cpuid\n\t"
        "rdtsc\n\t"
        : "=a"(lo), "=d"(hi)
        : "a"(0)
        : "%rbx", "%rcx"
    );
    return ((uint64_t)hi << 32) | lo;
}

static inline uint64_t rdtsc_end(void) {
    uint32_t lo, hi;
    __asm__ __volatile__ (
        "rdtsc\n\t"
        "cpuid\n\t"
        : "=a"(lo), "=d"(hi)
        :: "%rbx", "%rcx"
    );
    return ((uint64_t)hi << 32) | lo;
}

#define N_RUNS 100

uint64_t measure_function(void (*fn)(void)) {
    uint64_t min_cycles = UINT64_MAX;
    
    /* Warmup: ทำให้ cache warm ก่อน */
    for (int i = 0; i < 10; i++) {
        fn();
    }
    
    /* วัดจริง: เก็บค่าน้อยที่สุด */
    for (int i = 0; i < N_RUNS; i++) {
        uint64_t start = rdtsc_start();
        fn();
        uint64_t end = rdtsc_end();
        
        uint64_t elapsed = end - start;
        if (elapsed < min_cycles) {
            min_cycles = elapsed;
        }
    }
    
    return min_cycles;
}
```

```nasm
; NASM Assembly version ของ benchmark loop
section .data
    min_cycles  dq 0xFFFFFFFFFFFFFFFF  ; ค่าเริ่มต้น = MAX

section .text
global asm_benchmark_loop

; rdi = pointer to function to benchmark
; rsi = number of runs
asm_benchmark_loop:
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    
    mov     r12, rdi            ; function pointer
    mov     r13, rsi            ; N runs
    mov     r14, [min_cycles]   ; current minimum
    
.warmup:
    ; Warmup 10 runs
    mov     rcx, 10
.warmup_loop:
    call    r12
    dec     rcx
    jnz     .warmup_loop

.measure_loop:
    test    r13, r13
    jz      .done
    
    ; Start measurement
    xor     eax, eax
    cpuid
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    mov     r15, rax            ; r15 = start TSC
    
    ; Call function
    call    r12
    
    ; End measurement
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    xor     ebx, ebx
    cpuid
    
    ; elapsed = end - start
    sub     rax, r15
    
    ; update minimum
    cmp     rax, r14
    jae     .no_update
    mov     r14, rax
.no_update:
    
    dec     r13
    jmp     .measure_loop
    
.done:
    mov     [min_cycles], r14
    mov     rax, r14
    
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    ret
```

---

## 5. perf Tool — Linux Performance Counter

`perf` คือ profiling tool หลักบน Linux ใช้ hardware performance counters

### 5.1 perf stat — Overview

```bash
# ดู overview ของ performance counters
perf stat ./my_program

# ตัวอย่าง output:
# Performance counter stats for './my_program':
#
#       1,234.56 msec task-clock         #    0.998 CPUs utilized
#              5      context-switches   #    4.050 /sec
#              0      cpu-migrations     #    0.000 /sec
#            142      page-faults        #  115.040 /sec
#  3,456,789,012      cycles             #    2.800 GHz
#  5,678,901,234      instructions       #    1.64  insn per cycle
#    234,567,890      branches           #  190.031 M/sec
#      1,234,567      branch-misses      #    0.53% of all branches
#
#       1.237285734 seconds time elapsed

# รัน หลายครั้งเพื่อได้ค่าที่เสถียร
perf stat -r 5 ./my_program

# ดู events เฉพาะ
perf stat -e cycles,instructions,cache-misses,cache-references ./my_program

# ดู counter สำหรับ child processes ด้วย
perf stat -a ./my_program
```

### 5.2 perf record — Sampling Profiler

```bash
# Record โดย sample ด้วย cycle-based sampling
perf record -g ./my_program

# Record ด้วย frequency ที่กำหนด (1000 samples/sec)
perf record -F 1000 -g ./my_program

# Record เฉพาะ event ที่ต้องการ
perf record -e cache-misses:u -g ./my_program

# Record ทั้ง system (ต้องการ root)
perf record -a -g sleep 10

# Record ด้วย call graph (3 modes)
perf record --call-graph fp ./my_program   # frame pointer (เร็ว)
perf record --call-graph dwarf ./my_program # DWARF info (แม่นยำ)
perf record --call-graph lbr ./my_program  # Last Branch Record (Intel)
```

### 5.3 perf report — ดูผล Sampling

```bash
# ดูผลจาก perf.data
perf report

# ดูแบบ text (ไม่ใช้ TUI)
perf report --stdio

# ดู hotspot ที่ระดับ symbol
perf report --sort=symbol

# ดูด้วย call graph
perf report --call-graph

# Output ไปไฟล์
perf report --stdio > report.txt
```

### 5.4 perf annotate — Assembly-Level Analysis

```bash
# Annotate: แสดง assembly พร้อม percentage
perf annotate

# Annotate specific symbol
perf annotate my_hot_function

# Annotate แบบ text
perf annotate --stdio my_hot_function

# ตัวอย่าง output ของ perf annotate:
#  Percent |  Source code & Disassembly
# -----------------------------------------------
#          :  0000000000401234 <my_hot_function>:
#    0.12  :    401234: push   %rbp
#    0.45  :    401238: mov    %rsp, %rbp
#   45.67  :    40123c: vmovups (%rdi), %ymm0     ← hotspot!
#    2.34  :    401240: vaddps  %ymm1, %ymm0, %ymm0
#   38.91  :    401244: vmovups %ymm0, (%rsi)     ← hotspot!
```

---

## 6. perf list — ดู Hardware Events ที่รองรับ

```bash
# ดู events ทั้งหมด
perf list

# กรองเฉพาะ hardware events
perf list hw

# กรองเฉพาะ software events
perf list sw

# กรองเฉพาะ cache events
perf list cache

# กรองเฉพาะ tracepoints
perf list tracepoint

# ค้นหา event เฉพาะ
perf list | grep "cache"
perf list | grep "branch"
```

### 6.1 Hardware vs Software Events

```
Hardware PMU Events:
┌─────────────────────────────────────────────────────────────┐
│ นับโดย CPU hardware performance monitoring unit (PMU)       │
│                                                             │
│ ✓ แม่นยำมาก (นับทุก cycle)                                 │
│ ✓ Overhead ต่ำมาก                                          │
│ ✗ มีจำนวนจำกัด (4-8 counters พร้อมกัน)                    │
│ ✗ ขึ้นกับ CPU model                                        │
│                                                             │
│ Examples: cycles, instructions, cache-misses                │
└─────────────────────────────────────────────────────────────┘

Software Events:
┌─────────────────────────────────────────────────────────────┐
│ นับโดย kernel software counters                             │
│                                                             │
│ ✓ ไม่ขึ้นกับ CPU model                                     │
│ ✓ มีได้หลาย counters พร้อมกัน                              │
│ ✗ Overhead สูงกว่า (kernel trap)                           │
│ ✗ ไม่ fine-grained เท่า hardware                          │
│                                                             │
│ Examples: page-faults, context-switches, cpu-migrations     │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Key Performance Events

### 7.1 Cycles และ Instructions

```bash
# วัด cycles และ instructions พร้อมกัน
perf stat -e cycles,instructions ./my_program

# คำนวณ IPC (Instructions Per Cycle)
# IPC = instructions / cycles
# IPC สูง = ดี (CPU ทำงาน instructions หลายตัวพร้อมกัน)
# IPC ต่ำ = มีปัญหา (stalls, cache misses, branch mispredictions)
```

```
IPC Interpretation:
┌────────────────────────────────────────────────────┐
│ IPC > 3.0  : ดีมาก (modern out-of-order CPU)      │
│ IPC 2.0-3.0: ดี                                    │
│ IPC 1.0-2.0: พอใช้ได้                              │
│ IPC < 1.0  : มีปัญหา → ต้องสืบสวน                 │
└────────────────────────────────────────────────────┘

Modern CPUs (Intel Skylake+, AMD Zen 3+):
- Theoretical max IPC ≈ 4-6 (4-6 execution ports)
- Memory-bound code: IPC ต่ำมาก
- Compute-bound (คำนวณล้วน): IPC สูง
```

### 7.2 Cache Misses

```bash
# Cache miss events
perf stat -e \
    cache-references,\
    cache-misses,\
    L1-dcache-loads,\
    L1-dcache-load-misses,\
    L1-dcache-stores,\
    LLC-loads,\
    LLC-load-misses \
    ./my_program

# ตัวอย่าง output:
# 100,000,000  cache-references
#   5,000,000  cache-misses          # 5% miss rate
#              # ถ้า miss rate > 5% = ปัญหา cache
```

```
Cache Hierarchy Impact:
┌─────────────────────────────────────────────────────┐
│ L1 cache hit:  ~4 cycles                            │
│ L2 cache hit:  ~12 cycles                           │
│ L3 cache hit:  ~40 cycles                           │
│ RAM (DRAM):    ~200 cycles                          │
│ SSD:           ~50,000 cycles                       │
│                                                     │
│ ดังนั้น cache miss ทุก L3→RAM = เสีย 200 cycles    │
└─────────────────────────────────────────────────────┘
```

### 7.3 Branch Misses

```bash
# Branch prediction events
perf stat -e \
    branches,\
    branch-misses,\
    branch-load-misses \
    ./my_program

# Branch miss penalty:
# Intel: ~15-20 cycles per misprediction
# AMD:   ~15-20 cycles per misprediction
```

```nasm
; ตัวอย่าง Branch-friendly vs Branch-unfriendly code

; BAD: Branch หนักในลูป
bad_branch_loop:
    xor     rcx, rcx
.loop:
    cmp     rcx, 1000000
    jge     .done
    
    mov     rax, [data + rcx*8]
    cmp     rax, 0              ; branch ที่ไม่ predictable
    jl      .negative
    add     [pos_sum], rax
    jmp     .next
.negative:
    add     [neg_sum], rax
.next:
    inc     rcx
    jmp     .loop
.done:
    ret

; BETTER: ใช้ conditional move แทน branch
good_no_branch_loop:
    xor     rcx, rcx
    xor     r8, r8              ; pos_sum
    xor     r9, r9              ; neg_sum
.loop:
    cmp     rcx, 1000000
    jge     .done
    
    mov     rax, [data + rcx*8]
    test    rax, rax
    js      .is_neg             ; ใช้แค่เพื่อ add
    add     r8, rax             ; add to pos
    jmp     .next
.is_neg:
    add     r9, rax             ; add to neg
.next:
    ; หรือใช้ cmov:
    ; mov     rbx, rax
    ; neg     rbx
    ; test    rax, rax
    ; cmovs   rax, rbx         ; cmov ไม่มี branch penalty
    inc     rcx
    jmp     .loop
.done:
    mov     [pos_sum], r8
    mov     [neg_sum], r9
    ret
```

### 7.4 Page Faults

```bash
# วัด page faults
perf stat -e page-faults,minor-faults,major-faults ./my_program

# minor fault: page อยู่ใน memory แต่ยังไม่ map (เร็ว)
# major fault: ต้องโหลดจาก disk (ช้ามาก!)
```

---

## 8. IPC Analysis — Instructions Per Cycle

### 8.1 ดู IPC ในโค้ด

```bash
# วัด IPC
perf stat -e cycles,instructions ./my_program
# IPC จะแสดงเป็น "X.XX insn per cycle"

# ดู IPC ต่อ function
perf stat --per-thread -e cycles,instructions -- ./my_program
```

### 8.2 ปัจจัยที่ทำให้ IPC ต่ำ

```
Factors lowering IPC:
┌──────────────────────────────────────────────────────────────┐
│ 1. Memory stalls (cache misses) ← ปัญหาหลักที่สุด           │
│    Solution: Improve data locality, prefetch                 │
│                                                              │
│ 2. Branch mispredictions                                     │
│    Solution: เรียงข้อมูล, ใช้ CMOV, แก้ algorithm            │
│                                                              │
│ 3. Long dependency chains                                    │
│    Solution: Loop unrolling, instruction scheduling          │
│                                                              │
│ 4. Resource conflicts (execution ports)                      │
│    Solution: Mix instruction types                           │
│                                                              │
│ 5. Frontend bottleneck (instruction fetch/decode)           │
│    Solution: ลด branch, ใช้ loop unrolling                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Valgrind — Memory Profiling Suite

### 9.1 Callgrind — Call Graph Profiler

```bash
# รัน callgrind
valgrind --tool=callgrind ./my_program

# ดูผลด้วย callgrind_annotate
callgrind_annotate callgrind.out.*

# ดูด้วย KCachegrind (GUI)
kcachegrind callgrind.out.*

# Options ที่มีประโยชน์
valgrind --tool=callgrind \
         --callgrind-out-file=cg.out \
         --instr-atstart=no \     # ไม่วัดตั้งแต่ต้น (เปิดใน code)
         --collect-atstart=no \
         ./my_program
```

```c
/* เปิด/ปิด callgrind instrumentation ใน code */
#include <valgrind/callgrind.h>

void my_function(void) {
    CALLGRIND_START_INSTRUMENTATION;  /* เริ่มนับ */
    CALLGRIND_TOGGLE_COLLECT;         /* toggle collection */
    
    /* โค้ดที่ต้องการวัด */
    hot_loop();
    
    CALLGRIND_TOGGLE_COLLECT;
    CALLGRIND_STOP_INSTRUMENTATION;
}
```

### 9.2 Cachegrind — Cache Simulator

```bash
# รัน cachegrind
valgrind --tool=cachegrind ./my_program

# Output file: cachegrind.out.<pid>
# แสดง: I cache refs/misses, D cache refs/misses, LL cache

# อ่านผล
cg_annotate cachegrind.out.*

# ตัวอย่าง output:
# I   refs:      1,234,567,890
# I   misses:        1,234,567  (0.10%)
# D   refs:        567,890,123  (rd: 456,789,012  wr: 111,101,111)
# D   misses:        5,678,901  (1.00%)
# LL  refs:         6,913,468
# LL  misses:          123,456  (1.79%)
```

```
Cachegrind Metrics:
┌─────────────────────────────────────────────────────────────┐
│ Ir = Instruction reads (instruction fetches)                │
│ I1mr = L1 instruction miss rate                             │
│ ILmr = Last Level instruction miss rate                     │
│ Dr = Data reads                                             │
│ D1mr = L1 data read miss rate                               │
│ DLmr = Last Level data read miss rate                       │
│ Dw = Data writes                                            │
│ D1mw = L1 data write miss rate                              │
│ DLmw = Last Level data write miss rate                      │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 Massif — Heap Memory Profiler

```bash
# รัน massif (วัด heap usage)
valgrind --tool=massif ./my_program

# ดูผล
ms_print massif.out.*

# ดูด้วย Massif Visualizer (GUI)
massif-visualizer massif.out.*

# Options
valgrind --tool=massif \
         --heap=yes \
         --stacks=yes \          # วัด stack ด้วย
         --pages-as-heap=yes \   # วัด mmap ด้วย
         --time-unit=B \         # หน่วย: bytes allocated
         ./my_program
```

---

## 10. Intel VTune — Professional Profiler

VTune เป็น profiler ระดับ professional จาก Intel ใช้ hardware PMU เต็มรูปแบบ

```
VTune Analysis Types:
┌─────────────────────────────────────────────────────────────┐
│ 1. Hotspots Analysis                                        │
│    - หา function/line ที่ใช้เวลามากที่สุด                  │
│    - CPU sampling-based                                     │
│                                                             │
│ 2. Microarchitecture Exploration                            │
│    - วิเคราะห์ pipeline utilization                        │
│    - Front-end/Back-end bound analysis                      │
│    - Retiring, Bad Speculation, Front End Bound             │
│                                                             │
│ 3. Memory Access Analysis                                   │
│    - Cache miss location ใน code                           │
│    - Bandwidth utilization                                  │
│    - NUMA effects                                           │
│                                                             │
│ 4. Threading Analysis                                       │
│    - Concurrency, locks, waits                             │
│    - OpenMP overhead                                        │
└─────────────────────────────────────────────────────────────┘
```

```bash
# VTune command line
vtune -collect hotspots -result-dir r001 -- ./my_program
vtune -report hotspots -result-dir r001

vtune -collect memory-access -result-dir r002 -- ./my_program
vtune -report memory-access -result-dir r002

# Top-Down Microarchitecture Analysis (TMA)
vtune -collect uarch-exploration -result-dir r003 -- ./my_program
```

### 10.1 Top-Down Microarchitecture Analysis (TMA)

```
TMA Framework (Intel):
┌─────────────────────────────────────────────────────────┐
│                    Pipeline Slots                        │
│                         │                               │
│          ┌──────────────┴──────────────┐                │
│          │                             │                │
│     Retiring              Bad Speculation                │
│   (good work!)          (branch mispredict)              │
│                                                         │
│          ┌──────────────┬──────────────┐                │
│    Front-End Bound    Back-End Bound                     │
│   (fetch/decode)     (execution/memory)                  │
│                                                         │
│ Front-End:                 Back-End:                    │
│  - Branch mispredicts       - Memory bound              │
│  - Instruction cache miss    - Core bound               │
│  - Decode stalls            - Resource conflicts        │
└─────────────────────────────────────────────────────────┘
```

---

## 11. Microbenchmark Design

### 11.1 หลักการสำคัญ

```
Microbenchmark Rules:
┌─────────────────────────────────────────────────────────┐
│ 1. Warmup before measuring                              │
│    - ให้ CPU warm caches, JIT, branch predictors        │
│                                                         │
│ 2. Multiple iterations                                  │
│    - วัดหลายครั้ง ใช้ค่าต่ำสุด (minimum)               │
│                                                         │
│ 3. Statistical significance                             │
│    - ต้องมี enough samples                              │
│    - ดู variance ด้วย ไม่ใช่แค่ mean                   │
│                                                         │
│ 4. Prevent dead code elimination                        │
│    - Compiler อาจลบ code ที่ไม่ใช้ผล                   │
│    - ใช้ volatile หรือ escape/clobber                   │
│                                                         │
│ 5. Realistic workload                                   │
│    - ข้อมูลต้องเหมือน production                       │
└─────────────────────────────────────────────────────────┘
```

### 11.2 Warmup

```c
/* Warmup ที่ถูกต้อง */
void proper_benchmark(void) {
    /* Phase 1: Warmup (อย่างน้อย 100ms หรือ 1000 iterations) */
    for (int i = 0; i < WARMUP_ITERS; i++) {
        DoNotOptimize(function_to_bench(test_data));
    }
    
    /* Phase 2: Measurement */
    uint64_t min_time = UINT64_MAX;
    for (int i = 0; i < MEASURE_ITERS; i++) {
        uint64_t start = rdtsc_start();
        DoNotOptimize(function_to_bench(test_data));
        uint64_t end = rdtsc_end();
        
        uint64_t elapsed = end - start;
        if (elapsed < min_time) min_time = elapsed;
    }
    
    printf("Min cycles: %llu\n", (unsigned long long)min_time);
}
```

### 11.3 Prevent Dead Code Elimination

```c
/* ป้องกัน compiler ลบ code ที่เราวัด */

/* วิธีที่ 1: volatile (ช้า) */
volatile int result = compute();

/* วิธีที่ 2: DoNotOptimize (เหมือน Google Benchmark) */
static inline void DoNotOptimize(void *p) {
    __asm__ __volatile__("" : : "r,m"(p) : "memory");
}

/* วิธีที่ 3: ClobberMemory (บอก compiler ว่า memory เปลี่ยน) */
static inline void ClobberMemory(void) {
    __asm__ __volatile__("" : : : "memory");
}

/* ตัวอย่างการใช้งาน */
uint64_t start = rdtsc_start();
int result = my_function(data);
DoNotOptimize(&result);    /* ป้องกัน dead code elimination */
ClobberMemory();           /* ป้องกัน memory reorder */
uint64_t end = rdtsc_end();
```

```nasm
; NASM: ป้องกัน dead code ใน assembly ง่ายกว่า
; เพราะเราควบคุม instructions โดยตรง
section .text
global bench_function

bench_function:
    ; คำนวณ result
    ; ...
    
    ; ใช้ result (ป้องกัน CPU ไม่ทำ)
    ; เช่น store ไปที่ output pointer
    mov     [rdi], rax      ; store result → ป้องกัน elimination
    ret
```

---

## 12. หลีกเลี่ยง Measurement Artifacts

### 12.1 CPU Frequency Scaling (Turbo/C-states)

```bash
# ปัญหา: CPU เปลี่ยน frequency ระหว่างวัด
# ผลลัพธ์: measurements ไม่สม่ำเสมอ

# วิธีแก้ 1: ปิด Turbo Boost
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo

# วิธีแก้ 2: ตั้ง governor เป็น performance
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $cpu
done

# วิธีแก้ 3: ใช้ cpupower
cpupower frequency-set -g performance

# ตรวจสอบ current frequency
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

```c
/* วัด TSC frequency เพื่อแปลง cycles → seconds */
double measure_tsc_frequency(void) {
    struct timespec ts1, ts2;
    uint64_t tsc1, tsc2;
    
    clock_gettime(CLOCK_MONOTONIC, &ts1);
    tsc1 = rdtsc_start();
    
    /* รอ 100ms */
    usleep(100000);
    
    tsc2 = rdtsc_end();
    clock_gettime(CLOCK_MONOTONIC, &ts2);
    
    double elapsed_ns = (ts2.tv_sec - ts1.tv_sec) * 1e9 +
                        (ts2.tv_nsec - ts1.tv_nsec);
    uint64_t elapsed_cycles = tsc2 - tsc1;
    
    return elapsed_cycles / (elapsed_ns / 1e9);  /* Hz */
}
```

### 12.2 NUMA Effects

```
NUMA (Non-Uniform Memory Access):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  CPU 0 ──── Local Memory 0    CPU 1 ──── Local Memory 1    │
│       ↕ fast access                 ↕ fast access           │
│       └──────────── Remote ─────────┘                      │
│                  access (slow!)                             │
│                                                             │
│ Local memory: ~100ns latency                               │
│ Remote memory: ~200ns latency                              │
└─────────────────────────────────────────────────────────────┘
```

```bash
# วัดโดยไม่มี NUMA effects
# ผูก process ไว้กับ CPU และ memory node เดียวกัน
numactl --cpunodebind=0 --membind=0 ./my_program

# ดู NUMA topology
numactl --hardware
numactl --show

# ดู NUMA stats
cat /proc/meminfo | grep -i numa
numastat
```

### 12.3 Context Switches

```bash
# ลด context switches ด้วยการตั้ง priority
nice -n -20 ./my_program        # ให้ priority สูง
taskset -c 2 ./my_program       # ผูกกับ CPU 2
chrt -f 99 ./my_program         # real-time scheduling

# ดู context switches
perf stat -e context-switches ./my_program

# ใช้ isolcpus ใน kernel cmdline (persistent)
# isolcpus=2,3 → CPU 2,3 ไม่รัน normal tasks
```

```nasm
; ตรวจสอบ CPU migration ใน benchmark loop
section .text
global safe_benchmark

safe_benchmark:
    ; อ่าน processor ID ก่อน
    rdtscp                      ; ECX = processor ID
    mov     r15d, ecx           ; บันทึก starting CPU
    
    ; ... do measurement ...
    
    ; อ่านอีกครั้ง
    rdtscp
    cmp     ecx, r15d           ; เปรียบเทียบ
    jne     .invalidate         ; ถ้า migrate → ทิ้งผล
    
    ; ผลใช้ได้
    ret
    
.invalidate:
    mov     rax, -1             ; error indicator
    ret
```

---

## 13. Benchmark Framework Skeleton (C + Assembly)

### 13.1 Complete Framework

```c
/* benchmark_framework.h */
#ifndef BENCHMARK_FRAMEWORK_H
#define BENCHMARK_FRAMEWORK_H

#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <math.h>

#define WARMUP_ITERATIONS    100
#define MEASURE_ITERATIONS   1000

typedef struct {
    uint64_t min_cycles;
    uint64_t max_cycles;
    double   mean_cycles;
    double   stddev_cycles;
    uint64_t median_cycles;
} BenchResult;

/* รวบรวม samples */
typedef struct {
    uint64_t *data;
    int       count;
    int       capacity;
} SampleBuffer;

/* TSC reading */
static inline uint64_t bench_rdtsc_start(void) {
    uint32_t lo, hi;
    __asm__ __volatile__ (
        "cpuid\n\t"
        "rdtsc\n\t"
        : "=a"(lo), "=d"(hi)
        : "a"(0)
        : "%rbx", "%rcx"
    );
    return ((uint64_t)hi << 32) | lo;
}

static inline uint64_t bench_rdtsc_end(void) {
    uint32_t lo, hi;
    __asm__ __volatile__ (
        "rdtsc\n\t"
        "cpuid\n\t"
        : "=a"(lo), "=d"(hi)
        :: "%rbx", "%rcx"
    );
    return ((uint64_t)hi << 32) | lo;
}

/* Prevent optimization */
static inline void DoNotOptimize(void *val) {
    __asm__ __volatile__("" : : "r,m"(val) : "memory");
}

static inline void ClobberMemory(void) {
    __asm__ __volatile__("" ::: "memory");
}

/* Compare function for qsort */
static int compare_uint64(const void *a, const void *b) {
    uint64_t x = *(const uint64_t *)a;
    uint64_t y = *(const uint64_t *)b;
    return (x > y) - (x < y);
}

/* Run benchmark */
BenchResult run_benchmark(
    void (*fn)(void *arg),
    void *arg,
    int warmup_iters,
    int measure_iters
) {
    BenchResult result = {0};
    uint64_t *samples = malloc(measure_iters * sizeof(uint64_t));
    if (!samples) {
        fprintf(stderr, "OOM\n");
        return result;
    }
    
    /* Warmup phase */
    for (int i = 0; i < warmup_iters; i++) {
        fn(arg);
        ClobberMemory();
    }
    
    /* Measurement phase */
    for (int i = 0; i < measure_iters; i++) {
        uint64_t start = bench_rdtsc_start();
        fn(arg);
        uint64_t end = bench_rdtsc_end();
        samples[i] = end - start;
        ClobberMemory();
    }
    
    /* Compute statistics */
    qsort(samples, measure_iters, sizeof(uint64_t), compare_uint64);
    
    result.min_cycles = samples[0];
    result.max_cycles = samples[measure_iters - 1];
    result.median_cycles = samples[measure_iters / 2];
    
    double sum = 0;
    for (int i = 0; i < measure_iters; i++) {
        sum += samples[i];
    }
    result.mean_cycles = sum / measure_iters;
    
    double var = 0;
    for (int i = 0; i < measure_iters; i++) {
        double diff = samples[i] - result.mean_cycles;
        var += diff * diff;
    }
    result.stddev_cycles = sqrt(var / measure_iters);
    
    free(samples);
    return result;
}

void print_bench_result(const char *name, BenchResult *r) {
    printf("%-30s  min=%6llu  median=%6llu  mean=%7.1f  stddev=%7.1f  max=%6llu  cycles\n",
        name,
        (unsigned long long)r->min_cycles,
        (unsigned long long)r->median_cycles,
        r->mean_cycles,
        r->stddev_cycles,
        (unsigned long long)r->max_cycles
    );
}

#endif /* BENCHMARK_FRAMEWORK_H */
```

### 13.2 Assembly Function ที่จะ Benchmark

```nasm
; NASM: functions_to_bench.asm
section .data
    align 32
    test_data times 1024 dq 0   ; 1024 doubles

section .text
global sum_naive
global sum_simd
global sum_unrolled

; sum_naive: loop ธรรมดา
; rdi = double* data, rsi = int count
sum_naive:
    vxorpd  xmm0, xmm0, xmm0   ; sum = 0.0
    test    rsi, rsi
    jle     .done
    
.loop:
    vaddsd  xmm0, xmm0, [rdi]
    add     rdi, 8
    dec     rsi
    jnz     .loop
    
.done:
    ret

; sum_simd: SIMD ด้วย AVX
; rdi = double* data, rsi = int count (ต้องหาร 4 ลงตัว)
sum_simd:
    vxorpd  ymm0, ymm0, ymm0   ; accumulator = 0
    test    rsi, rsi
    jle     .done
    
.loop:
    vaddpd  ymm0, ymm0, [rdi]   ; 4 doubles ต่อครั้ง
    add     rdi, 32
    sub     rsi, 4
    jg      .loop
    
    ; horizontal add: sum 4 lanes
    vextractf128 xmm1, ymm0, 1
    vaddpd  xmm0, xmm0, xmm1
    vhaddpd xmm0, xmm0, xmm0
    
.done:
    vzeroupper
    ret

; sum_unrolled: unrolled ด้วย 4 accumulators
; rdi = double* data, rsi = int count
sum_unrolled:
    vxorpd  xmm0, xmm0, xmm0
    vxorpd  xmm1, xmm1, xmm1
    vxorpd  xmm2, xmm2, xmm2
    vxorpd  xmm3, xmm3, xmm3
    
.loop:
    cmp     rsi, 4
    jl      .remainder
    
    vaddsd  xmm0, xmm0, [rdi + 0]
    vaddsd  xmm1, xmm1, [rdi + 8]
    vaddsd  xmm2, xmm2, [rdi + 16]
    vaddsd  xmm3, xmm3, [rdi + 24]
    add     rdi, 32
    sub     rsi, 4
    jmp     .loop
    
.remainder:
    test    rsi, rsi
    jle     .combine
    vaddsd  xmm0, xmm0, [rdi]
    add     rdi, 8
    dec     rsi
    jmp     .remainder
    
.combine:
    vaddsd  xmm0, xmm0, xmm1
    vaddsd  xmm2, xmm2, xmm3
    vaddsd  xmm0, xmm0, xmm2
    ret
```

### 13.3 Main Benchmark Program

```c
/* main_bench.c */
#include "benchmark_framework.h"

extern void sum_naive(double *data, int count);
extern void sum_simd(double *data, int count);
extern void sum_unrolled(double *data, int count);

static double test_data[1024];
static volatile double sink;

void bench_naive(void *arg) {
    (void)arg;
    double result = sum_naive(test_data, 1024);
    DoNotOptimize(&result);
    sink = result;
}

void bench_simd(void *arg) {
    (void)arg;
    double result = sum_simd(test_data, 1024);
    DoNotOptimize(&result);
    sink = result;
}

void bench_unrolled(void *arg) {
    (void)arg;
    double result = sum_unrolled(test_data, 1024);
    DoNotOptimize(&result);
    sink = result;
}

int main(void) {
    /* Initialize data */
    for (int i = 0; i < 1024; i++) {
        test_data[i] = (double)i;
    }
    
    printf("Benchmark: sum of 1024 doubles\n");
    printf("%-30s  %6s  %8s  %8s  %8s  %6s\n",
           "Function", "min", "median", "mean", "stddev", "max");
    printf("%s\n", "---------------------------------------------------"
                   "-------------------------------");
    
    BenchResult r;
    
    r = run_benchmark(bench_naive,    NULL, 100, 1000);
    print_bench_result("sum_naive",    &r);
    
    r = run_benchmark(bench_simd,     NULL, 100, 1000);
    print_bench_result("sum_simd",     &r);
    
    r = run_benchmark(bench_unrolled, NULL, 100, 1000);
    print_bench_result("sum_unrolled", &r);
    
    return 0;
}
```

---

## 14. Google Benchmark — C++ Benchmark Framework

### 14.1 Concepts และการใช้งาน

```cpp
/* Google Benchmark syntax */
#include <benchmark/benchmark.h>

/* ฟังก์ชัน benchmark พื้นฐาน */
static void BM_StringCreation(benchmark::State& state) {
    for (auto _ : state) {
        std::string empty_string;
        benchmark::DoNotOptimize(empty_string);
    }
}
BENCHMARK(BM_StringCreation);

/* Benchmark ที่รับ parameter */
static void BM_VectorPush(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        v.reserve(state.range(0));
        for (int i = 0; i < state.range(0); ++i) {
            v.push_back(i);
        }
        benchmark::DoNotOptimize(v.data());
    }
    state.SetBytesProcessed(
        state.iterations() * state.range(0) * sizeof(int)
    );
}
BENCHMARK(BM_VectorPush)->Range(8, 8 << 10);  /* 8 to 8192 */

/* Benchmark Assembly function */
extern "C" void my_asm_function(int* data, int len);

static void BM_AsmFunction(benchmark::State& state) {
    int n = state.range(0);
    std::vector<int> data(n, 42);
    
    for (auto _ : state) {
        my_asm_function(data.data(), n);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(state.iterations() * n);
}
BENCHMARK(BM_AsmFunction)->RangeMultiplier(2)->Range(64, 64 << 10);

BENCHMARK_MAIN();
```

```bash
# Build และรัน Google Benchmark
g++ -O2 -std=c++17 bench.cpp -lbenchmark -lpthread -o bench
./bench

# Output:
# ---------------------------------------------------------
# Benchmark              Time             CPU   Iterations
# ---------------------------------------------------------
# BM_StringCreation      5 ns             5 ns  138388022
# BM_VectorPush/8       69 ns            69 ns    9983543
# BM_VectorPush/64     475 ns           475 ns    1459369
# BM_AsmFunction/64     12 ns            12 ns   58391042
```

### 14.2 Custom Counters

```cpp
static void BM_WithCustomCounters(benchmark::State& state) {
    int ops = 0;
    for (auto _ : state) {
        /* ... work ... */
        ops++;
    }
    
    state.counters["ops"] = ops;
    state.counters["ops_per_sec"] = benchmark::Counter(
        ops, benchmark::Counter::kIsRate
    );
    state.counters["throughput"] = benchmark::Counter(
        state.range(0) * state.iterations(),
        benchmark::Counter::kIsRate,
        benchmark::Counter::OneK::kIs1024
    );
}
```

---

## 15. Flamegraph Generation

### 15.1 สร้าง Flamegraph

```bash
# Step 1: Record ด้วย perf
perf record -F 99 -g --call-graph dwarf ./my_program
# หรือใช้ frame pointer
perf record -F 99 -g ./my_program

# Step 2: Export data
perf script > out.perf

# Step 3: ใช้ FlameGraph tools (จาก Brendan Gregg)
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph

./stackcollapse-perf.pl out.perf > out.folded
./flamegraph.pl out.folded > flamegraph.svg

# เปิด SVG ใน browser
firefox flamegraph.svg
```

### 15.2 อ่าน Flamegraph

```
Flamegraph Anatomy:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ████ main ████████████████████████████████████████████   │
│  ██ foo ████████  ████ bar ████████████████████████████   │
│  ██ baz ██████    ████ qux ██ ███ inner_loop ████████      │
│                                                             │
│  X axis = time (width = % of total samples)                │
│  Y axis = call depth (bottom = oldest frame)               │
│                                                             │
│  Wide bar = function ใช้เวลามาก                            │
│  Tall stack = deep call chain                              │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Differential Flamegraph (เปรียบเทียบ before/after)
./stackcollapse-perf.pl before.perf > before.folded
./stackcollapse-perf.pl after.perf  > after.folded
./difffolded.pl before.folded after.folded | ./flamegraph.pl > diff.svg
# แดง = เพิ่มขึ้น, น้ำเงิน = ลดลง

# OffCPU Flamegraph (เวลาที่ block/sleep)
perf record -e sched:sched_switch -a -g -- sleep 10
perf script | ./stackcollapse-perf.pl > off_cpu.folded
./flamegraph.pl --color=io off_cpu.folded > offcpu.svg
```

---

## 16. Hotspot Identification

### 16.1 กระบวนการหา Hotspot

```
Hotspot Identification Process:
┌─────────────────────────────────────────────────────────────┐
│ Level 1: Application level                                  │
│   perf stat → ดู IPC, cache miss rate, branch miss rate    │
│                                                             │
│ Level 2: Function level                                     │
│   perf report → หา function ที่ใช้ % สูง                   │
│                                                             │
│ Level 3: Line level                                         │
│   perf annotate → หา line ใน function นั้น                 │
│                                                             │
│ Level 4: Instruction level                                  │
│   perf annotate assembly → หา instruction ที่ stall        │
│                                                             │
│ Level 5: Microarchitecture level                            │
│   VTune / perf mem → หาว่า stall เกิดจากอะไร             │
└─────────────────────────────────────────────────────────────┘
```

### 16.2 เครื่องมือเพิ่มเติม

```bash
# gprof: classical profiler (sampling + instrumentation)
gcc -pg -O2 mycode.c -o mycode
./mycode
gprof mycode gmon.out > analysis.txt

# gperftools (Google Performance Tools)
LD_PRELOAD=/usr/lib/libprofiler.so CPUPROFILE=prof.out ./mycode
pprof --pdf ./mycode prof.out > prof.pdf

# Hotspot ใน perf report
perf report --sort=dso,sym --stdio | head -50

# ดู cache miss hotspot
perf mem record -t load ./my_program
perf mem report --sort=symbol
```

---

## 17. Assembly Annotation Interpretation

### 17.1 อ่าน perf annotate output

```
perf annotate output interpretation:

  3.45 :     4005c0: vmovdqu  (%rdi,%rax,1), %ymm1
              │
              └── 3.45% ของ samples ตกที่ instruction นี้
                  แต่ samples assign ให้ instruction ที่ "retire" ตอนนั้น
                  ไม่ใช่ instruction ที่ "dispatch" เสมอ

  หากเห็น % สูงที่:
  - load instructions → cache miss latency
  - div/sqrt instructions → long latency
  - branch instructions → misprediction recovery
  
  แต่ CPU pipeline ทำให้ % อาจตกที่ instruction ถัดไปจาก miss
```

### 17.2 ตัวอย่างการอ่าน Annotation

```nasm
; annotated output ตัวอย่าง:
;
;  Percent │ Assembly
; ─────────┼─────────────────────────────────────────────
;     0.12 │    push   rbp
;     0.08 │    mov    rbp, rsp
;           │    ; ← loop starts here
;    45.32  │    vmovups  ymm0, [rdi + rax]    ← HOTSPOT (memory load)
;     1.23  │    vaddps   ymm1, ymm0, ymm2
;    38.67  │    vmovups  [rsi + rax], ymm1    ← HOTSPOT (memory store)
;     0.45  │    add      rax, 32
;     0.34  │    cmp      rax, rdx
;     0.01  │    jl       loop_top
;
; วิเคราะห์:
; - load และ store คิดเป็น ~84% ของ cycles
; - นี่คือ memory-bound loop
; - แนวทาง: prefetch, software prefetching, loop blocking
```

### 17.3 ใช้ objdump + perf script

```bash
# Disassemble function ที่สนใจ
objdump -d --no-show-raw-insn -M intel ./my_program | \
    awk '/^[0-9a-f]+ <my_hot_function>:/,/^$/' > disasm.txt

# รวมกับ perf data
perf script -F ip,sym,symoff > symbols.txt

# ดู instruction mix
perf stat -e fp_arith_inst_retired.scalar_double,\
             fp_arith_inst_retired.128b_packed_double,\
             fp_arith_inst_retired.256b_packed_double \
    ./my_program
```

---

## 18. llvm-mca — Static Latency Analysis

### 18.1 พื้นฐาน llvm-mca

`llvm-mca` วิเคราะห์ static performance ของ assembly code
โดยไม่ต้องรัน program จริง

```bash
# ติดตั้ง
apt install llvm  # หรือ build จาก source

# รัน llvm-mca
llvm-mca -mcpu=skylake my_loop.s

# ระบุ CPU target
llvm-mca -mcpu=znver3 my_loop.s      # AMD Zen 3
llvm-mca -mcpu=icelake-server my_loop.s
```

### 18.2 ตัวอย่าง llvm-mca Analysis

```nasm
; my_loop.s — สำหรับ llvm-mca
; GAS syntax (llvm-mca ใช้ GAS)

.intel_syntax noprefix

# LLVM-MCA-BEGIN my_add_loop
.loop_top:
    vmovups ymm0, [rdi + rax]
    vmovups ymm1, [rsi + rax]
    vaddps  ymm2, ymm0, ymm1
    vmovups [rdx + rax], ymm2
    add     rax, 32
    cmp     rax, rcx
    jl      .loop_top
# LLVM-MCA-END
```

```bash
# llvm-mca output:
#
# Iterations: 100
# Instructions: 700
# Total Cycles: 432
# Total uOps: 700
# Dispatch Width: 4
# uOps Per Cycle: 1.62
# IPC: 1.62
# Block RThroughput: 3.5
#
# Instruction Info:
# [1]: #uOps
# [2]: Latency
# [3]: RThroughput
# [4]: MayLoad
# [5]: MayStore
# [6]: HasSideEffects (U)
#
# [1]  [2]  [3]  [4]  [5]  [6]  Instructions:
#  1    7   0.50  *         vmovups  ymm0, [rdi + rax]
#  1    7   0.50  *         vmovups  ymm1, [rsi + rax]
#  1    4   0.50            vaddps   ymm2, ymm0, ymm1
#  1    7   0.50       *    vmovups  [rdx + rax], ymm2
#  1    1   0.25            add      rax, 32
#  1    1   0.25            cmp      rax, rcx
#  1    1   0.50            jl       .loop_top
```

### 18.3 Bottleneck Analysis ด้วย llvm-mca

```bash
# ดู bottleneck analysis
llvm-mca -mcpu=skylake -bottleneck-analysis my_loop.s

# ดู timeline view
llvm-mca -mcpu=skylake -timeline my_loop.s

# ดู resource pressure
llvm-mca -mcpu=skylake -resource-pressure my_loop.s
```

```
Resource Pressure Output:
[0] - SKLPort0
[1] - SKLPort1
[2] - SKLPort2
[3] - SKLPort3
[4] - SKLPort4
[5] - SKLPort5
[6] - SKLPort6
[7] - SKLPort7

Resource pressure per iteration:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]
0.50   0.50   1.00   1.00   1.00   0.50   0.25   0.25

ดู Port 2,3,4 saturated → memory-bound!
```

---

## 19. Agner Fog Tools และข้อมูล

### 19.1 Agner Fog Instruction Tables

Agner Fog รวบรวมข้อมูล latency/throughput ของ instructions สำหรับ CPU ทุก family
ที่ http://www.agner.org/optimize/

```
ตาราง Latency/Throughput (Intel Skylake ตัวอย่าง):
┌──────────────────┬─────────┬────────────┬──────────────────┐
│ Instruction      │ Latency │ Throughput │ Execution ports  │
├──────────────────┼─────────┼────────────┼──────────────────┤
│ ADD r64, r64     │    1    │    0.25    │ p0156            │
│ IMUL r64, r64    │    3    │    1.00    │ p1               │
│ IDIV r64         │  35-88  │   21-74    │ p0               │
│ VMOVUPS ymm,m    │    7    │    0.50    │ p23              │
│ VADDPS ymm,ymm   │    4    │    0.50    │ p01              │
│ VMULPS ymm,ymm   │    4    │    0.50    │ p01              │
│ VFMADD132PS      │    4    │    0.50    │ p01              │
│ VSQRTPS ymm      │   11-14 │    3-6     │ p0               │
│ VDIVPS ymm       │  10-14  │    5-14    │ p0               │
└──────────────────┴─────────┴────────────┴──────────────────┘
```

### 19.2 Agner Fog Optimization Manuals

```
Agner Fog Manuals (free, ที่ agner.org):
1. "Optimizing subroutines in assembly language"
   - How to write fast assembly
   - Instruction scheduling
   - CPU-specific tips

2. "The microarchitecture of Intel, AMD, and VIA CPUs"
   - Pipeline details
   - Execution units
   - Cache organization

3. "Instruction tables"
   - Latency, throughput ทุก instruction
   - ทุก CPU family

4. "Calling conventions and object file formats"
   - ABI details

5. "C++ Optimization"
   - High-level optimization
```

### 19.3 ใช้ข้อมูล Agner Fog ใน Code

```nasm
; ตัวอย่าง: เลือก instruction โดยดู latency/throughput
; (Intel Skylake)

; คำนวณ a*b + c
; Option 1: แยก MUL + ADD
vmulps  ymm0, ymm1, ymm2    ; latency 4, throughput 0.5
vaddps  ymm0, ymm0, ymm3    ; latency 4, throughput 0.5
; Total latency: 8 cycles (serial dependency)

; Option 2: FMA (Fused Multiply-Add)
vfmadd213ps ymm3, ymm1, ymm2  ; a*b + c
; latency 4, throughput 0.5
; Total latency: 4 cycles (1 instruction !)
; → FMA เร็วกว่า 2x ในแง่ latency

; คำนวณ integer division
; IDIV r64 = latency 35-88 cycles!
; ถ้าหารด้วย power of 2 → ใช้ SAR แทน
sar     rax, 3      ; หาร 8 (รวดเร็ว)
; latency 1 cycle vs 35-88 cycles
```

---

## 20. Statistical Significance ใน Benchmarking

### 20.1 ทำไม Statistics สำคัญ

```
ตัวอย่าง: 2 implementations A และ B
A measurements (cycles): 100, 102, 101, 105, 103
B measurements (cycles): 98, 110, 99, 115, 101

Mean A = 102.2
Mean B = 104.6

→ A เร็วกว่า B หรือเปล่า? ยังบอกไม่ได้!
ต้องดู variance ด้วย
```

```c
/* Simple t-test สำหรับ benchmark comparison */
#include <math.h>
#include <stdio.h>

typedef struct {
    double mean;
    double stddev;
    int    n;
} Stats;

Stats compute_stats(uint64_t *data, int n) {
    Stats s = {0};
    s.n = n;
    
    double sum = 0;
    for (int i = 0; i < n; i++) sum += data[i];
    s.mean = sum / n;
    
    double var = 0;
    for (int i = 0; i < n; i++) {
        double d = data[i] - s.mean;
        var += d * d;
    }
    s.stddev = sqrt(var / n);
    
    return s;
}

/* Welch's t-test: ทดสอบว่า 2 ค่า mean แตกต่างกันจริงหรือเปล่า */
double welch_t_statistic(Stats a, Stats b) {
    double se_a = (a.stddev * a.stddev) / a.n;
    double se_b = (b.stddev * b.stddev) / b.n;
    return (a.mean - b.mean) / sqrt(se_a + se_b);
}

void compare_benchmarks(
    uint64_t *a_data, int a_n,
    uint64_t *b_data, int b_n,
    const char *a_name, const char *b_name
) {
    Stats a = compute_stats(a_data, a_n);
    Stats b = compute_stats(b_data, b_n);
    
    printf("%s: mean=%.1f stddev=%.1f\n", a_name, a.mean, a.stddev);
    printf("%s: mean=%.1f stddev=%.1f\n", b_name, b.mean, b.stddev);
    
    double t = welch_t_statistic(a, b);
    printf("t-statistic: %.2f\n", t);
    
    /* |t| > 2.0 ≈ 95% confidence ว่าต่างกันจริง */
    if (fabs(t) > 2.0) {
        if (a.mean < b.mean) {
            printf("Result: %s เร็วกว่า %s อย่างมีนัยสำคัญ\n",
                   a_name, b_name);
        } else {
            printf("Result: %s เร็วกว่า %s อย่างมีนัยสำคัญ\n",
                   b_name, a_name);
        }
        printf("Speedup: %.2fx\n", b.mean / a.mean);
    } else {
        printf("Result: ไม่ต่างกันอย่างมีนัยสำคัญ\n");
    }
}
```

---

## 21. Advanced Profiling Techniques

### 21.1 Hardware Performance Counters โดยตรง

```c
/* ใช้ Linux perf_event_open syscall โดยตรง */
#include <linux/perf_event.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <sys/ioctl.h>

static long perf_event_open(
    struct perf_event_attr *hw_event,
    pid_t pid, int cpu, int group_fd, unsigned long flags
) {
    return syscall(__NR_perf_event_open, hw_event, pid, cpu,
                   group_fd, flags);
}

int setup_cycle_counter(void) {
    struct perf_event_attr pe = {0};
    pe.type           = PERF_TYPE_HARDWARE;
    pe.size           = sizeof(struct perf_event_attr);
    pe.config         = PERF_COUNT_HW_CPU_CYCLES;
    pe.disabled       = 1;
    pe.exclude_kernel = 1;
    pe.exclude_hv     = 1;
    
    int fd = perf_event_open(&pe, 0, -1, -1, 0);
    if (fd == -1) {
        perror("perf_event_open");
        return -1;
    }
    return fd;
}

uint64_t count_cycles(int fd, void (*fn)(void)) {
    long long count;
    
    ioctl(fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(fd, PERF_EVENT_IOC_ENABLE, 0);
    
    fn();
    
    ioctl(fd, PERF_EVENT_IOC_DISABLE, 0);
    read(fd, &count, sizeof(long long));
    
    return (uint64_t)count;
}
```

### 21.2 Intel PT — Processor Trace

```bash
# Intel PT: record เต็ม instruction trace
perf record -e intel_pt// -o pt.data ./my_program

# decode trace
perf script --itrace=i1000ns ./pt.data

# ดู instruction-accurate profile
perf report --itrace=i100000 --stdio
```

### 21.3 AMD uProf

```bash
# AMD uProf: equivalent ของ VTune สำหรับ AMD
# ดาวน์โหลดจาก developer.amd.com

AMDuProfCLI collect --config tbp --output-dir /tmp/prof -- ./my_program
AMDuProfCLI report --output-dir /tmp/prof
```

---

## 22. Benchmark ของจริง — Memory Bandwidth Test

```nasm
; NASM: วัด memory bandwidth
; rdi = source, rsi = dest, rdx = size_bytes

section .text
global memcpy_bench

memcpy_bench:
    push    rbx
    mov     rbx, rdx
    shr     rbx, 5          ; size / 32 = number of 256-bit chunks
    
.loop:
    vmovups     ymm0, [rdi]
    vmovntps    [rsi], ymm0     ; non-temporal store (bypass cache)
    add         rdi, 32
    add         rsi, 32
    dec         rbx
    jnz         .loop
    
    sfence                      ; flush non-temporal stores
    pop         rbx
    ret
```

```c
/* วัด bandwidth จาก C */
#include <time.h>

double measure_bandwidth_gbps(
    void *src, void *dst, size_t bytes,
    int iterations
) {
    struct timespec start, end;
    
    /* warmup */
    for (int i = 0; i < 3; i++) {
        memcpy_bench(dst, src, bytes);
    }
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < iterations; i++) {
        memcpy_bench(dst, src, bytes);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double ns = (end.tv_sec - start.tv_sec) * 1e9 +
                (end.tv_nsec - start.tv_nsec);
    double total_bytes = (double)bytes * iterations;
    
    return (total_bytes / ns); /* GB/s */
}
```

---

## 23. Profiling ใน Production Systems

### 23.1 Continuous Profiling

```bash
# ใช้ async-profiler สำหรับ Java (concept สำหรับ JVM)
# สำหรับ C/C++ ใช้ perf แบบ continuous

# เปิด perf ใน background
perf record -F 99 -g -p $(pgrep my_server) -o /tmp/profile &
sleep 60
kill %1
perf report --stdio -i /tmp/profile > /tmp/report.txt
```

### 23.2 eBPF Profiling

```bash
# BCC tools (eBPF-based, low overhead)
# CPU profiling
profile-bpfcc -F 99 30 > out.stacks
./flamegraph.pl out.stacks > cpu.svg

# Off-CPU profiling  
offcputime-bpfcc 30 > out.stacks
./flamegraph.pl --color=io out.stacks > offcpu.svg

# Memory allocation profiling
memleak-bpfcc -p $(pgrep my_program)
```

---

## 24. สรุป Performance Analysis Workflow

```
Complete Performance Analysis Workflow:

Step 1: Macro Profiling
━━━━━━━━━━━━━━━━━━━━━━
perf stat → ดู IPC, cache miss, branch miss
ถ้า IPC ต่ำ → ไปขั้น 2

Step 2: Hotspot Finding
━━━━━━━━━━━━━━━━━━━━━━
perf record + perf report → หา function ที่ใช้เวลามาก
Flamegraph → ดู call hierarchy

Step 3: Micro Analysis
━━━━━━━━━━━━━━━━━━━━━
perf annotate → ดู assembly hotspot
llvm-mca → static analysis
Agner Fog tables → ดู latency/throughput

Step 4: Root Cause
━━━━━━━━━━━━━━━━━
Cache miss → ปรับ data layout, prefetch
Branch miss → ใช้ CMOV, เรียง data
Long latency → FMA, SIMD, unrolling

Step 5: Validate
━━━━━━━━━━━━━━━━
RDTSC benchmark → วัด min cycles
Statistical test → ยืนยันว่าดีขึ้นจริง
```

---

## 25. Quick Reference

### 25.1 Measurement Checklist

```
Before Benchmarking:
□ ปิด CPU frequency scaling (performance governor)
□ ปิด Hyper-Threading (เพื่อ isolation)
□ ผูก process กับ CPU เดียว (taskset)
□ ปิด ASLR (สำหรับ reproducibility)
  echo 0 > /proc/sys/kernel/randomize_va_space
□ รัน ด้วย nice -n -20

During Benchmarking:
□ Warmup ก่อนวัด (อย่างน้อย 10 iterations)
□ วัดหลายๆ ครั้ง (อย่างน้อย 100)
□ ใช้ minimum (ไม่ใช่ average) สำหรับ best case
□ ใช้ median สำหรับ typical case
□ เช็ค CPU migration (RDTSCP ECX)

Reporting:
□ รายงาน min, median, stddev
□ ระบุ CPU model และ frequency
□ ระบุ compiler flags
□ ระบุ OS และ kernel version
```

### 25.2 perf Quick Reference

```bash
# ดู overview
perf stat ./prog

# หา hotspot
perf record -g ./prog && perf report

# ดู assembly hotspot
perf record ./prog && perf annotate

# วัด specific events
perf stat -e cycles,instructions,cache-misses,branch-misses ./prog

# Flamegraph
perf record -F 99 -g ./prog
perf script | stackcollapse-perf.pl | flamegraph.pl > fg.svg
```

### 25.3 llvm-mca Quick Reference

```bash
# วิเคราะห์ loop
llvm-mca -mcpu=skylake loop.s

# ดู resource pressure
llvm-mca -mcpu=skylake -resource-pressure loop.s

# Timeline view
llvm-mca -mcpu=skylake -timeline loop.s

# Common CPU targets
# Intel: broadwell, skylake, icelake-server, alderlake
# AMD: znver1, znver2, znver3, znver4
```

---

## 26. โจทย์ปฏิบัติ (Exercises)

### Exercise 1: RDTSC Benchmark

```
เขียน C function ที่:
1. วัด latency ของ sqrt(x) ด้วย RDTSC
2. วัด latency ของ sqrtf(x) ด้วย RDTSC
3. วัด latency ของ vsqrtss instruction ด้วย RDTSC
4. วัดด้วย minimum of 1000 runs
5. รายงาน cycles สำหรับแต่ละแบบ
```

### Exercise 2: perf stat Analysis

```
1. เขียน matrix multiply ขนาด 256x256
2. รัน: perf stat -e cycles,instructions,cache-misses ./matmul
3. คำนวณ IPC
4. ดู cache miss rate
5. Optimize ด้วย loop tiling
6. วัดใหม่และเปรียบเทียบ
```

### Exercise 3: Flamegraph

```
1. เขียน program ที่มีหลาย functions
2. สร้าง Flamegraph
3. ระบุ hotspot function
4. Optimize hotspot
5. สร้าง Flamegraph ใหม่และเปรียบเทียบ
```

### Exercise 4: llvm-mca

```
1. เขียน SIMD loop สำหรับ dot product
2. วิเคราะห์ด้วย llvm-mca
3. ดู resource pressure
4. ระบุ bottleneck
5. แก้ด้วย instruction scheduling
6. วิเคราะห์อีกครั้ง
```

---

## สรุปท้ายบท

การ Profile และวิเคราะห์ประสิทธิภาพเป็นทักษะสำคัญ:

1. **RDTSC/RDTSCP** ใช้สำหรับ low-overhead timing, ต้องมี CPUID serialize
2. **ใช้ Minimum** ไม่ใช่ Average สำหรับ best-case timing
3. **perf** เป็นเครื่องมือหลักบน Linux ครอบคลุมทุก level
4. **Flamegraph** ช่วยมองเห็น call hierarchy และ hotspots
5. **llvm-mca** วิเคราะห์ assembly แบบ static ก่อนรัน
6. **Agner Fog** เป็น reference สำคัญสำหรับ instruction latency
7. **Statistical rigor** สำคัญ — ต้องมี warmup, multiple runs, variance analysis
8. **Measurement artifacts** ต้องควบคุม: frequency scaling, NUMA, context switches

การ optimize โดยไม่มี profiling ก่อน = **เสียเวลาเปล่า**
วัดก่อน optimize เสมอ!

---

*Part 068 — Assembly Language Course*
*หัวข้อ: Profiling and Performance Analysis*
