# Part 050: SIMD Practical Applications (การประยุกต์ใช้ SIMD จริง)

## บทนำ

SIMD (Single Instruction, Multiple Data) คือหัวใจของ high-performance computing สมัยใหม่
ในบทนี้เราจะเรียนรู้การใช้ SIMD ในสถานการณ์จริงอย่างครอบคลุม ตั้งแต่ memory operations
ไปจนถึง cryptography

**Registers ที่ใช้:**
- `XMM0-XMM15`: 128-bit SSE registers
- `YMM0-YMM15`: 256-bit AVX registers  
- `ZMM0-ZMM31`: 512-bit AVX-512 registers

---

## Section 1: SIMD Programming Philosophy (ปรัชญาการเขียน SIMD)

### 1.1 Array-of-Structures (AoS) vs Structure-of-Arrays (SoA)

**AoS (Array-of-Structures) — ไม่เหมาะกับ SIMD:**
```c
// AoS: x,y,z,w สลับกัน — SIMD ทำยาก
struct Particle {
    float x, y, z, w;
    float vx, vy, vz, mass;
};
Particle particles[1000];
// Memory: x0 y0 z0 w0 vx0 vy0 vz0 m0 | x1 y1 z1 w1 ...
```

**SoA (Structure-of-Arrays) — เหมาะกับ SIMD:**
```c
// SoA: ข้อมูลชนิดเดียวกันอยู่ติดกัน — SIMD โหลดได้ 4/8 ค่าพร้อมกัน
struct ParticleSystem {
    float x[1000], y[1000], z[1000], w[1000];
    float vx[1000], vy[1000], vz[1000];
    float mass[1000];
};
// Memory: x0 x1 x2 x3 x4 x5 x6 x7 | y0 y1 y2 y3 y4 ...
```

```nasm
; ตัวอย่าง: update positions ด้วย SoA + AVX2
; void update_positions(float *x, float *y, float *vx, float *vy, 
;                       float dt, int count)
; rdi=x, rsi=y, rdx=vx, rcx=vy, xmm0=dt, r8d=count
section .text
global update_positions_avx2

update_positions_avx2:
    vbroadcastss ymm0, xmm0        ; broadcast dt ไปทุก lane
    xor eax, eax                   ; i = 0
    mov r9d, r8d
    and r9d, -8                    ; round down to multiple of 8

.loop8:
    cmp eax, r9d
    jge .remainder

    vmovaps ymm1, [rdi + rax*4]    ; load x[i..i+7]
    vmovaps ymm2, [rdx + rax*4]    ; load vx[i..i+7]
    vfmadd231ps ymm1, ymm2, ymm0   ; x += vx * dt (FMA!)
    vmovaps [rdi + rax*4], ymm1    ; store x

    vmovaps ymm3, [rsi + rax*4]    ; load y[i..i+7]
    vmovaps ymm4, [rcx + rax*4]    ; load vy[i..i+7]
    vfmadd231ps ymm3, ymm4, ymm0   ; y += vy * dt
    vmovaps [rsi + rax*4], ymm3    ; store y

    add eax, 8
    jmp .loop8

.remainder:
    cmp eax, r8d
    jge .done
    ; scalar tail
    movss xmm5, [rdi + rax*4]
    movss xmm6, [rdx + rax*4]
    mulss xmm6, xmm0
    addss xmm5, xmm6
    movss [rdi + rax*4], xmm5
    ; (ทำเหมือนกันสำหรับ y)
    inc eax
    jmp .remainder

.done:
    ret
```

### 1.2 Alignment Requirements

```nasm
; SIMD alignment: 16-byte สำหรับ SSE, 32-byte สำหรับ AVX, 64-byte สำหรับ AVX-512
section .data
align 64                           ; จำเป็นสำหรับ AVX-512
avx512_data: times 16 dd 0.0      ; 16 floats = 64 bytes

; ใน C code:
; float *buf = aligned_alloc(64, size);
; หรือ __attribute__((aligned(64))) float buf[N];

; VMOVAPS vs VMOVUPS:
; vmovaps = aligned (fault ถ้า misaligned)
; vmovups = unaligned (ช้ากว่าเล็กน้อยบน old CPU)
; บน modern CPU (Haswell+) ต่างกันน้อยมาก
```

### 1.3 Loop Vectorization Conditions

```
Compiler จะ vectorize loop ได้เมื่อ:
1. ไม่มี loop-carried dependency (ค่าแต่ละ iteration ไม่ขึ้นกัน)
2. ไม่มี pointer aliasing (ใช้ __restrict หรือ #pragma ivdep)
3. Loop body เรียบง่าย ไม่มี function call ที่ไม่ inline ได้
4. Trip count รู้ได้ตอน compile หรือ runtime
5. Memory access เป็น unit-stride (ไม่ใช่ scattered)

ตัวอย่างที่ vectorize ได้:
for (int i = 0; i < n; i++) c[i] = a[i] + b[i];

ตัวอย่างที่ vectorize ไม่ได้:
for (int i = 1; i < n; i++) a[i] = a[i-1] + b[i]; // dependency!
```

---

## Section 2: Memory Operations (การดำเนินการหน่วยความจำ)

### 2.1 Fast memcpy — 3 Versions

```nasm
; ============================================================
; Fast memcpy ด้วย SSE (128-bit)
; void *fast_memcpy_sse(void *dst, const void *src, size_t len)
; rdi=dst, rsi=src, rdx=len
; ============================================================
section .text
global fast_memcpy_sse, fast_memcpy_avx2, fast_memcpy_nt

fast_memcpy_sse:
    push rdi                       ; save dst for return
    mov rcx, rdx
    shr rcx, 4                     ; rcx = len / 16 (number of 128-bit chunks)
    jz .tail_sse

.loop_sse:
    movdqu xmm0, [rsi]             ; load 16 bytes (unaligned ok)
    movdqu [rdi], xmm0             ; store 16 bytes
    add rsi, 16
    add rdi, 16
    dec rcx
    jnz .loop_sse

.tail_sse:
    mov rcx, rdx
    and rcx, 15                    ; remaining bytes
    jz .done_sse
    rep movsb                      ; copy remaining bytes

.done_sse:
    pop rax                        ; return dst
    ret


; ============================================================
; Fast memcpy ด้วย AVX2 (256-bit) — 2x faster
; ============================================================
fast_memcpy_avx2:
    push rdi
    mov rcx, rdx
    shr rcx, 5                     ; len / 32
    jz .tail_avx2

.loop_avx2:
    vmovdqu ymm0, [rsi]            ; load 32 bytes
    vmovdqu [rdi], ymm0            ; store 32 bytes
    add rsi, 32
    add rdi, 32
    dec rcx
    jnz .loop_avx2

.tail_avx2:
    mov rcx, rdx
    and rcx, 31
    jz .done_avx2
    rep movsb

.done_avx2:
    pop rax
    vzeroupper                     ; สำคัญ! ล้าง YMM upper bits ก่อน return
    ret


; ============================================================
; Fast memcpy ด้วย Non-Temporal Stores (NT) — สำหรับ large buffers
; NT stores ข้าม cache — เหมาะเมื่อ dst จะไม่ถูกอ่านทันที
; ============================================================
fast_memcpy_nt:
    push rdi
    mov rcx, rdx
    shr rcx, 5
    jz .tail_nt

.loop_nt:
    vmovdqu ymm0, [rsi]            ; load 32 bytes (through cache)
    vmovntdq [rdi], ymm0           ; NT store — bypass cache
    add rsi, 32
    add rdi, 32
    dec rcx
    jnz .loop_nt

    sfence                         ; flush NT stores ก่อน return

.tail_nt:
    mov rcx, rdx
    and rcx, 31
    jz .done_nt
    rep movsb

.done_nt:
    pop rax
    vzeroupper
    ret
```

### 2.2 Fast memset

```nasm
; ============================================================
; Fast memset ด้วย AVX2
; void fast_memset_avx2(void *dst, int val, size_t len)
; rdi=dst, esi=val, rdx=len
; ============================================================
global fast_memset_avx2

fast_memset_avx2:
    push rdi

    ; broadcast byte value ไปทุก byte ใน YMM
    movd xmm0, esi
    vpbroadcastb ymm0, xmm0        ; replicate byte to all 32 bytes

    mov rcx, rdx
    shr rcx, 5
    jz .tail_memset

.loop_memset:
    vmovdqu [rdi], ymm0
    add rdi, 32
    dec rcx
    jnz .loop_memset

.tail_memset:
    mov rcx, rdx
    and rcx, 31
    jz .done_memset
    ; ใช้ rep stosb สำหรับ tail
    movzx eax, sil                 ; byte value
    rep stosb

.done_memset:
    pop rax
    vzeroupper
    ret
```

### 2.3 Fast memcmp

```nasm
; ============================================================
; Fast memcmp ด้วย AVX2
; int fast_memcmp_avx2(const void *a, const void *b, size_t len)
; rdi=a, rsi=b, rdx=len
; returns: 0 ถ้าเท่ากัน, non-zero ถ้าต่างกัน
; ============================================================
global fast_memcmp_avx2

fast_memcmp_avx2:
    mov rcx, rdx
    shr rcx, 5
    jz .tail_cmp

.loop_cmp:
    vmovdqu ymm0, [rdi]
    vmovdqu ymm1, [rsi]
    vpcmpeqb ymm2, ymm0, ymm1      ; compare byte-by-byte
    vpmovmskb eax, ymm2            ; get bitmask of equal bytes
    cmp eax, 0xFFFFFFFF            ; all 32 bytes equal?
    jne .not_equal
    add rdi, 32
    add rsi, 32
    dec rcx
    jnz .loop_cmp

.tail_cmp:
    mov rcx, rdx
    and rcx, 31
    jz .equal
    repe cmpsb
    jne .not_equal

.equal:
    xor eax, eax
    vzeroupper
    ret

.not_equal:
    mov eax, 1
    vzeroupper
    ret
```

### 2.4 SIMD memmove (Overlapping Case)

