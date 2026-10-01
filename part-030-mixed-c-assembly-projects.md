# Part 030: Mixed C/Assembly Projects

## การสร้างโปรเจกต์ที่ผสมผสาน C และ Assembly

---

## บทนำ (Introduction)

การผสมผสาน C และ Assembly เป็นเทคนิคที่นักพัฒนาระดับมืออาชีพใช้เพื่อ:
- เพิ่มประสิทธิภาพจุดที่เป็น bottleneck ของโปรแกรม
- ควบคุม hardware โดยตรงในส่วนที่สำคัญ
- ใช้ SIMD instructions สำหรับการประมวลผลข้อมูลขนาดใหญ่
- สร้าง cryptographic functions ที่มีความปลอดภัยสูง

ในส่วนนี้เราจะเรียนรู้วิธีสร้างโปรเจกต์จริงที่รวม C และ Assembly เข้าด้วยกันอย่างมีระบบ

---

## 1. โครงสร้างโปรเจกต์ (Project Structure)

### 1.1 โครงสร้างไดเรกทอรีมาตรฐาน

```
mixed_project/
├── CMakeLists.txt          # Build system configuration
├── include/
│   ├── asm_funcs.h         # Header สำหรับ Assembly functions
│   └── utils.h             # Utility headers
├── src/
│   ├── main.c              # Main C program
│   ├── c_funcs.c           # C functions
│   └── asm/
│       ├── string_ops.asm  # String operations in Assembly
│       ├── math_ops.asm    # Math operations in Assembly
│       └── crypto.asm      # Cryptographic functions
├── tests/
│   ├── test_string.c       # Test harness for string functions
│   ├── test_math.c         # Test harness for math functions
│   └── test_crypto.c       # Test harness for crypto functions
└── benchmarks/
    └── benchmark.c         # Performance benchmarks
```

### 1.2 Header File สำหรับ Assembly Functions

```c
// include/asm_funcs.h
// ไฟล์ header สำหรับประกาศ functions ที่เขียนด้วย Assembly
// ต้องใช้ extern "C" ถ้าใช้กับ C++

#ifndef ASM_FUNCS_H
#define ASM_FUNCS_H

#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

// === String Functions ===
// ฟังก์ชันสำหรับจัดการ string ที่เขียนด้วย Assembly

// หาความยาว string (เร็วกว่า C standard library)
size_t asm_strlen(const char *str);

// คัดลอก string จาก src ไป dst
char* asm_strcpy(char *dst, const char *src);

// คัดลอกหน่วยความจำ n bytes
void* asm_memcpy(void *dst, const void *src, size_t n);

// เปรียบเทียบ string
int asm_strcmp(const char *s1, const char *s2);

// === Math Functions ===
// ฟังก์ชันคณิตศาสตร์ที่ optimized ด้วย SIMD

// คำนวณ dot product ของสองเวกเตอร์
float asm_dot_product(const float *a, const float *b, size_t n);

// คูณเมทริกซ์ 4x4
void asm_matrix_multiply_4x4(const float *a, const float *b, float *result);

// === Image Processing ===
// ฟังก์ชันประมวลผลภาพที่ใช้ SIMD

// แปลงภาพเป็น grayscale
void asm_rgb_to_grayscale(
    const uint8_t *rgb,    // input: RGB pixels
    uint8_t *gray,         // output: grayscale pixels
    size_t pixel_count     // จำนวน pixels
);

// Gaussian blur filter
void asm_blur_filter(
    const uint8_t *input,
    uint8_t *output,
    int width,
    int height
);

// === Cryptography ===
// ฟังก์ชัน cryptographic ที่ optimized

// XOR cipher
void asm_xor_cipher(
    const uint8_t *input,
    uint8_t *output,
    size_t length,
    const uint8_t *key,
    size_t key_length
);

// CRC32 checksum
uint32_t asm_crc32(const uint8_t *data, size_t length);

// SHA-256 core transform
void asm_sha256_transform(
    uint32_t state[8],
    const uint8_t block[64]
);

#ifdef __cplusplus
}
#endif

#endif // ASM_FUNCS_H
```

---

## 2. Build System ด้วย CMake

### 2.1 CMakeLists.txt หลัก

```cmake
# CMakeLists.txt
# Build system สำหรับโปรเจกต์ Mixed C/Assembly

cmake_minimum_required(VERSION 3.16)
project(MixedCAssembly C ASM_NASM)

# กำหนด standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# กำหนด NASM flags
set(CMAKE_ASM_NASM_FLAGS "-f elf64 -g -F dwarf")

# เพิ่ม optimization flags สำหรับ C
set(CMAKE_C_FLAGS_RELEASE "-O3 -march=native -DNDEBUG")
set(CMAKE_C_FLAGS_DEBUG "-O0 -g3 -fsanitize=address")

# รวม include directories
include_directories(${CMAKE_SOURCE_DIR}/include)

# รวบรวม Assembly source files
set(ASM_SOURCES
    src/asm/string_ops.asm
    src/asm/math_ops.asm
    src/asm/crypto.asm
)

# รวบรวม C source files
set(C_SOURCES
    src/main.c
    src/c_funcs.c
)

# สร้าง library จาก Assembly functions
add_library(asm_lib STATIC ${ASM_SOURCES})

# สร้าง executable หลัก
add_executable(main_app ${C_SOURCES})
target_link_libraries(main_app asm_lib m)

# Build tests
enable_testing()

# Test: String functions
add_executable(test_string tests/test_string.c)
target_link_libraries(test_string asm_lib)
add_test(NAME StringTests COMMAND test_string)

# Test: Math functions
add_executable(test_math tests/test_math.c)
target_link_libraries(test_math asm_lib m)
add_test(NAME MathTests COMMAND test_math)

# Benchmark
add_executable(benchmark benchmarks/benchmark.c)
target_link_libraries(benchmark asm_lib m)

# แสดง build information
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "C compiler: ${CMAKE_C_COMPILER}")
message(STATUS "NASM: ${CMAKE_ASM_NASM_COMPILER}")
```

### 2.2 คำสั่ง Build

```bash
# สร้าง build directory
mkdir build && cd build

# Configure สำหรับ Release build
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build ทุกอย่าง
make -j$(nproc)

# รัน tests
make test

# หรือรัน ctest โดยตรง
ctest --verbose

# Build สำหรับ Debug
cmake .. -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)
```

### 2.3 Makefile แบบ Manual (ไม่ใช้ CMake)

```makefile
# Makefile สำหรับโปรเจกต์ Mixed C/Assembly
# ใช้ได้บน Linux x86-64

CC = gcc
NASM = nasm
AR = ar

# Flags
CFLAGS = -O2 -Wall -Wextra -I./include
NASMFLAGS = -f elf64 -g -F dwarf
LDFLAGS = -lm

# Directories
SRCDIR = src
ASMDIR = src/asm
TESTDIR = tests
OBJDIR = obj

# Source files
C_SRCS = $(SRCDIR)/main.c $(SRCDIR)/c_funcs.c
ASM_SRCS = $(ASMDIR)/string_ops.asm $(ASMDIR)/math_ops.asm $(ASMDIR)/crypto.asm

# Object files
C_OBJS = $(patsubst $(SRCDIR)/%.c, $(OBJDIR)/%.o, $(C_SRCS))
ASM_OBJS = $(patsubst $(ASMDIR)/%.asm, $(OBJDIR)/%.o, $(ASM_SRCS))

# Targets
.PHONY: all clean test benchmark

all: $(OBJDIR) main_app

$(OBJDIR):
	mkdir -p $(OBJDIR)

# Compile C files
$(OBJDIR)/%.o: $(SRCDIR)/%.c
	$(CC) $(CFLAGS) -c $< -o $@

# Assemble NASM files
$(OBJDIR)/%.o: $(ASMDIR)/%.asm
	$(NASM) $(NASMFLAGS) $< -o $@

# Link everything
main_app: $(C_OBJS) $(ASM_OBJS)
	$(CC) $^ -o $@ $(LDFLAGS)

# Build and run tests
test: $(OBJDIR)/test_string $(OBJDIR)/test_math
	./$(OBJDIR)/test_string
	./$(OBJDIR)/test_math

$(OBJDIR)/test_string: $(TESTDIR)/test_string.c $(ASM_OBJS)
	$(CC) $(CFLAGS) $^ -o $@ $(LDFLAGS)

$(OBJDIR)/test_math: $(TESTDIR)/test_math.c $(ASM_OBJS)
	$(CC) $(CFLAGS) $^ -o $@ $(LDFLAGS)

# Build benchmark
benchmark: $(OBJDIR)/benchmark
	./$(OBJDIR)/benchmark

$(OBJDIR)/benchmark: $(SRCDIR)/benchmarks/benchmark.c $(ASM_OBJS)
	$(CC) $(CFLAGS) -O3 $^ -o $@ $(LDFLAGS)

clean:
	rm -rf $(OBJDIR) main_app
```

---

## 3. Profiling C Code เพื่อหา Hotspots

### 3.1 ใช้ gprof

```c
// src/profile_example.c
// ตัวอย่างโปรแกรมสำหรับ profiling

#include <stdio.h>
#include <string.h>
#include <time.h>

// ฟังก์ชันที่จะ profile
void process_strings(char **strings, int count) {
    for (int i = 0; i < count; i++) {
        size_t len = strlen(strings[i]);  // จุดที่อาจเป็น bottleneck
        // ทำงานกับ string...
        (void)len;
    }
}

void compute_math(float *data, int n) {
    float sum = 0;
    for (int i = 0; i < n; i++) {
        sum += data[i] * data[i];  // จุดที่อาจเป็น bottleneck
    }
}

int main() {
    // กำหนดให้ compile ด้วย -pg flag สำหรับ profiling
    // gcc -pg -O2 -o profile_test profile_example.c
    // ./profile_test
    // gprof profile_test gmon.out > analysis.txt
    
    printf("Building profile data...\n");
    return 0;
}
```

```bash
# คำสั่ง Profiling ด้วย gprof
# 1. Compile พร้อม profiling flags
gcc -pg -O2 -o profile_test src/profile_example.c

# 2. รันโปรแกรม (จะสร้างไฟล์ gmon.out)
./profile_test

# 3. วิเคราะห์ผลลัพธ์
gprof profile_test gmon.out > profile_analysis.txt
cat profile_analysis.txt

# ใช้ perf สำหรับ profiling ระดับ CPU
perf stat ./main_app
perf record ./main_app
perf report

# ใช้ valgrind callgrind
valgrind --tool=callgrind ./main_app
callgrind_annotate callgrind.out.*
```

