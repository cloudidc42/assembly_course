# Part 066: SIMD Optimization Patterns

## บทนำ (Introduction)

SIMD (Single Instruction, Multiple Data) เป็นเทคนิคที่ทรงพลังสำหรับการเร่งความเร็วของโปรแกรม โดยการทำงานกับข้อมูลหลายชิ้นพร้อมกันในคำสั่งเดียว ใน Part นี้เราจะเรียนรู้ patterns ต่างๆ สำหรับการใช้ SIMD อย่างมีประสิทธิภาพ

### เป้าหมายการเรียนรู้
- เข้าใจ data layout patterns (AoS vs SoA)
- รู้จัก horizontal vs vertical operations
- เรียนรู้ scatter/gather patterns
- ใช้ masked operations สำหรับ conditional SIMD
- จัดการ alignment issues
- Vectorize conditional code
- ใช้ lookup tables, sorting networks, histograms
- คำนวณ prefix sum ด้วย SIMD
- SIMD string operations
- Bit manipulation patterns
- Reduction operations
- Mixed-precision computations

---

## 1. Data Layout Patterns: AoS vs SoA

### 1.1 Array of Structures (AoS)

AoS คือ layout แบบดั้งเดิม ที่ข้อมูลของ object หนึ่งๆ อยู่ติดกัน

```c
// AoS - Array of Structures
struct Particle {
    float x, y, z;    // position
    float vx, vy, vz; // velocity
    float mass;
    float charge;
};

Particle particles[1000];
```

**Memory layout ของ AoS:**
```
[x0,y0,z0,vx0,vy0,vz0,m0,c0] [x1,y1,z1,vx1,vy1,vz1,m1,c1] ...
```

ปัญหา: ถ้าต้องการแค่ x coordinates, ต้อง stride ข้าม fields อื่น

### 1.2 Structure of Arrays (SoA)

SoA จัดเก็บ field เดียวกันของทุก objects ติดกัน

```c
// SoA - Structure of Arrays
struct Particles {
    float* x;      // [x0, x1, x2, ...]
    float* y;      // [y0, y1, y2, ...]
    float* z;      // [z0, z1, z2, ...]
    float* vx;     // [vx0, vx1, vx2, ...]
    float* vy;     // [vy0, vy1, vy2, ...]
    float* vz;     // [vz0, vz1, vz2, ...]
    float* mass;   // [m0, m1, m2, ...]
    float* charge; // [c0, c1, c2, ...]
    int count;
};
```

**Memory layout ของ SoA:**
```
x:  [x0, x1, x2, x3, x4, x5, x6, x7, ...]
y:  [y0, y1, y2, y3, y4, y5, y6, y7, ...]
z:  [z0, z1, z2, z3, z4, z5, z6, z7, ...]
vx: [vx0, vx1, vx2, vx3, vx4, vx5, vx6, vx7, ...]
```

### 1.3 การวิเคราะห์ AoS vs SoA

| Feature | AoS | SoA |
|---------|-----|-----|
| Cache locality (single element) | ดี | แย่ |
| Cache locality (single field) | แย่ | ดี |
| SIMD vectorization | ยาก | ง่าย |
| Code readability | ดีกว่า | ซับซ้อนกว่า |
| Random access | ดี | แย่ (multiple pointers) |
| Sequential processing | แย่ | ดีมาก |

### 1.4 SoA Transformation ใน Code

```nasm
; ตัวอย่าง: อัพเดท positions จาก velocities
; AoS version (ช้ากว่า)
; struct Particle { float x,y,z,vx,vy,vz; }
; particle size = 24 bytes

update_positions_aos:
    ; rdi = particles array
    ; rsi = count
    ; xmm0 = dt (delta time)
    
    xor     ecx, ecx
.loop:
    cmp     ecx, esi
    jge     .done
    
    ; Load x, vx for one particle
    movss   xmm1, [rdi + rcx*24]       ; x
    movss   xmm2, [rdi + rcx*24 + 12]  ; vx
    mulss   xmm2, xmm0                  ; vx * dt
    addss   xmm1, xmm2                  ; x + vx*dt
    movss   [rdi + rcx*24], xmm1        ; store x
    
    ; Load y, vy
    movss   xmm1, [rdi + rcx*24 + 4]   ; y
    movss   xmm2, [rdi + rcx*24 + 16]  ; vy
    mulss   xmm2, xmm0
    addss   xmm1, xmm2
    movss   [rdi + rcx*24 + 4], xmm1
    
    ; Load z, vz
    movss   xmm1, [rdi + rcx*24 + 8]   ; z
    movss   xmm2, [rdi + rcx*24 + 20]  ; vz
    mulss   xmm2, xmm0
    addss   xmm1, xmm2
    movss   [rdi + rcx*24 + 8], xmm1
    
    inc     ecx
    jmp     .loop
.done:
    ret

; SoA version (เร็วกว่า - vectorized)
update_positions_soa:
    ; rdi = x array
    ; rsi = y array  
    ; rdx = z array
    ; rcx = vx array
    ; r8  = vy array
    ; r9  = vz array
    ; xmm0 = dt
    ; [rsp+8] = count
    
    mov     eax, [rsp+8]    ; count
    
    vbroadcastss ymm0, xmm0  ; dt broadcast to all lanes
    
    xor     r10d, r10d
.loop:
    sub     eax, 8
    jl      .remainder
    
    ; Process 8 particles at once (AVX)
    vmovups ymm1, [rdi + r10*4]   ; x[0..7]
    vmovups ymm2, [rcx + r10*4]   ; vx[0..7]
    vfmadd231ps ymm1, ymm2, ymm0  ; x += vx * dt
    vmovups [rdi + r10*4], ymm1
    
    vmovups ymm1, [rsi + r10*4]   ; y[0..7]
    vmovups ymm2, [r8 + r10*4]    ; vy[0..7]
    vfmadd231ps ymm1, ymm2, ymm0  ; y += vy * dt
    vmovups [rsi + r10*4], ymm1
    
    vmovups ymm1, [rdx + r10*4]   ; z[0..7]
    vmovups ymm2, [r9 + r10*4]    ; vz[0..7]
    vfmadd231ps ymm1, ymm2, ymm0  ; z += vz * dt
    vmovups [rdx + r10*4], ymm1
    
    add     r10d, 8
    jmp     .loop
    
.remainder:
    add     eax, 8           ; restore count
    ; handle remaining elements scalar
.done:
    vzeroupper
    ret
```

### 1.5 AoSoA (Array of Structures of Arrays)

Hybrid layout สำหรับ balance ระหว่าง AoS และ SoA

```c
// AoSoA - 8-wide structure
struct Particles8 {
    float x[8];
    float y[8];
    float z[8];
    float vx[8];
    float vy[8];
    float vz[8];
};

Particles8 particles[N/8];
```

Layout ดีสำหรับ AVX2 (8 floats = 256-bit register)

---

## 2. Vectorization-Friendly Layouts

### 2.1 Alignment Requirements

```nasm
section .data
    align 32          ; 32-byte alignment สำหรับ AVX
array_avx:  resd 8    ; 8 x 32-bit = 256-bit

    align 16          ; 16-byte alignment สำหรับ SSE
array_sse:  resd 4    ; 4 x 32-bit = 128-bit
```

### 2.2 Padding สำหรับ Vectorization

```c
// Bad - size 12 bytes, ไม่ align ดี
struct Vec3 {
    float x, y, z;
};

// Good - size 16 bytes, align สำหรับ SSE
struct Vec3Padded {
    float x, y, z;
    float pad;  // padding
};

// หรือใช้ __attribute__((aligned(16)))
struct __attribute__((aligned(16))) Vec3Aligned {
    float x, y, z, w;
};
```

### 2.3 ตัวอย่าง Vectorization-Friendly Array Processing

```nasm
; คำนวณ dot products ของหลาย vec4 พร้อมกัน
; SoA layout: x[], y[], z[], w[] arrays
dot_products_soa:
    ; rdi = ax array (x components of A vectors)
    ; rsi = ay array
    ; rdx = az array
    ; rcx = aw array
    ; r8  = bx array (x components of B vectors)
    ; r9  = by array
    ; [rsp+8] = bz array
    ; [rsp+16] = bw array
    ; [rsp+24] = result array
    ; [rsp+32] = count
    
    mov     r10, [rsp+8]     ; bz
    mov     r11, [rsp+16]    ; bw
    mov     r12, [rsp+24]    ; result
    mov     eax, [rsp+32]    ; count
    
    xor     r13d, r13d       ; index
.loop:
    sub     eax, 8
    jl      .done
    
    vmovups ymm0, [rdi + r13*4]  ; ax[0..7]
    vmovups ymm1, [r8  + r13*4]  ; bx[0..7]
    vmulps  ymm4, ymm0, ymm1     ; ax * bx
    
    vmovups ymm0, [rsi + r13*4]  ; ay[0..7]
    vmovups ymm1, [r9  + r13*4]  ; by[0..7]
    vfmadd231ps ymm4, ymm0, ymm1 ; += ay * by
    
    vmovups ymm0, [rdx + r13*4]  ; az[0..7]
    vmovups ymm1, [r10 + r13*4]  ; bz[0..7]
    vfmadd231ps ymm4, ymm0, ymm1 ; += az * bz
    
    vmovups ymm0, [rcx + r13*4]  ; aw[0..7]
    vmovups ymm1, [r11 + r13*4]  ; bw[0..7]
    vfmadd231ps ymm4, ymm0, ymm1 ; += aw * bw
    
    vmovups [r12 + r13*4], ymm4  ; store results
    
    add     r13d, 8
    jmp     .loop
.done:
    vzeroupper
    ret
```

