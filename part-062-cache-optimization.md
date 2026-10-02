# Part 062: Cache Architecture และ Optimization

## บทนำ (Introduction)

Cache memory เป็นหนึ่งในปัจจัยสำคัญที่สุดที่ส่งผลต่อ performance ของโปรแกรม Modern CPU มี cache hierarchy หลายระดับ การเข้าใจว่า cache ทำงานอย่างไร และวิธีเขียนโค้ดให้ใช้ประโยชน์จาก cache ได้ดี จะช่วยให้โปรแกรมทำงานได้เร็วขึ้นอย่างมาก บางกรณีอาจเร็วขึ้นถึง 10-100x

**Memory Access Latency (approximate):**
```
Register:       0 cycles
L1 Cache:       4 cycles   (~1 ns)
L2 Cache:      12 cycles   (~3 ns)
L3 Cache:      40 cycles   (~10 ns)
Main Memory:  200 cycles   (~60 ns)
SSD:        ~100,000 ns
HDD:      ~5,000,000 ns
```

ความแตกต่างระหว่าง L1 cache hit กับ main memory access คือ ~50x ดังนั้นการเขียนโค้ดที่ cache-friendly จึงมีความสำคัญมาก

---

## 1. Cache Hierarchy

### 1.1 โครงสร้าง Cache ใน Modern CPU

```
CPU Core 0              CPU Core 1
┌─────────────────┐    ┌─────────────────┐
│  ┌───┐  ┌───┐  │    │  ┌───┐  ┌───┐  │
│  │L1I│  │L1D│  │    │  │L1I│  │L1D│  │
│  │32K│  │32K│  │    │  │32K│  │32K│  │
│  └───┘  └───┘  │    │  └───┘  └───┘  │
│  ┌─────────┐   │    │  ┌─────────┐   │
│  │   L2    │   │    │  │   L2    │   │
│  │  256KB  │   │    │  │  256KB  │   │
│  └─────────┘   │    │  └─────────┘   │
└────────┬────────┘    └────────┬────────┘
         │                     │
    ┌────┴─────────────────────┴────┐
    │           L3 Cache            │
    │          8MB - 32MB+          │
    │         (Shared LLC)          │
    └───────────────┬───────────────┘
                    │
    ┌───────────────┴───────────────┐
    │          Main Memory          │
    │      DDR4/DDR5 DRAM           │
    └───────────────────────────────┘
```

### 1.2 Cache ระดับต่างๆ

**L1 Cache (Level 1):**
- แยกเป็น L1 Instruction Cache (L1I) และ L1 Data Cache (L1D)
- ขนาด: 32KB - 64KB ต่อ cache
- Latency: 4-5 cycles
- Associativity: 8-way
- Private per core (แต่ละ core มีของตัวเอง)

**L2 Cache (Level 2):**
- Unified cache (ทั้ง instruction และ data)
- ขนาด: 256KB - 1MB
- Latency: 10-15 cycles
- Associativity: 8-way หรือ 16-way
- Private per core

**L3 Cache (Level 3 / LLC - Last Level Cache):**
- Shared ระหว่าง cores ทั้งหมด
- ขนาด: 8MB - 64MB+
- Latency: 30-50 cycles
- Associativity: 16-way หรือ higher
- Intel เรียกว่า "Shared LLC"

### 1.3 ตรวจสอบ Cache ของระบบ

```bash
# Linux - ดู cache info
cat /sys/devices/system/cpu/cpu0/cache/index0/size    # L1 Data
cat /sys/devices/system/cpu/cpu0/cache/index1/size    # L1 Instruction
cat /sys/devices/system/cpu/cpu0/cache/index2/size    # L2
cat /sys/devices/system/cpu/cpu0/cache/index3/size    # L3

# ดูรายละเอียด
for i in /sys/devices/system/cpu/cpu0/cache/index*/; do
    echo "Cache level $(cat $i/level) type $(cat $i/type): $(cat $i/size)"
done

# cpuid command
cpuid | grep -i cache

# lscpu
lscpu | grep -i cache
```

```bash
# Output ตัวอย่าง:
# Cache level 1 type Data: 32K
# Cache level 1 type Instruction: 32K
# Cache level 2 type Unified: 256K
# Cache level 3 type Unified: 8192K
```

---

## 2. Cache Line

### 2.1 Cache Line คืออะไร

Cache line เป็นหน่วยพื้นฐานของ cache memory ขนาด **64 bytes** สำหรับ x86-64 modern CPUs

เมื่อ CPU เข้าถึง memory แม้แต่ 1 byte, มันจะโหลด cache line ทั้ง 64 bytes เข้ามา

```
Memory Address Space:
┌──────────────────────────────────────────────────────────────┐
│ Byte 0   Byte 1   ... Byte 63 │ Byte 64  ... Byte 127        │
│ ←────── Cache Line 0 ────────→│←────── Cache Line 1 ────────→│
└──────────────────────────────────────────────────────────────┘
```

### 2.2 ทำไม Cache Line ถึงสำคัญ

**ตัวอย่าง: Array Traversal**

```c
// Array of integers: int arr[16]
// int = 4 bytes, 16 ints = 64 bytes = พอดี 1 cache line

int arr[16] = {1, 2, 3, ..., 16};

// การเข้าถึง arr[0] จะโหลด cache line ที่มี arr[0]-arr[15] ทั้งหมด
// ดังนั้น arr[1] ถึง arr[15] จะเป็น cache hit ทั้งหมด
```

**ตัวอย่างใน Assembly (NASM):**

```nasm
; cache_line_demo.asm
; สาธิตการทำงานของ cache line

section .data
    ; Array aligned to cache line boundary
    align 64
    arr: dd 1, 2, 3, 4, 5, 6, 7, 8
         dd 9, 10, 11, 12, 13, 14, 15, 16   ; 16 * 4 = 64 bytes = 1 cache line

section .text
global _start

_start:
    ; เข้าถึง arr[0] - โหลด cache line
    mov eax, [arr]          ; Cache MISS - โหลด 64 bytes (arr[0]-arr[15])
    
    ; เข้าถึง arr[4] - cache hit
    mov ebx, [arr + 16]     ; Cache HIT - อยู่ใน cache line เดียวกัน
    
    ; เข้าถึง arr[8] - cache hit
    mov ecx, [arr + 32]     ; Cache HIT
    
    ; เข้าถึง arr[15] - cache hit
    mov edx, [arr + 60]     ; Cache HIT
    
    ; Exit
    mov eax, 60
    xor edi, edi
    syscall
```

### 2.3 Cache Line Alignment

```nasm
; aligned_struct.asm
; ตัวอย่างการ align struct ให้ตรงกับ cache line

section .data

; BAD: struct ที่ไม่ได้ align - อาจข้าม 2 cache lines
bad_struct:
    db 0x00              ; 1 byte padding ทำให้ misalign
    counter: dq 0        ; 8 bytes counter
    value:   dd 0        ; 4 bytes value

; GOOD: struct ที่ align กับ cache line
align 64
good_struct:
    counter: dq 0        ; offset 0
    value:   dd 0        ; offset 8
    ; padding to fill cache line
    times 52 db 0        ; pad to 64 bytes

section .text
global _start

_start:
    ; Accessing aligned struct - guaranteed single cache line
    mov rax, [good_struct]      ; Load counter
    inc rax
    mov [good_struct], rax      ; Store counter
    
    mov eax, 60
    xor edi, edi
    syscall
```

### 2.4 CLFLUSH Instruction

```nasm
; clflush_demo.asm
; การใช้ CLFLUSH เพื่อ flush cache line กลับไป memory

section .data
    align 64
    data_buffer: times 64 db 0

section .text
global flush_cache_line

; flush_cache_line(void* addr)
; รับ address ใน rdi, flush cache line ที่มี address นั้น
flush_cache_line:
    clflush [rdi]        ; Flush cache line containing [rdi]
    ret

global flush_cache_line_opt

; flush_cache_line_opt(void* addr)  
; CLFLUSHOPT - optimized version, allows out-of-order execution
flush_cache_line_opt:
    clflushopt [rdi]     ; Optimized flush (weaker ordering)
    sfence               ; Ensure flush completes before continuing
    ret
```

---

## 3. Cache Associativity

### 3.1 Direct-Mapped Cache

ทุก memory address map ไปยัง cache set เดียวที่แน่นอน

```
Memory Address: [Tag | Set Index | Block Offset]

Example (4KB direct-mapped, 64-byte lines):
- Block Offset: bits [5:0]  (64 = 2^6, ต้องการ 6 bits)
- Set Index:    bits [11:6] (4KB/64B = 64 sets = 2^6, ต้องการ 6 bits)
- Tag:          bits [63:12] (remaining bits)

┌─────────────────────────────────────────────────────┐
│     Tag (52 bits)     │ Set Index (6 bits) │ Offset │
│                       │                   │(6 bits)│
└─────────────────────────────────────────────────────┘
```

**ข้อดี:** Simple, fast lookup
**ข้อเสีย:** Conflict misses สูง - สอง addresses ที่ต่างกัน 4KB จะแย่ง cache set เดียวกัน

```c
// ตัวอย่าง Conflict Miss ใน Direct-Mapped Cache
// ถ้า cache = 4KB, cache line = 64 bytes
// a[0] และ b[0] ที่ห่างกัน 4KB จะ map ไปยัง set เดียวกัน

int a[1024];  // อยู่ที่ 0x1000
int b[1024];  // อยู่ที่ 0x2000 (ห่างจาก a 4KB)

// การวนลูปนี้จะเกิด conflict miss ทุก iteration!
for (int i = 0; i < 1024; i++) {
    sum += a[i] + b[i];  // a[i] evicts b[i], b[i] evicts a[i]
}
```

### 3.2 Set-Associative Cache

```
N-Way Set Associative:
- Cache แบ่งเป็น sets
- แต่ละ set มี N ways (slots)
- Address map ไปยัง set เดียว แต่เลือก way ไหนก็ได้ใน set นั้น

4-Way Set Associative (4KB cache, 64B lines):
- 4KB / 64B / 4 ways = 16 sets
- Set Index: 4 bits (bits [9:6])
- Offset: 6 bits (bits [5:0])
- Tag: remaining bits

Set 0: [Way0] [Way1] [Way2] [Way3]
Set 1: [Way0] [Way1] [Way2] [Way3]
...
Set 15: [Way0] [Way1] [Way2] [Way3]
```

```
Address Breakdown สำหรับ 4-way set associative, 32KB L1D:
- 32KB / 64B line = 512 cache lines
- 512 / 4 ways = 128 sets

Bits:
[63:12] = Tag (52 bits)
[11:6]  = Set Index (7 bits, log2(128) = 7)
[5:0]   = Block Offset (6 bits, log2(64) = 6)
```