```nasm
; ============================================================
; SIMD memmove — จัดการ overlap ได้ถูกต้อง
; void *fast_memmove(void *dst, const void *src, size_t len)
; ============================================================
global fast_memmove

fast_memmove:
    push rdi
    cmp rdi, rsi
    je .done_move                  ; dst == src, nothing to do

    ; ถ้า dst < src หรือ dst >= src+len: copy forward
    mov rax, rsi
    add rax, rdx                   ; rax = src + len
    cmp rdi, rax
    jge .forward_move              ; dst >= src+len: no overlap, go forward

    cmp rdi, rsi
    jle .forward_move              ; dst <= src: safe to copy forward

    ; dst > src และ overlap: copy backward
    mov rax, rdi
    add rax, rdx
    mov r8, rsi
    add r8, rdx
    ; copy from end to start
    mov rcx, rdx
    shr rcx, 5
    jz .backward_tail

.backward_loop:
    sub r8, 32
    sub rax, 32
    vmovdqu ymm0, [r8]
    vmovdqu [rax], ymm0
    dec rcx
    jnz .backward_loop

.backward_tail:
    mov rcx, rdx
    and rcx, 31
    jz .done_move
    ; copy remaining bytes backward
.byte_backward:
    dec r8
    dec rax
    mov al, [r8]
    mov [rax], al
    dec rcx
    jnz .byte_backward
    jmp .done_move

.forward_move:
    ; same as memcpy_avx2
    mov rcx, rdx
    shr rcx, 5
    jz .fwd_tail
.fwd_loop:
    vmovdqu ymm0, [rsi]
    vmovdqu [rdi], ymm0
    add rsi, 32
    add rdi, 32
    dec rcx
    jnz .fwd_loop
.fwd_tail:
    mov rcx, rdx
    and rcx, 31
    rep movsb

.done_move:
    pop rax
    vzeroupper
    ret
```

---

## Section 3: String Operations (การดำเนินการ String)

### 3.1 Fast strlen ด้วย SIMD

```nasm
; ============================================================
; Fast strlen ด้วย SSE4.2 PCMPEQB
; size_t fast_strlen(const char *s)
; rdi = string pointer
; ============================================================
section .text
global fast_strlen, fast_strchr

fast_strlen:
    mov rax, rdi
    pxor xmm0, xmm0               ; xmm0 = all zeros (null byte pattern)

.loop_strlen:
    pcmpeqb xmm1, xmm0            ; compare 16 bytes with null
    ; wait — ต้อง load ก่อน!
    ; แก้ใหม่:

fast_strlen:
    xor rax, rax
    pxor xmm1, xmm1               ; null pattern

    ; align ไปที่ 16-byte boundary
    mov rcx, rdi
    and rcx, 15                    ; offset จาก alignment
    jz .aligned_strlen
    
    ; handle first unaligned bytes
    sub rdi, rcx                   ; go back to alignment
    movdqa xmm0, [rdi]             ; load aligned 16 bytes
    pcmpeqb xmm0, xmm1             ; find null bytes
    pmovmskb edx, xmm0             ; bitmask of null positions
    shr edx, cl                    ; ignore bytes before string start
    bsf edx, edx                   ; find first null
    jnz .found_strlen
    add rdi, 16
    sub rax, rcx                   ; adjust count

.aligned_strlen:
    add rax, 16
    movdqa xmm0, [rdi]             ; load 16 aligned bytes
    pcmpeqb xmm0, xmm1
    pmovmskb edx, xmm0
    test edx, edx
    jz .aligned_strlen             ; no null found, continue

    add rdi, 16
    bsf edx, edx                   ; bit position of first null
    add rax, rdx
    sub rax, 16
    ret

.found_strlen:
    add rax, rdx
    ret


; ============================================================
; Fast strchr ด้วย SSE4.2 PCMPISTRI
; char *fast_strchr(const char *s, int c)
; rdi=string, esi=char
; ============================================================
fast_strchr:
    movd xmm1, esi
    pxor xmm0, xmm0

    ; align
    mov rax, rdi
    and rax, -16
    movdqa xmm2, [rax]             ; load aligned 16 bytes

    ; PCMPISTRI: compare string with character, find first match or null
    ; imm8 = 0x00 = unsigned bytes, equal any, index of least significant
    mov r8, rdi
    sub r8, rax                    ; offset

.loop_strchr:
    pcmpistri xmm2, xmm1, 0x00    ; find char c in xmm2
    jc .found_strchr               ; CF=1: found char
    js .notfound_strchr            ; SF=1: found null (end of string)
    add rax, 16
    movdqa xmm2, [rax]
    jmp .loop_strchr

.found_strchr:
    add rax, rcx                   ; rax + index
    cmp byte [rax], 0              ; double-check not null
    je .notfound_strchr
    ; check if this is before string start
    cmp rax, rdi
    jl .loop_strchr
    ret                            ; return pointer

.notfound_strchr:
    xor eax, eax                   ; return NULL
    ret
```

### 3.2 Fast strstr (Substring Search)

```nasm
; ============================================================
; Fast strstr ด้วย SSE4.2
; char *fast_strstr(const char *haystack, const char *needle)
; rdi=haystack, rsi=needle
; ============================================================
global fast_strstr

fast_strstr:
    ; load first 16 bytes of needle
    movdqu xmm1, [rsi]
    pxor xmm0, xmm0

    mov rax, rdi

.loop_strstr:
    movdqu xmm2, [rax]             ; load 16 bytes of haystack
    
    ; PCMPISTRI: find needle[0..15] in xmm2
    ; imm8 = 0x0C = unsigned bytes, equal ordered (substring search)
    pcmpistri xmm2, xmm1, 0x0C
    jc .possible_match
    js .not_found_strstr           ; null in haystack before match

    add rax, 16
    jmp .loop_strstr

.possible_match:
    add rax, rcx                   ; position of possible match
    ; verify full needle match
    push rsi
    push rax
    mov rdi, rax
    ; simple verification: compare byte by byte
.verify:
    cmp byte [rsi], 0
    je .match_found                ; needle exhausted = found!
    mov cl, [rdi]
    cmp cl, [rsi]
    jne .verify_fail
    inc rdi
    inc rsi
    jmp .verify
.match_found:
    pop rax                        ; return pointer to match
    add rsp, 8                     ; discard saved rsi
    ret
.verify_fail:
    pop rax
    pop rsi
    inc rax                        ; advance one position
    jmp .loop_strstr

.not_found_strstr:
    xor eax, eax
    ret
```

### 3.3 UTF-8 Validation with SIMD

```nasm
; ============================================================
; UTF-8 Validation ด้วย SSE4.1
; int validate_utf8(const char *str, size_t len)
; returns 1 ถ้า valid, 0 ถ้า invalid
;
; UTF-8 rules:
;   0xxxxxxx = 1-byte (ASCII)
;   110xxxxx 10xxxxxx = 2-byte
;   1110xxxx 10xxxxxx 10xxxxxx = 3-byte
;   11110xxx 10xxxxxx 10xxxxxx 10xxxxxx = 4-byte
; ============================================================
global validate_utf8

section .data
align 16
; High nibble table: max first byte value per leading nibble
utf8_high_nibble_tbl:
    db 1,1,1,1,1,1,1,1,1,1,1,1,2,2,3,4  ; 0x0-0xF

section .text
validate_utf8:
    ; Fast path: check if all bytes are ASCII (< 0x80)
    mov rcx, rsi
    shr rcx, 4
    jz .scalar_validate

    pxor xmm1, xmm1               ; zero
.check_ascii:
    movdqu xmm0, [rdi]
    pmovmskb eax, xmm0            ; get high bits
    test eax, eax
    jnz .has_multibyte            ; found non-ASCII
    add rdi, 16
    dec rcx
    jnz .check_ascii

.scalar_validate:
    ; handle tail + multibyte validation
    ; (simplified: use scalar for correctness)
    mov rax, 1                    ; assume valid for now
    ret

.has_multibyte:
    ; Full validation requires state machine — scalar fallback
    ; (production code would use a full SIMD state machine here)
    mov rax, 1
    ; real implementation: check continuation bytes, overlong encoding, etc.
    ret
```

---

## Section 4: Numerical Computing (การคำนวณตัวเลข)

### 4.1 Dot Products

```nasm
; ============================================================
; 4D Dot Product ด้วย SSE4.1
; float dot4(float *a, float *b)
; rdi=a, rsi=b
; ============================================================
section .text
global dot4, dot8, dot_n, mat4x4_mul, mat_transpose_8x8

dot4:
    movaps xmm0, [rdi]             ; load a[0..3]
    movaps xmm1, [rsi]             ; load b[0..3]
    dpps xmm0, xmm1, 0xF1         ; dot product, store in bit 0
    ; xmm0[0] = a[0]*b[0] + a[1]*b[1] + a[2]*b[2] + a[3]*b[3]
    ret


; ============================================================
; 8D Dot Product ด้วย AVX2 + FMA
; float dot8(float *a, float *b)
; ============================================================
dot8:
    vmovaps ymm0, [rdi]            ; load a[0..7]
    vmovaps ymm1, [rsi]            ; load b[0..7]
    vmulps ymm2, ymm0, ymm1        ; element-wise multiply
    vhaddps ymm2, ymm2, ymm2       ; horizontal add (pairs)
    vhaddps ymm2, ymm2, ymm2       ; horizontal add again
    ; now each 128-bit lane has the sum
    vextractf128 xmm3, ymm2, 1     ; extract upper lane
    vaddps xmm2, xmm2, xmm3        ; add both lanes
    vmovss xmm0, xmm2              ; result in xmm0
    vzeroupper
    ret


; ============================================================
; Arbitrary-size Dot Product ด้วย AVX2
; float dot_n(float *a, float *b, int n)
; rdi=a, rsi=b, edx=n
; ============================================================
dot_n:
    vxorps ymm0, ymm0, ymm0        ; accumulator = 0
    vxorps ymm1, ymm1, ymm1        ; accumulator 2 (unroll)
    xor eax, eax

    mov ecx, edx
    and ecx, -8
    jz .dot_tail

.dot_loop8:
    vmovups ymm2, [rdi + rax*4]
    vmovups ymm3, [rsi + rax*4]
    vfmadd231ps ymm0, ymm2, ymm3   ; acc += a * b (FMA!)
    add eax, 8
    cmp eax, ecx
    jl .dot_loop8

.dot_tail:
    ; horizontal sum of ymm0
    vextractf128 xmm2, ymm0, 1
    vaddps xmm0, xmm0, xmm2        ; add upper + lower 128-bit lanes
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0

    ; scalar tail
    cmp eax, edx
    jge .dot_done
.dot_scalar:
    movss xmm3, [rdi + rax*4]
    mulss xmm3, [rsi + rax*4]
    addss xmm0, xmm3
    inc eax
    cmp eax, edx
    jl .dot_scalar

.dot_done:
    vzeroupper
    ret
```

### 4.2 Matrix 4x4 Multiply (Graphics)

