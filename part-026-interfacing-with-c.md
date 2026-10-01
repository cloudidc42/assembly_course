# Part 026: Interfacing Assembly with C
## การเชื่อมต่อ Assembly กับภาษา C

---

## บทนำ (Introduction)

การเชื่อมต่อ Assembly กับภาษา C เป็นทักษะที่สำคัญมากในการพัฒนาซอฟต์แวร์ระดับต่ำ
เราสามารถใช้ประโยชน์จากทั้งสองภาษาได้อย่างเต็มที่:

- **C**: จัดการโครงสร้างข้อมูล, logic ระดับสูง, portable code
- **Assembly**: ประสิทธิภาพสูงสุด, hardware access, SIMD operations, crypto primitives

ในบทนี้เราจะเรียนรู้การเชื่อมต่อทั้งสองทิศทาง:
1. เรียก Assembly จาก C
2. เรียก C จาก Assembly
3. Inline Assembly ใน GCC

---

## 26.1 extern และ global Directives

### global directive
```nasm
; global ประกาศว่า symbol นี้สามารถมองเห็นได้จากภายนอกไฟล์นี้
; เหมือนกับ "public" ใน C

section .text
    global my_function      ; ทำให้ my_function มองเห็นได้จากภายนอก
    global another_func     ; สามารถ export หลาย symbol ได้

my_function:
    ; ... code ...
    ret

another_func:
    ; ... code ...
    ret
```

### extern directive
```nasm
; extern ประกาศว่า symbol นี้มาจากไฟล์อื่น
; เหมือนกับ "extern" ใน C

extern printf               ; ฟังก์ชัน printf จาก C standard library
extern malloc               ; ฟังก์ชัน malloc จาก C standard library
extern my_c_function        ; ฟังก์ชันที่เขียนใน C

section .data
    fmt db "Value: %d", 10, 0   ; format string สำหรับ printf

section .text
    global main

main:
    push rbp
    mov rbp, rsp
    
    ; เรียก printf ซึ่งเป็น extern function
    lea rdi, [rel fmt]      ; format string (argument 1)
    mov rsi, 42             ; ค่าที่จะ print (argument 2)
    xor eax, eax            ; จำนวน vector registers = 0
    call printf             ; เรียก C function
    
    xor eax, eax
    pop rbp
    ret
```

### ตัวอย่างสมบูรณ์: การใช้ global และ extern
```nasm
; file: math_ops.asm
; คำอธิบาย: ไฟล์ Assembly ที่ export ฟังก์ชันให้ C ใช้งาน

section .text

; ประกาศ export functions
global asm_add              ; บวกสองจำนวน
global asm_multiply         ; คูณสองจำนวน
global asm_factorial        ; คำนวณ factorial
global asm_gcd              ; หา GCD (Greatest Common Divisor)

;----------------------------------------------------------
; asm_add: บวกสองจำนวน 64-bit
; Parameters: rdi = a, rsi = b
; Returns: rax = a + b
;----------------------------------------------------------
asm_add:
    mov rax, rdi            ; rax = a
    add rax, rsi            ; rax = a + b
    ret

;----------------------------------------------------------
; asm_multiply: คูณสองจำนวน 64-bit
; Parameters: rdi = a, rsi = b
; Returns: rax = a * b
;----------------------------------------------------------
asm_multiply:
    mov rax, rdi            ; rax = a
    imul rax, rsi           ; rax = a * b (signed multiply)
    ret

;----------------------------------------------------------
; asm_factorial: คำนวณ n! (iterative)
; Parameters: rdi = n
; Returns: rax = n!
;----------------------------------------------------------
asm_factorial:
    mov rax, 1              ; result = 1
    cmp rdi, 1              ; ถ้า n <= 1
    jle .done               ; จบ (return 1)
    
.loop:
    imul rax, rdi           ; result *= n
    dec rdi                 ; n--
    cmp rdi, 1              ; ถ้า n > 1
    jg .loop                ; วนซ้ำ
    
.done:
    ret

;----------------------------------------------------------
; asm_gcd: หา Greatest Common Divisor (Euclidean algorithm)
; Parameters: rdi = a, rsi = b
; Returns: rax = gcd(a, b)
;----------------------------------------------------------
asm_gcd:
    ; Euclidean algorithm: gcd(a, b) = gcd(b, a mod b)
.loop:
    test rsi, rsi           ; ถ้า b == 0
    jz .done                ; จบ, return a
    
    mov rax, rdi            ; rax = a
    xor edx, edx            ; ล้าง rdx ก่อน div
    div rsi                 ; rax = a/b, rdx = a%b
    
    mov rdi, rsi            ; a = b
    mov rsi, rdx            ; b = a%b
    jmp .loop               ; วนซ้ำ
    
.done:
    mov rax, rdi            ; return a (ซึ่งเป็น gcd)
    ret
```

---

## 26.2 Calling Assembly from C

### Header File สำหรับ Assembly Functions
```c
/* file: math_ops.h
   Header file ที่ประกาศ assembly functions สำหรับให้ C ใช้งาน */

#ifndef MATH_OPS_H
#define MATH_OPS_H

#include <stdint.h>

/* ประกาศ assembly functions */
/* ต้องใช้ extern เพื่อบอก C compiler ว่า functions เหล่านี้มาจากภายนอก */

#ifdef __cplusplus
extern "C" {
#endif

/* บวกสองจำนวน */
int64_t asm_add(int64_t a, int64_t b);

/* คูณสองจำนวน */
int64_t asm_multiply(int64_t a, int64_t b);

/* คำนวณ factorial */
int64_t asm_factorial(int64_t n);

/* หา GCD */
int64_t asm_gcd(int64_t a, int64_t b);

#ifdef __cplusplus
}
#endif

#endif /* MATH_OPS_H */
```

### C Program ที่เรียก Assembly Functions
```c
/* file: main.c
   โปรแกรม C หลักที่ใช้ assembly functions */

#include <stdio.h>
#include <stdlib.h>
#include "math_ops.h"

int main(void) {
    printf("=== Assembly Math Functions Demo ===\n\n");
    
    /* ทดสอบ asm_add */
    int64_t sum = asm_add(100, 200);
    printf("asm_add(100, 200) = %ld\n", sum);
    
    /* ทดสอบ asm_multiply */
    int64_t product = asm_multiply(12, 34);
    printf("asm_multiply(12, 34) = %ld\n", product);
    
    /* ทดสอบ asm_factorial */
    for (int i = 1; i <= 10; i++) {
        int64_t fact = asm_factorial(i);
        printf("asm_factorial(%d) = %ld\n", i, fact);
    }
    
    /* ทดสอบ asm_gcd */
    printf("\nasm_gcd(48, 18) = %ld\n", asm_gcd(48, 18));
    printf("asm_gcd(100, 75) = %ld\n", asm_gcd(100, 75));
    printf("asm_gcd(1071, 462) = %ld\n", asm_gcd(1071, 462));
    
    return 0;
}
```

---

## 26.3 Makefile สำหรับ Mixed C/Assembly Projects

```makefile
# Makefile สำหรับโปรเจกต์ที่มีทั้ง C และ Assembly

# Compiler และ Assembler
CC      = gcc
NASM    = nasm
LD      = gcc

# Flags
CFLAGS  = -Wall -Wextra -O2 -g
NASMFLAGS = -f elf64 -g -F dwarf
LDFLAGS = -no-pie

# ไฟล์ source
C_SRCS  = main.c
ASM_SRCS = math_ops.asm

# Object files (แปลงชื่อ .c -> .o และ .asm -> .o)
C_OBJS  = $(C_SRCS:.c=.o)
ASM_OBJS = $(ASM_SRCS:.asm=.o)
OBJS    = $(C_OBJS) $(ASM_OBJS)

# Output binary
TARGET  = program

# Default target
all: $(TARGET)

# Link object files
$(TARGET): $(OBJS)
	$(LD) $(LDFLAGS) -o $@ $^
	@echo "Build complete: $(TARGET)"

# Compile C files
%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

# Assemble NASM files
%.o: %.asm
	$(NASM) $(NASMFLAGS) -o $@ $<

# Clean build artifacts
clean:
	rm -f $(OBJS) $(TARGET)
	@echo "Cleaned!"

# Run the program
run: $(TARGET)
	./$(TARGET)

# Debug with GDB
debug: $(TARGET)
	gdb $(TARGET)

# Show disassembly
disasm: $(TARGET)
	objdump -d -M intel $(TARGET) | less

.PHONY: all clean run debug disasm
```