### 3.3 Fully Associative Cache

- Address map ไปยัง cache line ไหนก็ได้
- ใช้ใน TLB และ L1 cache บางรุ่น
- ข้อดี: ไม่มี conflict misses
- ข้อเสีย: ต้องค้นหาทุก cache line เมื่อ lookup (แก้ด้วย CAM - Content Addressable Memory)

### 3.4 การคำนวณ Set Index, Tag, Offset Bits

```
สูตร:
- Offset bits = log2(cache_line_size) = log2(64) = 6 bits
- Set index bits = log2(num_sets) = log2(cache_size / cache_line_size / associativity)
- Tag bits = address_bits - set_index_bits - offset_bits

ตัวอย่าง: Intel Core i7 L1D Cache
- Size: 32KB
- Line size: 64 bytes
- Associativity: 8-way

Calculation:
- Offset = log2(64) = 6 bits
- Sets = 32768 / 64 / 8 = 64 sets
- Set index = log2(64) = 6 bits
- Tag = 64 - 6 - 6 = 52 bits (for 64-bit address)
```

```nasm
; cache_index_calc.asm
; คำนวณ cache index สำหรับ address ที่กำหนด

; สมมติ L1D: 32KB, 64B line, 8-way → 64 sets
; Offset bits: 6, Set bits: 6, Tag bits: 52

%define OFFSET_BITS   6
%define SET_BITS      6
%define OFFSET_MASK   ((1 << OFFSET_BITS) - 1)    ; 0x3F
%define SET_MASK      ((1 << SET_BITS) - 1)        ; 0x3F

section .text
global get_cache_set_index

; rdi = memory address
; returns rax = set index
get_cache_set_index:
    mov rax, rdi
    shr rax, OFFSET_BITS        ; Remove offset bits
    and rax, SET_MASK            ; Extract set index bits
    ret

global get_cache_tag

; rdi = memory address
; returns rax = tag
get_cache_tag:
    mov rax, rdi
    shr rax, (OFFSET_BITS + SET_BITS)  ; Remove offset and set bits
    ret

global get_block_offset

; rdi = memory address  
; returns rax = block offset
get_block_offset:
    mov rax, rdi
    and rax, OFFSET_MASK         ; Extract offset bits
    ret
```

---

## 4. Cache Miss Types

### 4.1 Cold Miss / Compulsory Miss (First Access)

เกิดเมื่อเข้าถึง memory address เป็นครั้งแรก ไม่สามารถหลีกเลี่ยงได้ (เว้นแต่ใช้ prefetch)

```c
// Cold miss example
int arr[1000000];  // ยังไม่เคย access

// First pass - cold misses
for (int i = 0; i < 1000000; i++) {
    arr[i] = i;    // Cold miss ทุก 16 elements (64 bytes / 4 bytes per int)
}

// Second pass - warm hits
for (int i = 0; i < 1000000; i++) {
    sum += arr[i]; // Cache hits (ถ้า array พอดีกับ cache)
}
```

### 4.2 Capacity Miss (Working Set Too Large)

เกิดเมื่อ working set ใหญ่กว่า cache ไม่สามารถเก็บ data ทั้งหมดที่ต้องการได้

```c
// Capacity miss example
// L3 cache = 8MB
// Array = 32MB → ไม่พอดีกับ cache

double big_array[4000000];  // 32MB - ใหญ่กว่า L3

// วนลูปซ้ำ - แต่ละ pass เจอ capacity misses
for (int pass = 0; pass < 100; pass++) {
    for (int i = 0; i < 4000000; i++) {
        big_array[i] *= 2.0;  // Capacity misses เพราะ array ใหญ่กว่า cache
    }
}
```

### 4.3 Conflict Miss (Cache Thrashing)

เกิดใน direct-mapped หรือ low-associativity cache เมื่อสอง addresses map ไปยัง cache set เดียวกัน

```c
// Conflict miss example
// สมมติ 4-way 32KB L1D cache
// 32KB / 64B / 4 = 128 sets
// Stride ที่เกิด conflict = 128 * 64B = 8192 bytes = 8KB

double a[512];  // 4KB
double b[512];  // 4KB - อยู่ห่างจาก a พอดี 4KB? อาจเกิด conflict

// ถ้า &a[0] % 8192 == &b[0] % 8192 → conflict!
for (int i = 0; i < 512; i++) {
    sum += a[i] * b[i];  // อาจเกิด conflict misses
}

// FIX: เพิ่ม padding ระหว่าง arrays
double a_padded[512 + 16];  // เพิ่ม 128 bytes padding
double* a = a_padded;
double* b = a + 512 + 16;  // ทำให้ไม่ตรงกับ conflict address
```

---

## 5. False Sharing และ True Sharing

### 5.1 True Sharing

หลาย threads เข้าถึง data เดียวกัน ซึ่งเป็น cache coherence protocol ปกติ

```c
// True sharing - threads แย่งกัน modify ตัวแปรเดียวกัน
int shared_counter = 0;  // ทุก thread write ตัวแปรนี้

void increment_counter(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&shared_counter, 1);  // True sharing - ต้องการ sync
    }
}
```

### 5.2 False Sharing

หลาย threads เข้าถึง data คนละชิ้น แต่อยู่ใน cache line เดียวกัน ทำให้ cache line invalidation เกิดขึ้นโดยไม่จำเป็น

```c
// FALSE SHARING - Performance killer!
struct SharedData {
    int counter_a;    // Thread 0 writes this
    int counter_b;    // Thread 1 writes this
};
// counter_a และ counter_b อยู่ใน cache line เดียวกัน!
// เมื่อ Thread 0 write counter_a, Thread 1's cache line invalidated
// เมื่อ Thread 1 write counter_b, Thread 0's cache line invalidated

// ↑ แม้ทั้งสอง threads เขียน data คนละตัว แต่ก็ bounce cache line กันอยู่ตลอด
```

```c
// FIX: Pad struct ให้แต่ละ field อยู่ cache line แยกกัน
#define CACHE_LINE_SIZE 64

struct PaddedCounter {
    int counter;
    char padding[CACHE_LINE_SIZE - sizeof(int)];  // Pad to 64 bytes
};

struct PaddedCounter counter_a;  // อยู่ cache line ต่างกัน
struct PaddedCounter counter_b;  // อยู่ cache line ต่างกัน
```

**Assembly Example - False Sharing:**

```nasm
; false_sharing.asm
; สาธิต false sharing และวิธีแก้ไข

section .data

; BAD: data อยู่ cache line เดียวกัน
bad_data:
    thread0_counter: dq 0    ; offset 0
    thread1_counter: dq 0    ; offset 8 - same cache line as above!

; GOOD: แต่ละ counter อยู่ cache line ต่างกัน
align 64
good_counter0: dq 0          ; Cache line 0
times 56 db 0                ; Padding to fill 64 bytes

align 64
good_counter1: dq 0          ; Cache line 1 (separate!)
times 56 db 0                ; Padding

section .text
global increment_good_counter0

; Thread 0 increments counter0
increment_good_counter0:
    lock inc qword [good_counter0]   ; Modify only cache line 0
    ret

global increment_good_counter1

; Thread 1 increments counter1
increment_good_counter1:
    lock inc qword [good_counter1]   ; Modify only cache line 1 (no false sharing!)
    ret
```

### 5.3 ตรวจจับ False Sharing ด้วย perf

```bash
# ตรวจจับ cache coherence issues
perf stat -e cache-misses,cache-references,\
    mem_load_retired.l1_miss,mem_load_retired.l2_miss \
    ./your_program

# Linux perf สำหรับ false sharing
perf c2c record -- ./your_program
perf c2c report
```

---

## 6. Spatial Locality และ Temporal Locality

### 6.1 Spatial Locality

เข้าถึง data ที่อยู่ใกล้กันใน memory (ใช้ประโยชน์จาก cache line)

**การเข้าถึง Array แบบ Sequential (ดี):**

```nasm
; spatial_locality.asm
; Sequential access - spatial locality ดี

section .data
    N equ 1000000
    array: times N dd 0

section .text
global sum_array_sequential

; sum_array_sequential(int* arr, int n)
; rdi = arr, rsi = n
sum_array_sequential:
    xor eax, eax        ; sum = 0
    xor rcx, rcx        ; i = 0
    
.loop:
    cmp rcx, rsi
    jge .done
    
    add eax, [rdi + rcx*4]  ; arr[i] - sequential access!
    inc rcx
    jmp .loop
    
.done:
    ret

; WORSE: Strided access - poor spatial locality
; sum_array_strided(int* arr, int n, int stride)
; rdi = arr, rsi = n, rdx = stride
global sum_array_strided
sum_array_strided:
    xor eax, eax        ; sum = 0
    xor rcx, rcx        ; i = 0
    
.loop:
    cmp rcx, rsi
    jge .done
    
    add eax, [rdi + rcx*4]  ; arr[i*stride] - strided, worse cache behavior
    add rcx, rdx             ; i += stride
    jmp .loop
    
.done:
    ret
```

**Row-major vs Column-major Access:**

```nasm
; row_vs_col.asm
; ตัวอย่าง row-major (good) vs column-major (bad) สำหรับ matrix

; C-style arrays are row-major:
; matrix[row][col] อยู่ใน memory ตาม row
; matrix[0][0], matrix[0][1], ..., matrix[0][N-1], matrix[1][0], ...

%define N 1000
%define MATRIX_SIZE (N * N * 4)  ; N*N ints

section .bss
    matrix: resd N * N

section .text

; row_major_sum: access matrix[i][j] for i=0..N-1, j=0..N-1
; เดิน row by row - GOOD spatial locality
global row_major_sum
row_major_sum:
    push rbx
    push r12
    
    xor eax, eax        ; sum = 0
    xor rbx, rbx        ; i = 0
    
.outer:
    cmp rbx, N
    jge .done
    
    xor r12, r12        ; j = 0
.inner:
    cmp r12, N
    jge .next_row
    
    ; Access matrix[i][j] = matrix[i*N + j]
    mov rcx, rbx
    imul rcx, N
    add rcx, r12
    add eax, [matrix + rcx*4]   ; Sequential in memory!
    
    inc r12
    jmp .inner
    
.next_row:
    inc rbx
    jmp .outer
    
.done:
    pop r12
    pop rbx
    ret

; col_major_sum: access matrix[j][i] for i=0..N-1, j=0..N-1
; เดิน column by column - BAD spatial locality
global col_major_sum
col_major_sum:
    push rbx
    push r12
    
    xor eax, eax        ; sum = 0
    xor rbx, rbx        ; i = 0
    
.outer:
    cmp rbx, N
    jge .done
    
    xor r12, r12        ; j = 0
.inner:
    cmp r12, N
    jge .next_col
    
    ; Access matrix[j][i] - JUMPING N elements at a time
    mov rcx, r12        ; j
    imul rcx, N
    add rcx, rbx        ; j*N + i
    add eax, [matrix + rcx*4]   ; NOT sequential - cache miss every time!
    
    inc r12
    jmp .inner
    
.next_col:
    inc rbx
    jmp .outer
    
.done:
    pop r12
    pop rbx
    ret
```

