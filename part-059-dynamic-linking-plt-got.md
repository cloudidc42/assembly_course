# Part 059: Dynamic Linking, PLT/GOT Deep Dive

## บทนำ (Introduction)

Dynamic linking เป็นกลไกสำคัญในระบบปฏิบัติการสมัยใหม่ที่ทำให้โปรแกรมสามารถใช้งาน
shared libraries (.so files) ได้โดยไม่ต้องรวม code ทั้งหมดไว้ใน executable
บทนี้จะอธิบายกลไก PLT (Procedure Linkage Table) และ GOT (Global Offset Table)
อย่างละเอียด พร้อมตัวอย่างโปรแกรม Assembly ที่ทำงานได้จริง

---

## 1. ELF Dynamic Linking Overview

### 1.1 โครงสร้าง ELF (Executable and Linkable Format)

```
ELF File Layout:
┌─────────────────────────────────────┐
│  ELF Header                         │
│  (magic, arch, entry point, etc.)   │
├─────────────────────────────────────┤
│  Program Headers (Segments)         │
│  - PT_LOAD: loadable segments       │
│  - PT_DYNAMIC: dynamic linking info │
│  - PT_INTERP: path to ld.so         │
├─────────────────────────────────────┤
│  .text section (code)               │
├─────────────────────────────────────┤
│  .plt section (PLT stubs)           │
├─────────────────────────────────────┤
│  .got.plt section (GOT for PLT)     │
├─────────────────────────────────────┤
│  .data section                      │
├─────────────────────────────────────┤
│  .bss section                       │
├─────────────────────────────────────┤
│  Section Headers                    │
│  - .dynsym: dynamic symbols         │
│  - .dynstr: symbol name strings     │
│  - .rela.plt: PLT relocations       │
│  - .rela.dyn: dynamic relocations   │
└─────────────────────────────────────┘
```

### 1.2 ส่วนประกอบสำคัญ

**ELF Header fields ที่เกี่ยวข้อง:**
- `e_type`: ET_EXEC (executable) หรือ ET_DYN (shared library/PIE)
- `e_entry`: address ของ entry point
- `e_phoff`: offset ของ Program Header Table
- `e_shoff`: offset ของ Section Header Table

**PT_INTERP segment:**
- บอก kernel ว่าต้องใช้ dynamic linker ไหน
- ปกติเป็น `/lib64/ld-linux-x86-64.so.2`
- Kernel จะโหลด ld.so ก่อน และให้ ld.so resolve dependencies

**PT_DYNAMIC segment:**
- มี .dynamic section
- มีข้อมูล: DT_NEEDED (ชื่อ libraries ที่ต้องการ), DT_SYMTAB, DT_STRTAB, DT_JMPREL

### 1.3 Dynamic Linking Process (ภาพรวม)

```
1. Kernel โหลด ELF executable
2. Kernel อ่าน PT_INTERP → โหลด ld.so
3. ld.so ทำ:
   a. อ่าน .dynamic section
   b. โหลด shared libraries (DT_NEEDED)
   c. Map libraries เข้า memory
   d. Resolve symbols (lazy หรือ eager)
   e. Apply relocations
   f. รัน constructor functions (.init, .init_array)
4. Jump to program entry point (_start)
5. Program เรียก main()
6. เมื่อเรียก external function → ผ่าน PLT
7. PLT → GOT → ld.so (first call) หรือ function (subsequent calls)
```

---

## 2. Lazy Binding Mechanism

### 2.1 แนวคิด Lazy Binding

Lazy binding หมายความว่า address ของ function ใน shared library
จะถูก resolve เฉพาะเมื่อมีการเรียกใช้ครั้งแรกเท่านั้น
ซึ่งช่วยลด startup time โดยเฉพาะโปรแกรมที่ใช้ functions จำนวนมาก

```
First call:
  program → PLT[n] → GOT[n] (= PLT[n]+6) → PLT[0] → ld.so
  ld.so resolves address → เขียน GOT[n] → return to function

Subsequent calls:
  program → PLT[n] → GOT[n] (= real function address) → function
```

### 2.2 ขั้นตอนโดยละเอียด

```
Initial state (ก่อน first call):
  GOT[n] = address ของ PLT[n]+6 (กลับมาที่ PLT เพื่อ resolve)

Call sequence ครั้งแรก:
1. CALL puts@plt         ; โปรแกรมเรียก puts
2. JMP [GOT[puts]]      ; PLT stub: jump ผ่าน GOT
3. GOT[puts] = PLT+6    ; GOT ชี้กลับมาที่ PLT
4. PUSH reloc_index     ; push relocation index
5. JMP PLT[0]           ; jump to resolver
6. PUSH GOT[1]          ; push link_map pointer
7. JMP GOT[2]           ; jump to ld.so resolver
8. ld.so finds puts()   ; resolve symbol
9. เขียน address → GOT[puts]  ; update GOT
10. JMP puts@libc       ; jump to actual puts

Call sequence ครั้งต่อไป:
1. CALL puts@plt
2. JMP [GOT[puts]]      ; GOT มี real address แล้ว
3. ตรง → puts@libc     ; ไม่ผ่าน ld.so
```

---

## 3. PLT (Procedure Linkage Table) Structure

### 3.1 PLT[0]: Resolver Stub

PLT entry แรก (PLT[0]) เป็น special entry ที่ใช้เรียก dynamic linker

```asm
; PLT[0] - Resolver Stub
; Address: 0x401020 (ตัวอย่าง)
PLT0:
    PUSH QWORD PTR [rip + GOT+8]    ; push GOT[1] (link_map)
    JMP  QWORD PTR [rip + GOT+16]   ; jump to GOT[2] (dl_runtime_resolve)
    NOP
    NOP
    NOP
    NOP
```

**GOT special entries:**
- `GOT[0]` = address ของ .dynamic section
- `GOT[1]` = link_map pointer (filled by ld.so at startup)
- `GOT[2]` = address ของ `_dl_runtime_resolve` (filled by ld.so at startup)

### 3.2 PLT[n]: Function Stub

แต่ละ external function มี PLT entry ขนาด 16 bytes:

```asm
; PLT[n] - Function stub สำหรับ puts
; Address: 0x401030 (ตัวอย่าง)
puts@plt:
    JMP  QWORD PTR [rip + GOT_PUTS]  ; 6 bytes: ff 25 xx xx xx xx
    PUSH 0                            ; 5 bytes: 68 00 00 00 00 (reloc index)
    JMP  PLT0                         ; 5 bytes: e9 xx xx xx xx
```

**รายละเอียด:**
- `JMP [GOT_PUTS]`: 6 bytes - indirect jump ผ่าน GOT entry
- `PUSH 0`: 5 bytes - push relocation offset (index เข้า .rela.plt)
- `JMP PLT0`: 5 bytes - jump ไปยัง resolver
- รวม 16 bytes ต่อ entry

### 3.3 ตัวอย่าง PLT Section ใน objdump

```
Disassembly of section .plt:

0000000000401020 <puts@plt-0x10>:   ← PLT[0]
  401020: ff 35 e2 2f 00 00    pushq  0x2fe2(%rip)   # GOT[1]
  401026: ff 25 e4 2f 00 00    jmpq   *0x2fe4(%rip)  # GOT[2]
  40102c: 0f 1f 40 00          nopl   0x0(%rax)

0000000000401030 <puts@plt>:         ← PLT[1] = puts
  401030: ff 25 e2 2f 00 00    jmpq   *0x2fe2(%rip)  # GOT[puts]
  401036: 68 00 00 00 00       pushq  $0x0
  40103b: e9 e0 ff ff ff       jmpq   401020 <puts@plt-0x10>

0000000000401040 <printf@plt>:       ← PLT[2] = printf
  401040: ff 25 da 2f 00 00    jmpq   *0x2fda(%rip)  # GOT[printf]
  401046: 68 01 00 00 00       pushq  $0x1
  40104b: e9 d0 ff ff ff       jmpq   401020 <puts@plt-0x10>
```

---

## 4. GOT (Global Offset Table)

### 4.1 โครงสร้าง GOT

GOT มี 2 ส่วน:
1. `.got` - สำหรับ global variables
2. `.got.plt` - สำหรับ function pointers (ใช้กับ PLT)

```
.got.plt layout:
┌──────────────┬─────────────────────────────────────┐
│ GOT[0]       │ address ของ .dynamic section        │
│ (8 bytes)    │                                     │
├──────────────┼─────────────────────────────────────┤
│ GOT[1]       │ link_map pointer (ld.so fills this) │
│ (8 bytes)    │                                     │
├──────────────┼─────────────────────────────────────┤
│ GOT[2]       │ _dl_runtime_resolve (ld.so fills)   │
│ (8 bytes)    │                                     │
├──────────────┼─────────────────────────────────────┤
│ GOT[3]       │ GOT entry สำหรับ function 1         │
│ (8 bytes)    │ (initial: PLT[1]+6, then real addr) │
├──────────────┼─────────────────────────────────────┤
│ GOT[4]       │ GOT entry สำหรับ function 2         │
│ (8 bytes)    │                                     │
├──────────────┼─────────────────────────────────────┤
│ ...          │ ...                                 │
└──────────────┴─────────────────────────────────────┘
```

### 4.2 GOT[0]/GOT[1]/GOT[2] Special Entries

**GOT[0] - .dynamic address:**
```c
// ld.so fills GOT[0] with address of .dynamic section
// ใช้สำหรับ linker bookkeeping
```

**GOT[1] - link_map:**
```c
// struct link_map {
//     ElfW(Addr)  l_addr;         // load address offset
//     char        *l_name;        // full pathname of library
//     ElfW(Dyn)   *l_ld;         // .dynamic section
//     struct link_map *l_next;    // chain to next loaded library
//     struct link_map *l_prev;    // chain to previous
//     ...
// };
//
// GOT[1] ชี้ไปที่ link_map ของ executable
// ld.so ใช้เพื่อหา symbol tables
```

**GOT[2] - _dl_runtime_resolve:**
```c
// เป็น function ใน ld.so ที่ทำ symbol resolution
// signature (roughly):
// void _dl_runtime_resolve(link_map *l, Elf64_Rela *reloc);
//
// ld.so เขียน GOT[2] ตอน initialization
// PLT[0] จะ jump มาที่นี่เมื่อต้องการ resolve symbol
```