---

## 4. String Library Optimizations

### 4.1 Fast strlen ด้วย Assembly

```nasm
; src/asm/string_ops.asm
; String operations ที่ optimized ด้วย Assembly x86-64
; Author: Assembly Course Part 030

section .text
global asm_strlen
global asm_strcpy
global asm_memcpy
global asm_strcmp

;==============================================================================
; asm_strlen - หาความยาว string แบบ optimized
; Input:  RDI = pointer ไปยัง string
; Output: RAX = ความยาว string (ไม่รวม null terminator)
; Uses:   REPNE SCASB instruction สำหรับ vectorized search
;==============================================================================
asm_strlen:
    ; บันทึก registers ที่ใช้
    push    rcx
    push    rdi

    ; เก็บ original pointer
    mov     rcx, -1             ; ตั้งค่า counter ให้ใหญ่ที่สุด
    xor     al, al              ; AL = 0 (null byte ที่ต้องหา)
    
    ; REPNE SCASB: scan memory for byte
    ; RDI advances, RCX decrements ทุก iteration
    ; หยุดเมื่อ AL == byte ที่ RDI
    repne scasb
    
    ; คำนวณความยาว
    ; RCX = -(length + 2) หลังจาก REPNE SCASB
    not     rcx                 ; NOT เพื่อ invert bits
    dec     rcx                 ; ลบ 1 เพราะ count null terminator ด้วย
    
    mov     rax, rcx            ; ส่งค่ากลับทาง RAX
    
    ; คืนค่า registers
    pop     rdi
    pop     rcx
    ret

;==============================================================================
; asm_strlen_sse2 - หาความยาว string ด้วย SSE2 (16 bytes ต่อครั้ง)
; Input:  RDI = pointer ไปยัง string
; Output: RAX = ความยาว string
; หมายเหตุ: เร็วกว่า standard strlen มากสำหรับ string ยาว
;==============================================================================
asm_strlen_sse2:
    push    rbx
    
    xor     eax, eax            ; ตั้งค่า counter = 0
    
    ; สร้าง XMM register ที่มีแต่ null bytes (0x00)
    pxor    xmm0, xmm0          ; XMM0 = 0x00000000...
    
    ; ทำให้ address เป็น 16-byte aligned
    mov     rbx, rdi
    and     rbx, 15             ; rbx = offset from alignment
    
    test    rbx, rbx            ; ตรวจสอบว่า aligned แล้วหรือยัง
    jz      .aligned_loop       ; ถ้า aligned แล้ว ไปที่ loop หลัก
    
    ; จัดการ bytes ก่อน alignment
    sub     rdi, rbx            ; ย้อนกลับไปที่ aligned address
    
.aligned_loop:
    ; โหลด 16 bytes จาก memory
    movdqa  xmm1, [rdi]         ; โหลด 16 bytes (aligned)
    
    ; เปรียบเทียบกับ null byte
    pcmpeqb xmm1, xmm0          ; xmm1[i] = 0xFF ถ้า byte[i] == 0
    
    ; ตรวจสอบว่ามี null byte ไหม
    pmovmskb ecx, xmm1          ; ecx = bitmask ของ null bytes
    
    test    ecx, ecx            ; มี null byte ไหม?
    jnz     .found_null         ; ถ้าเจอ ไปคำนวณตำแหน่ง
    
    add     rdi, 16             ; เลื่อน pointer ไปข้างหน้า 16 bytes
    add     eax, 16             ; เพิ่ม counter
    jmp     .aligned_loop
    
.found_null:
    ; ecx มี bitmask: หา position ของ null byte แรก
    bsf     ecx, ecx            ; หา bit สุดท้ายที่เป็น 1 (First null position)
    add     rax, rcx            ; เพิ่ม offset ไปที่ counter
    
    ; ปรับผลลัพธ์สำหรับ alignment
    pop     rbx
    ret

;==============================================================================
; asm_strcpy - คัดลอก string จาก src ไป dst
; Input:  RDI = dst pointer, RSI = src pointer
; Output: RAX = dst pointer (เหมือน C standard)
; หมายเหตุ: ไม่ตรวจสอบ buffer overflow! ใช้อย่างระวัง
;==============================================================================
asm_strcpy:
    ; บันทึก destination pointer สำหรับ return value
    push    rdi
    
.copy_loop:
    ; โหลด byte จาก source
    mov     al, [rsi]           ; AL = byte จาก source
    mov     [rdi], al           ; เก็บ byte ไปที่ destination
    
    ; ตรวจสอบว่าเป็น null terminator ไหม
    test    al, al              ; set flags จาก AL
    jz      .done               ; ถ้าเป็น 0 เสร็จแล้ว
    
    ; เลื่อน pointer ไปข้างหน้า
    inc     rsi
    inc     rdi
    jmp     .copy_loop
    
.done:
    ; คืนค่า destination pointer
    pop     rax
    ret

;==============================================================================
; asm_strcpy_optimized - คัดลอก string แบบ 8 bytes ต่อครั้ง
; ใช้ QWORD operations สำหรับประสิทธิภาพที่ดีกว่า
; Input:  RDI = dst pointer, RSI = src pointer
; Output: RAX = dst pointer
;==============================================================================
asm_strcpy_optimized:
    push    rdi                 ; บันทึก dst สำหรับ return
    push    rbx
    
    ; ตรวจสอบ alignment
    mov     rax, rsi
    and     rax, 7              ; ตรวจสอบ 8-byte alignment
    jnz     .byte_copy          ; ถ้าไม่ aligned ใช้ byte copy
    
.qword_copy:
    ; โหลด 8 bytes
    mov     rbx, [rsi]
    
    ; ตรวจสอบว่ามี null byte ใน 8 bytes ไหม
    ; เทคนิค: (x - 0x0101010101010101) & ~x & 0x8080808080808080
    mov     rax, rbx
    sub     rax, 0x0101010101010101
    not     rbx
    and     rax, rbx
    and     rax, 0x8080808080808080
    jnz     .byte_copy          ; มี null byte ใน 8 bytes นี้
    
    ; ไม่มี null byte, copy ทั้ง 8 bytes
    mov     rbx, [rsi]
    mov     [rdi], rbx
    add     rsi, 8
    add     rdi, 8
    jmp     .qword_copy
    
.byte_copy:
    ; copy ทีละ byte จนกว่าจะเจอ null
    mov     al, [rsi]
    mov     [rdi], al
    test    al, al
    jz      .copy_done
    inc     rsi
    inc     rdi
    jmp     .byte_copy
    
.copy_done:
    pop     rbx
    pop     rax
    ret

;==============================================================================
; asm_memcpy - คัดลอก n bytes จาก src ไป dst
; Input:  RDI = dst, RSI = src, RDX = n (number of bytes)
; Output: RAX = dst
; Optimized: ใช้ REP MOVSQ สำหรับ aligned data
;==============================================================================
asm_memcpy:
    push    rdi                 ; บันทึก dst สำหรับ return
    push    rcx
    
    ; จัดการกรณี n = 0
    test    rdx, rdx
    jz      .memcpy_done
    
    ; copy 8 bytes ต่อครั้ง
    mov     rcx, rdx
    shr     rcx, 3              ; rcx = n / 8 (จำนวน QWORD ที่จะ copy)
    
    ; REP MOVSQ: copy rcx QWORDs จาก [rsi] ไป [rdi]
    rep movsq
    
    ; จัดการ bytes ที่เหลือ (< 8 bytes)
    mov     rcx, rdx
    and     rcx, 7              ; rcx = n % 8
    
    ; REP MOVSB: copy bytes ที่เหลือ
    rep movsb
    
.memcpy_done:
    pop     rcx
    pop     rax
    ret

;==============================================================================
; asm_memcpy_avx2 - memcpy ด้วย AVX2 (32 bytes ต่อครั้ง)
; ต้องการ CPU ที่รองรับ AVX2 (Intel Haswell หรือใหม่กว่า)
; Input:  RDI = dst, RSI = src, RDX = n
; Output: RAX = dst
;==============================================================================
asm_memcpy_avx2:
    push    rdi
    
    mov     rax, rdx
    shr     rax, 5              ; rax = n / 32
    test    rax, rax
    jz      .avx2_remainder
    
.avx2_loop:
    ; โหลด 32 bytes จาก source (unaligned)
    vmovdqu ymm0, [rsi]
    ; เก็บ 32 bytes ไปที่ destination (unaligned)
    vmovdqu [rdi], ymm0
    add     rsi, 32
    add     rdi, 32
    dec     rax
    jnz     .avx2_loop
    
.avx2_remainder:
    mov     rcx, rdx
    and     rcx, 31             ; จัดการ bytes ที่เหลือ
    rep movsb
    
    ; ล้าง YMM registers (สำคัญมาก!)
    vzeroupper
    
    pop     rax
    ret

;==============================================================================
; asm_strcmp - เปรียบเทียบ string
; Input:  RDI = s1, RSI = s2
; Output: RAX = negative ถ้า s1 < s2, 0 ถ้าเท่ากัน, positive ถ้า s1 > s2
;==============================================================================
asm_strcmp:
.compare_loop:
    ; โหลด bytes จากทั้งสอง string
    movzx   eax, byte [rdi]     ; EAX = *s1 (zero-extended)
    movzx   ecx, byte [rsi]     ; ECX = *s2 (zero-extended)
    
    ; ตรวจสอบว่า s1 จบแล้วไหม
    test    al, al
    jz      .end_of_string
    
    ; เปรียบเทียบ
    cmp     al, cl
    jne     .different
    
    ; เหมือนกัน เลื่อนไปข้างหน้า
    inc     rdi
    inc     rsi
    jmp     .compare_loop
    
.end_of_string:
    ; s1 จบแล้ว ตรวจสอบ s2
    test    cl, cl
    jnz     .different          ; s2 ยังมีข้อมูล = s1 < s2
    xor     eax, eax            ; ทั้งสองเท่ากัน
    ret
    
.different:
    sub     eax, ecx            ; คืนค่า difference
    ret
```

---

## 5. Math Optimizations

### 5.1 Dot Product และ Matrix Multiply