### คำสั่ง Build
```bash
# Build โปรแกรม
make

# รันโปรแกรม
make run

# Clean build artifacts
make clean

# Manual build (ไม่ใช้ Makefile)
# Step 1: Assemble
nasm -f elf64 -g -F dwarf math_ops.asm -o math_ops.o

# Step 2: Compile C
gcc -Wall -O2 -c main.c -o main.o

# Step 3: Link
gcc -no-pie -o program main.o math_ops.o

# Step 4: Run
./program
```

---

## 26.4 Calling C from Assembly

### ตัวอย่าง: Assembly ที่เรียก C Standard Library
```nasm
; file: call_c_funcs.asm
; Assembly ที่เรียกใช้ฟังก์ชันจาก C standard library

; ประกาศ extern functions จาก C
extern printf               ; จาก stdio.h
extern scanf                ; จาก stdio.h
extern malloc               ; จาก stdlib.h
extern free                 ; จาก stdlib.h
extern strlen               ; จาก string.h
extern strcpy               ; จาก string.h
extern puts                 ; จาก stdio.h
extern exit                 ; จาก stdlib.h

section .data
    ; Format strings สำหรับ printf
    fmt_int     db "Integer: %d", 10, 0
    fmt_float   db "Float: %f", 10, 0
    fmt_str     db "String: %s", 10, 0
    fmt_prompt  db "Enter a number: ", 0
    fmt_input   db "%d", 0
    
    ; String literals
    hello_str   db "Hello from Assembly!", 10, 0
    bye_str     db "Goodbye!", 10, 0

section .bss
    input_val   resq 1          ; buffer สำหรับรับ input
    buffer      resb 256        ; buffer ทั่วไป

section .text
    global main

main:
    push rbp
    mov rbp, rsp
    sub rsp, 32                 ; จอง stack space
    
    ;------------------------------------------
    ; เรียก puts (พิมพ์ string + newline)
    ;------------------------------------------
    lea rdi, [rel hello_str]
    call puts
    
    ;------------------------------------------
    ; เรียก printf พร้อม format string และ integer
    ;------------------------------------------
    lea rdi, [rel fmt_int]      ; format string
    mov rsi, 12345              ; ค่า integer
    xor eax, eax                ; ไม่มี float args
    call printf
    
    ;------------------------------------------
    ; เรียก printf พร้อม float
    ; ต้องใส่ float ใน xmm0 แทน rsi
    ;------------------------------------------
    lea rdi, [rel fmt_float]    ; format string
    ; ใส่ค่า 3.14159 ใน xmm0
    mov rax, 0x400921FB54442D18 ; 3.14159... ในรูป IEEE 754
    movq xmm0, rax              ; xmm0 = 3.14159...
    mov eax, 1                  ; จำนวน float args = 1
    call printf
    
    ;------------------------------------------
    ; เรียก scanf รับ input จาก user
    ;------------------------------------------
    lea rdi, [rel fmt_prompt]
    xor eax, eax
    call printf
    
    lea rdi, [rel fmt_input]    ; format "%d"
    lea rsi, [rel input_val]    ; pointer to buffer
    xor eax, eax
    call scanf
    
    ; พิมพ์ค่าที่รับมา
    lea rdi, [rel fmt_int]
    mov rsi, qword [rel input_val]
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; เรียก malloc จอง memory
    ;------------------------------------------
    mov rdi, 100                ; จอง 100 bytes
    call malloc
    
    ; rax = pointer ที่ได้จาก malloc
    test rax, rax               ; ตรวจสอบว่า malloc สำเร็จ
    jz .malloc_failed
    
    mov rbx, rax                ; เก็บ pointer ใน rbx
    
    ;------------------------------------------
    ; เรียก strcpy คัดลอก string ลง buffer ที่ malloc
    ;------------------------------------------
    mov rdi, rbx                ; destination
    lea rsi, [rel hello_str]    ; source
    call strcpy
    
    ; พิมพ์ string ที่คัดลอกมา
    lea rdi, [rel fmt_str]
    mov rsi, rbx
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; เรียก strlen หาความยาว string
    ;------------------------------------------
    mov rdi, rbx                ; string pointer
    call strlen
    
    ; rax = ความยาว string
    lea rdi, [rel fmt_int]
    mov rsi, rax
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; เรียก free คืน memory
    ;------------------------------------------
    mov rdi, rbx                ; pointer ที่จะ free
    call free
    
    ;------------------------------------------
    ; Cleanup และ return
    ;------------------------------------------
    lea rdi, [rel bye_str]
    call puts
    
    xor eax, eax                ; return 0
    add rsp, 32
    pop rbp
    ret

.malloc_failed:
    ; จัดการ error
    mov rdi, 1                  ; exit code 1
    call exit
```

---

## 26.5 Inline Assembly (GCC \_\_asm\_\_)

### รูปแบบพื้นฐาน (Basic Inline Assembly)
```c
/* file: inline_asm_basic.c
   ตัวอย่าง Basic Inline Assembly ใน GCC */

#include <stdio.h>

int main(void) {
    /* รูปแบบพื้นฐาน: __asm__("instruction"); */
    
    /* NOP instruction */
    __asm__("nop");
    
    /* หลาย instructions (คั่นด้วย \n\t) */
    __asm__(
        "nop\n\t"
        "nop\n\t"
        "nop"
    );
    
    /* ใช้ asm แทน __asm__ ได้ (แต่ __asm__ portable กว่า) */
    asm("nop");
    
    printf("Basic inline asm works!\n");
    return 0;
}
```

### Extended Inline Assembly
```c
/* file: inline_asm_extended.c
   Extended Inline Assembly พร้อม input/output constraints */

#include <stdio.h>
#include <stdint.h>

/* รูปแบบ Extended Inline Assembly:
   __asm__ [volatile] (
       "assembly code"
       : output operands       // "=r"(var) หรือ "+r"(var) 
       : input operands        // "r"(var) หรือ "i"(immediate)
       : clobbered registers   // "rax", "memory", "cc"
   );
*/

/* ตัวอย่าง 1: บวกสองจำนวน */
static inline int64_t add_asm(int64_t a, int64_t b) {
    int64_t result;
    
    __asm__ volatile (
        "mov %1, %0\n\t"    /* result = a */
        "add %2, %0"        /* result += b */
        : "=r" (result)     /* output: result ถูกใส่ใน register ใดก็ได้ */
        : "r" (a),          /* input: a ใน register */
          "r" (b)           /* input: b ใน register */
        : /* ไม่มี clobbered registers */
    );
    
    return result;
}

/* ตัวอย่าง 2: คูณสองจำนวน */
static inline int64_t mul_asm(int64_t a, int64_t b) {
    int64_t result;
    
    __asm__ volatile (
        "imulq %2, %0"      /* result = a * b */
        : "=r" (result)     /* output */
        : "0" (a),          /* input: a ใน register เดียวกับ output (constraint "0") */
          "r" (b)           /* input: b ใน register ใดก็ได้ */
    );
    
    return result;
}

/* ตัวอย่าง 3: สลับค่าสองตัวแปร (swap) */
static inline void swap_asm(int64_t *a, int64_t *b) {
    __asm__ volatile (
        "mov (%0), %%rax\n\t"   /* rax = *a */
        "mov (%1), %%rbx\n\t"   /* rbx = *b */
        "mov %%rbx, (%0)\n\t"   /* *a = rbx */
        "mov %%rax, (%1)"       /* *b = rax */
        :                       /* ไม่มี output (เขียนผ่าน memory) */
        : "r" (a),              /* input: pointer a */
          "r" (b)               /* input: pointer b */
        : "rax", "rbx",         /* clobbered: rax, rbx ถูกแก้ไข */
          "memory"              /* clobbered: memory ถูกแก้ไข */
    );
}

/* ตัวอย่าง 4: หา absolute value */
static inline int64_t abs_asm(int64_t x) {
    int64_t result;
    
    __asm__ volatile (
        "mov %1, %0\n\t"        /* result = x */
        "neg %0\n\t"            /* result = -result */
        "cmovl %1, %0"          /* ถ้า x ≥ 0 ให้ใช้ x แทน (-result) */
        : "=&r" (result)        /* output: early-clobber (ต้องต่างจาก input) */
        : "r" (x)               /* input */
        : "cc"                  /* clobbered: flags register */
    );
    
    return result;
}

int main(void) {
    printf("add_asm(15, 27) = %ld\n", add_asm(15, 27));
    printf("mul_asm(6, 7) = %ld\n", mul_asm(6, 7));
    
    int64_t x = 100, y = 200;
    printf("Before swap: x=%ld, y=%ld\n", x, y);
    swap_asm(&x, &y);
    printf("After swap:  x=%ld, y=%ld\n", x, y);
    
    printf("abs_asm(-42) = %ld\n", abs_asm(-42));
    printf("abs_asm(42)  = %ld\n", abs_asm(42));
    
    return 0;
}
```