### 4.3 การดู GOT ด้วย readelf

```bash
# ดู GOT entries
$ readelf -r program | grep -A 20 "Relocation section '.rela.plt'"

Relocation section '.rela.plt' at offset 0x4b0 contains 3 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
004004e8  00000003 R_X86_64_JUMP_SLOT  0000000000000000 puts@GLIBC_2.2.5 + 0
004004f0  00000004 R_X86_64_JUMP_SLOT  0000000000000000 printf@GLIBC_2.2.5 + 0
004004f8  00000005 R_X86_64_JUMP_SLOT  0000000000000000 exit@GLIBC_2.2.5 + 0

# Offset คือ GOT address ที่จะถูก update
# 0x4004e8 = GOT entry สำหรับ puts
```

---

## 5. Relocation Entries: RELA.PLT

### 5.1 Relocation Types

**R_X86_64_JUMP_SLOT:**
- ใช้สำหรับ PLT entries (lazy binding)
- ld.so จะเขียน address ของ function เข้า GOT entry

**R_X86_64_GLOB_DAT:**
- ใช้สำหรับ global variables
- เขียน address ของ symbol เข้า GOT

**R_X86_64_64:**
- 64-bit absolute relocation
- เขียน S + A เข้า address ที่กำหนด (S=symbol value, A=addend)

**R_X86_64_RELATIVE:**
- relative relocation สำหรับ PIE
- เขียน B + A (B=load base address)

### 5.2 Rela Structure

```c
typedef struct {
    Elf64_Addr  r_offset;   // address ที่ต้องการ modify
    Elf64_Xword r_info;     // symbol index + relocation type
    Elf64_Sxword r_addend;  // constant addend
} Elf64_Rela;

// Extract type และ symbol index:
#define ELF64_R_SYM(i)    ((i) >> 32)
#define ELF64_R_TYPE(i)   ((i) & 0xffffffffL)
```

### 5.3 ตัวอย่าง RELA.PLT entries

```
# readelf -r /bin/ls (ตัวอย่าง output)
Relocation section '.rela.plt' at offset 0x778 contains 22 entries:

  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000403fe0  000100000007 R_X86_64_JUMP_SLOT 0000000000000000 __cxa_finalize@GLIBC + 0
000000403fe8  000200000007 R_X86_64_JUMP_SLOT 0000000000000000 free@GLIBC_2.2.5 + 0
000000403ff0  000300000007 R_X86_64_JUMP_SLOT 0000000000000000 memcpy@GLIBC_2.14 + 0
...
```

---

## 6. Dynamic Linker (ld.so) Operation

### 6.1 ld.so Initialization

```
ld.so startup sequence:
1. Kernel passes control to ld.so entry point
2. ld.so ทำ self-relocation (bootstrap)
3. อ่าน environment variables:
   - LD_LIBRARY_PATH: เพิ่ม search path สำหรับ libraries
   - LD_PRELOAD: โหลด libraries ก่อน (สำหรับ hooking)
   - LD_DEBUG: enable debug output
   - LD_BIND_NOW: force eager binding
4. โหลด และ map shared libraries
5. Process relocations:
   - Eager: resolve ทุก symbols ทันที
   - Lazy: setup PLT stubs เท่านั้น
6. รัน DT_INIT functions
7. Transfer control to program entry point
```

### 6.2 Symbol Resolution Algorithm

```
เมื่อ ld.so ต้องการ resolve symbol "puts":
1. ค้นหาใน .dynsym ของ executable ก่อน
2. ถ้าไม่พบ → ค้นใน loaded libraries ตาม order:
   a. ตรวจ DT_NEEDED list ใน .dynamic
   b. โหลด library ถ้ายังไม่โหลด
   c. ค้นใน .dynsym ของแต่ละ library
3. ใช้ hash table (DT_GNU_HASH หรือ DT_HASH) เพื่อเร่งการค้นหา
4. พบแล้ว → เขียน address เข้า GOT entry
5. ถ้าไม่พบ → แจ้ง error และ abort
```

### 6.3 LD_DEBUG Output

```bash
# ดูการทำงานของ ld.so
$ LD_DEBUG=all ./program 2>&1 | head -50

     29576:     find library=libc.so.6 [0]; searching
     29576:      search cache=/etc/ld.so.cache
     29576:       trying file=/lib/x86_64-linux-gnu/libc.so.6
     29576:     calling init: /lib/x86_64-linux-gnu/libc.so.6
     29576:     initialize program: ./program
     29576:     transferring control: ./program

$ LD_DEBUG=bindings ./program 2>&1
     29577:     binding file ./program [0] to /lib/x86_64-linux-gnu/libc.so.6
               [0]: normal symbol `puts' [GLIBC_2.2.5]
```

---

## 7. LD_PRELOAD: Function Hooking

### 7.1 LD_PRELOAD Mechanism

LD_PRELOAD ช่วยให้เราโหลด shared library ก่อน libraries อื่น
ทำให้ symbols ของเราถูกค้นพบก่อน เป็น "override" ของ library functions

```bash
# syntax
$ LD_PRELOAD=/path/to/hook.so ./program

# ตัวอย่าง: hook malloc
$ LD_PRELOAD=./malloc_hook.so ./program
```

### 7.2 โปรแกรม: LD_PRELOAD Hook สำหรับ malloc

```c
// malloc_hook.c - hook malloc ด้วย LD_PRELOAD
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <stdint.h>

// ตัวแปรเก็บ pointer ไปยัง malloc จริง
static void* (*real_malloc)(size_t) = NULL;
static void  (*real_free)(void*)    = NULL;
static long  alloc_count = 0;
static long  total_bytes = 0;

// ฟังก์ชัน init - เรียกโดยอัตโนมัติเมื่อ library โหลด
__attribute__((constructor))
static void init_hooks(void) {
    // ดึง pointer ไปยัง malloc จริง
    real_malloc = dlsym(RTLD_NEXT, "malloc");
    real_free   = dlsym(RTLD_NEXT, "free");
    
    if (!real_malloc || !real_free) {
        fprintf(stderr, "Error: cannot find real malloc/free\n");
        exit(1);
    }
    
    fprintf(stderr, "[HOOK] malloc hook initialized\n");
}

// hook สำหรับ malloc
void* malloc(size_t size) {
    void* ptr = real_malloc(size);
    
    alloc_count++;
    total_bytes += size;
    
    fprintf(stderr, "[HOOK] malloc(%zu) = %p (total: %ld allocs, %ld bytes)\n",
            size, ptr, alloc_count, total_bytes);
    
    return ptr;
}

// hook สำหรับ free
void free(void* ptr) {
    if (ptr) {
        fprintf(stderr, "[HOOK] free(%p)\n", ptr);
    }
    real_free(ptr);
}

// destructor - เรียกเมื่อ library unload
__attribute__((destructor))
static void print_stats(void) {
    fprintf(stderr, "[HOOK] Final stats: %ld allocations, %ld total bytes\n",
            alloc_count, total_bytes);
}
```

### 7.3 Assembly Version ของ malloc hook

```asm
; malloc_hook_asm.asm - malloc hook ด้วย NASM Assembly
; สำหรับ Linux x86-64
;
; คอมไพล์:
;   nasm -f elf64 -o malloc_hook_asm.o malloc_hook_asm.asm
;   gcc -shared -o malloc_hook_asm.so malloc_hook_asm.o -ldl
;
; ใช้งาน:
;   LD_PRELOAD=./malloc_hook_asm.so ./program

section .data
    ; ข้อความสำหรับแสดงผล
    msg_init    db "[ASM HOOK] malloc hook loaded", 10, 0
    msg_malloc  db "[ASM HOOK] malloc(%zu) called", 10, 0
    msg_free    db "[ASM HOOK] free(%p) called", 10, 0
    msg_stats   db "[ASM HOOK] Stats: allocs=%ld bytes=%ld", 10, 0
    
    ; ชื่อ symbol สำหรับ dlsym
    sym_malloc  db "malloc", 0
    sym_free    db "free", 0

section .bss
    ; ตัวแปรเก็บ original functions
    real_malloc_ptr resq 1  ; pointer ไปยัง malloc จริง
    real_free_ptr   resq 1  ; pointer ไปยัง free จริง
    alloc_count     resq 1  ; จำนวน allocations
    total_bytes     resq 1  ; total bytes allocated

section .text
    global malloc
    global free
    extern dlsym
    extern fprintf
    extern stderr

; ค่า constant สำหรับ dlsym
RTLD_NEXT equ -1    ; ค้นหา next occurrence ใน library chain

;=============================================================================
; init_hooks - constructor function
; เรียกโดยอัตโนมัติเมื่อ shared library โหลด
;=============================================================================
init_hooks:
    ; บันทึก caller-saved registers
    push rbx
    push rbp
    sub  rsp, 8     ; align stack to 16 bytes
    
    ; ดึง pointer ไปยัง real malloc
    ; dlsym(RTLD_NEXT, "malloc")
    mov  rdi, RTLD_NEXT     ; handle = RTLD_NEXT
    lea  rsi, [rel sym_malloc]  ; symbol name = "malloc"
    call dlsym
    
    ; ตรวจสอบว่าพบ
    test rax, rax
    jz   .init_error
    
    ; บันทึก pointer
    mov  [rel real_malloc_ptr], rax
    
    ; ดึง pointer ไปยัง real free
    ; dlsym(RTLD_NEXT, "free")
    mov  rdi, RTLD_NEXT
    lea  rsi, [rel sym_free]
    call dlsym
    
    test rax, rax
    jz   .init_error
    
    mov  [rel real_free_ptr], rax
    
    ; แสดงข้อความว่า hook โหลดแล้ว
    mov  rdi, [rel stderr]
    lea  rsi, [rel msg_init]
    xor  eax, eax
    call fprintf
    
    add  rsp, 8
    pop  rbp
    pop  rbx
    ret

.init_error:
    ; error handling - ออกจากโปรแกรม
    mov  rdi, 1
    mov  rax, 60    ; syscall exit
    syscall