---

## 3. Horizontal Operations: HADD และ Pairwise Reduction

### 3.1 HADD (Horizontal Add)

HADD บวกคู่ elements ใน register เดียวกัน

```nasm
; SSE3: HADDPS - Horizontal Add Packed Singles
; xmm0 = [a, b, c, d]
; xmm1 = [e, f, g, h]
; หลัง haddps xmm0, xmm1:
; xmm0 = [a+b, c+d, e+f, g+h]

horizontal_sum_sse3:
    ; xmm0 = [a, b, c, d]
    haddps  xmm0, xmm0
    ; xmm0 = [a+b, c+d, a+b, c+d]
    haddps  xmm0, xmm0
    ; xmm0[0] = a+b+c+d
    ret

; AVX: VHADDPS
horizontal_sum_avx:
    ; ymm0 = [a, b, c, d, e, f, g, h]
    vhaddps ymm0, ymm0, ymm0
    ; ymm0 = [a+b, c+d, a+b, c+d, e+f, g+h, e+f, g+h]
    vhaddps ymm0, ymm0, ymm0
    ; ymm0[0] = a+b+c+d, ymm0[4] = e+f+g+h
    ; ยังต้อง sum สอง halves
    vextractf128 xmm1, ymm0, 1
    addss   xmm0, xmm1
    ret
```

### 3.2 Pairwise Reduction

```nasm
; Sum reduction ของ array 8 floats
reduce_sum_8:
    ; ymm0 = [a0, a1, a2, a3, a4, a5, a6, a7]
    
    ; Step 1: Sum adjacent pairs
    vpermilps ymm1, ymm0, 0b10110001  ; [a1,a0,a3,a2,a5,a4,a7,a6]
    vaddps    ymm0, ymm0, ymm1
    ; ymm0 = [a0+a1, a0+a1, a2+a3, a2+a3, a4+a5, a4+a5, a6+a7, a6+a7]
    
    ; Step 2: Sum pairs of pairs
    vpermilps ymm1, ymm0, 0b00001010  ; shuffle
    vaddps    ymm0, ymm0, ymm1
    
    ; Step 3: Sum high and low halves
    vextractf128 xmm1, ymm0, 1
    addss    xmm0, xmm1
    ; xmm0[0] = total sum
    ret
```

### 3.3 PHADDW / PHADDD (Integer Horizontal Add)

```nasm
; PHADDW: Horizontal add 16-bit integers
; xmm0 = [a0, a1, a2, a3, a4, a5, a6, a7] (16-bit each)
; xmm1 = [b0, b1, b2, b3, b4, b5, b6, b7]
; หลัง phaddw:
; xmm0 = [a0+a1, a2+a3, a4+a5, a6+a7, b0+b1, b2+b3, b4+b5, b6+b7]

horizontal_sum_i16:
    phaddw  xmm0, xmm0
    phaddw  xmm0, xmm0
    phaddw  xmm0, xmm0
    ; xmm0[0..15] = sum of all 8 shorts
    movsx   eax, word [rel .zero]
    pextrw  eax, xmm0, 0
    ret
```

---

## 4. Vertical Operations (Preferred): Lane-wise

### 4.1 ทำไม Vertical ดีกว่า Horizontal

Vertical operations (lane-wise) มีประสิทธิภาพดีกว่า:
- ทำงานแบบ independent ในแต่ละ lane
- ไม่มี data dependency ระหว่าง lanes
- Throughput สูงกว่า horizontal ops

```nasm
; Vertical (preferred): บวก corresponding elements
; a = [a0, a1, a2, a3]
; b = [b0, b1, b2, b3]
vertical_add:
    vaddps  ymm0, ymm1, ymm2   ; [a0+b0, a1+b1, a2+b2, a3+b3]
    ; Throughput: ~0.5 cycles (on modern CPUs)

; Horizontal (avoid when possible):
horizontal_add_example:
    haddps  xmm0, xmm1         ; [a0+a1, a2+a3, b0+b1, b2+b3]
    ; Throughput: ~3 cycles (more expensive)
```

### 4.2 ตัวอย่าง Vertical Operations

```nasm
; คำนวณ element-wise max ของ array
array_max_vertical:
    ; rdi = array a
    ; rsi = array b
    ; rdx = result
    ; ecx = count
    
    xor     eax, eax
.loop:
    sub     ecx, 8
    jl      .done
    
    vmovups ymm0, [rdi + rax*4]   ; a[0..7]
    vmovups ymm1, [rsi + rax*4]   ; b[0..7]
    vmaxps  ymm2, ymm0, ymm1      ; max element-wise
    vmovups [rdx + rax*4], ymm2
    
    add     eax, 8
    jmp     .loop
.done:
    vzeroupper
    ret

; คำนวณ element-wise clamp
array_clamp:
    ; clamp values to [min, max]
    vbroadcastss ymm_min, [rel min_val]
    vbroadcastss ymm_max, [rel max_val]
.loop:
    vmovups  ymm0, [rdi + rax*4]
    vmaxps   ymm0, ymm0, ymm_min    ; clamp to min
    vminps   ymm0, ymm0, ymm_max    ; clamp to max
    vmovups  [rdx + rax*4], ymm0
    add      eax, 8
    sub      ecx, 8
    jg       .loop
    ret
```

---

## 5. Scatter/Gather Patterns

### 5.1 Gather Operations

Gather โหลดข้อมูลจาก scattered memory locations โดยใช้ index array

```nasm
; AVX2 VGATHERDPS: Gather floats using 32-bit indices
gather_floats:
    ; rdi = base pointer
    ; ymm0 = indices [i0, i1, i2, i3, i4, i5, i6, i7]
    ; ymm1 = all-ones mask (which elements to gather)
    
    ; Gather: result[n] = base[indices[n]]
    vpcmpeqd    ymm1, ymm1, ymm1    ; all-ones mask
    vgatherdps  ymm2, [rdi + ymm0*4], ymm1
    ; ymm2 = [base[i0], base[i1], ..., base[i7]]
    ret

; ตัวอย่างจริง: gather vertices ตาม index buffer
gather_vertices:
    ; rdi = vertex x array
    ; rsi = index array
    ; rdx = output
    ; ecx = count
    
    xor     eax, eax
.loop:
    sub     ecx, 8
    jl      .done
    
    vmovdqu     ymm0, [rsi + rax*4]   ; load 8 indices
    vpcmpeqd    ymm1, ymm1, ymm1      ; mask = all enabled
    vgatherdps  ymm2, [rdi + ymm0*4], ymm1  ; gather x values
    vmovups     [rdx + rax*4], ymm2
    
    add     eax, 8
    jmp     .loop
.done:
    vzeroupper
    ret
```

### 5.2 เมื่อไหร่ควรใช้ Gather

```
ใช้ Gather เมื่อ:
- Access pattern ไม่ sequential แต่รู้ indices ล่วงหน้า
- Cost ของ scalar loads สูงมาก (หลาย cache misses)
- Index computation สามารถ vectorize ได้

ไม่ควรใช้ Gather เมื่อ:
- Access pattern sequential (ใช้ vmovups แทน)
- Gather indices สุ่มมาก (cache miss rate สูงอยู่ดี)
- มีน้อยกว่า ~4 elements (scalar เร็วกว่า)
```

### 5.3 Scatter Operations (AVX-512)

```nasm
; AVX-512: VSCATTERDPS
scatter_floats_avx512:
    ; zmm0 = values to scatter
    ; zmm1 = indices
    ; rdi = base pointer
    ; k1 = mask register
    
    kmovw       k1, 0xFFFF          ; all lanes active
    vscatterdps [rdi + zmm1*4]{k1}, zmm0
    ret
```

### 5.4 Emulating Scatter without AVX-512

```nasm
; Manual scatter สำหรับ AVX2
scatter_manual:
    ; ymm0 = values
    ; ymm1 = indices
    ; rdi = base
    
    ; Extract individual values and indices
    vmovd       eax, xmm0
    vmovd       ecx, xmm1
    mov         [rdi + rcx*4], eax
    
    vpermilps   xmm0, xmm0, 0b00000001
    vpermilps   xmm1, xmm1, 0b00000001
    vmovd       eax, xmm0
    vmovd       ecx, xmm1
    mov         [rdi + rcx*4], eax
    
    ; ... repeat for all 8 elements
    ret
```

---

## 6. Masked Operations: Conditional SIMD

### 6.1 SSE/AVX Masking

```nasm
; Conditional update: if (a[i] > threshold) result[i] = a[i]
masked_threshold:
    ; rdi = input array
    ; rsi = result array
    ; ecx = count
    ; xmm0 = threshold (broadcast ก่อน)
    
    vbroadcastss ymm_thresh, xmm0
    xor          eax, eax
.loop:
    sub     ecx, 8
    jl      .done
    
    vmovups ymm0, [rdi + rax*4]
    vcmpps  ymm1, ymm0, ymm_thresh, 14  ; cmp: a > threshold (GT)
    ; ymm1 = mask (0xFFFFFFFF where true, 0 where false)
    
    vblendvps ymm2, ymm_zero, ymm0, ymm1  ; select based on mask
    vmovups   [rsi + rax*4], ymm2
    
    add     eax, 8
    jmp     .loop
.done:
    vzeroupper
    ret
```

### 6.2 VBLENDVPS / VPBLENDVB