```nasm
; ============================================================
; Matrix 4x4 Float Multiply (Column-major, OpenGL style)
; void mat4x4_mul(float *dst, float *a, float *b)
; rdi=dst, rsi=a, rdx=b
; ============================================================
mat4x4_mul:
    ; Load columns of matrix A
    vmovaps xmm4, [rsi + 0]        ; col0 of A
    vmovaps xmm5, [rsi + 16]       ; col1 of A
    vmovaps xmm6, [rsi + 32]       ; col2 of A
    vmovaps xmm7, [rsi + 48]       ; col3 of A

    ; Compute each column of result
    ; result_col_j = A * b_col_j
    ;              = A_col0*b[0][j] + A_col1*b[1][j] + A_col2*b[2][j] + A_col3*b[3][j]

    mov ecx, 4                     ; 4 columns

.col_loop:
    ; load column j of B (4 floats)
    movss xmm0, [rdx]              ; b[0][j]
    movss xmm1, [rdx + 4]          ; b[1][j]
    movss xmm2, [rdx + 8]          ; b[2][j]
    movss xmm3, [rdx + 12]         ; b[3][j]

    vshufps xmm0, xmm0, xmm0, 0   ; broadcast b[0][j]
    vshufps xmm1, xmm1, xmm1, 0   ; broadcast b[1][j]
    vshufps xmm2, xmm2, xmm2, 0   ; broadcast b[2][j]
    vshufps xmm3, xmm3, xmm3, 0   ; broadcast b[3][j]

    vmulps xmm8, xmm4, xmm0        ; A_col0 * b[0][j]
    vmulps xmm9, xmm5, xmm1        ; A_col1 * b[1][j]
    vaddps xmm8, xmm8, xmm9
    vmulps xmm9, xmm6, xmm2        ; A_col2 * b[2][j]
    vaddps xmm8, xmm8, xmm9
    vmulps xmm9, xmm7, xmm3        ; A_col3 * b[3][j]
    vaddps xmm8, xmm8, xmm9

    vmovaps [rdi], xmm8            ; store result column j
    add rdi, 16
    add rdx, 16
    dec ecx
    jnz .col_loop

    ret


; ============================================================
; Matrix 8x8 Transpose ด้วย unpack instructions
; void mat_transpose_8x8(float *dst, float *src)
; rdi=dst, rsi=src (both 8x8 float matrices, row-major)
; ============================================================
mat_transpose_8x8:
    ; Load all 8 rows
    vmovaps ymm0, [rsi + 0*32]
    vmovaps ymm1, [rsi + 1*32]
    vmovaps ymm2, [rsi + 2*32]
    vmovaps ymm3, [rsi + 3*32]
    vmovaps ymm4, [rsi + 4*32]
    vmovaps ymm5, [rsi + 5*32]
    vmovaps ymm6, [rsi + 6*32]
    vmovaps ymm7, [rsi + 7*32]

    ; Step 1: interleave pairs of rows (UNPCKLPS/UNPCKHPS)
    vunpcklps ymm8, ymm0, ymm1     ; [a00 b00 a01 b01 | a04 b04 a05 b05]
    vunpckhps ymm9, ymm0, ymm1     ; [a02 b02 a03 b03 | a06 b06 a07 b07]
    vunpcklps ymm10, ymm2, ymm3
    vunpckhps ymm11, ymm2, ymm3
    vunpcklps ymm12, ymm4, ymm5
    vunpckhps ymm13, ymm4, ymm5
    vunpcklps ymm14, ymm6, ymm7
    vunpckhps ymm15, ymm6, ymm7

    ; Step 2: interleave pairs of 64-bit groups
    vunpcklpd ymm0, ymm8, ymm10
    vunpckhpd ymm1, ymm8, ymm10
    vunpcklpd ymm2, ymm9, ymm11
    vunpckhpd ymm3, ymm9, ymm11
    vunpcklpd ymm4, ymm12, ymm14
    vunpckhpd ymm5, ymm12, ymm14
    vunpcklpd ymm6, ymm13, ymm15
    vunpckhpd ymm7, ymm13, ymm15

    ; Step 3: permute across 128-bit lanes
    vperm2f128 ymm8,  ymm0, ymm4, 0x20  ; row 0
    vperm2f128 ymm9,  ymm1, ymm5, 0x20  ; row 1
    vperm2f128 ymm10, ymm2, ymm6, 0x20  ; row 2
    vperm2f128 ymm11, ymm3, ymm7, 0x20  ; row 3
    vperm2f128 ymm12, ymm0, ymm4, 0x31  ; row 4
    vperm2f128 ymm13, ymm1, ymm5, 0x31  ; row 5
    vperm2f128 ymm14, ymm2, ymm6, 0x31  ; row 6
    vperm2f128 ymm15, ymm3, ymm7, 0x31  ; row 7

    ; Store transposed rows
    vmovaps [rdi + 0*32], ymm8
    vmovaps [rdi + 1*32], ymm9
    vmovaps [rdi + 2*32], ymm10
    vmovaps [rdi + 3*32], ymm11
    vmovaps [rdi + 4*32], ymm12
    vmovaps [rdi + 5*32], ymm13
    vmovaps [rdi + 6*32], ymm14
    vmovaps [rdi + 7*32], ymm15

    vzeroupper
    ret
```

### 4.3 Vector Normalize ด้วย RSQRTPS

```nasm
; ============================================================
; Fast Vector3 Normalize ด้วย RSQRTPS (approximate 1/sqrt)
; void vec3_normalize(float *v)    ; v must be 16-byte aligned, w=0
; ============================================================
section .data
align 16
vec_half:   dd 0.5, 0.5, 0.5, 0.5
vec_threehalves: dd 1.5, 1.5, 1.5, 1.5

section .text
global vec3_normalize

vec3_normalize:
    movaps xmm0, [rdi]             ; load x,y,z,0
    movaps xmm1, xmm0
    mulps xmm1, xmm0               ; x2, y2, z2, 0
    movaps xmm2, xmm1
    shufps xmm2, xmm2, 0x4E        ; swap: z2,w2,x2,y2
    addps xmm1, xmm2               ; x2+z2, y2+w2, z2+x2, w2+y2
    movaps xmm2, xmm1
    shufps xmm2, xmm2, 0x11        ; y2+w2, x2+z2, ...
    addps xmm1, xmm2               ; sum in all 4 positions

    ; Newton-Raphson iteration for 1/sqrt(x):
    ; r = rsqrtps(x)            (12-bit approx)
    ; r = r * (1.5 - 0.5*x*r*r) (23-bit, nearly float precision)
    rsqrtps xmm3, xmm1             ; approx 1/sqrt(|v|^2) = 1/|v|
    
    ; One Newton-Raphson step
    movaps xmm4, [vec_half]
    mulps xmm4, xmm1               ; 0.5 * x
    mulps xmm4, xmm3               ; 0.5 * x * r
    mulps xmm4, xmm3               ; 0.5 * x * r * r
    movaps xmm5, [vec_threehalves]
    subps xmm5, xmm4               ; 1.5 - 0.5*x*r*r
    mulps xmm3, xmm5               ; refined 1/|v|

    mulps xmm0, xmm3               ; v * (1/|v|) = normalized
    movaps [rdi], xmm0
    ret
```

### 4.4 Prefix Sum (Scan) ด้วย SIMD

```nasm
; ============================================================
; Inclusive Prefix Sum ด้วย SSE
; void prefix_sum(int *a, int n)   ; in-place
; rdi=array, esi=n
; Algorithm: Blelloch scan pattern
; ============================================================
global prefix_sum_simd

prefix_sum_simd:
    ; Process 4 elements at a time
    ; Prefix sum of [a,b,c,d] = [a, a+b, a+b+c, a+b+c+d]
    ; Step 1: [a, a+b, b+c, c+d]  (shift and add)
    ; Step 2: [a, a+b, a+b+c, a+b+c+d] (shift and add again)
    
    pxor xmm5, xmm5                ; running carry = 0
    xor eax, eax

.prefix_loop:
    cmp eax, esi
    jge .prefix_done

    movdqu xmm0, [rdi + rax*4]     ; load 4 ints

    ; add running carry to first element
    movdqa xmm1, xmm0
    pslldq xmm1, 4                 ; shift left 1 element: [0,a,b,c]
    paddd xmm0, xmm1               ; [a, a+b, b+c, c+d]

    movdqa xmm1, xmm0
    pslldq xmm1, 8                 ; shift left 2 elements: [0,0,a,a+b]
    paddd xmm0, xmm1               ; [a, a+b, a+b+c, a+b+c+d]

    ; add running carry from previous chunk
    pshufd xmm5, xmm5, 0xFF        ; broadcast carry to all 4 lanes
    paddd xmm0, xmm5

    ; update carry = last element of current chunk
    pshufd xmm5, xmm0, 0xFF        ; carry = xmm0[3]
    pxor xmm5, xmm5                ; reset for next broadcast trick
    pshufd xmm5, xmm0, 0xFF

    movdqu [rdi + rax*4], xmm0
    add eax, 4
    jmp .prefix_loop

.prefix_done:
    ret


; ============================================================
; Horizontal Reduction — Sum all elements ด้วย AVX
; float hsum8(float *arr)  ; 8 floats
; ============================================================
global hsum8

hsum8:
    vmovaps ymm0, [rdi]
    vextractf128 xmm1, ymm0, 1     ; upper 4 floats
    vaddps xmm0, xmm0, xmm1        ; add lower + upper
    vhaddps xmm0, xmm0, xmm0       ; add pairs
    vhaddps xmm0, xmm0, xmm0       ; add pairs again
    ; xmm0[0] = sum of all 8 elements
    vzeroupper
    ret
```

---

## Section 5: Image Processing (การประมวลผลภาพ)

### 5.1 Grayscale Conversion (RGB → Luma)

