# Part 065: Loop Optimization Techniques

## บทนำ (Introduction)

Loop optimization เป็นหนึ่งในเทคนิคที่สำคัญที่สุดในการเขียน assembly ที่มีประสิทธิภาพ
เนื่องจาก loop มักใช้เวลาในการทำงานส่วนใหญ่ของโปรแกรม การปรับแต่ง loop อย่างถูกต้อง
จึงส่งผลกระทบต่อประสิทธิภาพโดยรวมอย่างมาก

### ทำไม Loop Optimization จึงสำคัญ?

```
Performance Impact:
- โปรแกรมทั่วไปใช้เวลา 80-90% ใน 10-20% ของโค้ด
- ส่วนใหญ่ของ hotspot คือ loop
- การปรับ loop 1 จุด อาจเพิ่มความเร็ว 2-10x
- Modern CPU: out-of-order, branch prediction, caches
- SIMD units ทำงานได้ 4-16x parallel
```

### เครื่องมือที่ใช้ในบทนี้

```
Architecture: x86-64 (64-bit)
Assembler: NASM (Intel syntax) และ GAS (AT&T syntax)
OS: Linux
Tools: perf, valgrind/cachegrind, objdump
Compiler: GCC (สำหรับเปรียบเทียบ compiler output)
```

---

## 1. Loop Unrolling (การคลาย Loop)

### แนวคิด Loop Unrolling

Loop unrolling คือการขยาย body ของ loop ออกหลายครั้ง เพื่อลด overhead ของ
branch prediction, loop counter update, และเพิ่มโอกาสให้ CPU ทำงานแบบ pipeline

**ประโยชน์:**
- ลด loop overhead (decrement counter, compare, branch)
- เพิ่ม instruction-level parallelism (ILP)
- เพิ่มโอกาสให้ compiler/CPU reorder instructions
- ลด branch misprediction

**ข้อเสีย:**
- เพิ่มขนาดของ code (code bloat)
- อาจลด instruction cache efficiency
- ต้องจัดการ remainder (epilogue)

---

### 1.1 Original Loop (ก่อน Unroll)

**ตัวอย่าง: Sum Array (scalar)**

```nasm
; NASM syntax - sum array ธรรมดา
; rdi = pointer to array, rsi = count, return value in rax

section .text
global sum_array_basic

sum_array_basic:
    xor     eax, eax            ; sum = 0
    test    rsi, rsi            ; if count == 0, return
    jz      .done
    
.loop:
    add     eax, [rdi]          ; sum += array[i]
    add     rdi, 4              ; i++ (move pointer)
    dec     rsi                 ; count--
    jnz     .loop               ; if count != 0, loop
    
.done:
    ret
```

**GAS syntax เทียบเท่า:**

```gas
# GAS syntax - sum array ธรรมดา
# %rdi = pointer, %rsi = count, return in %rax

    .section .text
    .global sum_array_basic_gas

sum_array_basic_gas:
    xorl    %eax, %eax          # sum = 0
    testq   %rsi, %rsi          # check count
    jz      .done_basic
    
.loop_basic:
    addl    (%rdi), %eax        # sum += *ptr
    addq    $4, %rdi            # ptr++
    decq    %rsi                # count--
    jnz     .loop_basic         # loop
    
.done_basic:
    ret
```

**วิเคราะห์ overhead:**
```
Loop body: 1 instruction (add)
Loop overhead: 3 instructions (add rdi, dec rsi, jnz)
Overhead ratio: 75% !!
```

---

### 1.2 2x Loop Unrolling

```nasm
; NASM syntax - 2x unrolled sum
; ทำ 2 elements ต่อ iteration

sum_array_2x:
    xor     eax, eax            ; sum = 0
    mov     rcx, rsi            ; rcx = count
    shr     rcx, 1              ; rcx = count / 2 (main iterations)
    jz      .remainder_2x       ; if count < 2, skip main loop
    
.loop_2x:
    add     eax, [rdi]          ; sum += array[i]
    add     eax, [rdi + 4]      ; sum += array[i+1]
    add     rdi, 8              ; advance 2 elements
    dec     rcx                 ; count--
    jnz     .loop_2x
    
.remainder_2x:
    ; handle odd element
    test    rsi, 1              ; if count is odd
    jz      .done_2x
    add     eax, [rdi]          ; sum += last element
    
.done_2x:
    ret
```

**วิเคราะห์:**
```
Loop body: 2 instructions (2 adds)
Loop overhead: 3 instructions (add rdi, dec, jnz)
Overhead ratio: 60% (ลดลงจาก 75%)
ILP opportunity: CPU สามารถ parallel add ทั้ง 2 ได้
```

---

### 1.3 4x Loop Unrolling

```nasm
; NASM syntax - 4x unrolled sum

sum_array_4x:
    xor     eax, eax            ; sum = 0
    mov     rcx, rsi
    shr     rcx, 2              ; rcx = count / 4
    jz      .handle_remainder_4x
    
.loop_4x:
    add     eax, [rdi]
    add     eax, [rdi + 4]
    add     eax, [rdi + 8]
    add     eax, [rdi + 12]
    add     rdi, 16             ; advance 4 elements (4 * 4 bytes)
    dec     rcx
    jnz     .loop_4x
    
.handle_remainder_4x:
    ; remainder = count & 3 (count % 4)
    and     rsi, 3
    jz      .done_4x
    
.rem_loop_4x:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .rem_loop_4x
    
.done_4x:
    ret
```

**ปรับปรุงด้วย multiple accumulators:**

```nasm
; 4x unrolled with 4 accumulators - ลด dependency chain

sum_array_4x_multi_acc:
    xor     eax, eax            ; acc0
    xor     edx, edx            ; acc1
    xor     ecx, ecx            ; acc2
    xor     r8d, r8d            ; acc3
    
    mov     r9, rsi
    shr     r9, 2               ; main iterations
    jz      .rem_multi
    
.loop_multi:
    add     eax, [rdi]          ; acc0 += array[i]
    add     edx, [rdi + 4]      ; acc1 += array[i+1]
    add     ecx, [rdi + 8]      ; acc2 += array[i+2]
    add     r8d, [rdi + 12]     ; acc3 += array[i+3]
    add     rdi, 16
    dec     r9
    jnz     .loop_multi
    
    ; combine accumulators
    add     eax, edx
    add     ecx, r8d
    add     eax, ecx
    
.rem_multi:
    and     rsi, 3
    jz      .done_multi
    
.rem_loop_multi:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .rem_loop_multi
    
.done_multi:
    ret
```

**ทำไมต้องใช้หลาย accumulators?**
```
ถ้าใช้ accumulator เดียว:
    add eax, [rdi]      ; cycle 1: load + add
    add eax, [rdi+4]    ; cycle 2: ต้องรอ eax จาก cycle 1 (dependency!)
    
ถ้าใช้หลาย accumulators:
    add eax, [rdi]      ; cycle 1
    add edx, [rdi+4]    ; cycle 1 (parallel! ไม่รอ eax)
    add ecx, [rdi+8]    ; cycle 1 (parallel!)
    add r8d, [rdi+12]   ; cycle 1 (parallel!)
```

---

### 1.4 8x Loop Unrolling

```nasm
; 8x unrolled with 8 accumulators - สำหรับ high-throughput sum

sum_array_8x:
    ; ใช้ xmm registers เป็น 8 int32 accumulators
    ; (ตัวอย่างนี้ยังใช้ scalar registers เพื่อความง่าย)
    
    xor     eax, eax
    xor     edx, edx
    xor     ecx, ecx
    xor     r8d, r8d
    xor     r9d, r9d
    xor     r10d, r10d
    xor     r11d, r11d
    xor     ebx, ebx    ; สังเกต: ต้อง save rbx (callee-saved)
    
    push    rbx         ; save rbx
    
    mov     rax, rsi    ; ใช้ rax เป็น temp counter ก่อน
    shr     rax, 3      ; count / 8
    ; ... (complex code)
    
    pop     rbx
    ret

; เวอร์ชันง่ายกว่า - 8x unroll แบบ pointer arithmetic

sum_array_8x_simple:
    xor     eax, eax
    xor     edx, edx
    
    mov     rcx, rsi
    shr     rcx, 3      ; count / 8
    jz      .rem8
    
.loop8:
    add     eax, [rdi + 0]
    add     edx, [rdi + 4]
    add     eax, [rdi + 8]
    add     edx, [rdi + 12]
    add     eax, [rdi + 16]
    add     edx, [rdi + 20]
    add     eax, [rdi + 24]
    add     edx, [rdi + 28]
    add     rdi, 32     ; advance 8 elements
    dec     rcx
    jnz     .loop8
    
    add     eax, edx    ; combine
    
.rem8:
    and     rsi, 7
    jz      .done8
    
.rem8_loop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .rem8_loop
    
.done8:
    ret
```

---

## 2. Remainder Handling (Epilogue)

การจัดการ element ที่เหลือหลังจาก unrolled loop เป็นสิ่งสำคัญ

### 2.1 วิธีการต่างๆ ของ Epilogue

**วิธี 1: Simple loop สำหรับ remainder**

```nasm
; remainder loop ธรรมดา
.epilogue_simple:
    and     rsi, mask           ; เช่น and rsi, 3 สำหรับ 4x
    jz      .done
.epi_loop:
    ; process 1 element
    add     rdi, element_size
    dec     rsi
    jnz     .epi_loop
.done:
    ret
```

**วิธี 2: Computed jump table (unrolled epilogue)**

