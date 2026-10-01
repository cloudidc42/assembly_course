# Part 012: Addressing Modes ใน x86

## Prerequisites and Learning Objectives

### สิ่งที่ต้องรู้ก่อน (Prerequisites)
- ผ่าน Part 001-011 แล้ว หรือมีความรู้พื้นฐาน Assembly
- เข้าใจ Registers และการใช้งาน (RAX, RBX, RCX, RDX, RSI, RDI, RSP, RBP)
- เข้าใจ Memory layout เบื้องต้น
- ติดตั้ง NASM และ GCC แล้ว

### วัตถุประสงค์การเรียนรู้ (Learning Objectives)
หลังจากเรียน Part นี้แล้ว คุณจะสามารถ:
1. อธิบาย Addressing Modes ทุกแบบใน x86/x64 ได้
2. เลือกใช้ Addressing Mode ที่เหมาะสมในแต่ละสถานการณ์
3. เข้าใจความแตกต่างระหว่าง 32-bit และ 64-bit Addressing
4. ใช้ LEA instruction สำหรับการคำนวณ Address ที่ซับซ้อน
5. เข้าใจ SIB byte และ RIP-relative addressing
6. หลีกเลี่ยง Common Mistakes ที่เกิดขึ้นบ่อย

---

## Theory: ทฤษฎี Addressing Modes

### 1. ความหมายของ Addressing Mode

**Addressing Mode** คือวิธีที่ processor ค้นหา operand (ข้อมูล) ที่จะนำมาใช้ในการประมวลผล
ในสถาปัตยกรรม x86/x64 มี Addressing Modes หลายแบบ แต่ละแบบมีจุดประสงค์และประสิทธิภาพที่แตกต่างกัน

```
instruction    source_operand, destination_operand
     │               │                 │
     │               │                 └── ตำแหน่งเก็บผลลัพธ์
     │               └── ที่มาของข้อมูล (Addressing Mode ต่างๆ)
     └── คำสั่งที่จะทำ
```

### 2. ประเภทของ Addressing Modes

#### 2.1 Immediate Addressing (การระบุค่าโดยตรง)

**นิยาม:** ค่าที่ใช้อยู่ในตัว Instruction เลย ไม่ต้องอ่านจาก Memory หรือ Register

```
MOV RAX, 42        ; ค่า 42 อยู่ใน instruction โดยตรง
ADD RBX, 100       ; เพิ่ม 100 ให้ RBX โดยตรง
MOV AL, 0xFF       ; ค่า hex 0xFF อยู่ใน instruction
```

**ข้อดี:** เร็วที่สุด เพราะไม่ต้องอ่านจาก Memory
**ข้อเสีย:** ค่าต้องคงที่ ณ เวลา Compile ไม่ยืดหยุ่น

```
┌─────────────────────────────────────┐
│  Instruction:  MOV RAX, 42          │
│                                     │
│  Encoding:  [B8] [2A 00 00 00]     │
│             opcode   immediate      │
│             (MOV)    (42 = 0x2A)   │
└─────────────────────────────────────┘
```

#### 2.2 Register Addressing (การระบุผ่าน Register)

**นิยาม:** ใช้ Register เป็นตัวอ้างอิงข้อมูล ค่าอยู่ใน Register

```
MOV RAX, RBX       ; คัดลอกค่าจาก RBX ไปยัง RAX
ADD RAX, RCX       ; บวก RCX เข้า RAX
XCHG RDX, RSI      ; สลับค่า RDX กับ RSI
```

**ข้อดี:** เร็วมาก เพราะ Register อยู่ใน CPU ไม่ต้องเข้า Memory
**ข้อเสีย:** มี Register จำกัด (16 ตัวใน x64)

#### 2.3 Direct/Absolute Memory Addressing (การระบุ Address โดยตรง)

**นิยาม:** ระบุ Address ของ Memory โดยตรงเป็นตัวเลข

```
section .data
    myVar dq 0      ; ตัวแปร 64-bit ที่ address คงที่

section .text
    MOV RAX, [myVar]        ; อ่านค่าจาก address ของ myVar
    MOV [myVar], RBX        ; เขียนค่าลงที่ address ของ myVar
    MOV QWORD [0x601000], 5 ; เขียนที่ absolute address (ไม่แนะนำ)
```

**หมายเหตุ:** ใน 64-bit mode มักใช้ RIP-relative แทน Absolute addressing

#### 2.4 Register Indirect Addressing [reg]

**นิยาม:** ใช้ Register เก็บ Address ของ Memory ที่จะอ่าน/เขียน

```
; RAX เก็บ Address
MOV RBX, [RAX]      ; อ่านค่าจาก address ที่ RAX ชี้ไป
MOV [RBX], RCX      ; เขียนค่าไปที่ address ที่ RBX ชี้ไป

; เทียบกับ C:
; int* ptr = ...;
; value = *ptr;      ←→  MOV RBX, [RAX]
; *ptr = value;      ←→  MOV [RBX], RCX
```

**การใช้งาน:** Pointer dereference, Array traversal

#### 2.5 Base + Displacement Addressing [reg + offset]

**นิยาม:** ใช้ Register เป็น Base Address แล้วบวก Constant offset

```
; RBP เป็น Base, offset เป็นค่าคงที่
MOV RAX, [RBP - 8]      ; อ่าน local variable จาก stack
MOV [RBP + 16], RDI     ; เขียน function argument บน stack
MOV AL,  [RBX + 4]      ; อ่าน byte ที่ offset 4 จาก struct

; เทียบกับ C struct:
; struct Point { int x; int y; };
; struct Point* p = ...;
; p->y  ←→  MOV EAX, [RBX + 4]   (assuming x at offset 0, y at offset 4)
```

#### 2.6 Indexed Addressing [base + index*scale + displacement]

**นิยาม:** รูปแบบที่ซับซ้อนที่สุด ใช้สำหรับ Array indexing

```
; รูปแบบ: [base + index * scale + displacement]
; scale: 1, 2, 4, หรือ 8 เท่านั้น

MOV RAX, [RBX + RCX*8]         ; array[RCX] ถ้า element size = 8
MOV RAX, [RBX + RCX*4 + 8]     ; array[RCX+2] สำหรับ int array
MOV AL,  [RBP + RAX*1 + 0]     ; byte array[RAX]

; เทียบกับ C:
; long array[] = ...;
; value = array[i];   ←→  MOV RAX, [array + RCX*8]
```

#### 2.7 RIP-Relative Addressing (x64 เท่านั้น)

**นิยาม:** ใน 64-bit mode ใช้ RIP (Instruction Pointer) เป็น Base Address
เป็นวิธีหลักในการอ้างอิง Global Data ใน x64