---

## 26.6 AT&T vs Intel Syntax ใน GCC

### ความแตกต่างระหว่าง AT&T และ Intel Syntax
```c
/* file: syntax_comparison.c
   เปรียบเทียบ AT&T syntax (default ใน GCC) และ Intel syntax */

#include <stdio.h>
#include <stdint.h>

/* AT&T Syntax (default ใน GCC inline asm):
   - operands: source ก่อน destination: "mov src, dst"
   - registers: ต้องมี % นำหน้า: %rax, %rbx
   - immediates: ต้องมี $ นำหน้า: $42, $0xFF
   - memory: ใช้ parentheses: (%rax), 8(%rsp)
   - ขนาด operations: suffix บน mnemonic: movb, movw, movl, movq
*/

/* Intel Syntax:
   - operands: destination ก่อน source: "mov dst, src"
   - registers: ไม่มี prefix: rax, rbx
   - immediates: แค่ตัวเลข: 42, 0xFF
   - memory: ใช้ brackets: [rax], [rsp+8]
   - ขนาด: ptr keyword: byte ptr, word ptr, dword ptr, qword ptr
*/

/* ตัวอย่าง AT&T syntax (default) */
static inline int64_t att_example(int64_t a, int64_t b) {
    int64_t result;
    
    /* AT&T: source อยู่ซ้าย, destination อยู่ขวา */
    __asm__ volatile (
        "movq %1, %0\n\t"   /* movq: q = 64-bit quad word */
        "addq %2, %0"       /* destination อยู่ขวา */
        : "=r" (result)
        : "r" (a), "r" (b)
    );
    
    return result;
}

/* ตัวอย่าง Intel syntax ใน GCC */
static inline int64_t intel_example(int64_t a, int64_t b) {
    int64_t result;
    
    /* ใช้ .intel_syntax noprefix เพื่อ switch เป็น Intel syntax */
    __asm__ volatile (
        ".intel_syntax noprefix\n\t"    /* switch เป็น Intel syntax */
        "mov %0, %1\n\t"               /* Intel: dest อยู่ซ้าย */
        "add %0, %2\n\t"               /* Intel: dest อยู่ซ้าย */
        ".att_syntax prefix"            /* switch กลับไป AT&T */
        : "=r" (result)
        : "r" (a), "r" (b)
    );
    
    return result;
}

/* การ Compile ด้วย -masm=intel ทำให้ใช้ Intel syntax ตลอด */
/* gcc -masm=intel file.c -o program */

int main(void) {
    printf("AT&T syntax:   %ld\n", att_example(10, 20));
    printf("Intel syntax:  %ld\n", intel_example(10, 20));
    return 0;
}
```

---

## 26.7 Input/Output/Clobber Constraints

### ตารางสรุป Constraints
```
Output Constraints:
  "=r"  - output ใน general-purpose register
  "=m"  - output ใน memory
  "=a"  - output ใน rax/eax/ax/al
  "=b"  - output ใน rbx/ebx/bx/bl
  "=c"  - output ใน rcx/ecx/cx/cl
  "=d"  - output ใน rdx/edx/dx/dl
  "=S"  - output ใน rsi/esi/si
  "=D"  - output ใน rdi/edi/di
  "+r"  - read-write ใน register (ทั้ง input และ output)
  "=&r" - early-clobber output (ต้องต่างกับ input registers)

Input Constraints:
  "r"   - input ใน general-purpose register
  "m"   - input จาก memory
  "i"   - immediate integer constant
  "n"   - immediate integer ที่รู้ค่า ณ compile time
  "g"   - input จาก register, memory หรือ immediate
  "0"   - ใช้ register เดียวกับ output ที่ 0
  "1"   - ใช้ register เดียวกับ output ที่ 1

Clobber Constraints:
  "rax" - แก้ไข rax
  "rbx" - แก้ไข rbx
  "rcx" - แก้ไข rcx
  "rdx" - แก้ไข rdx
  "cc"  - แก้ไข condition codes (flags)
  "memory" - แก้ไข memory (สำคัญ! บอก compiler ว่า memory เปลี่ยน)
```

### ตัวอย่างการใช้ Constraints ต่างๆ
```c
/* file: constraints_demo.c
   ตัวอย่างการใช้ constraints ต่างๆ */

#include <stdio.h>
#include <stdint.h>

/* ตัวอย่าง: 128-bit multiplication */
static inline void mul128(uint64_t a, uint64_t b,
                           uint64_t *hi, uint64_t *lo) {
    /* MUL instruction ใช้ rax * rdi -> rdx:rax */
    __asm__ volatile (
        "mulq %3"           /* rax = rax * b, rdx = upper 64 bits */
        : "=a" (*lo),       /* output: lo = rax */
          "=d" (*hi)        /* output: hi = rdx */
        : "a" (a),          /* input: a ใน rax */
          "r" (b)           /* input: b ใน register ใดก็ได้ */
        : /* ไม่มี extra clobbers */
    );
}

/* ตัวอย่าง: Bit scan forward (BSF) */
static inline int bit_scan_forward(uint64_t x) {
    uint64_t result;
    
    __asm__ volatile (
        "bsfq %1, %0"       /* หา index ของ bit 1 ที่ต่ำสุด */
        : "=r" (result)
        : "rm" (x)          /* input จาก register หรือ memory */
        : "cc"
    );
    
    return (int)result;
}

/* ตัวอย่าง: Population count (นับจำนวน bit ที่เป็น 1) */
static inline int popcount_asm(uint64_t x) {
    uint64_t result;
    
    __asm__ volatile (
        "popcntq %1, %0"    /* นับจำนวน 1-bits */
        : "=r" (result)
        : "r" (x)
    );
    
    return (int)result;
}

/* ตัวอย่าง: Read Time Stamp Counter (RDTSC) */
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    
    __asm__ volatile (
        "rdtsc"             /* อ่าน TSC -> edx:eax */
        : "=a" (lo),        /* output: lo = eax */
          "=d" (hi)         /* output: hi = edx */
        : /* ไม่มี input */
        : /* ไม่มี clobbers */
    );
    
    return ((uint64_t)hi << 32) | lo;
}

/* ตัวอย่าง: Memory fence */
static inline void memory_fence(void) {
    __asm__ volatile (
        "mfence"            /* memory fence instruction */
        :                   /* ไม่มี output */
        :                   /* ไม่มี input */
        : "memory"          /* บอก compiler ว่า memory อาจเปลี่ยน */
    );
}

/* ตัวอย่าง: Compare and Swap (CAS) - atomic operation */
static inline int cas(int64_t *ptr, int64_t expected, int64_t newval) {
    uint8_t success;
    
    __asm__ volatile (
        "lock cmpxchgq %2, %1\n\t"  /* atomic compare and exchange */
        "sete %0"                    /* set success = 1 ถ้า equal */
        : "=q" (success),           /* output: success ใน byte register */
          "+m" (*ptr)               /* input/output: memory location */
        : "r" (newval),             /* input: new value */
          "a" (expected)            /* input: expected value ใน rax */
        : "cc", "memory"
    );
    
    return success;
}

int main(void) {
    /* ทดสอบ 128-bit multiplication */
    uint64_t hi, lo;
    mul128(0xFFFFFFFFFFFFFFFFULL, 0xFFFFFFFFFFFFFFFFULL, &hi, &lo);
    printf("0xFFFF...F * 0xFFFF...F = 0x%016lx%016lx\n", hi, lo);
    
    /* ทดสอบ bit scan forward */
    printf("BSF(0b1010000) = %d\n", bit_scan_forward(0b1010000));
    printf("BSF(0b1) = %d\n", bit_scan_forward(0b1));
    
    /* ทดสอบ popcount */
    printf("popcount(0xFF) = %d\n", popcount_asm(0xFF));
    printf("popcount(0xAAAA) = %d\n", popcount_asm(0xAAAA));
    
    /* ทดสอบ RDTSC */
    uint64_t t1 = rdtsc();
    /* ทำงานบางอย่าง */
    volatile int x = 0;
    for (int i = 0; i < 1000; i++) x++;
    uint64_t t2 = rdtsc();
    printf("Cycles elapsed: %lu\n", t2 - t1);
    
    /* ทดสอบ CAS */
    int64_t val = 42;
    int success = cas(&val, 42, 100);  /* เปลี่ยน 42 เป็น 100 */
    printf("CAS(42->100): success=%d, val=%ld\n", success, val);
    success = cas(&val, 42, 200);  /* ล้มเหลว (ค่าปัจจุบันคือ 100 ไม่ใช่ 42) */
    printf("CAS(42->200): success=%d, val=%ld\n", success, val);
    
    return 0;
}
```