```nasm
; blend based on mask
; dst[i] = mask[i] ? a[i] : b[i]
blend_example:
    ; ymm0 = a values
    ; ymm1 = b values
    ; ymm2 = condition mask (sign bit determines selection)
    
    vblendvps ymm3, ymm1, ymm0, ymm2
    ; ymm3[i] = (ymm2[i] sign bit set) ? ymm0[i] : ymm1[i]
    ret

; Integer blend (byte-wise)
blend_bytes:
    ; xmm0 = a bytes
    ; xmm1 = b bytes
    ; xmm2 = mask (0xFF=take from a, 0x00=take from b)
    
    vpblendvb xmm3, xmm1, xmm0, xmm2
    ret
```

### 6.3 AVX-512 Mask Registers

```nasm
; AVX-512 ใช้ k registers สำหรับ masking
avx512_masked:
    ; k1 = predicate mask
    ; zmm0 = source
    ; zmm1 = destination (preserved where mask=0)
    
    vcmpps      k1, zmm0, zmm_thresh, 14  ; create mask
    vmovaps     zmm1{k1}, zmm0            ; conditional move
    ; หรือ: zmm1{k1}{z} สำหรับ zero masking
    
    ; Masked arithmetic
    vaddps      zmm2{k1}, zmm0, zmm1     ; add only where mask=1
    ret
```

### 6.4 ตัวอย่างจริง: ReLU Activation Function

```nasm
; ReLU: output = max(0, input)
relu_avx:
    ; rdi = input array
    ; rsi = output array
    ; ecx = count
    
    vpxor    ymm_zero, ymm_zero, ymm_zero  ; zero vector
    xor      eax, eax
.loop:
    sub     ecx, 8
    jl      .remainder
    
    vmovups ymm0, [rdi + rax*4]
    vmaxps  ymm0, ymm0, ymm_zero           ; max(x, 0)
    vmovups [rsi + rax*4], ymm0
    
    add     eax, 8
    jmp     .loop
.remainder:
    add     ecx, 8
    jz      .done
    ; handle remaining...
.done:
    vzeroupper
    ret
```

---

## 7. Alignment: ตรวจสอบและจัดการ Unaligned Head/Tail

### 7.1 ตรวจสอบ Alignment

```nasm
; ตรวจสอบว่า pointer align หรือไม่
check_alignment_32:
    ; rdi = pointer to check
    test    rdi, 31          ; check bits [4:0]
    jz      .aligned
    ; not aligned
    xor     eax, eax
    ret
.aligned:
    mov     eax, 1
    ret

; คำนวณ bytes ที่ต้อง process ก่อน alignment
calc_head_size:
    ; rdi = pointer
    ; returns: bytes to process before aligned part
    mov     eax, edi
    and     eax, 31          ; offset from 32-byte boundary
    jz      .already_aligned
    neg     eax
    add     eax, 32          ; bytes to next boundary
.already_aligned:
    ret
```

### 7.2 Handling Unaligned Head/Tail

```nasm
; Template สำหรับ aligned processing
process_with_alignment:
    ; rdi = array
    ; rsi = count
    
    ; คำนวณ head size
    mov     ecx, edi
    and     ecx, 31
    jz      .main_loop
    neg     ecx
    add     ecx, 32         ; bytes in head
    shr     ecx, 2          ; convert to floats
    
.head_loop:
    ; Process one element at a time
    movss   xmm0, [rdi]
    ; ... process
    add     rdi, 4
    dec     ecx
    jnz     .head_loop
    
.main_loop:
    ; Now rdi is 32-byte aligned
    mov     ecx, esi         ; remaining count
    and     ecx, ~7          ; multiples of 8
.main:
    sub     ecx, 8
    jl      .tail
    vmovaps ymm0, [rdi]      ; aligned load
    ; ... vectorized processing
    add     rdi, 32
    jmp     .main
    
.tail:
    add     ecx, 8           ; undo last subtraction
.tail_loop:
    test    ecx, ecx
    jz      .done
    movss   xmm0, [rdi]
    ; ... process
    add     rdi, 4
    dec     ecx
    jmp     .tail_loop
.done:
    vzeroupper
    ret
```

### 7.3 ใช้ Masked Loads (AVX-512 / AVX2)

```nasm
; AVX2: ใช้ VMASKMOVPS สำหรับ partial loads
partial_load_avx2:
    ; rdi = array
    ; ecx = remaining elements (1-7)
    
    ; สร้าง mask สำหรับ ecx elements
    ; mask[i] = (i < ecx) ? 0xFFFFFFFF : 0
    
    mov     eax, ecx
    ; ... สร้าง mask ใน ymm7
    
    vmaskmovps ymm0, ymm7, [rdi]   ; load only masked elements
    ; process ymm0
    vmaskmovps [rsi], ymm7, ymm0   ; store only masked elements
    ret
```

---

## 8. Vectorization ของ Conditional Code: Compute Both, Blend

### 8.1 Pattern: Compute Both Branches, Then Blend

```nasm
; if (condition) x = f(a) else x = g(a)
; SIMD version: compute f(a) and g(a) for all elements
; then blend based on condition mask

conditional_compute:
    ; ymm0 = input values
    ; ymm1 = condition mask
    
    ; Compute both branches
    ; Branch A: square root
    vsqrtps ymm2, ymm0      ; f(a) = sqrt(a)
    
    ; Branch B: reciprocal
    vrcpps  ymm3, ymm0      ; g(a) = 1/a (approx)
    
    ; Blend: select based on condition
    vblendvps ymm4, ymm3, ymm2, ymm1
    ; ymm4[i] = (mask[i]) ? sqrt(a[i]) : rcp(a[i])
    ret
```

### 8.2 ตัวอย่างจริง: Sigmoid Function

```nasm
; sigmoid(x) = 1 / (1 + exp(-x))
; ใช้ approximation เพื่อ avoid exp (ซับซ้อน)
; Fast approximation: sigmoid(x) ≈ 0.5 + 0.25*x for small |x|

sigmoid_approx_avx:
    ; ymm0 = input
    
    vbroadcastss ymm_half,  [rel .c_half]   ; 0.5
    vbroadcastss ymm_qtr,   [rel .c_qtr]    ; 0.25
    vbroadcastss ymm_one,   [rel .c_one]    ; 1.0
    vbroadcastss ymm_neg2,  [rel .c_neg2]   ; -2.0
    vbroadcastss ymm_pos2,  [rel .c_pos2]   ; 2.0
    vbroadcastss ymm_eps,   [rel .c_eps]    ; epsilon
    
    ; Clamp x to [-2, 2] for approximation validity
    vminps      ymm1, ymm0, ymm_pos2
    vmaxps      ymm1, ymm1, ymm_neg2
    
    ; Approximate: 0.5 + 0.25*x
    vfmadd213ps ymm1, ymm_qtr, ymm_half   ; 0.25*x + 0.5
    
    ; Clamp output to [0, 1]
    vminps  ymm1, ymm1, ymm_one
    vpxor   ymm2, ymm2, ymm2
    vmaxps  ymm1, ymm1, ymm2
    
    vmovaps ymm0, ymm1
    ret

section .data
    align 4
.c_half:  dd 0.5
.c_qtr:   dd 0.25
.c_one:   dd 1.0
.c_neg2:  dd -2.0
.c_pos2:  dd 2.0
.c_eps:   dd 0.001
```

### 8.3 Avoid Branch Divergence

```nasm
; Bad: branch ใน loop (unpredictable)
bad_conditional_loop:
    ; rdi = array a
    ; rsi = array b  
    ; rdx = result
    ; ecx = count
.loop:
    movss   xmm0, [rdi]
    movss   xmm1, [rsi]
    
    ; SLOW: branch ทุก iteration
    comiss  xmm0, xmm1
    jge     .take_a
    movss   [rdx], xmm1
    jmp     .next
.take_a:
    movss   [rdx], xmm0
.next:
    add     rdi, 4
    add     rsi, 4
    add     rdx, 4
    dec     ecx
    jnz     .loop
    ret

; Good: branchless SIMD
good_conditional_simd:
    ; Process 8 elements at once
.loop:
    vmovups ymm0, [rdi + rax*4]
    vmovups ymm1, [rsi + rax*4]
    vmaxps  ymm2, ymm0, ymm1         ; max(a, b) - branchless
    vmovups [rdx + rax*4], ymm2
    add     eax, 8
    sub     ecx, 8
    jg      .loop
    ret
```

---

## 9. Lookup Table ด้วย SIMD: PSHUFB สำหรับ 4-bit Lookups

### 9.1 PSHUFB Basics

PSHUFB (Packed Shuffle Bytes) ใช้ register หนึ่งเป็น shuffle control สำหรับ register อื่น

```nasm
; PSHUFB: xmm_dst[i] = xmm_src[xmm_ctrl[i] & 0xF]
; ถ้า xmm_ctrl[i] bit 7 = 1, output[i] = 0

pshufb_example:
    movdqa  xmm0, [rel .lookup_table]  ; 16-byte lookup table
    movdqa  xmm1, [rel .indices]       ; 16 indices (0-15)
    pshufb  xmm1, xmm0
    ; xmm1[i] = lookup_table[indices[i]]
    ret
```

### 9.2 4-bit Lookup Table