```nasm
; src/asm/math_ops.asm
; Mathematical operations ที่ optimized ด้วย SIMD
; ใช้ SSE/AVX instructions สำหรับ floating-point operations

section .text
global asm_dot_product
global asm_matrix_multiply_4x4
global asm_vector_add
global asm_vector_scale

;==============================================================================
; asm_dot_product - คำนวณ dot product ของสอง float arrays
; Input:  RDI = pointer ไปยัง array a
;         RSI = pointer ไปยัง array b  
;         RDX = จำนวน elements (n)
; Output: XMM0 = dot product (float)
; ใช้ SSE สำหรับ SIMD computation (4 floats ต่อครั้ง)
;==============================================================================
asm_dot_product:
    ; เริ่มต้น accumulator
    xorps   xmm0, xmm0          ; XMM0 = 0.0 (ผลรวมทั้งหมด)
    
    ; จัดการกรณี n = 0
    test    rdx, rdx
    jz      .dot_done
    
    ; คำนวณจำนวน iterations สำหรับ 4-element SIMD
    mov     rcx, rdx
    shr     rcx, 2              ; rcx = n / 4
    test    rcx, rcx
    jz      .scalar_dot         ; ถ้าน้อยกว่า 4 ใช้ scalar
    
.simd_loop:
    ; โหลด 4 floats จากทั้งสอง arrays
    movups  xmm1, [rdi]         ; XMM1 = a[0..3]
    movups  xmm2, [rsi]         ; XMM2 = b[0..3]
    
    ; คูณ element-wise
    mulps   xmm1, xmm2          ; XMM1[i] = a[i] * b[i]
    
    ; บวกเข้า accumulator
    addps   xmm0, xmm1          ; XMM0 += XMM1
    
    ; เลื่อน pointers
    add     rdi, 16             ; 4 floats * 4 bytes = 16 bytes
    add     rsi, 16
    dec     rcx
    jnz     .simd_loop
    
    ; Horizontal sum ของ XMM0 (4 floats -> 1 float)
    ; XMM0 = [a, b, c, d] -> a+b+c+d
    movaps  xmm1, xmm0
    shufps  xmm1, xmm0, 0x4E    ; XMM1 = [c, d, a, b]
    addps   xmm0, xmm1          ; XMM0 = [a+c, b+d, c+a, d+b]
    movaps  xmm1, xmm0
    shufps  xmm1, xmm0, 0x11    ; XMM1 = [b+d, a+c, b+d, a+c]
    addss   xmm0, xmm1          ; XMM0[0] = a+b+c+d
    
    ; จัดการ elements ที่เหลือ (n % 4)
    mov     rcx, rdx
    and     rcx, 3
    jz      .dot_done
    
.scalar_tail:
    ; คำนวณ elements ที่เหลือแบบ scalar
    movss   xmm1, [rdi]
    movss   xmm2, [rsi]
    mulss   xmm1, xmm2
    addss   xmm0, xmm1
    add     rdi, 4
    add     rsi, 4
    dec     rcx
    jnz     .scalar_tail
    jmp     .dot_done
    
.scalar_dot:
    ; คำนวณแบบ scalar ทั้งหมด
    mov     rcx, rdx
.scalar_loop:
    movss   xmm1, [rdi]
    movss   xmm2, [rsi]
    mulss   xmm1, xmm2
    addss   xmm0, xmm1
    add     rdi, 4
    add     rsi, 4
    dec     rcx
    jnz     .scalar_loop
    
.dot_done:
    ret

;==============================================================================
; asm_dot_product_avx - dot product ด้วย AVX (8 floats ต่อครั้ง)
; เร็วกว่า SSE version ประมาณ 2x
; Input/Output: เหมือนกับ asm_dot_product
;==============================================================================
asm_dot_product_avx:
    vxorps  ymm0, ymm0, ymm0    ; YMM0 = 0.0
    
    mov     rcx, rdx
    shr     rcx, 3              ; rcx = n / 8
    jz      .avx_remainder
    
.avx_loop:
    vmovups ymm1, [rdi]         ; โหลด 8 floats
    vmovups ymm2, [rsi]
    vfmadd231ps ymm0, ymm1, ymm2 ; ymm0 += ymm1 * ymm2 (FMA instruction)
    add     rdi, 32
    add     rsi, 32
    dec     rcx
    jnz     .avx_loop
    
.avx_remainder:
    ; Horizontal sum
    vextractf128 xmm1, ymm0, 1  ; แยก high 128 bits
    addps   xmm0, xmm1          ; บวม high + low
    
    ; จัดการ elements ที่เหลือ
    mov     rcx, rdx
    and     rcx, 7
    jz      .avx_done
    
.avx_scalar:
    movss   xmm1, [rdi]
    movss   xmm2, [rsi]
    mulss   xmm1, xmm2
    addss   xmm0, xmm1
    add     rdi, 4
    add     rsi, 4
    dec     rcx
    jnz     .avx_scalar
    
.avx_done:
    ; Horizontal sum ของ XMM0
    movaps  xmm1, xmm0
    shufps  xmm1, xmm0, 0x4E
    addps   xmm0, xmm1
    movaps  xmm1, xmm0
    shufps  xmm1, xmm0, 0x11
    addss   xmm0, xmm1
    
    vzeroupper                  ; ล้าง upper bits ของ YMM registers
    ret

;==============================================================================
; asm_matrix_multiply_4x4 - คูณเมทริกซ์ 4x4 float
; Input:  RDI = pointer ไปยัง matrix A (row-major, 16 floats)
;         RSI = pointer ไปยัง matrix B
;         RDX = pointer ไปยัง result matrix
; Output: ผลลัพธ์เก็บที่ [RDX]
; ใช้ SSE สำหรับ vectorized computation
;==============================================================================
asm_matrix_multiply_4x4:
    push    rbp
    push    rbx
    push    r12
    push    r13
    
    ; โหลด columns ของ matrix B
    ; B column 0: B[0], B[4], B[8], B[12]
    movss   xmm4, [rsi]
    movss   xmm5, [rsi + 16]
    movss   xmm6, [rsi + 32]
    movss   xmm7, [rsi + 48]
    
    ; วนลูปสำหรับแต่ละ row ของ A
    xor     rbx, rbx            ; row counter
    
.row_loop:
    cmp     rbx, 4
    jge     .mat_done
    
    ; โหลด row ของ A
    ; A[row*4 + 0..3]
    mov     r12, rbx
    imul    r12, 16             ; offset = row * 16 bytes
    
    movss   xmm0, [rdi + r12]       ; A[row][0]
    movss   xmm1, [rdi + r12 + 4]   ; A[row][1]
    movss   xmm2, [rdi + r12 + 8]   ; A[row][2]
    movss   xmm3, [rdi + r12 + 12]  ; A[row][3]
    
    ; คำนวณ result[row][col] สำหรับแต่ละ col
    xor     r13, r13            ; column counter
    
.col_loop:
    cmp     r13, 4
    jge     .next_row
    
    ; result[row][col] = sum(A[row][k] * B[k][col])
    ; โหลด column col ของ B
    mov     rcx, r13
    imul    rcx, 4              ; column offset
    
    movss   xmm8,  [rsi + rcx]        ; B[0][col]
    movss   xmm9,  [rsi + rcx + 16]   ; B[1][col]
    movss   xmm10, [rsi + rcx + 32]   ; B[2][col]
    movss   xmm11, [rsi + rcx + 48]   ; B[3][col]
    
    ; คำนวณ dot product
    mulss   xmm8,  xmm0         ; A[row][0] * B[0][col]
    mulss   xmm9,  xmm1         ; A[row][1] * B[1][col]
    mulss   xmm10, xmm2         ; A[row][2] * B[2][col]
    mulss   xmm11, xmm3         ; A[row][3] * B[3][col]
    
    addss   xmm8,  xmm9
    addss   xmm10, xmm11
    addss   xmm8,  xmm10
    
    ; เก็บผลลัพธ์
    mov     rcx, rbx
    imul    rcx, 16
    mov     rax, r13
    imul    rax, 4
    add     rcx, rax
    movss   [rdx + rcx], xmm8
    
    inc     r13
    jmp     .col_loop
    
.next_row:
    inc     rbx
    jmp     .row_loop
    
.mat_done:
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret
```

---

## 6. Image Processing

### 6.1 Grayscale Conversion และ Blur Filter