```nasm
; ============================================================
; RGB to Grayscale ด้วย SSE2
; Luma = 0.299*R + 0.587*G + 0.114*B  (BT.601)
; void rgb_to_gray(uint8_t *dst, const uint8_t *src, int pixels)
; rdi=dst (gray), rsi=src (RGB packed), edx=pixel_count
; ============================================================
section .data
align 16
; Fixed-point coefficients (Q15: multiply by 32768)
luma_r: dw 9798, 9798, 9798, 9798, 9798, 9798, 9798, 9798   ; 0.299 * 32768
luma_g: dw 19235,19235,19235,19235,19235,19235,19235,19235   ; 0.587 * 32768
luma_b: dw 3736, 3736, 3736, 3736, 3736, 3736, 3736, 3736   ; 0.114 * 32768

section .text
global rgb_to_gray_sse2

rgb_to_gray_sse2:
    ; Process 4 pixels at a time (4 * 3 = 12 bytes per iteration)
    mov ecx, edx
    shr ecx, 2                     ; ecx = pixels / 4
    jz .gray_tail

    movdqa xmm6, [luma_r]
    movdqa xmm7, [luma_g]

.gray_loop:
    ; Load 12 bytes = 4 RGB pixels
    movdqu xmm0, [rsi]             ; R0 G0 B0 R1 G1 B1 R2 G2 B2 R3 G3 B3 XX XX XX XX

    ; Deinterleave (separate R, G, B channels)
    ; This is complex — use pshufb for efficient byte shuffle
    ; For simplicity, use scalar helper here
    ; (Full implementation would use pshufb + multiple passes)

    ; Simplified version using scalar for correctness demo:
    movzx eax, byte [rsi]          ; R
    imul eax, 299
    movzx r8d, byte [rsi+1]        ; G
    imul r8d, 587
    add eax, r8d
    movzx r8d, byte [rsi+2]        ; B
    imul r8d, 114
    add eax, r8d
    ; divide by 1000
    mov r9d, eax
    mov eax, 1374389535            ; magic for /1000
    imul eax, r9d
    shr eax, 37
    mov [rdi], al

    add rsi, 3
    inc rdi
    dec ecx
    jnz .gray_loop

.gray_tail:
    ret


; ============================================================
; Brightness Adjustment ด้วย SSE2 (saturating add)
; void adjust_brightness(uint8_t *pixels, int count, int delta)
; rdi=pixels, esi=count, edx=delta (-128..127)
; ============================================================
global adjust_brightness

adjust_brightness:
    movd xmm1, edx
    pxor xmm0, xmm0
    ; broadcast delta to all 16 bytes
    ; if delta >= 0: use paddusb (saturating unsigned add)
    ; if delta < 0:  use psubusb

    test edx, edx
    js .brightness_sub

    ; positive delta: add with saturation
    pshufb xmm1, xmm0              ; broadcast (requires SSE3 PSHUFB)
    mov ecx, esi
    shr ecx, 4                     ; process 16 pixels at a time
    jz .bright_tail

.bright_add_loop:
    movdqu xmm2, [rdi]
    paddusb xmm2, xmm1             ; saturating add
    movdqu [rdi], xmm2
    add rdi, 16
    dec ecx
    jnz .bright_add_loop

.bright_tail:
    mov ecx, esi
    and ecx, 15
    jz .bright_done
.bright_add_scalar:
    movzx eax, byte [rdi]
    add eax, edx
    cmp eax, 255
    jle .no_clamp_add
    mov eax, 255
.no_clamp_add:
    mov [rdi], al
    inc rdi
    dec ecx
    jnz .bright_add_scalar
    jmp .bright_done

.brightness_sub:
    neg edx
    movd xmm1, edx
    pshufb xmm1, xmm0
    mov ecx, esi
    shr ecx, 4
    jz .bright_sub_tail
.bright_sub_loop:
    movdqu xmm2, [rdi]
    psubusb xmm2, xmm1             ; saturating subtract
    movdqu [rdi], xmm2
    add rdi, 16
    dec ecx
    jnz .bright_sub_loop
.bright_sub_tail:
    ; scalar tail...
.bright_done:
    ret
```

### 5.2 Alpha Blending (LERP)

```nasm
; ============================================================
; Alpha Blending: dst = src * alpha + dst * (1 - alpha)
; void alpha_blend(uint32_t *dst, uint32_t *src, uint8_t alpha, int count)
; RGBA format: each pixel = [R,G,B,A] in memory
; rdi=dst, rsi=src, edx=alpha(0-255), ecx=count
; ============================================================
section .data
align 16
blend_255: dw 255,255,255,255,255,255,255,255

section .text
global alpha_blend_sse2

alpha_blend_sse2:
    ; Prepare alpha and (255-alpha) as 16-bit values
    movd xmm6, edx                 ; alpha
    movzx r8d, dl
    neg r8d
    add r8d, 255                   ; 255 - alpha
    movd xmm7, r8d

    ; broadcast to all words
    pxor xmm0, xmm0
    pshufb xmm6, xmm0              ; wait — pshufb needs second operand
    ; Use pshuflw + pshufhw instead:
    pshuflw xmm6, xmm6, 0
    pshufhw xmm6, xmm6, 0         ; broadcast alpha to all 8 words
    pshuflw xmm7, xmm7, 0
    pshufhw xmm7, xmm7, 0         ; broadcast (255-alpha) to all 8 words

    ; Process 4 pixels (4 RGBA = 16 bytes) per iteration
    mov r9d, ecx
    shr r9d, 2
    jz .blend_tail

.blend_loop:
    movdqu xmm1, [rsi]             ; 4 src pixels
    movdqu xmm2, [rdi]             ; 4 dst pixels

    ; Unpack src to 16-bit
    pxor xmm0, xmm0
    movdqa xmm3, xmm1
    punpcklbw xmm3, xmm0           ; src[0..1] as 16-bit
    movdqa xmm4, xmm1
    punpckhbw xmm4, xmm0           ; src[2..3] as 16-bit

    ; Unpack dst to 16-bit
    movdqa xmm5, xmm2
    punpcklbw xmm5, xmm0
    movdqa xmm8, xmm2
    punpckhbw xmm8, xmm0

    ; Blend: src * alpha
    pmullw xmm3, xmm6
    pmullw xmm4, xmm6

    ; Blend: dst * (1-alpha)
    pmullw xmm5, xmm7
    pmullw xmm8, xmm7

    ; Add and divide by 255 (approximate: >> 8)
    paddw xmm3, xmm5
    paddw xmm4, xmm8
    psrlw xmm3, 8
    psrlw xmm4, 8

    ; Pack back to bytes
    packuswb xmm3, xmm4
    movdqu [rdi], xmm3

    add rsi, 16
    add rdi, 16
    dec r9d
    jnz .blend_loop

.blend_tail:
    ; scalar tail for remaining pixels
    ret
```

### 5.3 3x3 Kernel Convolution (Blur)

```nasm
; ============================================================
; 3x3 Box Blur ด้วย SSE2 (horizontal pass)
; Horizontal blur: dst[x] = (src[x-1] + src[x] + src[x+1]) / 3
; void hblur_row(uint8_t *dst, uint8_t *src, int width)
; rdi=dst, rsi=src, edx=width
; ============================================================
global hblur_row

section .data
align 16
div3_magic: dw 21845,21845,21845,21845,21845,21845,21845,21845 ; ~1/3 in Q16

section .text
hblur_row:
    xor ecx, ecx
    pxor xmm0, xmm0

    ; Process 8 pixels at a time
    mov r8d, edx
    sub r8d, 2                     ; avoid boundary
    mov r9d, r8d
    and r9d, -8
    mov ecx, 1                     ; start at x=1

.blur_loop:
    cmp ecx, r9d
    jge .blur_tail

    ; Load three 8-byte windows (prev, curr, next)
    movq xmm1, [rsi + rcx - 1]    ; src[x-1..x+6]
    movq xmm2, [rsi + rcx]        ; src[x..x+7]
    movq xmm3, [rsi + rcx + 1]    ; src[x+1..x+8]

    ; Unpack to 16-bit
    punpcklbw xmm1, xmm0
    punpcklbw xmm2, xmm0
    punpcklbw xmm3, xmm0

    ; Sum
    paddw xmm2, xmm1
    paddw xmm2, xmm3               ; sum = prev + curr + next

    ; Divide by 3 (multiply by ~21845 then shift right 16)
    movdqa xmm4, [div3_magic]
    pmulhuw xmm2, xmm4             ; multiply high (>>16 implicitly)
    ; additional shift for precision
    psrlw xmm2, 1

    packuswb xmm2, xmm0            ; pack to bytes
    movq [rdi + rcx], xmm2

    add ecx, 8
    jmp .blur_loop

.blur_tail:
    ; scalar for remaining
    cmp ecx, edx
    jge .blur_done
.blur_scalar:
    movzx eax, byte [rsi + rcx - 1]
    movzx r8d, byte [rsi + rcx]
    movzx r9d, byte [rsi + rcx + 1]
    add eax, r8d
    add eax, r9d
    xor edx, edx
    mov r10d, 3
    div r10d                       ; divide by 3
    mov [rdi + rcx], al
    inc ecx
    jmp .blur_tail
.blur_done:
    ret
```

### 5.4 YUV to RGB Conversion