```nasm
; Lookup table สำหรับ nibble values (0-15)
; ใช้ PSHUFB เพื่อ process 16 bytes พร้อมกัน

nibble_lookup:
    ; xmm0 = 16 bytes ที่ต้องการ lookup nibbles ของ
    ; สร้าง table สำหรับ low nibbles
    
    movdqa  xmm_lo_table, [rel .low_nibble_table]   ; table[0..15]
    movdqa  xmm_hi_table, [rel .high_nibble_table]  ; shifted table
    
    ; Extract low nibbles
    movdqa  xmm1, xmm0
    pand    xmm1, [rel .nibble_mask]    ; xmm1 = x & 0x0F
    pshufb  xmm_lo_table, xmm1          ; lookup low nibbles
    
    ; Extract high nibbles
    movdqa  xmm2, xmm0
    psrlw   xmm2, 4                     ; shift right 4 bits
    pand    xmm2, [rel .nibble_mask]    ; xmm2 = (x >> 4) & 0x0F
    pshufb  xmm_hi_table, xmm2          ; lookup high nibbles
    
    ; Combine results
    paddb   xmm_lo_table, xmm_hi_table
    movdqa  xmm0, xmm_lo_table
    ret

section .data
    align 16
.low_nibble_table:
    db 0, 1, 1, 2, 1, 2, 2, 3   ; popcount lookup for nibbles 0-7
    db 1, 2, 2, 3, 2, 3, 3, 4   ; popcount lookup for nibbles 8-15
.high_nibble_table:
    ; same as low
    db 0, 1, 1, 2, 1, 2, 2, 3
    db 1, 2, 2, 3, 2, 3, 3, 4
.nibble_mask:
    times 16 db 0x0F
```

### 9.3 ตัวอย่าง: Population Count ด้วย PSHUFB

```nasm
; นับจำนวน bits ที่เป็น 1 ใน 16 bytes พร้อมกัน
popcount_16bytes:
    ; xmm0 = 16 bytes input
    
    movdqa  xmm_lookup, [rel .popcnt_table]
    movdqa  xmm_mask, [rel .low_nibble_mask]
    
    ; Low nibbles
    movdqa  xmm1, xmm0
    pand    xmm1, xmm_mask
    movdqa  xmm_lo, xmm_lookup
    pshufb  xmm_lo, xmm1
    
    ; High nibbles
    movdqa  xmm2, xmm0
    psrlw   xmm2, 4
    pand    xmm2, xmm_mask
    movdqa  xmm_hi, xmm_lookup
    pshufb  xmm_hi, xmm2
    
    ; Sum
    paddb   xmm_lo, xmm_hi
    ; xmm_lo[i] = popcount(input[i])
    
    ; Sum all bytes
    pxor    xmm2, xmm2
    psadbw  xmm_lo, xmm2     ; sum absolute differences from 0
    ; xmm_lo[0] = total popcount
    movd    eax, xmm_lo
    ret

section .data
    align 16
.popcnt_table:
    db 0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4
.low_nibble_mask:
    times 16 db 0x0F
```

---

## 10. Sorting Networks (Bitonic Sort) ด้วย SIMD

### 10.1 Sorting Networks Concept

Sorting network เป็น sequence ของ compare-and-swap operations ที่ fixed:
- ไม่มี conditional branches
- เหมาะมากสำหรับ SIMD

```
Bitonic Sort Network สำหรับ 8 elements:
Step 1: compare (0,1), (2,3), (4,5), (6,7)
Step 2: compare (0,2), (1,3), (4,6), (5,7)
Step 3: compare (1,2), (5,6)
Step 4: compare (0,4), (1,5), (2,6), (3,7)
Step 5: compare (2,4), (3,5)
Step 6: compare (1,2), (3,4), (5,6)
```

### 10.2 SIMD Bitonic Sort Implementation

```nasm
; Sort 8 floats using bitonic sort network + SIMD
; Input: ymm0 = [a0, a1, a2, a3, a4, a5, a6, a7]
; Output: ymm0 = sorted values

bitonic_sort_8:
    ; Step 1: compare (0,1), (2,3), (4,5), (6,7)
    vperm2f128  ymm1, ymm0, ymm0, 0x01     ; swap 128-bit halves (for later)
    vpermilps   ymm2, ymm0, 0b10110001     ; swap adjacent pairs
    vmaxps      ymm3, ymm0, ymm2
    vminps      ymm4, ymm0, ymm2
    vblendps    ymm0, ymm3, ymm4, 0b01010101  ; merge min/max
    
    ; Step 2: compare (0,2), (1,3), (4,6), (5,7)
    vpermilps   ymm2, ymm0, 0b00001110     ; rotate within 128-bit lanes
    vmaxps      ymm3, ymm0, ymm2
    vminps      ymm4, ymm0, ymm2
    vblendps    ymm0, ymm3, ymm4, 0b00110011
    
    ; Step 3: compare (1,2), (5,6)
    vpermilps   ymm2, ymm0, 0b11000110
    vmaxps      ymm3, ymm0, ymm2
    vminps      ymm4, ymm0, ymm2
    vblendps    ymm0, ymm3, ymm4, 0b01100110
    
    ; Step 4: compare (0,4), (1,5), (2,6), (3,7)
    vperm2f128  ymm2, ymm0, ymm0, 0x01     ; swap halves
    vmaxps      ymm3, ymm0, ymm2
    vminps      ymm4, ymm0, ymm2
    vperm2f128  ymm5, ymm3, ymm4, 0x20     ; blend halves
    vmovaps     ymm0, ymm5
    
    ; Steps 5-6: finish merge
    ; ... (similar pattern)
    
    ret
```

### 10.3 ตัวอย่าง: Sort 4 Floats (SSE)

```nasm
; Sort 4 floats using SIMD compare-and-swap
sort4_sse:
    ; xmm0 = [a, b, c, d]
    
    ; Step 1: compare adjacent pairs (a,b) and (c,d)
    movaps  xmm1, xmm0
    shufps  xmm1, xmm1, 0b10110001  ; [b, a, d, c]
    minps   xmm2, xmm0, xmm1        ; [min(a,b), min(b,a), ...]
    maxps   xmm3, xmm0, xmm1
    blendps xmm0, xmm3, xmm2, 0b0101  ; [min(a,b), max(a,b), min(c,d), max(c,d)]
    
    ; Step 2: compare (0,2) and (1,3)
    movaps  xmm1, xmm0
    shufps  xmm1, xmm1, 0b01001110  ; [c, d, a, b]
    minps   xmm2, xmm0, xmm1
    maxps   xmm3, xmm0, xmm1
    blendps xmm0, xmm3, xmm2, 0b0011
    
    ; Step 3: compare (1,2)
    movaps  xmm1, xmm0
    shufps  xmm1, xmm1, 0b11000110  ; [d, c, b, a] partial
    minps   xmm2, xmm0, xmm1
    maxps   xmm3, xmm0, xmm1
    blendps xmm0, xmm3, xmm2, 0b0110
    
    ret
```

---

## 11. SIMD Histogram (แก้ปัญหา Conflicts)

### 11.1 ปัญหา Histogram ใน SIMD

```
ปัญหา: histogram[index]++ ไม่สามารถ parallelize ตรงๆ
เพราะ indices อาจซ้ำกัน (data hazard)

ตัวอย่าง: ถ้า input = [3, 5, 3, 2]
histogram[3] += 2 (ต้องทำสอง atomic updates)
```

### 11.2 วิธีแก้: Multiple Partial Histograms

```nasm
; Histogram แบบ SIMD โดยใช้ multiple partial histograms
simd_histogram:
    ; rdi = input array (bytes 0-255)
    ; rsi = histogram array (256 x uint32)
    ; ecx = count
    
    ; สร้าง 4 partial histograms เพื่อลด conflicts
    ; partial_hist[k][v] = count of v in elements k, k+4, k+8, ...
    
    push    rbp
    mov     rbp, rsp
    sub     rsp, 4*256*4          ; 4 partial histograms
    
    ; Zero partial histograms
    lea     rdi_hist, [rbp - 4*256*4]
    mov     ecx, 4*256
    xor     eax, eax
    rep stosd
    
    xor     eax, eax
.loop:
    sub     ecx, 4
    jl      .done
    
    movzx   r8d, byte [rdi + rax]
    movzx   r9d, byte [rdi + rax + 1]
    movzx   r10d, byte [rdi + rax + 2]
    movzx   r11d, byte [rdi + rax + 3]
    
    inc     dword [rbp - 4*256*4 + r8*4]
    inc     dword [rbp - 3*256*4 + r9*4]
    inc     dword [rbp - 2*256*4 + r10*4]
    inc     dword [rbp - 1*256*4 + r11*4]
    
    add     eax, 4
    jmp     .loop
    
.done:
    ; Merge partial histograms
    xor     ecx, ecx
.merge:
    vmovdqu ymm0, [rbp - 4*256*4 + rcx*4]
    vmovdqu ymm1, [rbp - 3*256*4 + rcx*4]
    vmovdqu ymm2, [rbp - 2*256*4 + rcx*4]
    vmovdqu ymm3, [rbp - 1*256*4 + rcx*4]
    
    vpaddd  ymm0, ymm0, ymm1
    vpaddd  ymm2, ymm2, ymm3
    vpaddd  ymm0, ymm0, ymm2
    
    vmovdqu [rsi + rcx*4], ymm0
    
    add     ecx, 8
    cmp     ecx, 256
    jl      .merge
    
    vzeroupper
    leave
    ret
```

### 11.3 AVX-512 Conflict Detection