```nasm
; src/asm/image_ops.asm
; Image processing functions ที่ใช้ SIMD

section .data
; น้ำหนักสำหรับ grayscale conversion (BT.601)
; Gray = 0.299*R + 0.587*G + 0.114*B
; คูณด้วย 256 เพื่อใช้ integer arithmetic
gray_r  dw  77   ; 0.299 * 256 ≈ 77
gray_g  dw  150  ; 0.587 * 256 ≈ 150
gray_b  dw  29   ; 0.114 * 256 ≈ 29

section .text
global asm_rgb_to_grayscale
global asm_blur_filter

;==============================================================================
; asm_rgb_to_grayscale - แปลงภาพ RGB เป็น Grayscale
; Input:  RDI = pointer ไปยัง RGB data (3 bytes per pixel: R,G,B)
;         RSI = pointer ไปยัง output grayscale buffer
;         RDX = จำนวน pixels
; Output: ไม่มี (ผลลัพธ์เก็บที่ [RSI])
; ใช้ integer multiplication สำหรับ speed
;==============================================================================
asm_rgb_to_grayscale:
    ; บันทึก registers
    push    rbx
    push    r12
    push    r13
    
    mov     rcx, rdx            ; loop counter = pixel count
    test    rcx, rcx
    jz      .gray_done
    
    ; โหลด weights
    movzx   r12d, word [gray_r] ; R weight = 77
    movzx   r13d, word [gray_g] ; G weight = 150
    movzx   ebx,  word [gray_b] ; B weight = 29
    
.pixel_loop:
    ; โหลด RGB values
    movzx   eax, byte [rdi]     ; R
    movzx   edx, byte [rdi + 1] ; G
    movzx   r8d, byte [rdi + 2] ; B
    
    ; คำนวณ Gray = (77*R + 150*G + 29*B) / 256
    imul    eax, r12d           ; eax = 77 * R
    imul    edx, r13d           ; edx = 150 * G
    imul    r8d, ebx            ; r8d = 29 * B
    
    add     eax, edx
    add     eax, r8d
    shr     eax, 8              ; หาร 256 (shift right 8 bits)
    
    ; จำกัดค่าให้อยู่ใน 0-255
    cmp     eax, 255
    jle     .store_pixel
    mov     eax, 255
    
.store_pixel:
    mov     [rsi], al           ; เก็บ grayscale value
    
    ; เลื่อน pointers
    add     rdi, 3              ; RGB = 3 bytes
    inc     rsi
    dec     rcx
    jnz     .pixel_loop
    
.gray_done:
    pop     r13
    pop     r12
    pop     rbx
    ret

;==============================================================================
; asm_rgb_to_grayscale_simd - แปลง Grayscale ด้วย SIMD (เร็วกว่า 4x)
; ประมวลผล 4 pixels พร้อมกัน ด้วย SSE2
; Input:  RDI = RGB data, RSI = output, RDX = pixel count
;==============================================================================
asm_rgb_to_grayscale_simd:
    push    rbx
    
    ; เตรียม constants ใน XMM registers
    ; weights สำหรับ R, G, B
    movdqa  xmm7, [rel gray_weights_simd]  ; weights
    
    mov     rcx, rdx
    shr     rcx, 2              ; process 4 pixels at a time
    jz      .gray_simd_remainder
    
.gray_simd_loop:
    ; โหลด 12 bytes (4 RGB pixels)
    movdqu  xmm0, [rdi]         ; โหลด 16 bytes (4 pixels + 4 extra)
    
    ; แยก R, G, B channels
    ; ใช้ pshufb หรือ manual extraction
    ; (simplified version)
    
    add     rdi, 12
    add     rsi, 4
    dec     rcx
    jnz     .gray_simd_loop
    
.gray_simd_remainder:
    ; จัดการ pixels ที่เหลือ
    pop     rbx
    ret

;==============================================================================
; asm_blur_filter - Simple box blur filter
; Input:  RDI = input image data (grayscale)
;         RSI = output buffer
;         EDX = width
;         ECX = height
; Output: ผลลัพธ์ที่ [RSI]
; Blur kernel: 1/9 * [1,1,1; 1,1,1; 1,1,1]
;==============================================================================
asm_blur_filter:
    push    rbp
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; เก็บ parameters
    mov     r12, rdi            ; input
    mov     r13, rsi            ; output
    movsxd  r14, edx            ; width
    movsxd  r15, ecx            ; height
    
    ; วนลูปสำหรับแต่ละ row (ข้าม border)
    mov     rbx, 1              ; row = 1 (ข้าม row แรก)
    
.blur_row:
    cmp     rbx, r15
    jge     .blur_row_done
    dec     r15                 ; height - 1
    cmp     rbx, r15
    jge     .blur_row_done
    inc     r15
    
    ; วนลูปสำหรับแต่ละ column
    mov     rbp, 1              ; col = 1
    
.blur_col:
    cmp     rbp, r14
    jge     .next_blur_row
    dec     r14                 ; width - 1
    cmp     rbp, r14
    jge     .next_blur_row
    inc     r14
    
    ; คำนวณ average ของ 3x3 neighborhood
    xor     eax, eax
    
    ; row-1
    mov     rcx, rbx
    dec     rcx
    imul    rcx, r14
    add     rcx, rbp
    
    movzx   edx, byte [r12 + rcx - 1]
    add     eax, edx
    movzx   edx, byte [r12 + rcx]
    add     eax, edx
    movzx   edx, byte [r12 + rcx + 1]
    add     eax, edx
    
    ; row (current)
    mov     rcx, rbx
    imul    rcx, r14
    add     rcx, rbp
    
    movzx   edx, byte [r12 + rcx - 1]
    add     eax, edx
    movzx   edx, byte [r12 + rcx]
    add     eax, edx
    movzx   edx, byte [r12 + rcx + 1]
    add     eax, edx
    
    ; row+1
    mov     rcx, rbx
    inc     rcx
    imul    rcx, r14
    add     rcx, rbp
    
    movzx   edx, byte [r12 + rcx - 1]
    add     eax, edx
    movzx   edx, byte [r12 + rcx]
    add     eax, edx
    movzx   edx, byte [r12 + rcx + 1]
    add     eax, edx
    
    ; หาร 9 สำหรับ average
    ; เทคนิค: x / 9 ≈ (x * 0x1C72) >> 20
    mov     ecx, eax
    imul    ecx, 0x1C72
    shr     ecx, 20             ; ecx = eax / 9 (approximate)
    
    ; เก็บผลลัพธ์
    mov     rdx, rbx
    imul    rdx, r14
    add     rdx, rbp
    mov     [r13 + rdx], cl
    
    inc     rbp
    jmp     .blur_col
    
.next_blur_row:
    inc     rbx
    jmp     .blur_row
    
.blur_row_done:
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret
```

---

## 7. Cryptography Functions

### 7.1 XOR Cipher, CRC32 และ SHA-256 Core