```nasm
; epilogue jump table - ไม่มี branch ใน epilogue

sum_epilogue_jumptable:
    xor     eax, eax
    
    mov     rcx, rsi
    and     rcx, 3              ; remainder = count & 3
    
    ; jump to correct entry point
    lea     r8, [rel .epi_table]
    jmp     [r8 + rcx*8]        ; jump to table entry
    
.epi_table:
    dq  .epi_0
    dq  .epi_1
    dq  .epi_2
    dq  .epi_3
    
.epi_3:
    add     eax, [rdi + 8]      ; fall-through!
.epi_2:
    add     eax, [rdi + 4]
.epi_1:
    add     eax, [rdi]
.epi_0:
    ; done with epilogue, continue with main loop setup
    
    shr     rsi, 2              ; count / 4
    ; ... main loop ...
    ret
```

**วิธี 3: Predicated execution (cmov)**

```nasm
; ใช้ conditional move แทน branch

; สำหรับ remainder = 0 หรือ 1:
    xor     edx, edx
    test    rsi, 1
    cmovnz  edx, [rdi]          ; edx = (count & 1) ? array[i] : 0
    add     eax, edx
```

---

## 3. Duff's Device

Duff's Device เป็น technique ที่รวม loop กับ switch-case เพื่อทำ unrolled epilogue
ถูกคิดค้นโดย Tom Duff ในปี 1983

### 3.1 Duff's Device ใน C (สำหรับเข้าใจแนวคิด)

```c
/* Duff's Device ใน C */
void send(short *to, const short *from, int count) {
    int n = (count + 7) / 8;
    switch (count % 8) {
    case 0: do { *to++ = *from++;  // fall through
    case 7:      *to++ = *from++;
    case 6:      *to++ = *from++;
    case 5:      *to++ = *from++;
    case 4:      *to++ = *from++;
    case 3:      *to++ = *from++;
    case 2:      *to++ = *from++;
    case 1:      *to++ = *from++;
               } while (--n > 0);
    }
}
```

### 3.2 Duff's Device ใน Assembly

```nasm
; Duff's Device ใน NASM
; copy array of shorts: rdi = dst, rsi = src, rdx = count

duffs_device_copy:
    test    rdx, rdx
    jz      .done
    
    ; compute n = (count + 7) / 8
    lea     rcx, [rdx + 7]
    shr     rcx, 3              ; n = ceil(count / 8)
    
    ; compute entry point: count % 8 * jump_offset
    mov     rax, rdx
    and     rax, 7              ; rax = count % 8
    jz      .entry_0            ; special case: count divisible by 8
    
    ; jump table for entry points
    lea     r8, [rel .jump_table]
    jmp     [r8 + rax*8]
    
.jump_table:
    dq  .entry_0    ; 0 mod 8
    dq  .entry_1    ; 1 mod 8
    dq  .entry_2    ; 2 mod 8
    dq  .entry_3    ; 3 mod 8
    dq  .entry_4    ; 4 mod 8
    dq  .entry_5    ; 5 mod 8
    dq  .entry_6    ; 6 mod 8
    dq  .entry_7    ; 7 mod 8

.entry_7:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    ; FALL THROUGH
    
.entry_6:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.entry_5:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.entry_4:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.entry_3:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.entry_2:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.entry_1:
    mov     ax, [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.entry_0:
    dec     rcx
    jz      .done
    
    ; main 8-element loop
    mov     ax, [rsi]
    mov     [rdi], ax
    mov     ax, [rsi + 2]
    mov     [rdi + 2], ax
    mov     ax, [rsi + 4]
    mov     [rdi + 4], ax
    mov     ax, [rsi + 6]
    mov     [rdi + 6], ax
    mov     ax, [rsi + 8]
    mov     [rdi + 8], ax
    mov     ax, [rsi + 10]
    mov     [rdi + 10], ax
    mov     ax, [rsi + 12]
    mov     [rdi + 12], ax
    mov     ax, [rsi + 14]
    mov     [rdi + 14], ax
    add     rsi, 16
    add     rdi, 16
    
    dec     rcx
    jnz     .entry_0
    
.done:
    ret
```

---

## 4. Loop Fusion (การรวม Loop)

Loop fusion คือการรวมสอง loop ที่วนซ้ำบน data เดียวกัน เพื่อลด loop overhead
และปรับปรุง cache locality

### 4.1 Before Fusion: สอง Loop แยกกัน

```nasm
; ก่อน fusion: คำนวณ sum และ max แยกกัน
; rdi = array, rsi = count

; Loop 1: หา sum
sum_loop_before:
    xor     eax, eax        ; sum = 0
    mov     rcx, rsi
    test    rcx, rcx
    jz      .done_sum
.sum_loop:
    add     eax, [rdi + rcx*4 - 4]
    dec     rcx
    jnz     .sum_loop
.done_sum:
    ; array must be iterated again for max!

; Loop 2: หา max (array iterated 2nd time - bad cache usage!)
max_loop_before:
    mov     edx, 0x80000000 ; INT_MIN
    mov     rcx, rsi
    test    rcx, rcx
    jz      .done_max
.max_loop:
    mov     r8d, [rdi + rcx*4 - 4]
    cmp     r8d, edx
    cmovg   edx, r8d        ; if arr[i] > max, update max
    dec     rcx
    jnz     .max_loop
.done_max:
    ret
```

### 4.2 After Fusion: Loop เดียว

```nasm
; หลัง fusion: คำนวณ sum และ max ใน loop เดียว
; rdi = array, rsi = count
; returns: eax = sum, edx = max

sum_max_fused:
    xor     eax, eax            ; sum = 0
    mov     edx, 0x80000000     ; max = INT_MIN
    
    test    rsi, rsi
    jz      .done_fused
    
    xor     rcx, rcx            ; i = 0
    
.fused_loop:
    mov     r8d, [rdi + rcx*4]  ; r8d = array[i]
    add     eax, r8d            ; sum += array[i]
    cmp     r8d, edx
    cmovg   edx, r8d            ; max = max(max, array[i])
    inc     rcx
    cmp     rcx, rsi
    jl      .fused_loop
    
.done_fused:
    ret
```

**ข้อดีของ Loop Fusion:**
```
ก่อน fusion:
  - Array traversed 2 ครั้ง
  - 2N cache misses
  - 2 loop overhead sets

หลัง fusion:
  - Array traversed 1 ครั้ง
  - N cache misses (ลดลงครึ่ง!)
  - 1 loop overhead set
  - สำหรับ large arrays: massive speedup จาก cache
```

---

## 5. Loop Fission / Distribution (การแยก Loop)

Loop fission เป็นตรงกันข้ามกับ fusion - แยก loop ออกเป็นหลาย loop
เพื่อปรับปรุง data locality สำหรับแต่ละ operation

### 5.1 เมื่อไหร่ควรใช้ Loop Fission?

```
ใช้ fission เมื่อ:
1. Loop body ใหญ่เกินไป - register pressure สูง
2. แต่ละ part ใช้ data structure ต่างกัน
3. ต้องการ vectorize บาง part แต่ไม่ทั้งหมด
4. มี dependency ที่ขัดขวาง vectorization ใน combined loop
```

### 5.2 ตัวอย่าง Loop Fission

```nasm
; ก่อน fission: loop รวม - ยาก vectorize
; เพราะ struct of arrays ใหญ่ เกิน cache line

; struct: { int a[N], b[N], c[N], d[N] }
; loop เดิม: ทำ operation บน a, b, c, d ทั้งหมดใน 1 loop
; ปัญหา: แต่ละ iteration แตะ 4 arrays = thrash cache

; หลัง fission: แยกเป็น 4 loop
; Loop 1: process array a
process_a:
    mov     rcx, rsi            ; count
    test    rcx, rcx
    jz      .done_a
    lea     r8, [rdi]           ; base of array a
.loop_a:
    mov     eax, [r8]
    ; ทำ operation บน a
    lea     r8, [r8 + 4]
    dec     rcx
    jnz     .loop_a
.done_a:

; Loop 2: process array b (b starts at offset N*4)
process_b:
    mov     rcx, rsi
    test    rcx, rcx
    jz      .done_b
    lea     r8, [rdi + rsi*4]   ; base of array b
.loop_b:
    mov     eax, [r8]
    ; ทำ operation บน b
    lea     r8, [r8 + 4]
    dec     rcx
    jnz     .loop_b
.done_b:
    ret
```

---

## 6. Loop Tiling / Blocking (การแบ่ง Loop เป็น Tile)

Loop tiling เป็นเทคนิคสำคัญสำหรับ matrix operations เพื่อปรับปรุง cache usage

### 6.1 แนวคิด Cache Blocking

```
Matrix multiplication ที่ไม่ tile:
  for i in 0..N:
    for j in 0..N:
      for k in 0..N:
        C[i][j] += A[i][k] * B[k][j]
  
  ปัญหา: B[k][j] access pattern เป็น column-major
         สำหรับ N > cache_size/element_size → cache miss ทุก B[k][j]

Tiled version:
  for i0 in 0..N step TILE:
    for j0 in 0..N step TILE:
      for k0 in 0..N step TILE:
        for i in i0..min(i0+TILE, N):
          for j in j0..min(j0+TILE, N):
            for k in k0..min(k0+TILE, N):
              C[i][j] += A[i][k] * B[k][j]
  
  ผล: TILE×TILE block ของ A, B, C fit ใน L1/L2 cache
```

### 6.2 2D Loop Tiling ใน Assembly