```
; NASM ทำให้ automatic ในหลายกรณี
section .data
    message db "Hello", 0

section .text
    LEA RDI, [rel message]   ; RIP-relative address ของ message
    MOV RAX, [rel myVar]     ; อ่าน myVar แบบ RIP-relative
```

**ข้อดี:** ทำให้ Code relocatable (Position Independent Code - PIC)
สำคัญมากสำหรับ Shared Libraries

---

### 3. SIB Byte (Scale-Index-Base)

**SIB** = Scale, Index, Base เป็น byte พิเศษในการ encoding ของ x86 instruction
ใช้เมื่อต้องการ indexed addressing ที่ซับซ้อน

```
┌─────────────────────────────────────────────┐
│  SIB Byte (8 bits):                         │
│  ┌──────┬────────┬────────┐                 │
│  │Scale │ Index  │  Base  │                 │
│  │2 bits│ 3 bits │ 3 bits │                 │
│  └──────┴────────┴────────┘                 │
│                                             │
│  Scale: 00=1, 01=2, 10=4, 11=8             │
│  Index: Register หนึ่งในนั้น (ยกเว้น RSP)  │
│  Base:  Register หนึ่งในนั้น               │
└─────────────────────────────────────────────┘
```

**ตัวอย่าง:**
```
MOV RAX, [RBX + RCX*4 + 8]
         ────  ─────────── ─
         Base   Index*Scale Disp
         
SIB = Scale(4→10) | Index(RCX) | Base(RBX)
```

---

### 4. Segment Override Prefixes

ใน Real Mode และ Protected Mode x86 มี Segment Registers:
- **CS** (Code Segment)
- **DS** (Data Segment)  
- **SS** (Stack Segment)
- **ES**, **FS**, **GS** (Extra Segments)

```
; Default: data access ใช้ DS
MOV RAX, [RBX]          ; same as MOV RAX, [DS:RBX]

; Segment Override
MOV RAX, [FS:RBX]       ; อ่านจาก FS segment (Thread Local Storage)
MOV RAX, [GS:0]         ; อ่านจาก GS segment (common ใน kernel/OS)

; Thread Local Storage (TLS) ใน Linux x64
MOV RAX, [FS:0x28]      ; Stack canary ใน Linux
```

**การใช้งานจริงใน x64 Linux:**
- `FS` register ชี้ไปที่ Thread Control Block (TCB)
- `GS` register ใช้ใน kernel mode

---

### 5. LEA vs MOV

#### LEA (Load Effective Address)
**LEA** คำนวณ Address แต่ **ไม่** อ่านค่าจาก Memory

```
; MOV อ่านค่าจาก Memory
MOV RAX, [RBX + RCX*4]    ; RAX = ค่าที่ address (RBX + RCX*4)

; LEA คำนวณ Address เท่านั้น
LEA RAX, [RBX + RCX*4]    ; RAX = RBX + RCX*4  (แค่บวกเลข!)
```

**การใช้ LEA สำหรับ Arithmetic (Trick!):**
```
; LEA ทำ multiply และ add ในคำสั่งเดียว!
LEA RAX, [RAX + RAX*4]    ; RAX = RAX * 5  (เร็วกว่า IMUL)
LEA RDX, [RBX + RBX*2]    ; RDX = RBX * 3
LEA RCX, [RAX + RBX + 8]  ; RCX = RAX + RBX + 8 (add 2 registers!)
```

---

### 6. 32-bit vs 64-bit Addressing Differences

| Feature | 32-bit (x86) | 64-bit (x86-64) |
|---------|-------------|-----------------|
| Address size | 32-bit | 64-bit (default) |
| Registers | EAX, EBX, ... | RAX, RBX, ... |
| Default addressing | Absolute | RIP-relative |
| Max memory | 4 GB | 16 Exabytes |
| Stack alignment | 4 bytes | 16 bytes |
| New registers | ไม่มี | R8-R15, RIP |
| Segment override | สำคัญมาก | ใช้น้อยลง |

**ข้อแตกต่างสำคัญ:**
```
; 32-bit
MOV EAX, [EBX + ECX*4]     ; ทำงานใน 32-bit

; 64-bit
MOV RAX, [RBX + RCX*4]     ; ทำงานใน 64-bit
MOV RAX, [RBX + R8*4]      ; ใช้ R8-R15 ได้ด้วย!

; ข้อระวัง: 32-bit operation ใน 64-bit mode จะ zero-extend!
MOV EAX, EBX               ; นี้จะ zero-extend ไปยัง RAX (upper 32-bits เป็น 0)
```

---

## Code Examples

### Example 1: Immediate และ Register Addressing พื้นฐาน

```nasm
; ===================================================
; File: addressing_basic.asm
; Description: ตัวอย่าง Immediate และ Register Addressing
; Compile: nasm -f elf64 addressing_basic.asm -o addressing_basic.o
;          gcc -no-pie addressing_basic.o -o addressing_basic
; Run: ./addressing_basic
; Expected Output:
;   RAX = 100
;   RBX = 200
;   Sum = 300
;   Product (LEA) = 500
; ===================================================

section .data
    fmt_val     db "RAX = %ld", 10, 0       ; format string สำหรับ printf
    fmt_val2    db "RBX = %ld", 10, 0
    fmt_sum     db "Sum = %ld", 10, 0
    fmt_prod    db "Product (LEA) = %ld", 10, 0

section .text
    global main
    extern printf

main:
    push rbp                    ; บันทึก base pointer
    mov rbp, rsp                ; ตั้งค่า stack frame
    sub rsp, 16                 ; จองพื้นที่ stack (aligned 16 bytes)

    ; ========================================
    ; Immediate Addressing
    ; ========================================
    mov rax, 100                ; RAX = 100 (immediate value)
    mov rbx, 200                ; RBX = 200 (immediate value)
    
    ; แสดงค่า RAX
    lea rdi, [rel fmt_val]      ; RDI = address ของ format string (RIP-relative)
    mov rsi, rax                ; RSI = ค่า RAX (100)
    xor eax, eax                ; EAX = 0 (จำนวน floating point args)
    call printf                 ; printf("RAX = %ld\n", 100)
    
    ; ========================================
    ; Register Addressing
    ; ========================================
    mov rax, 100                ; ตั้งค่า RAX อีกครั้ง
    mov rbx, 200                ; ตั้งค่า RBX

    ; แสดงค่า RBX
    lea rdi, [rel fmt_val2]     ; format string
    mov rsi, rbx                ; RSI = RBX (200)
    xor eax, eax
    call printf
    
    ; ========================================
    ; Register Arithmetic
    ; ========================================
    mov rax, 100
    mov rbx, 200
    add rax, rbx                ; RAX = RAX + RBX = 300 (register-to-register)
    
    ; แสดงผลบวก
    lea rdi, [rel fmt_sum]
    mov rsi, rax                ; RSI = 300
    xor eax, eax
    call printf
    
    ; ========================================
    ; LEA สำหรับ Arithmetic (Trick!)
    ; ========================================
    mov rax, 100                ; RAX = 100
    ; RAX * 5 โดยใช้ LEA
    ; [RAX + RAX*4] = RAX + 4*RAX = 5*RAX
    lea rax, [rax + rax*4]      ; RAX = 100 + 100*4 = 100 + 400 = 500
    
    ; แสดงผลคูณ
    lea rdi, [rel fmt_prod]
    mov rsi, rax                ; RSI = 500
    xor eax, eax
    call printf
    
    ; ========================================
    ; Cleanup and Return
    ; ========================================
    xor eax, eax                ; return 0
    mov rsp, rbp                ; คืนค่า stack pointer
    pop rbp                     ; คืนค่า base pointer
    ret
```