### 6.2 Temporal Locality

เข้าถึง data ซ้ำๆ ก่อนที่จะถูก evict ออกจาก cache

```nasm
; temporal_locality.asm
; ตัวอย่าง temporal locality

section .data
    hot_var: dq 0           ; Variable accessed frequently

section .text
global compute_with_locality

; Function ที่ reuse hot_var หลายครั้ง
compute_with_locality:
    push rbx
    
    mov rax, [hot_var]      ; Load once (cache miss)
    
    ; Reuse rax ใน register แทนที่จะ load จาก memory ซ้ำๆ
    ; นี่คือ temporal locality ที่ดีที่สุด - ใช้ register!
    imul rax, rax
    add rax, 42
    imul rax, 3
    sub rax, 7
    
    mov [hot_var], rax      ; Store once
    
    pop rbx
    ret
```

---

## 7. Matrix Multiplication Cache Analysis

### 7.1 ijk Order (Standard)

```c
// ijk order - standard matrix multiply
// A[M][K] * B[K][N] = C[M][N]
void matmul_ijk(double* A, double* B, double* C, int M, int K, int N) {
    for (int i = 0; i < M; i++) {
        for (int j = 0; j < N; j++) {
            double sum = 0.0;
            for (int k = 0; k < K; k++) {
                sum += A[i*K + k] * B[k*N + j];
            }
            C[i*N + j] = sum;
        }
    }
}
```

**Cache Analysis ของ ijk:**
```
Inner loop (k): A[i][k] - sequential ✓ (good)
               B[k][j] - stride N (bad!) - column access

Access pattern:
- A: เดินตาม row → sequential, cache-friendly ✓
- B: เดินตาม column → stride N, cache-unfriendly ✗
- C: เขียนครั้งเดียวต่อ (i,j) → OK
```

### 7.2 ikj Order (Cache-Friendly)

```c
// ikj order - better cache behavior
void matmul_ikj(double* A, double* B, double* C, int M, int K, int N) {
    for (int i = 0; i < M; i++) {
        for (int k = 0; k < K; k++) {
            double a_ik = A[i*K + k];  // Load once, reuse
            for (int j = 0; j < N; j++) {
                C[i*N + j] += a_ik * B[k*N + j];  // B: sequential ✓
            }
        }
    }
}
```

**Cache Analysis ของ ikj:**
```
Inner loop (j): A[i][k] - loaded once, kept in register ✓✓
               B[k][j] - sequential ✓✓
               C[i][j] - sequential ✓✓

ดีกว่า ijk มาก เพราะทั้ง B และ C access แบบ sequential
```

**Benchmark comparison (approximate):**
```
Matrix 1000x1000, double precision:
ijk order: ~15 seconds (many cache misses on B)
ikj order:  ~2 seconds (sequential access on B and C)
Blocked:  ~0.5 seconds (see section 8)
```

### 7.3 Assembly Implementation - ikj Order

```nasm
; matmul_ikj.asm
; Cache-friendly matrix multiplication (ikj order)
; void matmul_ikj(double* A, double* B, double* C, int M, int K, int N)
; rdi = A, rsi = B, rdx = C, rcx = M, r8 = K, r9 = N

section .text
global matmul_ikj

matmul_ikj:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    ; Save parameters
    mov r10, rdi        ; A
    mov r11, rsi        ; B
    mov r12, rdx        ; C
    ; M in rcx, K in r8, N in r9
    
    xor r13, r13        ; i = 0
    
.loop_i:
    cmp r13, rcx        ; i < M?
    jge .done
    
    xor r14, r14        ; k = 0
    
.loop_k:
    cmp r14, r8         ; k < K?
    jge .next_i
    
    ; Load A[i][k] into xmm0 - use once for all j
    ; A[i*K + k] * 8 bytes
    mov rax, r13        ; i
    imul rax, r8        ; i*K
    add rax, r14        ; i*K + k
    vmovsd xmm0, [r10 + rax*8]   ; a_ik = A[i][k]
    
    xor r15, r15        ; j = 0
    
.loop_j:
    cmp r15, r9         ; j < N?
    jge .next_k
    
    ; C[i][j] += a_ik * B[k][j]
    ; C index: i*N + j
    mov rax, r13        ; i
    imul rax, r9        ; i*N
    add rax, r15        ; i*N + j
    
    ; B index: k*N + j
    mov rbx, r14        ; k
    imul rbx, r9        ; k*N
    add rbx, r15        ; k*N + j
    
    vmovsd xmm1, [r11 + rbx*8]   ; B[k][j]
    vmulsd xmm1, xmm1, xmm0      ; a_ik * B[k][j]
    
    vmovsd xmm2, [r12 + rax*8]   ; C[i][j]
    vaddsd xmm2, xmm2, xmm1      ; C[i][j] += ...
    vmovsd [r12 + rax*8], xmm2   ; Store back
    
    inc r15
    jmp .loop_j
    
.next_k:
    inc r14
    jmp .loop_k
    
.next_i:
    inc r13
    jmp .loop_i
    
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 8. Loop Tiling / Blocking

### 8.1 แนวคิดของ Loop Tiling

Loop tiling (หรือ loop blocking) คือการแบ่งการคำนวณออกเป็น "blocks" ที่พอดีกับ cache เพื่อเพิ่ม temporal locality

```
ปกติ: Process entire matrix row by row
      → Working set > L2 cache → L2 misses

Tiled: Process B×B block at a time
       → B×B block fits in L2 cache → Better reuse
```

### 8.2 Tiled Matrix Multiplication

```c
// Blocked/Tiled matrix multiply
#define BLOCK_SIZE 64  // Tune for your cache size

void matmul_tiled(double* A, double* B, double* C, int N) {
    int bs = BLOCK_SIZE;
    
    for (int ii = 0; ii < N; ii += bs) {
        for (int jj = 0; jj < N; jj += bs) {
            for (int kk = 0; kk < N; kk += bs) {
                // Process B×B tile
                for (int i = ii; i < min(ii+bs, N); i++) {
                    for (int k = kk; k < min(kk+bs, N); k++) {
                        double a_ik = A[i*N + k];
                        for (int j = jj; j < min(jj+bs, N); j++) {
                            C[i*N + j] += a_ik * B[k*N + j];
                        }
                    }
                }
            }
        }
    }
}
```

### 8.3 Block Size Calculation

```
Block size calculation:
- ต้องการให้ 3 blocks (A-tile, B-tile, C-tile) พอดีกับ L2 cache
- L2 cache = 256KB
- แต่ละ block = B × B doubles = B² × 8 bytes
- 3 blocks = 3 × B² × 8 bytes ≤ 256KB × (2/3) (ใช้แค่ 2/3 เพื่อเผื่อ)
- B² ≤ 256*1024*2/3/3/8 = 7281
- B ≤ 85

ในทางปฏิบัติ ใช้ B = 64 (power of 2, ง่ายต่อ alignment)
```

### 8.4 Assembly - Tiled Matrix Multiply

```nasm
; tiled_matmul.asm
; Tiled/blocked matrix multiplication
; void tiled_matmul(double* A, double* B, double* C, int N, int block)

%define BLOCK 64

section .text
global tiled_matmul

; rdi = A, rsi = B, rdx = C, rcx = N
tiled_matmul:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 8         ; Align stack
    
    ; Parameters: rdi=A, rsi=B, rdx=C, rcx=N
    ; Save in callee-saved registers
    mov r8, rdi        ; A
    mov r9, rsi        ; B
    mov r10, rdx       ; C
    mov r11, rcx       ; N
    
    ; Outer tiling loops: ii, jj, kk
    xor r12, r12       ; ii = 0
    
.loop_ii:
    cmp r12, r11       ; ii < N?
    jge .done
    
    xor r13, r13       ; jj = 0
    
.loop_jj:
    cmp r13, r11       ; jj < N?
    jge .next_ii
    
    xor r14, r14       ; kk = 0
    
.loop_kk:
    cmp r14, r11       ; kk < N?
    jge .next_jj
    
    ; Inner loops: i, k, j within tile
    mov rbx, r12       ; i = ii
    
.loop_i:
    ; min(ii + BLOCK, N)
    mov rcx, r12
    add rcx, BLOCK
    cmp rcx, r11
    cmovg rcx, r11
    cmp rbx, rcx
    jge .next_kk
    
    mov rbp, r14       ; k = kk
    
.loop_k:
    ; min(kk + BLOCK, N)
    mov rax, r14
    add rax, BLOCK
    cmp rax, r11
    cmovg rax, r11
    cmp rbp, rax
    jge .next_i
    
    ; Load A[i][k] into xmm0
    mov rax, rbx        ; i
    imul rax, r11       ; i*N
    add rax, rbp        ; i*N + k
    vmovsd xmm0, [r8 + rax*8]   ; a_ik
    
    mov r15, r13        ; j = jj
    
.loop_j:
    ; min(jj + BLOCK, N)
    mov rax, r13
    add rax, BLOCK
    cmp rax, r11
    cmovg rax, r11
    cmp r15, rax
    jge .next_k
    
    ; B[k][j]
    mov rcx, rbp        ; k
    imul rcx, r11       ; k*N
    add rcx, r15        ; k*N + j
    vmovsd xmm1, [r9 + rcx*8]
    
    vmulsd xmm1, xmm1, xmm0   ; a_ik * B[k][j]
    
    ; C[i][j]
    mov rcx, rbx        ; i
    imul rcx, r11       ; i*N
    add rcx, r15        ; i*N + j
    vmovsd xmm2, [r10 + rcx*8]
    vaddsd xmm2, xmm2, xmm1
    vmovsd [r10 + rcx*8], xmm2
    
    inc r15
    jmp .loop_j
    
.next_k:
    inc rbp
    jmp .loop_k
    
.next_i:
    inc rbx
    jmp .loop_i
    
.next_kk:
    add r14, BLOCK
    jmp .loop_kk
    
.next_jj:
    add r13, BLOCK
    jmp .loop_jj
    
.next_ii:
    add r12, BLOCK
    jmp .loop_ii
    