;=============================================================================
; malloc - hooked version
; Input:  rdi = size (bytes to allocate)
; Output: rax = pointer to allocated memory (NULL on failure)
;=============================================================================
malloc:
    ; บันทึก argument และ registers
    push rbx
    push r12
    push r13
    sub  rsp, 8
    
    mov  r12, rdi   ; บันทึก size ไว้ใน r12
    
    ; เรียก real malloc
    ; real_malloc_ptr มี address ของ malloc จริง
    mov  rax, [rel real_malloc_ptr]
    
    ; ตรวจว่า hook initialized แล้ว
    test rax, rax
    jz   .malloc_not_initialized
    
    ; เรียก real malloc(size)
    ; rdi ยังมี size อยู่
    call rax
    
    mov  r13, rax   ; บันทึก return value (pointer)
    
    ; อัพเดต counters
    inc  qword [rel alloc_count]
    add  qword [rel total_bytes], r12
    
    ; แสดง log
    mov  rdi, [rel stderr]
    lea  rsi, [rel msg_malloc]
    mov  rdx, r12       ; size
    xor  eax, eax
    call fprintf
    
    ; คืนค่า pointer
    mov  rax, r13
    
    add  rsp, 8
    pop  r13
    pop  r12
    pop  rbx
    ret

.malloc_not_initialized:
    ; ถ้า hook ยังไม่ initialize ให้ทำ init ก่อน
    call init_hooks
    
    ; เรียก malloc ใหม่
    mov  rdi, r12
    add  rsp, 8
    pop  r13
    pop  r12
    pop  rbx
    jmp  malloc

;=============================================================================
; free - hooked version
; Input:  rdi = pointer to free
;=============================================================================
free:
    push rbx
    push r12
    sub  rsp, 8
    
    mov  r12, rdi   ; บันทึก pointer
    
    ; แสดง log (ถ้า pointer ไม่ใช่ NULL)
    test rdi, rdi
    jz   .free_null
    
    mov  rdi, [rel stderr]
    lea  rsi, [rel msg_free]
    mov  rdx, r12
    xor  eax, eax
    call fprintf

.free_null:
    ; เรียก real free
    mov  rax, [rel real_free_ptr]
    test rax, rax
    jz   .free_done
    
    mov  rdi, r12
    call rax

.free_done:
    add  rsp, 8
    pop  r12
    pop  rbx
    ret

;=============================================================================
; print_stats - destructor function
; เรียกเมื่อ library unload
;=============================================================================
print_stats:
    push rbx
    sub  rsp, 8
    
    mov  rdi, [rel stderr]
    lea  rsi, [rel msg_stats]
    mov  rdx, [rel alloc_count]
    mov  rcx, [rel total_bytes]
    xor  eax, eax
    call fprintf
    
    add  rsp, 8
    pop  rbx
    ret

;=============================================================================
; Constructor/Destructor tables
; ld.so จะเรียก functions ใน section นี้
;=============================================================================
section .init_array
    dq init_hooks   ; เรียก init_hooks เมื่อ load

section .fini_array
    dq print_stats  ; เรียก print_stats เมื่อ unload
```

---

## 8. dlopen/dlsym/dlclose

### 8.1 Dynamic Loading API

```c
#include <dlfcn.h>

// เปิด shared library
void* dlopen(const char *filename, int flags);

// ค้นหา symbol
void* dlsym(void *handle, const char *symbol);

// ปิด library
int dlclose(void *handle);

// ดู error
char* dlerror(void);
```

### 8.2 RTLD_LAZY vs RTLD_NOW

```c
RTLD_LAZY:
  - Resolve symbols เมื่อต้องการใช้เท่านั้น
  - เร็วกว่าใน load time
  - ถ้า symbol ไม่พบ จะ error เมื่อเรียกใช้ครั้งแรก
  - ใช้เมื่อ: performance สำคัญ หรือไม่แน่ว่าจะใช้ทุก function

RTLD_NOW:
  - Resolve ทุก symbols ทันทีที่ dlopen
  - ช้ากว่าใน load time
  - ถ้า symbol ไม่พบ จะ error ที่ dlopen
  - ใช้เมื่อ: ต้องการ fail fast หรือด้านความปลอดภัย

RTLD_GLOBAL:
  - Symbols ของ library นี้จะถูก export ให้ libraries อื่นเห็น
  - ใช้เมื่อ: library มี plugins ที่ต้องการ symbols จาก library นี้

RTLD_LOCAL:
  - Symbols จะไม่ถูก export (default)
  - ความปลอดภัยสูงกว่า
```

### 8.3 โปรแกรม: Plugin System ด้วย dlopen

```c
// plugin_host.c - ระบบ plugin ที่ใช้ dlopen
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <string.h>

// Interface ของ plugin - ต้อง implement
typedef struct {
    const char* name;
    const char* version;
    int (*init)(void);
    int (*process)(const char* input, char* output, size_t out_size);
    void (*cleanup)(void);
} PluginInterface;

// โหลดและรัน plugin
int load_and_run_plugin(const char* plugin_path, const char* input) {
    char output[1024] = {0};
    
    // เปิด plugin
    void* handle = dlopen(plugin_path, RTLD_LAZY | RTLD_LOCAL);
    if (!handle) {
        fprintf(stderr, "Error loading plugin: %s\n", dlerror());
        return -1;
    }
    
    // Clear error state
    dlerror();
    
    // ค้นหา get_plugin_interface function
    PluginInterface* (*get_interface)(void);
    *(void**)(&get_interface) = dlsym(handle, "get_plugin_interface");
    
    char* err = dlerror();
    if (err) {
        fprintf(stderr, "Error finding interface: %s\n", err);
        dlclose(handle);
        return -1;
    }
    
    // ดึง interface
    PluginInterface* plugin = get_interface();
    if (!plugin) {
        fprintf(stderr, "Plugin returned NULL interface\n");
        dlclose(handle);
        return -1;
    }
    
    printf("Loaded plugin: %s v%s\n", plugin->name, plugin->version);
    
    // Initialize plugin
    if (plugin->init() != 0) {
        fprintf(stderr, "Plugin init failed\n");
        dlclose(handle);
        return -1;
    }
    
    // Process
    if (plugin->process(input, output, sizeof(output)) != 0) {
        fprintf(stderr, "Plugin process failed\n");
    } else {
        printf("Plugin output: %s\n", output);
    }
    
    // Cleanup
    plugin->cleanup();
    
    // ปิด plugin
    dlclose(handle);
    return 0;
}

int main(int argc, char* argv[]) {
    if (argc < 3) {
        printf("Usage: %s <plugin.so> <input>\n", argv[0]);
        return 1;
    }
    
    return load_and_run_plugin(argv[1], argv[2]);
}
```

### 8.4 ตัวอย่าง Plugin (C)

```c
// reverse_plugin.c - plugin ที่ reverse string
#include <string.h>
#include <stdio.h>

typedef struct {
    const char* name;
    const char* version;
    int (*init)(void);
    int (*process)(const char* input, char* output, size_t out_size);
    void (*cleanup)(void);
} PluginInterface;

static int plugin_init(void) {
    printf("[Plugin] Reverse plugin initialized\n");
    return 0;
}

static int plugin_process(const char* input, char* output, size_t out_size) {
    size_t len = strlen(input);
    if (len >= out_size) return -1;
    
    for (size_t i = 0; i < len; i++) {
        output[i] = input[len - 1 - i];
    }
    output[len] = '\0';
    return 0;
}

static void plugin_cleanup(void) {
    printf("[Plugin] Reverse plugin cleanup\n");
}

static PluginInterface interface = {
    .name    = "ReversePlugin",
    .version = "1.0.0",
    .init    = plugin_init,
    .process = plugin_process,
    .cleanup = plugin_cleanup,
};

PluginInterface* get_plugin_interface(void) {
    return &interface;
}
```

### 8.5 Assembly: dlopen/dlsym Usage

```asm
; dlopen_demo.asm - ใช้ dlopen/dlsym ใน Assembly
; คอมไพล์:
;   nasm -f elf64 -o dlopen_demo.o dlopen_demo.asm
;   gcc -o dlopen_demo dlopen_demo.o -ldl
;
; โปรแกรมนี้ load libm.so แล้วเรียก sqrt() ผ่าน dlsym

section .data
    ; ชื่อ library
    libm_path   db "/lib/x86_64-linux-gnu/libm.so.6", 0
    libm_name   db "libm.so.6", 0    ; short name
    
    ; ชื่อ function
    sym_sqrt    db "sqrt", 0
    sym_pow     db "pow", 0
    sym_sin     db "sin", 0
    
    ; format strings
    fmt_loaded  db "Library loaded: handle=%p", 10, 0
    fmt_sqrt    db "sqrt(%f) = %f", 10, 0
    fmt_pow     db "pow(%f, %f) = %f", 10, 0
    fmt_error   db "Error: %s", 10, 0
    fmt_closed  db "Library closed successfully", 10, 0
    
    ; ค่าตัวเลขสำหรับทดสอบ
    val_4       dq 4.0     ; sqrt(4) = 2.0
    val_2       dq 2.0     ; pow(2, 10) = 1024.0
    val_10      dq 10.0

section .bss
    handle_ptr  resq 1     ; เก็บ dlopen handle
    sqrt_func   resq 1     ; เก็บ pointer ไปยัง sqrt
    pow_func    resq 1     ; เก็บ pointer ไปยัง pow

section .text
    global main
    extern dlopen
    extern dlsym
    extern dlclose
    extern dlerror
    extern printf
    extern puts

; RTLD flags
RTLD_LAZY   equ 0x1
RTLD_NOW    equ 0x2
RTLD_GLOBAL equ 0x100
RTLD_LOCAL  equ 0x0