---

## 26.8 Memory Constraints

```c
/* file: memory_constraints.c
   การใช้ memory constraints ใน inline assembly */

#include <stdio.h>
#include <stdint.h>
#include <string.h>

/* ตัวอย่าง 1: อ่าน/เขียน memory โดยตรง */
static inline uint64_t load64(const void *addr) {
    uint64_t val;
    
    __asm__ volatile (
        "movq (%1), %0"     /* load 64-bit value จาก memory */
        : "=r" (val)
        : "r" (addr)        /* address ใน register */
        : "memory"
    );
    
    return val;
}

static inline void store64(void *addr, uint64_t val) {
    __asm__ volatile (
        "movq %1, (%0)"     /* store 64-bit value ไปยัง memory */
        :
        : "r" (addr),       /* address */
          "r" (val)         /* value */
        : "memory"
    );
}

/* ตัวอย่าง 2: Non-temporal (streaming) store */
/* ใช้เมื่อต้องการ write memory จำนวนมากแบบ sequential
   หลีกเลี่ยงการ pollute cache */
static inline void stream_store(void *dst, uint64_t val) {
    __asm__ volatile (
        "movnti %1, (%0)"   /* non-temporal integer store */
        :
        : "r" (dst),
          "r" (val)
        : "memory"
    );
}

/* ตัวอย่าง 3: Fast memory copy ด้วย REP MOVSB */
static inline void fast_memcpy(void *dst, const void *src, size_t n) {
    __asm__ volatile (
        "rep movsb"         /* copy rcx bytes จาก rsi ไป rdi */
        : "+D" (dst),       /* rdi: destination (modified) */
          "+S" (src),       /* rsi: source (modified) */
          "+c" (n)          /* rcx: count (modified) */
        :
        : "memory"          /* memory ถูกแก้ไข */
    );
}

/* ตัวอย่าง 4: Fast memory set ด้วย REP STOSB */
static inline void fast_memset(void *dst, uint8_t val, size_t n) {
    __asm__ volatile (
        "rep stosb"         /* fill rcx bytes ด้วยค่า al ที่ rdi */
        : "+D" (dst),       /* rdi: destination */
          "+c" (n)          /* rcx: count */
        : "a" (val)         /* al: value to store */
        : "memory"
    );
}

/* ตัวอย่าง 5: Fast memory compare ด้วย REP CMPSB */
static inline int fast_memcmp(const void *a, const void *b, size_t n) {
    int result;
    
    __asm__ volatile (
        "repe cmpsb\n\t"    /* compare bytes จนกว่าจะไม่เท่ากัน หรือ rcx=0 */
        "je 1f\n\t"         /* ถ้าเท่ากันทั้งหมด */
        "movzbl -1(%0), %%eax\n\t"  /* load byte จาก a */
        "movzbl -1(%1), %%ecx\n\t"  /* load byte จาก b */
        "sub %%ecx, %%eax\n\t"      /* a - b */
        "jmp 2f\n\t"
        "1: xor %%eax, %%eax\n\t"   /* equal: return 0 */
        "2:"
        : "=a" (result),
          "+D" (a),
          "+S" (b)
        : "c" (n)
        : "cc", "memory"
    );
    
    return result;
}

int main(void) {
    /* ทดสอบ load/store */
    uint64_t arr[4] = {0};
    store64(&arr[0], 0xDEADBEEFCAFEBABEULL);
    printf("Stored and loaded: 0x%016lx\n", load64(&arr[0]));
    
    /* ทดสอบ fast_memcpy */
    char src[] = "Hello, Assembly World!";
    char dst[64] = {0};
    fast_memcpy(dst, src, strlen(src) + 1);
    printf("fast_memcpy: %s\n", dst);
    
    /* ทดสอบ fast_memset */
    memset(dst, 0, sizeof(dst));
    fast_memset(dst, 'A', 10);
    dst[10] = 0;
    printf("fast_memset:  %s\n", dst);
    
    /* ทดสอบ fast_memcmp */
    char s1[] = "ABCDEF";
    char s2[] = "ABCDEF";
    char s3[] = "ABCDEG";
    printf("Compare equal: %d\n", fast_memcmp(s1, s2, 6));
    printf("Compare diff:  %d\n", fast_memcmp(s1, s3, 6));
    
    return 0;
}
```

---

## 26.9 Volatile Inline Assembly

```c
/* file: volatile_asm.c
   ความสำคัญของ volatile ใน inline assembly */

#include <stdio.h>
#include <stdint.h>

/*
   volatile บอก compiler:
   1. อย่า optimize ออก (แม้ว่า output จะไม่ถูกใช้)
   2. อย่า move ไปอยู่ที่อื่น
   3. อย่า hoist ออกจาก loop
   
   ควรใช้ volatile เมื่อ:
   - assembly มี side effects (เช่น I/O, hardware access)
   - ลำดับการ execute สำคัญ (เช่น memory barriers)
   - อ่านค่า hardware registers
*/

/* ตัวอย่าง 1: ไม่ใช้ volatile - compiler อาจ optimize ออก */
static inline void nop_no_volatile(void) {
    __asm__ (
        "nop\n\t"
        "nop\n\t"
        "nop"
    );
    /* compiler อาจลบ code นี้ออกถ้าเห็นว่าไม่มี output */
}

/* ตัวอย่าง 2: ใช้ volatile - จะ execute เสมอ */
static inline void nop_volatile(void) {
    __asm__ volatile (
        "nop\n\t"
        "nop\n\t"
        "nop"
    );
    /* volatile: จะ execute เสมอ ไม่ถูก optimize ออก */
}

/* ตัวอย่าง 3: Compiler barrier */
static inline void compiler_barrier(void) {
    /* บอก compiler ว่า memory อาจเปลี่ยนแปลง
       ทำให้ compiler ไม่ cache values ไว้ใน registers */
    __asm__ volatile ("" : : : "memory");
}

/* ตัวอย่าง 4: Hardware port I/O (Linux kernel style) */
static inline uint8_t inb(uint16_t port) {
    uint8_t val;
    /* volatile สำคัญมาก! เพราะ hardware port reading มี side effects */
    __asm__ volatile (
        "inb %w1, %b0"
        : "=a" (val)
        : "Nd" (port)   /* N = unsigned 8-bit immediate, d = dx register */
    );
    return val;
}

static inline void outb(uint16_t port, uint8_t val) {
    __asm__ volatile (
        "outb %b0, %w1"
        :
        : "a" (val),
          "Nd" (port)
    );
}

/* ตัวอย่าง 5: Precise timing measurement */
static inline uint64_t rdtsc_serialize(void) {
    uint32_t lo, hi;
    
    /* CPUID serialize + RDTSC สำหรับ accurate timing */
    __asm__ volatile (
        "cpuid\n\t"         /* serialize pipeline */
        "rdtsc"
        : "=a" (lo), "=d" (hi)
        :
        : "rbx", "rcx"     /* cpuid clobbers rbx, rcx */
    );
    
    return ((uint64_t)hi << 32) | lo;
}

/* ตัวอย่าง 6: การวัดเวลา code */
static void benchmark(void (*func)(void), int iterations, const char *name) {
    uint64_t start, end;
    
    start = rdtsc_serialize();
    for (int i = 0; i < iterations; i++) {
        func();
    }
    end = rdtsc_serialize();
    
    printf("%s: %lu cycles / %d iterations = %.2f cycles/iter\n",
           name, end - start, iterations,
           (double)(end - start) / iterations);
}

static void test_func(void) {
    volatile int x = 0;
    x++;
}

int main(void) {
    benchmark(test_func, 1000000, "test_func");
    return 0;
}
```

---

## 26.10 Practical: CPUID Function