.done:
    add rsp, 8
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 9. Cache-Oblivious Algorithms

### 9.1 แนวคิด Cache-Oblivious

Cache-oblivious algorithms ทำงานได้ดีกับ cache ทุกขนาด โดยไม่ต้องรู้ขนาด cache จริงๆ ใช้ divide-and-conquer recursively

### 9.2 Cache-Oblivious Matrix Multiply

```c
// Cache-oblivious matrix multiply
// ใช้ recursive subdivision แทน explicit blocking
void co_matmul(double* A, double* B, double* C,
               int i0, int i1, int j0, int j1, int k0, int k1,
               int N) {
    int di = i1 - i0, dj = j1 - j0, dk = k1 - k0;
    
    // Base case: small enough to compute directly
    if (di + dj + dk <= 48) {  // Tunable threshold
        for (int i = i0; i < i1; i++) {
            for (int k = k0; k < k1; k++) {
                double a_ik = A[i*N + k];
                for (int j = j0; j < j1; j++) {
                    C[i*N + j] += a_ik * B[k*N + j];
                }
            }
        }
        return;
    }
    
    // Divide along longest dimension
    if (di >= dj && di >= dk) {
        int mid = (i0 + i1) / 2;
        co_matmul(A, B, C, i0, mid, j0, j1, k0, k1, N);
        co_matmul(A, B, C, mid, i1, j0, j1, k0, k1, N);
    } else if (dj >= di && dj >= dk) {
        int mid = (j0 + j1) / 2;
        co_matmul(A, B, C, i0, i1, j0, mid, k0, k1, N);
        co_matmul(A, B, C, i0, i1, mid, j1, k0, k1, N);
    } else {
        int mid = (k0 + k1) / 2;
        co_matmul(A, B, C, i0, i1, j0, j1, k0, mid, N);
        co_matmul(A, B, C, i0, i1, j0, j1, mid, k1, N);
    }
}
```

---

## 10. Matrix Transpose (Cache-Friendly)

### 10.1 Naive Transpose

```nasm
; naive_transpose.asm
; Naive matrix transpose - poor cache behavior

; void naive_transpose(double* src, double* dst, int N)
; rdi = src, rsi = dst, rcx = N

section .text
global naive_transpose

naive_transpose:
    push rbx
    push r12
    
    mov r8, rcx         ; N
    xor rbx, rbx        ; i = 0
    
.loop_i:
    cmp rbx, r8
    jge .done
    
    xor r12, r12        ; j = 0
    
.loop_j:
    cmp r12, r8
    jge .next_i
    
    ; dst[j][i] = src[i][j]
    ; src index: i*N + j
    mov rax, rbx
    imul rax, r8
    add rax, r12
    vmovsd xmm0, [rdi + rax*8]  ; src[i][j]
    
    ; dst index: j*N + i
    mov rax, r12
    imul rax, r8
    add rax, rbx
    vmovsd [rsi + rax*8], xmm0  ; dst[j][i]
    
    ; NOTE: dst access is strided by N! Poor spatial locality
    
    inc r12
    jmp .loop_j
    
.next_i:
    inc rbx
    jmp .loop_i
    
.done:
    pop r12
    pop rbx
    ret
```

### 10.2 Blocked/Tiled Transpose (Cache-Friendly)

```nasm
; tiled_transpose.asm
; Cache-friendly tiled matrix transpose

%define TILE_SIZE 32    ; 32×32 doubles = 8KB, fits in L1D

; void tiled_transpose(double* src, double* dst, int N)
; rdi = src, rsi = dst, rdx = N

section .text
global tiled_transpose

tiled_transpose:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 8
    
    mov r8, rdi         ; src
    mov r9, rsi         ; dst
    mov r10, rdx        ; N
    
    ; Tiling loops
    xor r11, r11        ; ii = 0
    
.loop_ii:
    cmp r11, r10
    jge .done
    
    xor r12, r12        ; jj = 0
    
.loop_jj:
    cmp r12, r10
    jge .next_ii
    
    ; Process tile [ii:ii+TILE_SIZE, jj:jj+TILE_SIZE]
    mov r13, r11        ; i = ii
    
.loop_i:
    ; min(ii + TILE_SIZE, N)
    mov rax, r11
    add rax, TILE_SIZE
    cmp rax, r10
    cmovg rax, r10
    cmp r13, rax
    jge .next_jj
    
    mov r14, r12        ; j = jj
    
.loop_j:
    ; min(jj + TILE_SIZE, N)
    mov rbx, r12
    add rbx, TILE_SIZE
    cmp rbx, r10
    cmovg rbx, r10
    cmp r14, rbx
    jge .next_i
    
    ; dst[j][i] = src[i][j]
    mov rax, r13        ; i
    imul rax, r10       ; i*N
    add rax, r14        ; i*N + j
    vmovsd xmm0, [r8 + rax*8]  ; src[i][j]
    
    mov rax, r14        ; j
    imul rax, r10       ; j*N
    add rax, r13        ; j*N + i
    vmovsd [r9 + rax*8], xmm0   ; dst[j][i]
    
    inc r14
    jmp .loop_j
    
.next_i:
    inc r13
    jmp .loop_i
    
.next_jj:
    add r12, TILE_SIZE
    jmp .loop_jj
    
.next_ii:
    add r11, TILE_SIZE
    jmp .loop_ii
    
.done:
    add rsp, 8
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

### 10.3 AVX2 Tiled Transpose

```nasm
; avx2_tiled_transpose.asm
; Tiled transpose ใช้ AVX2 สำหรับ 4×4 double block

; 4×4 matrix transpose ด้วย AVX2
; Input: 4 ymm registers (each holds 4 doubles = 1 row)
; Output: transposed 4 rows
%macro transpose4x4_avx2 4
    ; %1-%4: ymm registers holding 4 rows
    ; Result overwrites %1-%4 with transposed data
    
    vunpcklpd %1, %1, %2       ; [a0,b0,a2,b2]
    vunpckhpd %2, %1, %2       ; [a1,b1,a3,b3] -- wait, this is simplified
    ; ... (full implementation requires careful register management)
%endmacro

section .text
global transpose_4x4_double

; Transpose a 4×4 block of doubles
; rdi = src, rsi = dst, rdx = src_stride (in doubles), rcx = dst_stride
transpose_4x4_double:
    ; Load 4 rows of src
    vmovupd ymm0, [rdi]                      ; Row 0: a0 a1 a2 a3
    vmovupd ymm1, [rdi + rdx*8]              ; Row 1: b0 b1 b2 b3
    lea rax, [rdi + rdx*8]
    vmovupd ymm2, [rax + rdx*8]              ; Row 2: c0 c1 c2 c3
    lea rax, [rax + rdx*8]
    vmovupd ymm3, [rax + rdx*8]              ; Row 3: d0 d1 d2 d3
    
    ; Transpose using unpack
    vunpcklpd ymm4, ymm0, ymm1   ; a0 b0 a2 b2
    vunpckhpd ymm5, ymm0, ymm1   ; a1 b1 a3 b3
    vunpcklpd ymm6, ymm2, ymm3   ; c0 d0 c2 d2
    vunpckhpd ymm7, ymm2, ymm3   ; c1 d1 c3 d3
    
    vperm2f128 ymm0, ymm4, ymm6, 0x20   ; a0 b0 c0 d0 (col 0)
    vperm2f128 ymm1, ymm5, ymm7, 0x20   ; a1 b1 c1 d1 (col 1)
    vperm2f128 ymm2, ymm4, ymm6, 0x31   ; a2 b2 c2 d2 (col 2)
    vperm2f128 ymm3, ymm5, ymm7, 0x31   ; a3 b3 c3 d3 (col 3)
    
    ; Store transposed rows to dst
    vmovupd [rsi], ymm0
    vmovupd [rsi + rcx*8], ymm1
    lea rax, [rsi + rcx*8]
    vmovupd [rax + rcx*8], ymm2
    lea rax, [rax + rcx*8]
    vmovupd [rax + rcx*8], ymm3
    
    vzeroupper
    ret
```

---

## 11. Prefetch Instructions

### 11.1 Software Prefetch คืออะไร

Prefetch คือการบอก CPU ล่วงหน้าว่าจะต้องการ data ชุดไหน เพื่อให้ CPU โหลด data จาก memory เข้า cache ก่อนที่จะถึงเวลาใช้งาน

```
Timeline without prefetch:
t=0:  Load A[i]        → MISS, wait 200 cycles
t=200: Process A[i]
t=201: Load A[i+16]    → MISS, wait 200 cycles
t=401: Process A[i+16]
...

Timeline with prefetch:
t=0:  Prefetch A[i+16]  → Start loading in background
t=1:  Load A[i]          → MISS, wait 200 cycles
t=200: Process A[i]      → A[i+16] already loaded!
t=201: Load A[i+16]      → HIT! (was prefetched)
t=202: Process A[i+16]
...
```

### 11.2 Prefetch Instructions

```nasm
; prefetch_instructions.asm
; สาธิตการใช้ prefetch instructions

section .text
global prefetch_demo

; PREFETCHT0: Load to L1, L2, L3 cache
; PREFETCHT1: Load to L2, L3 cache (not L1)
; PREFETCHT2: Load to L3 cache only
; PREFETCHNTA: Non-temporal, bypass cache (for streaming data)

prefetch_t0_example:
    ; rdi = pointer to data
    prefetcht0 [rdi]        ; Prefetch to all cache levels (L1/L2/L3)
    ret

prefetch_t1_example:
    prefetcht1 [rdi]        ; Prefetch to L2 and L3, not L1
    ret

prefetch_t2_example:
    prefetcht2 [rdi]        ; Prefetch to L3 only
    ret

prefetch_nta_example:
    prefetchnta [rdi]       ; Non-temporal: bypass cache hierarchy
                             ; Good for streaming large data you'll use once
    ret
```

### 11.3 Software Prefetch Distance Calculation

```
Prefetch Distance Formula:
distance = latency_of_target_cache / time_per_iteration

Example:
- Main memory latency: 200 cycles
- Loop body takes: 10 cycles per iteration
- Prefetch distance: 200/10 = 20 iterations ahead

If accessing array with stride 64 bytes (1 cache line):
- Prefetch 20 * 64 = 1280 bytes = 20 cache lines ahead
```

```nasm
; prefetch_loop.asm
; การใช้ prefetch ใน loop

%define PREFETCH_DISTANCE 20    ; prefetch 20 iterations ahead
%define ELEM_SIZE 8              ; double = 8 bytes

section .text
global sum_with_prefetch