main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 32        ; stack frame
    
    ;------------------------------------------------------------------
    ; Step 1: โหลด libm.so ด้วย dlopen
    ;------------------------------------------------------------------
    lea  rdi, [rel libm_path]    ; filename
    mov  rsi, RTLD_LAZY          ; flags
    call dlopen
    
    ; ตรวจสอบ error
    test rax, rax
    jz   .error_open
    
    ; บันทึก handle
    mov  [rel handle_ptr], rax
    
    ; แสดง handle
    mov  rdi, fmt_loaded
    lea  rdi, [rel fmt_loaded]
    mov  rsi, rax
    xor  eax, eax
    call printf
    
    ;------------------------------------------------------------------
    ; Step 2: ดึง sqrt function ด้วย dlsym
    ;------------------------------------------------------------------
    mov  rdi, [rel handle_ptr]   ; handle
    lea  rsi, [rel sym_sqrt]     ; symbol name
    call dlsym
    
    test rax, rax
    jz   .error_sym
    
    mov  [rel sqrt_func], rax    ; บันทึก sqrt pointer
    
    ;------------------------------------------------------------------
    ; Step 3: เรียก sqrt(4.0)
    ;------------------------------------------------------------------
    movsd xmm0, [rel val_4]     ; argument: 4.0
    mov   rax, [rel sqrt_func]  ; load function pointer
    call  rax                   ; call sqrt(4.0) → result in xmm0
    
    ; แสดงผล
    lea   rdi, [rel fmt_sqrt]
    movsd xmm0, [rel val_4]     ; first arg: 4.0
    ; xmm1 มี result จาก sqrt แล้ว - แต่ต้อง save ก่อน call sqrt
    ; ทำใหม่ด้วยการ save result
    push  rax                   ; save
    movsd xmm1, xmm0            ; result อยู่ใน xmm0 จาก sqrt
    movsd xmm0, [rel val_4]     ; restore first arg for printf
    mov   eax, 2                ; 2 floating point args
    call  printf
    pop   rax
    
    ;------------------------------------------------------------------
    ; Step 4: ดึง pow function
    ;------------------------------------------------------------------
    mov  rdi, [rel handle_ptr]
    lea  rsi, [rel sym_pow]
    call dlsym
    
    test rax, rax
    jz   .error_sym
    
    mov  [rel pow_func], rax
    
    ;------------------------------------------------------------------
    ; Step 5: เรียก pow(2.0, 10.0)
    ;------------------------------------------------------------------
    movsd xmm0, [rel val_2]     ; base = 2.0
    movsd xmm1, [rel val_10]    ; exponent = 10.0
    mov   rax, [rel pow_func]
    call  rax                   ; pow(2, 10) → xmm0
    
    ; แสดงผล
    movsd xmm2, xmm0            ; result
    movsd xmm0, [rel val_2]     ; base
    movsd xmm1, [rel val_10]    ; exponent
    lea   rdi, [rel fmt_pow]
    mov   eax, 3
    call  printf
    
    ;------------------------------------------------------------------
    ; Step 6: ปิด library ด้วย dlclose
    ;------------------------------------------------------------------
    mov  rdi, [rel handle_ptr]
    call dlclose
    
    ; ตรวจสอบ result (0 = success)
    test eax, eax
    jnz  .error_close
    
    lea  rdi, [rel fmt_closed]
    call puts
    
    ; return 0
    xor  eax, eax
    leave
    ret

.error_open:
    call dlerror
    test rax, rax
    jz   .no_error_msg
    lea  rdi, [rel fmt_error]
    mov  rsi, rax
    xor  eax, eax
    call printf
.no_error_msg:
    mov  eax, 1
    leave
    ret

.error_sym:
    call dlerror
    test rax, rax
    jz   .no_error_msg2
    lea  rdi, [rel fmt_error]
    mov  rsi, rax
    xor  eax, eax
    call printf
.no_error_msg2:
    ; ปิด handle ก่อน exit
    mov  rdi, [rel handle_ptr]
    call dlclose
    mov  eax, 1
    leave
    ret

.error_close:
    call dlerror
    lea  rdi, [rel fmt_error]
    mov  rsi, rax
    xor  eax, eax
    call printf
    mov  eax, 1
    leave
    ret
```

---

## 9. Shared Library (.so) Creation

### 9.1 การสร้าง Shared Library

```bash
# วิธีสร้าง shared library จาก C:
gcc -fPIC -c mylib.c -o mylib.o      # compile with PIC
gcc -shared -o libmylib.so mylib.o   # create shared lib

# วิธีสร้างจาก Assembly:
nasm -f elf64 mylib.asm -o mylib.o
gcc -shared -o libmylib.so mylib.o

# ระบุ SONAME:
gcc -shared -Wl,-soname,libmylib.so.1 -o libmylib.so.1.0 mylib.o

# Install shared library:
cp libmylib.so /usr/local/lib/
ldconfig  # update ld cache
```

### 9.2 โปรแกรม: Creating a Shared Library with Assembly

```asm
; mathlib.asm - Shared library ที่สร้างด้วย Assembly
; ฟังก์ชัน: add, subtract, multiply, factorial, fibonacci
;
; คอมไพล์:
;   nasm -f elf64 -o mathlib.o mathlib.asm
;   gcc -shared -fPIC -o libmathasm.so mathlib.o
;
; ใช้งาน:
;   gcc -o program main.c -L. -lmathasm -Wl,-rpath,.
;   ./program

section .data
    ; ข้อความสำหรับ error
    err_overflow  db "Factorial overflow!", 10, 0
    err_negative  db "Error: negative input", 10, 0

section .text
    ; Export ฟังก์ชัน global
    global asm_add
    global asm_subtract
    global asm_multiply
    global asm_divide
    global asm_factorial
    global asm_fibonacci
    global asm_gcd
    global asm_is_prime

;=============================================================================
; long asm_add(long a, long b)
; บวก a + b แล้วคืนผล
;=============================================================================
asm_add:
    mov  rax, rdi   ; rax = a
    add  rax, rsi   ; rax = a + b
    ret

;=============================================================================
; long asm_subtract(long a, long b)
; ลบ a - b
;=============================================================================
asm_subtract:
    mov  rax, rdi   ; rax = a
    sub  rax, rsi   ; rax = a - b
    ret

;=============================================================================
; long asm_multiply(long a, long b)
; คูณ a * b
;=============================================================================
asm_multiply:
    mov  rax, rdi   ; rax = a
    imul rax, rsi   ; rax = a * b (signed multiply)
    ret

;=============================================================================
; long asm_divide(long a, long b)
; หาร a / b (integer division)
; คืน -1 ถ้า b == 0
;=============================================================================
asm_divide:
    ; ตรวจสอบ divide by zero
    test rsi, rsi
    jz   .div_zero
    
    mov  rax, rdi   ; rax = a (dividend)
    cqo             ; sign-extend rax into rdx:rax
    idiv rsi        ; rdx:rax / rsi → quotient in rax, remainder in rdx
    ret

.div_zero:
    mov  rax, -1    ; error: return -1
    ret

;=============================================================================
; long asm_factorial(int n)
; คำนวณ n! iteratively
; Input:  rdi = n (0 <= n <= 20)
; Output: rax = n! หรือ -1 ถ้า overflow
;=============================================================================
asm_factorial:
    ; ตรวจสอบ input
    test rdi, rdi
    js   .fact_negative      ; n < 0 → error
    
    ; 0! = 1, 1! = 1
    cmp  rdi, 1
    jle  .fact_one
    
    ; ตรวจสอบ overflow (n > 20 จะ overflow ใน 64-bit)
    cmp  rdi, 20
    jg   .fact_overflow
    
    mov  rax, 1     ; result = 1
    mov  rcx, rdi   ; counter = n

.fact_loop:
    ; result *= counter
    imul rax, rcx
    
    ; ตรวจสอบ overflow
    jo   .fact_overflow
    
    dec  rcx
    cmp  rcx, 1
    jg   .fact_loop
    
    ret

.fact_one:
    mov  rax, 1
    ret

.fact_negative:
.fact_overflow:
    mov  rax, -1    ; error
    ret

;=============================================================================
; long asm_fibonacci(int n)
; คำนวณ Fibonacci number ที่ n (iterative)
; Input:  rdi = n (0-indexed: fib(0)=0, fib(1)=1, fib(2)=1, ...)
; Output: rax = fib(n)
;=============================================================================
asm_fibonacci:
    ; F(0) = 0
    test rdi, rdi
    jz   .fib_zero
    
    ; F(1) = 1
    cmp  rdi, 1
    je   .fib_one
    
    mov  rax, 0     ; a = F(0) = 0
    mov  rcx, 1     ; b = F(1) = 1
    mov  rdx, 2     ; counter = 2

.fib_loop:
    ; temp = a + b
    mov  r8, rax
    add  r8, rcx    ; r8 = a + b = next Fibonacci
    
    mov  rax, rcx   ; a = b (เลื่อน window ไปข้างหน้า)
    mov  rcx, r8    ; b = a + b
    
    inc  rdx
    cmp  rdx, rdi
    jle  .fib_loop
    
    ; result อยู่ใน rcx
    mov  rax, rcx
    ret

.fib_zero:
    xor  rax, rax   ; return 0
    ret

.fib_one:
    mov  rax, 1     ; return 1
    ret

;=============================================================================
; long asm_gcd(long a, long b)
; หา Greatest Common Divisor ด้วย Euclidean algorithm
; Input:  rdi = a, rsi = b
; Output: rax = gcd(a, b)
;=============================================================================
asm_gcd:
    ; ถ้า b == 0, return a
    test rsi, rsi
    jz   .gcd_done
    
    ; Euclidean: gcd(a, b) = gcd(b, a mod b)
    mov  rax, rdi    ; rax = a
    cqo              ; sign extend
    idiv rsi         ; rdx = a mod b
    
    mov  rdi, rsi    ; a = b
    mov  rsi, rdx    ; b = a mod b
    jmp  asm_gcd     ; recursive (tail call)

.gcd_done:
    mov  rax, rdi
    ret

;=============================================================================
; int asm_is_prime(long n)
; ตรวจว่า n เป็นจำนวนเฉพาะหรือไม่
; Input:  rdi = n
; Output: rax = 1 ถ้า prime, 0 ถ้าไม่ใช่
;=============================================================================
asm_is_prime:
    ; n < 2 ไม่ใช่ prime
    cmp  rdi, 2
    jl   .not_prime
    
    ; 2 เป็น prime
    je   .is_prime
    
    ; เลขคู่ไม่ใช่ prime
    test rdi, 1     ; ตรวจ bit 0
    jz   .not_prime
    
    ; ทดสอบตัวหาร 3, 5, 7, ... ≤ sqrt(n)
    mov  rcx, 3     ; divisor = 3