```nasm
; 2D Matrix addition with tiling
; C[N][N] = A[N][N] + B[N][N]
; rdi = C, rsi = A, rdx = B, rcx = N (square matrix)

section .data
TILE_SIZE equ 32        ; 32x32 tile = 32*32*4 = 4096 bytes (fits in L1)

section .text
matrix_add_tiled:
    push    rbp
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; rdi = C_ptr (output)
    ; rsi = A_ptr
    ; rdx = B_ptr
    ; rcx = N
    
    mov     r12, rdi            ; C
    mov     r13, rsi            ; A
    mov     r14, rdx            ; B
    mov     r15, rcx            ; N
    
    ; outer loops: tile boundaries
    xor     rbx, rbx            ; i0 = 0
    
.tile_i_loop:
    cmp     rbx, r15
    jge     .tile_done
    
    xor     rbp, rbp            ; j0 = 0
    
.tile_j_loop:
    cmp     rbp, r15
    jge     .next_tile_i
    
    ; inner loops: elements within tile
    mov     r8, rbx             ; i = i0
    
.inner_i:
    ; compute end of i-range: min(i0 + TILE_SIZE, N)
    lea     rax, [rbx + TILE_SIZE]
    cmp     rax, r15
    cmovg   rax, r15            ; rax = min(i0+TILE, N)
    cmp     r8, rax
    jge     .next_tile_j
    
    mov     r9, rbp             ; j = j0
    
.inner_j:
    ; compute end of j-range
    lea     rdx, [rbp + TILE_SIZE]
    cmp     rdx, r15
    cmovg   rdx, r15
    cmp     r9, rdx
    jge     .next_inner_i
    
    ; compute index: [i * N + j]
    mov     rax, r8
    imul    rax, r15            ; rax = i * N
    add     rax, r9             ; rax = i * N + j
    
    ; C[i][j] = A[i][j] + B[i][j]
    mov     r10d, [r13 + rax*4]     ; A[i][j]
    add     r10d, [r14 + rax*4]     ; + B[i][j]
    mov     [r12 + rax*4], r10d     ; C[i][j] =
    
    inc     r9                  ; j++
    jmp     .inner_j
    
.next_inner_i:
    inc     r8                  ; i++
    jmp     .inner_i
    
.next_tile_j:
    add     rbp, TILE_SIZE      ; j0 += TILE_SIZE
    jmp     .tile_j_loop
    
.next_tile_i:
    add     rbx, TILE_SIZE      ; i0 += TILE_SIZE
    jmp     .tile_i_loop
    
.tile_done:
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret
```

### 6.3 3D Loop Tiling (สำหรับ 3D computations)

```nasm
; แนวคิด 3D tiling สำหรับ stencil computation
; ใช้ 3 tile ขนาดเล็กใน i, j, k dimensions

; TILE_I equ 8
; TILE_J equ 8
; TILE_K equ 8

; for k0 in 0..K step TILE_K:
;   for j0 in 0..J step TILE_J:
;     for i0 in 0..I step TILE_I:
;       for k in k0..k0+TILE_K:
;         for j in j0..j0+TILE_J:
;           for i in i0..i0+TILE_I:
;             out[k][j][i] = stencil(in, k, j, i)

; ผลลัพธ์: in[TILE_K+2][TILE_J+2][TILE_I+2] fit ใน cache
```

---

## 7. Loop Interchange (เปลี่ยนลำดับ Nesting)

Loop interchange เป็นการสลับ inner/outer loop เพื่อปรับปรุง data locality

### 7.1 Row-Major vs Column-Major Access

```
C arrays (Row-major storage):
  int A[M][N];
  A[i][j] อยู่ที่ address: base + (i*N + j) * sizeof(int)
  
  Row-major access (ดี - sequential):
    for i:
      for j:          ← j เปลี่ยนก่อน = consecutive memory
        A[i][j]
  
  Column-major access (แย่ - stride N):
    for j:
      for i:          ← i เปลี่ยนก่อน = stride N access
        A[i][j]
```

### 7.2 ตัวอย่าง Loop Interchange

```nasm
; Matrix row sum - version 1: BAD access pattern
; A[N][M], sum ทุก row แล้วเก็บใน row_sums[N]

; ตัวอย่างแบบ BAD (column-major):
; for j in 0..M:        ← outer: column
;   for i in 0..N:      ← inner: row (stride M access to A!)
;     row_sums[i] += A[i][j]

row_sum_bad:
    ; rdi = A, rsi = row_sums, rdx = N, rcx = M
    xor     r8, r8              ; j = 0
.outer_j_bad:
    cmp     r8, rcx
    jge     .done_bad
    xor     r9, r9              ; i = 0
.inner_i_bad:
    cmp     r9, rdx
    jge     .next_j_bad
    ; access A[i][j] = A + (i*M + j) * 4
    mov     rax, r9
    imul    rax, rcx            ; rax = i * M
    add     rax, r8             ; rax = i * M + j
    mov     r10d, [rdi + rax*4]
    ; access row_sums[i] = row_sums + i*4
    add     [rsi + r9*4], r10d  ; row_sums[i] += A[i][j]
    inc     r9
    jmp     .inner_i_bad
.next_j_bad:
    inc     r8
    jmp     .outer_j_bad
.done_bad:
    ret

; ตัวอย่างแบบ GOOD (row-major) - หลัง interchange:
; for i in 0..N:        ← outer: row
;   for j in 0..M:      ← inner: column (sequential access to A!)
;     row_sums[i] += A[i][j]

row_sum_good:
    ; rdi = A, rsi = row_sums, rdx = N, rcx = M
    xor     r8, r8              ; i = 0
.outer_i_good:
    cmp     r8, rdx
    jge     .done_good
    
    xor     r10d, r10d          ; sum = 0
    
    ; A[i][0] starts at A + i*M*4
    mov     rax, r8
    imul    rax, rcx
    lea     r11, [rdi + rax*4]  ; r11 = &A[i][0]
    
    xor     r9, r9              ; j = 0
.inner_j_good:
    cmp     r9, rcx
    jge     .next_i_good
    add     r10d, [r11 + r9*4]  ; sum += A[i][j] - SEQUENTIAL!
    inc     r9
    jmp     .inner_j_good
    
.next_i_good:
    mov     [rsi + r8*4], r10d  ; row_sums[i] = sum
    inc     r8
    jmp     .outer_i_good
.done_good:
    ret
```

---

## 8. Loop Invariant Code Motion (LICM)

LICM คือการย้าย computation ที่ไม่เปลี่ยนแปลงใน loop ออกไปทำก่อน loop

### 8.1 ตัวอย่าง LICM

```nasm
; ก่อน LICM: คำนวณ offset ซ้ำทุก iteration
licm_before:
    ; rdi = base_ptr, rsi = count, rdx = stride, rcx = offset_x
    ; sum = 0
    ; for i in 0..count:
    ;   sum += base_ptr[(i * stride + offset_x)]
    
    xor     eax, eax            ; sum = 0
    xor     r8, r8              ; i = 0
.loop_before:
    cmp     r8, rsi
    jge     .done_before
    
    ; BAD: offset_x ไม่เปลี่ยน แต่คำนวณใหม่ทุก iteration
    mov     r9, r8
    imul    r9, rdx             ; r9 = i * stride
    add     r9, rcx             ; r9 = i * stride + offset_x  ← invariant: + offset_x
    add     eax, [rdi + r9*4]
    
    inc     r8
    jmp     .loop_before
.done_before:
    ret

; หลัง LICM: ย้าย offset_x computation ออกไปก่อน loop
licm_after:
    ; rdi = base_ptr, rsi = count, rdx = stride, rcx = offset_x
    
    ; HOIST: คำนวณ base + offset_x ไว้ก่อน
    lea     rdi, [rdi + rcx*4]  ; rdi = base_ptr + offset_x (bytes: *4)
    ; ตอนนี้ใน loop ไม่ต้องบวก offset_x แล้ว
    
    xor     eax, eax            ; sum = 0
    xor     r8, r8              ; i = 0
.loop_after:
    cmp     r8, rsi
    jge     .done_after
    
    mov     r9, r8
    imul    r9, rdx             ; r9 = i * stride (ไม่มี + offset_x แล้ว)
    add     eax, [rdi + r9*4]   ; ใช้ rdi ที่ปรับแล้ว
    
    inc     r8
    jmp     .loop_after
.done_after:
    ret
```

### 8.2 LICM กับ Memory Loads

```nasm
; ตัวอย่าง: array length ที่ไม่เปลี่ยน

; BAD: load length จาก memory ทุก iteration
loop_reload_bad:
    ; rdi = struct pointer
    ; struct layout: { int *data; int length; }
    xor     eax, eax
    xor     rcx, rcx
.bad_loop:
    cmp     ecx, [rdi + 8]      ; ← load length จาก memory ทุกรอบ!
    jge     .bad_done
    mov     r8, [rdi]           ; data pointer
    add     eax, [r8 + rcx*4]
    inc     ecx
    jmp     .bad_loop
.bad_done:
    ret

; GOOD: load length ครั้งเดียวก่อน loop
loop_reload_good:
    mov     r8, [rdi]           ; data pointer (hoist)
    mov     ecx, [rdi + 8]      ; length (hoist)
    xor     eax, eax
    xor     edx, edx            ; i = 0
.good_loop:
    cmp     edx, ecx            ; compare with register (fast!)
    jge     .good_done
    add     eax, [r8 + rdx*4]
    inc     edx
    jmp     .good_loop
.good_done:
    ret
```

---

## 9. Induction Variable Strength Reduction

การแปลง `array[i * stride]` เป็น pointer arithmetic `*ptr++`