; double sum_with_prefetch(double* arr, int n)
; rdi = arr, rsi = n
sum_with_prefetch:
    vxorpd xmm0, xmm0, xmm0    ; sum = 0.0
    xor rcx, rcx                 ; i = 0
    
.loop:
    cmp rcx, rsi
    jge .done
    
    ; Prefetch data we'll need PREFETCH_DISTANCE iterations later
    lea rax, [rdi + rcx*ELEM_SIZE + PREFETCH_DISTANCE*ELEM_SIZE]
    prefetcht0 [rax]             ; เตรียม arr[i+20] ใน cache
    
    ; Process current element
    vaddsd xmm0, xmm0, [rdi + rcx*ELEM_SIZE]  ; sum += arr[i]
    
    inc rcx
    jmp .loop
    
.done:
    ; Return in xmm0 (double return value)
    ret
```

### 11.4 Prefetch ใน Loop ที่มี Multiple Arrays

```nasm
; multi_array_prefetch.asm
; Prefetch สำหรับหลาย arrays พร้อมกัน

%define PD 16   ; Prefetch Distance

section .text
global saxpy_prefetch

; void saxpy_prefetch(float* y, float* x, float a, int n)
; y[i] = a*x[i] + y[i]
; rdi = y, rsi = x, xmm0 = a, rdx = n
saxpy_prefetch:
    xor rcx, rcx        ; i = 0
    vbroadcastss ymm1, xmm0   ; Broadcast a to all lanes
    
    ; Main loop
.loop:
    lea rax, rcx
    add rax, PD         ; i + PD
    
    ; Prefetch both arrays
    prefetcht0 [rdi + rax*4]   ; Prefetch y[i+PD]
    prefetcht0 [rsi + rax*4]   ; Prefetch x[i+PD]
    
    ; Process 8 floats at once with AVX
    vmovups ymm2, [rsi + rcx*4]    ; x[i..i+7]
    vmovups ymm3, [rdi + rcx*4]    ; y[i..i+7]
    vfmadd213ps ymm2, ymm1, ymm3   ; ymm2 = a*x + y
    vmovups [rdi + rcx*4], ymm2    ; y[i..i+7] = result
    
    add rcx, 8          ; Process 8 elements per iteration
    
    lea rax, rcx
    cmp rax, rdx
    jl .loop
    
    vzeroupper
    ret
```

### 11.5 HW Prefetcher Behavior

Hardware prefetcher จะ detect patterns โดยอัตโนมัติ:

```
HW Prefetcher Types:
1. Stream Prefetcher: ตรวจจับ sequential access, prefetch ล่วงหน้า
2. Stride Prefetcher: ตรวจจับ constant stride access
3. Indirect Stream Prefetcher: รองรับ pointer chasing

Tips:
- HW prefetcher ทำงานได้ดีกับ sequential หรือ regular stride access
- SW prefetch จำเป็นสำหรับ irregular patterns
- ไม่ควร SW prefetch ถ้า HW prefetcher จัดการได้ (overhead)
- HW prefetcher มักใช้เวลาไม่กี่ accesses ในการ detect pattern
```

---

## 12. CLFLUSH และ CLFLUSHOPT

### 12.1 CLFLUSH - Cache Line Flush

```nasm
; clflush_usage.asm
; การใช้ CLFLUSH สำหรับ cache control

section .text

; CLFLUSH: Flush cache line to memory (invalidate from all caches)
; Usage: clflush [memory_address]

global flush_range

; void flush_range(void* start, size_t bytes)
; rdi = start, rsi = bytes
flush_range:
    push rbx
    
    mov rbx, rdi        ; current = start
    add rsi, rdi        ; end = start + bytes
    
.loop:
    cmp rbx, rsi
    jge .done
    
    clflush [rbx]       ; Flush cache line containing [rbx]
    add rbx, 64         ; Next cache line (64 bytes)
    jmp .loop
    
.done:
    ; mfence: ensure all flushes complete
    mfence
    
    pop rbx
    ret

; CLFLUSHOPT: Optimized flush (weaker ordering, allows pipelining)
global flush_range_opt

; void flush_range_opt(void* start, size_t bytes)
flush_range_opt:
    push rbx
    
    mov rbx, rdi
    add rsi, rdi        ; end
    
.loop:
    cmp rbx, rsi
    jge .done
    
    clflushopt [rbx]    ; Optimized flush - can be reordered!
    add rbx, 64
    jmp .loop
    
.done:
    sfence              ; sfence (not mfence) is sufficient after clflushopt
    
    pop rbx
    ret
```

### 12.2 Use Cases ของ CLFLUSH

```nasm
; clflush_usecases.asm

; Use case 1: Flush sensitive data from cache (security)
global secure_clear

; void secure_clear(void* secret, size_t len)
; Clear sensitive data and ensure it's not in cache
secure_clear:
    push rbx
    push r12
    
    mov rbx, rdi        ; ptr
    mov r12, rsi        ; len
    
    ; Zero out the memory
    xor eax, eax
.zero_loop:
    test r12, r12
    jz .flush_loop
    mov [rbx], al
    inc rbx
    dec r12
    jmp .zero_loop
    
    ; Reset ptr
    mov rbx, rdi
    mov r12, rsi
    
    ; Flush from cache
.flush_loop:
    cmp r12, 0
    jle .done
    clflush [rbx]
    add rbx, 64
    sub r12, 64
    jmp .flush_loop
    
.done:
    mfence
    pop r12
    pop rbx
    ret

; Use case 2: Measure memory access time (timing attack research)
global measure_access_time

; uint64_t measure_access_time(void* addr)
; Returns cycles to access addr
measure_access_time:
    mfence
    lfence
    rdtsc                       ; Read TSC: edx:eax
    mov r8d, eax
    mov r9d, edx
    lfence
    
    mov rax, [rdi]              ; Access the address
    
    lfence
    rdtsc
    lfence
    
    ; Calculate delta
    shl rdx, 32
    or rax, rdx                 ; rax = current TSC
    
    shl r9, 32
    or r8, r9                   ; r8 = start TSC  
    
    sub rax, r8                 ; delta
    ret
```

---

## 13. Performance Measurement

### 13.1 perf stat - Cache Statistics

```bash
# วัด cache performance พื้นฐาน
perf stat -e cache-misses,cache-references ./program

# ผลลัพธ์ตัวอย่าง:
# Performance counter stats for './program':
#
#     1,234,567    cache-misses              #    5.67 % of all cache refs
#    21,789,012    cache-references
#
#       0.123456789 seconds time elapsed

# L1 cache miss rate
perf stat -e L1-dcache-loads,L1-dcache-load-misses ./program

# ดูรายละเอียดทุก cache level
perf stat -e \
    L1-dcache-loads,L1-dcache-load-misses,\
    L2_RQSTS.ALL_DEMAND_REFERENCES,L2_RQSTS.DEMAND_DATA_RD_MISS,\
    LLC-loads,LLC-load-misses \
    ./program
```

### 13.2 perf stat Events ที่สำคัญ

```bash
# Intel-specific cache events
perf stat -e \
    mem_load_retired.l1_hit,\
    mem_load_retired.l1_miss,\
    mem_load_retired.l2_hit,\
    mem_load_retired.l2_miss,\
    mem_load_retired.l3_hit,\
    mem_load_retired.l3_miss \
    ./program

# False sharing detection
perf stat -e \
    machine_clears.memory_ordering,\
    mem_inst_retired.stlb_miss_loads,\
    mem_inst_retired.stlb_miss_stores \
    ./program

# Cache bandwidth
perf stat -e \
    offcore_requests.demand_data_rd,\
    offcore_requests.demand_rfo \
    ./program
```

### 13.3 perf record + perf report (Hotspot Analysis)

```bash
# Record cache miss hotspots
perf record -e cache-misses:pp ./program
perf report

# Record with call graph
perf record -g -e cache-misses ./program
perf report --stdio

# Sample at function level
perf record -e LLC-load-misses ./program
perf annotate main  # ดู cache misses per instruction
```

### 13.4 perf c2c (Cache-to-Cache)

```bash
# Detect false sharing (requires root or perf_event_paranoid <= 0)
perf c2c record -g -- ./multithreaded_program
perf c2c report --stdio

# Output แสดง:
# Shared Data Cache Line Table
# ...
# False sharing locations
```

---

## 14. Cachegrind Analysis

### 14.1 Valgrind Cachegrind

Cachegrind เป็น cache simulator ที่ให้ข้อมูล cache miss ระดับ instruction

```bash
# รัน Cachegrind
valgrind --tool=cachegrind ./program

# ดูผลลัพธ์
cg_annotate cachegrind.out.<pid>

# ผลลัพธ์ตัวอย่าง:
# I refs:      1,234,567
# I1 misses:       1,234  (0.10%)
# LLi misses:         12  (0.00%)
#
# D refs:        567,890  (234,567 rd + 333,323 wr)
# D1 misses:      12,345  (2.18%)
# LLd misses:      1,234  (0.22%)
```

### 14.2 Cachegrind Options

```bash
# ปรับ cache size ให้ตรงกับ hardware จริง
valgrind --tool=cachegrind \
    --I1=32768,8,64 \   # I1: 32KB, 8-way, 64B lines
    --D1=32768,8,64 \   # D1: 32KB, 8-way, 64B lines
    --LL=8388608,16,64 \ # L3: 8MB, 16-way, 64B lines
    ./program

# Annotate ทุก source line
cg_annotate --auto=yes cachegrind.out.*
```

### 14.3 ตีความผลลัพธ์ Cachegrind

```
Format ของ cg_annotate output:
Ir      I1mr    ILmr    Dr      D1mr    DLmr    Dw      D1mw    DLmw
123456  0       0       56789   1234    45      34567   234     12

Ir   = Total instruction reads
I1mr = I1 cache instruction read misses
ILmr = Last-level instruction cache read misses
Dr   = Total data reads
D1mr = D1 cache data read misses
DLmr = LL data read misses
Dw   = Total data writes
D1mw = D1 cache data write misses
DLmw = LL data write misses
```

### 14.4 KCachegrind - GUI Visualization

```bash
# ติดตั้ง KCachegrind
apt-get install kcachegrind  # Ubuntu/Debian
yum install kcachegrind       # RHEL/CentOS

# แปลงไฟล์และเปิด
callgrind_annotate cachegrind.out.*
kcachegrind cachegrind.out.*
```

---

## 15. Advanced Optimization Techniques

### 15.1 Array of Structures vs Structure of Arrays

```nasm
; aos_vs_soa.asm
; Array of Structures (AoS) vs Structure of Arrays (SoA)

; AoS (Array of Structures) - ปกติ
; struct Point { float x, y, z; }
; Point points[N];
; Memory: x0 y0 z0 x1 y1 z1 x2 y2 z2 ...