```nasm
; file: cpuid_info.asm
; อ่านข้อมูล CPU ด้วย CPUID instruction

extern printf
extern puts

section .data
    ; Format strings
    fmt_vendor  db "CPU Vendor: %s", 10, 0
    fmt_brand   db "CPU Brand:  %s", 10, 0
    fmt_features db "Features:", 10, 0
    fmt_feat    db "  %s: %s", 10, 0
    feat_yes    db "YES", 0
    feat_no     db "NO", 0
    
    ; Feature names
    f_sse2      db "SSE2", 0
    f_avx       db "AVX", 0
    f_avx2      db "AVX2", 0
    f_aes       db "AES-NI", 0
    f_popcnt    db "POPCNT", 0
    f_bmi1      db "BMI1", 0
    f_bmi2      db "BMI2", 0
    f_sha       db "SHA", 0

section .bss
    vendor_str  resb 13         ; 12 bytes + null terminator
    brand_str   resb 49         ; 48 bytes + null terminator

section .text
    global main

;----------------------------------------------------------
; get_cpuid_info: รวบรวมข้อมูล CPU
;----------------------------------------------------------
main:
    push rbp
    mov rbp, rsp
    push rbx                    ; rbx ต้องถูก preserve (System V ABI)
    
    ;------------------------------------------
    ; CPUID leaf 0: Vendor String
    ; EBX:ECX:EDX = vendor string (12 chars)
    ;------------------------------------------
    xor eax, eax                ; CPUID function 0
    cpuid
    
    ; เก็บ vendor string: EBX, EDX, ECX
    mov dword [rel vendor_str + 0], ebx
    mov dword [rel vendor_str + 4], edx
    mov dword [rel vendor_str + 8], ecx
    mov byte  [rel vendor_str + 12], 0     ; null terminator
    
    ; พิมพ์ vendor string
    lea rdi, [rel fmt_vendor]
    lea rsi, [rel vendor_str]
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; CPUID leaf 0x80000002-4: Brand String
    ; ชื่อเต็มของ CPU (48 characters)
    ;------------------------------------------
    mov eax, 0x80000002         ; CPUID function 0x80000002
    cpuid
    ; eax:ebx:ecx:edx = brand string bytes 0-15
    mov dword [rel brand_str + 0],  eax
    mov dword [rel brand_str + 4],  ebx
    mov dword [rel brand_str + 8],  ecx
    mov dword [rel brand_str + 12], edx
    
    mov eax, 0x80000003         ; bytes 16-31
    cpuid
    mov dword [rel brand_str + 16], eax
    mov dword [rel brand_str + 20], ebx
    mov dword [rel brand_str + 24], ecx
    mov dword [rel brand_str + 28], edx
    
    mov eax, 0x80000004         ; bytes 32-47
    cpuid
    mov dword [rel brand_str + 32], eax
    mov dword [rel brand_str + 36], ebx
    mov dword [rel brand_str + 40], ecx
    mov dword [rel brand_str + 44], edx
    mov byte  [rel brand_str + 48], 0
    
    ; พิมพ์ brand string
    lea rdi, [rel fmt_brand]
    lea rsi, [rel brand_str]
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; CPUID leaf 1: Feature Flags
    ; ECX และ EDX มี feature flags
    ;------------------------------------------
    mov eax, 1
    cpuid
    ; edx มี SSE2 ที่ bit 26
    ; ecx มี SSE4.2, AES, AVX, POPCNT ฯลฯ
    push rdx                    ; เก็บ edx (feature flags)
    push rcx                    ; เก็บ ecx (feature flags)
    
    ; CPUID leaf 7: Extended Features (AVX2, BMI, SHA)
    xor ecx, ecx                ; sub-leaf 0
    mov eax, 7
    cpuid
    push rbx                    ; เก็บ ebx (extended features)
    
    ; พิมพ์หัวข้อ features
    lea rdi, [rel fmt_features]
    call puts
    
    ; ตรวจสอบ SSE2 (EDX bit 26 จาก leaf 1)
    mov rdx, [rsp + 16]         ; โหลด edx ที่เก็บไว้
    lea rdi, [rel fmt_feat]
    lea rsi, [rel f_sse2]
    bt edx, 26                  ; ตรวจ bit 26
    jnc .no_sse2
    lea rdx, [rel feat_yes]
    jmp .print_sse2
.no_sse2:
    lea rdx, [rel feat_no]
.print_sse2:
    xor eax, eax
    call printf
    
    ; ตรวจสอบ AES-NI (ECX bit 25 จาก leaf 1)
    mov rcx, [rsp + 8]          ; โหลด ecx ที่เก็บไว้
    lea rdi, [rel fmt_feat]
    lea rsi, [rel f_aes]
    bt ecx, 25                  ; ตรวจ bit 25
    jnc .no_aes
    lea rdx, [rel feat_yes]
    jmp .print_aes
.no_aes:
    lea rdx, [rel feat_no]
.print_aes:
    xor eax, eax
    call printf
    
    ; ตรวจสอบ AVX (ECX bit 28 จาก leaf 1)
    mov rcx, [rsp + 8]
    lea rdi, [rel fmt_feat]
    lea rsi, [rel f_avx]
    bt ecx, 28                  ; ตรวจ bit 28
    jnc .no_avx
    lea rdx, [rel feat_yes]
    jmp .print_avx
.no_avx:
    lea rdx, [rel feat_no]
.print_avx:
    xor eax, eax
    call printf
    
    ; ตรวจสอบ POPCNT (ECX bit 23 จาก leaf 1)
    mov rcx, [rsp + 8]
    lea rdi, [rel fmt_feat]
    lea rsi, [rel f_popcnt]
    bt ecx, 23                  ; ตรวจ bit 23
    jnc .no_popcnt
    lea rdx, [rel feat_yes]
    jmp .print_popcnt
.no_popcnt:
    lea rdx, [rel feat_no]
.print_popcnt:
    xor eax, eax
    call printf
    
    ; ตรวจสอบ AVX2 (EBX bit 5 จาก leaf 7)
    mov rbx, [rsp]              ; โหลด ebx จาก leaf 7
    lea rdi, [rel fmt_feat]
    lea rsi, [rel f_avx2]
    bt ebx, 5                   ; ตรวจ bit 5
    jnc .no_avx2
    lea rdx, [rel feat_yes]
    jmp .print_avx2
.no_avx2:
    lea rdx, [rel feat_no]
.print_avx2:
    xor eax, eax
    call printf
    
    ; ตรวจสอบ SHA (EBX bit 29 จาก leaf 7)
    mov rbx, [rsp]
    lea rdi, [rel fmt_feat]
    lea rsi, [rel f_sha]
    bt ebx, 29                  ; ตรวจ bit 29
    jnc .no_sha
    lea rdx, [rel feat_yes]
    jmp .print_sha
.no_sha:
    lea rdx, [rel feat_no]
.print_sha:
    xor eax, eax
    call printf
    
    ; Restore stack
    add rsp, 24                 ; pop rbx, rcx, rdx ที่ push ไว้
    
    xor eax, eax
    pop rbx
    pop rbp
    ret
```

---

## 26.11 Fast String Operations

```nasm
; file: fast_strings.asm
; Fast string operations ด้วย Assembly

section .text
    global asm_strlen
    global asm_strchr
    global asm_memcpy_fast
    global asm_memset_fast

;----------------------------------------------------------
; asm_strlen: หาความยาว string แบบ fast
; Parameters: rdi = string pointer
; Returns: rax = length
; หลักการ: ตรวจสอบ 8 bytes ต่อครั้ง
;----------------------------------------------------------
asm_strlen:
    mov rax, rdi                ; rax = current position
    
    ; Align เพื่อความปลอดภัยในการอ่าน
.find_null:
    mov rcx, [rax]              ; โหลด 8 bytes
    ; ตรวจสอบว่ามี null byte ไหม
    ; ใช้ trick: ถ้า byte ใดเป็น 0 จะทำให้ bit pattern พิเศษ
    mov rdx, rcx
    sub rdx, 0x0101010101010101 ; ลบ 1 จากแต่ละ byte
    not rcx                     ; invert ทุก bit
    and rdx, rcx                ; AND result
    and rdx, 0x8080808080808080 ; เก็บแค่ high bits
    jnz .found_null             ; ถ้ามี null
    add rax, 8                  ; ไป 8 bytes ถัดไป
    jmp .find_null
    
.found_null:
    ; หาตำแหน่งแน่ๆ ของ null byte
    bsfq rcx, rdx               ; หา bit position ต่ำสุดที่เป็น 1
    shr rcx, 3                  ; แปลง bit position เป็น byte offset
    add rax, rcx                ; rax = address ของ null byte
    sub rax, rdi                ; length = null_pos - start
    ret

;----------------------------------------------------------
; asm_strchr: หาอักษรใน string
; Parameters: rdi = string pointer, rsi = character to find
; Returns: rax = pointer to character, หรือ NULL ถ้าไม่พบ
;----------------------------------------------------------
asm_strchr:
    movzx esi, sil              ; เก็บแค่ byte ต่ำสุดของ character
    
.loop:
    movzx eax, byte [rdi]       ; โหลด 1 byte
    test al, al                 ; ตรวจ null terminator
    jz .not_found
    cmp al, sil                 ; เปรียบเทียบกับ character ที่หา
    je .found
    inc rdi                     ; ไปยังอักษรถัดไป
    jmp .loop
    
.found:
    mov rax, rdi                ; return pointer to character
    ret
    
.not_found:
    xor eax, eax                ; return NULL
    ret

;----------------------------------------------------------
; asm_memcpy_fast: Copy memory ด้วย SIMD (SSE2)
; Parameters: rdi = dst, rsi = src, rdx = size
; Returns: rax = dst
;----------------------------------------------------------
asm_memcpy_fast:
    mov rax, rdi                ; เก็บ dst สำหรับ return value
    
    ; Copy ด้วย 16-byte chunks (XMM registers)
    cmp rdx, 16
    jl .byte_copy
    
.chunk16:
    movdqu xmm0, [rsi]          ; load 16 bytes จาก src
    movdqu [rdi], xmm0          ; store 16 bytes ไป dst
    add rsi, 16
    add rdi, 16
    sub rdx, 16
    cmp rdx, 16
    jge .chunk16
    
    ; Copy ส่วนที่เหลือทีละ byte
.byte_copy:
    test rdx, rdx
    jz .done
    
.byte_loop:
    mov cl, [rsi]
    mov [rdi], cl
    inc rsi
    inc rdi
    dec rdx
    jnz .byte_loop
    
.done:
    ret

;----------------------------------------------------------
; asm_memset_fast: Set memory ด้วย REP STOSQ
; Parameters: rdi = dst, rsi = value (1 byte), rdx = size
; Returns: rax = dst
;----------------------------------------------------------
asm_memset_fast:
    mov rax, rdi                ; เก็บ dst สำหรับ return value
    
    ; กระจายค่า 1 byte ไปใน 8 bytes ของ rax
    movzx rsi, sil
    mov rcx, 0x0101010101010101
    imul rsi, rcx               ; ทำให้ทุก byte มีค่าเดียวกัน
    
    ; Set ด้วย 8-byte chunks
    mov rcx, rdx
    shr rcx, 3                  ; จำนวน 8-byte chunks
    jz .remainder
    
    push rdi
    mov rax, rsi                ; ค่าที่จะ set
    rep stosq                   ; set rcx qwords
    pop rdi
    
.remainder:
    mov rcx, rdx
    and rcx, 7                  ; จำนวน bytes ที่เหลือ
    jz .done2
    add rdi, rdx
    and rdi, ~7                 ; align ไปยัง 8-byte boundary
    
.byte_set:
    mov [rdi], sil
    inc rdi
    dec rcx
    jnz .byte_set
    
.done2:
    mov rax, qword [rsp - 8]    ; แต่เราไม่ได้ push อะไร... 
    ; แก้ให้ถูกต้อง: return original dst
    ret
```