.prime_loop:
    ; ถ้า divisor * divisor > n → prime
    mov  rax, rcx
    imul rax, rcx   ; rax = divisor^2
    cmp  rax, rdi   ; divisor^2 > n?
    jg   .is_prime
    
    ; ทดสอบ n mod divisor
    mov  rax, rdi
    cqo
    idiv rcx        ; rdx = n mod divisor
    
    test rdx, rdx   ; remainder == 0?
    jz   .not_prime ; หารลงตัว → ไม่ใช่ prime
    
    add  rcx, 2     ; ข้ามเลขคู่: 3,5,7,9,...
    jmp  .prime_loop

.is_prime:
    mov  rax, 1
    ret

.not_prime:
    xor  rax, rax
    ret
```

### 9.3 Program ที่ใช้ Shared Library

```c
// main_mathlib.c - ใช้งาน mathlib shared library
#include <stdio.h>

// ประกาศ functions จาก libmathasm.so
extern long asm_add(long a, long b);
extern long asm_subtract(long a, long b);
extern long asm_multiply(long a, long b);
extern long asm_divide(long a, long b);
extern long asm_factorial(int n);
extern long asm_fibonacci(int n);
extern long asm_gcd(long a, long b);
extern int  asm_is_prime(long n);

int main(void) {
    printf("=== Assembly Math Library Demo ===\n\n");
    
    // ทดสอบ basic operations
    printf("Basic Operations:\n");
    printf("  add(10, 20)      = %ld\n", asm_add(10, 20));
    printf("  subtract(100, 37) = %ld\n", asm_subtract(100, 37));
    printf("  multiply(6, 7)   = %ld\n", asm_multiply(6, 7));
    printf("  divide(100, 7)   = %ld\n", asm_divide(100, 7));
    printf("\n");
    
    // ทดสอบ factorial
    printf("Factorial:\n");
    for (int i = 0; i <= 10; i++) {
        printf("  %2d! = %ld\n", i, asm_factorial(i));
    }
    printf("\n");
    
    // ทดสอบ Fibonacci
    printf("Fibonacci sequence (first 15):\n");
    printf("  ");
    for (int i = 0; i < 15; i++) {
        printf("%ld ", asm_fibonacci(i));
    }
    printf("\n\n");
    
    // ทดสอบ GCD
    printf("GCD:\n");
    printf("  gcd(48, 18) = %ld\n", asm_gcd(48, 18));
    printf("  gcd(100, 75) = %ld\n", asm_gcd(100, 75));
    printf("\n");
    
    // ทดสอบ prime
    printf("Prime numbers up to 50:\n  ");
    for (long n = 2; n <= 50; n++) {
        if (asm_is_prime(n)) {
            printf("%ld ", n);
        }
    }
    printf("\n");
    
    return 0;
}
```

---

## 10. Position Independent Code (PIC)

### 10.1 แนวคิด PIC

Position Independent Code คือ code ที่สามารถ execute ได้ที่ memory address ใดก็ได้
โดยไม่ต้อง modify code เอง สิ่งสำคัญคือต้องไม่ใช้ absolute addresses

```asm
; Non-PIC code (ไม่ดี สำหรับ shared library)
mov rax, [0x601020]     ; absolute address - ใช้ไม่ได้ใน .so

; PIC code (ถูกต้อง)
mov rax, [rel variable] ; RIP-relative: ใช้ offset จาก RIP
; หรือ
lea rax, [rip + var]    ; เหมือนกัน
```

### 10.2 RIP-Relative Addressing

```asm
; RIP-relative addressing ใน x86-64:
; Instruction encoding ใช้ 32-bit signed offset จาก RIP
;
; [rip + offset] หมายถึง: address = current_RIP + offset
; โดย RIP หลัง instruction ถูก execute

; ตัวอย่าง:
section .data
    my_var dq 42

section .text
    ; อ่านค่า my_var ด้วย RIP-relative
    mov rax, [rel my_var]     ; NASM syntax
    mov rax, [rip + my_var]   ; explicit (หมายถึงเหมือนกัน)
    
    ; ดู address ของ string ด้วย LEA + RIP-relative
    lea rsi, [rel my_string]  ; RIP-relative address
```

### 10.3 Global Variables ใน PIC: GOT Access

```asm
; การเข้าถึง global variable ใน PIC shared library
; ต้องผ่าน GOT เพราะ address ของ global อาจ relocate ได้
;
; Compiler-generated code สำหรับ: int x = global_var;

; Non-PIC (bad for .so):
;   mov eax, [global_var]    ; absolute address

; PIC via GOT:
;   lea rax, [rip + global_var@GOTPCREL]  ; get GOT entry address
;   mov eax, [rax]                         ; dereference GOT → actual address
;   mov eax, [rax]                         ; get value

; ใน NASM สำหรับ shared library:
section .data
    my_counter dd 0     ; global variable

section .text
global increment_counter
increment_counter:
    ; PIC: ใช้ RIP-relative สำหรับ local symbols
    ; (ไม่ต้องผ่าน GOT สำหรับ local symbols)
    add dword [rel my_counter], 1
    mov eax, [rel my_counter]
    ret
```

### 10.4 -fPIC Flag

```bash
# compile ด้วย -fPIC (Position Independent Code)
gcc -fPIC -c source.c -o source.o

# ความแตกต่าง:
# -fPIC: สำหรับ shared libraries ทุกประเภท
# -fpic: เหมือน -fPIC แต่ restrictive กว่า (limit GOT size)
# ไม่มี flag: สำหรับ executables เท่านั้น

# ดู object file ว่าใช้ PIC ไหม:
readelf -d libmylib.so | grep PIC
objdump -d libmylib.so | grep -i "rip"
```

### 10.5 โปรแกรม: PIC Assembly Library

```asm
; pic_library.asm - ตัวอย่าง PIC assembly code สำหรับ shared library
; แสดงวิธีการเข้าถึง data ด้วย RIP-relative addressing
;
; คอมไพล์:
;   nasm -f elf64 -o pic_library.o pic_library.asm
;   gcc -shared -o libpic_demo.so pic_library.o

section .data
    ; Local data - ใช้ RIP-relative ได้โดยตรง
    counter     dq 0
    hello_msg   db "Hello from PIC library!", 10, 0
    
    ; Format strings
    fmt_counter db "Counter: %ld", 10, 0
    fmt_addr    db "Function at: %p", 10, 0

section .rodata
    version_str db "PIC Demo Library v1.0", 0

section .text
    global pic_increment
    global pic_get_counter
    global pic_reset_counter
    global pic_get_version
    global pic_print_hello
    extern printf
    extern puts

;=============================================================================
; void pic_increment(long amount)
; เพิ่มค่า counter ด้วย RIP-relative addressing
;=============================================================================
pic_increment:
    ; ใช้ RIP-relative เพื่อเข้าถึง counter
    ; [rel counter] = [rip + offset_to_counter]
    ; offset ถูกคำนวณ ณ link time แต่ valid สำหรับทุก load address
    add  qword [rel counter], rdi
    ret

;=============================================================================
; long pic_get_counter(void)
; คืนค่า counter ปัจจุบัน
;=============================================================================
pic_get_counter:
    mov  rax, [rel counter]
    ret

;=============================================================================
; void pic_reset_counter(void)
; Reset counter เป็น 0
;=============================================================================
pic_reset_counter:
    mov  qword [rel counter], 0
    ret

;=============================================================================
; const char* pic_get_version(void)
; คืน pointer ไปยัง version string (RIP-relative)
;=============================================================================
pic_get_version:
    ; ใช้ LEA เพื่อได้ address ของ string ใน RIP-relative fashion
    lea  rax, [rel version_str]
    ret

;=============================================================================
; void pic_print_hello(void)
; แสดงข้อความ พร้อม address ของ function
;=============================================================================
pic_print_hello:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16
    
    ; แสดง hello message
    lea  rdi, [rel hello_msg]
    call puts
    
    ; แสดง current counter
    lea  rdi, [rel fmt_counter]
    mov  rsi, [rel counter]
    xor  eax, eax
    call printf
    
    ; แสดง address ของ function นี้ (PIC: address จะต่างกันทุกครั้ง)
    lea  rdi, [rel fmt_addr]
    lea  rsi, [rel pic_print_hello]
    xor  eax, eax
    call printf
    
    leave
    ret
```

---

## 11. RELRO: Partial vs Full RELRO

### 11.1 RELRO คืออะไร

RELRO (RELocation Read-Only) เป็น security mitigation ที่ทำให้ memory regions
ที่มี relocation data กลายเป็น read-only หลังจาก dynamic linker ทำงานเสร็จ

เป้าหมาย: ป้องกัน GOT overwrite attack

```bash
# ตรวจสอบ RELRO ของ binary:
$ checksec --file=./program
    Arch:     amd64-64-little
    RELRO:    Partial RELRO    ← หรือ Full RELRO หรือ No RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled

# หรือใช้ readelf:
$ readelf -l program | grep GNU_RELRO
  GNU_RELRO      0x002df4 0x0000000000603df4 0x0000000000603df4
                 0x0000000000000020c 0x0000000000000020c  R   0x1
```

### 11.2 Partial RELRO

```bash
# compile ด้วย Partial RELRO (default ใน GCC):
gcc -Wl,-z,relro program.c -o program

# ผล:
# - .got section (global variables) → read-only ✓
# - .got.plt section (function pointers) → WRITABLE ✗
#   (ต้องยังเป็น writable เพราะ lazy binding)
# - .dynamic section → read-only ✓
# - .ctors/.dtors → read-only ✓
```

### 11.3 Full RELRO

```bash
# compile ด้วย Full RELRO:
gcc -Wl,-z,relro,-z,now program.c -o program

# ผล:
# - .got section → read-only ✓
# - .got.plt section → read-only ✓  ← สิ่งสำคัญ!
# - Lazy binding ถูก disable
# - ld.so resolve ทุก symbols ตอน startup (eager binding)
# - ประสิทธิภาพ startup ต่ำกว่าเล็กน้อย
# - ปลอดภัยกว่า: GOT ไม่สามารถ overwrite ได้
```

### 11.4 GOT Overwrite Attack (ตัวอย่างเพื่อการศึกษา)

```
GOT Overwrite Attack:
1. โปรแกรมมีช่องโหว่ write primitive (เขียน arbitrary address ได้)
2. Attacker เขียน address ของ evil function เข้า GOT[puts]
3. เมื่อโปรแกรมเรียก puts → ตรง→ evil function แทน