```nasm
; AVX-512 มี VPCONFLICTD สำหรับตรวจสอบ conflicts
avx512_histogram:
    ; zmm0 = 16 indices
    ; zmm1 = 16 values to add
    
    vpconflictd     zmm2, zmm0          ; detect conflicts
    ; zmm2[i] = bitmask of earlier elements with same index
    
    vptestmd        k1, zmm2, zmm2
    ; k1[i] = 1 ถ้า element i มี conflict กับ element ก่อนหน้า
    
    ; Process non-conflicting elements first
    kmovw       k2, 0xFFFF
    kandnw      k2, k1, k2             ; non-conflicting elements
    
    ; ... scatter-add ใน loop จนกว่า conflicts จะ resolved
    ret
```

---

## 12. Prefix Sum (Scan) ด้วย SIMD

### 12.1 Exclusive Prefix Sum

```
Prefix sum: out[i] = in[0] + in[1] + ... + in[i-1]
ตัวอย่าง: in = [3, 1, 4, 1, 5, 9, 2, 6]
         out = [0, 3, 4, 8, 9, 14, 23, 25]
```

### 12.2 SIMD Prefix Sum สำหรับ 8 Elements

```nasm
; Prefix sum ของ 8 floats ใน ymm register
; Input:  ymm0 = [a0, a1, a2, a3, a4, a5, a6, a7]
; Output: ymm0 = [0, a0, a0+a1, a0+a1+a2, ...]

prefix_sum_8:
    ; Step 1: shift right by 1 and add
    vpermilps   ymm1, ymm0, 0b10010011  ; [0, a0, a1, a2, a4, a5, a6, 0] - approximate
    ; ต้องใส่ 0 ที่ position 0
    vpxor       ymm2, ymm2, ymm2
    vblendps    ymm1, ymm1, ymm2, 0b00010001  ; zero out position 0 in each half
    vaddps      ymm0, ymm0, ymm1
    ; ymm0 = [a0, a0+a1, a1+a2, a2+a3, a4, a4+a5, a5+a6, a6+a7]
    
    ; Step 2: shift right by 2 and add
    vpermilps   ymm1, ymm0, 0b01001110  ; [a2+a3, a3+..., a0, a1, ...]
    ; ... complex shuffling needed
    
    ; Simpler approach: use scalar prefix then vectorize
    ; หรือใช้ approach แบบ parallel prefix
    
    ret

; More practical: scalar-assisted prefix sum
prefix_sum_array:
    ; rdi = input array
    ; rsi = output array
    ; ecx = count
    
    ; Process in blocks of 8
    vpxor   ymm_sum, ymm_sum, ymm_sum   ; running total (scalar broadcast)
    xor     eax, eax
.block:
    sub     ecx, 8
    jl      .remainder
    
    vmovups ymm0, [rdi + rax*4]
    
    ; Inclusive scan within block
    vpermilps   ymm1, ymm0, 0b00000000  ; a0, a0, a0, a0, ...
    vxorps      ymm2, ymm2, ymm2
    vblendps    ymm1, ymm1, ymm2, 0b11111110  ; only keep a0 at pos 0
    vaddps      ymm_running, ymm0, ymm1  ; partial sums
    
    ; ... (full prefix sum is complex, show partial implementation)
    
    vmovups [rsi + rax*4], ymm_running
    add     eax, 8
    jmp     .block
    
.remainder:
    ; scalar tail
    ret
```

### 12.3 Parallel Prefix Sum Algorithm

```nasm
; Up-sweep (reduce) phase สำหรับ parallel prefix sum
; ทำงานบน power-of-2 array size

up_sweep:
    ; rdi = array
    ; ecx = n (size, power of 2)
    
    mov     eax, 1          ; stride = 1
.outer:
    shl     eax, 1          ; stride *= 2
    cmp     eax, ecx
    jg      .done
    
    ; สำหรับแต่ละ k = aead, 2*stride, 3*stride, ...
    ; array[k-1] += array[k - stride - 1]
    
    ; Vectorize inner loop
    xor     ebx, ebx
.inner:
    add     ebx, eax        ; k = stride, 2*stride, ...
    cmp     ebx, ecx
    jge     .inner_done
    
    mov     r8d, ebx
    sub     r8d, 1          ; k-1
    mov     r9d, ebx
    sub     r9d, eax        ; k - stride
    sub     r9d, 1          ; k - stride - 1
    
    movss   xmm0, [rdi + r8*4]
    addss   xmm0, [rdi + r9*4]
    movss   [rdi + r8*4], xmm0
    
    jmp     .inner
.inner_done:
    jmp     .outer
.done:
    ret
```

---

## 13. SIMD String Operations Patterns

### 13.1 PCMPISTRI / PCMPISTRM - String Comparison Instructions

```nasm
; SSE4.2 string comparison instructions
; PCMPISTRI: Packed Compare Implicit Length Strings, Return Index
; PCMPISTRM: Packed Compare Implicit Length Strings, Return Mask

; ค้นหา null terminator
find_null:
    ; rdi = string
    pxor        xmm0, xmm0     ; zero (target: find null byte)
    xor         eax, eax
.loop:
    movdqu      xmm1, [rdi + rax]
    pcmpistri   xmm0, xmm1, 0x08   ; equal each, unsigned bytes
    jz          .not_found_in_block
    ; CF=1 means null found
    jc          .found
    add         eax, 16
    jmp         .loop
.found:
    add         rax, rcx       ; eax + index = null position
    ret
.not_found_in_block:
    add         eax, 16
    jmp         .loop
```

### 13.2 SIMD strlen

```nasm
; Fast strlen ใช้ SSE4.2
strlen_sse42:
    ; rdi = string
    mov     rax, rdi
    
    ; ใช้ PCMPISTRI เพื่อหา null byte
    pxor    xmm0, xmm0
.loop:
    pcmpistri xmm0, [rax], 0x08   ; unsigned byte, equal each
    lea     rax, [rax + 16]
    jnz     .loop
    ; CF=0 means no null, continue
    ; CX = index of first null
    
    sub     rax, rdi
    sub     rax, 16         ; undo last add
    add     rax, rcx        ; add index within block
    ret
```

### 13.3 SIMD strstr (substring search)

```nasm
; ค้นหา pattern ใน string ด้วย SSE4.2
strstr_sse42:
    ; rdi = haystack
    ; rsi = needle (สมมติ <= 16 chars)
    
    movdqu  xmm1, [rsi]         ; load needle
    xor     eax, eax
.loop:
    pcmpistri xmm1, [rdi + rax], 0x0C  ; substring match
    ; CF=1: match found, CX = index
    jc      .found
    
    ; ZF=1: haystack ended
    jz      .not_found
    
    add     eax, 1           ; advance one byte (ใช้ CX เพื่อ efficiency)
    jmp     .loop
.found:
    lea     rax, [rdi + rax + rcx]  ; pointer to match
    ret
.not_found:
    xor     eax, eax
    ret
```

### 13.4 SIMD Character Classification

```nasm
; ตรวจสอบว่า bytes เป็น ASCII alphanumeric หรือไม่
is_alphanumeric_simd:
    ; xmm0 = 16 bytes to check
    ; returns: xmm0 = mask (0xFF where alphanumeric, 0x00 otherwise)
    
    ; Check if >= '0' (0x30)
    movdqa  xmm1, xmm0
    pcmpgtb xmm1, [rel .char_0_minus1]    ; x > '0'-1 → x >= '0'
    
    ; Check if <= '9' (0x39)
    movdqa  xmm2, [rel .char_9_plus1]
    pcmpgtb xmm2, xmm0                     ; '9'+1 > x → x <= '9'
    
    pand    xmm1, xmm2                      ; is digit
    
    ; Check 'A'-'Z' and 'a'-'z'
    ; ... similar pattern
    
    ; Combine masks
    ; por xmm1, xmm_upper, xmm_lower
    movdqa  xmm0, xmm1
    ret

section .data
    align 16
.char_0_minus1: times 16 db 0x2F   ; '0' - 1
.char_9_plus1:  times 16 db 0x3A   ; '9' + 1
```

---

## 14. Bit Manipulation Patterns: PTEST, PMOVMSKB

### 14.1 PTEST

PTEST ทดสอบ bits โดยไม่ modify registers

```nasm
; PTEST: set ZF ถ้า (a AND b) == 0
;        set CF ถ้า (a AND NOT b) == 0

ptest_examples:
    ; ตรวจสอบว่า xmm0 เป็น all-zeros
    pxor    xmm1, xmm1
    ptest   xmm0, xmm0      ; test if xmm0 & xmm0 == 0
    jz      .is_zero
    
    ; ตรวจสอบว่า xmm0 มีค่า negative floats
    pcmpeqd xmm1, xmm1
    psrld   xmm1, 1         ; xmm1 = 0x7FFFFFFF (sign bit mask inverted)
    ptest   xmm0, xmm1      ; test non-sign bits
    ; CF=1 ถ้า xmm0 มีแต่ sign bits (all negative or zero)
    
.is_zero:
    ret
```

### 14.2 PMOVMSKB

PMOVMSKB สกัด MSB ของแต่ละ byte เป็น integer bitmask