```nasm
; src/asm/crypto.asm
; Cryptographic functions ที่ optimized ด้วย Assembly
; หมายเหตุ: XOR cipher ไม่ปลอดภัยสำหรับการใช้งานจริง
;            ใช้เป็นตัวอย่างการเขียน Assembly เท่านั้น

section .data
; CRC32 lookup table (32 entries สำหรับ demo)
; ในการใช้งานจริงต้องมี 256 entries
align 16
crc32_table:
    ; Pre-computed CRC32 table values
    dd 0x00000000, 0x77073096, 0xEE0E612C, 0x990951BA
    dd 0x076DC419, 0x706AF48F, 0xE963A535, 0x9E6495A3
    dd 0x0EDB8832, 0x79DCB8A4, 0xE0D5E91B, 0x97D2D988
    dd 0x09B64C2B, 0x7EB17CBF, 0xE7B82D09, 0x90BF1779
    dd 0x1DB71064, 0x6AB020F2, 0xF3B97148, 0x84BE41DE
    dd 0x1ADAD47D, 0x6DDDE4EB, 0xF4D4B551, 0x83D385C7
    dd 0x136C9856, 0x646BA8C0, 0xFD62F97A, 0x8A65C9EC
    dd 0x14015C4F, 0x63066CD9, 0xFA0F3D63, 0x8D080DF5

; SHA-256 constants (first 8 values)
sha256_k:
    dd 0x428A2F98, 0x71374491, 0xB5C0FBCF, 0xE9B5DBA5
    dd 0x3956C25B, 0x59F111F1, 0x923F82A4, 0xAB1C5ED5
    dd 0xD807AA98, 0x12835B01, 0x243185BE, 0x550C7DC3
    dd 0x72BE5D74, 0x80DEB1FE, 0x9BDC06A7, 0xC19BF174
    dd 0xE49B69C1, 0xEFBE4786, 0x0FC19DC6, 0x240CA1CC
    dd 0x2DE92C6F, 0x4A7484AA, 0x5CB0A9DC, 0x76F988DA
    dd 0x983E5152, 0xA831C66D, 0xB00327C8, 0xBF597FC7
    dd 0xC6E00BF3, 0xD5A79147, 0x06CA6351, 0x14292967
    dd 0x27B70A85, 0x2E1B2138, 0x4D2C6DFC, 0x53380D13
    dd 0x650A7354, 0x766A0ABB, 0x81C2C92E, 0x92722C85
    dd 0xA2BFE8A1, 0xA81A664B, 0xC24B8B70, 0xC76C51A3
    dd 0xD192E819, 0xD6990624, 0xF40E3585, 0x106AA070
    dd 0x19A4C116, 0x1E376C08, 0x2748774C, 0x34B0BCB5
    dd 0x391C0CB3, 0x4ED8AA4A, 0x5B9CCA4F, 0x682E6FF3
    dd 0x748F82EE, 0x78A5636F, 0x84C87814, 0x8CC70208
    dd 0x90BEFFFA, 0xA4506CEB, 0xBEF9A3F7, 0xC67178F2

section .text
global asm_xor_cipher
global asm_crc32
global asm_sha256_transform

;==============================================================================
; asm_xor_cipher - XOR cipher (เหมาะสำหรับ learning เท่านั้น)
; Input:  RDI = input buffer
;         RSI = output buffer
;         RDX = data length
;         RCX = key buffer
;         R8  = key length
; Output: ไม่มี (ผลลัพธ์เก็บที่ [RSI])
;==============================================================================
asm_xor_cipher:
    ; ตรวจสอบ length
    test    rdx, rdx
    jz      .xor_done
    
    ; ตรวจสอบ key length
    test    r8, r8
    jz      .xor_done
    
    xor     r9, r9              ; key index = 0
    mov     r10, rdx            ; counter = data length
    
.xor_loop:
    ; โหลด data byte
    movzx   eax, byte [rdi]
    
    ; โหลด key byte (วนรอบตาม key length)
    movzx   ecx, byte [rcx + r9]
    
    ; XOR operation
    xor     eax, ecx
    
    ; เก็บผลลัพธ์
    mov     [rsi], al
    
    ; เลื่อน pointers
    inc     rdi
    inc     rsi
    
    ; เพิ่ม key index (wrap around)
    inc     r9
    cmp     r9, r8
    jl      .no_wrap
    xor     r9, r9              ; reset key index
    
.no_wrap:
    dec     r10
    jnz     .xor_loop
    
.xor_done:
    ret

;==============================================================================
; asm_xor_cipher_simd - XOR cipher ด้วย XMM (16 bytes ต่อครั้ง)
; สำหรับ key ขนาด 16 bytes (128-bit key)
; Input:  RDI = input, RSI = output, RDX = length, RCX = key (16 bytes)
;==============================================================================
asm_xor_cipher_simd:
    ; โหลด key ทั้ง 16 bytes เข้า XMM7
    movdqu  xmm7, [rcx]
    
    mov     rax, rdx
    shr     rax, 4              ; rax = length / 16
    jz      .xor_simd_remainder
    
.xor_simd_loop:
    movdqu  xmm0, [rdi]         ; โหลด 16 bytes ของ data
    pxor    xmm0, xmm7          ; XOR กับ key
    movdqu  [rsi], xmm0         ; เก็บผลลัพธ์
    
    add     rdi, 16
    add     rsi, 16
    dec     rax
    jnz     .xor_simd_loop
    
.xor_simd_remainder:
    ; จัดการ bytes ที่เหลือ
    mov     rcx, rdx
    and     rcx, 15
    jz      .xor_simd_done
    
    xor     r9, r9              ; key byte index
    mov     r10, [rel xor_simd_key_ptr]  ; pointer to key bytes
    
.xor_byte_loop:
    movzx   eax, byte [rdi]
    movzx   r8d, byte [r10 + r9]
    xor     eax, r8d
    mov     [rsi], al
    inc     rdi
    inc     rsi
    inc     r9
    and     r9, 15              ; key index mod 16
    dec     rcx
    jnz     .xor_byte_loop
    
.xor_simd_done:
    ret

;==============================================================================
; asm_crc32 - คำนวณ CRC32 checksum
; Input:  RDI = pointer ไปยัง data
;         RSI = data length
; Output: EAX = CRC32 value
; ใช้ CRC32 hardware instruction (SSE4.2)
;==============================================================================
asm_crc32:
    mov     eax, 0xFFFFFFFF     ; initial CRC value
    
    test    rsi, rsi
    jz      .crc_done
    
    mov     rcx, rsi
    
.crc_byte_loop:
    ; ใช้ CRC32 instruction (SSE4.2)
    ; หมายเหตุ: ต้องการ CPU ที่รองรับ SSE4.2
    crc32   eax, byte [rdi]
    inc     rdi
    dec     rcx
    jnz     .crc_byte_loop
    
.crc_done:
    not     eax                 ; final XOR
    ret

;==============================================================================
; asm_crc32_table - CRC32 ด้วย lookup table (compatible กับทุก CPU)
; Input:  RDI = data, RSI = length
; Output: EAX = CRC32
;==============================================================================
asm_crc32_table:
    mov     eax, 0xFFFFFFFF     ; initial value
    
    test    rsi, rsi
    jz      .crc_table_done
    
    lea     r8, [rel crc32_table]
    
.crc_table_loop:
    ; index = (crc XOR byte) AND 0xFF
    movzx   ecx, byte [rdi]
    xor     cl, al              ; cl = (crc ^ byte) & 0xFF
    movzx   ecx, cl
    
    ; lookup table และ shift
    shr     eax, 8
    xor     eax, [r8 + rcx * 4]
    
    inc     rdi
    dec     rsi
    jnz     .crc_table_loop
    
.crc_table_done:
    not     eax
    ret

;==============================================================================
; asm_sha256_transform - SHA-256 compression function core
; Input:  RDI = state array (8 uint32_t)
;         RSI = message block (64 bytes)
; Output: อัพเดท state ที่ [RDI]
; หมายเหตุ: นี่คือ inner loop ของ SHA-256 ที่ใช้เวลามากที่สุด
;==============================================================================
asm_sha256_transform:
    push    rbp
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; Allocate stack space สำหรับ message schedule W[64]
    sub     rsp, 256
    
    ; โหลด initial state
    ; a = state[0], b = state[1], ..., h = state[7]
    mov     eax, [rdi]          ; a
    mov     ebx, [rdi + 4]      ; b
    mov     ecx, [rdi + 8]      ; c
    mov     edx, [rdi + 12]     ; d
    mov     r8d, [rdi + 16]     ; e
    mov     r9d, [rdi + 20]     ; f
    mov     r10d, [rdi + 24]    ; g
    mov     r11d, [rdi + 28]    ; h
    
    ; เตรียม message schedule
    ; W[0..15] = โหลดจาก block (big-endian -> little-endian)
    xor     r12, r12
    
.prepare_w:
    cmp     r12, 16
    jge     .prepare_w_extend
    
    ; โหลด 4 bytes และ byte-swap (big-endian to little-endian)
    mov     r13d, [rsi + r12 * 4]
    bswap   r13d
    mov     [rsp + r12 * 4], r13d
    inc     r12
    jmp     .prepare_w
    
.prepare_w_extend:
    ; W[16..63] = sigma1(W[i-2]) + W[i-7] + sigma0(W[i-15]) + W[i-16]
    cmp     r12, 64
    jge     .sha256_rounds
    
    ; W[i-15]
    mov     r13d, [rsp + (r12-15) * 4]
    ; sigma0 = ROTR(x,7) XOR ROTR(x,18) XOR SHR(x,3)
    mov     r14d, r13d
    ror     r14d, 7
    mov     r15d, r13d
    ror     r15d, 18
    xor     r14d, r15d
    mov     r15d, r13d
    shr     r15d, 3
    xor     r14d, r15d          ; r14d = sigma0(W[i-15])
    
    ; W[i-2]
    mov     r13d, [rsp + (r12-2) * 4]
    ; sigma1 = ROTR(x,17) XOR ROTR(x,19) XOR SHR(x,10)
    mov     r15d, r13d
    ror     r15d, 17
    xor     r13d, r15d
    mov     r15d, [rsp + (r12-2) * 4]
    ror     r15d, 19
    xor     r13d, r15d
    mov     r15d, [rsp + (r12-2) * 4]
    shr     r15d, 10
    xor     r13d, r15d          ; r13d = sigma1(W[i-2])
    
    ; W[i] = sigma1 + W[i-7] + sigma0 + W[i-16]
    add     r13d, [rsp + (r12-7) * 4]
    add     r13d, r14d
    add     r13d, [rsp + (r12-16) * 4]
    mov     [rsp + r12 * 4], r13d
    
    inc     r12
    jmp     .prepare_w_extend
    
.sha256_rounds:
    ; ทำ 64 rounds ของ SHA-256
    xor     r12, r12
    lea     r13, [rel sha256_k]
    
.sha256_round_loop:
    cmp     r12, 64
    jge     .sha256_done
    
    ; T1 = h + Sigma1(e) + Ch(e,f,g) + K[i] + W[i]
    ; Sigma1(e) = ROTR(e,6) XOR ROTR(e,11) XOR ROTR(e,25)
    mov     ebp, r8d            ; e
    ror     ebp, 6
    mov     r14d, r8d
    ror     r14d, 11
    xor     ebp, r14d
    mov     r14d, r8d
    ror     r14d, 25
    xor     ebp, r14d           ; ebp = Sigma1(e)
    
    ; Ch(e,f,g) = (e AND f) XOR (NOT e AND g)
    mov     r14d, r8d           ; e
    and     r14d, r9d           ; e AND f
    mov     r15d, r8d           ; e
    not     r15d                ; NOT e
    and     r15d, r10d          ; NOT e AND g
    xor     r14d, r15d          ; Ch(e,f,g)
    
    ; T1 = h + Sigma1(e) + Ch(e,f,g) + K[i] + W[i]
    add     ebp, r11d           ; + h
    add     ebp, r14d           ; + Ch
    add     ebp, [r13 + r12*4]  ; + K[i]
    add     ebp, [rsp + r12*4]  ; + W[i]
    
    ; T2 = Sigma0(a) + Maj(a,b,c)
    ; Sigma0(a) = ROTR(a,2) XOR ROTR(a,13) XOR ROTR(a,22)
    mov     r14d, eax           ; a
    ror     r14d, 2
    mov     r15d, eax
    ror     r15d, 13
    xor     r14d, r15d
    mov     r15d, eax
    ror     r15d, 22
    xor     r14d, r15d          ; r14d = Sigma0(a)
    
    ; Maj(a,b,c) = (a AND b) XOR (a AND c) XOR (b AND c)
    mov     r15d, eax
    and     r15d, ebx           ; a AND b
    push    r15
    mov     r15d, eax
    and     r15d, ecx           ; a AND c
    xor     [rsp], r15d
    mov     r15d, ebx
    and     r15d, ecx           ; b AND c
    xor     r15d, [rsp]
    add     rsp, 8
    
    add     r14d, r15d          ; T2 = Sigma0(a) + Maj(a,b,c)
    
    ; อัพเดท state
    mov     r11d, r10d          ; h = g
    mov     r10d, r9d           ; g = f
    mov     r9d, r8d            ; f = e
    lea     r8d, [edx + ebp]    ; e = d + T1 (using lea for add)
    mov     edx, ecx            ; d = c
    mov     ecx, ebx            ; c = b
    mov     ebx, eax            ; b = a
    add     ebp, r14d           ; a = T1 + T2
    mov     eax, ebp
    
    inc     r12
    jmp     .sha256_round_loop
    
.sha256_done:
    ; เพิ่ม compressed chunk กลับเข้า state
    add     [rdi],      eax
    add     [rdi + 4],  ebx
    add     [rdi + 8],  ecx
    add     [rdi + 12], edx
    add     [rdi + 16], r8d
    add     [rdi + 20], r9d
    add     [rdi + 24], r10d
    add     [rdi + 28], r11d
    
    add     rsp, 256
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret

section .bss
xor_simd_key_ptr: resq 1
gray_weights_simd: resq 2
```

---

## 8. Testing Assembly Functions ด้วย C Test Harness

### 8.1 Test Framework