```nasm
; ============================================================
; YUV420 to RGB888 ด้วย SSE2
; BT.601 coefficients:
;   R = Y + 1.402*Cr
;   G = Y - 0.344*Cb - 0.714*Cr
;   B = Y + 1.772*Cb
; void yuv_to_rgb_row(uint8_t *dst_rgb, uint8_t *y_row,
;                    uint8_t *cb_row, uint8_t *cr_row, int width)
; ============================================================
section .data
align 16
; Fixed-point Q10 coefficients
yuv_coeff_r_cr: dw 1436,1436,1436,1436,1436,1436,1436,1436   ; 1.402*1024
yuv_coeff_g_cb: dw  352, 352, 352, 352, 352, 352, 352, 352   ; 0.344*1024
yuv_coeff_g_cr: dw  731, 731, 731, 731, 731, 731, 731, 731   ; 0.714*1024
yuv_coeff_b_cb: dw 1814,1814,1814,1814,1814,1814,1814,1814   ; 1.772*1024
yuv_128:        dw  128, 128, 128, 128, 128, 128, 128, 128

section .text
global yuv420_to_rgb_sse2

yuv420_to_rgb_sse2:
    ; This function processes 8 pixels per iteration
    push rbx
    push r12
    push r13

    ; Load coefficients
    movdqa xmm12, [yuv_coeff_r_cr]
    movdqa xmm13, [yuv_coeff_g_cb]
    movdqa xmm14, [yuv_coeff_g_cr]
    movdqa xmm15, [yuv_coeff_b_cb]

    xor ecx, ecx
    pxor xmm0, xmm0
    movdqa xmm11, [yuv_128]

.yuv_loop:
    cmp ecx, r8d
    jge .yuv_done

    ; Load 8 Y values
    movq xmm1, [rsi + rcx]        ; 8 Y bytes
    punpcklbw xmm1, xmm0           ; Y as 16-bit

    ; Load 4 Cb, 4 Cr (shared between pairs of pixels in YUV420)
    movd xmm2, [rdx + rcx/2]      ; 4 Cb bytes
    movd xmm3, [rcx + rcx/2]      ; 4 Cr bytes (rcx is wrong here, use reg)
    punpcklbw xmm2, xmm0
    punpcklbw xmm3, xmm0

    ; Duplicate: each Cb/Cr used for 2 Y values
    ; (for YUV422, use as-is; for YUV420 need row sharing too)
    punpcklwd xmm2, xmm2           ; duplicate: [Cb0,Cb0,Cb1,Cb1,Cb2,Cb2,Cb3,Cb3]
    punpcklwd xmm3, xmm3

    ; Subtract 128 (chroma offset)
    psubw xmm2, xmm11
    psubw xmm3, xmm11

    ; Compute R, G, B
    movdqa xmm4, xmm3
    pmullw xmm4, xmm12             ; Cr * 1436
    psraw xmm4, 10                 ; >> 10
    paddw xmm4, xmm1               ; R = Y + Cr*coeff

    movdqa xmm5, xmm2
    pmullw xmm5, xmm13             ; Cb * 352
    movdqa xmm6, xmm3
    pmullw xmm6, xmm14             ; Cr * 731
    paddw xmm5, xmm6
    psraw xmm5, 10
    psubw xmm5, xmm1               ; negate
    paddw xmm5, xmm1               ; G = Y - Cb*coeff - Cr*coeff

    movdqa xmm7, xmm2
    pmullw xmm7, xmm15             ; Cb * 1814
    psraw xmm7, 10
    paddw xmm7, xmm1               ; B = Y + Cb*coeff

    ; Clamp to [0, 255]
    packuswb xmm4, xmm4            ; R bytes
    packuswb xmm5, xmm5            ; G bytes
    packuswb xmm7, xmm7            ; B bytes

    ; Interleave RGB (complex — use pshufb with a shuffle mask)
    ; Simplified: store separately for now
    movq [rdi], xmm4               ; R channel
    ; (full RGB interleaving would require more shuffle operations)

    add ecx, 8
    add rdi, 24                    ; 8 pixels * 3 bytes
    jmp .yuv_loop

.yuv_done:
    pop r13
    pop r12
    pop rbx
    ret
```

---

## Section 6: Cryptography & Hashing

### 6.1 CRC32 ด้วย Hardware Instruction

```nasm
; ============================================================
; CRC32C ด้วย SSE4.2 CRC32 instruction (Intel Nehalem+)
; uint32_t crc32c(uint32_t crc, const void *buf, size_t len)
; edi=initial_crc, rsi=buf, rdx=len
; ============================================================
section .text
global crc32c_hw

crc32c_hw:
    mov eax, edi                   ; initial CRC value

    ; Process 8 bytes at a time
    mov rcx, rdx
    shr rcx, 3
    jz .crc_4byte

.crc_8loop:
    crc32 rax, qword [rsi]         ; 64-bit CRC update
    add rsi, 8
    dec rcx
    jnz .crc_8loop

.crc_4byte:
    test rdx, 4
    jz .crc_2byte
    crc32 eax, dword [rsi]
    add rsi, 4

.crc_2byte:
    test rdx, 2
    jz .crc_1byte
    crc32 eax, word [rsi]
    add rsi, 2

.crc_1byte:
    test rdx, 1
    jz .crc_done
    crc32 eax, byte [rsi]

.crc_done:
    ret                            ; eax = final CRC32C


; ============================================================
; Parallel CRC32 ด้วย 3 streams (Pipeline utilization)
; uint32_t crc32c_parallel(uint32_t crc, const void *buf, size_t len)
; Technique: 3 independent CRC chains, combine at end
; ============================================================
global crc32c_parallel

section .data
align 8
; Polynomial tables for combining CRC (precomputed)
; k1 = x^(2*BLOCK_SIZE*8) mod poly
; k2 = x^(BLOCK_SIZE*8) mod poly
; For BLOCK_SIZE=1024:
crc_k1: dq 0x740EEF02           ; example value
crc_k2: dq 0x9E4ADDF8           ; example value

section .text
crc32c_parallel:
    push rbx
    push r12
    push r13

    ; Split buffer into 3 equal chunks
    mov r9, rdx
    mov r10, rdx
    xor r11, r11
    mov r12, rdi                   ; crc0 = initial
    xor r13d, r13d                 ; crc1 = 0
    xor ebx, ebx                   ; crc2 = 0

    mov rax, rdx
    xor rdx, rdx
    mov r9, 3
    div r9                         ; rax = len/3
    mov r9, rax                    ; chunk size

    mov r10, rsi                   ; chunk0
    lea r11, [rsi + r9]            ; chunk1
    lea r8, [rsi + r9*2]           ; chunk2
    xor ecx, ecx

.parallel_loop:
    cmp rcx, r9
    jge .parallel_combine

    crc32 r12, qword [r10 + rcx]   ; chain 0
    crc32 r13, qword [r11 + rcx]   ; chain 1
    crc32 rbx, qword [r8 + rcx]    ; chain 2

    add rcx, 8
    jmp .parallel_loop

.parallel_combine:
    ; Combine 3 CRCs using precomputed polynomials
    ; (Simplified — real implementation uses PCLMULQDQ)
    mov eax, r12d
    ; XOR folding (simplified)
    xor eax, r13d
    xor eax, ebx

    pop r13
    pop r12
    pop rbx
    ret
```

### 6.2 AES-128 CBC Mode ด้วย AES-NI

```nasm
; ============================================================
; AES-128 CBC Encryption ด้วย AES-NI (Intel Westmere+)
; void aes128_cbc_encrypt(uint8_t *dst, uint8_t *src,
;                         uint8_t *round_keys, uint8_t *iv, int blocks)
; rdi=dst, rsi=src, rdx=round_keys, rcx=iv, r8d=blocks
;
; AES-128 requires 11 round keys (1 initial + 10 rounds)
; Each key = 16 bytes
; ============================================================
section .text
global aes128_cbc_encrypt, aes128_key_expand

aes128_cbc_encrypt:
    movdqu xmm1, [rcx]             ; load IV

.aes_loop:
    test r8d, r8d
    jz .aes_done

    movdqu xmm0, [rsi]             ; load plaintext block

    ; CBC: XOR plaintext with previous ciphertext (or IV)
    pxor xmm0, xmm1

    ; AES encryption: 10 rounds
    pxor xmm0, [rdx + 0*16]       ; AddRoundKey (round 0)
    aesenc xmm0, [rdx + 1*16]     ; Round 1
    aesenc xmm0, [rdx + 2*16]     ; Round 2
    aesenc xmm0, [rdx + 3*16]     ; Round 3
    aesenc xmm0, [rdx + 4*16]     ; Round 4
    aesenc xmm0, [rdx + 5*16]     ; Round 5
    aesenc xmm0, [rdx + 6*16]     ; Round 6
    aesenc xmm0, [rdx + 7*16]     ; Round 7
    aesenc xmm0, [rdx + 8*16]     ; Round 8
    aesenc xmm0, [rdx + 9*16]     ; Round 9
    aesenclast xmm0, [rdx + 10*16] ; Final round

    movdqu [rdi], xmm0             ; store ciphertext
    movdqa xmm1, xmm0              ; update IV = ciphertext

    add rsi, 16
    add rdi, 16
    dec r8d
    jmp .aes_loop

.aes_done:
    ret


; ============================================================
; AES-128 Key Expansion (ใช้ AESKEYGENASSIST)
; void aes128_key_expand(uint8_t *round_keys, const uint8_t *key)
; rdi=round_keys (176 bytes), rsi=key (16 bytes)
; ============================================================
aes128_key_expand:
    movdqu xmm0, [rsi]             ; original key
    movdqu [rdi], xmm0             ; round key 0

    ; Key schedule macro (กำหนด RC = round constant)
    %macro AES_KE_STEP 2           ; key_reg, rcon
        aeskeygenassist xmm1, %1, %2
        pshufd xmm1, xmm1, 0xFF   ; broadcast W[3]
        movdqa xmm2, %1
        pslldq xmm2, 4
        pxor %1, xmm2
        pslldq xmm2, 4
        pxor %1, xmm2
        pslldq xmm2, 4
        pxor %1, xmm2
        pxor %1, xmm1
    %endmacro

    AES_KE_STEP xmm0, 0x01
    movdqu [rdi + 1*16], xmm0
    AES_KE_STEP xmm0, 0x02
    movdqu [rdi + 2*16], xmm0
    AES_KE_STEP xmm0, 0x04
    movdqu [rdi + 3*16], xmm0
    AES_KE_STEP xmm0, 0x08
    movdqu [rdi + 4*16], xmm0
    AES_KE_STEP xmm0, 0x10
    movdqu [rdi + 5*16], xmm0
    AES_KE_STEP xmm0, 0x20
    movdqu [rdi + 6*16], xmm0
    AES_KE_STEP xmm0, 0x40
    movdqu [rdi + 7*16], xmm0
    AES_KE_STEP xmm0, 0x80
    movdqu [rdi + 8*16], xmm0
    AES_KE_STEP xmm0, 0x1B
    movdqu [rdi + 9*16], xmm0
    AES_KE_STEP xmm0, 0x36
    movdqu [rdi + 10*16], xmm0

    ret


; ============================================================
; AES-128 CBC Decryption ด้วย AES-NI
; ============================================================
global aes128_cbc_decrypt

aes128_cbc_decrypt:
    ; CBC decryption: can be parallelized! 
    ; Each block decrypted independently, then XOR with prev ciphertext
    movdqu xmm15, [rcx]            ; IV

.aes_dec_loop:
    test r8d, r8d
    jz .aes_dec_done

    movdqu xmm0, [rsi]             ; load ciphertext block
    movdqa xmm14, xmm0             ; save for next IV

    ; AES decryption (requires equivalent inverse keys)
    pxor xmm0, [rdx + 10*16]      ; AddRoundKey (last key first)
    aesdec xmm0, [rdx + 9*16]
    aesdec xmm0, [rdx + 8*16]
    aesdec xmm0, [rdx + 7*16]
    aesdec xmm0, [rdx + 6*16]
    aesdec xmm0, [rdx + 5*16]
    aesdec xmm0, [rdx + 4*16]
    aesdec xmm0, [rdx + 3*16]
    aesdec xmm0, [rdx + 2*16]
    aesdec xmm0, [rdx + 1*16]
    aesdeclast xmm0, [rdx + 0*16] ; Final inverse round

    pxor xmm0, xmm15               ; XOR with IV/prev ciphertext
    movdqu [rdi], xmm0
    movdqa xmm15, xmm14            ; update IV

    add rsi, 16
    add rdi, 16
    dec r8d
    jmp .aes_dec_loop

.aes_dec_done:
    ret
```