**คำอธิบาย:**
- `MOV RAX, 100` — Immediate addressing: ค่า 100 อยู่ใน instruction
- `MOV RBX, RAX` — Register addressing: คัดลอกระหว่าง registers
- `LEA RAX, [RAX + RAX*4]` — ใช้ LEA คูณด้วย 5 โดยไม่ใช้ IMUL

**Compile และ Run:**
```bash
nasm -f elf64 addressing_basic.asm -o addressing_basic.o
gcc -no-pie addressing_basic.o -o addressing_basic
./addressing_basic
```

**Expected Output:**
```
RAX = 100
RBX = 200
Sum = 300
Product (LEA) = 500
```

---

### Example 2: Direct Memory Addressing และ Pointer Dereference

```nasm
; ===================================================
; File: memory_addressing.asm
; Description: การอ่าน/เขียน Memory ด้วย Direct และ Indirect Addressing
; Compile: nasm -f elf64 memory_addressing.asm -o memory_addressing.o
;          gcc -no-pie memory_addressing.o -o memory_addressing
; Run: ./memory_addressing
; Expected Output:
;   Direct: value = 42
;   Modified: value = 99
;   Pointer: *ptr = 42
;   Array[3] = 4
; ===================================================

section .data
    ; Direct memory locations (Global variables)
    myValue     dq 42               ; int64_t myValue = 42
    myArray     dq 1, 2, 3, 4, 5    ; int64_t myArray[5] = {1,2,3,4,5}
    
    ; Format strings
    fmt_direct  db "Direct: value = %ld", 10, 0
    fmt_modify  db "Modified: value = %ld", 10, 0
    fmt_ptr     db "Pointer: *ptr = %ld", 10, 0
    fmt_arr     db "Array[3] = %ld", 10, 0

section .text
    global main
    extern printf

main:
    push rbp
    mov rbp, rsp
    sub rsp, 32                     ; จองพื้นที่ stack

    ; ========================================
    ; Direct Memory Addressing
    ; อ่านค่าจาก Global Variable โดยตรง
    ; ========================================
    mov rax, [rel myValue]          ; RAX = myValue (42)
                                    ; [rel label] คือ RIP-relative (แนะนำใน 64-bit)
    
    lea rdi, [rel fmt_direct]       ; format string
    mov rsi, rax                    ; ค่าที่อ่านได้
    xor eax, eax
    call printf                     ; Output: "Direct: value = 42"
    
    ; ========================================
    ; เขียนค่าลง Memory โดยตรง
    ; ========================================
    mov QWORD [rel myValue], 99     ; myValue = 99 (เขียนด้วย immediate)
    mov rax, [rel myValue]          ; อ่านกลับมาตรวจสอบ
    
    lea rdi, [rel fmt_modify]
    mov rsi, rax                    ; ค่าใหม่ = 99
    xor eax, eax
    call printf                     ; Output: "Modified: value = 99"
    
    ; ========================================
    ; Indirect Memory Addressing (Pointer)
    ; ใช้ Register เก็บ Address
    ; ========================================
    mov QWORD [rel myValue], 42     ; คืนค่าเดิม myValue = 42
    lea rbx, [rel myValue]          ; RBX = address ของ myValue (like: int* ptr = &myValue)
    mov rax, [rbx]                  ; RAX = *ptr = 42 (dereference pointer)
    
    lea rdi, [rel fmt_ptr]
    mov rsi, rax                    ; ค่าที่ dereference
    xor eax, eax
    call printf                     ; Output: "Pointer: *ptr = 42"
    
    ; ========================================
    ; Array Access ด้วย Indexed Addressing
    ; myArray[3] = element ที่ index 3
    ; ========================================
    lea rbx, [rel myArray]          ; RBX = base address ของ array
    mov rcx, 3                      ; RCX = index (3)
    mov rax, [rbx + rcx*8]          ; RAX = myArray[3]
                                    ; แต่ละ element เป็น dq (8 bytes)
                                    ; address = base + index * sizeof(element)
                                    ; = myArray + 3 * 8 = myArray + 24
    
    lea rdi, [rel fmt_arr]
    mov rsi, rax                    ; ค่า = 4 (index 0,1,2,3 → ค่า 1,2,3,4)
    xor eax, eax
    call printf                     ; Output: "Array[3] = 4"
    
    ; Return
    xor eax, eax
    mov rsp, rbp
    pop rbp
    ret
```

**Compile และ Run:**
```bash
nasm -f elf64 memory_addressing.asm -o memory_addressing.o
gcc -no-pie memory_addressing.o -o memory_addressing
./memory_addressing
```

**Expected Output:**
```
Direct: value = 42
Modified: value = 99
Pointer: *ptr = 42
Array[3] = 4
```

---

### Example 3: Stack Frame Addressing และ Base+Displacement