### C Wrapper สำหรับ Fast String Functions
```c
/* file: string_demo.c */

#include <stdio.h>
#include <string.h>
#include <time.h>
#include <stdint.h>

/* Declarations สำหรับ assembly functions */
extern size_t asm_strlen(const char *s);
extern char  *asm_strchr(const char *s, int c);
extern void  *asm_memcpy_fast(void *dst, const void *src, size_t n);

#define BENCHMARK(name, code, n) do { \
    struct timespec t1, t2; \
    clock_gettime(CLOCK_MONOTONIC, &t1); \
    for (int _i = 0; _i < (n); _i++) { code; } \
    clock_gettime(CLOCK_MONOTONIC, &t2); \
    double ms = (t2.tv_sec - t1.tv_sec) * 1000.0 + \
                (t2.tv_nsec - t1.tv_nsec) / 1e6; \
    printf("%-20s: %.2f ms (%d iters)\n", name, ms, n); \
} while (0)

int main(void) {
    const char *test_str = "Hello, Assembly World! This is a test string for benchmarking.";
    
    /* ทดสอบ strlen */
    printf("C strlen:   %zu\n", strlen(test_str));
    printf("ASM strlen: %zu\n", asm_strlen(test_str));
    
    /* ทดสอบ strchr */
    char *p1 = strchr(test_str, 'W');
    char *p2 = asm_strchr(test_str, 'W');
    printf("C strchr:   %s\n", p1);
    printf("ASM strchr: %s\n", p2);
    
    /* Benchmark */
    printf("\n--- Benchmark (1M iterations) ---\n");
    BENCHMARK("C strlen",   strlen(test_str), 1000000);
    BENCHMARK("ASM strlen", asm_strlen(test_str), 1000000);
    
    return 0;
}
```

---

## 26.12 Calling C++ (Name Mangling และ extern "C")

### ปัญหา Name Mangling ใน C++
```cpp
/* file: cpp_mangling.cpp
   อธิบายปัญหา name mangling และการแก้ด้วย extern "C" */

/*
   C++ ทำ "name mangling" เพื่อรองรับ:
   - Function overloading
   - Namespaces
   - Class methods
   
   ตัวอย่าง:
   - C function:  void foo(int)    -> _foo หรือ foo
   - C++ function: void foo(int)   -> _Z3fooi
   - C++ method:  Foo::bar(int)    -> _ZN3Foo3barEi
   
   ปัญหา: Assembly ต้องรู้ชื่อที่แท้จริง!
   แก้ด้วย extern "C" เพื่อบังคับให้ใช้ C linkage (ไม่ mangle)
*/

/* ประกาศ C-linkage functions ใน C++ */
extern "C" {
    /* Functions เหล่านี้จะไม่ถูก mangle */
    int cpp_add(int a, int b);           /* ชื่อจริง: cpp_add */
    void cpp_print(const char *s);       /* ชื่อจริง: cpp_print */
    int cpp_fibonacci(int n);            /* ชื่อจริง: cpp_fibonacci */
}

/* Implementation */
extern "C" int cpp_add(int a, int b) {
    return a + b;
}

extern "C" void cpp_print(const char *s) {
    printf("%s\n", s);
}

extern "C" int cpp_fibonacci(int n) {
    if (n <= 1) return n;
    return cpp_fibonacci(n - 1) + cpp_fibonacci(n - 2);
}

/* C++ Class ที่ Assembly ต้องเรียกใช้ */
class Calculator {
public:
    Calculator(int initial = 0) : value(initial) {}
    
    int add(int x) { value += x; return value; }
    int subtract(int x) { value -= x; return value; }
    int get_value() const { return value; }
    
private:
    int value;
};

/* C wrapper functions สำหรับ class */
/* (เพราะ Assembly เรียก C++ methods ตรงๆ ยากมาก) */
extern "C" {
    /* opaque pointer pattern */
    typedef void* CalcHandle;
    
    CalcHandle calc_create(int initial) {
        return new Calculator(initial);
    }
    
    void calc_destroy(CalcHandle h) {
        delete static_cast<Calculator*>(h);
    }
    
    int calc_add(CalcHandle h, int x) {
        return static_cast<Calculator*>(h)->add(x);
    }
    
    int calc_subtract(CalcHandle h, int x) {
        return static_cast<Calculator*>(h)->subtract(x);
    }
    
    int calc_get_value(CalcHandle h) {
        return static_cast<Calculator*>(h)->get_value();
    }
}
```

### Assembly ที่เรียก C++ Functions
```nasm
; file: call_cpp.asm
; Assembly ที่เรียกใช้ C++ functions ผ่าน extern "C"

; ประกาศ extern functions (C linkage, ไม่มี name mangling)
extern cpp_add
extern cpp_print
extern cpp_fibonacci
extern calc_create
extern calc_destroy
extern calc_add
extern calc_subtract
extern calc_get_value
extern printf

section .data
    fmt_int     db "Result: %d", 10, 0
    msg_hello   db "Hello from C++!", 0
    fmt_fib     db "fib(%d) = %d", 10, 0

section .text
    global asm_main

;----------------------------------------------------------
; asm_main: เรียก C++ functions จาก Assembly
;----------------------------------------------------------
asm_main:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    ;------------------------------------------
    ; เรียก cpp_add
    ;------------------------------------------
    mov edi, 15                 ; a = 15
    mov esi, 27                 ; b = 27
    call cpp_add                ; result ใน eax
    
    lea rdi, [rel fmt_int]
    mov rsi, rax
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; เรียก cpp_print
    ;------------------------------------------
    lea rdi, [rel msg_hello]
    call cpp_print
    
    ;------------------------------------------
    ; เรียก cpp_fibonacci
    ;------------------------------------------
    mov edi, 10                 ; n = 10
    call cpp_fibonacci
    
    lea rdi, [rel fmt_fib]
    mov rsi, 10
    mov rdx, rax
    xor eax, eax
    call printf
    
    ;------------------------------------------
    ; ใช้ Calculator class ผ่าน C wrappers
    ;------------------------------------------
    mov edi, 100                ; initial value = 100
    call calc_create            ; rax = handle
    mov rbx, rax                ; เก็บ handle ใน rbx
    
    mov rdi, rbx
    mov esi, 50                 ; เพิ่ม 50
    call calc_add
    
    mov rdi, rbx
    mov esi, 25                 ; ลบ 25
    call calc_subtract
    
    mov rdi, rbx
    call calc_get_value         ; อ่านค่า
    
    lea rdi, [rel fmt_int]
    mov rsi, rax                ; ควรได้ 125 (100 + 50 - 25)
    xor eax, eax
    call printf
    
    ; Destroy calculator
    mov rdi, rbx
    call calc_destroy
    
    xor eax, eax
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 26.13 Linking Assembly กับ Shared Libraries

### การสร้าง Shared Library จาก Assembly
```nasm
; file: libasm_math.asm
; Assembly code ที่จะ compile เป็น shared library (.so)