ป้องกันด้วย Full RELRO:
  - GOT เป็น read-only หลัง ld.so setup
  - Write ไปที่ GOT จะ segfault
```

### 11.5 Memory Map ของ Partial vs Full RELRO

```
Partial RELRO memory layout:
0x400000 - .text     (r-x: execute only)
0x401000 - .rodata   (r--: read only)
0x402000 - .got      (r--: read only ← protected)
0x402020 - .got.plt  (rw-: read-write ← vulnerable!)
0x402040 - .data     (rw-)
0x402060 - .bss      (rw-)

Full RELRO memory layout:
0x400000 - .text     (r-x)
0x401000 - .rodata   (r--)
0x402000 - .got      (r--: read only)
0x402020 - .got.plt  (r--: read only ← protected!)
0x402040 - .data     (rw-)
0x402060 - .bss      (rw-)
```

---

## 12. PLT/GOT Patching at Runtime

### 12.1 การแก้ไข GOT ณ Runtime

เป็นเทคนิคที่ใช้ใน:
- Debugging/tracing
- Hot-patching (แก้ bug โดยไม่ restart)
- Instrumentation

**หมายเหตุ**: เทคนิคนี้จะไม่ทำงานกับ Full RELRO

```c
// got_patch.c - patching GOT entry at runtime
// ใช้ Partial RELRO เท่านั้น!
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <stdint.h>
#include <string.h>

// hook function สำหรับแทนที่ puts
int my_puts_hook(const char* s) {
    printf("[HOOKED] puts called with: \"%s\"\n", s);
    return 0;
}

// ดึง GOT entry address ของ function
// สำหรับ demo: ใช้วิธีง่ายๆ (production ต้องใช้ parser ที่ซับซ้อนกว่า)
void* get_got_entry(void* plt_entry) {
    // PLT stub: ff 25 XX XX XX XX (JMP [RIP+offset])
    uint8_t* plt = (uint8_t*)plt_entry;
    
    // ตรวจว่าเป็น JMP [RIP+offset] (ff 25)
    if (plt[0] != 0xff || plt[1] != 0x25) {
        fprintf(stderr, "Not a PLT stub\n");
        return NULL;
    }
    
    // อ่าน 32-bit offset (little-endian)
    int32_t offset;
    memcpy(&offset, plt + 2, 4);
    
    // GOT address = address after instruction (plt+6) + offset
    return (void*)((uintptr_t)(plt + 6) + offset);
}

// Patch GOT entry
int patch_got_entry(void* got_entry_ptr, void* new_func) {
    // ทำให้ memory เป็น writable ชั่วคราว
    uintptr_t page_addr = (uintptr_t)got_entry_ptr & ~(4095UL);
    
    if (mprotect((void*)page_addr, 4096, PROT_READ | PROT_WRITE) != 0) {
        perror("mprotect failed (Full RELRO?)");
        return -1;
    }
    
    // เขียน new function address เข้า GOT
    *(void**)got_entry_ptr = new_func;
    
    // restore permissions (optional: security best practice)
    mprotect((void*)page_addr, 4096, PROT_READ);
    
    return 0;
}

extern int puts(const char* s);  // declares puts

int main(void) {
    // ก่อน patch
    puts("Before GOT patch - this goes to real puts");
    
    // ดึง GOT entry address
    // puts@plt address อยู่ที่ puts function pointer
    void* got_entry = get_got_entry((void*)puts);
    if (!got_entry) {
        fprintf(stderr, "Cannot find GOT entry\n");
        return 1;
    }
    
    printf("GOT entry for puts at: %p\n", got_entry);
    printf("Current GOT value: %p\n", *(void**)got_entry);
    
    // Patch GOT: แทนที่ puts ด้วย hook ของเรา
    if (patch_got_entry(got_entry, (void*)my_puts_hook) != 0) {
        fprintf(stderr, "Patch failed\n");
        return 1;
    }
    
    printf("GOT patched! New value: %p\n", *(void**)got_entry);
    
    // หลัง patch - puts ควรเรียก hook แทน
    puts("After GOT patch - goes to hook");
    puts("Second call - also hooked");
    
    return 0;
}
```

### 12.2 Assembly PLT/GOT Patching

```asm
; plt_got_patch.asm - GOT patching ใน Assembly
; โปรแกรมนี้แสดงการอ่านและแก้ไข GOT entries
;
; คอมไพล์ด้วย Partial RELRO (ไม่ใส่ -z now):
;   nasm -f elf64 -o plt_got_patch.o plt_got_patch.asm
;   gcc -Wl,-z,relro -o plt_got_patch plt_got_patch.o
;   (อย่าใส่ -z now)

section .data
    ; ข้อความทดสอบ
    msg_before  db "Before patch: calling puts normally", 10, 0
    msg_after   db "After patch: this calls our hook!", 10, 0
    msg_hooked  db "[HOOK ACTIVE] puts was called!", 10, 0
    msg_info    db "PLT addr: %p | GOT addr: %p | GOT val: %p", 10, 0
    fmt_ptr     db "Address: %p", 10, 0

section .bss
    saved_puts  resq 1  ; เก็บ original puts address

section .text
    global main
    extern puts
    extern printf
    extern mprotect

PAGE_SIZE equ 4096
PROT_READ  equ 1
PROT_WRITE equ 2
PROT_EXEC  equ 4

;=============================================================================
; hook_puts - replacement สำหรับ puts
; เราจะ patch GOT[puts] ให้ชี้มาที่นี่
;=============================================================================
hook_puts:
    push rbp
    mov  rbp, rsp
    push rbx
    sub  rsp, 8
    
    mov  rbx, rdi   ; บันทึก argument (string pointer)
    
    ; แสดงข้อความว่า hook ถูกเรียก
    lea  rdi, [rel msg_hooked]
    call puts_real  ; เรียก real puts ผ่าน saved pointer
    
    ; เรียก puts จริงด้วย argument เดิม
    mov  rdi, rbx
    call puts_real
    
    ; คืน success (non-negative = success)
    xor  eax, eax
    inc  eax
    
    add  rsp, 8
    pop  rbx
    leave
    ret

; เรียก real puts ผ่าน saved pointer
puts_real:
    mov  rax, [rel saved_puts]
    test rax, rax
    jz   .use_plt
    jmp  rax        ; jump to saved puts

.use_plt:
    jmp  puts       ; fallback ถ้ายังไม่ save

;=============================================================================
; get_plt_addr - ดึง address ของ puts@plt
; Output: rax = address ของ PLT stub
;=============================================================================
get_plt_addr:
    ; puts ใน linker script จะอ้างถึง PLT entry
    ; เราดึง address โดยตรงจาก symbol
    lea  rax, [rel puts]
    ret

;=============================================================================
; read_got_from_plt - อ่าน GOT address จาก PLT stub
; Input:  rdi = PLT stub address
; Output: rax = GOT entry address
;         rdx = current GOT value
;=============================================================================
read_got_from_plt:
    ; PLT stub format:
    ; ff 25 XX XX XX XX  JMP QWORD PTR [RIP+offset]
    ;
    ; Byte 0: ff
    ; Byte 1: 25
    ; Bytes 2-5: 32-bit signed offset (little-endian)
    ; RIP after instruction = PLT_addr + 6
    
    ; ตรวจ opcode
    movzx eax, byte [rdi]
    cmp   eax, 0xff
    jne   .not_plt
    
    movzx eax, byte [rdi+1]
    cmp   eax, 0x25
    jne   .not_plt
    
    ; อ่าน offset (32-bit signed)
    movsxd rax, dword [rdi+2]
    
    ; GOT address = (PLT + 6) + offset
    lea  rdx, [rdi+6]   ; address after instruction
    add  rax, rdx       ; GOT entry address
    
    ; อ่าน current GOT value
    mov  rdx, [rax]     ; current value in GOT
    ret

.not_plt:
    xor  rax, rax
    xor  rdx, rdx
    ret

;=============================================================================
; make_writable - ทำให้ page เป็น writable
; Input:  rdi = address
;=============================================================================
make_writable:
    push rbx
    mov  rbx, rdi
    
    ; คำนวณ page address (round down to page boundary)
    mov  rax, rbx
    and  rax, ~(PAGE_SIZE - 1)    ; align to page
    
    ; mprotect(page_addr, PAGE_SIZE, PROT_READ|PROT_WRITE)
    mov  rdi, rax
    mov  rsi, PAGE_SIZE
    mov  edx, PROT_READ | PROT_WRITE
    call mprotect
    
    pop  rbx
    ret

;=============================================================================
; main - โปรแกรมหลัก
;=============================================================================
main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 48

    ;------------------------------------------------------------------
    ; Step 1: เรียก puts ก่อน patch (lazy binding → GOT จะ resolve)
    ;------------------------------------------------------------------
    lea  rdi, [rel msg_before]
    call puts           ; ← ครั้งแรก ld.so จะ resolve และเขียน GOT
    
    ;------------------------------------------------------------------
    ; Step 2: ดึง PLT address และ GOT address
    ;------------------------------------------------------------------
    call get_plt_addr
    mov  [rsp], rax             ; บันทึก PLT address
    
    mov  rdi, rax
    call read_got_from_plt
    ; rax = GOT entry address
    ; rdx = current GOT value (real puts address)
    
    mov  [rsp+8], rax           ; GOT entry address
    mov  [rsp+16], rdx          ; current GOT value
    
    ; บันทึก real puts address
    mov  [rel saved_puts], rdx
    
    ; แสดงข้อมูล
    lea  rdi, [rel msg_info]
    mov  rsi, [rsp]             ; PLT address
    mov  rdx, [rsp+8]           ; GOT address
    mov  rcx, [rsp+16]          ; GOT value (real puts)
    xor  eax, eax
    call printf
    
    ;------------------------------------------------------------------
    ; Step 3: Patch GOT entry
    ;------------------------------------------------------------------
    
    ; ทำให้ GOT page เป็น writable
    mov  rdi, [rsp+8]           ; GOT entry address
    call make_writable
    
    ; เขียน address ของ hook ของเราเข้า GOT
    mov  rax, [rsp+8]           ; GOT entry address
    lea  rdx, [rel hook_puts]   ; address ของ hook
    mov  [rax], rdx             ; patch GOT!
    
    ;------------------------------------------------------------------
    ; Step 4: เรียก puts หลัง patch - ควรเรียก hook ของเรา
    ;------------------------------------------------------------------
    lea  rdi, [rel msg_after]
    call puts               ; ← จะเรียก hook_puts แทน puts จริง!
    
    ;------------------------------------------------------------------
    ; Step 5: cleanup
    ;------------------------------------------------------------------
    xor  eax, eax
    leave
    ret