```nasm
; ===================================================
; File: stack_addressing.asm
; Description: การใช้ Base+Displacement Addressing กับ Stack
;              จำลองการทำงานของ Local Variables ใน Function
; Compile: nasm -f elf64 stack_addressing.asm -o stack_addressing.o
;          gcc -no-pie stack_addressing.o -o stack_addressing
; Run: ./stack_addressing
; Expected Output:
;   a = 10
;   b = 20
;   c = 30
;   Sum = 60
;   Stack layout demo complete
; ===================================================

section .data
    fmt_a       db "a = %ld", 10, 0
    fmt_b       db "b = %ld", 10, 0
    fmt_c       db "c = %ld", 10, 0
    fmt_sum     db "Sum = %ld", 10, 0
    fmt_done    db "Stack layout demo complete", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชันที่มี local variables บน stack
; void demo_stack_frame()
demo_stack_frame:
    ; ===== Function Prologue =====
    push rbp                        ; บันทึก caller's RBP
    mov rbp, rsp                    ; ตั้งค่า frame pointer
    sub rsp, 32                     ; จอง 32 bytes สำหรับ local variables
                                    ; (align ไปที่ 16-byte boundary)
    
    ; Stack layout หลังจาก prologue:
    ; [RBP + 8]  = return address (push โดย call)
    ; [RBP + 0]  = saved RBP
    ; [RBP - 8]  = local variable a
    ; [RBP - 16] = local variable b  
    ; [RBP - 24] = local variable c
    ; [RBP - 32] = (padding/alignment)
    
    ; ===== กำหนดค่า Local Variables =====
    mov QWORD [rbp - 8],  10        ; a = 10 (Base+Displacement addressing)
    mov QWORD [rbp - 16], 20        ; b = 20
    mov QWORD [rbp - 24], 30        ; c = 30
    
    ; ===== แสดงค่า a =====
    lea rdi, [rel fmt_a]
    mov rsi, [rbp - 8]              ; อ่าน a = 10
    xor eax, eax
    call printf
    
    ; ===== แสดงค่า b =====
    lea rdi, [rel fmt_b]
    mov rsi, [rbp - 16]             ; อ่าน b = 20
    xor eax, eax
    call printf
    
    ; ===== แสดงค่า c =====
    lea rdi, [rel fmt_c]
    mov rsi, [rbp - 24]             ; อ่าน c = 30
    xor eax, eax
    call printf
    
    ; ===== คำนวณ a + b + c =====
    mov rax, [rbp - 8]              ; RAX = a (10)
    add rax, [rbp - 16]             ; RAX += b (20) → RAX = 30
    add rax, [rbp - 24]             ; RAX += c (30) → RAX = 60
    mov [rbp - 32], rax             ; เก็บผลลัพธ์ใน local var ชั่วคราว
    
    ; แสดง Sum
    lea rdi, [rel fmt_sum]
    mov rsi, [rbp - 32]             ; อ่าน sum = 60
    xor eax, eax
    call printf
    
    ; ===== Function Epilogue =====
    mov rsp, rbp                    ; คืนค่า stack pointer
    pop rbp                         ; คืนค่า base pointer
    ret

main:
    push rbp
    mov rbp, rsp
    
    ; เรียก demo function
    call demo_stack_frame
    
    ; แสดงข้อความเสร็จสิ้น
    lea rdi, [rel fmt_done]
    xor eax, eax
    call printf
    
    xor eax, eax                    ; return 0
    pop rbp
    ret
```

**Compile และ Run:**
```bash
nasm -f elf64 stack_addressing.asm -o stack_addressing.o
gcc -no-pie stack_addressing.o -o stack_addressing
./stack_addressing
```

**Expected Output:**
```
a = 10
b = 20
c = 30
Sum = 60
Stack layout demo complete
```

---

### Example 4: Indexed Addressing กับ Array และ Struct

```nasm
; ===================================================
; File: indexed_addressing.asm
; Description: การใช้ Indexed Addressing สำหรับ Array และ Struct
; Compile: nasm -f elf64 indexed_addressing.asm -o indexed_addressing.o
;          gcc -no-pie indexed_addressing.o -o indexed_addressing
; Run: ./indexed_addressing
; Expected Output:
;   Array sum = 150
;   Max value = 50
;   Point.x = 10, Point.y = 20
;   Matrix[1][2] = 6
; ===================================================

section .data
    ; int64_t numbers[5] = {10, 20, 30, 40, 50}
    numbers     dq 10, 20, 30, 40, 50
    num_count   equ 5               ; จำนวน elements
    
    ; struct Point { int64_t x; int64_t y; }
    ; struct Point p = {10, 20}
    point_x     dq 10
    point_y     dq 20
    
    ; int64_t matrix[3][3] = {{1,2,3},{4,5,6},{7,8,9}}
    matrix      dq 1, 2, 3          ; row 0
                dq 4, 5, 6          ; row 1
                dq 7, 8, 9          ; row 2
    
    ; Format strings
    fmt_sum     db "Array sum = %ld", 10, 0
    fmt_max     db "Max value = %ld", 10, 0
    fmt_point   db "Point.x = %ld, Point.y = %ld", 10, 0
    fmt_matrix  db "Matrix[1][2] = %ld", 10, 0

section .text
    global main
    extern printf

; ฟังก์ชัน: คำนวณผลรวม array
; Input: RDI = base address, RSI = count
; Output: RAX = sum
array_sum:
    push rbp
    mov rbp, rsp
    
    xor rax, rax                    ; RAX = 0 (accumulator)
    xor rcx, rcx                    ; RCX = 0 (index)
    
.loop:
    cmp rcx, rsi                    ; if index >= count
    jge .done                       ;   jump to done
    
    add rax, [rdi + rcx*8]          ; RAX += array[index]
                                    ; Indexed addressing: base=RDI, index=RCX, scale=8
    inc rcx                         ; index++
    jmp .loop
    
.done:
    pop rbp
    ret

; ฟังก์ชัน: หาค่าสูงสุดใน array
; Input: RDI = base address, RSI = count
; Output: RAX = max value
array_max:
    push rbp
    mov rbp, rsp
    
    mov rax, [rdi]                  ; RAX = array[0] (ค่าเริ่มต้น)
    mov rcx, 1                      ; เริ่มจาก index 1
    
.loop:
    cmp rcx, rsi                    ; if index >= count
    jge .done                       ;   done
    
    mov rdx, [rdi + rcx*8]          ; RDX = array[index]
    cmp rdx, rax                    ; if array[index] > current max
    jle .no_update
    mov rax, rdx                    ; update max
    
.no_update:
    inc rcx
    jmp .loop
    
.done:
    pop rbp
    ret

main:
    push rbp
    mov rbp, rsp
    sub rsp, 16

    ; ========================================
    ; Array Sum ด้วย Indexed Addressing
    ; ========================================
    lea rdi, [rel numbers]          ; RDI = address ของ array
    mov rsi, num_count              ; RSI = 5
    call array_sum                  ; RAX = 10+20+30+40+50 = 150
    
    mov [rsp], rax                  ; เก็บผลลัพธ์ไว้ก่อน
    lea rdi, [rel fmt_sum]
    mov rsi, [rsp]
    xor eax, eax
    call printf                     ; Output: "Array sum = 150"
    
    ; ========================================
    ; Array Max ด้วย Indexed Addressing
    ; ========================================
    lea rdi, [rel numbers]
    mov rsi, num_count
    call array_max                  ; RAX = 50
    
    lea rdi, [rel fmt_max]
    mov rsi, rax
    xor eax, eax
    call printf                     ; Output: "Max value = 50"
    
    ; ========================================
    ; Struct Access ด้วย Base+Displacement
    ; ========================================
    lea rbx, [rel point_x]          ; RBX = address ของ struct start
    mov rax, [rbx]                  ; RAX = point.x = 10   [offset 0]
    mov rdx, [rbx + 8]              ; RDX = point.y = 20   [offset 8]
    
    lea rdi, [rel fmt_point]
    mov rsi, rax                    ; x = 10
    mov rdx, rdx                    ; y = 20 (already in RDX)
    xor eax, eax
    call printf                     ; Output: "Point.x = 10, Point.y = 20"
    
    ; ========================================
    ; 2D Array (Matrix) Access
    ; matrix[row][col] = matrix[row*3 + col]
    ; matrix[1][2] = index 1*3 + 2 = 5
    ; ========================================
    lea rbx, [rel matrix]           ; RBX = base address ของ matrix
    mov rcx, 1                      ; row = 1
    mov rdx, 2                      ; col = 2
    
    ; คำนวณ index = row * cols + col = 1 * 3 + 2 = 5
    imul rcx, 3                     ; RCX = 1 * 3 = 3
    add rcx, rdx                    ; RCX = 3 + 2 = 5
    
    mov rax, [rbx + rcx*8]          ; RAX = matrix[5] = 6
                                    ; (นับจาก 0: 1,2,3,4,5,6 → index 5 = 6)
    
    lea rdi, [rel fmt_matrix]
    mov rsi, rax
    xor eax, eax
    call printf                     ; Output: "Matrix[1][2] = 6"
    
    xor eax, eax
    mov rsp, rbp
    pop rbp
    ret
```