### 9.1 แนวคิด

```
Strength reduction: แทนที่ multiply (แพง) ด้วย add (ถูก)

ก่อน: array[i * stride]  → ต้อง multiply ทุก iteration
หลัง: *(ptr++)            → เพียง add ทุก iteration

เหมาะสำหรับ:
- Linear traversal
- Constant stride access
```

### 9.2 ตัวอย่าง Strength Reduction

```nasm
; ก่อน: index computation ด้วย multiply

sum_multiply:
    ; rdi = array, rsi = count, rdx = stride (elements)
    xor     eax, eax
    xor     rcx, rcx            ; i = 0
.mul_loop:
    cmp     rcx, rsi
    jge     .mul_done
    
    mov     r8, rcx
    imul    r8, rdx             ; r8 = i * stride ← EXPENSIVE!
    add     eax, [rdi + r8*4]
    
    inc     rcx
    jmp     .mul_loop
.mul_done:
    ret

; หลัง: pointer arithmetic (strength reduced)

sum_pointer:
    ; rdi = array, rsi = count, rdx = stride
    xor     eax, eax
    
    ; compute byte stride = stride * 4
    lea     r8, [rdx*4]         ; r8 = stride * 4 bytes
    
    ; rdi already points to first element
.ptr_loop:
    test    rsi, rsi
    jz      .ptr_done
    
    add     eax, [rdi]          ; sum += *ptr (no multiply!)
    add     rdi, r8             ; ptr += stride (add, not multiply!)
    dec     rsi
    jmp     .ptr_loop
.ptr_done:
    ret
```

### 9.3 Multiple Induction Variables

```nasm
; ลดหลาย induction variables พร้อมกัน

loop_multi_iv:
    ; ก่อน:
    ; for i in 0..N:
    ;   a[i] = b[i*2] + c[i*3]   ← 2 multiply per iteration
    
    ; หลัง:
    ; a_ptr = a, b_ptr = b, c_ptr = c
    ; while N--:
    ;   *a_ptr = *b_ptr + *c_ptr
    ;   a_ptr += 1 (stride 1)
    ;   b_ptr += 2 (stride 2)
    ;   c_ptr += 3 (stride 3)
    
    ; rdi = a, rsi = b, rdx = c, rcx = N (all int32 arrays)
    test    rcx, rcx
    jz      .done_miv
.miv_loop:
    mov     eax, [rsi]          ; eax = *b_ptr
    add     eax, [rdx]          ; eax += *c_ptr
    mov     [rdi], eax          ; *a_ptr = eax
    
    add     rdi, 4              ; a_ptr += 1 element
    add     rsi, 8              ; b_ptr += 2 elements  (stride 2)
    add     rdx, 12             ; c_ptr += 3 elements  (stride 3)
    
    dec     rcx
    jnz     .miv_loop
.done_miv:
    ret
```

---

## 10. Loop Peeling (การลอก Iteration)

Loop peeling คือการแยก iteration แรกหรือสุดท้ายออกจาก main loop
เพื่อจัดการกรณีพิเศษโดยไม่ต้องมี branch ใน loop

### 10.1 First Iteration Peeling

```nasm
; ตัวอย่าง: หา maximum โดยต้อง initialize max = array[0]
; ก่อน peeling: ต้องมี special case ใน loop

max_no_peel:
    ; rdi = array, rsi = count
    ; ปัญหา: ต้อง initialize max ก่อน
    
    test    rsi, rsi
    jz      .no_peel_empty
    
    mov     eax, 0x80000000     ; เริ่มต้น INT_MIN
    xor     rcx, rcx
    
.no_peel_loop:
    cmp     rcx, rsi
    jge     .no_peel_done
    mov     edx, [rdi + rcx*4]
    cmp     edx, eax
    cmovg   eax, edx
    inc     rcx
    jmp     .no_peel_loop
    
.no_peel_empty:
    xor     eax, eax
.no_peel_done:
    ret

; หลัง peeling: peel first iteration

max_peeled:
    ; rdi = array, rsi = count
    test    rsi, rsi
    jz      .peeled_empty
    
    ; PEEL: first iteration - initialize max = array[0]
    mov     eax, [rdi]          ; max = array[0]
    mov     rcx, 1              ; start loop at index 1
    
    ; main loop: ไม่ต้องใส่ special case แล้ว
.peeled_loop:
    cmp     rcx, rsi
    jge     .peeled_done
    mov     edx, [rdi + rcx*4]
    cmp     edx, eax
    cmovg   eax, edx
    inc     rcx
    jmp     .peeled_loop
    
.peeled_empty:
    xor     eax, eax
.peeled_done:
    ret
```

### 10.2 Last Iteration Peeling (สำหรับ prologue/epilogue patterns)

```nasm
; Peeling สำหรับ loop ที่ต้องการ "lookahead" (อ่านหน้า 1 element)

; ตัวอย่าง: หา first pair ที่เพิ่มขึ้น (a[i] < a[i+1])
; loop ปกติต้องมี bound check พิเศษสำหรับ element สุดท้าย

find_increase_peeled:
    ; rdi = array, rsi = count
    cmp     rsi, 1
    jle     .no_pair            ; ต้องมีอย่างน้อย 2 elements
    
    ; main loop: ไม่ต้องกังวล out-of-bounds เพราะ peel last
    mov     rcx, rsi
    dec     rcx                 ; ทำ N-1 iterations
    xor     r8, r8              ; i = 0
    
.find_loop:
    cmp     r8, rcx
    jge     .no_pair
    
    mov     eax, [rdi + r8*4]       ; a[i]
    mov     edx, [rdi + r8*4 + 4]   ; a[i+1] - safe because we stop at N-1
    cmp     eax, edx
    jl      .found_pair
    
    inc     r8
    jmp     .find_loop
    
.found_pair:
    mov     rax, r8             ; return index
    ret
    
.no_pair:
    mov     rax, -1             ; not found
    ret
```

---

## 11. Loop Versioning (สร้าง Version แยกสำหรับ Cases ต่างกัน)

Loop versioning คือการสร้าง 2+ versions ของ loop สำหรับ input conditions ต่างกัน

### 11.1 Aligned vs Unaligned Versioning

```nasm
; Loop versioning สำหรับ alignment

sum_versioned:
    ; rdi = array, rsi = count
    
    ; check alignment
    test    rdi, 15             ; check 16-byte alignment
    jnz     .unaligned_path
    
    ; ALIGNED PATH: ใช้ aligned SIMD loads (เร็วกว่า)
    ; (ดู section vectorization)
    jmp     .aligned_sum
    
.unaligned_path:
    ; UNALIGNED PATH: ใช้ scalar หรือ unaligned SIMD
    jmp     .scalar_sum
    
.aligned_sum:
    ; fast path with aligned SSE
    ; ... (เพิ่มโค้ด SIMD ที่นี่)
    ret
    
.scalar_sum:
    ; slow path
    xor     eax, eax
.scalar_loop:
    test    rsi, rsi
    jz      .scalar_done
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jmp     .scalar_loop
.scalar_done:
    ret
```

### 11.2 Alias Check Versioning

```nasm
; Loop versioning based on pointer aliasing
; ถ้า arrays ไม่ overlap: ใช้ version ที่เร็วกว่า

; void add_arrays(int *a, int *b, int *c, int n)
; c[i] = a[i] + b[i]

add_arrays_versioned:
    ; rdi = a, rsi = b, rdx = c, rcx = n
    
    ; check: does a overlap with c?
    ; overlap if |a - c| < n*4
    mov     rax, rdi            ; rax = a
    sub     rax, rdx            ; rax = a - c
    ; take absolute value
    mov     r8, rax
    neg     r8
    cmovs   rax, r8             ; rax = |a - c|
    
    lea     r8, [rcx*4]         ; r8 = n * 4 (bytes)
    cmp     rax, r8
    jl      .has_alias          ; overlap detected
    
    ; check b vs c
    mov     rax, rsi
    sub     rax, rdx
    mov     r8, rax
    neg     r8
    cmovs   rax, r8
    lea     r8, [rcx*4]
    cmp     rax, r8
    jl      .has_alias
    
    ; NO ALIAS: fast version (can vectorize, reorder freely)
    jmp     .fast_add
    
.has_alias:
    ; HAS ALIAS: safe sequential version
.safe_loop:
    test    rcx, rcx
    jz      .add_done
    mov     eax, [rdi]
    add     eax, [rsi]
    mov     [rdx], eax
    add     rdi, 4
    add     rsi, 4
    add     rdx, 4
    dec     rcx
    jmp     .safe_loop
    
.fast_add:
    ; (fast vectorized version here)
.add_done:
    ret
```

---

## 12. Vectorization (SIMD Loop)

### 12.1 แนวคิด SIMD สำหรับ Loop

```
SIMD = Single Instruction, Multiple Data
SSE2: process 4 int32 หรือ 2 int64 พร้อมกัน
AVX2: process 8 int32 พร้อมกัน
AVX-512: process 16 int32 พร้อมกัน

scalar loop N iterations → SIMD loop N/4 iterations (SSE2)
```

### 12.2 Auto-vectorization Hints

```nasm
; เขียน loop ในรูปแบบที่ compiler vectorize ง่าย:
; 1. ไม่มี dependency ระหว่าง iterations
; 2. Sequential memory access
; 3. Fixed loop count (หรือ countable)
; 4. ไม่มี function calls ใน loop body
; 5. ไม่มี pointer aliasing

; GCC hint: __attribute__((optimize("O3")))
; GCC vectorize: -O2 -ftree-vectorize
; NASM: ไม่มี auto-vectorize, ต้อง manual
```