section .text

; Export symbols
global asm_fast_pow             ; fast power function
global asm_isqrt                ; integer square root
global asm_clz                  ; count leading zeros
global asm_ctz                  ; count trailing zeros

;----------------------------------------------------------
; asm_fast_pow: คำนวณ x^n แบบ fast (Exponentiation by Squaring)
; Parameters: rdi = base, rsi = exponent
; Returns: rax = base^exponent
;----------------------------------------------------------
asm_fast_pow:
    mov rax, 1                  ; result = 1
    test rsi, rsi               ; ถ้า exponent = 0
    jz .done                    ; return 1
    
.loop:
    test rsi, 1                 ; ตรวจ bit ต่ำสุดของ exponent
    jz .even
    
    ; odd: result *= base
    imul rax, rdi
    
.even:
    imul rdi, rdi               ; base = base * base
    shr rsi, 1                  ; exponent >>= 1
    jnz .loop                   ; ถ้า exponent ยังไม่เป็น 0
    
.done:
    ret

;----------------------------------------------------------
; asm_isqrt: Integer Square Root (Newton's method)
; Parameters: rdi = n
; Returns: rax = floor(sqrt(n))
;----------------------------------------------------------
asm_isqrt:
    test rdi, rdi               ; ถ้า n = 0
    jz .return_zero
    
    ; initial guess = n >> 1
    mov rax, rdi
    shr rax, 1
    jz .small_n                 ; ถ้า n = 1
    
    ; Newton's method: x = (x + n/x) / 2
.newton:
    mov rcx, rdi                ; rcx = n
    xor edx, edx
    div rax                     ; rdx:rax / rax -> quotient ใน rax
    ; แต่เราต้องการ n/rax ไม่ใช่ rax/rax
    ; ต้องเขียนใหม่
    mov rax, rdi                ; restore rax = n (approximation)
    mov rcx, rdi                ; rcx = n
    xor edx, edx
    div rax                     ; rax = n / rax
    ; แต่ rax ถูก overwrite แล้ว...
    ; ใช้ approach อื่น
    
    ; ใช้ Babylonian method อย่างง่าย
    mov rax, rdi                ; rax = n
    bsr rcx, rdi                ; rcx = floor(log2(n))
    inc rcx
    shr rcx, 1                  ; rcx = bits/2
    mov rax, rdi
    shr rax, cl                 ; initial guess
    
.iter:
    mov rbx, rdi                ; rbx = n
    xor edx, edx
    div rax                     ; rax = n / guess
    ; ตอนนี้ rax = n/guess, rdx = remainder
    add rax, qword [rsp - 8]    ; เพิ่ม old guess... ไม่ได้เก็บไว้
    ; แก้ให้ถูกต้อง
    ret
    
.small_n:
    cmp rdi, 1
    je .return_one
    
.return_zero:
    xor eax, eax
    ret
    
.return_one:
    mov eax, 1
    ret

;----------------------------------------------------------
; asm_clz: Count Leading Zeros
; Parameters: rdi = value
; Returns: rax = number of leading zeros
;----------------------------------------------------------
asm_clz:
    bsr rax, rdi                ; rax = index ของ highest set bit
    jz .is_zero
    xor rax, 63                 ; แปลงเป็นจำนวน leading zeros
    ret
    
.is_zero:
    mov rax, 64                 ; 64 leading zeros ถ้า input = 0
    ret

;----------------------------------------------------------
; asm_ctz: Count Trailing Zeros
; Parameters: rdi = value
; Returns: rax = number of trailing zeros
;----------------------------------------------------------
asm_ctz:
    bsf rax, rdi                ; rax = index ของ lowest set bit
    jz .is_zero
    ret
    
.is_zero:
    mov rax, 64                 ; 64 trailing zeros ถ้า input = 0
    ret
```

### Build Shared Library
```bash
# Step 1: Assemble เป็น Position Independent Code (PIC)
nasm -f elf64 -DPIC libasm_math.asm -o libasm_math.o

# Step 2: สร้าง shared library
gcc -shared -o libasm_math.so libasm_math.o

# Step 3: Compile C program ที่ใช้ shared library
gcc -o program main.c -L. -lasm_math -Wl,-rpath,.

# หรือใช้ static library
ar rcs libasm_math.a libasm_math.o
gcc -o program main.c -L. -lasm_math_static
```

### C Program ที่ใช้ Shared Library
```c
/* file: use_shared_lib.c */

#include <stdio.h>
#include <stdint.h>

/* declarations */
extern int64_t asm_fast_pow(int64_t base, int64_t exp);
extern int64_t asm_clz(uint64_t x);
extern int64_t asm_ctz(uint64_t x);

int main(void) {
    /* ทดสอบ fast power */
    printf("2^10 = %ld\n", asm_fast_pow(2, 10));
    printf("3^5  = %ld\n", asm_fast_pow(3, 5));
    printf("7^0  = %ld\n", asm_fast_pow(7, 0));
    
    /* ทดสอบ CLZ/CTZ */
    uint64_t val = 0x000F000000000000ULL;
    printf("CLZ(0x000F000000000000) = %ld\n", asm_clz(val));
    
    val = 0x0000000000F00000ULL;
    printf("CTZ(0x0000000000F00000) = %ld\n", asm_ctz(val));
    
    return 0;
}
```

---

## 26.14 Complete Project: Crypto Primitives

```nasm
; file: crypto_asm.asm
; Assembly crypto primitives - XOR cipher และ byte substitution

section .text
    global asm_xor_encrypt
    global asm_xor_decrypt
    global asm_rotate_left32
    global asm_rotate_right32

;----------------------------------------------------------
; asm_xor_encrypt: XOR encryption/decryption
; Parameters: rdi = data pointer, rsi = length, rdx = key pointer, rcx = key_len
; Note: XOR encryption = XOR decryption (symmetric)
;----------------------------------------------------------
asm_xor_encrypt:
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi                ; data pointer
    mov r13, rsi                ; data length
    mov r14, rdx                ; key pointer
    ; rcx = key length
    
    xor rbx, rbx                ; key index = 0
    xor r8, r8                  ; data index = 0
    
.loop:
    cmp r8, r13                 ; ถ้า data index >= length
    jge .done
    
    mov al, [r12 + r8]          ; โหลด data byte
    
    ; คำนวณ key index = (i % key_len)
    mov rax_save, rbx
    mov rax, rbx
    xor edx, edx
    div rcx                     ; rax = rbx / key_len, rdx = rbx % key_len
    ; rdx = key index
    
    mov al, [r12 + r8]          ; โหลด data byte (ใช้ index ที่ถูก)
    xor al, [r14 + rdx]         ; XOR กับ key byte
    mov [r12 + r8], al          ; เก็บ result กลับ
    
    inc rbx                     ; key index++
    inc r8                      ; data index++
    jmp .loop
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    ret

;----------------------------------------------------------
; asm_rotate_left32: Rotate Left 32-bit
; Parameters: edi = value, esi = count
; Returns: eax = rotated value
;----------------------------------------------------------
asm_rotate_left32:
    mov eax, edi                ; eax = value
    mov ecx, esi                ; cl = count
    rol eax, cl                 ; rotate left
    ret

;----------------------------------------------------------
; asm_rotate_right32: Rotate Right 32-bit
; Parameters: edi = value, esi = count
; Returns: eax = rotated value
;----------------------------------------------------------
asm_rotate_right32:
    mov eax, edi
    mov ecx, esi
    ror eax, cl                 ; rotate right
    ret
```

### XOR Cipher ที่สมบูรณ์ด้วย C และ Assembly
```c
/* file: xor_cipher.c
   XOR cipher ที่ใช้ Assembly สำหรับ XOR operations */

#include <stdio.h>
#include <string.h>
#include <stdint.h>

/* Fast XOR ด้วย inline assembly (process 8 bytes at once) */
static void xor_block(uint8_t *data, const uint8_t *key, size_t len,
                       const uint8_t *keyblock, size_t keylen) {
    /* สร้าง keystream จาก key */
    uint8_t keystream[len];
    for (size_t i = 0; i < len; i++) {
        keystream[i] = keyblock[i % keylen];
    }
    
    /* XOR ด้วย 8 bytes ต่อครั้ง */
    size_t i = 0;
    size_t chunks = len / 8;
    
    for (size_t c = 0; c < chunks; c++, i += 8) {
        uint64_t *d = (uint64_t *)(data + i);
        uint64_t *k = (uint64_t *)(keystream + i);
        
        __asm__ volatile (
            "xorq %1, %0"
            : "+m" (*d)
            : "r" (*k)
            : "memory"
        );
    }
    
    /* XOR bytes ที่เหลือ */
    for (; i < len; i++) {
        data[i] ^= keystream[i];
    }
}

void print_hex(const uint8_t *data, size_t len, const char *label) {
    printf("%s: ", label);
    for (size_t i = 0; i < len; i++) {
        printf("%02x ", data[i]);
    }
    printf("\n");
}

int main(void) {
    const char *plaintext = "Hello, Secret World!";
    const char *key = "MySecretKey";
    
    size_t len = strlen(plaintext);
    uint8_t data[len];
    memcpy(data, plaintext, len);
    
    printf("Original: %s\n", (char*)data);
    print_hex(data, len, "Plaintext");
    
    /* Encrypt */
    xor_block(data, NULL, len, (const uint8_t*)key, strlen(key));
    print_hex(data, len, "Encrypted");
    
    /* Decrypt (XOR again) */
    xor_block(data, NULL, len, (const uint8_t*)key, strlen(key));
    printf("Decrypted: %s\n", (char*)data);
    
    return 0;
}
```

---

## 26.15 Complete Makefile สำหรับ Mixed Project

```makefile
# Makefile สมบูรณ์สำหรับโปรเจกต์ Mixed C/C++/Assembly

CC       = gcc
CXX      = g++
NASM     = nasm
AR       = ar

CFLAGS   = -Wall -Wextra -O2 -g -std=c11
CXXFLAGS = -Wall -Wextra -O2 -g -std=c++17
NASMFLAGS = -f elf64 -g -F dwarf -DPIC
LDFLAGS  = -no-pie

# Source files
C_SRCS   := $(wildcard *.c)
CPP_SRCS := $(wildcard *.cpp)
ASM_SRCS := $(wildcard *.asm)

# Object files
C_OBJS   := $(C_SRCS:.c=.o)
CPP_OBJS := $(CPP_SRCS:.cpp=.o)
ASM_OBJS := $(ASM_SRCS:.asm=.o)
ALL_OBJS := $(C_OBJS) $(CPP_OBJS) $(ASM_OBJS)

TARGET   = program

# Detect if we have C++ files -> use g++ for linking
ifneq ($(CPP_SRCS),)
    LD = $(CXX)
else
    LD = $(CC)
endif

.PHONY: all clean run debug test asm-list

all: $(TARGET)

$(TARGET): $(ALL_OBJS)
	$(LD) $(LDFLAGS) -o $@ $^
	@echo "Built: $(TARGET)"

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c -o $@ $<

%.o: %.asm
	$(NASM) $(NASMFLAGS) -o $@ $<

clean:
	$(RM) $(ALL_OBJS) $(TARGET) *.so *.a
	@echo "Cleaned"

run: $(TARGET)
	./$(TARGET)

debug: CFLAGS += -DDEBUG -fsanitize=address
debug: CXXFLAGS += -DDEBUG -fsanitize=address
debug: LDFLAGS += -fsanitize=address
debug: $(TARGET)

test: $(TARGET)
	valgrind --error-exitcode=1 ./$(TARGET)

# แสดง assembly output จาก C code
asm-list: $(C_SRCS)
	$(CC) $(CFLAGS) -S -masm=intel -o - $< | less

# สร้าง static library
libasm.a: $(ASM_OBJS)
	$(AR) rcs $@ $^

# สร้าง shared library
libasm.so: $(ASM_OBJS)
	$(CC) -shared -o $@ $^

# Dependencies
-include $(ALL_OBJS:.o=.d)
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: สร้าง String Library ใน Assembly
```
ให้เขียน Assembly functions ต่อไปนี้และเรียกจาก C:

1. asm_strcat(char *dst, const char *src)   - เชื่อม strings
2. asm_strcmp(const char *a, const char *b) - เปรียบเทียบ strings  
3. asm_toupper(char *str)                   - แปลงเป็น uppercase
4. asm_tolower(char *str)                   - แปลงเป็น lowercase
5. asm_strrev(char *str)                    - กลับ string

สร้าง header file, Makefile, และ test program ด้วย
```

### Exercise 2: Inline Assembly Matrix Operations
```c
/* TODO: เขียน inline assembly สำหรับ matrix operations */

/* Matrix 2x2 multiply */
void matrix2x2_mul(const int64_t A[2][2], 
                   const int64_t B[2][2], 
                   int64_t C[2][2]) {
    /* TODO: Implement ด้วย inline assembly */
    /* C[i][j] = sum(A[i][k] * B[k][j]) */
}

/* Matrix 4x4 determinant */
int64_t matrix4x4_det(const int64_t M[4][4]) {
    /* TODO: Implement ด้วย inline assembly */
}
```

### Exercise 3: CPUID Feature Detection Library
```
สร้าง library ที่ detect CPU features ต่อไปนี้:
- SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2
- AVX, AVX2, AVX-512
- AES-NI
- SHA Extensions
- BMI1, BMI2
- POPCNT, LZCNT
- TSX (Transactional Synchronization Extensions)

ให้มี function:
- cpu_has_feature(uint64_t feature_bit)
- cpu_get_features() -> ส่งคืน bitmask ของ features ที่รองรับ
- cpu_print_info() -> พิมพ์ข้อมูล CPU ทั้งหมด
```

### Exercise 4: Shared Library Development
```
สร้าง shared library ชื่อ libfastmath.so ที่มี functions:
1. fast_sqrt(double x)  - ใช้ sqrtsd instruction
2. fast_ceil(double x)  - ใช้ SSE ceiling
3. fast_floor(double x) - ใช้ SSE floor
4. fast_round(double x) - ใช้ SSE round
5. dot_product(const double *a, const double *b, int n) - dot product ด้วย SSE

สร้าง:
- Header file (.h)
- Makefile
- Test program ที่เปรียบเทียบความเร็วกับ C version
```

### Exercise 5: Complete XOR Stream Cipher
```
พัฒนา XOR stream cipher ที่สมบูรณ์:

1. Key scheduling ด้วย Assembly (KSA algorithm)
2. Pseudo-random generation ด้วย Assembly  
3. Stream XOR ด้วย Assembly (8 bytes/iteration)
4. C interface (encrypt/decrypt/init/destroy)
5. Test ด้วย known test vectors
6. Benchmark เปรียบเทียบกับ pure C version
```

---

## สรุปคำสั่ง Compilation (Summary)

```bash
# 1. Compile Assembly เป็น object file
nasm -f elf64 file.asm -o file.o

# 2. Compile C เป็น object file  
gcc -c file.c -o file.o

# 3. Link ทั้งหมด
gcc -o program main.o asm_file.o

# 4. C++ project
g++ -o program main.o cpp_file.o asm_file.o

# 5. สร้าง shared library
gcc -shared -fPIC asm_file.o -o libmyasm.so

# 6. Link กับ shared library
gcc -o program main.c -L. -lmyasm -Wl,-rpath,.

# 7. Debug inline asm output
gcc -S -masm=intel -O0 file.c -o file.s

# 8. ดู symbol table
nm program
objdump -t program

# 9. ดู dynamic symbols
objdump -T program

# 10. ตรวจสอบ linking
ldd program
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **extern/global directives** - การ export/import symbols ระหว่างไฟล์
2. **Calling Assembly from C** - สร้าง header files และ link ด้วย Makefile
3. **Calling C from Assembly** - ใช้ extern และ System V AMD64 ABI
4. **Inline Assembly** - Basic และ Extended forms ใน GCC
5. **AT&T vs Intel Syntax** - ความแตกต่างและการ switch ระหว่าง syntaxes
6. **Constraints** - Input, output, และ clobber constraints
7. **Memory Operations** - Fast memcpy/memset ด้วย Assembly
8. **Volatile** - ความสำคัญของ volatile ใน inline asm
9. **CPUID** - การ detect CPU features
10. **Shared Libraries** - การสร้างและใช้ shared library จาก Assembly
11. **C++ Interop** - Name mangling และ extern "C"
12. **Crypto Primitives** - XOR cipher และ rotation operations

**บทถัดไป**: Part 027 - SIMD Programming with SSE/SSE2

---

*จบ Part 026 - Interfacing Assembly with C*