**Compile และ Run:**
```bash
nasm -f elf64 indexed_addressing.asm -o indexed_addressing.o
gcc -no-pie indexed_addressing.o -o indexed_addressing
./indexed_addressing
```

**Expected Output:**
```
Array sum = 150
Max value = 50
Point.x = 10, Point.y = 20
Matrix[1][2] = 6
```

---

### Example 5: LEA Tricks และ Complex Address Calculations

```nasm
; ===================================================
; File: lea_tricks.asm
; Description: เทคนิคการใช้ LEA สำหรับ Arithmetic ที่ซับซ้อน
;              และ RIP-relative addressing
; Compile: nasm -f elf64 lea_tricks.asm -o lea_tricks.o
;          gcc -no-pie lea_tricks.o -o lea_tricks
; Run: ./lea_tricks
; Expected Output:
;   n*3 = 30
;   n*5 = 50
;   n*9 = 90
;   n*10 = 100
;   n*25 = 250
;   complex = 37 (3*a + 2*b + 1)
; ===================================================

section .data
    fmt_mul3    db "n*3 = %ld", 10, 0
    fmt_mul5    db "n*5 = %ld", 10, 0
    fmt_mul9    db "n*9 = %ld", 10, 0
    fmt_mul10   db "n*10 = %ld", 10, 0
    fmt_mul25   db "n*25 = %ld", 10, 0
    fmt_complex db "complex = %ld (3*a + 2*b + 1)", 10, 0

section .text
    global main
    extern printf

; ===================================================
; LEA Tricks: การคูณโดยไม่ใช้ IMUL
; ===================================================
main:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov rbx, 10                     ; n = 10 (ค่าทดสอบ)

    ; ========================================
    ; n * 3 = n + n*2 = n + n*2
    ; LEA RAX, [RBX + RBX*2] → RBX + RBX*2 = 1*RBX + 2*RBX = 3*RBX
    ; ========================================
    lea rax, [rbx + rbx*2]          ; RAX = RBX * 3 = 30
    
    lea rdi, [rel fmt_mul3]
    mov rsi, rax
    xor eax, eax
    call printf

    ; ========================================
    ; n * 5 = n + n*4 = n*1 + n*4
    ; LEA RAX, [RBX + RBX*4]
    ; ========================================
    lea rax, [rbx + rbx*4]          ; RAX = RBX * 5 = 50
    
    lea rdi, [rel fmt_mul5]
    mov rsi, rax
    xor eax, eax
    call printf

    ; ========================================
    ; n * 9 = n + n*8
    ; ========================================
    lea rax, [rbx + rbx*8]          ; RAX = RBX * 9 = 90
    
    lea rdi, [rel fmt_mul9]
    mov rsi, rax
    xor eax, eax
    call printf

    ; ========================================
    ; n * 10 = n*2 * 5
    ; ขั้นที่ 1: RDX = RBX * 2
    ; ขั้นที่ 2: RAX = RDX + RDX*4 = RDX * 5 = (RBX*2) * 5 = RBX * 10
    ; ========================================
    lea rdx, [rbx*2]                ; RDX = RBX * 2 = 20
    lea rax, [rdx + rdx*4]          ; RAX = RDX * 5 = 20 * 5 = 100
    
    lea rdi, [rel fmt_mul10]
    mov rsi, rax
    xor eax, eax
    call printf

    ; ========================================
    ; n * 25 = n * 5 * 5
    ; ขั้นที่ 1: RDX = RBX * 5
    ; ขั้นที่ 2: RAX = RDX + RDX*4 = RDX * 5 = (RBX*5) * 5 = RBX * 25
    ; ========================================
    lea rdx, [rbx + rbx*4]          ; RDX = RBX * 5 = 50
    lea rax, [rdx + rdx*4]          ; RAX = RDX * 5 = 250
    
    lea rdi, [rel fmt_mul25]
    mov rsi, rax
    xor eax, eax
    call printf

    ; ========================================
    ; Complex Address Calculation
    ; result = 3*a + 2*b + 1
    ; a = 7, b = 8 → 3*7 + 2*8 + 1 = 21 + 16 + 1 = 38
    ; Wait! Let me use a=8, b=6: 3*8+2*6+1 = 24+12+1 = 37
    ; ========================================
    mov rcx, 8                      ; a = 8
    mov rdx, 6                      ; b = 6
    
    ; step 1: R8 = 3*a
    lea r8, [rcx + rcx*2]           ; R8 = a * 3 = 24
    
    ; step 2: R9 = 2*b
    lea r9, [rdx*2]                 ; R9 = b * 2 = 12
    
    ; step 3: RAX = R8 + R9 + 1
    lea rax, [r8 + r9 + 1]          ; RAX = 24 + 12 + 1 = 37
    
    lea rdi, [rel fmt_complex]
    mov rsi, rax
    xor eax, eax
    call printf

    xor eax, eax
    mov rsp, rbp
    pop rbp
    ret
```