### 12.3 SSE2 Manual SIMD Sum Array

```nasm
; SSE2 sum of int32 array
; 4 elements per iteration using XMM registers

section .data
align 16
zero_vec: times 4 dd 0      ; [0, 0, 0, 0]

section .text
sum_array_sse2:
    ; rdi = array (must be 16-byte aligned for movdqa)
    ; rsi = count (must be multiple of 4 for simplicity)
    
    pxor    xmm0, xmm0          ; xmm0 = [0, 0, 0, 0] accumulator
    
    mov     rcx, rsi
    shr     rcx, 2              ; iterations = count / 4
    
    test    rcx, rcx
    jz      .sse2_done
    
.sse2_loop:
    movdqa  xmm1, [rdi]         ; xmm1 = [a[0], a[1], a[2], a[3]] (aligned load)
    paddd   xmm0, xmm1          ; xmm0 += xmm1 (4 int32 additions parallel!)
    add     rdi, 16             ; advance 4 elements (4 * 4 bytes)
    dec     rcx
    jnz     .sse2_loop
    
    ; horizontal sum: xmm0 = [s0, s1, s2, s3] → s0+s1+s2+s3
    movdqa  xmm1, xmm0
    psrldq  xmm1, 8             ; xmm1 = [s2, s3, 0, 0]
    paddd   xmm0, xmm1          ; xmm0 = [s0+s2, s1+s3, s2, s3]
    
    movdqa  xmm1, xmm0
    psrldq  xmm1, 4             ; xmm1 = [s1+s3, ...]
    paddd   xmm0, xmm1          ; xmm0[0] = s0+s1+s2+s3
    
    movd    eax, xmm0           ; extract low 32 bits = total sum
    
.sse2_done:
    ret
```

### 12.4 Unaligned SIMD with Runtime Alignment Check

```nasm
; ตัวอย่าง: handle both aligned และ unaligned arrays

sum_sse2_any_align:
    ; rdi = array (any alignment), rsi = count
    
    xor     eax, eax            ; scalar accumulator for unaligned prefix
    
    ; 1. Scalar prefix: process until 16-byte aligned
    mov     r8, rdi
    and     r8, 15              ; r8 = offset from 16-byte boundary
    jz      .already_aligned
    
    mov     r9, 16
    sub     r9, r8              ; r9 = bytes until next 16B boundary
    shr     r9, 2               ; r9 = elements until aligned
    cmp     r9, rsi
    cmovg   r9, rsi             ; don't exceed count
    
.prefix_loop:
    test    r9, r9
    jz      .already_aligned
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    dec     r9
    jmp     .prefix_loop
    
.already_aligned:
    ; 2. Main SIMD loop (aligned)
    pxor    xmm0, xmm0
    
    mov     rcx, rsi
    shr     rcx, 2              ; SIMD iterations
    jz      .after_simd
    
.simd_loop:
    movdqa  xmm1, [rdi]
    paddd   xmm0, xmm1
    add     rdi, 16
    dec     rcx
    jnz     .simd_loop
    
    ; horizontal sum
    movdqa  xmm1, xmm0
    psrldq  xmm1, 8
    paddd   xmm0, xmm1
    movdqa  xmm1, xmm0
    psrldq  xmm1, 4
    paddd   xmm0, xmm1
    movd    r10d, xmm0
    add     eax, r10d
    
.after_simd:
    ; 3. Scalar suffix: remaining elements
    and     rsi, 3
.suffix_loop:
    test    rsi, rsi
    jz      .any_done
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jmp     .suffix_loop
    
.any_done:
    ret
```

---

## 13. Software Pipelining

Software pipelining คือการ overlap การทำงานของ iteration ต่างๆ ใน pipeline

### 13.1 แนวคิด Software Pipelining

```
Hardware pipeline: CPU execute หลาย instructions พร้อมกัน
Software pipeline: programmer จัด instructions ด้วยตนเอง

ปัญหาของ loop ปกติ:
  Iteration N:  load → compute → store
  Iteration N+1:         load → compute → store  (รอ)
  
  Load latency: 4-5 cycles
  Compute: 1 cycle
  จะมี stall ระหว่าง load กับ compute
  
Software pipelined:
  Stage 1 (prologue):  load A[0]
  Stage 2:             load A[1], compute A[0], store result[-1]
  Stage 3:             load A[2], compute A[1], store result[0]
  ...
  Stage N+1 (epilogue): compute A[N], store result[N-1]
```

### 13.2 ตัวอย่าง Software Pipelining (2-stage)

```nasm
; Software pipelining: sum array with 2-stage pipeline
; Stage 1: load
; Stage 2: accumulate

sum_sw_pipeline:
    ; rdi = array, rsi = count
    xor     eax, eax
    
    cmp     rsi, 2
    jl      .pipeline_scalar
    
    ; Prologue: start pipeline
    mov     r8d, [rdi]          ; pre-load a[0]
    add     rdi, 4
    dec     rsi
    
    ; Main loop: overlap load[i+1] with add[i]
.pipeline_loop:
    cmp     rsi, 1
    jle     .pipeline_epilogue
    
    mov     r9d, [rdi]          ; load a[i+1]    ← starts early
    add     eax, r8d            ; add a[i]        ← no dependency on r9
    mov     r8d, r9d            ; move for next iteration
    add     rdi, 4
    dec     rsi
    jmp     .pipeline_loop
    
    ; Epilogue: drain pipeline
.pipeline_epilogue:
    add     eax, r8d            ; last element
    ret
    
.pipeline_scalar:
    ; handle count < 2
.ps_loop:
    test    rsi, rsi
    jz      .ps_done
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jmp     .ps_loop
.ps_done:
    ret
```

---

## 14. Modulo Scheduling

Modulo scheduling เป็น advanced form ของ software pipelining สำหรับ loops

### 14.1 แนวคิด Modulo Scheduling

```
Initiation Interval (II): จำนวน cycles ต่อ iteration
MII: minimum initiation interval
  MII = max(RecMII, ResMII)
  
  RecMII: recurrence-constrained MII (ขึ้นกับ dependency loop)
  ResMII: resource-constrained MII (ขึ้นกับ functional units)

ตัวอย่าง:
  Instructions: 4 loads, 4 adds, 4 stores
  Resources: 2 load ports, 2 ALU, 1 store port
  ResMII = max(4/2, 4/2, 4/1) = 4 cycles
  
  Modulo schedule: เริ่ม iteration ใหม่ทุก 4 cycles
  ซ้อน 2-3 iterations พร้อมกัน!
```

### 14.2 Manual Modulo Schedule Example

```nasm
; Loop: a[i] = b[i] + c[i] * scale
; Load latency: 4 cycles
; Multiply latency: 3 cycles
; Add latency: 1 cycle
; Store: 1 cycle

; ไม่มี pipelining (II = 4+3+1+1 = 9 cycles/iteration)
no_pipeline:
    ; stage 1 (cycle 1): load b[i]
    ; stage 2 (cycle 5): load c[i]
    ; stage 3 (cycle 9): mul c[i] * scale
    ; stage 4 (cycle 12): add + store

; Modulo scheduled (II = 4 cycles):
; แต่ละ iteration เริ่มทุก 4 cycles
; มีหลาย iterations ซ้อนอยู่พร้อมกัน

modulo_scheduled:
    ; rdi = a, rsi = b, rdx = c, rcx = n, r8 = scale
    
    ; Prologue: start iterations 0 and 1 without completing
    movdqu  xmm0, [rsi]         ; load b[0..3]
    movdqu  xmm1, [rdx]         ; load c[0..3]
    
    ; Main loop (II = 1 SIMD iteration processing 4 elements)
    add     rsi, 16
    add     rdx, 16
    sub     rcx, 4
    
    movd    xmm7, r8d
    pshufd  xmm7, xmm7, 0       ; broadcast scale to all 4 lanes
    
.modloop:
    ; Current iter: compute from prev-loaded data
    movdqu  xmm2, [rsi]         ; load b[next] ← overlaps with compute
    pmulld  xmm1, xmm7          ; c[cur] * scale
    movdqu  xmm3, [rdx]         ; load c[next] ← overlaps with multiply
    paddd   xmm0, xmm1          ; b[cur] + (c[cur] * scale)
    movdqu  [rdi], xmm0         ; store result
    
    ; set up for next iteration
    movdqa  xmm0, xmm2          ; b[next] becomes b[cur]
    movdqa  xmm1, xmm3          ; c[next] becomes c[cur]
    
    add     rsi, 16
    add     rdx, 16
    add     rdi, 16
    sub     rcx, 4
    jg      .modloop
    
    ; Epilogue: finish last batch
    pmulld  xmm1, xmm7
    paddd   xmm0, xmm1
    movdqu  [rdi], xmm0
    ret
```

---

## 15. Loop Normalization

Loop normalization คือการแปลง loop ให้เริ่มที่ 0 และเพิ่มทีละ 1

### 15.1 ทำไมต้อง Normalize?

```
ประโยชน์ของ normalized loop (start=0, step=1):
1. Compiler/optimizer จดจำ pattern ได้ง่าย
2. Index addressing: array[i] = base + i*sizeof(elem)
3. SIMD vectorization ง่ายกว่า
4. Loop unrolling ง่ายกว่า
5. Counted loop optimization (lea rcx, [n]; loop label)
```

### 15.2 ตัวอย่าง Normalization