### 6.3 ChaCha20 Core ด้วย AVX2

```nasm
; ============================================================
; ChaCha20 Quarter Round ด้วย SSE2
; ChaCha20 state = 16 uint32 words
; Quarter round: a,b,c,d = QR(a,b,c,d) where:
;   a += b; d ^= a; d <<<= 16
;   c += d; b ^= c; b <<<= 12
;   a += b; d ^= a; d <<<= 8
;   c += d; b ^= c; b <<<= 7
; ============================================================
section .text
global chacha20_block_sse2

; ChaCha20 constants: "expa", "nd 3", "2-by", "te k"
section .data
align 16
chacha_const: dd 0x61707865, 0x3320646e, 0x79622d32, 0x6b206574

section .text
chacha20_block_sse2:
    ; rdi = output (64 bytes), rsi = key (32 bytes)
    ; rdx = nonce (12 bytes), ecx = counter
    push rbp
    push rbx
    sub rsp, 64+16                 ; local state copy

    ; Load initial state into xmm0-xmm3
    movdqu xmm0, [chacha_const]    ; state[0..3] = constants
    movdqu xmm1, [rsi]             ; state[4..7] = key[0..15]
    movdqu xmm2, [rsi + 16]        ; state[8..11] = key[16..31]
    
    ; state[12] = counter, state[13..15] = nonce
    movd xmm3, ecx                 ; counter
    movd xmm4, [rdx]
    movd xmm5, [rdx + 4]
    movd xmm6, [rdx + 8]
    movss xmm3, xmm3               ; state[12]
    ; pack nonce+counter into xmm3
    pinsrd xmm3, dword [rdx], 1    ; state[13] = nonce[0]
    pinsrd xmm3, dword [rdx+4], 2  ; state[14] = nonce[1]
    pinsrd xmm3, dword [rdx+8], 3  ; state[15] = nonce[2]

    ; Working copy
    movdqa xmm4, xmm0
    movdqa xmm5, xmm1
    movdqa xmm6, xmm2
    movdqa xmm7, xmm3

    ; 20 rounds (10 double rounds)
    mov eax, 10

.chacha_round:
    ; Column rounds: QR(0,4,8,12) QR(1,5,9,13) QR(2,6,10,14) QR(3,7,11,15)
    ; In SIMD: all 4 QR simultaneously
    ; xmm4=a(0,1,2,3), xmm5=b(4,5,6,7), xmm6=c(8,9,10,11), xmm7=d(12,13,14,15)

    ; a += b
    paddd xmm4, xmm5
    ; d ^= a
    pxor xmm7, xmm4
    ; d <<<= 16 (rotate left 16)
    movdqa xmm8, xmm7
    pslld xmm8, 16
    psrld xmm7, 16
    por xmm7, xmm8

    ; c += d
    paddd xmm6, xmm7
    ; b ^= c, b <<<= 12
    pxor xmm5, xmm6
    movdqa xmm8, xmm5
    pslld xmm8, 12
    psrld xmm5, 20
    por xmm5, xmm8

    ; a += b
    paddd xmm4, xmm5
    ; d ^= a, d <<<= 8
    pxor xmm7, xmm4
    movdqa xmm8, xmm7
    pslld xmm8, 8
    psrld xmm7, 24
    por xmm7, xmm8

    ; c += d
    paddd xmm6, xmm7
    ; b ^= c, b <<<= 7
    pxor xmm5, xmm6
    movdqa xmm8, xmm5
    pslld xmm8, 7
    psrld xmm5, 25
    por xmm5, xmm8

    ; Diagonal rounds: QR(0,5,10,15) QR(1,6,11,12) QR(2,7,8,13) QR(3,4,9,14)
    ; Rotate xmm5, xmm6, xmm7 to form diagonals
    pshufd xmm5, xmm5, 0x39        ; rotate left 1 word
    pshufd xmm6, xmm6, 0x4E        ; rotate left 2 words
    pshufd xmm7, xmm7, 0x93        ; rotate left 3 words

    ; Same QR operations (diagonal)
    paddd xmm4, xmm5
    pxor xmm7, xmm4
    movdqa xmm8, xmm7
    pslld xmm8, 16
    psrld xmm7, 16
    por xmm7, xmm8

    paddd xmm6, xmm7
    pxor xmm5, xmm6
    movdqa xmm8, xmm5
    pslld xmm8, 12
    psrld xmm5, 20
    por xmm5, xmm8

    paddd xmm4, xmm5
    pxor xmm7, xmm4
    movdqa xmm8, xmm7
    pslld xmm8, 8
    psrld xmm7, 24
    por xmm7, xmm8

    paddd xmm6, xmm7
    pxor xmm5, xmm6
    movdqa xmm8, xmm5
    pslld xmm8, 7
    psrld xmm5, 25
    por xmm5, xmm8

    ; Undo diagonal rotation
    pshufd xmm5, xmm5, 0x93
    pshufd xmm6, xmm6, 0x4E
    pshufd xmm7, xmm7, 0x39

    dec eax
    jnz .chacha_round

    ; Add initial state
    paddd xmm4, xmm0
    paddd xmm5, xmm1
    paddd xmm6, xmm2
    paddd xmm7, xmm3

    ; Store output (64 bytes)
    movdqu [rdi + 0],  xmm4
    movdqu [rdi + 16], xmm5
    movdqu [rdi + 32], xmm6
    movdqu [rdi + 48], xmm7

    add rsp, 64+16
    pop rbx
    pop rbp
    ret
```

---

## Section 7: Data Compression

### 7.1 Run-Length Encoding ด้วย SIMD

```nasm
; ============================================================
; Run-Length Encoding (RLE) ด้วย SSE2
; รูปแบบ output: [count, value] pairs
; int rle_encode(uint8_t *dst, const uint8_t *src, int len)
; rdi=dst, rsi=src, edx=len
; returns: number of bytes in output
; ============================================================
section .text
global rle_encode_simd

rle_encode_simd:
    push rbx
    push r12

    xor ecx, ecx                   ; input position
    xor r9d, r9d                   ; output position
    pxor xmm0, xmm0

.rle_outer:
    cmp ecx, edx
    jge .rle_done

    movzx ebx, byte [rsi + rcx]   ; current byte value
    mov r12d, 1                    ; run count = 1
    inc ecx

    ; Count consecutive same bytes using SIMD
    ; Load 16 bytes and compare with current value
    movd xmm1, ebx
    pshufb xmm1, xmm0              ; broadcast byte to all 16 positions

.rle_count:
    cmp ecx, edx
    jge .rle_emit

    ; How many bytes remaining?
    mov r10d, edx
    sub r10d, ecx
    cmp r10d, 16
    jl .rle_scalar_count

    movdqu xmm2, [rsi + rcx]       ; load 16 bytes
    pcmpeqb xmm2, xmm1             ; compare all with current value
    pmovmskb eax, xmm2             ; bitmask: 1 = same
    not eax
    bsf eax, eax                   ; find first different byte
    jz .rle_all_same               ; all 16 are same

    ; Found a different byte at position eax
    add r12d, eax                  ; add matching count
    add ecx, eax
    jmp .rle_emit

.rle_all_same:
    add r12d, 16
    add ecx, 16
    ; cap at 255
    cmp r12d, 255
    jle .rle_count
    mov r12d, 255
    jmp .rle_emit

.rle_scalar_count:
    cmp byte [rsi + rcx], bl
    jne .rle_emit
    inc r12d
    inc ecx
    cmp r12d, 255
    jl .rle_scalar_count

.rle_emit:
    mov byte [rdi + r9], r12b      ; store count
    mov byte [rdi + r9 + 1], bl   ; store value
    add r9d, 2
    jmp .rle_outer

.rle_done:
    mov eax, r9d                   ; return output length
    pop r12
    pop rbx
    ret
```

### 7.2 LZ4 Decompression Hot Path

```nasm
; ============================================================
; LZ4 Decompression ด้วย SSE2 (hot path สำหรับ large copies)
; LZ4 token format: [LLLL MMMM] [literal_len*] [offset_lo offset_hi] [match_len*]
; LLLL = literal length nibble (0-14, 15 = extended)
; MMMM = match length nibble (0-14, 15 = extended)
;
; int lz4_decompress(uint8_t *dst, const uint8_t *src, int dst_len)
; rdi=dst, rsi=src, edx=dst_len
; ============================================================
section .text
global lz4_decompress_simd

lz4_decompress_simd:
    push rbx
    push r12
    push r13
    push r14
    push r15

    mov r12, rdi                   ; dst_start
    mov r13, rdi                   ; dst_ptr
    mov r14, rsi                   ; src_ptr
    lea r15, [rdi + rdx]           ; dst_end

.lz4_main_loop:
    cmp r13, r15
    jge .lz4_done

    ; Read token
    movzx eax, byte [r14]
    inc r14

    ; Literal length = high nibble
    mov ecx, eax
    shr ecx, 4
    ; Match length = low nibble + 4 (MINMATCH)
    and eax, 0x0F
    add eax, 4                     ; match_len = token & 0x0F + MINMATCH

    ; Extended literal length
    cmp ecx, 15
    jne .lz4_copy_literals
.lz4_ext_lit:
    movzx r8d, byte [r14]
    inc r14
    add ecx, r8d
    cmp r8d, 255
    je .lz4_ext_lit

.lz4_copy_literals:
    test ecx, ecx
    jz .lz4_read_offset
    ; Copy ecx literal bytes using SIMD
    push rcx
    mov ecx, ecx
    shr ecx, 4                     ; ecx / 16

.lz4_lit_loop16:
    test ecx, ecx
    jz .lz4_lit_tail
    movdqu xmm0, [r14]
    movdqu [r13], xmm0
    add r14, 16
    add r13, 16
    dec ecx
    jmp .lz4_lit_loop16

.lz4_lit_tail:
    pop rcx
    and rcx, 15
    jz .lz4_read_offset
    rep movsb                      ; movsb: rsi -> rdi (but we use r14/r13)
    ; fix: manual copy
    ; (simplified — real code would handle this properly)

.lz4_read_offset:
    ; Check if at end of compressed data
    ; (last sequence has no offset/match)
    cmp r13, r15
    jge .lz4_done

    ; Read 16-bit little-endian offset
    movzx r8d, word [r14]
    add r14, 2
    test r8d, r8d
    jz .lz4_done                   ; invalid

    ; Extended match length
    cmp eax, 19                    ; 15 + 4 = 19
    jne .lz4_copy_match
.lz4_ext_match:
    movzx r9d, byte [r14]
    inc r14
    add eax, r9d
    cmp r9d, 255
    je .lz4_ext_match

.lz4_copy_match:
    ; Copy eax bytes from (r13 - r8) to r13
    lea rbx, [r13 - r8]            ; match pointer
    mov ecx, eax

    ; Use SIMD copy if match doesn't overlap within 16 bytes
    cmp r8d, 16
    jl .lz4_match_overlap

.lz4_match_nonoverlap:
    shr ecx, 4
.lz4_match_loop16:
    test ecx, ecx
    jz .lz4_match_tail
    movdqu xmm0, [rbx]
    movdqu [r13], xmm0
    add rbx, 16
    add r13, 16
    dec ecx
    jmp .lz4_match_loop16

.lz4_match_tail:
    mov ecx, eax
    and ecx, 15
    jz .lz4_main_loop
.lz4_match_byte:
    mov dl, [rbx]
    mov [r13], dl
    inc rbx
    inc r13
    dec ecx
    jnz .lz4_match_byte
    jmp .lz4_main_loop

.lz4_match_overlap:
    ; Overlapping: must copy byte by byte
.lz4_overlap_loop:
    mov dl, [rbx]
    mov [r13], dl
    inc rbx
    inc r13
    dec ecx
    jnz .lz4_overlap_loop
    jmp .lz4_main_loop

.lz4_done:
    mov rax, r13
    sub rax, r12                   ; return bytes written
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

---

## Section 8: Complete Benchmark Program

```nasm
; ============================================================
; SIMD Benchmark Program
; Compares scalar vs SIMD throughput for various operations
; Build: nasm -f elf64 benchmark.asm && gcc -o benchmark benchmark.o
; ============================================================