```nasm
; PMOVMSKB: extract MSB (bit 7) from each byte
pmovmskb_examples:
    ; xmm0 = 16 bytes
    pmovmskb eax, xmm0
    ; eax = 16-bit mask, bit i = MSB of byte i
    
    ; ใช้งาน: ตรวจสอบว่า bytes ไหนเป็น negative (signed)
    
    ; ตรวจสอบว่า string มี non-ASCII (bit 7 set)
    movdqu  xmm0, [rdi]
    pmovmskb eax, xmm0
    test    eax, eax         ; any high bits set?
    jnz     .has_non_ascii
    
    ; นับ matching elements
    pcmpeqb xmm0, xmm1       ; compare
    pmovmskb eax, xmm0
    popcnt  eax, eax          ; count matching bytes
    ret

; ใช้ PMOVMSKB สำหรับ vectorized string search
pmovmskb_search:
    ; rdi = string
    ; al = target byte
    movd        xmm1, eax
    pxor        xmm2, xmm2
    pshufb      xmm1, xmm2    ; broadcast byte to all positions
    
    xor         eax, eax
.loop:
    movdqu      xmm0, [rdi + rax]
    pcmpeqb     xmm0, xmm1    ; compare each byte with target
    pmovmskb    ecx, xmm0     ; extract match mask
    
    test        ecx, ecx
    jnz         .found         ; found if any bit set
    
    add         eax, 16
    ; check for end of string...
    jmp         .loop
.found:
    bsf         ecx, ecx       ; find first set bit
    add         eax, ecx       ; index = block_start + bit_index
    ret
```

---

## 15. Population Count ด้วย SIMD

### 15.1 POPCNT instruction

```nasm
; Hardware POPCNT instruction (SSE4.2)
popcount_single:
    popcnt  eax, edi    ; 32-bit popcount
    ret

popcount_64:
    popcnt  rax, rdi    ; 64-bit popcount
    ret
```

### 15.2 SIMD Population Count สำหรับ Arrays

```nasm
; นับ total bits ใน array ด้วย SIMD
popcount_array_simd:
    ; rdi = array of uint64
    ; ecx = count (number of 64-bit elements)
    
    xor     rax, rax            ; total count
.loop:
    dec     ecx
    js      .done
    
    popcnt  rdx, [rdi + rcx*8]
    add     rax, rdx
    jmp     .loop
.done:
    ret

; ด้วย VPSHUFB method สำหรับ byte-level popcount
popcount_bytes_avx2:
    ; ymm0 = 32 bytes
    
    vmovdqa ymm_lookup, [rel .popcount_nibble]
    vmovdqa ymm_mask, [rel .nibble_mask_32]
    
    ; Low nibbles
    vpand   ymm1, ymm0, ymm_mask
    vpshufb ymm_lo, ymm_lookup, ymm1
    
    ; High nibbles
    vpsrlw  ymm2, ymm0, 4
    vpand   ymm2, ymm2, ymm_mask
    vpshufb ymm_hi, ymm_lookup, ymm2
    
    ; Sum nibble popcounts
    vpaddb  ymm0, ymm_lo, ymm_hi
    ; ymm0[i] = popcount of byte i
    
    ; Sum all bytes
    vpxor   ymm1, ymm1, ymm1
    vpsadbw ymm0, ymm0, ymm1    ; sum 8 bytes in each 64-bit group
    ; ymm0 contains sums in positions 0, 4 of each 128-bit lane
    
    ; Extract and sum results
    vextracti128 xmm1, ymm0, 1
    paddq    xmm0, xmm1
    movd     eax, xmm0
    pextrd   edx, xmm0, 2
    add      eax, edx
    
    vzeroupper
    ret

section .data
    align 32
.popcount_nibble:
    db 0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4  ; 16 bytes
    db 0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4  ; repeat for 32-byte
.nibble_mask_32:
    times 32 db 0x0F
```

---

## 16. SIMD Reduction: Sum/Min/Max ทุก Lanes

### 16.1 Horizontal Sum (Float)

```nasm
; Sum all 8 floats ใน ymm0
hsum_ymm:
    ; ymm0 = [a0, a1, a2, a3, a4, a5, a6, a7]
    
    vextractf128 xmm1, ymm0, 1          ; xmm1 = [a4, a5, a6, a7]
    vaddps       xmm0, xmm0, xmm1       ; xmm0 = [a0+a4, a1+a5, a2+a6, a3+a7]
    
    vpermilps    xmm1, xmm0, 0b00001110  ; [a2+a6, a3+a7, a0+a4, a1+a5]
    vaddps       xmm0, xmm0, xmm1       ; xmm0 = [a0+a4+a2+a6, ...]
    
    vpermilps    xmm1, xmm0, 0b00000001  ; [a1+..., a0+...]
    vaddss       xmm0, xmm0, xmm1       ; xmm0[0] = total sum
    
    vzeroupper
    ret

; Sum all 4 floats ใน xmm0 (SSE)
hsum_xmm:
    movaps   xmm1, xmm0
    shufps   xmm1, xmm1, 0b00001110  ; rotate
    addps    xmm0, xmm1
    movaps   xmm1, xmm0
    shufps   xmm1, xmm1, 0b00000001
    addss    xmm0, xmm1
    ret
```

### 16.2 Horizontal Min/Max

```nasm
; Find min of all 8 floats ใน ymm0
hmin_ymm:
    vextractf128 xmm1, ymm0, 1
    vminps       xmm0, xmm0, xmm1    ; pairwise min of halves
    
    vpermilps    xmm1, xmm0, 0b00001110
    vminps       xmm0, xmm0, xmm1
    
    vpermilps    xmm1, xmm0, 0b00000001
    vminss       xmm0, xmm0, xmm1    ; xmm0[0] = global min
    
    vzeroupper
    ret

; Find max of all 8 floats ใน ymm0
hmax_ymm:
    vextractf128 xmm1, ymm0, 1
    vmaxps       xmm0, xmm0, xmm1
    
    vpermilps    xmm1, xmm0, 0b00001110
    vmaxps       xmm0, xmm0, xmm1
    
    vpermilps    xmm1, xmm0, 0b00000001
    vmaxss       xmm0, xmm0, xmm1
    
    vzeroupper
    ret

; Find min and its index
hmin_with_index:
    ; ymm0 = values
    ; ymm1 = [0, 1, 2, 3, 4, 5, 6, 7] (indices)
    
    vextractf128 xmm2, ymm0, 1    ; high half values
    vextractf128 xmm3, ymm1, 1    ; high half indices
    
    ; Compare: which is smaller?
    vcmpps   xmm4, xmm0, xmm2, 1  ; xmm0 < xmm2?
    vblendvps xmm5, xmm3, xmm1, xmm4  ; select indices
    vblendvps xmm0, xmm2, xmm0, xmm4  ; select values
    vmovaps  xmm1, xmm5
    
    ; Now reduce xmm0/xmm1 (4 elements)
    vpermilps xmm2, xmm0, 0b00001110
    vpermilps xmm3, xmm1, 0b00001110
    vcmpps    xmm4, xmm0, xmm2, 1
    vblendvps xmm5, xmm3, xmm1, xmm4
    vblendvps xmm0, xmm2, xmm0, xmm4
    vmovaps   xmm1, xmm5
    
    vpermilps xmm2, xmm0, 0b00000001
    vpermilps xmm3, xmm1, 0b00000001
    vcmpss    xmm4, xmm0, xmm2, 1
    ; Select final min and index
    ; xmm0[0] = min value, xmm1[0] = min index
    
    vzeroupper
    ret
```

### 16.3 Integer Reduction

```nasm
; Sum all 8 int32 ใน ymm0
hsum_i32_ymm:
    vextracti128 xmm1, ymm0, 1
    vpaddd       xmm0, xmm0, xmm1
    
    vphaddd      xmm0, xmm0, xmm0   ; horizontal add pairs
    vphaddd      xmm0, xmm0, xmm0
    
    movd         eax, xmm0
    vzeroupper
    ret

; Sum all 16 bytes ใน xmm0 (using PSADBW)
hsum_u8_xmm:
    pxor     xmm1, xmm1
    psadbw   xmm0, xmm1     ; sum absolute differences from 0 = sum of bytes
    movd     eax, xmm0
    ; High 8 bytes sum in xmm0[64]
    pextrq   rdx, xmm0, 1
    add      eax, edx
    ret
```

---

## 17. Mixed-Precision: คำนวณใน Float, สะสมใน Double

### 17.1 ทำไมต้องใช้ Mixed-Precision

```
ปัญหา: Float (32-bit) มี precision แค่ ~7 digits
       ถ้า sum elements เยอะ error สะสม

แก้โดย: 
- คำนวณ intermediate results ใน float (เร็ว)
- สะสม accumulator ใน double (precise)
```

### 17.2 Mixed-Precision Dot Product

```nasm
; คำนวณ dot product ใน float, accumulate ใน double
mixed_precision_dot:
    ; rdi = array a (floats)
    ; rsi = array b (floats)
    ; ecx = count
    
    vxorpd   ymm_acc, ymm_acc, ymm_acc  ; double accumulator
    xor      eax, eax
.loop:
    sub      ecx, 8
    jl       .remainder
    
    ; Compute 8 products in float
    vmovups  ymm0, [rdi + rax*4]
    vmovups  ymm1, [rsi + rax*4]
    vmulps   ymm2, ymm0, ymm1           ; float products
    
    ; Convert to double and accumulate (4 at a time)
    vcvtps2pd ymm3, xmm2                ; low 4 floats to 4 doubles
    vextractf128 xmm2_hi, ymm2, 1
    vcvtps2pd ymm4, xmm2_hi             ; high 4 floats to 4 doubles
    
    vaddpd   ymm_acc, ymm_acc, ymm3     ; accumulate
    vaddpd   ymm_acc, ymm_acc, ymm4
    
    add      eax, 8
    jmp      .loop
    
.remainder:
    add      ecx, 8
    test     ecx, ecx
    jz       .done
.scalar:
    movss    xmm0, [rdi + rax*4]
    movss    xmm1, [rsi + rax*4]
    mulss    xmm0, xmm1
    vcvtss2sd xmm0, xmm0, xmm0
    addsd    [rel .acc_double], xmm0   ; scalar accumulate
    inc      eax
    dec      ecx
    jnz      .scalar
    
.done:
    ; Horizontal sum of ymm_acc (4 doubles)
    vextractf128 xmm1, ymm_acc, 1
    vaddpd   xmm_acc, xmm_acc, xmm1
    vhaddpd  xmm_acc, xmm_acc, xmm_acc
    ; xmm_acc[0] = total dot product (double)
    
    vzeroupper
    ret
```