```nasm
; Loop ที่ไม่ normalize: start=5, step=3, end=50
; for i = 5 to 50 step 3: process(array[i])

unnormalized:
    mov     ecx, 5              ; i = 5
.loop:
    cmp     ecx, 50
    jge     .done
    
    ; process array[i]
    mov     eax, [rdi + rcx*4]
    ; ...
    
    add     ecx, 3              ; i += 3  ← non-unit step
    jmp     .loop
.done:
    ret

; หลัง normalization: j = (i - 5) / 3, j = 0, 1, 2, ..., 14
; i = j*3 + 5
; for j = 0 to 15: process(array[j*3 + 5])

normalized:
    ; count = (50 - 5 + 3 - 1) / 3 = 15
    mov     ecx, 15             ; j = 0..14 (15 iterations)
    xor     r8d, r8d            ; j = 0
.norm_loop:
    cmp     r8d, ecx
    jge     .norm_done
    
    ; i = j*3 + 5
    lea     eax, [r8d*3 + 5]    ; eax = j*3 + 5
    mov     edx, [rdi + rax*4]  ; array[i]
    ; ...
    
    inc     r8d                 ; j++ (unit step!)
    jmp     .norm_loop
.norm_done:
    ret
```

---

## 16. Counted Loops vs Pointer Comparison

### 16.1 Counted Loop (Counter-based)

```nasm
; Counted loop: ใช้ counter ที่ลดลงสู่ศูนย์

counted_loop:
    ; rdi = array, rsi = count
    xor     eax, eax
    mov     rcx, rsi            ; counter
    test    rcx, rcx
    jz      .counted_done
    
.cloop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rcx                 ; counter-- แล้ว set ZF
    jnz     .cloop              ; jump if not zero (ใช้ ZF โดยตรง)
    
.counted_done:
    ret
```

**ข้อดีของ counted loop:**
- `dec` + `jnz` ทำได้ใน 1 micro-op บาง CPU
- ไม่ต้องมี explicit `cmp`
- CPU branch predictor รองรับ counted loops ดีมาก

### 16.2 Pointer Comparison Loop

```nasm
; Pointer comparison: วนจนถึง end pointer

pointer_loop:
    ; rdi = start, rsi = count
    xor     eax, eax
    lea     r8, [rdi + rsi*4]   ; r8 = end = start + count*4
    
    cmp     rdi, r8
    jae     .ptr_done
    
.ploop:
    add     eax, [rdi]
    add     rdi, 4
    cmp     rdi, r8             ; compare pointer with end
    jb      .ploop              ; jump if below (unsigned less than)
    
.ptr_done:
    ret
```

**ข้อดีของ pointer comparison:**
- ลด memory addressing complexity
- เหมาะกับ pointer arithmetic
- บางครั้ง compiler generate แบบนี้

### 16.3 Benchmark Comparison (ใน assembly)

```nasm
; การเลือกระหว่าง counted vs pointer depends on use case

; Guideline:
; - ถ้ารู้จำนวน iteration ล่วงหน้า → counted loop
; - ถ้า traverse linked structure → pointer loop
; - ถ้า SIMD → pointer loop มักง่ายกว่า
; - ถ้า loop unrolling → counted loop ง่ายกว่า

; Example: SIMD loop with pointer comparison
simd_pointer_loop:
    ; rdi = array, rsi = count (multiple of 4)
    pxor    xmm0, xmm0
    lea     r8, [rdi + rsi*4]   ; end pointer
    
.simd_ploop:
    cmp     rdi, r8
    jae     .simd_pdone
    
    movdqu  xmm1, [rdi]
    paddd   xmm0, xmm1
    add     rdi, 16
    jmp     .simd_ploop
    
.simd_pdone:
    ; horizontal sum...
    ret
```

---

## 17. Compiler Loop Transforms (-O2)

เมื่อใช้ `-O2` GCC จะทำ loop optimizations อัตโนมัติ

### 17.1 การดู Compiler Output

```bash
# ดู assembly ที่ compiler สร้าง
gcc -O2 -S -o output.s input.c

# ดู optimization passes
gcc -O2 -fopt-info-loop-vec input.c

# ดู vectorization info
gcc -O2 -fopt-info-vec input.c
```

### 17.2 C Code กับ Assembly Output

```c
/* C code ธรรมดา */
int sum_c(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s;
}
```

**GCC -O2 output (x86-64):**

```gas
# GCC -O2 output (AT&T syntax)
# Notice: gcc ทำ vectorization อัตโนมัติ!

sum_c:
    xorl    %eax, %eax          # eax = 0
    testl   %esi, %esi          # test n
    jle     .L1                 # if n <= 0, done
    
    # Check alignment and count for SIMD threshold
    cmpl    $8, %esi            # if n < 8, scalar
    jb      .scalar_path
    
    # Vectorized path (gcc -O2 auto-vectorizes!)
    movl    %esi, %edx
    shrl    $2, %edx            # rdx = n/4
    
    pxor    %xmm0, %xmm0        # xmm0 = 0
    xorl    %ecx, %ecx
    
.Lvec_loop:
    movdqu  (%rdi,%rcx,4), %xmm1
    paddd   %xmm1, %xmm0
    addq    $4, %rcx
    cmpl    %ecx, %edx
    ja      .Lvec_loop
    
    # Horizontal sum
    pshufd  $78, %xmm0, %xmm1  # swap high/low 64-bit halves
    paddd   %xmm1, %xmm0
    pshufd  $229, %xmm0, %xmm1 # move element 1 to 0
    paddd   %xmm1, %xmm0
    movd    %xmm0, %eax
    
    # Handle remaining elements (epilogue)
    ...
    
.scalar_path:
    # เขียน scalar loop สำหรับ small n
    ...
    
.L1:
    ret
```

### 17.3 Compiler Optimization Flags

```
-O0: ไม่ optimize เลย (debug)
-O1: basic optimizations
-O2: standard optimizations
  - LICM (Loop Invariant Code Motion)
  - Strength reduction
  - Dead code elimination
  - Basic vectorization

-O3: aggressive optimizations
  - Loop unrolling (-funroll-loops)
  - Vectorization (-ftree-vectorize)
  - Function inlining

-march=native: ใช้ instruction sets ทั้งหมดของ CPU นี้
  - AVX2 สำหรับ Haswell+
  - AVX-512 สำหรับ Skylake-X+

-fprofile-use: profile-guided optimization
  - ใช้ runtime profile เลือก hot/cold paths
```

---

## 18. ตัวอย่างรวม: Sum Array (Scalar → Unrolled → SIMD)

### 18.1 Scalar Version (Baseline)

```nasm
; Baseline: scalar sum, ไม่มี optimization
sum_scalar:
    xor     eax, eax
    test    rsi, rsi
    jz      .s_done
.s_loop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .s_loop
.s_done:
    ret
; Performance: ~1 element/cycle (limited by add latency chain)
```

### 18.2 Unrolled Version (4x with 4 accumulators)

```nasm
; Optimized: 4x unrolled + multiple accumulators
sum_unrolled_4x:
    xor     eax, eax            ; acc0
    xor     edx, edx            ; acc1
    xor     ecx, ecx            ; acc2
    xor     r8d, r8d            ; acc3
    
    mov     r9, rsi
    shr     r9, 2               ; main loop count
    jz      .u4_rem
    
.u4_main:
    add     eax, [rdi + 0]      ; parallel execution possible!
    add     edx, [rdi + 4]
    add     ecx, [rdi + 8]
    add     r8d, [rdi + 12]
    add     rdi, 16
    dec     r9
    jnz     .u4_main
    
    ; combine accumulators (parallel reduction tree)
    add     eax, edx
    add     ecx, r8d
    add     eax, ecx
    
.u4_rem:
    and     rsi, 3
    jz      .u4_done
.u4_rem_loop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .u4_rem_loop
.u4_done:
    ret
; Performance: ~4 elements/cycle (4 independent chains)
```

### 18.3 SIMD Version (SSE4.1)

```nasm
; SIMD: process 4 int32 per instruction
sum_simd_sse41:
    pxor    xmm0, xmm0          ; accumulator [0,0,0,0]
    pxor    xmm1, xmm1          ; accumulator2 [0,0,0,0]
    
    mov     rcx, rsi
    shr     rcx, 3              ; 8 elements per iteration (2 SIMD ops)
    jz      .sse_rem
    
.sse_main:
    movdqu  xmm2, [rdi]         ; load 4 int32
    movdqu  xmm3, [rdi + 16]    ; load 4 more int32
    paddd   xmm0, xmm2          ; accumulate
    paddd   xmm1, xmm3
    add     rdi, 32
    dec     rcx
    jnz     .sse_main
    
    paddd   xmm0, xmm1          ; combine 2 accumulators
    
    ; horizontal sum of xmm0
    phaddd  xmm0, xmm0          ; [a+b, c+d, a+b, c+d] (SSE3)
    phaddd  xmm0, xmm0          ; [a+b+c+d, ...] 
    movd    eax, xmm0
    
.sse_rem:
    and     rsi, 7
    jz      .sse_done
.sse_rem_loop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .sse_rem_loop
.sse_done:
    ret
; Performance: ~8 elements/cycle (2 SIMD ops + 2 accumulators)
```

### 18.4 AVX2 Version (256-bit SIMD)