**Compile และ Run:**
```bash
nasm -f elf64 lea_tricks.asm -o lea_tricks.o
gcc -no-pie lea_tricks.o -o lea_tricks
./lea_tricks
```

**Expected Output:**
```
n*3 = 30
n*5 = 50
n*9 = 90
n*10 = 100
n*25 = 250
complex = 37 (3*a + 2*b + 1)
```

---

### Example 6: Segment Override และ Memory Size Specifiers

```nasm
; ===================================================
; File: memory_sizes.asm
; Description: การใช้ Memory Size Specifiers (BYTE, WORD, DWORD, QWORD)
;              และการเข้าถึง Memory ในขนาดต่างๆ
; Compile: nasm -f elf64 memory_sizes.asm -o memory_sizes.o
;          gcc -no-pie memory_sizes.o -o memory_sizes
; Run: ./memory_sizes
; Expected Output:
;   Full QWORD = 305419896 (0x12345678)
;   DWORD part = 305419896
;   WORD part = 22136 (0x5678)
;   BYTE part = 120 (0x78)
;   Packed value = 72623859790382856 (0x0102030405060708)
; ===================================================

section .data
    ; ค่า 64-bit
    big_value   dq 0x0000000012345678     ; qword ที่มีเฉพาะ lower 32-bit
    
    ; ค่าที่มีข้อมูลทุก byte
    packed      dq 0x0102030405060708     ; แต่ละ byte มีค่าต่างกัน
    
    fmt_qword   db "Full QWORD = %ld (0x%lX)", 10, 0
    fmt_dword   db "DWORD part = %ld", 10, 0
    fmt_word    db "WORD part = %ld (0x%04lX)", 10, 0
    fmt_byte    db "BYTE part = %ld (0x%02lX)", 10, 0
    fmt_packed  db "Packed value = %lu (0x%016lX)", 10, 0

section .text
    global main
    extern printf

main:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    ; ========================================
    ; อ่าน 64-bit (QWORD)
    ; ========================================
    mov rax, [rel big_value]        ; อ่านทั้ง 64 bits
    
    lea rdi, [rel fmt_qword]
    mov rsi, rax
    mov rdx, rax                    ; สำหรับ hex format
    xor eax, eax
    call printf

    ; ========================================
    ; อ่าน 32-bit (DWORD) — lower 32 bits
    ; ========================================
    mov eax, DWORD [rel big_value]  ; อ่านแค่ 32 bits (zero-extends to RAX)
    
    lea rdi, [rel fmt_dword]
    mov rsi, rax                    ; RAX ถูก zero-extend แล้ว
    xor eax, eax
    call printf

    ; ========================================
    ; อ่าน 16-bit (WORD) — lower 16 bits
    ; ========================================
    xor rax, rax                    ; clear RAX ก่อน
    mov ax, WORD [rel big_value]    ; อ่านแค่ 16 bits (ไม่ zero-extend!)
                                    ; AX = lower 16 bits = 0x5678 = 22136
    
    lea rdi, [rel fmt_word]
    movzx rsi, ax                   ; zero-extend AX to RSI
    mov rdx, rsi                    ; สำหรับ hex format
    xor eax, eax
    call printf

    ; ========================================
    ; อ่าน 8-bit (BYTE) — lowest byte
    ; ========================================
    xor rax, rax
    mov al, BYTE [rel big_value]    ; อ่านแค่ 8 bits
                                    ; AL = lowest byte = 0x78 = 120
    
    lea rdi, [rel fmt_byte]
    movzx rsi, al                   ; zero-extend AL to RSI
    mov rdx, rsi                    ; สำหรับ hex format
    xor eax, eax
    call printf

    ; ========================================
    ; แสดง Packed Value
    ; ========================================
    mov rax, [rel packed]           ; อ่านค่า packed
    
    lea rdi, [rel fmt_packed]
    mov rsi, rax
    mov rdx, rax
    xor eax, eax
    call printf

    xor eax, eax
    mov rsp, rbp
    pop rbp
    ret
```

**Compile และ Run:**
```bash
nasm -f elf64 memory_sizes.asm -o memory_sizes.o
gcc -no-pie memory_sizes.o -o memory_sizes
./memory_sizes
```

**Expected Output:**
```
Full QWORD = 305419896 (0x12345678)
DWORD part = 305419896
WORD part = 22136 (0x5678)
BYTE part = 120 (0x78)
Packed value = 72623859790382856 (0x0102030405060708)
```

---

## Common Mistakes and Pitfalls

### 1. ลืม Size Specifier ทำให้เกิด Ambiguous Error

```nasm
; WRONG: NASM ไม่รู้ว่าจะเขียนกี่ bytes
MOV [RBX], 42           ; Error: operation size not specified

; CORRECT: ระบุขนาดอย่างชัดเจน
MOV BYTE  [RBX], 42     ; เขียน 1 byte
MOV WORD  [RBX], 42     ; เขียน 2 bytes
MOV DWORD [RBX], 42     ; เขียน 4 bytes
MOV QWORD [RBX], 42     ; เขียน 8 bytes
```

### 2. 32-bit Operation ใน 64-bit Mode Zero-Extends (Silent Bug!)

```nasm
; ตัวอย่าง Bug ที่อาจไม่สังเกตเห็น
MOV RAX, 0xFFFFFFFFFFFFFFFF    ; RAX = -1 (all bits set)
MOV EAX, 0                     ; คิดว่าแค่ clear lower 32 bits...
; แต่!! จริงๆ RAX = 0 ทั้งหมด! (zero-extended to 64 bits)

; ถ้าต้องการ clear แค่ lower 32 bits และเก็บ upper 32 bits ไว้:
; ไม่มีทาง! 32-bit operation ใน 64-bit mode จะ zero-extend เสมอ
; ต้องใช้วิธีอื่น เช่น:
AND RAX, 0xFFFFFFFF00000000    ; เก็บ upper 32 bits
```

### 3. Stack Alignment ผิด ทำให้ crash

```nasm
; WRONG: ไม่ align stack ก่อน call
call some_function        ; อาจ crash ถ้า callee ใช้ SSE/AVX

; CORRECT: ตรวจสอบ alignment (RSP ต้อง divisible by 16 ก่อน call)
; เมื่อ push RBP และ call (ทำให้ RSP เลื่อน 8+8=16 bytes)
push rbp
mov rbp, rsp
; RSP ตอนนี้ aligned ที่ 16 bytes แล้ว (ถ้า stack ก่อน call aligned)
call some_function        ; OK!
```

### 4. Base Register ใช้ RSP เป็น Index ไม่ได้ (SIB limitation)

