# Part 022: Stack Frames และ Local Variables

## หลักสูตร Assembly Programming ระดับมืออาชีพ
### ตั้งแต่พื้นฐานถึงระดับโลก

---

## สารบัญ

1. [ทบทวน Stack และ Call Stack](#ทบทวน-stack-และ-call-stack)
2. [โครงสร้าง Stack Frame](#โครงสร้าง-stack-frame)
3. [การสร้าง Stack Frame: PUSH EBP / MOV EBP,ESP](#การสร้าง-stack-frame)
4. [การเข้าถึง Parameters: [EBP+8], [EBP+12]](#การเข้าถึง-parameters)
5. [Local Variables: [EBP-4], [EBP-8]](#local-variables)
6. [ENTER/LEAVE Shorthand](#enterleave-shorthand)
7. [Frame Pointer Elimination (FPO)](#frame-pointer-elimination-fpo)
8. [การตรวจสอบ Stack Frames ใน GDB](#การตรวจสอบ-stack-frames-ใน-gdb)
9. [Variable Alignment บน Stack](#variable-alignment-บน-stack)
10. [alloca() เทียบเท่าใน Assembly](#alloca-เทียบเท่าใน-assembly)
11. [VLA (Variable Length Arrays)](#vla-variable-length-arrays)
12. [Frame Chaining](#frame-chaining)
13. [Unwind Information](#unwind-information)
14. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ทบทวน Stack และ Call Stack

### Stack คืออะไร

Stack เป็นโครงสร้างข้อมูลแบบ LIFO (Last In, First Out) ที่ CPU ใช้จัดการการเรียกฟังก์ชัน
โดย ESP (Extended Stack Pointer) ชี้ไปยัง top of stack เสมอ

```
Memory Layout (High Address at top):
┌──────────────────────┐ ← High Address (0xFFFFFFFF)
│   Kernel Space       │
├──────────────────────┤
│   Stack (grows ↓)    │ ← ESP ชี้ที่นี่ (top of stack)
│          │           │
│          ▼           │
├──────────────────────┤
│   ...empty space...  │
├──────────────────────┤
│          ▲           │
│          │           │
│   Heap (grows ↑)     │
├──────────────────────┤
│   BSS Segment        │
├──────────────────────┤
│   Data Segment       │
├──────────────────────┤
│   Text Segment       │ ← Low Address
└──────────────────────┘
```

### Stack Operations พื้นฐาน

```nasm
; PUSH ลด ESP แล้วเขียนข้อมูล
PUSH eax    ; ESP = ESP - 4 แล้วเขียน EAX ไปที่ [ESP]

; POP อ่านข้อมูลแล้วเพิ่ม ESP
POP ebx     ; อ่านจาก [ESP] ใส่ EBX แล้ว ESP = ESP + 4
```

### Call Stack คืออะไร

Call Stack เป็นการ stack หลาย Stack Frame ซ้อนกัน
แต่ละ Stack Frame ตรงกับการเรียกฟังก์ชันหนึ่งครั้ง

```
Call Stack ตัวอย่าง:
┌──────────────────────┐ ← Top of Stack (ESP)
│   Local Vars (c)     │
│   Local Vars (b)     │
│   Saved EBP (b)      │
│   Return Addr (b)    │
│   Parameters (b)     │ ← Stack Frame ของฟังก์ชัน b()
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│   Local Vars (a)     │
│   Saved EBP (a)      │
│   Return Addr (a)    │ ← Stack Frame ของฟังก์ชัน a()
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
│   Local Vars (main)  │
│   Saved EBP (main)   │ ← Stack Frame ของ main()
└──────────────────────┘ ← Bottom of Stack
```

---

## โครงสร้าง Stack Frame

Stack Frame มาตรฐาน (x86 32-bit) มีโครงสร้างดังนี้:

```
Stack Frame Layout สำหรับ func(int a, int b):
                    
Higher Address ↑
┌──────────────────────┐
│   Parameter b        │ ← [EBP + 12]  (+12 bytes จาก EBP)
├──────────────────────┤
│   Parameter a        │ ← [EBP + 8]   (+8 bytes จาก EBP)
├──────────────────────┤
│   Return Address     │ ← [EBP + 4]   (ที่อยู่กลับของผู้เรียก)
├──────────────────────┤
│   Saved EBP          │ ← [EBP + 0]   (EBP ชี้ที่นี่)
├──────────────────────┤
│   Local Var 1        │ ← [EBP - 4]   (-4 bytes จาก EBP)
├──────────────────────┤
│   Local Var 2        │ ← [EBP - 8]   (-8 bytes จาก EBP)
├──────────────────────┤
│   ...                │
└──────────────────────┘ ← ESP (ชี้ที่ตำแหน่งต่ำสุดของ frame)
Lower Address ↓
```

### ทำไมต้องใช้ EBP เป็น Frame Pointer

EBP (Base Pointer) ทำหน้าที่เป็นจุดอ้างอิงคงที่ตลอดการทำงานของฟังก์ชัน
ทำให้ address ของ parameters และ local variables ไม่เปลี่ยนแปลง
แม้ ESP จะเปลี่ยนไปเมื่อมีการ PUSH/POP

```nasm
; ถ้าไม่มี frame pointer:
; ESP เปลี่ยนไปเรื่อยๆ ทำให้ยากในการอ้างอิง variables
PUSH eax        ; ESP - 4
PUSH ebx        ; ESP - 8
; ตอนนี้ parameter แรก อยู่ที่ [ESP + 16] (ต้องคำนวณเอง!)

; ถ้ามี frame pointer (EBP):
; EBP คงที่ตลอด ทำให้ parameter แรกอยู่ที่ [EBP+8] เสมอ
PUSH eax        ; ESP เปลี่ยน แต่ EBP ไม่เปลี่ยน
PUSH ebx        ; parameter ยังอยู่ที่ [EBP+8] เสมอ
```

---

## การสร้าง Stack Frame

### Function Prologue (ส่วนเริ่มต้นของฟังก์ชัน)

```nasm
; Function Prologue มาตรฐาน
my_function:
    ; ขั้นตอนที่ 1: บันทึก EBP ของผู้เรียก
    push    ebp
    ; ตอนนี้ Stack มี: [ESP] = old EBP
    
    ; ขั้นตอนที่ 2: ตั้ง EBP ให้ชี้ที่ ESP ปัจจุบัน
    mov     ebp, esp
    ; ตอนนี้ EBP = ESP = ตำแหน่งของ saved EBP
    
    ; ขั้นตอนที่ 3: จองพื้นที่สำหรับ local variables
    sub     esp, 16     ; จอง 16 bytes สำหรับ local variables
    
    ; ตอนนี้ Stack มี:
    ; [EBP + 8]  = parameter แรก
    ; [EBP + 4]  = return address
    ; [EBP + 0]  = saved EBP  ← EBP ชี้ที่นี่
    ; [EBP - 4]  = local variable 1
    ; [EBP - 8]  = local variable 2
    ; [EBP - 12] = local variable 3
    ; [EBP - 16] = local variable 4  ← ESP ชี้ที่นี่
    
    ; ... code ของฟังก์ชัน ...
```

### Function Epilogue (ส่วนสิ้นสุดของฟังก์ชัน)

```nasm
    ; Function Epilogue มาตรฐาน
    
    ; ขั้นตอนที่ 1: คืนค่า ESP กลับไปที่ EBP
    mov     esp, ebp
    ; หรือใช้ LEAVE instruction ซึ่งทำ mov esp,ebp แล้ว pop ebp
    
    ; ขั้นตอนที่ 2: คืนค่า EBP ของผู้เรียก
    pop     ebp
    
    ; ขั้นตอนที่ 3: กลับไปยัง caller
    ret
    ; หรือ ret N เพื่อ clean stack (stdcall convention)
```

### ตัวอย่างสมบูรณ์: ฟังก์ชัน add สองจำนวน

```nasm
; ไฟล์: stack_frame_basic.asm
; การสร้าง Stack Frame พื้นฐาน
; วิธีคอมไพล์: nasm -f elf32 stack_frame_basic.asm -o stack_frame_basic.o
;              gcc -m32 stack_frame_basic.o -o stack_frame_basic -no-pie

section .data
    fmt_result  db "ผลลัพธ์ %d + %d = %d", 10, 0
    fmt_enter   db "เข้าสู่ฟังก์ชัน add_numbers", 10, 0
    fmt_exit    db "ออกจากฟังก์ชัน add_numbers", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: int add_numbers(int a, int b)
; ส่งคืน: ผลบวกของ a และ b
add_numbers:
    ; === Function Prologue ===
    push    ebp                 ; บันทึก old EBP
    mov     ebp, esp            ; ตั้ง EBP ใหม่
    sub     esp, 8              ; จองพื้นที่ local variables (2 ints = 8 bytes)
    
    ; บันทึก registers ที่ใช้ (callee-saved registers)
    push    ebx
    push    esi
    
    ; === ดึงค่า Parameters ===
    ; [EBP + 8]  = parameter a (parameter แรก)
    ; [EBP + 12] = parameter b (parameter ที่สอง)
    mov     eax, [ebp + 8]      ; eax = a
    mov     ebx, [ebp + 12]     ; ebx = b
    
    ; === ใช้ Local Variables ===
    ; [EBP - 4] = local_a (สำเนาของ a)
    ; [EBP - 8] = local_b (สำเนาของ b)
    mov     [ebp - 4], eax      ; local_a = a
    mov     [ebp - 8], ebx      ; local_b = b
    
    ; === คำนวณ ===
    mov     eax, [ebp - 4]      ; โหลด local_a
    add     eax, [ebp - 8]      ; eax = local_a + local_b
    
    ; ผลลัพธ์อยู่ใน EAX (ตาม calling convention)
    
    ; === Function Epilogue ===
    pop     esi                 ; คืน registers
    pop     ebx
    
    mov     esp, ebp            ; คืน ESP
    pop     ebp                 ; คืน EBP ของ caller
    ret                         ; กลับไปยัง caller

main:
    ; === Function Prologue ของ main ===
    push    ebp
    mov     ebp, esp
    sub     esp, 8              ; จองพื้นที่ local variables
    
    ; === เรียกฟังก์ชัน add_numbers(10, 25) ===
    ; Push parameters ลำดับย้อนกลับ (right to left - cdecl convention)
    push    25                  ; parameter b
    push    10                  ; parameter a
    call    add_numbers         ; เรียกฟังก์ชัน
    add     esp, 8              ; clean stack (cdecl: caller cleans)
    ; ตอนนี้ EAX = ผลลัพธ์ = 35
    
    ; บันทึกผลลัพธ์
    mov     [ebp - 4], eax      ; local_result = eax
    
    ; แสดงผล
    push    dword [ebp - 4]     ; ผลลัพธ์
    push    25                  ; b
    push    10                  ; a
    push    fmt_result
    call    printf
    add     esp, 16
    
    ; === Function Epilogue ของ main ===
    mov     eax, 0              ; return 0
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
# คอมไพล์ไฟล์ assembly
nasm -f elf32 stack_frame_basic.asm -o stack_frame_basic.o

# Link กับ C library
gcc -m32 stack_frame_basic.o -o stack_frame_basic -no-pie

# รัน
./stack_frame_basic
# Output: ผลลัพธ์ 10 + 25 = 35
```

---

## การเข้าถึง Parameters

### กฎการเข้าถึง Parameters ใน cdecl Convention

Parameters ถูก push จากขวาไปซ้าย (right to left)
ทำให้ parameter แรกอยู่ที่ [EBP+8] เสมอ

```
ตัวอย่าง: func(int a, int b, int c)
                                    
Stack หลัง CALL และ PUSH EBP:
┌────────────────┐
│  c (3rd param) │ [EBP + 16]  pushed ก่อน
├────────────────┤
│  b (2nd param) │ [EBP + 12]
├────────────────┤
│  a (1st param) │ [EBP + 8]   pushed หลัง
├────────────────┤
│ Return Address │ [EBP + 4]   CALL instruction เพิ่มให้
├────────────────┤
│   Saved EBP    │ [EBP + 0]   PUSH EBP เพิ่มให้  ← EBP ชี้
└────────────────┘
```

### ตัวอย่าง: ฟังก์ชันที่มี Parameters หลายตัว

```nasm
; ไฟล์: multiple_params.asm
; การเข้าถึง Parameters หลายตัวใน Stack Frame
; วิธีคอมไพล์: nasm -f elf32 multiple_params.asm -o multiple_params.o
;              gcc -m32 multiple_params.o -o multiple_params -no-pie

section .data
    fmt_calc    db "คำนวณ: (%d + %d) * %d - %d = %d", 10, 0
    fmt_show    db "a=%d, b=%d, c=%d, d=%d", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: int calculate(int a, int b, int c, int d)
; สูตร: (a + b) * c - d
; Parameters:
;   a = [EBP + 8]
;   b = [EBP + 12]
;   c = [EBP + 16]
;   d = [EBP + 20]
calculate:
    push    ebp
    mov     ebp, esp
    
    ; ไม่ต้องการ local variables ในตัวอย่างนี้
    ; แต่จองไว้เพื่อ alignment
    sub     esp, 4
    
    ; ดึง parameters
    mov     eax, [ebp + 8]      ; eax = a
    mov     ebx, [ebp + 12]     ; ebx = b
    mov     ecx, [ebp + 16]     ; ecx = c
    mov     edx, [ebp + 20]     ; edx = d
    
    ; คำนวณ (a + b)
    add     eax, ebx            ; eax = a + b
    
    ; คูณด้วย c
    imul    eax, ecx            ; eax = (a + b) * c
    
    ; ลบ d
    sub     eax, edx            ; eax = (a + b) * c - d
    
    ; EAX = ผลลัพธ์
    mov     esp, ebp
    pop     ebp
    ret

; ฟังก์ชัน: void show_params(int a, int b, int c, int d)
; แสดง parameters ทั้งหมด
show_params:
    push    ebp
    mov     ebp, esp
    
    ; แสดงค่า parameters
    push    dword [ebp + 20]    ; d
    push    dword [ebp + 16]    ; c
    push    dword [ebp + 12]    ; b
    push    dword [ebp + 8]     ; a
    push    fmt_show
    call    printf
    add     esp, 20
    
    mov     esp, ebp
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    sub     esp, 4              ; local: int result
    
    ; เรียก show_params(5, 3, 4, 2)
    push    2                   ; d (push ขวาก่อน)
    push    4                   ; c
    push    3                   ; b
    push    5                   ; a (push ซ้ายหลัง)
    call    show_params
    add     esp, 16
    
    ; เรียก calculate(5, 3, 4, 2)
    ; สูตร: (5 + 3) * 4 - 2 = 30
    push    2                   ; d
    push    4                   ; c
    push    3                   ; b
    push    5                   ; a
    call    calculate
    add     esp, 16
    
    mov     [ebp - 4], eax      ; บันทึกผลลัพธ์
    
    ; แสดงผล
    push    dword [ebp - 4]     ; result
    push    2                   ; d
    push    4                   ; c
    push    3                   ; b
    push    5                   ; a
    push    fmt_calc
    call    printf
    add     esp, 24
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 multiple_params.asm -o multiple_params.o
gcc -m32 multiple_params.o -o multiple_params -no-pie
./multiple_params
# Output:
# a=5, b=3, c=4, d=2
# คำนวณ: (5 + 3) * 4 - 2 = 30
```

### Parameters ขนาดใหญ่กว่า 4 bytes

```nasm
; ตัวอย่างการส่ง 64-bit integer (long long) เป็น parameter
; long long func(long long a, int b)
; a ใช้สอง slot: [EBP+8] (low 32 bits) และ [EBP+12] (high 32 bits)
; b อยู่ที่ [EBP+16]

func_64bit:
    push    ebp
    mov     ebp, esp
    
    ; โหลด 64-bit parameter a
    mov     eax, [ebp + 8]      ; low 32 bits ของ a
    mov     edx, [ebp + 12]     ; high 32 bits ของ a
    ; EDX:EAX = a (64-bit)
    
    ; โหลด int parameter b
    mov     ecx, [ebp + 16]     ; b
    
    ; ... ทำงานกับ a และ b ...
    
    mov     esp, ebp
    pop     ebp
    ret
```

---

## Local Variables

### การจัดการ Local Variables

```nasm
; ตัวอย่าง Local Variables หลายชนิด
; int func() {
;     int   x = 10;         // 4 bytes
;     char  c = 'A';        // 1 byte (aligned to 4)
;     short s = 100;        // 2 bytes (aligned to 4)
;     int   arr[3];         // 12 bytes
; }

func_locals:
    push    ebp
    mov     ebp, esp
    
    ; จองพื้นที่สำหรับ local variables
    ; ต้องจัดให้ aligned ด้วย
    ; x:   4 bytes  → [EBP - 4]
    ; c:   4 bytes  → [EBP - 8]  (aligned แม้จะใช้แค่ 1 byte)
    ; s:   4 bytes  → [EBP - 12] (aligned แม้จะใช้แค่ 2 bytes)
    ; arr: 12 bytes → [EBP - 24] (3 * 4 bytes)
    ; รวม: 24 bytes
    sub     esp, 24
    
    ; กำหนดค่า local variables
    mov     dword [ebp - 4], 10     ; x = 10
    mov     byte  [ebp - 8], 'A'    ; c = 'A'
    mov     word  [ebp - 12], 100   ; s = 100
    
    ; กำหนดค่า array arr[0], arr[1], arr[2]
    ; arr อยู่ที่ [EBP - 24] ถึง [EBP - 13]
    ; arr[0] = [EBP - 24]
    ; arr[1] = [EBP - 20]
    ; arr[2] = [EBP - 16]
    mov     dword [ebp - 24], 1     ; arr[0] = 1
    mov     dword [ebp - 20], 2     ; arr[1] = 2
    mov     dword [ebp - 16], 3     ; arr[2] = 3
    
    ; เข้าถึง array ด้วย index
    ; arr[i] = [EBP - 24 + i*4]
    ; ตัวอย่าง: อ่าน arr[1]
    mov     eax, 1                  ; index = 1
    lea     ecx, [ebp - 24]         ; ecx = &arr[0]
    mov     eax, [ecx + eax*4]      ; eax = arr[1]
    
    mov     esp, ebp
    pop     ebp
    ret
```

### ตัวอย่างสมบูรณ์: Local Variables และ Arrays

```nasm
; ไฟล์: local_variables.asm
; การใช้ Local Variables และ Arrays ใน Stack Frame
; วิธีคอมไพล์: nasm -f elf32 local_variables.asm -o local_variables.o
;              gcc -m32 local_variables.o -o local_variables -no-pie

section .data
    fmt_vars    db "x=%d, c=%c, s=%d", 10, 0
    fmt_arr     db "arr[%d] = %d", 10, 0
    fmt_sum     db "ผลรวม array = %d", 10, 0
    fmt_title   db "=== ทดสอบ Local Variables ===", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: void demo_locals(void)
; แสดงการใช้ local variables หลายชนิด
demo_locals:
    push    ebp
    mov     ebp, esp
    
    ; จัดสรร local variables:
    ; [ebp -  4] = int x
    ; [ebp -  8] = int c (char แต่ aligned เป็น int)
    ; [ebp - 12] = int s (short แต่ aligned เป็น int)
    ; [ebp - 24] = int arr[3] (12 bytes)
    sub     esp, 24
    
    ; เริ่มต้น local variables
    mov     dword [ebp - 4], 42     ; x = 42
    mov     dword [ebp - 8], 'Z'    ; c = 'Z'
    mov     dword [ebp - 12], 999   ; s = 999
    
    ; แสดงค่า local variables
    push    dword [ebp - 12]        ; s
    push    dword [ebp - 8]         ; c
    push    dword [ebp - 4]         ; x
    push    fmt_vars
    call    printf
    add     esp, 16
    
    ; เริ่มต้น array
    mov     dword [ebp - 24], 10    ; arr[0] = 10
    mov     dword [ebp - 20], 20    ; arr[1] = 20
    mov     dword [ebp - 16], 30    ; arr[2] = 30
    
    ; แสดงและรวม array
    xor     ecx, ecx                ; i = 0
    xor     esi, esi                ; sum = 0
    
.loop:
    cmp     ecx, 3
    jge     .done
    
    ; อ่าน arr[i]
    lea     eax, [ebp - 24]         ; eax = &arr
    mov     edx, [eax + ecx*4]      ; edx = arr[i]
    add     esi, edx                ; sum += arr[i]
    
    ; แสดง arr[i]
    push    edx                     ; arr[i]
    push    ecx                     ; i
    push    fmt_arr
    call    printf
    add     esp, 12
    
    inc     ecx
    jmp     .loop
    
.done:
    ; แสดงผลรวม
    push    esi
    push    fmt_sum
    call    printf
    add     esp, 8
    
    mov     esp, ebp
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    
    ; แสดง title
    push    fmt_title
    call    printf
    add     esp, 4
    
    ; เรียกฟังก์ชัน
    call    demo_locals
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 local_variables.asm -o local_variables.o
gcc -m32 local_variables.o -o local_variables -no-pie
./local_variables
```

**ผลลัพธ์ที่คาดหวัง:**
```
=== ทดสอบ Local Variables ===
x=42, c=Z, s=999
arr[0] = 10
arr[1] = 20
arr[2] = 30
ผลรวม array = 60
```

---

## ENTER/LEAVE Shorthand

### ENTER instruction

ENTER เป็น instruction ที่รวม prologue code ให้อัตโนมัติ

```nasm
; ENTER bytes, level
; bytes = จำนวน bytes สำหรับ local variables
; level = nesting level (0 สำหรับฟังก์ชันปกติ)

; แทน:
;   push    ebp
;   mov     ebp, esp
;   sub     esp, 16

; ใช้:
    enter   16, 0       ; จอง 16 bytes สำหรับ local variables

; ทั้งสองแบบเหมือนกัน แต่ ENTER ช้ากว่าในหน่วยประมวลผลสมัยใหม่
```

### LEAVE instruction

LEAVE รวม epilogue code ให้อัตโนมัติ

```nasm
; LEAVE ทำงานเทียบเท่ากับ:
;   mov     esp, ebp
;   pop     ebp

; แทน:
;   mov     esp, ebp
;   pop     ebp
;   ret

; ใช้:
    leave
    ret
```

### ตัวอย่าง ENTER/LEAVE

```nasm
; ไฟล์: enter_leave.asm
; การใช้ ENTER/LEAVE instructions
; วิธีคอมไพล์: nasm -f elf32 enter_leave.asm -o enter_leave.o
;              gcc -m32 enter_leave.o -o enter_leave -no-pie

section .data
    fmt_result  db "ผลลัพธ์ = %d", 10, 0
    fmt_both    db "Traditional: %d, ENTER/LEAVE: %d", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: int traditional_prologue(int a, int b)
; ใช้ prologue/epilogue แบบดั้งเดิม
traditional_prologue:
    push    ebp             ; แบบดั้งเดิม
    mov     ebp, esp
    sub     esp, 8          ; จอง 2 local variables
    
    mov     eax, [ebp + 8]
    add     eax, [ebp + 12]
    mov     [ebp - 4], eax  ; local_sum = a + b
    mov     eax, [ebp - 4]
    
    mov     esp, ebp
    pop     ebp
    ret

; ฟังก์ชัน: int enter_leave_version(int a, int b)
; ใช้ ENTER/LEAVE
enter_leave_version:
    enter   8, 0            ; เทียบเท่า push ebp; mov ebp,esp; sub esp,8
    
    mov     eax, [ebp + 8]
    add     eax, [ebp + 12]
    mov     [ebp - 4], eax  ; local_sum = a + b
    mov     eax, [ebp - 4]
    
    leave                   ; เทียบเท่า mov esp,ebp; pop ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    sub     esp, 8          ; 2 local variables
    
    ; เรียกแบบดั้งเดิม
    push    20
    push    15
    call    traditional_prologue
    add     esp, 8
    mov     [ebp - 4], eax  ; บันทึกผลลัพธ์แรก
    
    ; เรียกแบบ ENTER/LEAVE
    push    20
    push    15
    call    enter_leave_version
    add     esp, 8
    mov     [ebp - 8], eax  ; บันทึกผลลัพธ์ที่สอง
    
    ; แสดงผลลัพธ์เปรียบเทียบ
    push    dword [ebp - 8]
    push    dword [ebp - 4]
    push    fmt_both
    call    printf
    add     esp, 12
    
    xor     eax, eax
    leave
    ret
```

**หมายเหตุ:** ENTER/LEAVE นั้น syntactically สะดวก แต่บน CPU สมัยใหม่
การใช้ PUSH/MOV/SUB และ MOV/POP ตามลำดับมักเร็วกว่า
เนื่องจาก microcode implementation ของ ENTER ไม่ได้ optimize ดีนัก

---

## Frame Pointer Elimination (FPO)

FPO คือเทคนิคที่ compiler ไม่ใช้ EBP เป็น frame pointer
เพื่อให้ EBP ใช้เป็น general-purpose register แทน

### ข้อดีและข้อเสียของ FPO

**ข้อดี:**
- ได้ register เพิ่มอีกหนึ่งตัว (EBP)
- ลดจำนวน instruction ใน prologue/epilogue
- ประสิทธิภาพดีขึ้นเล็กน้อย

**ข้อเสีย:**
- Debug ยากขึ้น (ไม่มี frame chain)
- Stack trace อาจไม่สมบูรณ์ใน debugger
- Exception handling ซับซ้อนขึ้น

### FPO ใน Assembly

```nasm
; ฟังก์ชันแบบ FPO (ไม่ใช้ EBP)
func_fpo:
    ; ไม่มี prologue สำหรับ frame pointer
    ; ใช้ ESP โดยตรง
    
    sub     esp, 8              ; จอง local variables
    
    ; Parameter แรก = [ESP + 12]  (8 bytes local + 4 bytes return addr)
    ; Parameter สอง = [ESP + 16]
    mov     eax, [esp + 12]     ; a
    add     eax, [esp + 16]     ; a + b
    
    mov     [esp], eax          ; local_var = a + b
    mov     eax, [esp]          ; return value
    
    add     esp, 8              ; คืน stack
    ret
    
; หมายเหตุ: เมื่อมีการ PUSH/POP หรือ CALL ใน function body
; offset จาก ESP จะเปลี่ยนไปต้องคำนวณใหม่ทุกครั้ง
```

### ตัวอย่าง: เปรียบเทียบ FPO กับ Non-FPO

```nasm
; ไฟล์: fpo_comparison.asm
; เปรียบเทียบ FPO กับ non-FPO
; วิธีคอมไพล์: nasm -f elf32 fpo_comparison.asm -o fpo_comparison.o
;              gcc -m32 fpo_comparison.o -o fpo_comparison -no-pie

section .data
    fmt_nofpo   db "Non-FPO result: %d", 10, 0
    fmt_fpo     db "FPO result: %d", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชันแบบปกติ (non-FPO)
multiply_nofpo:
    push    ebp
    mov     ebp, esp
    sub     esp, 4              ; local: int temp
    
    mov     eax, [ebp + 8]      ; a
    imul    eax, [ebp + 12]     ; eax = a * b
    mov     [ebp - 4], eax      ; temp = a * b
    
    ; ใช้ EBP ทำงานต่อได้สะดวก
    mov     eax, [ebp - 4]      ; return temp
    
    mov     esp, ebp
    pop     ebp
    ret

; ฟังก์ชันแบบ FPO (EBP ว่างสำหรับใช้เป็น general register)
multiply_fpo:
    ; ไม่มี push ebp / mov ebp,esp
    sub     esp, 4              ; จองพื้นที่สำหรับ local variable
    
    ; [esp + 4]  = return address
    ; [esp + 8]  = parameter a
    ; [esp + 12] = parameter b
    ; [esp + 0]  = local temp
    
    mov     eax, [esp + 8]      ; a
    imul    eax, [esp + 12]     ; eax = a * b
    
    ; ตอนนี้ EBP ว่าง สามารถใช้เป็น general register ได้
    mov     ebp, eax            ; ใช้ EBP เป็น temp register
    ; ... ทำงานอื่นๆ กับ EBP ...
    mov     eax, ebp            ; return value
    
    add     esp, 4              ; คืน stack
    ret

main:
    push    ebp
    mov     ebp, esp
    
    ; เรียก non-FPO
    push    6
    push    7
    call    multiply_nofpo
    add     esp, 8
    
    push    eax
    push    fmt_nofpo
    call    printf
    add     esp, 8
    
    ; เรียก FPO
    push    6
    push    7
    call    multiply_fpo
    add     esp, 8
    
    push    eax
    push    fmt_fpo
    call    printf
    add     esp, 8
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

---

## การตรวจสอบ Stack Frames ใน GDB

### คำสั่ง GDB สำหรับ Stack Frames

```bash
# คอมไพล์พร้อม debug symbols
nasm -f elf32 -g -F dwarf stack_frame_basic.asm -o stack_frame_basic.o
gcc -m32 -g stack_frame_basic.o -o stack_frame_basic -no-pie

# เริ่ม GDB
gdb ./stack_frame_basic
```

### คำสั่ง GDB ที่สำคัญ

```gdb
# ตั้ง breakpoint
(gdb) break add_numbers       ; หยุดที่ฟังก์ชัน add_numbers
(gdb) break *add_numbers+5    ; หยุดที่ offset 5 bytes

# รันโปรแกรม
(gdb) run

# ดู Stack Frame ปัจจุบัน
(gdb) frame                   ; แสดง frame ปัจจุบัน
(gdb) info frame              ; แสดงรายละเอียด frame

# ดู Stack backtrace
(gdb) backtrace               ; หรือ bt - แสดง call stack ทั้งหมด
(gdb) bt full                 ; แสดงพร้อม local variables

# ดูค่า registers
(gdb) info registers          ; ดู registers ทั้งหมด
(gdb) print $esp              ; ดูค่า ESP
(gdb) print $ebp              ; ดูค่า EBP

# ดู memory
(gdb) x/16xw $esp             ; แสดง 16 words จาก ESP
(gdb) x/8xw $ebp-24          ; แสดง 8 words จาก EBP-24

# ดู local variables
(gdb) info locals             ; ดู local variables ทั้งหมด

# ดู parameters
(gdb) info args               ; ดู function parameters

# เดิน code
(gdb) stepi                   ; si - เดิน 1 instruction
(gdb) nexti                   ; ni - เดิน 1 instruction (ข้าม CALL)
(gdb) continue                ; c - รันต่อ

# ดู assembly
(gdb) disassemble             ; ดู assembly ของ function ปัจจุบัน
(gdb) disassemble /m          ; พร้อม source code
```

### Script GDB อัตโนมัติ

```bash
# สร้างไฟล์ gdb_commands.txt
cat > gdb_commands.txt << 'EOF'
break add_numbers
run
echo === Stack Frame เมื่อเข้าสู่ add_numbers ===
info frame
echo === Registers ===
info registers eax ebx esp ebp
echo === Stack memory ===
x/8xw $ebp-8
echo === Parameters ===
x/4xw $ebp
echo === ค่า Parameters ===
print *(int*)($ebp+8)
print *(int*)($ebp+12)
continue
quit
EOF

# รัน GDB พร้อม script
gdb -batch -x gdb_commands.txt ./stack_frame_basic
```

### ตัวอย่าง GDB Session

```
(gdb) break add_numbers
Breakpoint 1 at 0x...

(gdb) run
Breakpoint 1, add_numbers () at stack_frame_basic.asm:XX

(gdb) info frame
Stack level 0, frame at 0xffffd000:
 eip = 0x... in add_numbers; saved eip = 0x...
 called by frame at 0xffffd020
 Arglist at 0xffffcff8, args: 
 Locals at 0xffffcff8, Previous frame's sp = 0xffffd000

(gdb) x/8xw $ebp-4
0xffffd000:  [EBP value]  [return addr]  [param a]  [param b]
            ^^saved EBP   ^^ret addr     ^^10       ^^25

(gdb) backtrace
#0  add_numbers () at stack_frame_basic.asm:XX
#1  0x... in main () at stack_frame_basic.asm:XX
```

---

## Variable Alignment บน Stack

### ทำไมต้อง Align Variables

CPU เข้าถึงข้อมูลที่ aligned ได้เร็วกว่า
x86 ต้องการ 4-byte alignment สำหรับ 32-bit values
x86-64 ต้องการ 8-byte alignment สำหรับ 64-bit values
SSE ต้องการ 16-byte alignment

```
การ Align ที่ผิด vs ถูกต้อง:

ผิด (unaligned):
Byte: 0  1  2  3  4  5  6  7
      [  char c   ][   int x   ]
       ↑ c starts at 0 (OK)
                  ↑ x starts at 3 (WRONG! int ควรอยู่ที่ multiple of 4)

ถูกต้อง (aligned):
Byte: 0  1  2  3  4  5  6  7
      [char c][pad][   int x   ]
       ↑ c at 0     ↑ x at 4 (OK! multiple of 4)
```

### การ Align Stack ใน Assembly

```nasm
; ไฟล์: stack_alignment.asm
; การ align stack สำหรับ SIMD operations
; วิธีคอมไพล์: nasm -f elf32 stack_alignment.asm -o stack_alignment.o
;              gcc -m32 stack_alignment.o -o stack_alignment -no-pie -msse2

section .data
    fmt_addr    db "Address: 0x%x, Aligned: %s", 10, 0
    str_yes     db "YES", 0
    str_no      db "NO", 0

section .bss
    ; ตัวแปรใน BSS segment (aligned เองโดยอัตโนมัติ)
    aligned_var resd 4      ; 4 dwords = 16 bytes

section .text
    global main
    extern printf

; ฟังก์ชัน: void check_alignment(void)
; ตรวจสอบ alignment ของตัวแปรบน stack
check_alignment:
    push    ebp
    mov     ebp, esp
    
    ; จัดสรร local variables และ align
    ; ต้องการ alignment 16 bytes สำหรับ SSE
    sub     esp, 32         ; จอง 32 bytes
    
    ; Align ESP ให้เป็น 16-byte boundary
    and     esp, 0xFFFFFFF0 ; clear 4 bits ล่างสุด
    
    ; ตอนนี้ ESP aligned กับ 16 bytes
    ; [ESP + 0..15]  = สำหรับ SSE data
    ; [ESP + 16..31] = ตัวแปรอื่นๆ
    
    ; ตรวจสอบว่า aligned หรือเปล่า
    mov     eax, esp
    and     eax, 0x0F       ; ดู 4 bits ล่าง
    
    ; แสดงผล
    push    str_no
    test    eax, eax
    jnz     .not_aligned
    push    str_yes
    jmp     .show
.not_aligned:
    push    str_no
.show:
    push    esp
    push    fmt_addr
    call    printf
    add     esp, 12
    
    mov     esp, ebp
    pop     ebp
    ret

; ฟังก์ชัน: void demo_struct_alignment(void)
; แสดง struct layout บน stack
demo_struct_alignment:
    push    ebp
    mov     ebp, esp
    
    ; struct ใน C:
    ; struct {
    ;     char  a;        // 1 byte, offset 0
    ;     // padding 3 bytes
    ;     int   b;        // 4 bytes, offset 4
    ;     char  c;        // 1 byte, offset 8
    ;     short d;        // 2 bytes, offset 10
    ;     // padding 2 bytes?? 
    ;     int   e;        // 4 bytes, offset 12
    ; };  total = 16 bytes
    
    ; บน stack เราต้องจัดสรรเองให้ถูก
    sub     esp, 16
    
    ; ตำแหน่ง fields:
    ; a = [ebp - 16]  (1 byte, char)
    ; b = [ebp - 12]  (4 bytes, int, aligned to 4)
    ; c = [ebp - 8]   (1 byte, char)
    ; d = [ebp - 6]   (2 bytes, short, aligned to 2)
    ; e = [ebp - 4]   (4 bytes, int... wait - misaligned)
    
    ; ต้องระวัง: จัดสรรเอง ต้องคำนวณ alignment เอง
    ; ดีกว่าใช้ struct ใน C แล้วให้ compiler จัดการ
    
    mov     byte  [ebp - 16], 'A'   ; a = 'A'
    mov     dword [ebp - 12], 42    ; b = 42
    mov     byte  [ebp - 8], 'B'    ; c = 'B'
    mov     word  [ebp - 6], 100    ; d = 100
    ; e จะต้องอยู่ที่ [ebp-4] แต่ต้องการ 4 bytes...
    ; ถ้าต้องการ aligned ต้องจัดสรรเพิ่ม
    
    mov     esp, ebp
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    
    call    check_alignment
    call    demo_struct_alignment
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

### Stack Alignment สำหรับ SSE/AVX

```nasm
; ตัวอย่าง: 16-byte aligned stack สำหรับ SSE
sse_function:
    push    ebp
    mov     ebp, esp
    
    ; ต้องการ 16-byte aligned memory สำหรับ movaps
    sub     esp, 48         ; จอง 48 bytes
    and     esp, 0xFFFFFFF0 ; align to 16 bytes
    
    ; ตอนนี้ ESP aligned กับ 16 bytes
    ; สามารถใช้ MOVAPS (aligned) แทน MOVUPS (unaligned) ได้
    
    ; โหลด 4 floats จาก aligned memory
    ; (ตัวอย่างแบบ conceptual)
    ; movaps  xmm0, [esp]
    ; movaps  xmm1, [esp + 16]
    ; addps   xmm0, xmm1
    ; movaps  [esp + 32], xmm0
    
    mov     esp, ebp
    pop     ebp
    ret
```

---

## alloca() เทียบเท่าใน Assembly

### alloca() คืออะไร

`alloca()` เป็น C function ที่จัดสรรหน่วยความจำบน stack แบบ dynamic
(ขนาดไม่ทราบตอน compile time)

ใน Assembly เราทำได้โดยลด ESP ลงเท่ากับขนาดที่ต้องการ

```nasm
; C code เทียบเท่า:
; void* alloca_equivalent(size_t size) {
;     // ลด ESP ลง size bytes
;     // คืน pointer ไปที่ allocated memory
; }

; ใน Assembly:
alloca_demo:
    push    ebp
    mov     ebp, esp
    
    ; รับ size จาก parameter
    mov     eax, [ebp + 8]      ; size
    
    ; Align ขึ้น 4 bytes
    add     eax, 3              ; size + 3
    and     eax, 0xFFFFFFFC     ; ตัด 2 bits ล่างสุด (round up to 4)
    
    ; จัดสรรบน stack
    sub     esp, eax            ; ESP -= aligned_size
    
    ; EAX ตอนนี้เป็น pointer ไปยัง allocated memory
    mov     eax, esp
    
    ; ใช้ memory ที่ [eax]...
    
    ; NOTE: ไม่ต้อง free เอง! เมื่อ function return
    ; ESP กลับไปที่ EBP ทำให้ memory ถูก "free" อัตโนมัติ
    mov     esp, ebp
    pop     ebp
    ret
```

### ตัวอย่างสมบูรณ์: Dynamic Stack Allocation

```nasm
; ไฟล์: alloca_demo.asm
; การจัดสรร memory บน stack แบบ dynamic (alloca equivalent)
; วิธีคอมไพล์: nasm -f elf32 alloca_demo.asm -o alloca_demo.o
;              gcc -m32 alloca_demo.o -o alloca_demo -no-pie

section .data
    fmt_alloc   db "จัดสรร %d bytes บน stack, ptr = 0x%x", 10, 0
    fmt_sum     db "ผลรวมของ %d ตัวเลข = %d", 10, 0
    fmt_arr     db "arr[%d] = %d", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: int sum_dynamic(int n)
; รับ n จาก user และสร้าง array บน stack แบบ dynamic
; เติมค่า 1..n แล้วหาผลรวม
sum_dynamic:
    push    ebp
    mov     ebp, esp
    push    ebx                 ; เก็บ callee-saved register
    push    esi
    push    edi
    
    ; รับ n
    mov     ecx, [ebp + 8]      ; n = parameter
    mov     ebx, ecx            ; เก็บ n ไว้
    
    ; คำนวณขนาด: n * 4 bytes (สำหรับ int array)
    mov     eax, ecx
    shl     eax, 2              ; eax = n * 4
    
    ; Align ขึ้น 4 bytes
    add     eax, 3
    and     eax, 0xFFFFFFFC
    
    ; จัดสรร dynamic array บน stack
    sub     esp, eax            ; ESP -= size
    mov     edi, esp            ; EDI = pointer to array
    
    ; แสดงข้อมูล allocation
    push    edi                 ; ptr
    push    ebx                 ; n
    push    fmt_alloc
    call    printf
    add     esp, 12
    
    ; เติมค่า 1..n ลงใน array
    xor     esi, esi            ; i = 0
.fill_loop:
    cmp     esi, ebx
    jge     .fill_done
    
    lea     eax, [esi + 1]      ; eax = i + 1
    mov     [edi + esi*4], eax  ; arr[i] = i + 1
    inc     esi
    jmp     .fill_loop
.fill_done:
    
    ; แสดงค่าในArray
    xor     esi, esi
.show_loop:
    cmp     esi, ebx
    jge     .show_done
    
    mov     eax, [edi + esi*4]  ; arr[i]
    push    eax                 ; value
    push    esi                 ; index
    push    fmt_arr
    call    printf
    add     esp, 12
    
    inc     esi
    jmp     .show_loop
.show_done:
    
    ; คำนวณผลรวม
    xor     eax, eax            ; sum = 0
    xor     esi, esi            ; i = 0
.sum_loop:
    cmp     esi, ebx
    jge     .sum_done
    
    add     eax, [edi + esi*4]  ; sum += arr[i]
    inc     esi
    jmp     .sum_loop
.sum_done:
    
    ; แสดงผลรวม
    push    eax
    push    ebx
    push    fmt_sum
    call    printf
    add     esp, 12
    
    ; ผลลัพธ์ = sum
    ; (EAX ถูก clobber โดย printf ต้องคำนวณใหม่)
    xor     eax, eax
    xor     esi, esi
.calc_return:
    cmp     esi, ebx
    jge     .calc_done
    add     eax, [edi + esi*4]
    inc     esi
    jmp     .calc_return
.calc_done:
    
    ; Epilogue (EBP จะ restore ESP ให้ก่อน pop array)
    pop     edi
    pop     esi
    pop     ebx
    mov     esp, ebp            ; คืน ESP ซึ่ง "free" dynamic array อัตโนมัติ
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    
    ; เรียก sum_dynamic(5) - สร้าง array [1,2,3,4,5] บน stack
    push    5
    call    sum_dynamic
    add     esp, 4
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 alloca_demo.asm -o alloca_demo.o
gcc -m32 alloca_demo.o -o alloca_demo -no-pie
./alloca_demo
```

---

## VLA (Variable Length Arrays)

### VLA ใน C99

ใน C99 สามารถสร้าง array ที่ขนาดกำหนดตอน runtime ได้:

```c
void process(int n) {
    int arr[n];  // VLA - ขนาดกำหนดตอน runtime
    // ...
}
```

Compiler แปล VLA เป็น code ที่ลด ESP dynamically คล้าย alloca()

### VLA ใน Assembly

```nasm
; ไฟล์: vla_demo.asm
; การสร้าง Variable Length Arrays บน Stack
; เลียนแบบ C99 VLA
; วิธีคอมไพล์: nasm -f elf32 vla_demo.asm -o vla_demo.o
;              gcc -m32 vla_demo.o -o vla_demo -no-pie

section .data
    fmt_vla     db "VLA[%d] ขนาด %d ints, addr=0x%x", 10, 0
    fmt_val     db "  [%d] = %d", 10, 0
    fmt_total   db "ผลรวม = %d", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: int process_vla(int n, int fill_value)
; สร้าง VLA ขนาด n บน stack เติมด้วย fill_value
; คืนผลรวม
process_vla:
    push    ebp
    mov     ebp, esp
    push    ebx
    push    esi
    push    edi
    
    ; บันทึก stack pointer ไว้สำหรับ deallocate VLA
    ; (สำคัญมากสำหรับ VLA!)
    mov     ebx, esp            ; บันทึก ESP ก่อนสร้าง VLA
    
    ; รับ parameters
    mov     ecx, [ebp + 8]      ; n
    mov     edx, [ebp + 12]     ; fill_value
    
    ; คำนวณขนาด VLA (n * 4 bytes)
    mov     eax, ecx
    shl     eax, 2              ; n * 4
    add     eax, 15             ; round up ไป multiple of 16
    and     eax, 0xFFFFFFF0
    
    ; สร้าง VLA บน stack
    sub     esp, eax
    mov     edi, esp            ; edi = pointer ไปยัง VLA
    
    ; แสดงข้อมูล VLA
    push    edi
    push    ecx
    push    ecx
    push    fmt_vla
    call    printf
    add     esp, 16
    
    ; เติมค่าทุก element
    xor     esi, esi
.fill:
    cmp     esi, ecx
    jge     .filled
    mov     [edi + esi*4], edx
    inc     esi
    jmp     .fill
.filled:
    
    ; แสดงค่า (แค่ 5 ตัวแรก)
    xor     esi, esi
    mov     eax, ecx
    cmp     eax, 5
    jle     .show_all
    mov     eax, 5
.show_all:
    mov     ecx, eax
.show:
    cmp     esi, ecx
    jge     .shown
    mov     eax, [edi + esi*4]
    push    eax
    push    esi
    push    fmt_val
    call    printf
    add     esp, 12
    inc     esi
    jmp     .show
.shown:
    
    ; คำนวณผลรวม
    mov     ecx, [ebp + 8]      ; n อีกครั้ง
    xor     eax, eax
    xor     esi, esi
.sum:
    cmp     esi, ecx
    jge     .summed
    add     eax, [edi + esi*4]
    inc     esi
    jmp     .sum
.summed:
    
    push    eax
    push    fmt_total
    call    printf
    add     esp, 8
    
    ; "Deallocate" VLA โดยคืน ESP
    mov     esp, ebx            ; คืน ESP ที่บันทึกไว้
    
    ; Return sum (คำนวณอีกครั้ง)
    mov     ecx, [ebp + 8]
    xor     eax, eax
    xor     esi, esi
.calc:
    cmp     esi, ecx
    jge     .done
    add     eax, [edi + esi*4]
    inc     esi
    jmp     .calc
.done:
    
    pop     edi
    pop     esi
    pop     ebx
    mov     esp, ebp
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    
    ; ทดสอบ VLA ขนาด 8 เติมด้วย 5
    push    5                   ; fill_value
    push    8                   ; n
    call    process_vla
    add     esp, 8
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 vla_demo.asm -o vla_demo.o
gcc -m32 vla_demo.o -o vla_demo -no-pie
./vla_demo
```

---

## Frame Chaining

Frame Chaining คือการที่ saved EBP values สร้างเป็น linked list
ที่เชื่อมต่อ stack frames ทั้งหมดเข้าหากัน

```
Frame Chain Diagram:

EBP → [Saved EBP] → [Saved EBP] → [Saved EBP] → NULL
       ↑               ↑               ↑
   main's frame    caller's frame  grandcaller's frame
```

### การ Walk Frame Chain

```nasm
; ไฟล์: frame_chain.asm
; การ walk frame chain เพื่อดู call stack
; วิธีคอมไพล์: nasm -f elf32 frame_chain.asm -o frame_chain.o
;              gcc -m32 frame_chain.o -o frame_chain -no-pie

section .data
    fmt_frame   db "Frame #%d: EBP=0x%08x, RetAddr=0x%08x", 10, 0
    fmt_chain   db "=== Frame Chain Walk ===", 10, 0
    fmt_bottom  db "=== หยุดที่ frame #%d (EBP=0x%08x) ===", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: void walk_frames(void)
; Walk frame chain และแสดงแต่ละ frame
walk_frames:
    push    ebp
    mov     ebp, esp
    push    ebx
    push    esi
    push    edi
    
    ; แสดง header
    push    fmt_chain
    call    printf
    add     esp, 4
    
    ; เริ่ม walk จาก EBP ปัจจุบัน
    mov     ebx, ebp            ; current frame pointer
    xor     esi, esi            ; frame counter
    
.walk:
    ; ตรวจสอบว่า EBP valid หรือเปล่า
    ; EBP ต้องไม่เป็น 0 และต้องอยู่ในขอบเขต stack
    test    ebx, ebx
    jz      .done
    
    cmp     ebx, 0x80000000     ; ตรวจสอบ upper bound (rough check)
    jg      .done
    
    ; ดึง return address ([EBP+4])
    mov     edi, [ebx + 4]      ; return address
    
    ; แสดงข้อมูล frame
    push    edi                 ; return address
    push    ebx                 ; EBP value
    push    esi                 ; frame number
    push    fmt_frame
    call    printf
    add     esp, 16
    
    ; ไปยัง frame ก่อนหน้า
    mov     eax, [ebx]          ; load saved EBP
    
    ; ตรวจว่า saved EBP > current EBP (stack grows down)
    cmp     eax, ebx
    jle     .done               ; ถ้าไม่ใช่ ให้หยุด
    
    mov     ebx, eax            ; เดินไปยัง frame ก่อนหน้า
    inc     esi
    
    cmp     esi, 20             ; จำกัดไม่เกิน 20 frames
    jl      .walk
    
.done:
    push    ebx
    push    esi
    push    fmt_bottom
    call    printf
    add     esp, 12
    
    pop     edi
    pop     esi
    pop     ebx
    mov     esp, ebp
    pop     ebp
    ret

; ฟังก์ชัน C level: deep_function → middle_function → main → walk
deep_function:
    push    ebp
    mov     ebp, esp
    
    call    walk_frames
    
    mov     esp, ebp
    pop     ebp
    ret

middle_function:
    push    ebp
    mov     ebp, esp
    
    call    deep_function
    
    mov     esp, ebp
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    
    call    middle_function
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 frame_chain.asm -o frame_chain.o
gcc -m32 frame_chain.o -o frame_chain -no-pie
./frame_chain
```

---

## Unwind Information

### ทำไมต้องมี Unwind Information

เมื่อเกิด exception หรือเมื่อ debugger ต้องการ backtrace
runtime ต้องสามารถ "unwind" stack ได้
Unwind information บอกว่าแต่ละ function ทำอะไรกับ stack

### DWARF Unwind Information

```nasm
; ตัวอย่างการเพิ่ม DWARF CFI directives สำหรับ unwind information
; ใช้ใน Linux x86 ELF binaries

section .text
    global my_function_with_cfi

my_function_with_cfi:
    ; CFI (Call Frame Information) directives
    .cfi_startproc                  ; เริ่ม CFI สำหรับฟังก์ชันนี้
    
    push    ebp
    .cfi_def_cfa_offset 8           ; CFA = ESP + 8 (หลัง push ebp)
    .cfi_offset ebp, -8             ; EBP อยู่ที่ CFA - 8
    
    mov     ebp, esp
    .cfi_def_cfa_register ebp       ; ตอนนี้ CFA = EBP + 4
    
    sub     esp, 16
    push    ebx
    .cfi_offset ebx, -12            ; EBX อยู่ที่ CFA - 12
    
    ; ... code ...
    
    pop     ebx
    .cfi_restore ebx                ; EBX restored
    
    mov     esp, ebp
    pop     ebp
    .cfi_def_cfa esp, 4             ; CFA = ESP + 4 อีกครั้ง
    .cfi_restore ebp                ; EBP restored
    
    ret
    .cfi_endproc                    ; สิ้นสุด CFI
```

### ตัวอย่างสมบูรณ์พร้อม CFI

```nasm
; ไฟล์: unwind_info.asm
; การเพิ่ม CFI directives สำหรับ proper unwind information
; วิธีคอมไพล์: nasm -f elf32 -g -F dwarf unwind_info.asm -o unwind_info.o
;              gcc -m32 -g unwind_info.o -o unwind_info -no-pie

; หมายเหตุ: NASM ไม่รองรับ .cfi_* directives โดยตรง
; สำหรับ CFI ใช้ GAS (GNU Assembler) หรือ compile C code ด้วย -g

; ตัวอย่างนี้แสดง code ที่เทียบเท่ากับ:
; int add_with_cfi(int a, int b) {
;     int local = a + b;
;     return local;
; }

section .text
    global add_with_cfi

add_with_cfi:
    push    ebp
    mov     ebp, esp
    sub     esp, 4          ; local variable
    
    mov     eax, [ebp + 8]
    add     eax, [ebp + 12]
    mov     [ebp - 4], eax  ; local = a + b
    
    mov     eax, [ebp - 4]  ; return local
    
    mov     esp, ebp
    pop     ebp
    ret
```

### การสร้าง Debug Info ที่ดี

```bash
# คอมไพล์พร้อม debug symbols เต็มรูปแบบ
nasm -f elf32 -g -F dwarf program.asm -o program.o

# Link
gcc -m32 -g program.o -o program -no-pie

# ดู DWARF info
readelf --debug-dump=frames program   # ดู frame info
readelf --debug-dump=info program     # ดู debug info ทั้งหมด

# ดู stack unwinding
objdump --dwarf=frames program

# ใช้ addr2line เพื่อแปลง address เป็น line number
addr2line -e program 0x08048400
```

---

## ตัวอย่างโปรแกรมขนาดใหญ่: Function Call Chain

```nasm
; ไฟล์: complete_example.asm
; ตัวอย่างสมบูรณ์ที่รวมทุกแนวคิด
; - Stack frames ที่ซ้อนกัน
; - Parameters และ local variables
; - Dynamic allocation
; - Frame chain
; วิธีคอมไพล์: nasm -f elf32 complete_example.asm -o complete_example.o
;              gcc -m32 complete_example.o -o complete_example -no-pie

section .data
    fmt_fibonacci   db "Fibonacci(%d) = %d", 10, 0
    fmt_factorial   db "Factorial(%d) = %d", 10, 0
    fmt_power       db "Power(%d, %d) = %d", 10, 0
    fmt_sep         db "----------------------------", 10, 0
    fmt_header      db "=== Complete Stack Frame Demo ===", 10, 0
    fmt_stack_use   db "Stack used: ~%d bytes", 10, 0

section .text
    global main
    extern printf

; ====================================================
; ฟังก์ชัน: int fibonacci(int n)
; คำนวณ Fibonacci แบบ recursive
; Stack Frame ซ้อนหลายชั้น
; ====================================================
fibonacci:
    push    ebp
    mov     ebp, esp
    sub     esp, 8          ; [ebp-4] = temp, [ebp-8] = fib(n-2)
    push    ebx
    push    esi
    
    mov     eax, [ebp + 8]  ; n
    
    ; base cases
    cmp     eax, 0
    je      .fib_zero
    cmp     eax, 1
    je      .fib_one
    
    ; recursive case: fib(n) = fib(n-1) + fib(n-2)
    
    ; คำนวณ fib(n-1)
    dec     eax
    push    eax
    call    fibonacci
    add     esp, 4
    mov     [ebp - 4], eax  ; บันทึก fib(n-1)
    
    ; คำนวณ fib(n-2)
    mov     eax, [ebp + 8]
    sub     eax, 2
    push    eax
    call    fibonacci
    add     esp, 4
    mov     [ebp - 8], eax  ; บันทึก fib(n-2)
    
    ; รวม fib(n-1) + fib(n-2)
    mov     eax, [ebp - 4]
    add     eax, [ebp - 8]
    jmp     .fib_done
    
.fib_zero:
    xor     eax, eax
    jmp     .fib_done
.fib_one:
    mov     eax, 1
    
.fib_done:
    pop     esi
    pop     ebx
    mov     esp, ebp
    pop     ebp
    ret

; ====================================================
; ฟังก์ชัน: int factorial(int n)
; คำนวณ factorial แบบ iterative ด้วย local variables
; ====================================================
factorial:
    push    ebp
    mov     ebp, esp
    sub     esp, 8          ; [ebp-4] = result, [ebp-8] = i
    
    ; factorial = 1
    mov     dword [ebp - 4], 1  ; result = 1
    mov     dword [ebp - 8], 1  ; i = 1
    
    ; loop: result *= i, i++
.fact_loop:
    mov     eax, [ebp - 8]  ; i
    cmp     eax, [ebp + 8]  ; i > n?
    jg      .fact_done
    
    imul    eax, [ebp - 4]  ; eax = i * result
    mov     [ebp - 4], eax  ; result = eax
    
    inc     dword [ebp - 8] ; i++
    jmp     .fact_loop
    
.fact_done:
    mov     eax, [ebp - 4]  ; return result
    mov     esp, ebp
    pop     ebp
    ret

; ====================================================
; ฟังก์ชัน: int power(int base, int exp)
; คำนวณ base^exp แบบ iterative
; ====================================================
power:
    push    ebp
    mov     ebp, esp
    sub     esp, 4          ; [ebp-4] = result
    
    mov     dword [ebp - 4], 1  ; result = 1
    mov     ecx, [ebp + 12]     ; exp
    
.power_loop:
    test    ecx, ecx
    jz      .power_done
    
    mov     eax, [ebp - 4]
    imul    eax, [ebp + 8]      ; result *= base
    mov     [ebp - 4], eax
    
    dec     ecx
    jmp     .power_loop
    
.power_done:
    mov     eax, [ebp - 4]
    mov     esp, ebp
    pop     ebp
    ret

; ====================================================
; ฟังก์ชัน: void run_demonstrations(int limit)
; รัน demo ทั้งหมด
; ====================================================
run_demonstrations:
    push    ebp
    mov     ebp, esp
    sub     esp, 12         ; local vars: [ebp-4]=i, [ebp-8]=result, [ebp-12]=limit
    push    ebx
    
    mov     eax, [ebp + 8]  ; limit
    mov     [ebp - 12], eax
    
    ; แสดง Fibonacci series
    xor     ebx, ebx        ; i = 0
.fib_demo:
    cmp     ebx, [ebp - 12]
    jge     .fib_demo_done
    
    push    ebx
    call    fibonacci
    add     esp, 4
    
    push    eax             ; result
    push    ebx             ; n
    push    fmt_fibonacci
    call    printf
    add     esp, 12
    
    inc     ebx
    jmp     .fib_demo
.fib_demo_done:
    
    ; separator
    push    fmt_sep
    call    printf
    add     esp, 4
    
    ; แสดง Factorial series
    mov     ebx, 1          ; i = 1
.fact_demo:
    cmp     ebx, [ebp - 12]
    jg      .fact_demo_done
    
    push    ebx
    call    factorial
    add     esp, 4
    
    push    eax
    push    ebx
    push    fmt_factorial
    call    printf
    add     esp, 12
    
    inc     ebx
    jmp     .fact_demo
.fact_demo_done:
    
    ; separator
    push    fmt_sep
    call    printf
    add     esp, 4
    
    ; แสดง Power series: 2^i
    xor     ebx, ebx        ; i = 0
.power_demo:
    cmp     ebx, [ebp - 12]
    jg      .power_demo_done
    
    push    ebx             ; exp
    push    2               ; base
    call    power
    add     esp, 8
    
    push    eax             ; result
    push    ebx             ; exp
    push    2               ; base
    push    fmt_power
    call    printf
    add     esp, 16
    
    inc     ebx
    jmp     .power_demo
.power_demo_done:
    
    pop     ebx
    mov     esp, ebp
    pop     ebp
    ret

main:
    push    ebp
    mov     ebp, esp
    sub     esp, 4
    
    ; แสดง header
    push    fmt_header
    call    printf
    add     esp, 4
    
    push    fmt_sep
    call    printf
    add     esp, 4
    
    ; รัน demo ด้วย limit = 8
    push    8
    call    run_demonstrations
    add     esp, 4
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 complete_example.asm -o complete_example.o
gcc -m32 complete_example.o -o complete_example -no-pie
./complete_example
```

---

## Stack Frame ใน x86-64 (64-bit)

ใน 64-bit มีความแตกต่างสำคัญ:

```nasm
; x86-64 Calling Convention (System V AMD64 ABI)
; Parameters ถูกส่งผ่าน registers ก่อน:
;   rdi = 1st param
;   rsi = 2nd param
;   rdx = 3rd param
;   rcx = 4th param
;   r8  = 5th param
;   r9  = 6th param
;   stack = parameters ที่เกิน 6 ตัว

; Stack ต้อง aligned ที่ 16 bytes ก่อน CALL

; ตัวอย่าง 64-bit function
add_64:
    push    rbp             ; บันทึก RBP (8 bytes)
    mov     rbp, rsp
    sub     rsp, 16         ; จอง local vars
    
    ; Parameters อยู่ใน registers (ไม่ต้องอ่านจาก stack)
    ; rdi = a, rsi = b
    
    mov     [rbp - 4], edi  ; บันทึก a เป็น local var
    mov     [rbp - 8], esi  ; บันทึก b เป็น local var
    
    mov     eax, [rbp - 4]
    add     eax, [rbp - 8]
    ; return value ใน EAX/RAX
    
    leave                   ; mov rsp,rbp; pop rbp
    ret
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Stack Frame

สร้างฟังก์ชัน `int max3(int a, int b, int c)` ที่คืนค่ามากที่สุดใน 3 ค่า
โดยใช้ local variables เก็บค่า intermediate ที่เหมาะสม

```nasm
; โครงสร้างที่ให้ทำ:
; ไฟล์: exercise1.asm
; วิธีคอมไพล์: nasm -f elf32 exercise1.asm -o exercise1.o
;              gcc -m32 exercise1.o -o exercise1 -no-pie

section .data
    fmt_result  db "max(%d, %d, %d) = %d", 10, 0

section .text
    global main
    extern printf

; TODO: เขียนฟังก์ชัน max3 ที่นี่
; Hints:
; - Parameters: a=[EBP+8], b=[EBP+12], c=[EBP+16]
; - Local vars: สำหรับเก็บค่า current max
; - ใช้ CMP และ conditional JMP

max3:
    ; เติม code ที่นี่
    ret

main:
    push    ebp
    mov     ebp, esp
    
    ; ทดสอบ max3(5, 9, 3)
    push    3
    push    9
    push    5
    call    max3
    add     esp, 12
    
    push    eax
    push    3
    push    9
    push    5
    push    fmt_result
    call    printf
    add     esp, 20
    
    ; ทดสอบ max3(100, 50, 75)
    push    75
    push    50
    push    100
    call    max3
    add     esp, 12
    
    push    eax
    push    75
    push    50
    push    100
    push    fmt_result
    call    printf
    add     esp, 20
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**เฉลย:**
```nasm
max3:
    push    ebp
    mov     ebp, esp
    sub     esp, 4          ; [ebp-4] = current_max
    
    mov     eax, [ebp + 8]      ; eax = a
    mov     [ebp - 4], eax      ; current_max = a
    
    mov     ebx, [ebp + 12]     ; ebx = b
    cmp     ebx, [ebp - 4]
    jle     .check_c
    mov     [ebp - 4], ebx      ; current_max = b
    
.check_c:
    mov     ecx, [ebp + 16]     ; ecx = c
    cmp     ecx, [ebp - 4]
    jle     .done
    mov     [ebp - 4], ecx      ; current_max = c
    
.done:
    mov     eax, [ebp - 4]      ; return current_max
    mov     esp, ebp
    pop     ebp
    ret
```

### แบบฝึกหัดที่ 2: Nested Function Calls

เขียนฟังก์ชัน `int sum_of_squares(int n)` ที่คำนวณ 1² + 2² + ... + n²
โดยใช้ฟังก์ชัน helper `int square(int x)` ที่แยกต่างหาก

```nasm
; ไฟล์: exercise2.asm
; วิธีคอมไพล์: nasm -f elf32 exercise2.asm -o exercise2.o
;              gcc -m32 exercise2.o -o exercise2 -no-pie

section .data
    fmt_result  db "sum_of_squares(%d) = %d", 10, 0

section .text
    global main
    extern printf

; TODO: เขียน square(int x) ที่นี่

; TODO: เขียน sum_of_squares(int n) ที่นี่

main:
    push    ebp
    mov     ebp, esp
    
    ; ทดสอบ: sum_of_squares(5) = 1+4+9+16+25 = 55
    push    5
    call    sum_of_squares
    add     esp, 4
    
    push    eax
    push    5
    push    fmt_result
    call    printf
    add     esp, 12
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

**เฉลย:**
```nasm
square:
    push    ebp
    mov     ebp, esp
    
    mov     eax, [ebp + 8]
    imul    eax, eax        ; eax = x * x
    
    mov     esp, ebp
    pop     ebp
    ret

sum_of_squares:
    push    ebp
    mov     ebp, esp
    sub     esp, 8          ; [ebp-4]=sum, [ebp-8]=i
    push    ebx
    
    mov     dword [ebp - 4], 0  ; sum = 0
    mov     dword [ebp - 8], 1  ; i = 1
    
.loop:
    mov     eax, [ebp - 8]      ; i
    cmp     eax, [ebp + 8]      ; i > n?
    jg      .done
    
    ; เรียก square(i)
    push    dword [ebp - 8]
    call    square
    add     esp, 4
    
    add     [ebp - 4], eax      ; sum += square(i)
    inc     dword [ebp - 8]     ; i++
    jmp     .loop
    
.done:
    mov     eax, [ebp - 4]
    pop     ebx
    mov     esp, ebp
    pop     ebp
    ret
```

### แบบฝึกหัดที่ 3: Dynamic Array

เขียนฟังก์ชัน `int find_max_in_array(int n)` ที่:
1. สร้าง array ขนาด n บน stack (alloca equivalent)
2. เติมค่า random (ใช้ pattern แทน: arr[i] = (i*7 + 3) % 100)
3. หาค่าสูงสุดและคืนกลับ

```nasm
; ไฟล์: exercise3.asm
; วิธีคอมไพล์: nasm -f elf32 exercise3.asm -o exercise3.o
;              gcc -m32 exercise3.o -o exercise3 -no-pie

section .data
    fmt_result  db "max ใน array ขนาด %d = %d", 10, 0

section .text
    global main
    extern printf

; TODO: เขียนฟังก์ชัน find_max_in_array ที่นี่

main:
    push    ebp
    mov     ebp, esp
    
    push    10
    call    find_max_in_array
    add     esp, 4
    
    push    eax
    push    10
    push    fmt_result
    call    printf
    add     esp, 12
    
    xor     eax, eax
    mov     esp, ebp
    pop     ebp
    ret
```

### แบบฝึกหัดที่ 4: Stack Frame Inspector

เขียนฟังก์ชัน `void inspect_frame(void)` ที่:
1. แสดงค่า EBP ปัจจุบัน
2. แสดง return address ([EBP+4])
3. Walk back 3 frames และแสดงข้อมูลแต่ละ frame

ใช้ความรู้จาก Frame Chaining section

### แบบฝึกหัดที่ 5: มิกซ์ Parameter Types

เขียนฟังก์ชัน C:
```c
double weighted_average(int n, int* weights, int* values);
```

ใน Assembly (32-bit) โดย:
- n อยู่ที่ [EBP+8]
- weights pointer อยู่ที่ [EBP+12]
- values pointer อยู่ที่ [EBP+16]
- ใช้ FPU stack สำหรับ floating point

---

## สรุปสิ่งสำคัญ

### Stack Frame Offsets Quick Reference

```
x86 32-bit Standard Frame Layout:
┌─────────────────────────────┐
│ [EBP + 4*n+4]  nthparameter │ ← higher address
│ ...                         │
│ [EBP + 16]     3rd param    │
│ [EBP + 12]     2nd param    │
│ [EBP + 8]      1st param    │
│ [EBP + 4]      return addr  │
│ [EBP + 0]      saved EBP    │ ← EBP ชี้ที่นี่
│ [EBP - 4]      local var 1  │
│ [EBP - 8]      local var 2  │
│ [EBP - 4*n]    nth local    │ ← ESP ชี้ที่ตำแหน่งต่ำสุด
└─────────────────────────────┘ ← lower address
```

### Calling Convention ที่ต้องจำ (cdecl)

| สิ่ง | รายละเอียด |
|------|------------|
| Parameter order | Push right-to-left |
| Return value | EAX (int/pointer), EDX:EAX (64-bit) |
| Stack cleanup | Caller (add esp, N) |
| Callee-saved | EBX, ESI, EDI, EBP |
| Caller-saved | EAX, ECX, EDX |

### ขั้นตอน Prologue/Epilogue

```
Prologue:               Epilogue:
  push  ebp               mov  esp, ebp
  mov   ebp, esp          pop  ebp
  sub   esp, N            ret
  push  ebx
  push  esi
  push  edi
```

### Common Mistakes และวิธีหลีกเลี่ยง

1. **ลืม align stack** → ใช้ `and esp, 0xFFFFFFF0` ก่อนเรียก SSE functions
2. **นับ bytes ผิด** → วาด diagram stack frame ก่อน code
3. **ลืม clean stack หลัง CALL** → นับ bytes ที่ push และ `add esp, N`
4. **ทำลาย callee-saved registers** → push/pop EBX, ESI, EDI ถ้าใช้
5. **VLA ไม่ restore ESP** → บันทึก ESP ไว้ก่อนสร้าง VLA แล้ว restore

---

## คำสั่งรวม

```bash
# คอมไพล์พื้นฐาน
nasm -f elf32 file.asm -o file.o
gcc -m32 file.o -o file -no-pie

# คอมไพล์พร้อม debug
nasm -f elf32 -g -F dwarf file.asm -o file.o
gcc -m32 -g file.o -o file -no-pie

# ดู assembly output ของ C code (เพื่อเรียนรู้)
gcc -m32 -O0 -S -fno-omit-frame-pointer file.c -o file.s

# Debug ด้วย GDB
gdb ./file
(gdb) break function_name
(gdb) run
(gdb) info frame
(gdb) backtrace
(gdb) x/16xw $esp

# ดู binary
objdump -d -M intel file      # disassemble
readelf -a file               # ดู ELF headers
nm file                       # ดู symbols
```

---

## Part ถัดไป

**Part 023:** Calling Conventions เปรียบเทียบ (cdecl, stdcall, fastcall, thiscall)
- cdecl vs stdcall: ใครทำความสะอาด stack
- fastcall: ส่ง parameters ผ่าน registers
- thiscall: สำหรับ C++ member functions
- การเรียกฟังก์ชัน Windows API
- System V AMD64 ABI (64-bit Linux)
- Windows x64 calling convention

---

*หลักสูตร Assembly Programming - Part 022*
*Stack Frames และ Local Variables*