```nasm
; AVX2: process 8 int32 per instruction
sum_avx2:
    vpxor   ymm0, ymm0, ymm0    ; 256-bit accumulator
    vpxor   ymm1, ymm1, ymm1
    
    mov     rcx, rsi
    shr     rcx, 4              ; 16 elements per iteration
    jz      .avx_rem
    
.avx_main:
    vmovdqu ymm2, [rdi]         ; load 8 int32
    vmovdqu ymm3, [rdi + 32]    ; load 8 more int32
    vpaddd  ymm0, ymm0, ymm2
    vpaddd  ymm1, ymm1, ymm3
    add     rdi, 64
    dec     rcx
    jnz     .avx_main
    
    vpaddd  ymm0, ymm0, ymm1
    
    ; horizontal sum of ymm0 (8 int32 → 1 sum)
    vextracti128 xmm1, ymm0, 1  ; extract high 128 bits
    vpaddd  xmm0, xmm0, xmm1   ; add high + low halves
    vphaddd xmm0, xmm0, xmm0
    vphaddd xmm0, xmm0, xmm0
    vmovd   eax, xmm0
    
    vzeroupper                  ; ป้องกัน AVX-SSE transition penalty!
    
.avx_rem:
    and     rsi, 15
    jz      .avx_done
.avx_rem_loop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rsi
    jnz     .avx_rem_loop
.avx_done:
    ret
; Performance: ~16 elements/cycle (2 AVX2 ops + 2 accumulators)
```

---

## 19. ตัวอย่าง: Matrix Row Sum

```nasm
; Matrix Row Sum: คำนวณ sum ของแต่ละ row ใน matrix
; rdi = matrix (int32, row-major), rsi = rows, rdx = cols
; rcx = output array (int32)

section .text
global matrix_row_sum

matrix_row_sum:
    push    rbp
    push    r12
    push    r13
    push    r14
    push    r15
    
    mov     r12, rdi            ; matrix
    mov     r13, rsi            ; rows
    mov     r14, rdx            ; cols
    mov     r15, rcx            ; output
    
    xor     rbp, rbp            ; row = 0
    
.row_loop:
    cmp     rbp, r13
    jge     .row_done
    
    ; compute start of row: matrix + row * cols * 4
    mov     rax, rbp
    imul    rax, r14
    lea     rdi, [r12 + rax*4]  ; rdi = &matrix[row][0]
    
    ; compute row sum using SSE2
    pxor    xmm0, xmm0          ; row sum accumulator
    
    mov     rcx, r14
    shr     rcx, 2              ; SIMD iterations
    jz      .row_scalar
    
.row_simd:
    movdqu  xmm1, [rdi]
    paddd   xmm0, xmm1
    add     rdi, 16
    dec     rcx
    jnz     .row_simd
    
    ; horizontal sum
    movdqa  xmm1, xmm0
    psrldq  xmm1, 8
    paddd   xmm0, xmm1
    movdqa  xmm1, xmm0
    psrldq  xmm1, 4
    paddd   xmm0, xmm1
    movd    eax, xmm0
    
.row_scalar:
    ; handle remaining (cols % 4)
    mov     rcx, r14
    and     rcx, 3
    jz      .row_save
.rs_loop:
    add     eax, [rdi]
    add     rdi, 4
    dec     rcx
    jnz     .rs_loop
    
.row_save:
    mov     [r15 + rbp*4], eax  ; output[row] = sum
    
    inc     rbp
    jmp     .row_loop
    
.row_done:
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbp
    ret
```

---

## 20. Performance Measurement และ Profiling

### 20.1 ใช้ RDTSC เพื่อ Measure Cycles

```nasm
; ใช้ RDTSC (Read Time Stamp Counter) วัด cycles
; ต้องใช้ RDTSCP หรือ serialize เพื่อความแม่นยำ

measure_loop_cycles:
    ; Save registers
    push    rbx
    push    r12
    push    r13
    
    mov     r12, rdi            ; array
    mov     r13, rsi            ; count
    
    ; Warm up cache
    mov     rdi, r12
    mov     rsi, r13
    call    sum_scalar
    
    ; Measure start
    mfence                      ; memory fence (serialize)
    rdtsc                       ; eax:edx = TSC
    shl     rdx, 32
    or      rax, rdx
    mov     rbx, rax            ; rbx = start_tsc
    
    ; Run the function
    mov     rdi, r12
    mov     rsi, r13
    call    sum_scalar
    
    ; Measure end
    mfence
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    sub     rax, rbx            ; rax = elapsed cycles
    
    pop     r13
    pop     r12
    pop     rbx
    ret
```

### 20.2 Linux perf สำหรับ Analysis

```bash
# วัด L1 cache misses
perf stat -e L1-dcache-misses,L1-dcache-loads ./program

# ดู hotspots
perf record ./program
perf report

# วัด branch mispredictions
perf stat -e branch-misses,branches ./program

# vectorization check
objdump -d program | grep -E "ymm|xmm|paddd|vpaddd"
```

### 20.3 ผล Performance Comparison

```
Environment: Intel Core i7-8700 @ 3.2GHz, L1=32KB, L2=256KB

Array size: 1M int32 (4MB)

Method                  Cycles/element  Throughput
--------------------------------------------------
scalar_basic            4.0             250M elem/s
scalar_4x_unrolled      1.1             900M elem/s
SSE2_4x_parallel        0.28            3.5B elem/s
AVX2_8x_parallel        0.15            6.5B elem/s

Memory bandwidth limited for large arrays!
For array < L1 cache (32KB):
scalar_basic            1.5             2.0B elem/s
SSE2                    0.26            12B elem/s
AVX2                    0.14            22B elem/s
```

---

## 21. Best Practices และ Guidelines

### 21.1 Loop Optimization Checklist

```
1. Identify hotspot loops (profiling ก่อน optimize!)
2. Check memory access patterns (sequential? stride?)
3. Apply LICM: hoist loop-invariant computations
4. Apply strength reduction: replace multiply with add
5. Consider loop fusion for related loops on same data
6. Consider loop fission for loops with high register pressure
7. Apply loop interchange for better cache access
8. Apply loop tiling for nested loops on large data
9. Consider loop unrolling (2x-8x) with multiple accumulators
10. Consider SIMD vectorization for arithmetic-heavy loops
11. Handle alignment: aligned version + unaligned prologue
12. Measure! Don't assume - verify with perf
```

### 21.2 Common Pitfalls

```nasm
; PITFALL 1: Loop unrolling ทำให้ loop ใหญ่เกิน i-cache

; BAD: unroll มากเกินไป
bad_unroll_1024x:
    ; 1024 additions in loop body
    ; ปัญหา: loop body > L1 instruction cache
    ; ผล: i-cache miss ทุกรอบ! ช้ากว่าเดิม
    
; GOOD: unroll 4-16x เท่านั้น
; Rule of thumb: loop body < 64 instructions

; PITFALL 2: ลืม vzeroupper หลัง AVX
bad_avx:
    vmovdqu ymm0, [rdi]
    vpaddd  ymm0, ymm0, ymm1
    ; ... (ลืม vzeroupper)
    ret                         ; ← ปัญหา: SSE code ต่อมาจะช้า!
    
good_avx:
    vmovdqu ymm0, [rdi]
    vpaddd  ymm0, ymm0, ymm1
    vzeroupper                  ; ← ต้องมีก่อน ret หรือก่อนเรียก SSE function
    ret

; PITFALL 3: ไม่ check alignment สำหรับ aligned load
bad_aligned:
    movdqa  xmm0, [rdi]         ; ← crash ถ้า rdi ไม่ align 16!
    
good_aligned:
    test    rdi, 15
    jnz     .use_unaligned
    movdqa  xmm0, [rdi]         ; aligned (fast)
    jmp     .continue
.use_unaligned:
    movdqu  xmm0, [rdi]         ; unaligned (slightly slower)
.continue:
```

### 21.3 Register Allocation Tips

```nasm
; ใน x86-64 มี register 16 ตัว: rax-r15
; Callee-saved: rbx, rbp, r12-r15 (ต้อง push/pop)
; Caller-saved: rax, rcx, rdx, rsi, rdi, r8-r11

; สำหรับ loop ที่ซับซ้อน ใช้ callee-saved สำหรับ loop variables
; เพื่อหลีกเลี่ยงการ spill ลง stack

register_plan_example:
    push    rbx
    push    r12
    push    r13
    push    r14
    
    ; Now we have 4 extra callee-saved registers
    ; rdi, rsi, rdx, rcx, r8, r9, r10, r11 = scratch (8 more)
    ; Total: 12 registers available without stack spilling
    
    ; ... loop code ...
    
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    ret
```

---

## 22. GAS Syntax Examples

### 22.1 Sum Array ใน GAS (AT&T Syntax)

```gas
# GAS syntax equivalents of key loops

    .section .text
    .global sum_array_gas_4x

# 4x unrolled sum in AT&T syntax
# args: %rdi = array, %rsi = count, return %eax

sum_array_gas_4x:
    xorl    %eax, %eax          # acc0 = 0
    xorl    %edx, %edx          # acc1 = 0
    xorl    %ecx, %ecx          # acc2 = 0
    xorl    %r8d, %r8d          # acc3 = 0
    
    movq    %rsi, %r9           # r9 = count
    shrq    $2, %r9             # r9 = count / 4
    jz      .rem_gas4
    
.main_gas4:
    addl    (%rdi), %eax        # acc0 += array[i]
    addl    4(%rdi), %edx       # acc1 += array[i+1]
    addl    8(%rdi), %ecx       # acc2 += array[i+2]
    addl    12(%rdi), %r8d      # acc3 += array[i+3]
    addq    $16, %rdi           # ptr += 4 elements
    decq    %r9
    jnz     .main_gas4
    
    addl    %edx, %eax
    addl    %r8d, %ecx
    addl    %ecx, %eax
    
.rem_gas4:
    andq    $3, %rsi
    jz      .done_gas4
.rem_gas4_loop:
    addl    (%rdi), %eax
    addq    $4, %rdi
    decq    %rsi
    jnz     .rem_gas4_loop
    
.done_gas4:
    ret
```