```nasm
; WRONG: RSP ไม่สามารถเป็น Index register ใน SIB
MOV RAX, [RBX + RSP*4]   ; Error! RSP ไม่สามารถเป็น Index

; CORRECT: ใช้ Register อื่น
MOV RCX, RSP
MOV RAX, [RBX + RCX*4]   ; OK ใช้ RCX แทน
```

### 5. ลืม `rel` ใน NASM 64-bit Mode

```nasm
section .data
    myvar dq 42

section .text
; WRONG: ใน default NASM configuration, นี้อาจ generate absolute address
; ซึ่งจะ fail ใน position-independent code
MOV RAX, [myvar]          ; อาจไม่ทำงานถูกต้องใน PIE executable

; CORRECT: ใช้ rel สำหรับ RIP-relative (แนะนำเสมอใน 64-bit)
MOV RAX, [rel myvar]      ; ทำงานถูกต้องเสมอ
LEA RDI, [rel myvar]      ; สำหรับ address

; หรือใน NASM เปิด DEFAULT REL ที่ต้นไฟล์:
default rel               ; ทำให้ทุก memory reference เป็น RIP-relative
```

### 6. Sign Extension ใน 32-bit Displacement

```nasm
; Displacement ขนาด 32-bit จะถูก sign-extended เป็น 64-bit!
MOV RAX, [RBX + 0x80000000]   ; 0x80000000 sign-extended = 0xFFFFFFFF80000000
                                ; นี่อาจไม่ใช่สิ่งที่ต้องการ!

; ถ้าต้องการ address > 2GB จาก base:
MOV RDX, 0x80000000            ; ใส่ค่าใน register ก่อน
ADD RBX, RDX                   ; แล้วค่อย add
MOV RAX, [RBX]                 ; แล้วอ่าน
```

### 7. Memory-to-Memory Operation ไม่ได้!

```nasm
; WRONG: x86 ไม่อนุญาต memory-to-memory operation ในคำสั่งเดียว
MOV [dest], [src]         ; Error! ทำไม่ได้!

; CORRECT: ต้องผ่าน Register
MOV RAX, [src]            ; อ่านจาก src ลง register
MOV [dest], RAX           ; เขียนจาก register ไป dest
```

---

## Advanced Techniques

### 1. RIP-Relative Addressing สำหรับ Position Independent Code (PIC)

```nasm
; ===================================================
; Position Independent Code (PIC) สำหรับ Shared Libraries
; ===================================================

default rel         ; ทำให้ทุก [] reference เป็น RIP-relative

section .data
    global_var  dq 0

section .text
    global pic_function

pic_function:
    push rbp
    mov rbp, rsp
    
    ; ใน PIC code ต้องใช้ RIP-relative เสมอ
    mov rax, [global_var]           ; OK เพราะ default rel เปิดอยู่
    lea rdi, [global_var]           ; ได้ address ของ global_var
    
    pop rbp
    ret
```

### 2. Thread Local Storage ด้วย FS Segment

```nasm
; การอ่าน Thread Local Storage ใน Linux x64
; FS register ชี้ไปที่ Thread Control Block

section .text
    global read_tls_example

read_tls_example:
    ; อ่าน Stack Canary จาก TLS (ตำแหน่ง FS:0x28 ใน Linux)
    mov rax, [fs:0x28]              ; Stack canary value
    
    ; อ่าน Thread ID จาก TLS (ตำแหน่ง FS:0 ใน glibc)
    ; mov rax, [fs:0]               ; Pointer to TLS struct itself
    
    ret
```

### 3. VSIB Addressing (Vector SIB) สำหรับ AVX2

```nasm
; VSIB = Vector SIB ใช้ YMM/ZMM register เป็น index
; ต้องการ AVX2 หรือ AVX-512

; ตัวอย่าง (ต้องการ CPU ที่รองรับ AVX2)
; VPGATHERDD YMM0, [RBX + YMM1*4], YMM2
; อ่าน 8 integers พร้อมกัน โดยใช้ indices ใน YMM1
```

### 4. LEA สำหรับ Pointer Arithmetic ที่ซับซ้อน

```nasm
; คำนวณ address ของ element ใน 2D array
; element_ptr = base + row * row_size + col * element_size
; สมมติ: rows=4, cols=4, element_size=8 bytes
; row_size = 4 * 8 = 32 bytes

; Input: RBX = base, RCX = row, RDX = col
calc_2d_element:
    push rbp
    mov rbp, rsp
    
    ; step 1: R8 = row * row_size (= row * 32)
    lea r8, [rcx*4]             ; R8 = row * 4
    sal r8, 3                   ; R8 *= 8 (shift left 3 = *8)
                                ; R8 = row * 32
    
    ; step 2: R9 = col * element_size (= col * 8)  
    lea r9, [rdx*8]             ; R9 = col * 8
    
    ; step 3: RAX = base + R8 + R9
    lea rax, [rbx + r8 + r9]    ; complex address calculation ด้วย LEA เดียว
    ; แต่จริงๆ ต้องใช้หลาย step เพราะ LEA รับ 2 operands เท่านั้น
    
    pop rbp
    ret
```

### 5. การใช้ Address Sizes ต่างกัน (Address Size Override Prefix)

```nasm
; ใน 64-bit code ปกติจะใช้ 64-bit addresses
; แต่สามารถใช้ 32-bit address ได้ด้วย prefix 0x67

; ใน NASM ทำได้โดย:
; a32 MOV RAX, [EBX]        ; ใช้ EBX เป็น 32-bit address (zero-extended)
; a32 prefix ทำให้ใช้ 32-bit addressing

; ใช้กรณี: ต้องการ wrap around ที่ 4GB boundary
```

---

## Exercises

### Exercise 1: Basic Addressing Modes (ระดับ: ง่าย)

**โจทย์:** เขียนโปรแกรม Assembly ที่:
1. กำหนดตัวแปร `x = 15` และ `y = 25` ใน memory
2. อ่านทั้งสองค่า
3. คำนวณ `z = x * 3 + y * 2 + 10` โดยใช้ LEA
4. แสดงผลลัพธ์

**Hint:**
```nasm
; x*3 + y*2 + 10
; Step 1: R8 = x * 3 = [rcx + rcx*2]  (สมมติ RCX = x)
; Step 2: R9 = y * 2 = [rdx*2]         (สมมติ RDX = y)
; Step 3: RAX = R8 + R9 + 10
```

**Expected Output:** `z = 105`

---

### Exercise 2: Array Traversal (ระดับ: ปานกลาง)