; SoA (Structure of Arrays) - cache-friendly สำหรับ SIMD
; float xs[N], ys[N], zs[N];
; Memory: x0 x1 x2 ... xN-1 | y0 y1 y2 ... yN-1 | ...

section .data
    N equ 1024

    ; AoS layout
    align 64
    points_aos:
    %rep N
        dd 0.0  ; x
        dd 0.0  ; y
        dd 0.0  ; z
        dd 0.0  ; pad (to align to 16 bytes)
    %endrep

    ; SoA layout
    align 64
    xs: times N dd 0.0   ; All x values
    align 64
    ys: times N dd 0.0   ; All y values
    align 64
    zs: times N dd 0.0   ; All z values

section .text
global sum_x_aos

; sum_x_aos: sum all x values from AoS layout
; xmm0 = return value (sum)
sum_x_aos:
    vxorps ymm0, ymm0, ymm0     ; sum = 0
    xor rcx, rcx
    
.loop:
    cmp rcx, N
    jge .done
    
    ; Load x from AoS: stride = 16 bytes
    vmovss xmm1, [points_aos + rcx*16]   ; points[i].x
    vaddss xmm0, xmm0, xmm1
    
    inc rcx
    jmp .loop
    
.done:
    ret   ; Bad: stride access, poor SIMD utilization

global sum_x_soa

; sum_x_soa: sum all x values from SoA layout
sum_x_soa:
    vxorps ymm0, ymm0, ymm0     ; sum vector = 0
    xor rcx, rcx
    
.loop:
    cmp rcx, N
    jge .done
    
    ; Load 8 x values at once! Sequential access
    vmovups ymm1, [xs + rcx*4]   ; xs[i..i+7] - 8 floats
    vaddps ymm0, ymm0, ymm1       ; Vectorized add!
    
    add rcx, 8
    jmp .loop
    
.done:
    ; Horizontal sum of ymm0
    vextractf128 xmm1, ymm0, 1
    vaddps xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0
    
    vzeroupper
    ret   ; Good: sequential, SIMD-friendly
```

### 15.2 Data Alignment for Cache Performance

```nasm
; alignment_demo.asm
; ตัวอย่าง data alignment และผลต่อ performance

section .data

; Misaligned data (starts at odd offset)
misaligned_data:
    db 0x01                 ; 1 byte to cause misalignment
    misaligned_array: times 1024 dd 0   ; starts at odd offset

; Aligned data
align 64                    ; Align to cache line boundary
aligned_array: times 1024 dd 0

section .text
global sum_misaligned

; sum_misaligned: sum array that may cross cache line boundaries
sum_misaligned:
    xor eax, eax
    xor rcx, rcx
    
.loop:
    cmp rcx, 1024
    jge .done
    add eax, [misaligned_array + rcx*4]   ; May split across cache lines
    inc rcx
    jmp .loop
    
.done:
    ret

global sum_aligned

; sum_aligned: sum cache-line-aligned array
sum_aligned:
    vxorps ymm0, ymm0, ymm0   ; sum = 0 (8 floats)
    xor rcx, rcx
    
.loop:
    cmp rcx, 1024
    jge .done
    vmovaps ymm1, [aligned_array + rcx*4]  ; Aligned load (vmovaps not vmovups)
    vaddps ymm0, ymm0, ymm1
    add rcx, 8
    jmp .loop
    
.done:
    ; Horizontal reduction
    vextractf128 xmm1, ymm0, 1
    vaddps xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0
    
    vzeroupper
    ret
```

### 15.3 Non-Temporal Stores (Streaming Writes)

```nasm
; nontemporal_stores.asm
; Non-temporal stores สำหรับ write-only large data

; ปกติ: write ไปยัง memory → cache ต้อง read-modify-write → wastes cache
; NT stores: bypass cache hierarchy → ไม่ pollute cache

section .text
global memset_nontemporal

; void memset_nontemporal(void* dst, int val, size_t n)
; rdi = dst, esi = val (as int, will broadcast to all bytes)
; rdx = n (in bytes, must be multiple of 32)
memset_nontemporal:
    ; Broadcast val to all bytes of ymm0
    movd xmm0, esi
    vpbroadcastd ymm0, xmm0    ; val in all 8 dwords
    
    xor rcx, rcx               ; offset = 0
    
.loop:
    cmp rcx, rdx
    jge .done
    
    ; Non-temporal store: bypass cache
    vmovntdq [rdi + rcx], ymm0  ; Store 32 bytes without caching
    add rcx, 32
    jmp .loop
    
.done:
    sfence          ; Ensure NT stores complete before returning
    vzeroupper
    ret

; Normal memset for comparison (pollutes cache)
global memset_normal
memset_normal:
    movd xmm0, esi
    vpbroadcastd ymm0, xmm0
    
    xor rcx, rcx
    
.loop:
    cmp rcx, rdx
    jge .done
    
    vmovdqu [rdi + rcx], ymm0   ; Normal store (goes through cache)
    add rcx, 32
    jmp .loop
    
.done:
    vzeroupper
    ret

; memcpy_nontemporal: streaming copy
global memcpy_nontemporal

; void memcpy_nontemporal(void* dst, void* src, size_t n)
memcpy_nontemporal:
    xor rcx, rcx
    
.loop:
    cmp rcx, rdx
    jge .done
    
    vmovdqa ymm0, [rsi + rcx]    ; Load (this DOES go through cache)
    vmovntdq [rdi + rcx], ymm0   ; Store bypasses cache
    add rcx, 32
    jmp .loop
    
.done:
    sfence
    vzeroupper
    ret
```

---

## 16. Cache Performance Patterns

### 16.1 Linked List vs Array Traversal

```nasm
; linked_vs_array.asm
; สาธิตความต่างระหว่าง linked list และ array

; Array traversal - sequential, cache-friendly
global sum_array

; int sum_array(int* arr, int n)
sum_array:
    xor eax, eax
    xor rcx, rcx
.loop:
    cmp rcx, rsi
    jge .done
    add eax, [rdi + rcx*4]   ; Sequential - predictable, HW prefetch works
    inc rcx
    jmp .loop
.done:
    ret

; Linked list traversal - random pointer chasing, cache-unfriendly
; struct Node { int value; Node* next; }
; node layout: [int value (4 bytes)] [pad 4 bytes] [Node* next (8 bytes)]
%define NODE_VALUE  0
%define NODE_NEXT   8
%define NODE_SIZE   16

global sum_linked_list

; int sum_linked_list(Node* head)
; rdi = head
sum_linked_list:
    xor eax, eax
    mov rcx, rdi            ; current = head
.loop:
    test rcx, rcx           ; current == NULL?
    jz .done
    add eax, [rcx + NODE_VALUE]   ; value - random memory address!
    mov rcx, [rcx + NODE_NEXT]    ; Follow pointer - cache miss likely!
    jmp .loop
.done:
    ret
```

### 16.2 Binary Search vs Linear Search

```nasm
; search_cache.asm
; Binary search vs linear search สำหรับ cache behavior

; Linear search - sequential, cache-friendly
; O(n) แต่ cache behavior ดีมาก
global linear_search

; int linear_search(int* arr, int n, int target)
; rdi = arr, rsi = n, edx = target
linear_search:
    xor rcx, rcx
.loop:
    cmp rcx, rsi
    jge .not_found
    cmp [rdi + rcx*4], edx
    je .found
    inc rcx
    jmp .loop
.found:
    mov eax, ecx
    ret
.not_found:
    mov eax, -1
    ret

; Binary search - O(log n) แต่ random access, cache misses บ่อยขึ้น
global binary_search

; int binary_search(int* arr, int n, int target)
binary_search:
    xor eax, eax            ; left = 0
    mov rcx, rsi            ; right = n
    dec rcx
.loop:
    cmp rax, rcx
    jg .not_found
    
    lea r8, [rax + rcx]
    shr r8, 1               ; mid = (left+right)/2
    
    cmp [rdi + r8*4], edx
    je .found
    jl .go_right
    
    ; target < arr[mid], go left
    lea rcx, [r8 - 1]
    jmp .loop
    
.go_right:
    lea rax, [r8 + 1]
    jmp .loop
    
.found:
    mov eax, r8d
    ret
.not_found:
    mov eax, -1
    ret
```

---

## 17. Complete Optimization Example: Convolution

```nasm
; convolution_optimized.asm
; 1D convolution ที่ optimize แล้วสำหรับ cache

; void convolve_naive(float* out, float* in, float* kernel, int n, int k)
; rdi=out, rsi=in, rdx=kernel, rcx=n, r8=k
section .text
global convolve_naive

convolve_naive:
    push rbx
    push r12
    push r13
    
    xor rbx, rbx        ; i = 0
.loop_i:
    cmp rbx, rcx        ; i < n?
    jge .done
    
    vxorps xmm0, xmm0, xmm0    ; sum = 0
    
    xor r12, r12        ; j = 0
.loop_j:
    cmp r12, r8         ; j < k?
    jge .store
    
    ; out[i] += in[i+j] * kernel[j]
    lea r13, [rbx + r12]
    vmovss xmm1, [rsi + r13*4]   ; in[i+j]
    vmovss xmm2, [rdx + r12*4]   ; kernel[j]
    vfmadd231ss xmm0, xmm1, xmm2
    
    inc r12
    jmp .loop_j
    
.store:
    vmovss [rdi + rbx*4], xmm0   ; out[i] = sum
    
    inc rbx
    jmp .loop_i
    
.done:
    pop r13
    pop r12
    pop rbx
    ret

; Cache-optimized version using tiling and prefetch
global convolve_optimized

%define TILE 64
%define PD 8

convolve_optimized:
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 8
    
    mov r9, rdi         ; out
    mov r10, rsi        ; in
    mov r11, rdx        ; kernel
    ; n in rcx, k in r8
    
    ; Outer tile loop
    xor r12, r12        ; ii = 0
    
.loop_ii:
    cmp r12, rcx
    jge .done
    
    ; Inner loop with prefetch
    mov rbx, r12        ; i = ii
    
.loop_i:
    ; min(ii + TILE, n)
    mov rax, r12
    add rax, TILE
    cmp rax, rcx
    cmovg rax, rcx
    cmp rbx, rax
    jge .next_ii
    
    ; Prefetch output location
    lea rax, [rbx + PD]
    prefetcht0 [r9 + rax*4]
    
    vxorps xmm0, xmm0, xmm0
    xor r13, r13        ; j = 0
    
