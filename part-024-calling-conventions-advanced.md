# Part 024: Calling Conventions ขั้นสูง - stdcall, fastcall, Microsoft x64 ABI

## ภาพรวม (Overview)

ในบทนี้เราจะเรียนรู้ **Calling Conventions ขั้นสูง** ที่ใช้จริงในการพัฒนาซอฟต์แวร์ระดับมืออาชีพ
โดยเฉพาะการทำงานกับ Windows API และการเข้าใจความแตกต่างระหว่าง x86 และ x64

---

## สารบัญ (Table of Contents)

1. [ทบทวน cdecl และ Calling Conventions พื้นฐาน](#1-ทบทวน)
2. [stdcall - Windows API Convention](#2-stdcall)
3. [fastcall - การส่ง Arguments ผ่าน Register](#3-fastcall)
4. [Microsoft x64 ABI](#4-microsoft-x64-abi)
5. [Shadow Space (Home Space)](#5-shadow-space)
6. [XMM Registers สำหรับ Float Arguments](#6-xmm-registers)
7. [Windows vs Linux x64 Differences](#7-windows-vs-linux-x64)
8. [thiscall สำหรับ C++ Methods](#8-thiscall)
9. [vectorcall สำหรับ SIMD](#9-vectorcall)
10. [การเรียก Windows API จาก Assembly](#10-windows-api)
11. [MessageBox API](#11-messagebox)
12. [CreateFile และ File I/O](#12-createfile)
13. [VirtualAlloc - Memory Management](#13-virtualalloc)
14. [LoadLibrary และ GetProcAddress](#14-loadlibrary)
15. [แบบฝึกหัด](#15-แบบฝึกหัด)

---

## 1. ทบทวน Calling Conventions พื้นฐาน

ก่อนเข้าสู่เนื้อหาขั้นสูง มาทบทวน Calling Conventions ที่เรียนมาแล้ว:

```
┌─────────────────┬──────────────┬────────────────┬──────────────────────┐
│ Convention      │ Args Order   │ Stack Cleanup  │ ใช้ใน               │
├─────────────────┼──────────────┼────────────────┼──────────────────────┤
│ cdecl           │ Right→Left   │ Caller         │ C/C++ ทั่วไป (Linux) │
│ stdcall         │ Right→Left   │ Callee         │ Windows API (32-bit) │
│ fastcall        │ ECX,EDX,Stk  │ Callee         │ MSVC /fastcall       │
│ Microsoft x64   │ RCX,RDX,R8,R9│ Caller         │ Windows 64-bit       │
│ System V AMD64  │ RDI,RSI,RDX..│ Caller         │ Linux 64-bit         │
│ thiscall        │ ECX=this,Stk │ Callee         │ C++ Methods (MSVC)   │
│ vectorcall      │ XMM/YMM,Regs │ Caller         │ SIMD-heavy code      │
└─────────────────┴──────────────┴────────────────┴──────────────────────┘
```

### ทำไม Calling Conventions ถึงสำคัญ?

```nasm
; ตัวอย่างปัญหาที่เกิดจาก Convention ผิด
; ถ้า C code เรียกฟังก์ชันที่เขียนด้วย Assembly
; และ Convention ไม่ตรงกัน จะเกิดปัญหาดังนี้:

; 1. Stack Corruption - ค่า stack pointer ผิดพลาด
; 2. Wrong return values - ค่าที่ return กลับมาผิด
; 3. Register clobbering - register ที่ควร preserve ถูกทำลาย
; 4. Crash หรือ Undefined Behavior
```

---

## 2. stdcall - Windows API Convention

### หลักการของ stdcall

**stdcall** เป็น Convention หลักของ Windows API 32-bit โดยมีลักษณะดังนี้:

- Arguments ถูก push บน stack **จากขวาไปซ้าย** (เหมือน cdecl)
- **Callee** (ฟังก์ชันที่ถูกเรียก) รับผิดชอบการ cleanup stack
- ใช้คำสั่ง `RET n` เพื่อ pop arguments ออกจาก stack
- Return value อยู่ใน `EAX` (หรือ `EAX:EDX` สำหรับ 64-bit value)

```
Stack layout เมื่อเรียก stdcall function(a, b, c):

การ push: c ก่อน, แล้ว b, แล้ว a
          ↓
[ESP+0 ] = Return Address    ← ESP ชี้ที่นี่เมื่อเข้าฟังก์ชัน
[ESP+4 ] = a (argument 1)
[ESP+8 ] = b (argument 2)
[ESP+12] = c (argument 3)
```

### ตัวอย่างที่ 1: ฟังก์ชัน stdcall อย่างง่าย

```nasm
; ============================================================
; ไฟล์: stdcall_basic.asm
; คำอธิบาย: ตัวอย่าง stdcall convention พื้นฐาน
; ระบบ: Windows 32-bit (NASM)
; คอมไพล์: nasm -f win32 stdcall_basic.asm -o stdcall_basic.obj
;           link /subsystem:console stdcall_basic.obj
; ============================================================

bits 32
global _main
global _AddNumbers@8        ; @8 = ขนาด arguments ที่ callee จะ cleanup (2 args × 4 bytes)

extern _printf

section .data
    format_str  db "Result: %d", 10, 0
    newline     db 10, 0

section .text

; ─────────────────────────────────────────────────────────
; ฟังก์ชัน AddNumbers ใช้ stdcall convention
; Prototype: int __stdcall AddNumbers(int a, int b)
; Arguments: [ESP+8] = a, [ESP+4+4] = b (หลัง push EBP)
; ─────────────────────────────────────────────────────────
_AddNumbers@8:
    ; Standard function prologue
    push    ebp                 ; บันทึก base pointer เดิม
    mov     ebp, esp            ; ตั้ง base pointer ใหม่
    
    ; ตอนนี้ stack layout เป็น:
    ; [EBP+0 ] = ค่าเดิมของ EBP
    ; [EBP+4 ] = Return Address
    ; [EBP+8 ] = argument a (first argument)
    ; [EBP+12] = argument b (second argument)
    
    ; ดึง arguments
    mov     eax, [ebp+8]        ; eax = a
    mov     ecx, [ebp+12]       ; ecx = b
    
    ; คำนวณ
    add     eax, ecx            ; eax = a + b
    
    ; Standard function epilogue
    pop     ebp                 ; คืนค่า base pointer
    
    ; stdcall: callee cleanup stack
    ; RET 8 = return + pop 8 bytes (2 arguments × 4 bytes each)
    ret     8

; ─────────────────────────────────────────────────────────
; ฟังก์ชัน main
; ─────────────────────────────────────────────────────────
_main:
    push    ebp
    mov     ebp, esp
    
    ; เรียก AddNumbers(10, 20)
    ; stdcall: push arguments จากขวาไปซ้าย
    push    20                  ; argument b = 20 (push ก่อน)
    push    10                  ; argument a = 10 (push หลัง)
    call    _AddNumbers@8       ; เรียกฟังก์ชัน
    ; หลัง call: ไม่ต้องทำ ADD ESP, 8 เพราะ stdcall cleanup เอง
    
    ; แสดงผล
    push    eax                 ; result
    push    format_str
    call    _printf
    add     esp, 8              ; printf ใช้ cdecl ต้อง cleanup เอง
    
    ; Return 0
    xor     eax, eax
    pop     ebp
    ret
```

### ตัวอย่างที่ 2: stdcall กับ Multiple Arguments

```nasm
; ============================================================
; ไฟล์: stdcall_multiargs.asm
; คำอธิบาย: stdcall กับหลาย arguments และ local variables
; ============================================================

bits 32
global _main
global _CalculateArea@12    ; 3 arguments × 4 bytes = 12

extern _printf

section .data
    fmt_area    db "Area of %dx%d with margin %d = %d", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; int __stdcall CalculateArea(int width, int height, int margin)
; คำนวณ: (width - 2*margin) * (height - 2*margin)
; ─────────────────────────────────────────────────────────
_CalculateArea@12:
    push    ebp
    mov     ebp, esp
    
    ; จัดสรร local variable 1 ตัว (4 bytes)
    sub     esp, 4              ; [EBP-4] = temp variable
    
    ; ดึง arguments
    ; [EBP+8 ] = width
    ; [EBP+12] = height  
    ; [EBP+16] = margin
    
    mov     eax, [ebp+8]        ; eax = width
    mov     ecx, [ebp+16]       ; ecx = margin
    add     ecx, ecx            ; ecx = 2 * margin
    sub     eax, ecx            ; eax = width - 2*margin
    mov     [ebp-4], eax        ; เก็บค่าชั่วคราว
    
    mov     eax, [ebp+12]       ; eax = height
    sub     eax, ecx            ; eax = height - 2*margin
    
    imul    eax, [ebp-4]        ; eax = (width-2m) * (height-2m)
    
    ; Epilogue
    mov     esp, ebp            ; คืนค่า ESP (ลบ local variables)
    pop     ebp
    ret     12                  ; cleanup 3 args × 4 bytes = 12

_main:
    push    ebp
    mov     ebp, esp
    
    ; เรียก CalculateArea(100, 80, 5)
    push    5                   ; margin (push ก่อน = rightmost)
    push    80                  ; height
    push    100                 ; width (push หลัง = leftmost)
    call    _CalculateArea@12
    ; ไม่ต้องทำ add esp, 12 เพราะ stdcall cleanup ให้แล้ว
    
    ; แสดงผล: Area of 100x80 with margin 5 = 8100
    push    eax                 ; result
    push    dword 5             ; margin
    push    dword 80            ; height
    push    dword 100           ; width
    push    fmt_area
    call    _printf
    add     esp, 20             ; cdecl cleanup: 5 args × 4 bytes
    
    xor     eax, eax
    pop     ebp
    ret
```

### ความแตกต่างระหว่าง stdcall และ cdecl

```
cdecl:
┌─────────────┐    ┌─────────────────────────────────────────┐
│ Caller      │    │                                         │
│ push arg2   │    │  push arg2                              │
│ push arg1   │    │  push arg1                              │
│ call func   │ →  │  call func                              │
│ add esp,8   │ ←  │  ; CALLER cleans up ← ตรงนี้ต่างกัน    │
└─────────────┘    └─────────────────────────────────────────┘

stdcall:
┌─────────────┐    ┌─────────────────────────────────────────┐
│ Caller      │    │                                         │
│ push arg2   │    │  push arg2                              │
│ push arg1   │    │  push arg1                              │
│ call func   │ →  │  call func                              │
│ ; no cleanup│ ←  │  ; CALLEE cleans up (ret 8) ← ต่างกัน  │
└─────────────┘    └─────────────────────────────────────────┘
```

### ข้อดีของ stdcall

1. **Code ขนาดเล็กกว่า**: ถ้าฟังก์ชันถูกเรียกหลายครั้ง การ cleanup ทำแค่ที่เดียว (ใน callee) แทนที่จะทำทุกจุดที่เรียก
2. **Windows API ใช้ทั้งหมด**: ทุก Win32 API ใช้ stdcall
3. **Slightly safer**: ลด chance ของ stack imbalance bugs

---

## 3. fastcall - การส่ง Arguments ผ่าน Register

### หลักการของ __fastcall (32-bit)

**fastcall** เร็วกว่า stdcall เพราะส่ง arguments แรกสองตัวผ่าน registers:

- **Argument 1** → `ECX`
- **Argument 2** → `EDX`
- **Arguments ที่เหลือ** → Stack (จากขวาไปซ้าย เหมือน stdcall)
- **Callee** cleanup stack (เหมือน stdcall)
- ชื่อ decorated: `@name@n` (มี @ ทั้งหน้าและหลัง)

```
fastcall function(a, b, c, d):

ECX = a (argument 1)
EDX = b (argument 2)
Stack:
  [ESP+0] = Return Address
  [ESP+4] = c (argument 3)
  [ESP+8] = d (argument 4)
```

### ตัวอย่างที่ 3: fastcall Convention

```nasm
; ============================================================
; ไฟล์: fastcall_demo.asm
; คำอธิบาย: __fastcall convention ใน 32-bit
; คอมไพล์: nasm -f win32 fastcall_demo.asm -o fastcall_demo.obj
; ============================================================

bits 32
global _main
global @MultiplyAdd@16      ; fastcall: @name@n format, n=16 (4 args × 4 bytes)

extern _printf

section .data
    fmt_result  db "__fastcall result: %d", 10, 0
    fmt_fast    db "ECX=%d EDX=%d Stack1=%d Stack2=%d -> %d", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; int __fastcall MultiplyAdd(int a, int b, int c, int d)
; คำนวณ: a * b + c * d
; Arguments: ECX=a, EDX=b, [ESP+4]=c, [ESP+8]=d
; ─────────────────────────────────────────────────────────
@MultiplyAdd@16:
    push    ebp
    mov     ebp, esp
    
    ; ใน fastcall ต้องบันทึก ECX และ EDX ก่อน (ถ้าต้องการใช้ arguments นาน)
    ; เพราะ ECX/EDX อาจถูก overwrite
    
    ; [EBP+0 ] = ค่าเดิม EBP
    ; [EBP+4 ] = Return Address
    ; [EBP+8 ] = c (argument 3)
    ; [EBP+12] = d (argument 4)
    ; ECX = a (argument 1) ← ผ่าน register
    ; EDX = b (argument 2) ← ผ่าน register
    
    ; คำนวณ a * b
    mov     eax, ecx            ; eax = a
    imul    eax, edx            ; eax = a * b
    
    ; บันทึก a*b
    push    eax                 ; เก็บ a*b ไว้ใน stack ชั่วคราว
    
    ; คำนวณ c * d
    mov     eax, [ebp+8]        ; eax = c
    imul    eax, [ebp+12]       ; eax = c * d
    
    ; a*b + c*d
    pop     ecx                 ; ecx = a*b
    add     eax, ecx            ; eax = a*b + c*d
    
    pop     ebp
    ret     8                   ; cleanup stack args only (c,d = 8 bytes)
                                ; ECX,EDX ไม่ต้อง cleanup เพราะอยู่ใน register

; ─────────────────────────────────────────────────────────
; ฟังก์ชัน fastcall_test เพื่อทดสอบ
; ─────────────────────────────────────────────────────────
_main:
    push    ebp
    mov     ebp, esp
    
    ; เรียก MultiplyAdd(3, 4, 5, 6) = 3*4 + 5*6 = 12 + 30 = 42
    ; fastcall: ส่ง 2 args แรกผ่าน register, ที่เหลือ push
    push    6                   ; d = 6 (argument 4, push ก่อน)
    push    5                   ; c = 5 (argument 3)
    mov     edx, 4              ; b = 4 (argument 2 ใน EDX)
    mov     ecx, 3              ; a = 3 (argument 1 ใน ECX)
    call    @MultiplyAdd@16
    ; ผลลัพธ์อยู่ใน EAX = 42
    
    push    eax
    push    fmt_result
    call    _printf
    add     esp, 8
    
    xor     eax, eax
    pop     ebp
    ret
```

### ตัวอย่างที่ 4: เปรียบเทียบ Performance ระหว่าง cdecl, stdcall, fastcall

```nasm
; ============================================================
; ไฟล์: convention_perf.asm
; คำอธิบาย: เปรียบเทียบ overhead ของแต่ละ convention
; ============================================================

bits 32
global _main

extern _printf
extern _GetTickCount        ; Windows API สำหรับจับเวลา

section .data
    fmt_time    db "%s: %d ms for 1 million calls", 10, 0
    str_cdecl   db "cdecl  ", 0
    str_stdcall db "stdcall", 0
    str_fast    db "fastcall", 0
    ITERATIONS  equ 1000000

section .bss
    start_time  resd 1
    end_time    resd 1

section .text

; cdecl version
_Sum_cdecl:
    push    ebp
    mov     ebp, esp
    mov     eax, [ebp+8]
    add     eax, [ebp+12]
    pop     ebp
    ret                     ; caller cleanup

; stdcall version  
_Sum_stdcall@8:
    push    ebp
    mov     ebp, esp
    mov     eax, [ebp+8]
    add     eax, [ebp+12]
    pop     ebp
    ret     8               ; callee cleanup

; fastcall version
@Sum_fastcall@8:
    ; ECX = a, EDX = b
    lea     eax, [ecx+edx]  ; eax = a + b (ไม่มี stack args ต้อง cleanup)
    ret

_main:
    push    ebp
    mov     ebp, esp
    sub     esp, 4          ; local: loop counter
    
    ; ─── Test cdecl ───
    call    _GetTickCount
    mov     [start_time], eax
    
    mov     dword [ebp-4], ITERATIONS   ; loop counter
.cdecl_loop:
    push    20
    push    10
    call    _Sum_cdecl
    add     esp, 8          ; cdecl cleanup
    dec     dword [ebp-4]
    jnz     .cdecl_loop
    
    call    _GetTickCount
    sub     eax, [start_time]
    push    eax
    push    str_cdecl
    push    fmt_time
    call    _printf
    add     esp, 12
    
    ; ─── Test stdcall ───
    call    _GetTickCount
    mov     [start_time], eax
    
    mov     dword [ebp-4], ITERATIONS
.stdcall_loop:
    push    20
    push    10
    call    _Sum_stdcall@8
    ; ไม่ต้องทำ cleanup!
    dec     dword [ebp-4]
    jnz     .stdcall_loop
    
    call    _GetTickCount
    sub     eax, [start_time]
    push    eax
    push    str_stdcall
    push    fmt_time
    call    _printf
    add     esp, 12
    
    ; ─── Test fastcall ───
    call    _GetTickCount
    mov     [start_time], eax
    
    mov     dword [ebp-4], ITERATIONS
.fastcall_loop:
    mov     ecx, 10         ; argument 1
    mov     edx, 20         ; argument 2
    call    @Sum_fastcall@8
    dec     dword [ebp-4]
    jnz     .fastcall_loop
    
    call    _GetTickCount
    sub     eax, [start_time]
    push    eax
    push    str_fast
    push    fmt_time
    call    _printf
    add     esp, 12
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

---

## 4. Microsoft x64 ABI

### ภาพรวม Microsoft x64 ABI

เมื่อย้ายมา 64-bit ทุกอย่างเปลี่ยนไปมาก Microsoft x64 ABI (ใช้ใน Windows 64-bit) มีลักษณะดังนี้:

```
Integer/Pointer Arguments:
├── Argument 1 → RCX
├── Argument 2 → RDX
├── Argument 3 → R8
├── Argument 4 → R9
└── Arguments 5+ → Stack (right-to-left)

Float Arguments:
├── Argument 1 → XMM0
├── Argument 2 → XMM1
├── Argument 3 → XMM2
├── Argument 4 → XMM3
└── Arguments 5+ → Stack

Return Values:
├── Integer ≤ 64-bit → RAX
├── Float → XMM0
└── Large struct → caller allocates, pointer in RCX

Preserved Registers (Callee-saved):
    RBX, RBP, RDI, RSI, RSP, R12, R13, R14, R15
    XMM6-XMM15

Volatile Registers (Caller-saved):
    RAX, RCX, RDX, R8, R9, R10, R11
    XMM0-XMM5
```

### ตัวอย่างที่ 5: Microsoft x64 ABI พื้นฐาน

```nasm
; ============================================================
; ไฟล์: msvc_x64_basic.asm
; คำอธิบาย: Microsoft x64 ABI พื้นฐาน
; ระบบ: Windows 64-bit (NASM)
; คอมไพล์: nasm -f win64 msvc_x64_basic.asm -o msvc_x64_basic.obj
;           link /subsystem:console /entry:main msvc_x64_basic.obj
; ============================================================

bits 64
default rel

global main

extern printf

section .data
    fmt_args    db "a=%lld b=%lld c=%lld d=%lld", 10, 0
    fmt_result  db "Sum = %lld", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; int64_t SumFour(int64_t a, int64_t b, int64_t c, int64_t d)
; Arguments: RCX=a, RDX=b, R8=c, R9=d
; Return: RAX = a+b+c+d
; ─────────────────────────────────────────────────────────
SumFour:
    push    rbp
    mov     rbp, rsp
    
    ; ใน x64 ต้อง align stack 16 bytes
    ; และต้องจัดสรร shadow space (32 bytes) สำหรับ callee ก่อนจะเรียกใครต่อ
    ; แต่ในฟังก์ชันนี้ไม่ได้เรียกฟังก์ชันอื่น จึงไม่ต้อง
    
    ; Arguments อยู่ใน registers:
    ; RCX = a
    ; RDX = b
    ; R8  = c
    ; R9  = d
    
    mov     rax, rcx            ; rax = a
    add     rax, rdx            ; rax = a + b
    add     rax, r8             ; rax = a + b + c
    add     rax, r9             ; rax = a + b + c + d
    
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
; ฟังก์ชัน main
; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    
    ; Align stack และจัดสรร shadow space
    ; Shadow space = 32 bytes (สำหรับ callee บันทึก args ถ้าต้องการ)
    sub     rsp, 32 + 8         ; 32 bytes shadow + 8 bytes alignment
    
    ; เรียก SumFour(10, 20, 30, 40)
    mov     rcx, 10             ; a = 10
    mov     rdx, 20             ; b = 20
    mov     r8,  30             ; c = 30
    mov     r9,  40             ; d = 40
    call    SumFour
    ; ผลลัพธ์ใน RAX = 100
    
    ; แสดงผล
    mov     rdx, rax            ; rdx = result (argument 2 สำหรับ printf)
    lea     rcx, [fmt_result]   ; rcx = format string (argument 1)
    call    printf
    
    ; cleanup และ return
    add     rsp, 32 + 8
    xor     eax, eax
    pop     rbp
    ret
```

---

## 5. Shadow Space (Home Space)

### Shadow Space คืออะไร?

**Shadow Space** (หรือ Home Space) คือพื้นที่ 32 bytes บน stack ที่ **caller ต้องจัดสรร**ก่อนเรียกฟังก์ชัน เพื่อให้ callee สามารถ "spill" register arguments ลง stack ได้เมื่อต้องการ

```
Stack layout ใน Microsoft x64 ABI:

[RSP+0 ] = Return Address
[RSP+8 ] = Shadow Space for RCX (argument 1)  ← 8 bytes
[RSP+16] = Shadow Space for RDX (argument 2)  ← 8 bytes
[RSP+24] = Shadow Space for R8  (argument 3)  ← 8 bytes
[RSP+32] = Shadow Space for R9  (argument 4)  ← 8 bytes
[RSP+40] = Argument 5 (if any)
[RSP+48] = Argument 6 (if any)
...
```

**หมายเหตุสำคัญ**: Caller ต้องจัดสรร shadow space แต่ไม่จำเป็นต้องเขียนค่าลงไป Callee อาจใช้หรือไม่ใช้ก็ได้

### ตัวอย่างที่ 6: การใช้ Shadow Space อย่างถูกต้อง

```nasm
; ============================================================
; ไฟล์: shadow_space.asm
; คำอธิบาย: การจัดการ Shadow Space ใน Microsoft x64
; ============================================================

bits 64
default rel

global main
extern printf
extern strlen

section .data
    greeting    db "Hello, World!", 0
    fmt_len     db "String '%s' has length %lld", 10, 0
    fmt_debug   db "In ProcessString: ptr=0x%llx", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; void ProcessString(const char* str)
; Arguments: RCX = str
; 
; ฟังก์ชันนี้ spill argument ลง shadow space เพื่อเรียกฟังก์ชันอื่น
; ─────────────────────────────────────────────────────────
ProcessString:
    push    rbp
    mov     rbp, rsp
    
    ; จัดสรร shadow space สำหรับฟังก์ชันที่จะเรียก + alignment
    sub     rsp, 32 + 8         ; 32 bytes shadow + 8 pad for alignment
    
    ; บันทึก RCX (string pointer) ลง shadow space ของตัวเอง
    ; เพราะหลังจากเรียก strlen แล้ว RCX จะถูก overwrite
    mov     [rbp+16], rcx       ; shadow space ของเราอยู่ที่ [RBP+16] ถึง [RBP+47]
                                ; (RBP+8=return addr, RBP+16=shadow for arg1, etc.)
    
    ; เรียก strlen(str)
    ; RCX ยังคงชี้ที่ str อยู่ (ไม่ต้องตั้งค่าใหม่)
    call    strlen
    ; RAX = length of string
    
    ; คืนค่า string pointer จาก shadow space
    mov     rcx, [rbp+16]       ; rcx = str (คืนค่าเดิม)
    
    ; แสดงผล: printf(fmt_len, str, length)
    mov     r8, rax             ; r8 = length (argument 3)
    mov     rdx, rcx            ; rdx = str (argument 2)
    lea     rcx, [fmt_len]      ; rcx = format (argument 1)
    call    printf
    
    add     rsp, 32 + 8
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; เรียก ProcessString(greeting)
    lea     rcx, [greeting]
    call    ProcessString
    
    add     rsp, 32 + 8
    xor     eax, eax
    pop     rbp
    ret
```

### ตัวอย่างที่ 7: Nested Function Calls กับ Shadow Space

```nasm
; ============================================================
; ไฟล์: nested_calls.asm
; คำอธิบาย: Nested calls ใน x64 และการจัดการ shadow space
; ============================================================

bits 64
default rel

global main
extern printf
extern malloc
extern free

section .data
    fmt_alloc   db "Allocated %lld bytes at 0x%llx", 10, 0
    fmt_free    db "Freed memory at 0x%llx", 10, 0
    fmt_fail    db "Allocation failed!", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; void* SafeAlloc(size_t size)
; เรียก malloc และตรวจสอบผลลัพธ์
; Arguments: RCX = size
; Return: RAX = pointer (หรือ NULL ถ้า fail)
; ─────────────────────────────────────────────────────────
SafeAlloc:
    push    rbp
    mov     rbp, rsp
    push    rbx                 ; preserve callee-saved register
    sub     rsp, 32 + 8         ; shadow space + alignment
    
    ; บันทึก size ใน RBX (callee-saved, จะยังอยู่หลัง malloc)
    mov     rbx, rcx
    
    ; เรียก malloc(size)
    ; RCX ยังเป็น size อยู่
    call    malloc
    ; RAX = pointer หรือ NULL
    
    ; ตรวจสอบผลลัพธ์
    test    rax, rax
    jz      .alloc_failed
    
    ; แสดงผล: printf(fmt_alloc, size, pointer)
    mov     r8, rax             ; r8 = pointer (argument 3)
    mov     rdx, rbx            ; rdx = size (argument 2)
    lea     rcx, [fmt_alloc]    ; rcx = format (argument 1)
    push    rax                 ; บันทึก pointer ก่อนเรียก printf (RAX จะถูก overwrite)
    call    printf
    pop     rax                 ; คืนค่า pointer
    jmp     .done

.alloc_failed:
    lea     rcx, [fmt_fail]
    call    printf
    xor     eax, eax            ; return NULL

.done:
    add     rsp, 32 + 8
    pop     rbx                 ; restore callee-saved register
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
; void SafeFree(void* ptr)
; เรียก free และแสดงข้อมูล
; Arguments: RCX = ptr
; ─────────────────────────────────────────────────────────
SafeFree:
    push    rbp
    mov     rbp, rsp
    push    rbx
    sub     rsp, 32 + 8
    
    ; บันทึก pointer ใน RBX
    mov     rbx, rcx
    
    ; free(ptr)
    call    free
    
    ; แสดงผล: printf(fmt_free, ptr)
    mov     rdx, rbx
    lea     rcx, [fmt_free]
    call    printf
    
    add     rsp, 32 + 8
    pop     rbx
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; จัดสรร 1024 bytes
    mov     rcx, 1024
    call    SafeAlloc
    
    ; ถ้าสำเร็จ ให้ free
    test    rax, rax
    jz      .skip_free
    
    mov     rcx, rax
    call    SafeFree

.skip_free:
    add     rsp, 32 + 8
    xor     eax, eax
    pop     rbp
    ret
```

---

## 6. XMM Registers สำหรับ Float Arguments

### Float Arguments ใน Microsoft x64

ใน Microsoft x64 ABI ถ้า argument เป็น float หรือ double จะใช้ **XMM registers** แทน integer registers:

```
Function call: func(int a, double b, int c, double d)

RCX  = a  (argument 1, integer)
XMM1 = b  (argument 2, double - ใช้ XMM1 ไม่ใช่ XMM0 เพราะนับตำแหน่ง!)
R8   = c  (argument 3, integer)
XMM3 = d  (argument 4, double - ใช้ XMM3 เพราะตำแหน่งที่ 4)

หลักการ: ตำแหน่ง argument (ไม่ว่าชนิดใด) กำหนด register ที่ใช้
```

### ตัวอย่างที่ 8: Mixed Integer และ Float Arguments

```nasm
; ============================================================
; ไฟล์: float_args_x64.asm
; คำอธิบาย: Float arguments ใน Microsoft x64 ABI
; คอมไพล์: nasm -f win64 float_args_x64.asm -o float_args_x64.obj
; ============================================================

bits 64
default rel

global main
extern printf

section .data
    ; Format string สำหรับแสดง float
    fmt_float   db "Circle: radius=%.2f, area=%.6f, perimeter=%.6f", 10, 0
    
    ; Constants
    pi          dq 3.14159265358979323846  ; double precision PI
    two_pi      dq 6.28318530717958647692
    
section .text

; ─────────────────────────────────────────────────────────
; void CalcCircle(double radius)
; คำนวณ area และ perimeter ของวงกลม
; Arguments: XMM0 = radius (argument 1 เป็น double ใช้ XMM0)
; ─────────────────────────────────────────────────────────
CalcCircle:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 48             ; พื้นที่สำหรับ local + shadow + alignment
    
    ; บันทึก radius (XMM0 จะถูก overwrite ตอน call printf)
    movsd   [rbp-8], xmm0       ; บันทึก radius
    
    ; คำนวณ area = pi * r^2
    movsd   xmm0, [rbp-8]       ; xmm0 = radius
    movsd   xmm1, xmm0          ; xmm1 = radius
    mulsd   xmm0, xmm1          ; xmm0 = radius^2
    mulsd   xmm0, [pi]          ; xmm0 = pi * radius^2
    movsd   [rbp-16], xmm0      ; บันทึก area
    
    ; คำนวณ perimeter = 2 * pi * r
    movsd   xmm0, [rbp-8]       ; xmm0 = radius
    mulsd   xmm0, [two_pi]      ; xmm0 = 2*pi*radius
    movsd   [rbp-24], xmm0      ; บันทึก perimeter
    
    ; printf(fmt_float, radius, area, perimeter)
    ; Microsoft x64: float args ไปใน XMM registers ตามตำแหน่ง
    ; RCX = fmt (arg 1, integer/pointer)
    ; XMM1 = radius (arg 2, double)
    ; XMM2 = area (arg 3, double)
    ; XMM3 = perimeter (arg 4, double)
    lea     rcx, [fmt_float]
    movsd   xmm1, [rbp-8]       ; radius
    movsd   xmm2, [rbp-16]      ; area
    movsd   xmm3, [rbp-24]      ; perimeter
    
    ; สำหรับ printf กับ variadic: ต้องใส่ค่า XMM ใน integer registers ด้วย
    ; (variadic convention ใน Windows x64 ต้องการทั้งสอง)
    movq    rdx, xmm1
    movq    r8,  xmm2
    movq    r9,  xmm3
    
    call    printf
    
    add     rsp, 48
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; เรียก CalcCircle(5.0)
    ; XMM0 = 5.0 (radius)
    movsd   xmm0, [radius_val]
    call    CalcCircle
    
    add     rsp, 32 + 8
    xor     eax, eax
    pop     rbp
    ret

section .data
    radius_val  dq 5.0
```

---

## 7. Windows vs Linux x64 Differences

### ความแตกต่างหลักระหว่าง Microsoft x64 และ System V AMD64

```
┌─────────────────────┬──────────────────────┬──────────────────────────┐
│ คุณสมบัติ           │ Microsoft x64        │ System V AMD64 (Linux)   │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Integer args        │ RCX, RDX, R8, R9     │ RDI, RSI, RDX, RCX,     │
│                     │                      │ R8, R9 (6 registers!)    │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Float args          │ XMM0-XMM3            │ XMM0-XMM7                │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Shadow Space        │ 32 bytes (REQUIRED)  │ ไม่มี!                   │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Red Zone            │ ไม่มี                │ 128 bytes below RSP       │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Callee-saved regs   │ RBX,RBP,RDI,RSI,     │ RBX,RBP,R12-R15         │
│                     │ RSP,R12-R15,XMM6-15  │ (XMM ไม่ต้อง preserve)  │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Stack alignment     │ 16 bytes ที่ CALL    │ 16 bytes ที่ CALL        │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ Variadic functions  │ Float ต้องอยู่ใน    │ AL = จำนวน XMM args       │
│                     │ XMM และ integer reg  │ ที่ใช้                   │
└─────────────────────┴──────────────────────┴──────────────────────────┘
```

### ตัวอย่างที่ 9: Cross-platform x64 Code

```nasm
; ============================================================
; ไฟล์: cross_platform_x64.asm
; คำอธิบาย: เขียน code ที่ compile ได้ทั้ง Windows และ Linux
; ============================================================

bits 64
default rel

; ─── Conditional Compilation ───
%ifdef WINDOWS
    %define ARG1 rcx
    %define ARG2 rdx
    %define ARG3 r8
    %define ARG4 r9
    %define SHADOW_SPACE 32
%else
    ; Linux / System V AMD64
    %define ARG1 rdi
    %define ARG2 rsi
    %define ARG3 rdx
    %define ARG4 rcx
    %define SHADOW_SPACE 0
%endif

global main
extern printf

section .data
    fmt_platform    db "Platform-neutral code: %lld + %lld = %lld", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; int64_t Add(int64_t a, int64_t b)
; Works on both Windows and Linux
; ─────────────────────────────────────────────────────────
Add:
    ; ใช้ macro แทน register names จริง
    mov     rax, ARG1           ; rax = a
    add     rax, ARG2           ; rax = a + b
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, SHADOW_SPACE + 32  ; Shadow space + local vars + alignment
    
    ; เรียก Add(100, 200)
    mov     ARG1, 100
    mov     ARG2, 200
    call    Add
    ; RAX = 300
    
    ; printf(fmt_platform, 100, 200, result)
    mov     ARG4, rax           ; result
    mov     ARG3, 200           ; b
    mov     ARG2, 100           ; a
    lea     ARG1, [fmt_platform]
    call    printf
    
    add     rsp, SHADOW_SPACE + 32
    xor     eax, eax
    pop     rbp
    ret
```

### ตัวอย่างที่ 10: Red Zone ใน Linux x64

```nasm
; ============================================================
; ไฟล์: red_zone.asm
; คำอธิบาย: การใช้ Red Zone ใน Linux x64
; คอมไพล์: nasm -f elf64 red_zone.asm -o red_zone.obj (Linux only)
; ============================================================

bits 64
default rel
global main

extern printf

section .data
    fmt_result  db "Result: %d", 10, 0

section .text

; ─────────────────────────────────────────────────────────
; Red Zone: 128 bytes below RSP ที่ signal handlers จะไม่แตะ
; เราสามารถใช้พื้นที่นี้ได้โดยไม่ต้อง adjust RSP
; (Linux only - ไม่มีใน Windows)
; ─────────────────────────────────────────────────────────
LeafFunction:
    ; ฟังก์ชันนี้ไม่เรียกฟังก์ชันอื่น (leaf function)
    ; สามารถใช้ Red Zone [RSP-8] ถึง [RSP-128]
    
    ; เก็บ arguments ใน red zone โดยไม่ adjust RSP
    mov     [rsp-8],  rdi       ; บันทึก a
    mov     [rsp-16], rsi       ; บันทึก b
    
    ; คำนวณบางอย่าง
    mov     rax, rdi
    imul    rax, rsi            ; rax = a * b
    
    ; ใช้ค่าที่เก็บใน red zone
    add     rax, [rsp-8]        ; rax = a*b + a
    add     rax, [rsp-16]       ; rax = a*b + a + b
    
    ret

main:
    push    rbp
    mov     rbp, rsp
    ; ไม่ต้อง sub rsp ถ้าเป็น leaf function และใช้ red zone!
    
    ; เรียก LeafFunction(5, 7) = 5*7 + 5 + 7 = 47
    mov     rdi, 5
    mov     rsi, 7
    call    LeafFunction
    
    ; แสดงผล
    mov     rsi, rax
    lea     rdi, [fmt_result]
    xor     al, al              ; al = 0 (ไม่มี XMM arguments)
    call    printf
    
    xor     eax, eax
    pop     rbp
    ret
```

---

## 8. thiscall สำหรับ C++ Methods

### thiscall Convention

**thiscall** ใช้สำหรับ C++ member functions ใน MSVC (32-bit):

- **`this` pointer** ส่งผ่าน `ECX`
- Arguments อื่น ส่งผ่าน stack (จากขวาไปซ้าย)
- **Callee** cleanup stack (เหมือน stdcall)

```
C++ code:
class MyClass {
    int Add(int a, int b);  // thiscall
};

Assembly:
; ECX = this pointer
; [ESP+4] = a
; [ESP+8] = b
```

### ตัวอย่างที่ 11: Simulating thiscall

```nasm
; ============================================================
; ไฟล์: thiscall_demo.asm
; คำอธิบาย: จำลอง thiscall convention
; ============================================================

bits 32
global _main
extern _printf
extern _malloc
extern _free

section .data
    fmt_result  db "Counter value: %d", 10, 0
    fmt_create  db "Counter created at 0x%08x", 10, 0

; ─── "Class" layout ───
; struct Counter {
;     int value;      offset 0
;     int step;       offset 4
; };

; offset constants
struc Counter
    .value  resd 1      ; +0
    .step   resd 1      ; +4
endstruc
COUNTER_SIZE equ 8

section .text

; ─────────────────────────────────────────────────────────
; Counter* Counter_Create(int initial, int step)
; cdecl: สร้าง Counter object
; ─────────────────────────────────────────────────────────
_Counter_Create:
    push    ebp
    mov     ebp, esp
    push    ebx
    sub     esp, 4
    
    ; malloc(COUNTER_SIZE)
    push    COUNTER_SIZE
    call    _malloc
    add     esp, 4
    test    eax, eax
    jz      .fail
    
    mov     ebx, eax                ; ebx = this pointer
    
    ; ตั้งค่า initial
    mov     ecx, [ebp+8]            ; ecx = initial
    mov     [ebx+Counter.value], ecx
    
    ; ตั้งค่า step
    mov     ecx, [ebp+12]           ; ecx = step
    mov     [ebx+Counter.step], ecx
    
    mov     eax, ebx                ; return this pointer

.fail:
    pop     ebx
    mov     esp, ebp
    pop     ebp
    ret

; ─────────────────────────────────────────────────────────
; void Counter_Increment(Counter* this)
; thiscall: ECX = this pointer
; ─────────────────────────────────────────────────────────
_Counter_Increment:
    ; ECX = this pointer (thiscall convention)
    ; ไม่มี arguments บน stack
    
    mov     eax, [ecx+Counter.step]
    add     [ecx+Counter.value], eax
    
    ret                             ; callee cleanup (no stack args)

; ─────────────────────────────────────────────────────────
; int Counter_GetValue(Counter* this)
; thiscall: ECX = this pointer, return EAX = value
; ─────────────────────────────────────────────────────────
_Counter_GetValue:
    mov     eax, [ecx+Counter.value]
    ret

; ─────────────────────────────────────────────────────────
; void Counter_Add(Counter* this, int amount)
; thiscall: ECX = this, [ESP+4] = amount
; ─────────────────────────────────────────────────────────
_Counter_Add@4:                 ; @4 = 1 stack argument × 4 bytes
    push    ebp
    mov     ebp, esp
    
    ; ECX = this
    ; [EBP+8] = amount
    mov     eax, [ebp+8]
    add     [ecx+Counter.value], eax
    
    pop     ebp
    ret     4                   ; callee cleanup: 1 stack arg = 4 bytes

; ─────────────────────────────────────────────────────────
_main:
    push    ebp
    mov     ebp, esp
    
    ; สร้าง Counter(0, 5) - เริ่มที่ 0, step = 5
    push    5                   ; step
    push    0                   ; initial value
    call    _Counter_Create
    add     esp, 8
    
    ; eax = this pointer
    push    eax                 ; บันทึก this ไว้

    ; แสดงที่อยู่
    push    eax
    push    fmt_create
    call    _printf
    add     esp, 8
    
    pop     eax                 ; คืน this pointer
    mov     ebx, eax            ; เก็บ this ใน EBX
    
    ; เรียก thiscall functions
    mov     ecx, ebx
    call    _Counter_Increment  ; value += 5 → 5
    
    mov     ecx, ebx
    call    _Counter_Increment  ; value += 5 → 10
    
    mov     ecx, ebx
    call    _Counter_Increment  ; value += 5 → 15
    
    ; Add(100)
    push    100
    mov     ecx, ebx
    call    _Counter_Add@4      ; value += 100 → 115
    
    ; GetValue()
    mov     ecx, ebx
    call    _Counter_GetValue   ; eax = 115
    
    push    eax
    push    fmt_result
    call    _printf
    add     esp, 8
    
    ; Free memory
    push    ebx
    call    _free
    add     esp, 4
    
    xor     eax, eax
    pop     ebp
    ret
```

---

## 9. vectorcall สำหรับ SIMD

### vectorcall Convention

**vectorcall** ถูกพัฒนาโดย Microsoft เพื่อรองรับ SIMD operations ที่มีประสิทธิภาพสูง:

- Float/double arguments → XMM0-XMM5 (ถึง 6 ตัว!)
- SIMD vectors (\_\_m128, \_\_m256) → XMM/YMM registers
- Integer arguments → standard (RCX/RDX/R8/R9 ใน x64)
- Callee สามารถใช้ XMM registers ได้โดยตรง

### ตัวอย่างที่ 12: SIMD กับ vectorcall

```nasm
; ============================================================
; ไฟล์: vectorcall_demo.asm
; คำอธิบาย: SIMD operations ด้วย vectorcall-style convention
; คอมไพล์: nasm -f win64 vectorcall_demo.asm -o vectorcall_demo.obj
; ============================================================

bits 64
default rel

global main
extern printf

section .data
    fmt_vec4    db "Vector: [%.2f, %.2f, %.2f, %.2f]", 10, 0
    fmt_dot     db "Dot product: %.6f", 10, 0
    
    ; Test vectors
    vec_a   dd 1.0, 2.0, 3.0, 4.0
    vec_b   dd 5.0, 6.0, 7.0, 8.0

section .text

; ─────────────────────────────────────────────────────────
; __m128 VectorAdd(__m128 a, __m128 b)
; vectorcall: XMM0 = a, XMM1 = b
; Return: XMM0 = a + b
; ─────────────────────────────────────────────────────────
VectorAdd:
    addps   xmm0, xmm1          ; Packed float add: xmm0[0..3] += xmm1[0..3]
    ret

; ─────────────────────────────────────────────────────────
; float DotProduct4(__m128 a, __m128 b)
; คำนวณ dot product ของ 4-element vectors
; XMM0 = a, XMM1 = b
; Return: XMM0 (lowest element) = dot product
; ─────────────────────────────────────────────────────────
DotProduct4:
    mulps   xmm0, xmm1          ; xmm0 = [a0*b0, a1*b1, a2*b2, a3*b3]
    
    ; Horizontal add: sum all 4 elements
    haddps  xmm0, xmm0          ; xmm0 = [a0b0+a1b1, a2b2+a3b3, a0b0+a1b1, a2b2+a3b3]
    haddps  xmm0, xmm0          ; xmm0 = [sum, sum, sum, sum]
    
    ret

; ─────────────────────────────────────────────────────────
; void PrintVector4(__m128 v)
; แสดง 4 floats ใน XMM register
; XMM0 = v
; ─────────────────────────────────────────────────────────
PrintVector4:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 48
    
    ; Extract 4 floats จาก XMM0
    movaps  [rbp-16], xmm0      ; บันทึก vector
    
    ; Convert to double สำหรับ printf
    cvtss2sd    xmm1, [rbp-16+0]    ; element 0 → xmm1
    cvtss2sd    xmm2, [rbp-16+4]    ; element 1 → xmm2
    cvtss2sd    xmm3, [rbp-16+8]    ; element 2 → xmm3
    
    ; element 3 ต้องใส่บน stack (argument 5)
    movss   xmm4, [rbp-16+12]
    cvtss2sd    xmm4, xmm4
    movsd   [rsp+32], xmm4      ; argument 5 บน stack
    
    ; printf: ต้องใส่ double ใน integer regs ด้วยสำหรับ variadic
    lea     rcx, [fmt_vec4]
    movq    rdx, xmm1
    movq    r8,  xmm2
    movq    r9,  xmm3
    call    printf
    
    add     rsp, 48
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 16 + 8    ; shadow + xmm save + alignment
    
    ; โหลด vectors
    movups  xmm0, [vec_a]       ; xmm0 = [1,2,3,4]
    movups  xmm1, [vec_b]       ; xmm1 = [5,6,7,8]
    
    ; บันทึก vectors ก่อนเรียกฟังก์ชัน
    movaps  [rbp-16], xmm0
    movaps  [rbp-32], xmm1
    
    ; แสดง vector a
    movaps  xmm0, [rbp-16]
    call    PrintVector4
    
    ; แสดง vector b
    movaps  xmm0, [rbp-32]
    call    PrintVector4
    
    ; VectorAdd(a, b)
    movaps  xmm0, [rbp-16]
    movaps  xmm1, [rbp-32]
    call    VectorAdd
    ; xmm0 = [6,8,10,12]
    call    PrintVector4
    
    ; DotProduct4(a, b) = 1*5 + 2*6 + 3*7 + 4*8 = 5+12+21+32 = 70
    movaps  xmm0, [rbp-16]
    movaps  xmm1, [rbp-32]
    call    DotProduct4
    ; xmm0 = 70.0
    
    ; printf dot product
    cvtss2sd    xmm1, xmm0
    movq    rdx, xmm1
    lea     rcx, [fmt_dot]
    call    printf
    
    add     rsp, 32 + 16 + 8
    xor     eax, eax
    pop     rbp
    ret
```

---

## 10. การเรียก Windows API จาก Assembly

### ภาพรวม Windows API

Windows API ทุกฟังก์ชันใช้ **stdcall (32-bit)** หรือ **Microsoft x64 ABI (64-bit)**:

```
Windows API Categories:
├── Kernel32.dll  - File, Process, Memory, Thread management
├── User32.dll    - Windows, Messages, Input, GUI
├── GDI32.dll     - Graphics Device Interface
├── Advapi32.dll  - Registry, Security, Services
├── Ws2_32.dll    - Winsock (networking)
└── Ntdll.dll     - NT Native API (low-level)
```

### วิธีเรียก Windows API (32-bit)

```nasm
; Windows 32-bit: ใช้ stdcall
; 1. Import function จาก DLL
extern _MessageBoxA@16      ; @16 = 4 args × 4 bytes

; 2. เรียกใช้
push    MB_OK               ; uType (argument 4)
push    caption_str         ; lpCaption (argument 3)
push    message_str         ; lpText (argument 2)
push    NULL                ; hWnd (argument 1)
call    _MessageBoxA@16     ; stdcall: callee cleans up
```

### วิธีเรียก Windows API (64-bit)

```nasm
; Windows 64-bit: ใช้ Microsoft x64 ABI
; 1. Import function
extern MessageBoxA

; 2. เรียกใช้ (ต้อง align stack + shadow space)
sub     rsp, 32 + 8         ; shadow space

xor     ecx, ecx            ; hWnd = NULL
lea     rdx, [message_str]  ; lpText
lea     r8,  [caption_str]  ; lpCaption
mov     r9d, MB_OK          ; uType
call    MessageBoxA

add     rsp, 32 + 8
```

---

## 11. MessageBox API

### ตัวอย่างที่ 13: MessageBox (32-bit)

```nasm
; ============================================================
; ไฟล์: msgbox32.asm
; คำอธิบาย: เรียก MessageBox ผ่าน Windows API (32-bit)
; คอมไพล์: nasm -f win32 msgbox32.asm -o msgbox32.obj
;           link /subsystem:windows msgbox32.obj user32.lib kernel32.lib
; ============================================================

bits 32
global _WinMain@16

extern _MessageBoxA@16
extern _ExitProcess@4

; MessageBox constants
MB_OK               equ 0
MB_OKCANCEL         equ 1
MB_YESNO            equ 4
MB_ICONINFORMATION  equ 0x40
MB_ICONWARNING      equ 0x30
MB_ICONERROR        equ 0x10

IDOK    equ 1
IDCANCEL equ 2
IDYES   equ 6
IDNO    equ 7

section .data
    msg_text    db "สวัสดี! นี่คือ Assembly MessageBox", 0
    msg_caption db "Assembly Program", 0
    msg_confirm db "คุณต้องการดำเนินการต่อหรือไม่?", 0
    cap_confirm db "Confirm", 0
    msg_yes     db "คุณเลือก Yes!", 0
    msg_no      db "คุณเลือก No", 0
    cap_result  db "Result", 0

section .text

_WinMain@16:
    push    ebp
    mov     ebp, esp
    
    ; ─── แสดง Information MessageBox ───
    push    MB_OK | MB_ICONINFORMATION
    push    msg_caption
    push    msg_text
    push    0                   ; hWnd = NULL
    call    _MessageBoxA@16     ; stdcall: ไม่ต้อง cleanup
    
    ; ─── แสดง Yes/No MessageBox ───
    push    MB_YESNO | MB_ICONWARNING
    push    cap_confirm
    push    msg_confirm
    push    0
    call    _MessageBoxA@16
    ; eax = IDYES หรือ IDNO
    
    cmp     eax, IDYES
    je      .user_said_yes
    
.user_said_no:
    push    MB_OK
    push    cap_result
    push    msg_no
    push    0
    call    _MessageBoxA@16
    jmp     .done
    
.user_said_yes:
    push    MB_OK
    push    cap_result
    push    msg_yes
    push    0
    call    _MessageBoxA@16
    
.done:
    ; ExitProcess(0)
    push    0
    call    _ExitProcess@4
    ; ไม่ return เพราะ ExitProcess ไม่ return
```

### ตัวอย่างที่ 14: MessageBox (64-bit)

```nasm
; ============================================================
; ไฟล์: msgbox64.asm
; คำอธิบาย: MessageBox ใน Windows 64-bit
; คอมไพล์: nasm -f win64 msgbox64.asm -o msgbox64.obj
;           link /subsystem:windows /entry:WinMain msgbox64.obj user32.lib
; ============================================================

bits 64
default rel

global WinMain

extern MessageBoxA
extern ExitProcess

; Constants
MB_OK               equ 0
MB_YESNO            equ 4
MB_ICONQUESTION     equ 0x20
MB_ICONINFORMATION  equ 0x40
IDYES               equ 6

section .data
    msg_hello   db "Hello from 64-bit Assembly!", 0
    cap_hello   db "x64 Assembly", 0
    msg_ask     db "Do you want to see another message?", 0
    cap_ask     db "Question", 0
    msg_bye     db "Goodbye!", 0
    cap_bye     db "Bye", 0

section .text

WinMain:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8         ; shadow space + alignment
    
    ; MessageBoxA(NULL, msg_hello, cap_hello, MB_OK|MB_ICONINFORMATION)
    xor     ecx, ecx                    ; hWnd = NULL
    lea     rdx, [msg_hello]            ; lpText
    lea     r8,  [cap_hello]            ; lpCaption
    mov     r9d, MB_OK | MB_ICONINFORMATION  ; uType
    call    MessageBoxA
    
    ; MessageBoxA(NULL, msg_ask, cap_ask, MB_YESNO|MB_ICONQUESTION)
    xor     ecx, ecx
    lea     rdx, [msg_ask]
    lea     r8,  [cap_ask]
    mov     r9d, MB_YESNO | MB_ICONQUESTION
    call    MessageBoxA
    
    cmp     eax, IDYES
    jne     .skip_bye
    
    ; แสดง goodbye
    xor     ecx, ecx
    lea     rdx, [msg_bye]
    lea     r8,  [cap_bye]
    mov     r9d, MB_OK
    call    MessageBoxA

.skip_bye:
    add     rsp, 32 + 8
    
    ; ExitProcess(0)
    xor     ecx, ecx
    call    ExitProcess
```

---

## 12. CreateFile และ File I/O

### ตัวอย่างที่ 15: File Operations (64-bit)

```nasm
; ============================================================
; ไฟล์: file_io_x64.asm
; คำอธิบาย: File I/O ผ่าน Windows API (64-bit)
; คอมไพล์: nasm -f win64 file_io_x64.asm -o file_io_x64.obj
;           link /subsystem:console /entry:main file_io_x64.obj kernel32.lib
; ============================================================

bits 64
default rel

global main

extern CreateFileA
extern WriteFile
extern ReadFile
extern CloseHandle
extern GetLastError
extern ExitProcess

; CreateFile constants
GENERIC_READ            equ 0x80000000
GENERIC_WRITE           equ 0x40000000
GENERIC_READWRITE       equ 0xC0000000
FILE_SHARE_READ         equ 0x00000001
CREATE_ALWAYS           equ 2
OPEN_EXISTING           equ 3
FILE_ATTRIBUTE_NORMAL   equ 0x80
INVALID_HANDLE_VALUE    equ -1

section .data
    filename    db "test_output.txt", 0
    write_data  db "Hello from Assembly! This is a test file.", 13, 10, 0
    write_data_len equ $ - write_data - 1  ; ไม่นับ null terminator

section .bss
    file_handle     resq 1
    bytes_written   resd 1
    read_buffer     resb 256
    bytes_read      resd 1

section .text

; ─────────────────────────────────────────────────────────
; HANDLE OpenFileForWrite(const char* filename)
; Return: RAX = file handle
; ─────────────────────────────────────────────────────────
OpenFileForWrite:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; CreateFileA(
    ;   lpFileName,       RCX = filename
    ;   dwDesiredAccess,  RDX = GENERIC_WRITE
    ;   dwShareMode,      R8  = 0 (exclusive)
    ;   lpSecurityAttr,   R9  = NULL
    ;   dwCreationDisp,   [RSP+32] = CREATE_ALWAYS
    ;   dwFlagsAndAttr,   [RSP+40] = FILE_ATTRIBUTE_NORMAL
    ;   hTemplateFile     [RSP+48] = NULL
    ; )
    
    ; Arguments 1-4 (registers)
    ; RCX ยังเป็น filename อยู่
    mov     rdx, GENERIC_WRITE
    xor     r8,  r8             ; ShareMode = 0
    xor     r9,  r9             ; SecurityAttr = NULL
    
    ; Arguments 5-7 (stack, หลัง shadow space)
    mov     dword [rsp+32], CREATE_ALWAYS
    mov     dword [rsp+40], FILE_ATTRIBUTE_NORMAL
    mov     qword [rsp+48], 0   ; hTemplateFile = NULL
    
    call    CreateFileA
    ; RAX = file handle หรือ INVALID_HANDLE_VALUE
    
    add     rsp, 32 + 8
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; ─── เปิด/สร้างไฟล์ ───
    lea     rcx, [filename]
    call    OpenFileForWrite
    
    ; ตรวจสอบผลลัพธ์
    cmp     rax, INVALID_HANDLE_VALUE
    je      .file_error
    
    mov     [file_handle], rax  ; บันทึก file handle
    
    ; ─── เขียนข้อมูล ───
    ; WriteFile(hFile, lpBuffer, nBytesToWrite, lpBytesWritten, lpOverlapped)
    mov     rcx, [file_handle]                  ; hFile
    lea     rdx, [write_data]                   ; lpBuffer
    mov     r8d, write_data_len                 ; nBytesToWrite
    lea     r9,  [bytes_written]                ; lpBytesWritten
    mov     qword [rsp+32], 0                   ; lpOverlapped = NULL
    call    WriteFile
    
    test    eax, eax
    jz      .write_error
    
    ; ─── ปิดไฟล์ ───
    mov     rcx, [file_handle]
    call    CloseHandle
    
    ; ─── เปิดไฟล์อีกครั้งเพื่ออ่าน ───
    ; CreateFileA for reading
    lea     rcx, [filename]
    mov     rdx, GENERIC_READ
    mov     r8d, FILE_SHARE_READ
    xor     r9,  r9
    mov     dword [rsp+32], OPEN_EXISTING
    mov     dword [rsp+40], FILE_ATTRIBUTE_NORMAL
    mov     qword [rsp+48], 0
    call    CreateFileA
    
    cmp     rax, INVALID_HANDLE_VALUE
    je      .file_error
    
    mov     [file_handle], rax
    
    ; ─── อ่านข้อมูล ───
    ; ReadFile(hFile, lpBuffer, nBytesToRead, lpBytesRead, lpOverlapped)
    mov     rcx, [file_handle]
    lea     rdx, [read_buffer]
    mov     r8d, 255
    lea     r9,  [bytes_read]
    mov     qword [rsp+32], 0
    call    ReadFile
    
    ; ─── ปิดไฟล์ ───
    mov     rcx, [file_handle]
    call    CloseHandle
    
    jmp     .done

.file_error:
.write_error:
    ; Error handling (simplified)
    call    GetLastError
    ; EAX = error code

.done:
    add     rsp, 32 + 8
    xor     ecx, ecx
    call    ExitProcess
```

---

## 13. VirtualAlloc - Memory Management

### ตัวอย่างที่ 16: Dynamic Memory Allocation (64-bit)

```nasm
; ============================================================
; ไฟล์: virtual_alloc.asm
; คำอธิบาย: VirtualAlloc สำหรับ dynamic memory
; ============================================================

bits 64
default rel

global main

extern VirtualAlloc
extern VirtualFree
extern RtlFillMemory
extern ExitProcess

; VirtualAlloc constants
MEM_COMMIT          equ 0x1000
MEM_RESERVE         equ 0x2000
MEM_COMMIT_RESERVE  equ 0x3000
MEM_RELEASE         equ 0x8000
PAGE_READWRITE      equ 0x04
PAGE_EXECUTE_READ   equ 0x20
PAGE_EXECUTE_READWRITE equ 0x40

section .bss
    mem_ptr     resq 1
    mem_ptr2    resq 1

section .text

; ─────────────────────────────────────────────────────────
; void* AllocAndFill(size_t size, uint8_t fill_byte)
; จัดสรรหน่วยความจำและเติมค่า
; RCX = size, RDX = fill_byte
; Return: RAX = pointer
; ─────────────────────────────────────────────────────────
AllocAndFill:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    rsi
    sub     rsp, 32 + 8
    
    mov     rbx, rcx            ; บันทึก size
    mov     rsi, rdx            ; บันทึก fill_byte
    
    ; VirtualAlloc(NULL, size, MEM_COMMIT|MEM_RESERVE, PAGE_READWRITE)
    ; RCX = lpAddress (NULL = ให้ OS เลือก)
    ; RDX = dwSize
    ; R8  = flAllocationType
    ; R9  = flProtect
    xor     ecx, ecx            ; lpAddress = NULL
    mov     rdx, rbx            ; dwSize = size
    mov     r8d, MEM_COMMIT_RESERVE
    mov     r9d, PAGE_READWRITE
    call    VirtualAlloc
    ; RAX = allocated address หรือ NULL ถ้า fail
    
    test    rax, rax
    jz      .done               ; return NULL
    
    push    rax                 ; บันทึก pointer
    
    ; RtlFillMemory(Destination, Length, Fill)
    ; (Windows kernel function สำหรับเติมหน่วยความจำ)
    mov     rcx, rax            ; Destination
    mov     rdx, rbx            ; Length = size
    mov     r8b, sil            ; Fill = fill_byte
    call    RtlFillMemory
    
    pop     rax                 ; คืน pointer

.done:
    add     rsp, 32 + 8
    pop     rsi
    pop     rbx
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
; int FreeMemory(void* ptr)
; ปลดปล่อยหน่วยความจำที่จัดสรรด้วย VirtualAlloc
; RCX = ptr
; Return: RAX = success (non-zero) หรือ fail (zero)
; ─────────────────────────────────────────────────────────
FreeMemory:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; VirtualFree(lpAddress, dwSize, dwFreeType)
    ; เมื่อ dwFreeType = MEM_RELEASE ต้องใช้ dwSize = 0
    ; RCX = lpAddress
    ; RDX = dwSize = 0
    ; R8  = dwFreeType = MEM_RELEASE
    ; RCX ยังเป็น ptr อยู่
    xor     edx, edx            ; dwSize = 0
    mov     r8d, MEM_RELEASE    ; dwFreeType
    call    VirtualFree
    ; RAX = non-zero ถ้าสำเร็จ
    
    add     rsp, 32 + 8
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; จัดสรร 4096 bytes (1 page) และเติม 0xAB
    mov     rcx, 4096           ; size = 4096 bytes (1 page)
    mov     rdx, 0xAB           ; fill value
    call    AllocAndFill
    
    test    rax, rax
    jz      .alloc_failed
    
    mov     [mem_ptr], rax      ; บันทึก pointer
    
    ; ใช้หน่วยความจำ: เขียนค่าต่างๆ
    mov     rbx, [mem_ptr]
    
    ; เขียน pattern
    mov     qword [rbx+0],   0x0102030405060708
    mov     qword [rbx+8],   0x090A0B0C0D0E0F10
    mov     dword [rbx+16],  0xDEADBEEF
    
    ; ตรวจสอบว่าค่าถูกต้อง
    mov     rax, [rbx+0]
    ; rax ควรจะเป็น 0x0102030405060708
    
    ; ปล่อยหน่วยความจำ
    mov     rcx, [mem_ptr]
    call    FreeMemory
    
    xor     ecx, ecx
    call    ExitProcess

.alloc_failed:
    mov     ecx, 1
    call    ExitProcess
```

### ตัวอย่างที่ 17: Executable Memory (สำหรับ JIT Compilation)

```nasm
; ============================================================
; ไฟล์: jit_demo.asm
; คำอธิบาย: สร้าง executable memory สำหรับ JIT
; หมายเหตุ: ตัวอย่างนี้เพื่อการศึกษาเท่านั้น
; ============================================================

bits 64
default rel

global main

extern VirtualAlloc
extern VirtualProtect
extern VirtualFree
extern ExitProcess

MEM_COMMIT_RESERVE      equ 0x3000
MEM_RELEASE             equ 0x8000
PAGE_READWRITE          equ 0x04
PAGE_EXECUTE_READ       equ 0x20

section .data
    ; Machine code สำหรับ: int add(int a, int b) { return a + b; }
    ; Windows x64: RCX=a, RDX=b, return RAX
    jit_code:
        db 0x8D, 0x04, 0x0A     ; LEA EAX, [RDX+RCX]
        db 0xC3                  ; RET
    jit_code_size equ $ - jit_code

section .bss
    exec_mem        resq 1
    old_protect     resd 1

section .text

main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; จัดสรร RW memory ก่อน (จะ change protection ทีหลัง)
    xor     ecx, ecx
    mov     rdx, 4096
    mov     r8d, MEM_COMMIT_RESERVE
    mov     r9d, PAGE_READWRITE
    call    VirtualAlloc
    
    test    rax, rax
    jz      .fail
    
    mov     [exec_mem], rax
    
    ; Copy JIT code เข้าไปใน allocated memory
    mov     rdi, rax
    lea     rsi, [jit_code]
    mov     ecx, jit_code_size
    rep     movsb
    
    ; เปลี่ยน protection เป็น Execute+Read
    ; VirtualProtect(lpAddress, dwSize, flNewProtect, lpflOldProtect)
    mov     rcx, [exec_mem]
    mov     rdx, 4096
    mov     r8d, PAGE_EXECUTE_READ
    lea     r9,  [old_protect]
    call    VirtualProtect
    
    ; เรียก JIT function ที่ compile แล้ว
    ; ฟังก์ชัน: add(5, 7) ควรได้ 12
    mov     ecx, 5              ; a = 5
    mov     edx, 7              ; b = 7
    call    [exec_mem]          ; indirect call ไปยัง JIT code
    ; EAX = 12
    
    ; Free memory
    mov     rcx, [exec_mem]
    xor     edx, edx
    mov     r8d, MEM_RELEASE
    call    VirtualFree

.fail:
    xor     ecx, ecx
    call    ExitProcess
```

---

## 14. LoadLibrary และ GetProcAddress

### ตัวอย่างที่ 18: Dynamic Loading (64-bit)

```nasm
; ============================================================
; ไฟล์: dynamic_load.asm
; คำอธิบาย: LoadLibrary และ GetProcAddress สำหรับ Dynamic Linking
; ============================================================

bits 64
default rel

global main

extern LoadLibraryA
extern GetProcAddress
extern FreeLibrary
extern ExitProcess
extern GetStdHandle
extern WriteConsoleA

STD_OUTPUT_HANDLE   equ -11

section .data
    ; DLL ที่จะโหลด
    dll_user32      db "user32.dll", 0
    dll_kernel32    db "kernel32.dll", 0
    
    ; Function names
    fn_beep         db "Beep", 0
    fn_sleep        db "Sleep", 0
    fn_messagebox   db "MessageBoxA", 0
    
    ; Messages
    msg_loaded      db "DLL loaded successfully!", 13, 10, 0
    msg_failed      db "Failed to load DLL!", 13, 10, 0
    msg_box_text    db "Loaded via GetProcAddress!", 0
    msg_box_cap     db "Dynamic Loading", 0

section .bss
    hUser32         resq 1      ; user32.dll handle
    hKernel32       resq 1      ; kernel32.dll handle
    pfnBeep         resq 1      ; pointer to Beep function
    pfnSleep        resq 1      ; pointer to Sleep function
    pfnMessageBox   resq 1      ; pointer to MessageBoxA
    hStdOut         resq 1
    bytes_written   resd 1

section .text

; ─────────────────────────────────────────────────────────
; void WriteString(const char* str, int len)
; เขียนข้อความไปยัง console
; RCX = str, RDX = len
; ─────────────────────────────────────────────────────────
WriteString:
    push    rbp
    mov     rbp, rsp
    push    rbx
    push    rsi
    sub     rsp, 32 + 8
    
    mov     rbx, rcx            ; บันทึก str
    mov     rsi, rdx            ; บันทึก len
    
    ; GetStdHandle(STD_OUTPUT_HANDLE)
    mov     ecx, STD_OUTPUT_HANDLE
    call    GetStdHandle
    mov     [hStdOut], rax
    
    ; WriteConsoleA(hConsoleOutput, lpBuffer, nCharsToWrite, lpWritten, NULL)
    mov     rcx, [hStdOut]
    mov     rdx, rbx
    mov     r8d, esi
    lea     r9,  [bytes_written]
    mov     qword [rsp+32], 0
    call    WriteConsoleA
    
    add     rsp, 32 + 8
    pop     rsi
    pop     rbx
    pop     rbp
    ret

; ─────────────────────────────────────────────────────────
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 8
    
    ; ─── โหลด user32.dll ───
    lea     rcx, [dll_user32]
    call    LoadLibraryA
    ; RAX = module handle หรือ NULL ถ้า fail
    
    test    rax, rax
    jz      .load_failed
    
    mov     [hUser32], rax
    
    ; ─── โหลด kernel32.dll ───
    lea     rcx, [dll_kernel32]
    call    LoadLibraryA
    
    test    rax, rax
    jz      .load_failed
    
    mov     [hKernel32], rax
    
    ; ─── หา function pointers ───
    ; GetProcAddress(hModule, lpProcName)
    
    ; Beep
    mov     rcx, [hKernel32]
    lea     rdx, [fn_beep]
    call    GetProcAddress
    mov     [pfnBeep], rax
    
    ; Sleep
    mov     rcx, [hKernel32]
    lea     rdx, [fn_sleep]
    call    GetProcAddress
    mov     [pfnSleep], rax
    
    ; MessageBoxA
    mov     rcx, [hUser32]
    lea     rdx, [fn_messagebox]
    call    GetProcAddress
    mov     [pfnMessageBox], rax
    
    ; ─── ใช้ functions ที่โหลดมา ───
    
    ; เรียก Beep(750, 500) - เสียง 750Hz นาน 500ms
    test    qword [pfnBeep], -1
    jz      .skip_beep
    mov     ecx, 750            ; frequency (Hz)
    mov     edx, 300            ; duration (ms)
    call    [pfnBeep]           ; indirect call

.skip_beep:
    ; เรียก MessageBoxA ผ่าน function pointer
    test    qword [pfnMessageBox], -1
    jz      .skip_msgbox
    
    xor     ecx, ecx
    lea     rdx, [msg_box_text]
    lea     r8,  [msg_box_cap]
    mov     r9d, 0              ; MB_OK
    call    [pfnMessageBox]

.skip_msgbox:
    ; Sleep(1000) - รอ 1 วินาที
    test    qword [pfnSleep], -1
    jz      .skip_sleep
    mov     ecx, 1000
    call    [pfnSleep]

.skip_sleep:
    ; ─── ปลดปล่อย DLLs ───
    mov     rcx, [hUser32]
    call    FreeLibrary
    
    mov     rcx, [hKernel32]
    call    FreeLibrary
    
    xor     ecx, ecx
    call    ExitProcess

.load_failed:
    xor     ecx, ecx
    call    ExitProcess
```

### ตัวอย่างที่ 19: Windows Registry Access

```nasm
; ============================================================
; ไฟล์: registry_demo.asm
; คำอธิบาย: อ่านข้อมูลจาก Windows Registry
; ============================================================

bits 64
default rel

global main

extern RegOpenKeyExA
extern RegQueryValueExA
extern RegCloseKey
extern ExitProcess

; Registry constants
HKEY_LOCAL_MACHINE  equ 0x80000002
HKEY_CURRENT_USER   equ 0x80000001
KEY_READ            equ 0x20019
REG_SZ              equ 1      ; null-terminated string

section .data
    reg_path    db "SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion", 0
    value_name  db "ProductName", 0

section .bss
    hKey            resq 1
    product_name    resb 256
    name_size       resd 1
    value_type      resd 1

section .text

main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32 + 16

    ; ตั้งค่า name_size
    mov     dword [name_size], 255
    
    ; RegOpenKeyExA(hKey, lpSubKey, ulOptions, samDesired, phkResult)
    ; RCX = hKey (HKEY_LOCAL_MACHINE)
    ; RDX = lpSubKey
    ; R8  = ulOptions = 0
    ; R9  = samDesired = KEY_READ
    ; [RSP+32] = phkResult
    mov     ecx, HKEY_LOCAL_MACHINE
    lea     rdx, [reg_path]
    xor     r8d, r8d
    mov     r9d, KEY_READ
    lea     rax, [hKey]
    mov     [rsp+32], rax
    call    RegOpenKeyExA
    
    test    eax, eax            ; ERROR_SUCCESS = 0
    jnz     .reg_error
    
    ; RegQueryValueExA(hKey, lpValueName, lpReserved, lpType, lpData, lpcbData)
    mov     rcx, [hKey]         ; hKey
    lea     rdx, [value_name]   ; lpValueName = "ProductName"
    xor     r8d, r8d            ; lpReserved = NULL
    lea     r9,  [value_type]   ; lpType
    ; Arguments 5-6 บน stack
    lea     rax, [product_name]
    mov     [rsp+32], rax       ; lpData
    lea     rax, [name_size]
    mov     [rsp+40], rax       ; lpcbData
    call    RegQueryValueExA
    
    test    eax, eax
    jnz     .close_key
    
    ; product_name ตอนนี้มีชื่อ OS เช่น "Windows 10 Pro"

.close_key:
    mov     rcx, [hKey]
    call    RegCloseKey

.reg_error:
    add     rsp, 32 + 16
    xor     ecx, ecx
    call    ExitProcess
```

---

## สรุปตาราง Calling Conventions

```
┌─────────────────────────────────────────────────────────────────────┐
│                  สรุป Calling Conventions ทั้งหมด                   │
├──────────────┬──────────────────┬────────────┬──────────────────────┤
│ Convention   │ Register Args    │ Cleanup    │ ใช้ใน               │
├──────────────┼──────────────────┼────────────┼──────────────────────┤
│ cdecl (32)   │ ไม่มี (stack)   │ Caller     │ C/C++ Linux/Mac      │
│ stdcall (32) │ ไม่มี (stack)   │ Callee     │ Win32 API            │
│ fastcall(32) │ ECX, EDX        │ Callee     │ MSVC /Og             │
│ thiscall(32) │ ECX=this        │ Callee     │ C++ MSVC methods     │
│ MS x64       │ RCX,RDX,R8,R9  │ Caller     │ Windows 64-bit ALL   │
│ SysV AMD64   │ RDI,RSI,RDX,   │ Caller     │ Linux/Mac 64-bit     │
│              │ RCX,R8,R9      │            │                      │
│ vectorcall   │ XMM0-XMM5+     │ Caller     │ SIMD-heavy code      │
│              │ integer regs   │            │                      │
└──────────────┴──────────────────┴────────────┴──────────────────────┘

Shadow Space:
├── Microsoft x64: 32 bytes (ALWAYS required before any call)
└── System V AMD64: ไม่มี (แต่มี Red Zone 128 bytes ด้านล่าง RSP)

Stack Alignment:
├── 32-bit: 4 bytes
└── 64-bit: 16 bytes ที่จุด CALL (RSP % 16 == 8 ก่อน call เพราะ call push 8 bytes)
```

---

## 15. แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: ระดับง่าย - stdcall Calculator

```
งาน: สร้างโปรแกรม calculator ด้วย stdcall (32-bit Windows)

ฟังก์ชันที่ต้องสร้าง:
1. int __stdcall Add(int a, int b)
2. int __stdcall Subtract(int a, int b)
3. int __stdcall Multiply(int a, int b)
4. int __stdcall Divide(int a, int b)  ← ต้องตรวจสอบ division by zero

เป้าหมาย:
- ใช้ stdcall convention อย่างถูกต้อง
- แต่ละฟังก์ชันต้องทำ stack cleanup ด้วย RET n
- แสดงผลลัพธ์ด้วย printf (cdecl)
```

```nasm
; โค้ดเริ่มต้น - เติมให้สมบูรณ์
bits 32
global _main
extern _printf

section .data
    fmt_add  db "Add(%d, %d) = %d", 10, 0
    fmt_sub  db "Sub(%d, %d) = %d", 10, 0
    fmt_mul  db "Mul(%d, %d) = %d", 10, 0
    fmt_div  db "Div(%d, %d) = %d", 10, 0
    fmt_err  db "Division by zero!", 10, 0

section .text

; TODO: เขียน Add@8
_Add@8:
    ; เติม code ที่นี่
    ret     8

; TODO: เขียน Subtract@8
_Subtract@8:
    ; เติม code ที่นี่
    ret     8

; TODO: เขียน Multiply@8
_Multiply@8:
    ; เติม code ที่นี่
    ret     8

; TODO: เขียน Divide@8 (ตรวจ division by zero!)
_Divide@8:
    ; เติม code ที่นี่
    ret     8

_main:
    ; ทดสอบทุกฟังก์ชัน
    ; TODO: เรียกฟังก์ชันและแสดงผล
    ret
```

### แบบฝึกหัดที่ 2: ระดับกลาง - x64 String Functions

```
งาน: สร้าง string library ด้วย Microsoft x64 ABI

ฟังก์ชันที่ต้องสร้าง:
1. size_t MyStrlen(const char* str)        - RCX=str
2. char* MyStrcpy(char* dst, const char* src)  - RCX=dst, RDX=src
3. int MyStrcmp(const char* a, const char* b)  - RCX=a, RDX=b
4. char* MyStrcat(char* dst, const char* src)  - RCX=dst, RDX=src

ข้อกำหนด:
- ใช้ Microsoft x64 ABI ถูกต้อง
- จัดการ shadow space อย่างถูกต้อง
- ต้อง preserve callee-saved registers (RBX, R12-R15, dll.)
- ทดสอบทุกฟังก์ชัน

Hint สำหรับ MyStrlen:
; ใช้ SCASB หรือ loop เพื่อหา null terminator
; ระวัง: RCX เป็น arg register แต่ก็เป็น loop counter
; ต้องเลือก register ให้เหมาะสม
```

### แบบฝึกหัดที่ 3: ระดับกลาง - Windows GUI Application

```
งาน: สร้าง Windows GUI application ด้วย Assembly 64-bit

ความต้องการ:
1. แสดง "Welcome" MessageBox ตอนเริ่มต้น
2. ถาม Yes/No ว่าต้องการทำต่อ
3. ถ้า Yes: แสดงข้อมูลระบบ (ชื่อ Computer, OS Version)
4. ถ้า No: แสดง "Goodbye" และออก

APIs ที่ต้องใช้:
- MessageBoxA / MessageBoxW
- GetComputerNameA
- GetVersionExA หรือ RtlGetVersion
```

### แบบฝึกหัดที่ 4: ระดับยาก - Dynamic Plugin System

```
งาน: สร้าง plugin system ที่โหลด DLL dynamically

ความต้องการ:
1. โหลด DLL จาก path ที่กำหนด
2. ค้นหา exported function "PluginInit", "PluginExecute", "PluginCleanup"
3. เรียก PluginInit() เพื่อ initialize
4. เรียก PluginExecute(data, size) เพื่อประมวลผล
5. เรียก PluginCleanup() เพื่อ cleanup
6. Unload DLL

สร้าง:
- main.asm - plugin loader
- plugin.asm - DLL plugin (compile เป็น DLL)
- นำ 2 ส่วนมาทดสอบร่วมกัน
```

### แบบฝึกหัดที่ 5: ระดับผู้เชี่ยวชาญ - JIT Compiler Basics

```
งาน: สร้าง JIT compiler อย่างง่าย

ความต้องการ:
1. รับ "bytecode" เป็น array ของ bytes
2. แปลง bytecode เป็น x64 machine code:
   - Bytecode 0x01 a b = ADD a, b
   - Bytecode 0x02 a b = SUB a, b
   - Bytecode 0x03 a b = MUL a, b
   - Bytecode 0xFF = RETURN
3. จัดสรร executable memory (VirtualAlloc + VirtualProtect)
4. Copy machine code ลงไป
5. Execute the JIT-compiled code
6. แสดงผลลัพธ์

ข้อแนะนำ:
- เริ่มด้วย x64 machine code ง่ายๆ ก่อน
- ใช้ registers R10, R11 สำหรับ operands (volatile, ไม่ต้อง preserve)
- ทำให้ JIT code ใช้ Microsoft x64 ABI
```

---

## คำสั่ง Compilation สรุป

```bash
# ─── 32-bit Windows ───
# stdcall / fastcall / thiscall
nasm -f win32 program.asm -o program.obj
link /subsystem:console program.obj kernel32.lib user32.lib

# 32-bit Windows GUI
nasm -f win32 gui.asm -o gui.obj
link /subsystem:windows gui.obj kernel32.lib user32.lib

# ─── 64-bit Windows ───
# Microsoft x64 ABI
nasm -f win64 program.asm -o program.obj
link /subsystem:console /entry:main program.obj kernel32.lib user32.lib

# 64-bit Windows GUI
nasm -f win64 gui.asm -o gui.obj
link /subsystem:windows /entry:WinMain gui.obj kernel32.lib user32.lib

# ─── 64-bit Linux (System V AMD64) ───
nasm -f elf64 program.asm -o program.o
ld -o program program.o

# หรือใช้ gcc เป็น linker
nasm -f elf64 program.asm -o program.o
gcc -no-pie -o program program.o

# ─── Cross-platform ด้วย conditional compile ───
# Windows:
nasm -f win64 -DWINDOWS program.asm -o program.obj

# Linux:
nasm -f elf64 -DLINUX program.asm -o program.o

# ─── สร้าง DLL (Windows 64-bit) ───
nasm -f win64 plugin.asm -o plugin.obj
link /dll /out:plugin.dll plugin.obj kernel32.lib
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **stdcall**: Callee cleanup stack, ใช้ `RET n`, ชื่อ decorated ด้วย `@n`
2. **fastcall**: ECX/EDX สำหรับ 2 args แรก, เร็วกว่า stdcall
3. **Microsoft x64 ABI**: RCX/RDX/R8/R9, Shadow Space 32 bytes (REQUIRED)
4. **XMM Registers**: สำหรับ float/double arguments ตามตำแหน่ง
5. **Windows vs Linux x64**: ความแตกต่างใน register usage และ shadow space
6. **thiscall**: ECX = `this` pointer, ใช้ใน C++ methods
7. **vectorcall**: XMM0-XMM5 สำหรับ SIMD operations
8. **Windows APIs**: MessageBox, CreateFile, VirtualAlloc, LoadLibrary

### จุดสำคัญที่ต้องจำ

```
✓ stdcall: RET n (n = bytes ของ stack arguments)
✓ Microsoft x64: ต้อง allocate shadow space 32 bytes ก่อน CALL
✓ Stack alignment: 16 bytes ที่จุดที่ CALL ถูกเรียก
✓ Callee-saved registers: ต้อง preserve RBX, RBP, RSI, RDI, R12-R15 (MS x64)
✓ Volatile registers: RAX, RCX, RDX, R8, R9, R10, R11 อาจถูกเปลี่ยนได้
✓ Windows APIs ใช้ Microsoft x64 ABI ทั้งหมดใน 64-bit
```

### บทต่อไป (Part 025)

**Part 025: Inline Assembly ใน C/C++** จะครอบคลุม:
- `__asm` และ `asm volatile` ใน GCC
- MSVC inline assembly
- Constraints ใน GCC inline assembly
- เมื่อไหร่ควรใช้ Inline Assembly
- การ mix C/C++ กับ Assembly อย่างมีประสิทธิภาพ

---

*Part 024 - Assembly Programming Course*  
*หัวข้อ: stdcall, fastcall, Microsoft x64 ABI*  
*ระดับ: Intermediate to Advanced*