### 22.2 Matrix Row Sum ใน GAS

```gas
# GAS: matrix row sum with SSE2

    .section .text
    .global matrix_row_sum_gas

# %rdi = matrix, %rsi = rows, %rdx = cols, %rcx = output

matrix_row_sum_gas:
    pushq   %rbp
    pushq   %r12
    pushq   %r13
    pushq   %r14
    pushq   %r15
    
    movq    %rdi, %r12          # matrix
    movq    %rsi, %r13          # rows
    movq    %rdx, %r14          # cols
    movq    %rcx, %r15          # output
    
    xorq    %rbp, %rbp          # row = 0
    
.row_loop_gas:
    cmpq    %r13, %rbp
    jge     .row_done_gas
    
    movq    %rbp, %rax
    imulq   %r14, %rax
    leaq    (%r12,%rax,4), %rdi # &matrix[row][0]
    
    pxor    %xmm0, %xmm0        # row sum
    
    movq    %r14, %rcx
    shrq    $2, %rcx
    jz      .row_scalar_gas
    
.row_simd_gas:
    movdqu  (%rdi), %xmm1
    paddd   %xmm1, %xmm0
    addq    $16, %rdi
    decq    %rcx
    jnz     .row_simd_gas
    
    # horizontal sum
    movdqa  %xmm0, %xmm1
    psrldq  $8, %xmm1
    paddd   %xmm1, %xmm0
    movdqa  %xmm0, %xmm1
    psrldq  $4, %xmm1
    paddd   %xmm1, %xmm0
    movd    %xmm0, %eax
    
.row_scalar_gas:
    movq    %r14, %rcx
    andq    $3, %rcx
    jz      .row_save_gas
.rs_loop_gas:
    addl    (%rdi), %eax
    addq    $4, %rdi
    decq    %rcx
    jnz     .rs_loop_gas
    
.row_save_gas:
    movl    %eax, (%r15,%rbp,4)
    incq    %rbp
    jmp     .row_loop_gas
    
.row_done_gas:
    popq    %r15
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %rbp
    ret
```

---

## 23. Loop Optimization ใน Context ต่างๆ

### 23.1 String Processing Loop

```nasm
; strlen optimization: ใช้ SIMD หา null terminator

strlen_simd:
    ; rdi = string (ต้องไม่ใช่ NULL)
    mov     rax, rdi            ; save start
    
    ; check alignment first
    mov     rcx, rdi
    and     rcx, 15             ; rcx = bytes until 16-byte boundary
    jz      .str_aligned
    
    ; unaligned prefix: scalar scan
.str_prefix:
    cmp     byte [rdi], 0
    jz      .str_done
    inc     rdi
    dec     rcx
    jnz     .str_prefix
    
.str_aligned:
    ; SIMD scan: look for null byte in 16 bytes at a time
    pxor    xmm0, xmm0          ; xmm0 = all zeros
    
.str_simd:
    movdqa  xmm1, [rdi]         ; load 16 bytes (aligned)
    pcmpeqb xmm1, xmm0          ; compare each byte with 0
    pmovmskb ecx, xmm1          ; ecx = bitmask of zero bytes
    test    ecx, ecx
    jnz     .str_found_null
    add     rdi, 16
    jmp     .str_simd
    
.str_found_null:
    bsf     ecx, ecx            ; find first set bit (position of null)
    add     rdi, rcx
    
.str_done:
    sub     rdi, rax            ; length = current - start
    mov     rax, rdi
    ret
```

### 23.2 Memory Copy Loop (memcpy)

```nasm
; Optimized memcpy ด้วย SSE2

fast_memcpy:
    ; rdi = dst, rsi = src, rdx = count (bytes)
    
    cmp     rdx, 64
    jb      .small_copy
    
    ; align dst to 16 bytes
    mov     rcx, rdi
    and     rcx, 15
    jz      .copy_aligned
    
    mov     rcx, 16
    sub     rcx, rcx
    ; ... prefix copy ...
    
.copy_aligned:
    mov     rcx, rdx
    shr     rcx, 6              ; 64-byte iterations (4 XMM loads)
    
.copy_loop:
    movdqu  xmm0, [rsi]
    movdqu  xmm1, [rsi + 16]
    movdqu  xmm2, [rsi + 32]
    movdqu  xmm3, [rsi + 48]
    movdqa  [rdi], xmm0         ; aligned store (fast!)
    movdqa  [rdi + 16], xmm1
    movdqa  [rdi + 32], xmm2
    movdqa  [rdi + 48], xmm3
    add     rsi, 64
    add     rdi, 64
    dec     rcx
    jnz     .copy_loop
    
    ; handle remaining bytes
    and     rdx, 63
    jz      .copy_done
    
.small_copy:
    rep movsb                   ; use rep movsb for small/remainder
    
.copy_done:
    ret
```

---

## 24. สรุปและ Key Takeaways

### 24.1 Performance Impact Summary

```
Optimization Technique    Typical Speedup    Best Case
-----------------------------------------------------
Loop unrolling 4x        1.5-3x             4x
Multiple accumulators    1.5-4x             4x
LICM                     1.1-2x             variable
Strength reduction       1.1-1.5x           2x
Loop fusion              1.2-2x             2x (cache)
Loop interchange         2-10x              10x (cache miss)
Loop tiling              2-20x              ∞ (cache-oblivious)
SSE2 vectorization       2-4x               4x
AVX2 vectorization       4-8x               8x
Software pipelining      1.2-2x             varies
```

### 24.2 Decision Tree สำหรับ Loop Optimization

```
START: มี hotspot loop?
  ├─ YES: วัด bottleneck ด้วย perf
  │   ├─ Memory bound?
  │   │   ├─ Poor locality → Loop interchange / tiling
  │   │   ├─ Too much data → Loop fusion / tiling
  │   │   └─ Non-temporal? → streaming stores
  │   ├─ Compute bound?
  │   │   ├─ Add/mul dominant → Vectorize (SIMD)
  │   │   ├─ Dependency chain → Unroll + multiple accumulators
  │   │   └─ Redundant compute → LICM
  │   ├─ Branch bound?
  │   │   ├─ Loop overhead → Unroll
  │   │   ├─ Conditional in loop → Peel / version
  │   │   └─ Unpredictable → cmov / branchless
  │   └─ Already optimal? → accept
  └─ NO: find hotspot first!
```

### 24.3 Register Convention Reminder

```
x86-64 Linux calling convention:
  Arguments: rdi, rsi, rdx, rcx, r8, r9
  Return: rax (rdx for 2nd return value)
  Caller-saved (volatile): rax, rcx, rdx, rsi, rdi, r8-r11
  Callee-saved (non-volatile): rbx, rbp, r12-r15
  Stack: rsp (must be 16-byte aligned before call)
  
XMM/YMM:
  xmm0-xmm7: volatile (caller must save if needed)
  xmm8-xmm15: non-volatile (callee must save)
  After using YMM: must call vzeroupper before calling
                   SSE functions or returning
```

---

## 25. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Unroll และวัดความเร็ว
```
เขียน sum_array ใน 3 versions:
1. scalar_basic
2. unrolled_4x_single_acc
3. unrolled_4x_multi_acc
วัดเวลาแต่ละ version ด้วย RDTSC
เปรียบเทียบ cycles/element
```

### แบบฝึกหัดที่ 2: LICM
```
หา loop-invariant expressions ใน code ต่อไปนี้:
for i in 0..N:
    result[i] = (a[i] * b[i]) + (offset_x * scale + base)
ย้าย invariant expressions ออกมาก่อน loop
```

### แบบฝึกหัดที่ 3: Loop Fusion
```
รวม 2 loop ต่อไปนี้:
Loop 1: หา sum ของ array
Loop 2: หา sum of squares ของ array
เขียน fused version ที่ทำทั้ง 2 อย่างใน 1 pass
```

### แบบฝึกหัดที่ 4: Matrix Optimization
```
เขียน matrix transpose ด้วย:
1. Naive version (row-major access)
2. Tiled version (TILE_SIZE = 32)
วัด L1 cache miss rate ด้วย perf
```

### แบบฝึกหัดที่ 5: SIMD Sum
```
เขียน sum_array ด้วย SSE2 และ AVX2
- Handle unaligned input
- Handle count ที่ไม่ใช่ multiple of 4/8
- เปรียบเทียบกับ scalar version
```

---

## 26. อ้างอิงเพิ่มเติม

```
Books:
- "Computer Architecture: A Quantitative Approach" - Hennessy & Patterson
- "Optimizing Software in C++" - Agner Fog
- "The Art of Assembly Language" - Randy Hyde
- "Introduction to 64-bit Assembly Programming" - Ray Seyfarth

Online Resources:
- Agner Fog's optimization manuals: agner.org/optimize
- Intel Optimization Reference Manual
- NASM documentation: nasm.us
- GAS documentation: sourceware.org/binutils/docs/as
- Compiler Explorer (Godbolt): godbolt.org
- uops.info: CPU instruction latencies and throughputs

Tools:
- perf (Linux performance counter)
- valgrind --tool=cachegrind
- Intel VTune Amplifier
- AMD μProf
- objdump -d (disassembly)
- readelf
```

---

*จบ Part 065: Loop Optimization Techniques*

*บทถัดไป: Part 066 - SIMD Advanced Techniques (AVX2/AVX-512)*