.loop_j:
    cmp r13, r8
    jge .store_tile
    
    ; Prefetch input
    lea rax, [rbx + r13 + PD]
    prefetcht0 [r10 + rax*4]
    
    lea rax, [rbx + r13]
    vmovss xmm1, [r10 + rax*4]
    vmovss xmm2, [r11 + r13*4]
    vfmadd231ss xmm0, xmm1, xmm2
    
    inc r13
    jmp .loop_j
    
.store_tile:
    vmovss [r9 + rbx*4], xmm0
    
    inc rbx
    jmp .loop_i
    
.next_ii:
    add r12, TILE
    jmp .loop_ii
    
.done:
    add rsp, 8
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 18. GAS Syntax Examples

### 18.1 Matrix Multiply in GAS Syntax

```asm
# matmul_gas.s
# Matrix multiplication cache-friendly in GAS syntax
# AT&T syntax: src, dst (opposite of Intel/NASM)

    .section .text
    .global matmul_row_major
    .type matmul_row_major, @function

# void matmul_row_major(double* A, double* B, double* C, int N)
# %rdi = A, %rsi = B, %rdx = C, %ecx = N
matmul_row_major:
    pushq   %rbp
    pushq   %rbx
    pushq   %r12
    pushq   %r13
    pushq   %r14
    pushq   %r15
    
    movq    %rdi, %r8       # A
    movq    %rsi, %r9       # B
    movq    %rdx, %r10      # C
    movslq  %ecx, %r11      # N
    
    xorq    %r12, %r12      # i = 0
    
.Lloop_i:
    cmpq    %r11, %r12
    jge     .Ldone
    
    xorq    %r14, %r14      # k = 0
    
.Lloop_k:
    cmpq    %r11, %r14
    jge     .Lnext_i
    
    # a_ik = A[i*N + k]
    movq    %r12, %rax
    imulq   %r11, %rax
    addq    %r14, %rax
    vmovsd  (%r8,%rax,8), %xmm0
    
    xorq    %r15, %r15      # j = 0
    
.Lloop_j:
    cmpq    %r11, %r15
    jge     .Lnext_k
    
    # B[k*N + j]
    movq    %r14, %rbx
    imulq   %r11, %rbx
    addq    %r15, %rbx
    vmovsd  (%r9,%rbx,8), %xmm1
    
    vmulsd  %xmm0, %xmm1, %xmm1   # a_ik * B[k][j]
    
    # C[i*N + j]
    movq    %r12, %rax
    imulq   %r11, %rax
    addq    %r15, %rax
    vmovsd  (%r10,%rax,8), %xmm2
    vaddsd  %xmm1, %xmm2, %xmm2
    vmovsd  %xmm2, (%r10,%rax,8)
    
    incq    %r15
    jmp     .Lloop_j
    
.Lnext_k:
    incq    %r14
    jmp     .Lloop_k
    
.Lnext_i:
    incq    %r12
    jmp     .Lloop_i
    
.Ldone:
    popq    %r15
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %rbx
    popq    %rbp
    ret

    .size matmul_row_major, .-matmul_row_major
```

### 18.2 Prefetch in GAS Syntax

```asm
# prefetch_gas.s
# Software prefetch ใน GAS syntax

    .section .text
    .global sum_with_prefetch_gas
    .type sum_with_prefetch_gas, @function

# double sum_with_prefetch_gas(double* arr, int n)
# %rdi = arr, %esi = n
sum_with_prefetch_gas:
    vxorpd  %xmm0, %xmm0, %xmm0    # sum = 0.0
    xorl    %ecx, %ecx               # i = 0
    movslq  %esi, %rsi
    
.Lloop:
    cmpq    %rsi, %rcx
    jge     .Ldone
    
    # Prefetch 16 iterations ahead
    leaq    128(%rdi,%rcx,8), %rax   # &arr[i+16]
    prefetcht0  (%rax)
    
    # Process current element
    vaddsd  (%rdi,%rcx,8), %xmm0, %xmm0
    
    incq    %rcx
    jmp     .Lloop
    
.Ldone:
    ret                              # Return in xmm0

    .size sum_with_prefetch_gas, .-sum_with_prefetch_gas
```

---

## 19. Benchmarking Framework

### 19.1 Minimal Benchmarking ด้วย RDTSC

```nasm
; bench_framework.asm
; Framework สำหรับวัด cache performance

section .text
global rdtsc_start

; uint64_t rdtsc_start()
; Returns TSC value for timing start (serializing)
rdtsc_start:
    mfence
    lfence
    rdtsc
    shl rdx, 32
    or rax, rdx
    lfence
    ret

global rdtsc_end

; uint64_t rdtsc_end()  
; Returns TSC value for timing end
rdtsc_end:
    rdtsc
    lfence
    shl rdx, 32
    or rax, rdx
    ret

; Warmup cache before measuring
global warmup_cache

; void warmup_cache(void* ptr, size_t bytes)
warmup_cache:
    xor rcx, rcx
.loop:
    cmp rcx, rsi
    jge .done
    mov rax, [rdi + rcx]   ; Touch each cache line
    add rcx, 64            ; Next cache line
    jmp .loop
.done:
    ret
```

### 19.2 Complete Benchmark Program

```nasm
; cache_benchmark.asm
; Complete benchmark สำหรับ cache analysis

%define ARRAY_SIZE  (1 << 24)   ; 16M elements = 64MB
%define REPEAT      10

section .bss
    array: resd ARRAY_SIZE

section .data
    fmt_result: db "Access pattern: %s, Time: %lu cycles", 10, 0

section .text
extern printf
global main

main:
    push rbp
    mov rbp, rsp
    sub rsp, 32
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    ; Initialize array
    mov rcx, ARRAY_SIZE
    xor rbx, rbx
.init:
    dec rcx
    mov [array + rcx*4], ecx
    test rcx, rcx
    jnz .init
    
    ; Test 1: Sequential access
    mfence
    lfence
    rdtsc
    mov r12d, eax
    mov r13d, edx
    lfence
    
    ; Sequential sum
    xor eax, eax
    xor rcx, rcx
.seq_loop:
    cmp rcx, ARRAY_SIZE
    jge .seq_done
    add eax, [array + rcx*4]
    inc rcx
    jmp .seq_loop
.seq_done:
    mov [rbp-4], eax    ; Prevent optimization
    
    rdtsc
    lfence
    
    ; Calculate time
    sub eax, r12d
    sbb edx, r13d
    
    ; eax = elapsed cycles (lower 32 bits)
    ; (simplified - full 64-bit in real code)
    
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    mov rsp, rbp
    pop rbp
    ret
```

---

## 20. สรุปและ Best Practices

### 20.1 Cache Optimization Checklist

```
1. Data Layout:
   □ ใช้ SoA แทน AoS เมื่อ process เฉพาะบาง fields
   □ Align hot data ให้ตรงกับ cache line boundary
   □ Pad struct เพื่อหลีกเลี่ยง false sharing ใน multi-threaded code
   □ Group frequently-accessed data ไว้ด้วยกัน

2. Access Patterns:
   □ Access array แบบ sequential เมื่อทำได้
   □ สำหรับ 2D array ใช้ row-major order (C default)
   □ ใช้ loop tiling สำหรับ matrix operations ขนาดใหญ่
   □ เลือก ijk order ที่เหมาะสมสำหรับ matrix multiply (ikj ดีที่สุด)

3. Prefetching:
   □ ใช้ PREFETCHT0 สำหรับ data ที่จะใช้เร็วๆ นี้
   □ คำนวณ prefetch distance จาก memory latency / loop time
   □ ระวัง: SW prefetch มี overhead อย่า prefetch ถ้า HW ทำได้
   □ ใช้ PREFETCHNTA สำหรับ streaming data ที่ใช้ครั้งเดียว

4. Reducing Cache Pollution:
   □ ใช้ Non-temporal stores (MOVNT*) สำหรับ write-only large buffers
   □ เรียก CLFLUSH หลังประมวลผลเสร็จถ้าไม่ต้องการ data ใน cache อีก
   □ แยก hot data และ cold data ออกจากกัน

5. Multi-threading:
   □ Pad per-thread data ให้อยู่คนละ cache line (64 bytes)
   □ ใช้ __attribute__((aligned(64))) หรือ alignas(64) ใน C++
   □ Minimize shared mutable state ระหว่าง cores
```

### 20.2 Quick Reference - Cache Sizes และ Latencies

```
Intel Core (Skylake/Tiger Lake):
┌──────────┬──────────┬─────────────┬──────────┬─────────────┐
│ Cache    │ Size     │ Associativity│ Line Size│ Latency     │
├──────────┼──────────┼─────────────┼──────────┼─────────────┤
│ L1D      │ 32 KB    │ 8-way       │ 64 B     │ 4 cycles    │
│ L1I      │ 32 KB    │ 8-way       │ 64 B     │ 4 cycles    │
│ L2       │ 256 KB   │ 4-way       │ 64 B     │ 12 cycles   │
│ L3 (LLC) │ 8-32 MB  │ 16-way      │ 64 B     │ 40+ cycles  │
│ DRAM     │ GBs      │ n/a         │ n/a      │ 200+ cycles │
└──────────┴──────────┴─────────────┴──────────┴─────────────┘
```

### 20.3 Performance Rule of Thumb

```
ถ้า working set:
- ≤ L1 (32KB):   ใช้ได้เต็มที่, latency ~4 cycles
- ≤ L2 (256KB):  ดีมาก, latency ~12 cycles
- ≤ L3 (8MB+):   ดีพอใช้, latency ~40 cycles
- > L3:          Performance-limited by memory bandwidth

สำหรับ matrix NxN:
- N ≤ 64:   fit in L1
- N ≤ 256:  fit in L2 (roughly)
- N ≤ 1024: fit in L3
- N > 1024: need tiling
```

### 20.4 Tools Summary

```bash
# Quick performance analysis
perf stat -e cache-misses,cache-references ./program

# Detailed cache profiling
valgrind --tool=cachegrind ./program && cg_annotate cachegrind.out.*

# False sharing detection
perf c2c record -- ./program && perf c2c report

# Memory access patterns
perf mem record -- ./program && perf mem report

# Cache miss hotspots
perf record -e LLC-load-misses -g ./program
perf report --stdio
```

---

## 21. ตัวอย่างจริง: Optimizing a Real Function

### 21.1 ก่อน Optimize: Dot Product

```nasm
; dot_product_naive.asm
; Naive dot product - ไม่ได้ optimize

; float dot_product_naive(float* a, float* b, int n)
section .text
global dot_product_naive

dot_product_naive:
    vxorps xmm0, xmm0, xmm0    ; result = 0
    xor rcx, rcx
.loop:
    cmp rcx, rdx
    jge .done
    vmovss xmm1, [rdi + rcx*4]
    vmovss xmm2, [rsi + rcx*4]
    vfmadd231ss xmm0, xmm1, xmm2
    inc rcx
    jmp .loop
.done:
    ret
```