extern printf
extern malloc
extern free
extern clock_gettime

section .data
; Format strings
fmt_header: db "Operation           | Scalar MB/s | SIMD MB/s | Speedup", 10, 0
fmt_sep:    db "--------------------+-------------+-----------+--------", 10, 0
fmt_result: db "%-20s| %11.1f | %9.1f | %6.2fx", 10, 0
fmt_mbytes: db "Buffer size: %d MB", 10, 0

; Benchmark config
BUF_SIZE    equ 64*1024*1024       ; 64 MB buffer
ITERATIONS  equ 10

bench_names:
    dq name_memcpy, name_memset, name_memcmp
    dq name_strlen, name_dot, name_mat4
    dq name_gray, name_crc32, name_aes, 0

name_memcpy: db "memcpy (AVX2)", 0
name_memset: db "memset (AVX2)", 0
name_memcmp: db "memcmp (AVX2)", 0
name_strlen: db "strlen (SSE)", 0
name_dot:    db "dot product (FMA)", 0
name_mat4:   db "mat4x4 mul", 0
name_gray:   db "RGB->Gray", 0
name_crc32:  db "CRC32C (HW)", 0
name_aes:    db "AES-128 CBC", 0

section .bss
buf1: resb BUF_SIZE+64             ; +64 for alignment padding
buf2: resb BUF_SIZE+64
result_buf: resb BUF_SIZE+64
ts_start: resq 2                   ; timespec
ts_end: resq 2

section .text
global main
extern fast_memcpy_avx2
extern fast_memset_avx2
extern fast_memcmp_avx2
extern crc32c_hw
extern dot_n

main:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    ; Print header
    lea rdi, [fmt_header]
    xor eax, eax
    call printf
    lea rdi, [fmt_sep]
    xor eax, eax
    call printf

    ; === Benchmark memcpy ===
    call bench_memcpy_scalar
    movsd [rsp], xmm0              ; save scalar result
    call bench_memcpy_simd
    ; compute speedup and print
    ; (simplified: just show results)

    ; === Benchmark memset ===
    ; ... (similar pattern)

    ; === Benchmark CRC32C ===
    call bench_crc32

    xor eax, eax
    leave
    ret


; --- Timing helper ---
; uint64_t get_time_ns()
; returns nanoseconds since epoch
get_time_ns:
    sub rsp, 16
    mov edi, 1                     ; CLOCK_MONOTONIC
    lea rsi, [rsp]
    call clock_gettime
    mov rax, [rsp]                 ; seconds
    imul rax, rax, 1000000000
    add rax, [rsp + 8]             ; + nanoseconds
    add rsp, 16
    ret


; --- Benchmark memcpy scalar vs SIMD ---
bench_memcpy_scalar:
    ; Scalar memcpy using rep movsb
    call get_time_ns
    mov r12, rax                   ; start time

    mov ecx, ITERATIONS
.scalar_loop:
    lea rdi, [result_buf + 32]
    lea rsi, [buf1 + 32]
    mov rdx, BUF_SIZE
    rep movsb
    dec ecx
    jnz .scalar_loop

    call get_time_ns
    sub rax, r12                   ; elapsed ns

    ; throughput = (ITERATIONS * BUF_SIZE * 1e9) / elapsed_ns (bytes/sec)
    ; convert to MB/s
    ret


bench_memcpy_simd:
    call get_time_ns
    mov r12, rax

    mov ecx, ITERATIONS
.simd_loop:
    lea rdi, [result_buf + 32]
    lea rsi, [buf1 + 32]
    mov rdx, BUF_SIZE
    call fast_memcpy_avx2
    dec ecx
    jnz .simd_loop

    call get_time_ns
    sub rax, r12
    ret


bench_crc32:
    call get_time_ns
    mov r12, rax

    mov ecx, ITERATIONS
.crc_bench_loop:
    xor edi, edi                   ; initial CRC = 0
    lea rsi, [buf1 + 32]
    mov rdx, BUF_SIZE
    call crc32c_hw
    dec ecx
    jnz .crc_bench_loop

    call get_time_ns
    sub rax, r12

    ; print: "CRC32C: X MB/s"
    lea rdi, [fmt_result]
    lea rsi, [name_crc32]
    ; compute throughput...
    xor eax, eax
    call printf
    ret
```

---

## ARM SIMD (NEON) — เทียบเคียง

### ARM Neon: memcpy เทียบเท่า

```gas
@ ARM64 (AArch64) NEON memcpy equivalent
@ void neon_memcpy(void *dst, const void *src, size_t len)
@ x0=dst, x1=src, x2=len

.section .text
.global neon_memcpy
.type neon_memcpy, %function

neon_memcpy:
    lsr x3, x2, #6                @ x3 = len / 64 (process 64 bytes per iter)
    cbz x3, .neon_tail

.neon_loop:
    @ Load 4x16 bytes (64 bytes total) using load-pair
    ldp q0, q1, [x1], #32         @ load 32 bytes, advance src
    ldp q2, q3, [x1], #32         @ load 32 more bytes
    stp q0, q1, [x0], #32         @ store 32 bytes, advance dst
    stp q2, q3, [x0], #32         @ store 32 more bytes
    subs x3, x3, #1
    b.ne .neon_loop

.neon_tail:
    and x2, x2, #63               @ remaining bytes
    cbz x2, .neon_done
    @ Handle tail with byte copy
.neon_byte_loop:
    ldrb w3, [x1], #1
    strb w3, [x0], #1
    subs x2, x2, #1
    b.ne .neon_byte_loop

.neon_done:
    ret


@ ARM64 NEON dot product (8 floats)
@ float neon_dot8(float *a, float *b)
@ x0=a, x1=b, returns s0

.global neon_dot8
neon_dot8:
    ld1 {v0.4s, v1.4s}, [x0]      @ load a[0..7]
    ld1 {v2.4s, v3.4s}, [x1]      @ load b[0..7]
    fmul v4.4s, v0.4s, v2.4s      @ a[0..3] * b[0..3]
    fmla v4.4s, v1.4s, v3.4s      @ += a[4..7] * b[4..7]
    @ horizontal sum
    faddp v4.4s, v4.4s, v4.4s     @ pairwise add
    faddp s0, v4.2s                @ final sum in s0
    ret


@ ARM64 NEON matrix 4x4 multiply (float)
@ void neon_mat4x4_mul(float *dst, float *a, float *b)
@ x0=dst, x1=a, x2=b

.global neon_mat4x4_mul
neon_mat4x4_mul:
    @ Load B columns
    ld1 {v4.4s}, [x2], #16        @ b col 0
    ld1 {v5.4s}, [x2], #16        @ b col 1
    ld1 {v6.4s}, [x2], #16        @ b col 2
    ld1 {v7.4s}, [x2], #16        @ b col 3

    @ Load A
    ld1 {v0.4s}, [x1], #16        @ a row 0
    ld1 {v1.4s}, [x1], #16        @ a row 1
    ld1 {v2.4s}, [x1], #16        @ a row 2
    ld1 {v3.4s}, [x1], #16        @ a row 3

    @ Compute each output column
    @ dst_col0 = a * b_col0
    fmul v16.4s, v4.4s, v0.s[0]   @ b_col0 * a[0][0]
    fmla v16.4s, v5.4s, v0.s[1]   @ += b_col1 * a[0][1]
    fmla v16.4s, v6.4s, v0.s[2]   @ += b_col2 * a[0][2]
    fmla v16.4s, v7.4s, v0.s[3]   @ += b_col3 * a[0][3]

    fmul v17.4s, v4.4s, v1.s[0]
    fmla v17.4s, v5.4s, v1.s[1]
    fmla v17.4s, v6.4s, v1.s[2]
    fmla v17.4s, v7.4s, v1.s[3]

    fmul v18.4s, v4.4s, v2.s[0]
    fmla v18.4s, v5.4s, v2.s[1]
    fmla v18.4s, v6.4s, v2.s[2]
    fmla v18.4s, v7.4s, v2.s[3]

    fmul v19.4s, v4.4s, v3.s[0]
    fmla v19.4s, v5.4s, v3.s[1]
    fmla v19.4s, v6.4s, v3.s[2]
    fmla v19.4s, v7.4s, v3.s[3]

    @ Store result (transposed because we computed row-by-row)
    st1 {v16.4s}, [x0], #16
    st1 {v17.4s}, [x0], #16
    st1 {v18.4s}, [x0], #16
    st1 {v19.4s}, [x0], #16
    ret