```c
// tests/test_string.c
// Test harness สำหรับ Assembly string functions
// ใช้ simple assertion framework

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>
#include "asm_funcs.h"

// Simple test framework
#define TEST_PASS  "\033[32mPASS\033[0m"
#define TEST_FAIL  "\033[31mFAIL\033[0m"

static int test_count = 0;
static int pass_count = 0;

// Macro สำหรับ testing
#define TEST(name, condition) do { \
    test_count++; \
    if (condition) { \
        printf("[%s] %s\n", TEST_PASS, name); \
        pass_count++; \
    } else { \
        printf("[%s] %s\n", TEST_FAIL, name); \
        printf("  Failed at %s:%d\n", __FILE__, __LINE__); \
    } \
} while(0)

// ทดสอบ asm_strlen
void test_strlen(void) {
    printf("\n=== Testing asm_strlen ===\n");
    
    // Test 1: string ปกติ
    TEST("strlen empty string",
         asm_strlen("") == 0);
    
    TEST("strlen single char",
         asm_strlen("a") == 1);
    
    TEST("strlen hello",
         asm_strlen("hello") == 5);
    
    TEST("strlen long string",
         asm_strlen("The quick brown fox jumps over the lazy dog") == 43);
    
    // Test 2: เปรียบเทียบกับ standard strlen
    const char *test_strings[] = {
        "",
        "a",
        "hello, world!",
        "This is a longer test string with some content",
        NULL
    };
    
    for (int i = 0; test_strings[i] != NULL; i++) {
        size_t asm_len = asm_strlen(test_strings[i]);
        size_t c_len = strlen(test_strings[i]);
        
        char test_name[100];
        snprintf(test_name, sizeof(test_name),
                 "strlen matches libc for: \"%s\"",
                 test_strings[i]);
        TEST(test_name, asm_len == c_len);
    }
    
    // Test 3: null pointer safety (ถ้า implementation รองรับ)
    // TEST("strlen null handling", asm_strlen(NULL) == 0);  // อย่า test นี้
}

// ทดสอบ asm_strcpy
void test_strcpy(void) {
    printf("\n=== Testing asm_strcpy ===\n");
    
    char dst[256];
    
    // Test 1: copy string ปกติ
    memset(dst, 0xFF, sizeof(dst));  // เติม dst ด้วยค่าทดสอบ
    asm_strcpy(dst, "hello");
    TEST("strcpy hello",
         strcmp(dst, "hello") == 0);
    
    // Test 2: copy empty string
    memset(dst, 0xFF, sizeof(dst));
    asm_strcpy(dst, "");
    TEST("strcpy empty string",
         dst[0] == '\0');
    
    // Test 3: copy long string
    const char *long_str = "This is a longer test string for assembly strcpy";
    memset(dst, 0, sizeof(dst));
    asm_strcpy(dst, long_str);
    TEST("strcpy long string",
         strcmp(dst, long_str) == 0);
    
    // Test 4: return value ต้องเป็น dst pointer
    memset(dst, 0, sizeof(dst));
    char *ret = asm_strcpy(dst, "test");
    TEST("strcpy returns dst pointer",
         ret == dst);
}

// ทดสอบ asm_memcpy
void test_memcpy(void) {
    printf("\n=== Testing asm_memcpy ===\n");
    
    uint8_t src[256], dst[256];
    
    // เตรียม test data
    for (int i = 0; i < 256; i++) {
        src[i] = (uint8_t)i;
    }
    
    // Test 1: copy 1 byte
    memset(dst, 0, sizeof(dst));
    asm_memcpy(dst, src, 1);
    TEST("memcpy 1 byte",
         dst[0] == 0 && dst[1] == 0);
    
    // Test 2: copy 16 bytes
    memset(dst, 0, sizeof(dst));
    asm_memcpy(dst, src, 16);
    TEST("memcpy 16 bytes",
         memcmp(dst, src, 16) == 0);
    
    // Test 3: copy 100 bytes
    memset(dst, 0, sizeof(dst));
    asm_memcpy(dst, src, 100);
    TEST("memcpy 100 bytes",
         memcmp(dst, src, 100) == 0);
    
    // Test 4: copy ขนาดเป็น multiple of 8
    memset(dst, 0, sizeof(dst));
    asm_memcpy(dst, src, 64);
    TEST("memcpy 64 bytes (8-byte aligned)",
         memcmp(dst, src, 64) == 0);
    
    // Test 5: copy 0 bytes
    memset(dst, 0xAA, sizeof(dst));
    asm_memcpy(dst, src, 0);
    TEST("memcpy 0 bytes",
         dst[0] == 0xAA);  // ไม่ควร modify dst
    
    // Test 6: return value
    memset(dst, 0, sizeof(dst));
    void *ret = asm_memcpy(dst, src, 10);
    TEST("memcpy returns dst",
         ret == (void*)dst);
}

// ทดสอบ asm_strcmp
void test_strcmp(void) {
    printf("\n=== Testing asm_strcmp ===\n");
    
    // Test 1: strings เท่ากัน
    TEST("strcmp equal strings",
         asm_strcmp("hello", "hello") == 0);
    
    TEST("strcmp empty strings",
         asm_strcmp("", "") == 0);
    
    // Test 2: strings ไม่เท่ากัน
    TEST("strcmp s1 < s2",
         asm_strcmp("abc", "abd") < 0);
    
    TEST("strcmp s1 > s2",
         asm_strcmp("abd", "abc") > 0);
    
    // Test 3: different lengths
    TEST("strcmp prefix",
         asm_strcmp("abc", "abcd") < 0);
    
    TEST("strcmp longer prefix",
         asm_strcmp("abcd", "abc") > 0);
    
    // Test 4: เปรียบเทียบกับ standard strcmp
    const char *pairs[][2] = {
        {"hello", "hello"},
        {"abc", "def"},
        {"", "a"},
        {"test", ""},
        {NULL, NULL}
    };
    
    for (int i = 0; pairs[i][0] != NULL; i++) {
        int asm_result = asm_strcmp(pairs[i][0], pairs[i][1]);
        int c_result = strcmp(pairs[i][0], pairs[i][1]);
        
        // ทั้งสอง result ต้องมี sign เดียวกัน
        int asm_sign = (asm_result > 0) - (asm_result < 0);
        int c_sign = (c_result > 0) - (c_result < 0);
        
        char test_name[100];
        snprintf(test_name, sizeof(test_name),
                 "strcmp sign matches: \"%s\" vs \"%s\"",
                 pairs[i][0], pairs[i][1]);
        TEST(test_name, asm_sign == c_sign);
    }
}

// Performance benchmark
void benchmark_string_functions(void) {
    printf("\n=== Performance Benchmark ===\n");
    
    const char *test_string = "Hello, World! This is a test string for benchmarking.";
    const size_t ITERATIONS = 10000000;
    
    // Benchmark asm_strlen vs strlen
    struct timespec start, end;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (size_t i = 0; i < ITERATIONS; i++) {
        volatile size_t len = asm_strlen(test_string);
        (void)len;
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double asm_time = (end.tv_sec - start.tv_sec) * 1e9 +
                      (end.tv_nsec - start.tv_nsec);
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (size_t i = 0; i < ITERATIONS; i++) {
        volatile size_t len = strlen(test_string);
        (void)len;
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double c_time = (end.tv_sec - start.tv_sec) * 1e9 +
                    (end.tv_nsec - start.tv_nsec);
    
    printf("strlen benchmark (%zu iterations):\n", ITERATIONS);
    printf("  asm_strlen: %.2f ms\n", asm_time / 1e6);
    printf("  libc strlen: %.2f ms\n", c_time / 1e6);
    printf("  Speedup: %.2fx\n", c_time / asm_time);
}

int main(void) {
    printf("Assembly String Functions Test Suite\n");
    printf("=====================================\n");
    
    test_strlen();
    test_strcpy();
    test_memcpy();
    test_strcmp();
    benchmark_string_functions();
    
    printf("\n=====================================\n");
    printf("Results: %d/%d tests passed\n", pass_count, test_count);
    
    return (pass_count == test_count) ? 0 : 1;
}
```

### 8.2 Test สำหรับ Math Functions

```c
// tests/test_math.c
// Test harness สำหรับ Assembly math functions

#include <stdio.h>
#include <math.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>
#include "asm_funcs.h"

#define EPSILON 1e-5f

// เปรียบเทียบ float values
static int float_eq(float a, float b) {
    return fabsf(a - b) < EPSILON;
}

// ทดสอบ dot product
void test_dot_product(void) {
    printf("\n=== Testing asm_dot_product ===\n");
    
    // Test 1: dot product ง่ายๆ
    float a[] = {1.0f, 2.0f, 3.0f};
    float b[] = {4.0f, 5.0f, 6.0f};
    float result = asm_dot_product(a, b, 3);
    // 1*4 + 2*5 + 3*6 = 4 + 10 + 18 = 32
    printf("[%s] dot product basic: got %.4f, expected 32.0\n",
           float_eq(result, 32.0f) ? "\033[32mPASS\033[0m" : "\033[31mFAIL\033[0m",
           result);
    
    // Test 2: dot product ยาว
    const int N = 1000;
    float *x = malloc(N * sizeof(float));
    float *y = malloc(N * sizeof(float));
    
    float expected = 0.0f;
    for (int i = 0; i < N; i++) {
        x[i] = (float)i;
        y[i] = 1.0f;
        expected += (float)i;
    }
    
    float asm_result = asm_dot_product(x, y, N);
    printf("[%s] dot product 1000 elements: got %.4f, expected %.4f\n",
           float_eq(asm_result, expected) ? "\033[32mPASS\033[0m" : "\033[31mFAIL\033[0m",
           asm_result, expected);
    
    free(x);
    free(y);
    
    // Test 3: zero vector
    float zeros[] = {0.0f, 0.0f, 0.0f};
    float ones[] = {1.0f, 2.0f, 3.0f};
    result = asm_dot_product(zeros, ones, 3);
    printf("[%s] dot product with zeros: got %.4f, expected 0.0\n",
           float_eq(result, 0.0f) ? "\033[32mPASS\033[0m" : "\033[31mFAIL\033[0m",
           result);
}

// ทดสอบ matrix multiply
void test_matrix_multiply(void) {
    printf("\n=== Testing asm_matrix_multiply_4x4 ===\n");
    
    // Identity matrix
    float I[16] = {
        1.0f, 0.0f, 0.0f, 0.0f,
        0.0f, 1.0f, 0.0f, 0.0f,
        0.0f, 0.0f, 1.0f, 0.0f,
        0.0f, 0.0f, 0.0f, 1.0f
    };
    
    float A[16] = {
        1.0f, 2.0f, 3.0f, 4.0f,
        5.0f, 6.0f, 7.0f, 8.0f,
        9.0f, 10.0f, 11.0f, 12.0f,
        13.0f, 14.0f, 15.0f, 16.0f
    };
    
    float result[16];
    
    // Test 1: A * I = A
    asm_matrix_multiply_4x4(A, I, result);
    int pass = 1;
    for (int i = 0; i < 16; i++) {
        if (!float_eq(result[i], A[i])) {
            pass = 0;
            break;
        }
    }
    printf("[%s] matrix * identity = matrix\n",
           pass ? "\033[32mPASS\033[0m" : "\033[31mFAIL\033[0m");
    
    // Test 2: I * A = A
    asm_matrix_multiply_4x4(I, A, result);
    pass = 1;
    for (int i = 0; i < 16; i++) {
        if (!float_eq(result[i], A[i])) {
            pass = 0;
            break;
        }
    }
    printf("[%s] identity * matrix = matrix\n",
           pass ? "\033[32mPASS\033[0m" : "\033[31mFAIL\033[0m");
}

// Benchmark
void benchmark_math(void) {
    printf("\n=== Math Performance Benchmark ===\n");
    
    const int N = 10000;
    const int ITER = 10000;
    
    float *a = malloc(N * sizeof(float));
    float *b = malloc(N * sizeof(float));
    
    for (int i = 0; i < N; i++) {
        a[i] = (float)rand() / RAND_MAX;
        b[i] = (float)rand() / RAND_MAX;
    }
    
    struct timespec start, end;
    
    // Benchmark asm dot product
    clock_gettime(CLOCK_MONOTONIC, &start);
    volatile float sum = 0;
    for (int i = 0; i < ITER; i++) {
        sum += asm_dot_product(a, b, N);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    (void)sum;
    
    double asm_time = (end.tv_sec - start.tv_sec) * 1e9 +
                      (end.tv_nsec - start.tv_nsec);
    
    // Benchmark C dot product
    clock_gettime(CLOCK_MONOTONIC, &start);
    sum = 0;
    for (int iter = 0; iter < ITER; iter++) {
        float s = 0;
        for (int i = 0; i < N; i++) {
            s += a[i] * b[i];
        }
        sum += s;
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double c_time = (end.tv_sec - start.tv_sec) * 1e9 +
                    (end.tv_nsec - start.tv_nsec);
    
    printf("dot product benchmark (N=%d, %d iterations):\n", N, ITER);
    printf("  asm_dot_product: %.2f ms\n", asm_time / 1e6);
    printf("  C dot_product: %.2f ms\n", c_time / 1e6);
    printf("  Speedup: %.2fx\n", c_time / asm_time);
    
    free(a);
    free(b);
}

int main(void) {
    srand(42);
    
    printf("Assembly Math Functions Test Suite\n");
    printf("====================================\n");
    
    test_dot_product();
    test_matrix_multiply();
    benchmark_math();
    
    return 0;
}
```