### 17.3 Kahan Summation ใน SIMD (สำหรับ High Precision)

```nasm
; Kahan compensated summation
kahan_sum_simd:
    ; rdi = float array
    ; ecx = count
    
    vxorps   ymm_sum,   ymm_sum,  ymm_sum   ; sum = 0
    vxorps   ymm_comp,  ymm_comp, ymm_comp  ; compensation = 0
    xor      eax, eax
.loop:
    sub      ecx, 8
    jl       .done
    
    vmovups  ymm_input, [rdi + rax*4]      ; load 8 floats
    
    ; y = input - compensation
    vsubps   ymm_y, ymm_input, ymm_comp
    
    ; t = sum + y
    vaddps   ymm_t, ymm_sum, ymm_y
    
    ; compensation = (t - sum) - y
    vsubps   ymm_comp, ymm_t, ymm_sum
    vsubps   ymm_comp, ymm_comp, ymm_y
    
    ; sum = t
    vmovaps  ymm_sum, ymm_t
    
    add      eax, 8
    jmp      .loop
.done:
    ; Reduce ymm_sum
    ; ... horizontal sum of ymm_sum
    vzeroupper
    ret
```

---

## 18. Code Example: Complete SIMD Dot Product (ทุก x86 Variants)

### 18.1 Scalar Version (Reference)

```nasm
; Scalar dot product (baseline)
dot_product_scalar:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    ; returns: xmm0 = dot product
    
    xorps   xmm0, xmm0      ; accumulator = 0
    xor     eax, eax
.loop:
    dec     ecx
    js      .done
    
    movss   xmm1, [rdi + rax*4]
    mulss   xmm1, [rsi + rax*4]
    addss   xmm0, xmm1
    
    inc     eax
    jmp     .loop
.done:
    ret
```

### 18.2 SSE Version

```nasm
; SSE dot product (4 floats per iteration)
dot_product_sse:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    
    xorps   xmm0, xmm0      ; accumulator
    xor     eax, eax
    
    mov     r8d, ecx
    and     r8d, ~3          ; floor to multiple of 4
    
.vectorized:
    sub     r8d, 4
    js      .scalar_part
    
    movups  xmm1, [rdi + rax*4]
    movups  xmm2, [rsi + rax*4]
    mulps   xmm1, xmm2          ; element-wise multiply
    addps   xmm0, xmm1          ; accumulate
    
    add     eax, 4
    jmp     .vectorized
    
.scalar_part:
    mov     r8d, ecx
    and     r8d, 3              ; remainder
.scalar:
    test    r8d, r8d
    jz      .reduce
    movss   xmm1, [rdi + rax*4]
    mulss   xmm1, [rsi + rax*4]
    addss   xmm0, xmm1
    inc     eax
    dec     r8d
    jmp     .scalar
    
.reduce:
    ; Horizontal sum xmm0 = [a+b+c+d, ...]
    movaps  xmm1, xmm0
    shufps  xmm1, xmm1, 0b00001110  ; [c, d, a, b]
    addps   xmm0, xmm1               ; [a+c, b+d, c+a, d+b]
    movaps  xmm1, xmm0
    shufps  xmm1, xmm1, 0b00000001  ; [b+d, a+c, ...]
    addss   xmm0, xmm1               ; xmm0[0] = sum
    ret
```

### 18.3 SSE3 Version (HADDPS)

```nasm
; SSE3 dot product using HADDPS
dot_product_sse3:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    
    xorps   xmm_acc, xmm_acc, xmm_acc
    xor     eax, eax
    
    mov     r8d, ecx
    and     r8d, ~3
    
.loop:
    sub     r8d, 4
    js      .reduce
    
    movups  xmm1, [rdi + eax*4]
    movups  xmm2, [rsi + eax*4]
    mulps   xmm1, xmm2
    addps   xmm_acc, xmm1
    
    add     eax, 4
    jmp     .loop
    
.reduce:
    ; ใช้ haddps สำหรับ reduction
    haddps  xmm_acc, xmm_acc   ; [a+b, c+d, a+b, c+d]
    haddps  xmm_acc, xmm_acc   ; [a+b+c+d, ...]
    ; xmm_acc[0] = sum
    ret
```

### 18.4 SSE4.1 Version (DPPS)

```nasm
; SSE4.1 ใช้ DPPS instruction
dot_product_sse41:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    
    ; สำหรับ exactly 4 elements
    movups  xmm0, [rdi]
    movups  xmm1, [rsi]
    dpps    xmm0, xmm1, 0xFF   ; dot product: sum all 4, store to all
    ; xmm0[0] = dot product
    
    ; สำหรับ array ใหญ่:
    xorps   xmm_acc, xmm_acc, xmm_acc
    xor     eax, eax
    mov     r8d, ecx
    and     r8d, ~3
.loop:
    sub     r8d, 4
    js      .done
    
    movups  xmm0, [rdi + eax*4]
    dpps    xmm0, [rsi + eax*4], 0xF1  ; dp, accumulate to lane 0
    addss   xmm_acc, xmm0
    
    add     eax, 4
    jmp     .loop
.done:
    movaps  xmm0, xmm_acc
    ret
```

### 18.5 AVX Version

```nasm
; AVX dot product (8 floats per iteration)
dot_product_avx:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    
    vxorps  ymm_acc, ymm_acc, ymm_acc
    xor     eax, eax
    
    mov     r8d, ecx
    and     r8d, ~7            ; multiples of 8
    
.loop:
    sub     r8d, 8
    js      .reduce
    
    vmovups ymm0, [rdi + eax*4]
    vmovups ymm1, [rsi + eax*4]
    vmulps  ymm0, ymm0, ymm1
    vaddps  ymm_acc, ymm_acc, ymm0
    
    add     eax, 8
    jmp     .loop
    
.reduce:
    ; Horizontal sum ymm_acc (8 floats → 1)
    vextractf128 xmm1, ymm_acc, 1    ; high 4 floats
    vaddps   xmm_acc, xmm_acc, xmm1  ; sum halves
    
    vpermilps xmm1, xmm_acc, 0b00001110
    vaddps    xmm_acc, xmm_acc, xmm1
    
    vpermilps xmm1, xmm_acc, 0b00000001
    vaddss    xmm_acc, xmm_acc, xmm1
    
    vmovss    xmm0, xmm_acc          ; return in xmm0
    vzeroupper
    ret
```

### 18.6 AVX2 + FMA Version

```nasm
; AVX2 + FMA dot product (most efficient for Haswell+)
dot_product_avx2_fma:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    
    vxorps  ymm0, ymm0, ymm0    ; acc0
    vxorps  ymm1, ymm1, ymm1    ; acc1 (unrolled)
    vxorps  ymm2, ymm2, ymm2    ; acc2
    vxorps  ymm3, ymm3, ymm3    ; acc3
    
    xor     eax, eax
    mov     r8d, ecx
    and     r8d, ~31            ; multiples of 32 (4x8)
    
.loop:
    sub     r8d, 32
    js      .reduce
    
    ; Unroll 4x8 = 32 elements
    vfmadd231ps ymm0, ymm4,  [rdi + eax*4]      ; needs loading
    ; Better with pre-loaded registers:
    vmovups ymm4, [rdi + eax*4]
    vmovups ymm5, [rdi + eax*4 + 32]
    vmovups ymm6, [rdi + eax*4 + 64]
    vmovups ymm7, [rdi + eax*4 + 96]
    
    vfmadd231ps ymm0, ymm4, [rsi + eax*4]
    vfmadd231ps ymm1, ymm5, [rsi + eax*4 + 32]
    vfmadd231ps ymm2, ymm6, [rsi + eax*4 + 64]
    vfmadd231ps ymm3, ymm7, [rsi + eax*4 + 96]
    
    add     eax, 32
    jmp     .loop
    
.reduce:
    ; Combine 4 accumulators
    vaddps  ymm0, ymm0, ymm1
    vaddps  ymm2, ymm2, ymm3
    vaddps  ymm0, ymm0, ymm2
    
    ; Horizontal sum
    vextractf128 xmm1, ymm0, 1
    vaddps   xmm0, xmm0, xmm1
    vpermilps xmm1, xmm0, 0b00001110
    vaddps   xmm0, xmm0, xmm1
    vpermilps xmm1, xmm0, 0b00000001
    vaddss   xmm0, xmm0, xmm1
    
    vzeroupper
    ret
```

### 18.7 AVX-512 Version