### 21.2 หลัง Optimize: Dot Product

```nasm
; dot_product_optimized.asm
; Optimized dot product with AVX, unrolling, prefetch

; float dot_product_opt(float* a, float* b, int n)
; rdi = a (aligned to 32 bytes)
; rsi = b (aligned to 32 bytes)
; edx = n

section .text
global dot_product_opt

%define UNROLL  4       ; Unroll 4x
%define PD      64      ; Prefetch distance in bytes

dot_product_opt:
    ; Clear accumulators (4 independent accumulators for ILP)
    vxorps ymm0, ymm0, ymm0
    vxorps ymm1, ymm1, ymm1
    vxorps ymm2, ymm2, ymm2
    vxorps ymm3, ymm3, ymm3
    
    movslq edx, edx
    xor rcx, rcx
    
    ; Main loop: process 32 floats per iteration (4 YMM * 8 floats)
    lea rax, [rcx + 8*UNROLL]
    
.main_loop:
    lea rax, [rcx + 8*UNROLL]
    cmp rax, rdx
    jg .cleanup
    
    ; Prefetch ahead
    prefetcht0 [rdi + rcx*4 + PD*4]
    prefetcht0 [rsi + rcx*4 + PD*4]
    
    ; Load and multiply 32 floats
    vmovaps ymm4, [rdi + rcx*4 + 0]     ; a[i..i+7]
    vmovaps ymm5, [rdi + rcx*4 + 32]    ; a[i+8..i+15]
    vmovaps ymm6, [rdi + rcx*4 + 64]    ; a[i+16..i+23]
    vmovaps ymm7, [rdi + rcx*4 + 96]    ; a[i+24..i+31]
    
    vfmadd231ps ymm0, ymm4, [rsi + rcx*4 + 0]
    vfmadd231ps ymm1, ymm5, [rsi + rcx*4 + 32]
    vfmadd231ps ymm2, ymm6, [rsi + rcx*4 + 64]
    vfmadd231ps ymm3, ymm7, [rsi + rcx*4 + 96]
    
    add rcx, 8*UNROLL   ; 32 floats processed
    jmp .main_loop
    
.cleanup:
    ; Handle remaining elements
.tail:
    cmp rcx, rdx
    jge .reduce
    
    vmovss xmm4, [rdi + rcx*4]
    vfmadd231ss xmm0, xmm4, [rsi + rcx*4]
    inc rcx
    jmp .tail
    
.reduce:
    ; Combine 4 accumulators
    vaddps ymm0, ymm0, ymm1
    vaddps ymm2, ymm2, ymm3
    vaddps ymm0, ymm0, ymm2
    
    ; Horizontal sum of ymm0
    vextractf128 xmm1, ymm0, 1
    vaddps xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0
    
    vzeroupper
    ret
```

---

## 22. การ Build และ Test

### 22.1 Makefile

```makefile
# Makefile สำหรับ cache optimization examples

CC = gcc
NASM = nasm
AS = as

CFLAGS = -O2 -march=native -g
NASMFLAGS = -f elf64 -g -dwarf
ASFLAGS = --64

all: cache_demo matrix_bench dot_product_bench

# NASM assembly objects
%.o: %.asm
	$(NASM) $(NASMFLAGS) -o $@ $<

# GAS assembly objects
%.o: %.s
	$(AS) $(ASFLAGS) -o $@ $<

# Cache demo
cache_demo: cache_demo_main.c naive_transpose.o tiled_transpose.o
	$(CC) $(CFLAGS) -o $@ $^

# Matrix benchmark
matrix_bench: matrix_bench.c matmul_ikj.o tiled_matmul.o
	$(CC) $(CFLAGS) -o $@ $^

# Dot product benchmark
dot_product_bench: dot_bench.c dot_product_naive.o dot_product_optimized.o
	$(CC) $(CFLAGS) -o $@ $^

# Run with perf
perf_test: matrix_bench
	perf stat -e cache-misses,cache-references,\
	    L1-dcache-load-misses,LLC-load-misses \
	    ./matrix_bench

clean:
	rm -f *.o cache_demo matrix_bench dot_product_bench

.PHONY: all clean perf_test
```

### 22.2 C Harness สำหรับ Test

```c
// cache_test_main.c
// C main function สำหรับ test assembly functions

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <stdint.h>

#define N 1024
#define ALIGN 64

// Declare external assembly functions
extern void naive_transpose(double* src, double* dst, int n);
extern void tiled_transpose(double* src, double* dst, int n);
extern void matmul_ikj(double* A, double* B, double* C, int M, int K, int n);
extern void tiled_matmul(double* A, double* B, double* C, int n);
extern double dot_product_naive(float* a, float* b, int n);
extern float dot_product_opt(float* a, float* b, int n);

// Read TSC for timing
static inline uint64_t rdtsc() {
    uint32_t hi, lo;
    __asm__ volatile("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

// Allocate aligned memory
static void* alloc_aligned(size_t bytes) {
    void* ptr;
    if (posix_memalign(&ptr, ALIGN, bytes) != 0) {
        perror("posix_memalign");
        exit(1);
    }
    return ptr;
}

void test_transpose() {
    printf("=== Matrix Transpose Test ===\n");
    
    double* src = alloc_aligned(N * N * sizeof(double));
    double* dst_naive = alloc_aligned(N * N * sizeof(double));
    double* dst_tiled = alloc_aligned(N * N * sizeof(double));
    
    // Initialize
    for (int i = 0; i < N*N; i++) src[i] = (double)i;
    memset(dst_naive, 0, N*N*sizeof(double));
    memset(dst_tiled, 0, N*N*sizeof(double));
    
    // Warmup
    naive_transpose(src, dst_naive, N);
    tiled_transpose(src, dst_tiled, N);
    
    // Benchmark
    uint64_t t0, t1;
    
    t0 = rdtsc();
    for (int r = 0; r < 10; r++) naive_transpose(src, dst_naive, N);
    t1 = rdtsc();
    printf("Naive transpose: %lu cycles (avg %lu per call)\n",
           t1-t0, (t1-t0)/10);
    
    t0 = rdtsc();
    for (int r = 0; r < 10; r++) tiled_transpose(src, dst_tiled, N);
    t1 = rdtsc();
    printf("Tiled transpose: %lu cycles (avg %lu per call)\n",
           t1-t0, (t1-t0)/10);
    
    // Verify correctness
    int ok = 1;
    for (int i = 0; i < N && ok; i++)
        for (int j = 0; j < N && ok; j++)
            if (dst_naive[j*N+i] != dst_tiled[j*N+i]) {
                printf("MISMATCH at [%d][%d]\n", i, j);
                ok = 0;
            }
    printf("Correctness: %s\n\n", ok ? "PASS" : "FAIL");
    
    free(src); free(dst_naive); free(dst_tiled);
}

int main() {
    printf("Cache Optimization Benchmark\n");
    printf("Matrix size: %d x %d\n\n", N, N);
    
    test_transpose();
    
    return 0;
}
```

---

## 23. Hardware Performance Counters Reference

### 23.1 Intel Performance Events

```
PMU Events สำหรับ Cache Analysis:

L1D Cache:
  mem_load_retired.l1_hit         - L1D data cache hits
  mem_load_retired.l1_miss        - L1D data cache misses
  L1-dcache-loads                 - Total L1D loads (alias)
  L1-dcache-load-misses           - L1D load misses (alias)

L2 Cache:
  mem_load_retired.l2_hit         - L2 hits
  mem_load_retired.l2_miss        - L2 misses
  l2_rqsts.all_demand_data_rd     - All L2 demand data reads
  l2_rqsts.demand_data_rd_miss    - L2 demand read misses

LLC (L3):
  mem_load_retired.l3_hit         - L3 hits
  mem_load_retired.l3_miss        - L3 misses
  LLC-loads                       - LLC load references
  LLC-load-misses                 - LLC load misses

DRAM:
  offcore_requests.demand_data_rd - DRAM data reads
  offcore_response.demand_data_rd.l3_miss - Actual DRAM accesses

False Sharing:
  machine_clears.memory_ordering  - Memory ordering machine clears
  mem_inst_retired.lock_loads     - Locked loads
  
TLB:
  dtlb_load_misses.miss_causes_a_walk   - DTLB misses (load)
  dtlb_store_misses.miss_causes_a_walk  - DTLB misses (store)
```

### 23.2 AMD Performance Events

```
AMD Zen Architecture Cache Events:
  bp_l1_btb_correct           - L1 BTB correct predicts
  l2_cache_req_stat.ic_fill_hit_s    - IC fill hit
  l2_cache_req_stat.ic_fill_miss     - IC fill miss
  l2_cache_req_stat.dc_hit_in_s      - DC hit
  l2_cache_req_stat.dc_miss          - DC miss
  
  l3_comb_clstr_state.request_miss   - L3 miss
  l3_comb_clstr_state.request_hit    - L3 hit
```

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เกี่ยวกับ cache architecture และวิธีเพิ่มประสิทธิภาพโค้ดให้ทำงานได้ดีกับ cache:

**Key Takeaways:**

1. **Cache Hierarchy**: L1D/L1I (32KB) → L2 (256KB) → L3 (8MB+) → DRAM แต่ละระดับมี latency ต่างกันมาก

2. **Cache Line = 64 bytes**: ทุก memory access โหลด 64 bytes เข้ามา ใช้ประโยชน์จากนี้ด้วยการ access แบบ sequential

3. **Miss Types**: Cold miss (ครั้งแรก), Capacity miss (data ใหญ่กว่า cache), Conflict miss (หลาย addresses ชนกัน)

4. **False Sharing**: หลาย threads เขียน data คนละตัวแต่อยู่ใน cache line เดียวกัน แก้ด้วย padding ให้ครบ 64 bytes

5. **Loop Order**: ikj order ดีกว่า ijk สำหรับ matrix multiply เพราะ sequential access

6. **Loop Tiling**: แบ่ง computation เป็น blocks ที่พอดีกับ cache เพิ่ม temporal locality

7. **Prefetch**: ใช้ PREFETCHT0/T1/T2/NTA เพื่อ hide memory latency

8. **Tools**: `perf stat`, `perf c2c`, `cachegrind`, `kcachegrind`

การเขียนโค้ดที่ cache-friendly อาจทำให้โปรแกรมทำงานเร็วขึ้น 10-100x โดยไม่ต้องเปลี่ยน algorithm!

---

*Part 062 จบแล้ว - ต่อไป Part 063: SIMD Advanced Techniques*