---

## 9. โปรเจกต์สมบูรณ์: Image Blur with SIMD

### 9.1 Main Application

```c
// src/image_blur_main.c
// โปรเจกต์สมบูรณ์: Image blur ด้วย SIMD Assembly
// รองรับ PPM image format (เรียบง่ายที่สุดสำหรับตัวอย่าง)

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "asm_funcs.h"

// โครงสร้างสำหรับเก็บ image data
typedef struct {
    int width;
    int height;
    uint8_t *data;     // RGB data (width * height * 3 bytes)
} Image;

// โหลด PPM image
Image* load_ppm(const char *filename) {
    FILE *f = fopen(filename, "rb");
    if (!f) {
        fprintf(stderr, "Cannot open file: %s\n", filename);
        return NULL;
    }
    
    Image *img = malloc(sizeof(Image));
    if (!img) {
        fclose(f);
        return NULL;
    }
    
    // อ่าน PPM header
    char magic[3];
    if (fscanf(f, "%2s", magic) != 1 || strcmp(magic, "P6") != 0) {
        fprintf(stderr, "Not a P6 PPM file\n");
        free(img);
        fclose(f);
        return NULL;
    }
    
    int maxval;
    if (fscanf(f, "%d %d %d", &img->width, &img->height, &maxval) != 3) {
        fprintf(stderr, "Invalid PPM header\n");
        free(img);
        fclose(f);
        return NULL;
    }
    fgetc(f);  // skip newline
    
    // โหลด pixel data
    size_t data_size = (size_t)img->width * img->height * 3;
    img->data = malloc(data_size);
    if (!img->data) {
        free(img);
        fclose(f);
        return NULL;
    }
    
    if (fread(img->data, 1, data_size, f) != data_size) {
        fprintf(stderr, "Error reading pixel data\n");
        free(img->data);
        free(img);
        fclose(f);
        return NULL;
    }
    
    fclose(f);
    return img;
}

// บันทึก grayscale image เป็น PGM
int save_pgm(const char *filename, const uint8_t *data, int width, int height) {
    FILE *f = fopen(filename, "wb");
    if (!f) {
        fprintf(stderr, "Cannot create file: %s\n", filename);
        return -1;
    }
    
    fprintf(f, "P5\n%d %d\n255\n", width, height);
    fwrite(data, 1, (size_t)width * height, f);
    fclose(f);
    return 0;
}

// ทำลาย image
void free_image(Image *img) {
    if (img) {
        free(img->data);
        free(img);
    }
}

// วัดเวลา
static double get_time_ms(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000.0 + ts.tv_nsec / 1e6;
}

// C version สำหรับเปรียบเทียบประสิทธิภาพ
void c_rgb_to_grayscale(const uint8_t *rgb, uint8_t *gray, size_t n) {
    for (size_t i = 0; i < n; i++) {
        gray[i] = (uint8_t)(
            (77 * rgb[i*3 + 0] +
            150 * rgb[i*3 + 1] +
             29 * rgb[i*3 + 2]) >> 8
        );
    }
}

void c_blur_filter(const uint8_t *input, uint8_t *output, int width, int height) {
    for (int row = 1; row < height - 1; row++) {
        for (int col = 1; col < width - 1; col++) {
            int sum = 0;
            for (int dr = -1; dr <= 1; dr++) {
                for (int dc = -1; dc <= 1; dc++) {
                    sum += input[(row + dr) * width + (col + dc)];
                }
            }
            output[row * width + col] = (uint8_t)(sum / 9);
        }
    }
    // Copy border pixels
    memcpy(output, input, width);
    memcpy(output + (height-1)*width, input + (height-1)*width, width);
    for (int r = 0; r < height; r++) {
        output[r * width] = input[r * width];
        output[r * width + width - 1] = input[r * width + width - 1];
    }
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: %s <input.ppm> [output.pgm]\n", argv[0]);
        printf("  Converts PPM to grayscale PGM with blur filter\n");
        printf("  Compares C vs Assembly performance\n");
        return 1;
    }
    
    const char *input_file = argv[1];
    const char *output_file = argc > 2 ? argv[2] : "output.pgm";
    
    // โหลด image
    printf("Loading image: %s\n", input_file);
    Image *img = load_ppm(input_file);
    if (!img) {
        return 1;
    }
    
    printf("Image size: %dx%d pixels\n", img->width, img->height);
    
    size_t pixel_count = (size_t)img->width * img->height;
    uint8_t *gray_asm  = malloc(pixel_count);
    uint8_t *gray_c    = malloc(pixel_count);
    uint8_t *blur_asm  = malloc(pixel_count);
    uint8_t *blur_c    = malloc(pixel_count);
    
    if (!gray_asm || !gray_c || !blur_asm || !blur_c) {
        fprintf(stderr, "Memory allocation failed\n");
        free_image(img);
        return 1;
    }
    
    // === Grayscale Conversion ===
    printf("\n--- Grayscale Conversion ---\n");
    
    // Assembly version
    double t_start = get_time_ms();
    asm_rgb_to_grayscale(img->data, gray_asm, pixel_count);
    double asm_gray_time = get_time_ms() - t_start;
    
    // C version
    t_start = get_time_ms();
    c_rgb_to_grayscale(img->data, gray_c, pixel_count);
    double c_gray_time = get_time_ms() - t_start;
    
    printf("Assembly: %.3f ms\n", asm_gray_time);
    printf("C:        %.3f ms\n", c_gray_time);
    printf("Speedup:  %.2fx\n", c_gray_time / asm_gray_time);
    
    // ตรวจสอบความถูกต้อง
    int gray_match = memcmp(gray_asm, gray_c, pixel_count) == 0;
    printf("Results match: %s\n", gray_match ? "YES" : "NO");
    
    // === Blur Filter ===
    printf("\n--- Blur Filter ---\n");
    
    // Assembly version
    t_start = get_time_ms();
    asm_blur_filter(gray_asm, blur_asm, img->width, img->height);
    double asm_blur_time = get_time_ms() - t_start;
    
    // C version
    t_start = get_time_ms();
    c_blur_filter(gray_c, blur_c, img->width, img->height);
    double c_blur_time = get_time_ms() - t_start;
    
    printf("Assembly: %.3f ms\n", asm_blur_time);
    printf("C:        %.3f ms\n", c_blur_time);
    printf("Speedup:  %.2fx\n", c_blur_time / asm_blur_time);
    
    // บันทึกผลลัพธ์
    printf("\nSaving output: %s\n", output_file);
    save_pgm(output_file, blur_asm, img->width, img->height);
    
    // ทำลาย memory
    free(gray_asm);
    free(gray_c);
    free(blur_asm);
    free(blur_c);
    free_image(img);
    
    printf("Done!\n");
    return 0;
}
```

### 9.2 คำสั่ง Build และ Run

```bash
# Build โปรเจกต์ทั้งหมด
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4

# รัน image blur
./image_blur_app sample.ppm output.pgm

# รัน tests
./test_string
./test_math

# Build ด้วย manual Makefile
make all

# Build สำหรับ debug
cmake .. -DCMAKE_BUILD_TYPE=Debug
make -j4
./test_string    # รัน tests ใน debug mode

# ดู assembly output
objdump -d -M intel src/asm/string_ops.o | head -100

# ตรวจสอบ symbols
nm libasm_lib.a

# Benchmark พร้อม perf
perf stat ./benchmark
perf record -g ./benchmark
perf report --stdio
```

---

## 10. Game Engine Physics Calculation

### 10.1 SIMD Physics Calculations