**โจทย์:** เขียนฟังก์ชัน Assembly ที่ค้นหาค่าในอาร์เรย์:
```
int64_t arr[] = {15, 42, 7, 99, 33, 50, 11, 88};
```
1. หาค่าต่ำสุดและค่าสูงสุด
2. คำนวณค่าเฉลี่ย (integer division)
3. แสดงผลลัพธ์ทั้งหมด

**Hint:**
```nasm
; ใช้ Indexed addressing: [base + index*8]
; Loop จาก index 0 ถึง n-1
; เปรียบเทียบและ update min/max
```

**Expected Output:**
```
Min = 7
Max = 99
Average = 43
```

---

### Exercise 3: Struct Manipulation (ระดับ: ปานกลาง)

**โจทย์:** สร้าง "Struct" ใน Assembly สำหรับนักเรียน:
```c
struct Student {
    int64_t id;       // offset 0
    int64_t score;    // offset 8
    int64_t grade;    // offset 16  (0=F,1=D,2=C,3=B,4=A)
};
```

เขียนฟังก์ชัน:
1. `set_student(ptr, id, score)` — กำหนดข้อมูลและคำนวณ grade
2. `get_grade(ptr)` — ส่งคืน grade
3. สร้าง array ของ 3 นักเรียนและแสดงผล

**Hint:**
```nasm
; grade ตาม score:
; 90+ = 4 (A), 80-89 = 3 (B), 70-79 = 2 (C), 60-69 = 1 (D), else = 0 (F)
; ใช้ cmp และ conditional jumps
; sizeof(Student) = 24 bytes
; student[i] = base + i * 24
```

---

### Exercise 4: Bubble Sort (ระดับ: ยาก)

**โจทย์:** เขียน Bubble Sort ใน Assembly:
```
Input:  [64, 25, 12, 22, 11]
Output: [11, 12, 22, 25, 64]
```

ต้องใช้:
- Indexed addressing สำหรับเข้าถึง array elements
- Nested loops (outer loop และ inner loop)
- Swap elements โดยใช้ register

**Hint:**
```nasm
; Bubble Sort Algorithm:
; for i = 0 to n-1:
;   for j = 0 to n-i-2:
;     if arr[j] > arr[j+1]:
;       swap arr[j], arr[j+1]
;
; ใช้ [base + j*8] และ [base + j*8 + 8] สำหรับ arr[j] และ arr[j+1]
```

---

### Exercise 5: String Operations (ระดับ: ยาก)

**โจทย์:** เขียนฟังก์ชัน Assembly สำหรับจัดการ String:
1. `str_length(ptr)` — คำนวณความยาว string (null-terminated)
2. `str_reverse(ptr, len)` — กลับ string in-place
3. `str_count_char(ptr, char)` — นับจำนวน character ที่ระบุ

**Test:** `"Hello, World!"` 
- Length = 13
- Reversed = `"!dlroW ,olleH"`
- Count of 'l' = 3

**Hint:**
```nasm
; str_length:
;   ใช้ indirect addressing: [RDI + RCX] หรือ SCASB instruction
;   loop จนเจอ null byte (0)

; str_reverse:
;   ใช้ two-pointer technique
;   ptr_left = ptr, ptr_right = ptr + len - 1
;   loop: swap [ptr_left], [ptr_right]; left++; right--
;   ใช้ BYTE PTR สำหรับอ่าน/เขียนทีละ byte
```

---

## Summary

### สรุป Addressing Modes ทั้งหมด

| Mode | Syntax | ตัวอย่าง | การใช้งาน |
|------|--------|---------|-----------|
| Immediate | `value` | `MOV RAX, 42` | ค่าคงที่ |
| Register | `reg` | `MOV RAX, RBX` | Copy registers |
| Direct | `[label]` | `MOV RAX, [myVar]` | Global variables |
| Indirect | `[reg]` | `MOV RAX, [RBX]` | Pointer deref |
| Base+Disp | `[reg+off]` | `MOV RAX, [RBP-8]` | Local vars, struct fields |
| Indexed | `[b+i*s+d]` | `MOV RAX, [RBX+RCX*8]` | Array access |
| RIP-relative | `[rel label]` | `LEA RDI, [rel msg]` | PIC code |

### ลำดับความเร็ว (เร็ว → ช้า)
```
Register > Immediate > Cache (L1→L2→L3) > Main Memory
```

### กฎสำคัญที่ต้องจำ

1. **ทุก Memory access ต้องมี Size specifier** เมื่อ ambiguous
2. **32-bit operation zero-extends** upper 32 bits ใน 64-bit mode
3. **RSP ไม่สามารถเป็น Index** ใน SIB byte
4. **ใช้ `rel` เสมอ** ใน 64-bit code สำหรับ data access
5. **LEA ไม่อ่าน Memory** แค่คำนวณ address
6. **Memory-to-Memory ไม่ได้** ต้องผ่าน register
7. **Scale ใน SIB** มีแค่ 1, 2, 4, 8
8. **Stack alignment** ต้อง 16 bytes ก่อน `call`

### Quick Reference: LEA Multiplication Table

```
n * 2  = LEA RAX, [RBX*2]
n * 3  = LEA RAX, [RBX + RBX*2]
n * 4  = LEA RAX, [RBX*4]
n * 5  = LEA RAX, [RBX + RBX*4]
n * 8  = LEA RAX, [RBX*8]
n * 9  = LEA RAX, [RBX + RBX*8]
n * 6  = lea rdx, [rbx+rbx*2]; lea rax, [rdx*2]    (n*3 then *2)
n * 7  = lea rdx, [rbx*8];     sub rdx, rbx         (n*8 - n)
n * 10 = lea rdx, [rbx*2];     lea rax, [rdx+rdx*4] (n*2 then *5)
```

### Tools สำหรับตรวจสอบ Addressing

```bash
# ดู machine code ที่ NASM generate
nasm -f elf64 file.asm -l file.lst    # listing file
cat file.lst                           # ดู hex encoding

# ดู disassembly
objdump -d -M intel file.o            # disassemble object file

# GDB สำหรับ debug
gdb ./program
(gdb) break main
(gdb) run
(gdb) x/10xg $rsp                     # ดู 10 qwords บน stack
(gdb) info registers                   # ดู register values
(gdb) nexti                            # execute one instruction
```

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 013: Arithmetic Operations ใน x86**
จะครอบคลุม:
- ADD, SUB, MUL, IMUL, DIV, IDIV
- ADC, SBB (Add/Subtract with Carry)
- INC, DEC
- NEG (Negate)
- MULX, ADCX, ADOX (new instructions)
- Multi-precision arithmetic
- Overflow detection และ handling

---

*Part 012 เสร็จสมบูรณ์ — Addressing Modes ใน x86*
*ต่อไป: Part 013 - Arithmetic Operations*