```

---

## 13. โปรแกรมสมบูรณ์: LD_PRELOAD Hook สำหรับ malloc

```asm
; malloc_tracer.asm - LD_PRELOAD hook สมบูรณ์สำหรับ malloc/free/realloc
; แสดง call stack-like information และ memory leak detection
;
; คอมไพล์:
;   nasm -f elf64 -o malloc_tracer.o malloc_tracer.asm
;   gcc -shared -o libmalloc_tracer.so malloc_tracer.o -ldl
;
; ใช้งาน:
;   LD_PRELOAD=./libmalloc_tracer.so ./your_program
;
; คุณสมบัติ:
;   - log ทุก malloc/free/realloc call
;   - ตรวจ double-free
;   - แสดง stats ตอน exit

section .data
    ; Banner
    banner      db "=== malloc tracer active ===", 10, 0
    banner_end  db "=== malloc tracer stats ===", 10, 0
    
    ; ชื่อ symbols
    sym_malloc  db "malloc", 0
    sym_free    db "free", 0
    sym_realloc db "realloc", 0
    sym_calloc  db "calloc", 0
    
    ; Format strings
    fmt_malloc  db "[MALLOC] size=%zu ptr=%p", 10, 0
    fmt_free    db "[FREE]   ptr=%p", 10, 0
    fmt_realloc db "[REALLOC] ptr=%p size=%zu -> new_ptr=%p", 10, 0
    fmt_calloc  db "[CALLOC] nmemb=%zu size=%zu ptr=%p", 10, 0
    fmt_dfree   db "[ERROR] Double-free detected! ptr=%p", 10, 0
    fmt_stats   db "[STATS] allocs=%ld frees=%ld reallocs=%ld\n"
                db "        total_allocated=%ld bytes", 10, 0
    fmt_leak    db "[LEAK]  %ld allocations not freed (%ld bytes)", 10, 0
    
    ; error messages
    err_init    db "ERROR: dlsym failed during init", 10, 0

section .bss
    ; Original function pointers
    orig_malloc  resq 1
    orig_free    resq 1
    orig_realloc resq 1
    orig_calloc  resq 1
    
    ; Statistics
    stat_allocs   resq 1
    stat_frees    resq 1
    stat_reallocs resq 1
    stat_bytes    resq 1
    
    ; Initialization flag
    initialized   resb 1

section .text
    global malloc
    global free
    global realloc
    global calloc
    extern dlsym
    extern fprintf
    extern fputs
    extern stderr

RTLD_NEXT equ -1

;=============================================================================
; tracer_init - initialize hook
; ดึง original function pointers ด้วย dlsym(RTLD_NEXT, ...)
;=============================================================================
tracer_init:
    push rbp
    mov  rbp, rsp
    push rbx
    sub  rsp, 8
    
    ; ป้องกัน double initialization
    movzx eax, byte [rel initialized]
    test  eax, eax
    jnz   .init_done
    
    ; mark as initialized ก่อน (ป้องกัน recursion)
    mov byte [rel initialized], 1
    
    ; ดึง original malloc
    mov  rdi, RTLD_NEXT
    lea  rsi, [rel sym_malloc]
    call dlsym
    test rax, rax
    jz   .init_error
    mov  [rel orig_malloc], rax
    
    ; ดึง original free
    mov  rdi, RTLD_NEXT
    lea  rsi, [rel sym_free]
    call dlsym
    test rax, rax
    jz   .init_error
    mov  [rel orig_free], rax
    
    ; ดึง original realloc
    mov  rdi, RTLD_NEXT
    lea  rsi, [rel sym_realloc]
    call dlsym
    test rax, rax
    jz   .init_error
    mov  [rel orig_realloc], rax
    
    ; ดึง original calloc
    mov  rdi, RTLD_NEXT
    lea  rsi, [rel sym_calloc]
    call dlsym
    test rax, rax
    jz   .init_error
    mov  [rel orig_calloc], rax
    
    ; แสดง banner
    mov  rdi, [rel stderr]
    lea  rsi, [rel banner]
    call fputs

.init_done:
    add  rsp, 8
    pop  rbx
    leave
    ret

.init_error:
    mov  rdi, [rel stderr]
    lea  rsi, [rel err_init]
    call fputs
    ; ออกจากโปรแกรม
    mov  eax, 60
    mov  edi, 1
    syscall

;=============================================================================
; malloc hook
; Input:  rdi = size
; Output: rax = allocated pointer
;=============================================================================
malloc:
    push rbp
    mov  rbp, rsp
    push r12
    sub  rsp, 8
    
    mov  r12, rdi   ; บันทึก size
    
    ; initialize ถ้ายังไม่ได้ init
    movzx eax, byte [rel initialized]
    test  eax, eax
    jnz   .malloc_initialized
    call  tracer_init

.malloc_initialized:
    ; เรียก original malloc
    mov  rdi, r12
    call qword [rel orig_malloc]
    push rax        ; บันทึก return value
    
    ; อัพเดต stats
    inc  qword [rel stat_allocs]
    add  qword [rel stat_bytes], r12
    
    ; log
    mov  rdi, [rel stderr]
    lea  rsi, [rel fmt_malloc]
    mov  rdx, r12           ; size
    mov  rcx, [rsp]         ; ptr
    xor  eax, eax
    call fprintf
    
    pop  rax        ; คืน pointer
    add  rsp, 8
    pop  r12
    leave
    ret

;=============================================================================
; free hook
; Input:  rdi = pointer
;=============================================================================
free:
    push rbp
    mov  rbp, rsp
    push r12
    sub  rsp, 8
    
    mov  r12, rdi   ; บันทึก pointer
    
    ; initialize ถ้ายังไม่ได้ init
    movzx eax, byte [rel initialized]
    test  eax, eax
    jnz   .free_initialized
    call  tracer_init

.free_initialized:
    ; log free (ถ้าไม่ใช่ NULL)
    test r12, r12
    jz   .free_null
    
    ; อัพเดต stats
    inc  qword [rel stat_frees]
    
    ; log
    mov  rdi, [rel stderr]
    lea  rsi, [rel fmt_free]
    mov  rdx, r12           ; pointer
    xor  eax, eax
    call fprintf

.free_null:
    ; เรียก original free
    mov  rdi, r12
    call qword [rel orig_free]
    
    add  rsp, 8
    pop  r12
    leave
    ret

;=============================================================================
; realloc hook
; Input:  rdi = old_ptr, rsi = new_size
; Output: rax = new pointer
;=============================================================================
realloc:
    push rbp
    mov  rbp, rsp
    push r12
    push r13
    sub  rsp, 8
    
    mov  r12, rdi   ; old_ptr
    mov  r13, rsi   ; new_size
    
    ; initialize
    movzx eax, byte [rel initialized]
    test  eax, eax
    jnz   .realloc_initialized
    call  tracer_init

.realloc_initialized:
    ; เรียก original realloc
    mov  rdi, r12
    mov  rsi, r13
    call qword [rel orig_realloc]
    push rax        ; new_ptr
    
    ; อัพเดต stats
    inc  qword [rel stat_reallocs]
    
    ; log
    mov  rdi, [rel stderr]
    lea  rsi, [rel fmt_realloc]
    mov  rdx, r12       ; old_ptr
    mov  rcx, r13       ; new_size
    mov  r8, [rsp]      ; new_ptr
    xor  eax, eax
    call fprintf
    
    pop  rax
    add  rsp, 8
    pop  r13
    pop  r12
    leave
    ret

;=============================================================================
; calloc hook
; Input:  rdi = nmemb, rsi = size
; Output: rax = pointer
;=============================================================================
calloc:
    push rbp
    mov  rbp, rsp
    push r12
    push r13
    sub  rsp, 8
    
    mov  r12, rdi   ; nmemb
    mov  r13, rsi   ; size
    
    ; initialize
    movzx eax, byte [rel initialized]
    test  eax, eax
    jnz   .calloc_initialized
    call  tracer_init

.calloc_initialized:
    ; เรียก original calloc
    mov  rdi, r12
    mov  rsi, r13
    call qword [rel orig_calloc]
    push rax
    
    ; คำนวณ total size
    mov  rax, r12
    imul rax, r13       ; total = nmemb * size
    add  qword [rel stat_allocs], 1
    add  qword [rel stat_bytes], rax
    
    ; log
    mov  rdi, [rel stderr]
    lea  rsi, [rel fmt_calloc]
    mov  rdx, r12       ; nmemb
    mov  rcx, r13       ; size
    mov  r8, [rsp]      ; ptr
    xor  eax, eax
    call fprintf
    
    pop  rax
    add  rsp, 8
    pop  r13
    pop  r12
    leave
    ret

;=============================================================================
; print_final_stats - destructor: แสดง statistics ตอน exit
;=============================================================================
print_final_stats:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16
    
    ; แสดง banner
    mov  rdi, [rel stderr]
    lea  rsi, [rel banner_end]
    call fputs
    
    ; แสดง stats
    mov  rdi, [rel stderr]
    lea  rsi, [rel fmt_stats]
    mov  rdx, [rel stat_allocs]
    mov  rcx, [rel stat_frees]
    mov  r8,  [rel stat_reallocs]
    mov  r9,  [rel stat_bytes]
    xor  eax, eax
    call fprintf
    
    ; ตรวจ leaks: allocs - frees = leaks
    mov  rax, [rel stat_allocs]
    sub  rax, [rel stat_frees]
    test rax, rax
    jle  .no_leaks
    
    ; มี leaks
    mov  rdi, [rel stderr]
    lea  rsi, [rel fmt_leak]
    mov  rdx, rax           ; leak count
    ; (ไม่สามารถรู้ exact bytes ได้โดยไม่ track ทุก ptr)
    xor  rcx, rcx
    xor  eax, eax
    call fprintf