```nasm
; src/asm/physics.asm
; Game engine physics calculations ด้วย SIMD
; ใช้สำหรับ particle systems, collision detection

section .data
align 16
gravity_vec: dd -9.81, 0.0, 0.0, 0.0  ; gravity vector (y-axis down)

section .text
global asm_update_particles
global asm_check_aabb_collision
global asm_compute_forces

;==============================================================================
; asm_update_particles - อัพเดท particle positions และ velocities
; โครงสร้าง Particle: [x, y, z, vx, vy, vz, ax, ay, az, mass] (10 floats = 40 bytes)
; Input:  RDI = pointer ไปยัง particle array
;         RSI = จำนวน particles
;         XMM0 = delta time (dt)
; Output: ไม่มี (แก้ไข array ใน place)
;==============================================================================
asm_update_particles:
    push    rbp
    push    rbx
    
    ; dt ใน XMM0 (scalar)
    ; Broadcast dt ไปยัง XMM1 (ทุก lanes)
    shufps  xmm0, xmm0, 0       ; XMM0 = [dt, dt, dt, dt]
    
    mov     rcx, rsi            ; loop counter
    test    rcx, rcx
    jz      .particles_done
    
.particle_loop:
    ; โหลด position (x, y, z)
    movups  xmm1, [rdi]         ; XMM1 = [x, y, z, vx]
    movups  xmm2, [rdi + 16]    ; XMM2 = [vy, vz, ax, ay]
    
    ; velocity: v' = v + a * dt
    ; acceleration components: ax, ay, az
    movups  xmm3, [rdi + 24]    ; XMM3 = [vz, ax, ay, az]
    
    ; ใช้ method แบบง่าย (Euler integration)
    ; position update: p' = p + v * dt
    
    ; โหลด velocity (vx, vy, vz)
    movss   xmm4, [rdi + 12]    ; vx
    movss   xmm5, [rdi + 16]    ; vy
    movss   xmm6, [rdi + 20]    ; vz
    
    ; โหลด acceleration (ax, ay, az)
    movss   xmm7, [rdi + 24]    ; ax
    movss   xmm8, [rdi + 28]    ; ay
    movss   xmm9, [rdi + 32]    ; az
    
    ; Update velocity: v' = v + a * dt
    mulss   xmm7, xmm0          ; ax * dt
    mulss   xmm8, xmm0          ; ay * dt
    mulss   xmm9, xmm0          ; az * dt
    
    addss   xmm4, xmm7          ; vx' = vx + ax*dt
    addss   xmm5, xmm8          ; vy' = vy + ay*dt
    addss   xmm6, xmm9          ; vz' = vz + az*dt
    
    ; Update position: p' = p + v' * dt
    movss   xmm10, xmm4
    movss   xmm11, xmm5
    movss   xmm12, xmm6
    
    mulss   xmm10, xmm0         ; vx' * dt
    mulss   xmm11, xmm0         ; vy' * dt
    mulss   xmm12, xmm0         ; vz' * dt
    
    addss   [rdi],      xmm10   ; x' = x + vx'*dt
    addss   [rdi + 4],  xmm11   ; y' = y + vy'*dt
    addss   [rdi + 8],  xmm12   ; z' = z + vz'*dt
    
    ; เก็บ velocity ใหม่
    movss   [rdi + 12], xmm4    ; vx'
    movss   [rdi + 16], xmm5    ; vy'
    movss   [rdi + 20], xmm6    ; vz'
    
    ; เลื่อนไปยัง particle ถัดไป (40 bytes per particle)
    add     rdi, 40
    dec     rcx
    jnz     .particle_loop
    
.particles_done:
    pop     rbx
    pop     rbp
    ret

;==============================================================================
; asm_check_aabb_collision - ตรวจสอบ Axis-Aligned Bounding Box collision
; Input:  RDI = box1 [min_x, min_y, min_z, max_x, max_y, max_z]
;         RSI = box2 [min_x, min_y, min_z, max_x, max_y, max_z]
; Output: EAX = 1 ถ้า collision, 0 ถ้าไม่มี
; ใช้ SSE comparison สำหรับ 3 axes พร้อมกัน
;==============================================================================
asm_check_aabb_collision:
    ; โหลด box1 min/max
    movups  xmm0, [rdi]         ; XMM0 = [min_x1, min_y1, min_z1, max_x1]
    movups  xmm1, [rdi + 12]    ; XMM1 = [max_x1, max_y1, max_z1, ...]
    
    ; โหลด box2 min/max
    movups  xmm2, [rsi]         ; XMM2 = [min_x2, min_y2, min_z2, max_x2]
    movups  xmm3, [rsi + 12]    ; XMM3 = [max_x2, max_y2, max_z2, ...]
    
    ; ตรวจสอบ overlap ใน 3 axes
    ; Collision ถ้า: max1 >= min2 AND max2 >= min1
    
    ; max1 >= min2
    movups  xmm4, xmm1          ; max1
    cmpps   xmm4, xmm2, 5       ; xmm4[i] = (max1[i] >= min2[i]) ? 0xFFFFFFFF : 0
    
    ; max2 >= min1
    movups  xmm5, xmm3          ; max2
    cmpps   xmm5, xmm0, 5       ; xmm5[i] = (max2[i] >= min1[i]) ? 0xFFFFFFFF : 0
    
    ; AND ทั้งสอง conditions
    andps   xmm4, xmm5
    
    ; ตรวจสอบ 3 axes (x, y, z)
    movmskps eax, xmm4          ; EAX = bitmask ของ comparison results
    and     eax, 0x7            ; mask 3 axes (bits 0, 1, 2)
    cmp     eax, 0x7            ; ต้องผ่านทั้ง 3 axes
    sete    al                  ; AL = 1 ถ้า collision
    movzx   eax, al
    
    ret
```

---

## 11. Debugging Mixed C/Assembly Projects

### 11.1 การใช้ GDB กับ Mixed Code

```bash
# Compile ด้วย debug information
# สำหรับ C files
gcc -g3 -O0 -c src/main.c -o obj/main.o

# สำหรับ Assembly (NASM ต้องใส่ -g -F dwarf)
nasm -f elf64 -g -F dwarf src/asm/string_ops.asm -o obj/string_ops.o

# Link
gcc -g obj/main.o obj/string_ops.o -o debug_app

# เริ่ม GDB
gdb ./debug_app

# GDB commands ที่มีประโยชน์
(gdb) break asm_strlen          # breakpoint ที่ Assembly function
(gdb) break main.c:25           # breakpoint ที่ C code
(gdb) run                       # เริ่มโปรแกรม
(gdb) stepi                     # execute 1 instruction
(gdb) nexti                     # execute instruction (ข้าม calls)
(gdb) info registers            # แสดง registers ทั้งหมด
(gdb) x/16xb $rdi               # แสดง 16 bytes ที่ RDI
(gdb) disassemble               # disassemble current function
(gdb) print $rax                # print register value
(gdb) display $xmm0             # แสดง XMM0 ทุก step
(gdb) layout asm                # แสดง assembly view
(gdb) layout reg                # แสดง register view
```

### 11.2 Valgrind Memory Check

```bash
# ตรวจสอบ memory errors
valgrind --tool=memcheck \
         --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         ./debug_app

# Profiling ด้วย callgrind
valgrind --tool=callgrind \
         --callgrind-out-file=callgrind.out \
         ./main_app

# วิเคราะห์ผลลัพธ์
kcachegrind callgrind.out    # GUI
callgrind_annotate callgrind.out src/asm/string_ops.asm

# Address Sanitizer (เร็วกว่า valgrind)
gcc -fsanitize=address -g -o asan_app src/main.c obj/string_ops.o
./asan_app
```

---

## 12. แบบฝึกหัด (Exercises)

### Exercise 1: String Functions
```
สร้างฟังก์ชัน Assembly ต่อไปนี้:
1. asm_strcat - ต่อ string สอง string เข้าด้วยกัน
2. asm_strncpy - คัดลอก string ไม่เกิน n characters
3. asm_memset - เติม memory ด้วย byte value

ต้องทำ:
- เขียน Assembly function ด้วย NASM
- เขียน C test harness
- Benchmark เทียบกับ libc
```

### Exercise 2: SIMD Optimization
```
Optimize ฟังก์ชันต่อไปนี้ด้วย SSE/AVX:
1. asm_vector_normalize - normalize float vector
2. asm_find_max - หาค่าสูงสุดใน float array
3. asm_threshold - กรอง float array (ถ้า < threshold = 0)

ข้อกำหนด:
- ใช้ SSE2 (ทำงานได้บน CPU รุ่นเก่า)
- Process 4 elements ต่อ instruction
- ต้องผ่าน unit tests ทั้งหมด
```

### Exercise 3: CRC32 Table
```
สร้าง CRC32 implementation ที่สมบูรณ์:
1. สร้าง full 256-entry lookup table ใน Assembly
2. ใช้ table-based CRC32 computation
3. Verify ผลลัพธ์กับ known CRC32 values:
   - CRC32("") = 0x00000000
   - CRC32("123456789") = 0xCBF43926
```

### Exercise 4: Image Processing Pipeline
```
สร้าง image processing pipeline:
1. Load grayscale PGM image
2. Apply Gaussian blur (5x5 kernel)
3. Apply edge detection (Sobel operator)
4. Save result

ทุกขั้นตอนต้องมี:
- C reference implementation
- Assembly optimized version
- Performance comparison
```

### Exercise 5: โปรเจกต์ขั้นสูง
```
สร้าง simple audio DSP library:
1. asm_fft_butterfly - FFT butterfly operation
2. asm_apply_window - apply Hanning window function
3. asm_mix_channels - mix multiple audio channels

ใช้ SSE สำหรับ float processing
เขียน test ที่ verify correctness ด้วย known inputs
```

---

## สรุป (Summary)

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Project Structure | โครงสร้างไดเรกทอรีมาตรฐาน, CMake, Makefile |
| Build System | CMake configuration, NASM flags, linking |
| Profiling | gprof, perf, callgrind |
| String Ops | strlen (REPNE SCASB), strcpy, memcpy (REP MOVSQ) |
| Math Ops | Dot product (SSE), matrix multiply (4x4) |
| Image Processing | RGB to grayscale, blur filter |
| Cryptography | XOR cipher, CRC32, SHA-256 core |
| Testing | C test harness, assertions, benchmarks |
| Physics | Particle system, AABB collision |
| Debugging | GDB, Valgrind, AddressSanitizer |

### Key Takeaways:

1. **Inline Assembly เทียบกับ External Assembly**: External Assembly files (`.asm`) ให้ความยืดหยุ่นและ maintainability มากกว่า

2. **ABI Compliance**: ต้องเคารพ System V AMD64 ABI (RDI, RSI, RDX, RCX, R8, R9 สำหรับ arguments)

3. **SIMD หลาย version**: SSE2 รองรับ CPU รุ่นเก่า, AVX2 เร็วกว่า 2x, แต่ต้องการ CPU ใหม่กว่า

4. **Testing สำคัญมาก**: ทุก Assembly function ต้องมี C test harness

5. **Profiling ก่อน Optimize**: อย่า optimize blindly - ใช้ profiler เพื่อหา hotspot ที่แท้จริง

---

## คำสั่ง Quick Reference

```bash
# Build โปรเจกต์
cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j4

# รัน tests
cd build && ctest --verbose

# Profile
perf stat ./build/main_app
valgrind --tool=callgrind ./build/main_app

# Debug Assembly
gdb ./build/debug_app
(gdb) layout asm
(gdb) break asm_strlen
(gdb) run

# Check assembly output
objdump -d -M intel build/libasm_lib.a
```

---

**ส่วนต่อไป**: Part 031 - Advanced SIMD: AVX-512 และ Vector Intrinsics