```nasm
; AVX-512 dot product (16 floats per iteration)
dot_product_avx512:
    ; rdi = float array a
    ; rsi = float array b
    ; ecx = count
    
    vxorps  zmm0, zmm0, zmm0    ; acc0
    vxorps  zmm1, zmm1, zmm1    ; acc1
    
    xor     eax, eax
    mov     r8d, ecx
    and     r8d, ~31            ; multiples of 32
    
.loop:
    sub     r8d, 32
    js      .reduce
    
    vmovups zmm2, [rdi + eax*4]
    vmovups zmm3, [rdi + eax*4 + 64]
    
    vfmadd231ps zmm0, zmm2, [rsi + eax*4]
    vfmadd231ps zmm1, zmm3, [rsi + eax*4 + 64]
    
    add     eax, 32
    jmp     .loop
    
.reduce:
    vaddps  zmm0, zmm0, zmm1
    
    ; Reduce zmm0 (16 floats)
    vextractf32x8 ymm1, zmm0, 1
    vaddps   ymm0, ymm0, ymm1
    vextractf128 xmm1, ymm0, 1
    vaddps   xmm0, xmm0, xmm1
    vpermilps xmm1, xmm0, 0b00001110
    vaddps   xmm0, xmm0, xmm1
    vpermilps xmm1, xmm0, 0b00000001
    vaddss   xmm0, xmm0, xmm1
    
    vzeroupper
    ret
```

### 18.8 GAS Syntax Version

```gas
# AVX2 dot product ใน GAS syntax
# .intel_syntax noprefix ใช้งานได้เหมือน NASM
# หรือใช้ AT&T syntax:

    .intel_syntax noprefix
    .text
    .globl dot_product_gas
    .type dot_product_gas, @function

dot_product_gas:
    # rdi = a, rsi = b, edx = count
    vxorps  ymm0, ymm0, ymm0
    xor     eax, eax
    mov     ecx, edx
    and     ecx, -8          # floor to multiple of 8
    
1:  # local label
    sub     ecx, 8
    js      2f
    
    vmovups ymm1, [rdi + rax*4]
    vfmadd231ps ymm0, ymm1, [rsi + rax*4]
    
    add     eax, 8
    jmp     1b

2:
    vextractf128 xmm1, ymm0, 1
    vaddps   xmm0, xmm0, xmm1
    vpermilps xmm1, xmm0, 0x0E
    vaddps   xmm0, xmm0, xmm1
    vpermilps xmm1, xmm0, 0x01
    vaddss   xmm0, xmm0, xmm1
    
    vzeroupper
    ret
    .att_syntax prefix
```

---

## 19. Performance Comparison และ Benchmarking

### 19.1 ตัวอย่าง Performance Numbers (Approximate)

```
Dot Product Performance (10M float pairs, Intel Core i7-12700K):

Method          | Time    | Speedup | Throughput
----------------|---------|---------|----------
Scalar (C)      | 12.5 ms | 1.0x    | 800 Mops/s
SSE (4-wide)    | 3.4 ms  | 3.7x    | 2.9 Gops/s
AVX2 (8-wide)   | 1.8 ms  | 6.9x    | 5.5 Gops/s
AVX2+FMA        | 1.2 ms  | 10.4x   | 8.3 Gops/s
AVX-512 (16)    | 0.7 ms  | 17.8x   | 14.3 Gops/s
```

### 19.2 Profiling Tips

```nasm
; ใช้ RDTSC สำหรับ cycle counting
rdtsc_start:
    cpuid               ; serialize
    rdtsc               ; read time stamp
    shl     rdx, 32
    or      rax, rdx    ; rax = 64-bit TSC
    mov     [rel .tsc_start], rax
    ret

rdtsc_end:
    rdtscp              ; read and get processor ID
    shl     rdx, 32
    or      rax, rdx
    sub     rax, [rel .tsc_start]  ; elapsed cycles
    ret
```

---

## 20. Best Practices และ Common Pitfalls

### 20.1 Do's และ Don'ts

**DO:**
```nasm
; ✓ ใช้ aligned loads เมื่อเป็นไปได้
vmovaps ymm0, [aligned_ptr]     ; 32-byte aligned

; ✓ ใช้ vzeroupper หลัง AVX operations
vzeroupper

; ✓ Unroll loops เพื่อ hide latency
; ✓ ใช้ FMA แทน separate mul+add
vfmadd231ps ymm0, ymm1, ymm2    ; a += b*c

; ✓ ใช้ SoA layout สำหรับ SIMD
; ✓ Prefetch data ก่อนใช้
prefetcht0 [rdi + 128]
```

**DON'T:**
```nasm
; ✗ อย่าลืม vzeroupper (ทำให้ AVX-SSE transition penalty)
; ✗ อย่าใช้ unaligned loads บ่อยๆ
vmovups ymm0, [bad_ptr]         ; ช้ากว่า vmovaps

; ✗ อย่า mix AVX และ legacy SSE โดยไม่มี vzeroupper
; ✗ อย่าใช้ horizontal ops มากเกินจำเป็น

; ✗ อย่า scatter/gather เมื่อ sequential access ใช้ได้
```

### 20.2 AVX-SSE Transition Penalty

```nasm
; ปัญหา: ใช้ AVX instruction แล้วกลับมาใช้ SSE โดยไม่มี vzeroupper
; ทำให้เกิด large penalty บน Sandy Bridge/Ivy Bridge

; Wrong:
vmovaps ymm0, [rdi]       ; AVX operation
; ...
movaps  xmm0, [rsi]       ; SSE operation - PENALTY!

; Correct:
vmovaps ymm0, [rdi]
; ...
vzeroupper                 ; clear upper bits
movaps  xmm0, [rsi]       ; Now safe
```

### 20.3 Register Pressure

```nasm
; จัดการ register pressure:
; x86-64 มี 16 XMM/YMM/ZMM registers
; ต้อง balance ระหว่าง:
; - Loop unrolling (ต้องการ registers มากขึ้น)
; - Accumulator count
; - Spill/reload cost

; ตัวอย่าง: 4 accumulators สำหรับ FMA pipeline
vxorps  ymm0, ymm0, ymm0    ; acc0
vxorps  ymm1, ymm1, ymm1    ; acc1
vxorps  ymm2, ymm2, ymm2    ; acc2
vxorps  ymm3, ymm3, ymm3    ; acc3
; ใช้ ymm4-ymm15 สำหรับ input data
```

---

## 21. สรุป (Summary)

### 21.1 Key Takeaways

1. **Data Layout**: SoA ดีกว่า AoS สำหรับ SIMD operations บน single fields
2. **Vertical > Horizontal**: ใช้ lane-wise operations แทน horizontal เมื่อเป็นไปได้
3. **Alignment**: ทำ aligned access เมื่อทำได้, handle head/tail แยก
4. **Compute Both**: สำหรับ conditional code, compute both branches แล้ว blend
5. **PSHUFB**: ทรงพลังสำหรับ lookup tables และ shuffling
6. **FMA**: ใช้ FMA แทน separate multiply-add เสมอ
7. **vzeroupper**: ต้องเรียกหลัง AVX code ก่อน SSE code

### 21.2 Decision Tree สำหรับ SIMD Optimization

```
มี data ที่ process หลายชิ้นพร้อมกันไหม?
├── ใช่ → Layout เป็น sequential ไหม?
│   ├── ใช่ → ใช้ vmovups/vmovaps + arithmetic
│   └── ไม่ → ใช้ gather หรือ transpose ก่อน
│
มี conditional operations ไหม?
├── ใช่ → Compute both, then blend (vblendvps/vcmpps)
│
มี horizontal reduction ไหม?
├── ใช่ → ใช้ hsum/hmin/hmax pattern
│       (หลีกเลี่ยง haddps ถ้าทำได้)
│
มี string operations ไหม?
└── ใช่ → ใช้ PCMPISTRI/PMOVMSKB/PSHUFB patterns
```

### 21.3 Instruction Set ที่ใช้บ่อย

| Category | Instructions |
|----------|-------------|
| Load/Store | vmovaps, vmovups, vmaskmovps |
| Arithmetic | vaddps, vmulps, vfmadd231ps |
| Compare | vcmpps, vpcmpeqd, pcmpistri |
| Shuffle | vpermilps, vperm2f128, vpshufb |
| Blend | vblendps, vblendvps, vpblendvb |
| Reduction | vhaddps, vextractf128, vextracti128 |
| Bit ops | ptest, pmovmskb, vpconflictd |
| Convert | vcvtps2pd, vcvtss2sd |

---

## แบบฝึกหัด (Exercises)

1. **AoS to SoA**: แปลง AoS layout เป็น SoA สำหรับ struct ที่มี 6 fields, ทดสอบ performance

2. **Bitonic Sort**: implement bitonic sort สำหรับ 16 integers ด้วย SSE4.1

3. **SIMD Histogram**: สร้าง 256-bucket histogram สำหรับ grayscale image ด้วย AVX2

4. **Prefix Sum**: implement parallel prefix sum สำหรับ array ขนาด 1024 floats

5. **String Search**: implement SIMD version ของ memchr ด้วย SSE4.2

6. **Mixed Precision**: คำนวณ matrix-vector product ด้วย float input แต่ double accumulator

7. **Dot Product Benchmark**: เปรียบเทียบทุก versions (scalar, SSE, AVX2, AVX-512)

---

## อ้างอิง (References)

- Intel Intrinsics Guide: https://www.intel.com/content/www/us/en/docs/intrinsics-guide/
- Agner Fog's Optimization Manuals
- "Hacker's Delight" - Henry S. Warren Jr.
- Intel® 64 and IA-32 Architectures Software Developer Manuals
- "Computer Organization and Design" - Patterson & Hennessy
- SIMD programming examples ใน Intel Software Development Emulator

---

*Part 066 จบ | Part ต่อไป: Part 067 - AVX-512 Advanced Features*