.no_leaks:
    leave
    ret

;=============================================================================
; Constructor/Destructor tables
;=============================================================================
section .init_array
    dq tracer_init

section .fini_array
    dq print_final_stats
```

---

## 14. สรุปและ Commands อ้างอิง

### 14.1 Commands สำคัญ

```bash
#---------------------------------------------------------------------
# วิเคราะห์ ELF binary
#---------------------------------------------------------------------

# ดู ELF header
readelf -h program

# ดู segments
readelf -l program

# ดู sections
readelf -S program

# ดู dynamic section
readelf -d program

# ดู symbol table
readelf -s program

# ดู relocation entries
readelf -r program

# ดู GOT/PLT
objdump -d -j .plt program   # PLT
objdump -d -j .got program   # GOT (ต้องหลัง run)

#---------------------------------------------------------------------
# Compile ด้วย options ต่างๆ
#---------------------------------------------------------------------

# Shared library
gcc -fPIC -shared -o libfoo.so foo.c

# Assembly shared library
nasm -f elf64 -o foo.o foo.asm
gcc -shared -o libfoo.so foo.o

# Enable/Disable RELRO
gcc -Wl,-z,relro,-z,now -o prog main.c   # Full RELRO
gcc -Wl,-z,relro -o prog main.c          # Partial RELRO
gcc -Wl,-z,norelro -o prog main.c        # No RELRO

# Force eager binding
gcc -Wl,-z,now -o prog main.c            # LD_BIND_NOW equivalent

# Link ด้วย shared library ที่สร้างเอง
gcc -o program main.c -L. -lmylib -Wl,-rpath,.

#---------------------------------------------------------------------
# LD_PRELOAD
#---------------------------------------------------------------------

# ใช้ LD_PRELOAD
LD_PRELOAD=./my_hook.so ./program

# หลาย libraries
LD_PRELOAD="./hook1.so ./hook2.so" ./program

# ปิด lazy binding (eager)
LD_BIND_NOW=1 ./program

# Debug dynamic linker
LD_DEBUG=all ./program 2>&1 | less
LD_DEBUG=libs ./program     # แสดงการโหลด libraries
LD_DEBUG=symbols ./program  # แสดง symbol lookups
LD_DEBUG=bindings ./program # แสดง symbol bindings

#---------------------------------------------------------------------
# Security checks
#---------------------------------------------------------------------

# checksec (ต้องติดตั้ง: apt install checksec)
checksec --file=program
checksec --file=*.so

# patchelf - แก้ไข ELF properties
patchelf --set-interpreter /lib64/ld-linux-x86-64.so.2 program
patchelf --set-rpath '$ORIGIN' libfoo.so
patchelf --replace-needed libold.so libnew.so program
```

### 14.2 Quick Reference: PLT/GOT

```
PLT (Procedure Linkage Table):
├── PLT[0]: Resolver stub (เรียก ld.so)
│   ├── PUSH GOT[1] (link_map)
│   └── JMP  GOT[2] (_dl_runtime_resolve)
│
├── PLT[1]: Function 1 stub (e.g., puts)
│   ├── JMP  [GOT_puts]    ← indirect jump ผ่าน GOT
│   ├── PUSH reloc_index   ← push index เข้า .rela.plt
│   └── JMP  PLT[0]        ← เรียก resolver
│
└── PLT[n]: Function n stub
    ├── JMP  [GOT_funcn]
    ├── PUSH n-1
    └── JMP  PLT[0]

GOT (Global Offset Table) - .got.plt:
├── GOT[0] = .dynamic address
├── GOT[1] = link_map (ld.so fills)
├── GOT[2] = _dl_runtime_resolve (ld.so fills)
├── GOT[3] = puts addr (initial: PLT[1]+6, then real puts)
├── GOT[4] = printf addr
└── GOT[n] = funcn addr

RELRO protection:
├── No RELRO:      GOT = writable (vulnerable)
├── Partial RELRO: .got = RO, .got.plt = RW (lazy binding OK)
└── Full RELRO:    all GOT = RO (eager binding, most secure)
```

### 14.3 Compilation Scripts

```bash
#!/bin/bash
# build_all.sh - script สำหรับ build ทุก examples ในบทนี้

set -e  # exit on error

echo "Building Part 059 examples..."

# ตรวจสอบ tools
command -v nasm  >/dev/null || { echo "nasm not found"; exit 1; }
command -v gcc   >/dev/null || { echo "gcc not found"; exit 1; }

#------------------------------------------------------------------
# Build math shared library (Assembly)
#------------------------------------------------------------------
echo "Building libmathasm.so..."
nasm -f elf64 -o mathlib.o mathlib.asm
gcc -shared -o libmathasm.so mathlib.o
echo "  → libmathasm.so"

# Build test program
gcc -o test_mathlib main_mathlib.c -L. -lmathasm -Wl,-rpath,.
echo "  → test_mathlib"

#------------------------------------------------------------------
# Build PIC demo library
#------------------------------------------------------------------
echo "Building libpic_demo.so..."
nasm -f elf64 -o pic_library.o pic_library.asm
gcc -shared -o libpic_demo.so pic_library.o
echo "  → libpic_demo.so"

#------------------------------------------------------------------
# Build malloc tracer (LD_PRELOAD)
#------------------------------------------------------------------
echo "Building libmalloc_tracer.so..."
nasm -f elf64 -o malloc_tracer.o malloc_tracer.asm
gcc -shared -o libmalloc_tracer.so malloc_tracer.o -ldl
echo "  → libmalloc_tracer.so"

#------------------------------------------------------------------
# Build dlopen demo
#------------------------------------------------------------------
echo "Building dlopen_demo..."
nasm -f elf64 -o dlopen_demo.o dlopen_demo.asm
gcc -o dlopen_demo dlopen_demo.o -ldl
echo "  → dlopen_demo"

#------------------------------------------------------------------
# Build reverse plugin
#------------------------------------------------------------------
echo "Building reverse_plugin.so..."
gcc -fPIC -shared -o reverse_plugin.so reverse_plugin.c
echo "  → reverse_plugin.so"

# Build plugin host
echo "Building plugin_host..."
gcc -o plugin_host plugin_host.c -ldl
echo "  → plugin_host"

echo ""
echo "=== Build complete! ==="
echo ""
echo "Test commands:"
echo "  ./test_mathlib"
echo "  LD_PRELOAD=./libmalloc_tracer.so ./test_mathlib"
echo "  ./dlopen_demo"
echo "  ./plugin_host ./reverse_plugin.so 'Hello World'"
echo "  LD_DEBUG=bindings ./test_mathlib 2>&1 | head -20"
```

---

## 15. การทดสอบและ Debugging

### 15.1 ดู PLT/GOT ใน GDB

```
# รัน gdb
$ gdb ./program

# ดู PLT entries
(gdb) info plt
PLT  Address          Symbol
----  -------          ------
  0  0x401020          (reserved)
  1  0x401030          puts
  2  0x401040          printf

# ดู GOT entries
(gdb) info got
GOT for /home/user/program:
.got.plt:
  GOT entry 0 [.dynamic] = 0x403e10
  GOT entry 1 [link_map] = 0x7ffff7ffe2e0
  GOT entry 2 [dl_resolve] = 0x7ffff7fda920
  GOT entry 3 [puts@GLIBC_2.2.5] = 0x401036  ← ก่อน first call
  
# หลัง run และเรียก puts:
  GOT entry 3 [puts@GLIBC_2.2.5] = 0x7ffff7e2f430  ← resolved!

# breakpoint ที่ PLT
(gdb) break puts@plt
(gdb) run
# เมื่อหยุด:
(gdb) disas         # ดู PLT stub
(gdb) x/1xg 0x403fe8   # ดู GOT entry สำหรับ puts
(gdb) stepi         # step ทีละ instruction
```

### 15.2 ใช้ ltrace ดู Library Calls

```bash
# ltrace แสดง library function calls
$ ltrace ./program
puts("Hello World")                      = 12
printf("Count: %d\n", 42)               = 11
malloc(100)                              = 0x5555556f2670
free(0x5555556f2670)                     = <void>
+++ exited (status 0) +++
```

### 15.3 ใช้ strace ดู Syscalls ระหว่าง Dynamic Linking

```bash
$ strace -e trace=open,mmap,read ./program 2>&1 | head -30
# จะเห็น:
# openat(.., "libc.so.6", ...)  ← โหลด library
# mmap(..., libc.so.6)          ← map เข้า memory
# ...
```

---

## 16. สรุปบทเรียน

### สิ่งที่ได้เรียนรู้:

1. **ELF Dynamic Linking**: โครงสร้าง ELF, segments, dynamic section
2. **Lazy Binding**: กลไกการ resolve symbols เมื่อต้องการใช้
3. **PLT Structure**: PLT[0] resolver, PLT[n] function stubs (16 bytes each)
4. **GOT Structure**: special entries (GOT[0-2]), function pointers
5. **Relocation**: RELA.PLT, R_X86_64_JUMP_SLOT
6. **ld.so Operation**: initialization, symbol resolution algorithm
7. **LD_PRELOAD**: function hooking สำหรับ debugging/instrumentation
8. **dlopen/dlsym**: dynamic loading สำหรับ plugin systems
9. **PIC**: RIP-relative addressing, GOT access สำหรับ globals
10. **RELRO**: Partial vs Full, GOT overwrite protection

### Best Practices:

- ใช้ **Full RELRO** สำหรับ production binaries (`-Wl,-z,relro,-z,now`)
- ใช้ **PIE** (`-pie -fPIE`) สำหรับ ASLR support
- ใช้ **LD_PRELOAD** อย่างระวัง (security implication)
- เข้าใจ **lazy vs eager binding** tradeoffs

---

*จบ Part 059: Dynamic Linking, PLT/GOT Deep Dive*

*ในบทถัดไป: Part 060 - Linux Kernel Internals ใน Assembly*