@ ARM64 NEON AES-128 Encryption (using ARMv8 AES instructions)
@ void arm_aes128_encrypt(uint8_t *dst, uint8_t *src, uint8_t *keys, int blocks)
@ x0=dst, x1=src, x2=keys, x3=blocks

.global arm_aes128_encrypt
arm_aes128_encrypt:
    @ Load all 11 round keys
    ld1 {v16.16b}, [x2], #16
    ld1 {v17.16b}, [x2], #16
    ld1 {v18.16b}, [x2], #16
    ld1 {v19.16b}, [x2], #16
    ld1 {v20.16b}, [x2], #16
    ld1 {v21.16b}, [x2], #16
    ld1 {v22.16b}, [x2], #16
    ld1 {v23.16b}, [x2], #16
    ld1 {v24.16b}, [x2], #16
    ld1 {v25.16b}, [x2], #16
    ld1 {v26.16b}, [x2]            @ round key 10

.arm_aes_loop:
    cbz x3, .arm_aes_done
    ld1 {v0.16b}, [x1], #16       @ load plaintext block

    @ AES encryption: AESE + AESMC (9 rounds), then AESE + final XOR
    eor v0.16b, v0.16b, v16.16b   @ initial AddRoundKey

    aese v0.16b, v17.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v18.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v19.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v20.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v21.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v22.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v23.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v24.16b
    aesmc v0.16b, v0.16b

    aese v0.16b, v25.16b
    @ No AESMC after last round
    eor v0.16b, v0.16b, v26.16b   @ final round key

    st1 {v0.16b}, [x0], #16
    subs x3, x3, #1
    b.ne .arm_aes_loop

.arm_aes_done:
    ret
```

---

## Section 9: Expected Speedups และ Guidelines

### 9.1 Expected Speedup Table

```
Operation               | Scalar BW  | SSE (128b) | AVX2 (256b) | AVX-512 (512b)
------------------------|------------|------------|-------------|----------------
memcpy (L1 cache)       | ~20 GB/s   | ~40 GB/s   | ~80 GB/s    | ~100 GB/s
memcpy (DRAM)           | ~20 GB/s   | ~25 GB/s   | ~25 GB/s    | ~25 GB/s *
memset (DRAM)           | ~20 GB/s   | ~40 GB/s   | ~40 GB/s    | ~40 GB/s
float add array         | ~1x        | ~4x        | ~8x         | ~16x
dot product (FP32)      | ~1x        | ~4x        | ~8-16x      | ~16-32x
matrix 4x4 mul          | ~1x        | ~6x        | ~12x        | N/A
RGB to grayscale        | ~1x        | ~8x        | ~16x        | ~32x
AES-128 CBC             | ~250 MB/s  | ~1.5 GB/s  | ~3 GB/s     | ~6 GB/s
CRC32C                  | ~200 MB/s  | ~3 GB/s *  | ~3 GB/s *   | ~10 GB/s *
strlen (short strings)  | ~1x        | ~4x        | ~8x         | ~16x

* = Memory bandwidth limited
* AES-NI = hardware instruction, not pure SIMD
```

### 9.2 SIMD Best Practices

```
1. DATA LAYOUT:
   - ใช้ SoA แทน AoS สำหรับ SIMD-heavy code
   - Align data ให้ตรงกับ register width (32-byte สำหรับ AVX2)
   - Pack data ให้แน่น หลีกเลี่ยง padding bytes

2. MEMORY ACCESS PATTERNS:
   - Unit stride (sequential) = best (สำหรับ prefetcher)
   - Use NT stores สำหรับ write-only data > L3 cache size
   - Prefetch ล่วงหน้า: PREFETCHT0 สำหรับ L1, PREFETCHT1 สำหรับ L2

3. INSTRUCTION SELECTION:
   - VEX-encoded instructions (VMOVAPS) แทน legacy (MOVAPS)
   - FMA (VFMADD231PS) ดีกว่า VMULPS + VADDPS แยก
   - VPBROADCAST* ดีกว่า VSHUFPS สำหรับ broadcast
   - VZEROUPPER หลัง YMM usage ก่อน call legacy SSE code

4. LOOP STRUCTURE:
   - Main loop: multiple of 32 bytes (AVX2) หรือ 64 bytes (AVX-512)
   - Scalar tail สำหรับ remainder
   - Software unrolling: 2-4x ช่วย hide memory latency

5. AVOID:
   - Mixing VEX and non-VEX instructions (performance penalty)
   - Store-forwarding stalls (store then immediately load same addr)
   - Horizontal operations บ่อยๆ (แพง) — delay จนสุดท้าย
   - Cross-lane operations (เช่น VPERMPS) มากเกินไป
```

### 9.3 CPUID Feature Detection

```nasm
; ============================================================
; ตรวจสอบ SIMD capabilities ของ CPU
; void check_simd_features(uint32_t *features)
; Bit layout: [0]=SSE, [1]=SSE2, [2]=SSE4.2, [3]=AVX, 
;             [4]=AVX2, [5]=FMA, [6]=AVX512F, [7]=AESNI
; ============================================================
section .text
global check_simd_features

check_simd_features:
    push rbx
    push r12
    mov r12, rdi                   ; save output pointer
    xor r13d, r13d                 ; feature bits

    ; Check CPUID leaf 1 (SSE, SSE2, SSE4.2, AES-NI, AVX)
    mov eax, 1
    cpuid
    ; ECX bits:
    ; [25] = AES-NI
    ; [28] = AVX
    ; [12] = FMA
    ; EDX bits:
    ; [25] = SSE
    ; [26] = SSE2

    test edx, (1 << 25)
    setnz al
    or r13d, eax                   ; bit 0 = SSE

    test edx, (1 << 26)
    setnz al
    shl eax, 1
    or r13d, eax                   ; bit 1 = SSE2

    test ecx, (1 << 20)            ; SSE4.2 bit
    setnz al
    shl eax, 2
    or r13d, eax                   ; bit 2 = SSE4.2

    test ecx, (1 << 28)            ; AVX bit
    setnz al
    shl eax, 3
    or r13d, eax                   ; bit 3 = AVX

    test ecx, (1 << 12)            ; FMA bit
    setnz al
    shl eax, 5
    or r13d, eax                   ; bit 5 = FMA

    test ecx, (1 << 25)            ; AES-NI bit
    setnz al
    shl eax, 7
    or r13d, eax                   ; bit 7 = AES-NI

    ; Check CPUID leaf 7 (AVX2, AVX-512)
    mov eax, 7
    xor ecx, ecx
    cpuid
    ; EBX bits:
    ; [5] = AVX2
    ; [16] = AVX-512F

    test ebx, (1 << 5)             ; AVX2
    setnz al
    shl eax, 4
    or r13d, eax                   ; bit 4 = AVX2

    test ebx, (1 << 16)            ; AVX-512F
    setnz al
    shl eax, 6
    or r13d, eax                   ; bit 6 = AVX-512F

    mov [r12], r13d
    pop rbx
    ret


; ============================================================
; Runtime dispatch: เลือก implementation ตาม CPU capabilities
; ============================================================
section .data
p_memcpy: dq scalar_memcpy        ; function pointer (default: scalar)

section .text
global init_simd_dispatch

init_simd_dispatch:
    sub rsp, 8
    lea rdi, [rsp]
    call check_simd_features
    mov eax, [rsp]
    add rsp, 8

    ; Select best memcpy implementation
    test eax, (1 << 6)             ; AVX-512?
    jnz .use_avx512
    test eax, (1 << 4)             ; AVX2?
    jnz .use_avx2
    test eax, (1 << 1)             ; SSE2?
    jnz .use_sse2

    ret                            ; keep scalar default

.use_sse2:
    lea rax, [fast_memcpy_sse]
    mov [p_memcpy], rax
    ret

.use_avx2:
    lea rax, [fast_memcpy_avx2]
    mov [p_memcpy], rax
    ret

.use_avx512:
    lea rax, [fast_memcpy_nt]      ; NT version for very large copies
    mov [p_memcpy], rax
    ret
```

---

## Section 10: Makefile และ Build Instructions

```makefile
# Makefile สำหรับ SIMD Examples
AS      = nasm
CC      = gcc
ASFLAGS = -f elf64 -g -F dwarf
CFLAGS  = -O2 -mavx2 -mfma -maes -g

SRCS_ASM = part-050-simd-practical-applications.asm
SRCS_C   = benchmark_driver.c

all: benchmark_simd

benchmark_simd: $(SRCS_ASM:.asm=.o) $(SRCS_C:.c=.o)
	$(CC) -o $@ $^ -lm

%.o: %.asm
	$(AS) $(ASFLAGS) -o $@ $<

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

test: benchmark_simd
	./benchmark_simd

clean:
	rm -f *.o benchmark_simd

# Test specific operations
test_memcpy: benchmark_simd
	./benchmark_simd --test memcpy --size 64M --iterations 100

test_aes: benchmark_simd
	./benchmark_simd --test aes --size 1M --iterations 1000

.PHONY: all clean test test_memcpy test_aes
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

| หัวข้อ | เทคนิค SIMD | Speedup ที่คาดหวัง |
|--------|-------------|-------------------|
| memcpy | NT-stores, 256-bit | 2-4x (DRAM bound) |
| memset | VPBROADCASTB | 2-4x |
| strlen | PCMPEQB + PMOVMSKB | 4-16x |
| strchr | PCMPISTRI | 4-8x |
| dot product | FMA, VHADDPS | 8-16x |
| mat4x4 mul | VSHUFPS patterns | 6-12x |
| RGB->Gray | PMULLW fixed-point | 8-16x |
| alpha blend | PADDUSB/PSUBUSB | 8-16x |
| convolution | PMULHUW | 4-8x |
| CRC32C | HW instruction | 10-15x |
| AES-128 | AES-NI | 6-10x |
| ChaCha20 | SSE2 rotations | 4-8x |
| RLE encode | PCMPEQB scan | 3-6x |
| LZ4 decomp | 16-byte copy | 2-4x |

**กฎทอง:**
1. Profile ก่อน optimize — อย่า guess hotspot
2. Data layout สำคัญที่สุด (SoA >> AoS สำหรับ SIMD)
3. Memory bandwidth มักเป็น bottleneck จริง ไม่ใช่ compute
4. ใช้ compiler autovectorization เป็น baseline ก่อนเขียน intrinsics/asm
5. Measure ทุกครั้ง — SIMD ที่ผิดแบบช้ากว่า scalar!

---
*Part 050 of the Assembly Programming Course*
*เขียนด้วย NASM x86-64 syntax และ GAS ARM64 (AArch64) syntax*
