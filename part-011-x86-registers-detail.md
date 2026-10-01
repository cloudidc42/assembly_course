# Part 011: x86 Registers อย่างละเอียด

## Prerequisites and Learning Objectives

### สิ่งที่ควรรู้ก่อน (Prerequisites)
- ความเข้าใจพื้นฐาน Assembly syntax (Part 001-010)
- การทำงานของ CPU เบื้องต้น
- ระบบเลขฐานสอง ฐานสิบหก
- การติดตั้ง NASM และ linker

### จุดประสงค์การเรียนรู้ (Learning Objectives)
หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. อธิบายโครงสร้างและหน้าที่ของ register ทุกตัวใน x86 architecture ได้
2. เข้าใจการ aliasing ระหว่าง AL/AH/AX/EAX/RAX
3. ใช้ segment registers อย่างถูกต้อง
4. อ่านและตีความ EFLAGS bits ทุกตัวได้
5. เขียนโปรแกรมที่ใช้ register ต่างๆ อย่างมีประสิทธิภาพ
6. เข้าใจ REX prefix และ extended registers ใน 64-bit mode
7. รู้จัก XMM/YMM/ZMM registers สำหรับ SIMD operations

---

## ส่วนที่ 1: ภาพรวม Register Architecture

### 1.1 Register คืออะไร?

**Register** คือหน่วยความจำขนาดเล็กที่อยู่ภายใน CPU โดยตรง ซึ่งมีความเร็วในการเข้าถึงสูงมากกว่า RAM หลายร้อยถึงหลายพันเท่า CPU ใช้ register เพื่อเก็บข้อมูลชั่วคราวระหว่างการคำนวณ

```
Memory Hierarchy (ลำดับความเร็ว):
┌─────────────────────────────────────────────────────┐
│  Registers (< 1 ns)    ← เร็วที่สุด, ขนาดเล็กที่สุด │
│  L1 Cache (1-4 ns)                                  │
│  L2 Cache (4-12 ns)                                 │
│  L3 Cache (12-50 ns)                                │
│  RAM (50-100 ns)                                    │
│  SSD (10,000-100,000 ns)  ← ช้าที่สุด              │
└─────────────────────────────────────────────────────┘
```

### 1.2 วิวัฒนาการของ x86 Registers

```
8-bit era (8086 - 1978):
  AH AL  BH BL  CH CL  DH DL  (8-bit registers)

16-bit era (80286 - 1982):
  AX     BX     CX     DX     (16-bit)
  SI  DI  SP  BP               (index/pointer registers)
  CS  DS  ES  SS               (segment registers)

32-bit era (80386 - 1985):
  EAX    EBX    ECX    EDX    (32-bit extended)
  ESI EDI ESP EBP              (extended index/pointers)
  FS  GS                       (additional segment registers)
  EFLAGS                       (extended flags)
  EIP                          (instruction pointer)

64-bit era (AMD64/EM64T - 2003):
  RAX    RBX    RCX    RDX    (64-bit)
  RSI RDI RSP RBP RIP          
  R8-R15                       (8 new general-purpose registers)
  XMM0-XMM15                  (128-bit SIMD)
  YMM0-YMM15                  (256-bit SIMD, AVX)
  ZMM0-ZMM31                  (512-bit SIMD, AVX-512)
```

---

## ส่วนที่ 2: General Purpose Registers (GPR)

### 2.1 Register Aliasing - การซ้อนทับของ Registers

หัวใจสำคัญของ x86 คือ register aliasing ที่ทำให้ register เดียวกันสามารถเข้าถึงได้ในหลายขนาด:

```
RAX (64-bit) - Register ทั้งหมด 64 bits
┌────────────────────────────────────────────────────────────────┐
│                            RAX (64 bits)                       │
└────────────────────────────────────────────────────────────────┘
                                    ┌───────────────────────────┐
                                    │       EAX (32 bits)       │
                                    └───────────────────────────┘
                                                    ┌───────────┐
                                                    │ AX (16 b) │
                                                    └───────────┘
                                                    ┌─────┬─────┐
                                                    │ AH  │ AL  │
                                                    │(8b) │(8b) │
                                                    └─────┴─────┘

Bit positions:
Bit: 63      32 31      16 15   8 7    0
     |         | |         | |     | |   |
     [   High  ] [   Low   ] [ AH  ][AL ]
     [            EAX              ]
     [                   AX        ]
     [                      RAX                           ]
```

### 2.2 RAX/EAX/AX/AH/AL - Accumulator Register

**หน้าที่หลัก**: ใช้สำหรับการคำนวณ arithmetic, เก็บผลลัพธ์, และใช้ใน system calls

```nasm
; ตัวอย่างการใช้งาน RAX ในรูปแบบต่างๆ
section .data
    num dd 0xFF12ABCD      ; ตัวเลข 32-bit

section .text
    global _start
_start:
    ; โหลดค่า 64-bit
    mov rax, 0x123456789ABCDEF0   ; RAX = ค่าทั้งหมด
    
    ; อ่านส่วน 32-bit ล่าง
    ; EAX = 0x9ABCDEF0 (32 bits ล่าง)
    ; หมายเหตุ: การใช้ EAX จะ zero-extend เป็น 64-bit โดยอัตโนมัติ
    
    ; อ่านส่วน 16-bit ล่าง
    ; AX = 0xDEF0 (16 bits ล่าง)
    
    ; อ่านส่วน high byte ของ AX
    ; AH = 0xDE (bits 15-8)
    
    ; อ่านส่วน low byte ของ AX  
    ; AL = 0xF0 (bits 7-0)
    
    mov eax, 0          ; exit code 0
    mov ebx, 0
    int 0x80
```

### 2.3 RBX/EBX/BX/BH/BL - Base Register

**หน้าที่หลัก**: ใช้เป็น base address สำหรับ memory addressing, และเป็น callee-saved register

```nasm
; RBX เป็น callee-saved register - ต้อง preserve ค่าไว้
; ถ้า function ใช้ RBX ต้อง push/pop ก่อน/หลังใช้งาน

section .text
    global my_function
my_function:
    push rbx            ; บันทึกค่า RBX เดิม (callee-saved)
    
    mov rbx, [rdi]      ; ใช้ RBX เป็น base pointer
    mov rax, [rbx + 8]  ; เข้าถึง memory ผ่าน RBX
    mov rax, [rbx + 16]
    
    pop rbx             ; คืนค่า RBX เดิม
    ret
```

### 2.4 RCX/ECX/CX/CH/CL - Counter Register

**หน้าที่หลัก**: ใช้เป็น loop counter, shift count, และ parameter ที่ 4 ใน Linux x86-64 calling convention

```nasm
; ตัวอย่าง: RCX ใช้กับ loop instructions
section .text
    global _start
_start:
    mov rcx, 10         ; ลูป 10 ครั้ง
    mov rax, 0          ; ผลรวม = 0
    mov rbx, 1          ; ตัวนับเริ่มที่ 1
    
loop_start:
    add rax, rbx        ; rax += rbx (บวกสะสม)
    inc rbx             ; rbx++
    loop loop_start     ; ลด rcx แล้วกระโดดถ้า rcx != 0
    
    ; rax = 1+2+3+...+10 = 55
    ; CL ใช้เป็น shift count
    mov rbx, 0xFF
    mov cl, 4           ; จำนวนบิตที่ shift
    shl rbx, cl         ; shift left 4 bits
    ; rbx = 0xFF0
    
    mov eax, 1
    xor edi, edi
    syscall
```

### 2.5 RDX/EDX/DX/DH/DL - Data Register

**หน้าที่หลัก**: ใช้ใน multiply/divide operations, I/O port address, parameter ที่ 3 ใน calling convention

```nasm
; ตัวอย่าง: การใช้ RDX ใน multiplication และ division
section .text
    global _start
_start:
    ; Multiplication: RDX:RAX = RAX * operand
    mov rax, 1000000000     ; 10^9
    mov rbx, 1000000000
    mul rbx                 ; RDX:RAX = RAX * RBX
    ; RDX มีส่วน high ของผลคูณ
    ; RAX มีส่วน low ของผลคูณ
    
    ; Division: RAX = RDX:RAX / divisor, RDX = remainder
    mov rax, 100
    xor rdx, rdx            ; ต้อง clear RDX ก่อน divide!
    mov rbx, 7
    div rbx                 ; RAX = 100/7 = 14, RDX = 100%7 = 2
    
    mov eax, 60
    xor edi, edi
    syscall
```

### 2.6 RSI/ESI - Source Index Register

**หน้าที่หลัก**: ใช้เป็น source pointer ใน string operations, parameter ที่ 2 ใน calling convention

```nasm
section .data
    src db "Hello, World!", 0
    dst times 20 db 0

section .text
    global _start
_start:
    ; RSI = source, RDI = destination สำหรับ string operations
    lea rsi, [src]      ; RSI = address ของ source string
    lea rdi, [dst]      ; RDI = address ของ destination
    
    ; copy string โดยใช้ RSI/RDI
copy_loop:
    mov al, [rsi]       ; โหลด byte จาก source
    mov [rdi], al       ; เก็บ byte ไป destination
    inc rsi             ; เลื่อน source pointer
    inc rdi             ; เลื่อน destination pointer
    test al, al         ; ตรวจสอบ null terminator
    jnz copy_loop       ; ถ้าไม่ใช่ null ให้ copy ต่อ
    
    mov eax, 60
    xor edi, edi
    syscall
```

### 2.7 RDI/EDI - Destination Index Register

**หน้าที่หลัก**: ใช้เป็น destination pointer ใน string operations, parameter แรกใน calling convention

```nasm
; ตัวอย่าง: memset using RDI
section .bss
    buffer resb 256

section .text
    global _start
_start:
    ; RDI = destination address
    lea rdi, [buffer]   ; RDI = buffer address
    mov rax, 0x4141414141414141  ; ค่า 'A' ซ้ำ 8 ครั้ง
    mov rcx, 256/8      ; จำนวน 64-bit stores
    
    ; ใช้ rep stosq เพื่อ fill memory
    rep stosq           ; เก็บ RAX ไปที่ [RDI], เลื่อน RDI, ลด RCX
    
    mov eax, 60
    xor edi, edi
    syscall
```

### 2.8 RSP/ESP - Stack Pointer

**หน้าที่หลัก**: ชี้ไปยัง top of stack - สำคัญมาก ห้ามใช้เพื่ออื่น!

```nasm
; Stack pointer ทำงานแบบ LIFO (Last In, First Out)
; Stack โตลงล่าง (จาก high address ไป low address)

section .text
    global _start
_start:
    ; ดู RSP ปัจจุบัน
    mov rax, rsp        ; บันทึก stack pointer
    
    ; PUSH = RSP -= 8, แล้ว [RSP] = value
    push rbx            ; RSP -= 8, [RSP] = RBX
    push rcx            ; RSP -= 8, [RSP] = RCX
    
    ; POP = value = [RSP], RSP += 8
    pop rcx             ; RCX = [RSP], RSP += 8
    pop rbx             ; RBX = [RSP], RSP += 8
    
    ; Stack frame สำหรับ local variables
    sub rsp, 32         ; จอง 32 bytes สำหรับ local vars
    mov qword [rsp], 100    ; local var 1
    mov qword [rsp+8], 200  ; local var 2
    add rsp, 32         ; คืน stack space
    
    mov eax, 60
    xor edi, edi
    syscall
```

### 2.9 RBP/EBP - Base Pointer

**หน้าที่หลัก**: ชี้ไปยัง base ของ stack frame ปัจจุบัน, ใช้ access local variables และ parameters

```nasm
; Stack Frame ปกติ
; [rbp+16] = parameter 1 (ถ้า push ไว้)
; [rbp+8]  = return address (push โดย CALL)
; [rbp+0]  = saved RBP (push โดยเรา)
; [rbp-8]  = local variable 1
; [rbp-16] = local variable 2

section .text
    global my_func
my_func:
    push rbp            ; บันทึก base pointer เดิม
    mov rbp, rsp        ; ตั้ง base pointer ใหม่
    sub rsp, 32         ; จอง space สำหรับ local vars
    
    ; เข้าถึง local variables ผ่าน RBP
    mov qword [rbp-8], 42       ; local var 1 = 42
    mov qword [rbp-16], 100     ; local var 2 = 100
    
    mov rax, [rbp-8]    ; อ่าน local var 1
    add rax, [rbp-16]   ; บวก local var 2
    
    ; Cleanup
    leave               ; mov rsp, rbp; pop rbp
    ret
```

---

## ส่วนที่ 3: Extended Registers ใน 64-bit Mode (R8-R15)

### 3.1 R8-R15 General Purpose Registers

ใน 64-bit mode มี registers เพิ่มเติมอีก 8 ตัว ซึ่งไม่มีใน 32-bit mode:

```
R8  = R8D (32-bit) = R8W (16-bit) = R8B (8-bit)
R9  = R9D          = R9W          = R9B
R10 = R10D         = R10W         = R10B
R11 = R11D         = R11W         = R11B
R12 = R12D         = R12W         = R12B
R13 = R13D         = R13W         = R13B
R14 = R14D         = R14W         = R14B
R15 = R15D         = R15W         = R15B
```

**หมายเหตุ**: R8-R15 ต้องใช้ REX prefix ใน machine code ทำให้ instruction ยาวขึ้น 1 byte

```nasm
; ตัวอย่าง: ใช้ R8-R15 สำหรับ parameters และ local vars
section .text
    global complex_function
; Linux x86-64 calling convention:
; RDI, RSI, RDX, RCX, R8, R9 = parameters 1-6
; R10, R11 = caller-saved (ไม่ต้อง preserve)
; R12-R15, RBX, RBP = callee-saved (ต้อง preserve)

complex_function:
    push r12            ; callee-saved
    push r13
    push r14
    push r15
    
    ; parameters ผ่านทาง:
    ; RDI = param1, RSI = param2, RDX = param3
    ; RCX = param4, R8 = param5, R9 = param6
    
    mov r12, rdi        ; บันทึก param1 ใน callee-saved register
    mov r13, rsi        ; บันทึก param2
    
    ; ทำงาน...
    mov rax, r12
    add rax, r13
    
    pop r15
    pop r14
    pop r13
    pop r12
    ret
```

### 3.2 REX Prefix ใน 64-bit

REX prefix คือ byte พิเศษที่วางก่อน instruction เพื่อ:
1. ระบุว่าใช้ 64-bit operand
2. เข้าถึง R8-R15
3. เข้าถึง uniform byte registers (SPL, BPL, SIL, DIL)

```
REX prefix byte structure:
Bit 7 6 5 4 3 2 1 0
    0 1 0 0 W R X B

W = 0: 32-bit operand size (default)
W = 1: 64-bit operand size
R = Extension of ModRM.reg field (ขยาย R8-R15)
X = Extension of SIB.index field
B = Extension of ModRM.rm or SIB.base (ขยาย R8-R15)

ตัวอย่าง:
mov rax, rbx     → 48 89 D8  (REX.W=1 = 64-bit operation)
mov r8, r9       → 4D 89 C8  (REX.W=1, REX.R=1, REX.B=1)
mov r8d, r9d     → 45 89 C8  (REX.R=1, REX.B=1, no REX.W)
```

---

## ส่วนที่ 4: Segment Registers

### 4.1 Segment Registers และการทำงาน

```
CS (Code Segment)  - ชี้ไปยัง segment ที่มี code
DS (Data Segment)  - ชี้ไปยัง segment ที่มี data (default สำหรับ data access)
ES (Extra Segment) - segment พิเศษ (ใช้กับ string operations ฝั่ง destination)
FS (F Segment)     - ใช้สำหรับ Thread Local Storage (TLS) ใน Linux/Windows
GS (G Segment)     - ใช้สำหรับ kernel data ใน Linux, TLS ใน Windows
SS (Stack Segment) - ชี้ไปยัง stack segment
```

ใน 64-bit long mode บน Linux/Windows ที่ใช้ flat memory model:
- CS, DS, ES, SS มีค่า base = 0 (ไม่มีความหมายในแง่ segmentation จริง)
- FS และ GS ยังคงมีประโยชน์สำหรับ per-thread/per-cpu data

```nasm
; ตัวอย่าง: การอ่าน Thread Local Storage ผ่าน FS
; ใน Linux, FS:0 ชี้ไปยัง thread control block (TCB)
section .text
    global get_thread_id
get_thread_id:
    ; อ่านค่าจาก FS-relative address
    mov rax, qword [fs:0]   ; อ่าน self-pointer ใน TCB
    ; ค่าที่ได้คือ address ของ TCB เอง
    ret
```

### 4.2 Segment Override Prefixes

```nasm
; ปกติ DS ใช้เป็น default สำหรับ data access
mov rax, [rbx]          ; ใช้ DS โดย default (DS:[RBX])

; Override ด้วย segment prefix
mov rax, [fs:rbx]       ; ใช้ FS segment (FS:[RBX])
mov rax, [gs:0]         ; อ่าน GS:0 (ใช้ใน kernel สำหรับ per-cpu data)

; ES ใช้กับ string instructions
; rep movsb ใช้ DS:[RSI] → ES:[RDI]
```

---

## ส่วนที่ 5: EFLAGS/RFLAGS Register อย่างละเอียด

### 5.1 โครงสร้าง FLAGS Register

EFLAGS เป็น 32-bit register (RFLAGS เป็น 64-bit ใน x86-64) ที่เก็บ status bits จากการทำงานของ CPU:

```
RFLAGS (64-bit):
Bit 63-22: Reserved (ต้องเป็น 0)
Bit 21: ID   - Identification Flag
Bit 20: VIP  - Virtual Interrupt Pending
Bit 19: VIF  - Virtual Interrupt Flag
Bit 18: AC   - Alignment Check / Access Control
Bit 17: VM   - Virtual-8086 Mode
Bit 16: RF   - Resume Flag
Bit 14: NT   - Nested Task
Bit 13-12: IOPL - I/O Privilege Level (2 bits)
Bit 11: OF  - Overflow Flag      ← สำคัญมาก
Bit 10: DF  - Direction Flag     ← ควบคุม string operations
Bit 9:  IF  - Interrupt Enable Flag
Bit 8:  TF  - Trap Flag (single-step debugging)
Bit 7:  SF  - Sign Flag          ← สำคัญมาก
Bit 6:  ZF  - Zero Flag          ← สำคัญมาก
Bit 4:  AF  - Auxiliary Carry Flag
Bit 2:  PF  - Parity Flag
Bit 0:  CF  - Carry Flag         ← สำคัญมาก
```

### 5.2 Status Flags ที่ใช้บ่อย

#### CF - Carry Flag (bit 0)
- Set เมื่อ unsigned operation มี carry/borrow
- ใช้ใน multi-precision arithmetic (ADC, SBB)

```nasm
; CF ตัวอย่าง
mov al, 0xFF
add al, 1       ; AL = 0, CF = 1 (carry เกิดขึ้น)
                ; 0xFF + 1 = 0x100, แต่ใส่ได้แค่ 8 bits = 0x00, carry = 1

mov rax, 0xFFFFFFFFFFFFFFFF
add rax, 1      ; RAX = 0, CF = 1

; ใช้ CF สำหรับ 128-bit addition:
; low = rax + rbx, high = rcx + rdx + carry
mov rax, 0xFFFFFFFFFFFFFFFF
mov rbx, 1
mov rcx, 0
mov rdx, 0
add rax, rbx    ; rax = 0, CF = 1
adc rcx, rdx    ; rcx = 0 + 0 + CF = 1  (128-bit result = 0x10000000000000000)
```

#### ZF - Zero Flag (bit 6)
- Set เมื่อผลลัพธ์เป็น 0
- ใช้มากใน conditional jumps

```nasm
; ZF ตัวอย่าง
mov eax, 5
sub eax, 5      ; EAX = 0, ZF = 1
jz zero_result  ; กระโดดถ้า ZF = 1

mov eax, 10
test eax, eax   ; ทำ AND แต่ไม่เก็บผล, set flags
jz skip         ; ถ้า EAX = 0 กระโดด
; ถ้า EAX != 0 ทำงานต่อ
skip:
```

#### SF - Sign Flag (bit 7)
- เท่ากับ bit สูงสุดของผลลัพธ์
- Set เมื่อผลลัพธ์เป็น negative (ใน signed arithmetic)

```nasm
; SF ตัวอย่าง
mov eax, 10
sub eax, 20     ; EAX = -10 = 0xFFFFFFF6, SF = 1 (MSB = 1)

mov eax, -1     ; EAX = 0xFFFFFFFF
test eax, eax   ; SF = 1 เพราะ MSB = 1
js negative     ; กระโดดถ้า SF = 1
```

#### OF - Overflow Flag (bit 11)
- Set เมื่อ signed operation มี overflow
- ต่างจาก CF ตรงที่ OF เป็นสำหรับ signed arithmetic

```nasm
; OF ตัวอย่าง
mov al, 127     ; AL = 0x7F (maximum positive 8-bit signed)
add al, 1       ; AL = 128 = 0x80 = -128 ใน signed! OF = 1
                ; ในแง่ signed: 127 + 1 = -128 (ผิด!) → OF set

mov al, -128    ; AL = 0x80
sub al, 1       ; AL = 0x7F = 127, OF = 1 (underflow)
                ; ใน signed: -128 - 1 = 127 (ผิด!) → OF set

; ตรวจ overflow หลัง signed operation
jo overflow_handler   ; Jump if Overflow
```

#### DF - Direction Flag (bit 10)
- ควบคุมทิศทางของ string operations
- DF = 0: forward (RSI/RDI เพิ่มขึ้น)
- DF = 1: backward (RSI/RDI ลดลง)

```nasm
; DF ตัวอย่าง
cld             ; Clear Direction Flag (DF = 0, forward)
lea rsi, [src]
lea rdi, [dst]
mov rcx, len
rep movsb       ; copy forward

std             ; Set Direction Flag (DF = 1, backward)
; ระวัง! ต้อง cld ก่อนออกจาก function
; ABI กำหนดให้ DF = 0 เมื่อเรียก/return function
cld             ; reset DF
```

#### PF - Parity Flag (bit 2)
- Set เมื่อจำนวน 1-bits ใน byte ต่ำสุดของผลลัพธ์เป็นเลขคู่

```nasm
; PF ตัวอย่าง (ใช้น้อยมากในโปรแกรมทั่วไป)
mov al, 0b00000111  ; 3 ones → PF = 0 (odd)
test al, al         ; ตรวจ parity
jnp odd_parity      ; Jump if Not Parity (PF = 0)

mov al, 0b00001111  ; 4 ones → PF = 1 (even)
```

#### AF - Auxiliary Carry Flag (bit 4)
- Carry จาก bit 3 ไป bit 4
- ใช้ใน BCD (Binary Coded Decimal) arithmetic

```nasm
; AF ใช้กับ DAA/DAS instructions (BCD arithmetic)
mov al, 0x09    ; BCD 9
add al, 0x01    ; 9 + 1 = 0x0A, AF = 1 (carry from bit 3)
daa             ; Decimal Adjust Accumulator → AL = 0x10 (BCD 10)
```

### 5.3 การ Manipulate FLAGS

```nasm
; อ่าน FLAGS
pushf           ; push EFLAGS onto stack (32-bit)
pushfq          ; push RFLAGS onto stack (64-bit)
pop rax         ; อ่านค่า flags
and rax, 0x40   ; ดู ZF (bit 6)
shr rax, 6      ; shift ให้ ZF เป็น bit 0
; ถ้า rax = 1, ZF = 1

; เขียน FLAGS
pushfq          ; บันทึก flags ปัจจุบัน
pop rax
or rax, 0x100   ; set TF (trap flag, bit 8) สำหรับ single-step
push rax
popfq           ; โหลด flags ใหม่
```

---

## ส่วนที่ 6: Pointer Registers

### 6.1 RIP - Instruction Pointer

RIP ชี้ไปยัง instruction ถัดไปที่จะ execute (ไม่ใช่ที่กำลัง execute อยู่!):

```nasm
; ไม่สามารถอ่าน/เขียน RIP โดยตรง (ยกเว้นบางกรณี)
; แต่สามารถใช้ RIP-relative addressing:

section .data
    myvar dq 42

section .text
    global _start
_start:
    ; RIP-relative addressing (เป็น default ใน 64-bit NASM)
    mov rax, [rel myvar]    ; อ่าน myvar ผ่าน RIP-relative
    ; หรือ
    lea rax, [rel myvar]    ; เอา address ของ myvar
    
    ; call/jmp เปลี่ยน RIP:
    call some_function  ; push return address, set RIP = some_function
    jmp somewhere       ; set RIP = somewhere
```

---

## ส่วนที่ 7: SIMD Registers (XMM/YMM/ZMM)

### 7.1 XMM Registers (128-bit, SSE)

XMM registers มี 16 ตัว (XMM0-XMM15) ใน 64-bit mode:

```
XMM register (128 bits):
┌───────────────────────────────────────────────────────┐
│                     XMM0 (128 bits)                   │
│  float3  │  float2  │  float1  │  float0              │  (4 x 32-bit float)
│     double1         │     double0                     │  (2 x 64-bit double)
│  int3    │  int2    │  int1    │  int0                │  (4 x 32-bit int)
│ b15│b14│b13│b12│b11│b10│b9│b8│b7│b6│b5│b4│b3│b2│b1│b0│  (16 x 8-bit byte)
└───────────────────────────────────────────────────────┘
```

### 7.2 YMM Registers (256-bit, AVX)

YMM เป็น extension ของ XMM:
```
YMM register (256 bits):
┌─────────────────────────────────────────────────────────────────────────────┐
│                          YMM0 (256 bits)                                    │
│  [XMM0 upper 128 bits]              │  [XMM0 lower 128 bits = XMM0]        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 ZMM Registers (512-bit, AVX-512)

```
ZMM register (512 bits):
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                               ZMM0 (512 bits)                                       │
│  [YMM0 upper 256 bits]                              │  [YMM0 lower 256 bits = YMM0] │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.4 SSE/AVX Calling Conventions

```
Linux x86-64 (System V AMD64 ABI):
- XMM0-XMM7: ใช้ส่ง floating-point parameters (caller-saved)
- XMM0, XMM1: ใช้ return floating-point values
- XMM8-XMM15: caller-saved (ไม่ต้อง preserve)
- YMM0-YMM15: ถ้าใช้ AVX, upper 128 bits ต้อง preserve (ระวัง AVX-SSE transition penalty)
```

---

## ส่วนที่ 8: Linux x86-64 Calling Convention

### 8.1 Register Usage Summary

```
Register   | Role                          | Caller/Callee Saved
-----------|-------------------------------|--------------------
RAX        | Return value, temp            | Caller-saved
RBX        | General, base pointer         | Callee-saved
RCX        | 4th argument, counter         | Caller-saved
RDX        | 3rd argument, temp            | Caller-saved
RSI        | 2nd argument                  | Caller-saved
RDI        | 1st argument                  | Caller-saved
RBP        | Frame pointer                 | Callee-saved
RSP        | Stack pointer                 | Callee-saved (always)
R8         | 5th argument                  | Caller-saved
R9         | 6th argument                  | Caller-saved
R10        | Temporary, syscall#           | Caller-saved
R11        | Temporary, flags-save         | Caller-saved
R12        | General purpose               | Callee-saved
R13        | General purpose               | Callee-saved
R14        | General purpose               | Callee-saved
R15        | General purpose               | Callee-saved
XMM0-XMM7 | FP args / return values       | Caller-saved
XMM8-XMM15| Temporary                     | Caller-saved
```

**Caller-saved**: ถ้าคุณเรียก function อื่น registers เหล่านี้อาจเปลี่ยนค่า - คุณต้อง save เองถ้าต้องการ
**Callee-saved**: ถ้า function ของคุณใช้ registers เหล่านี้ คุณต้อง save และ restore ก่อน return

---

## Code Examples

### Example 1: Register Demonstration Program

```nasm
; file: register_demo.asm
; แสดงการใช้งาน registers ต่างๆ
; Compile: nasm -f elf64 register_demo.asm -o register_demo.o
; Link:    ld register_demo.o -o register_demo
; Run:     ./register_demo
; Expected output:
;   Register Demo Program
;   RAX = 42
;   RBX = 100
;   Sum = 142

section .data
    msg_header  db "Register Demo Program", 10, 0
    msg_rax     db "RAX = ", 0
    msg_rbx     db "RBX = ", 0
    msg_sum     db "Sum = ", 0
    newline     db 10, 0

section .bss
    num_buf resb 32     ; buffer สำหรับแปลงตัวเลข

section .text
    global _start

; ฟังก์ชัน: พิมพ์ string ผ่าน sys_write
; Input: RDI = address ของ string (null-terminated)
print_str:
    push rdi            ; บันทึก pointer เดิม
    push rbx
    
    mov rbx, rdi        ; RBX = pointer
    xor rcx, rcx        ; RCX = 0 (length counter)
    
.find_end:
    cmp byte [rbx + rcx], 0   ; ตรวจ null terminator
    je .found_end
    inc rcx                    ; length++
    jmp .find_end
    
.found_end:
    ; sys_write(1, string, length)
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; file descriptor: stdout
    pop rbx             ; คืน RBX ที่ push ไว้ (แต่ต้อง adjust เพราะ push rdi)
    ; แก้ไข: ใช้วิธีที่ถูกต้องกว่า
    pop rdi             ; คืน string pointer
    push rdi
    push rbx
    mov rsi, rdi        ; RSI = string address
    ; rcx ยังมีค่า length อยู่
    mov rdx, rcx        ; RDX = length
    syscall
    
    pop rbx
    pop rdi
    ret

; ฟังก์ชัน: แปลง integer เป็น string
; Input: RAX = number, RDI = buffer address
; Output: RDI = pointer to string, RCX = length
int_to_str:
    push rbx
    push r8
    
    mov rbx, rdi        ; RBX = buffer end
    add rbx, 30         ; ชี้ไปที่ท้าย buffer
    mov byte [rbx], 0   ; null terminator
    dec rbx
    
    mov r8, 10          ; หาร 10
    
    test rax, rax       ; ตรวจสอบว่า 0 หรือไม่
    jnz .convert_loop
    mov byte [rbx], '0' ; ถ้า 0 แค่ใส่ '0'
    jmp .done
    
.convert_loop:
    test rax, rax
    jz .done
    
    xor rdx, rdx        ; clear RDX สำหรับ div
    div r8              ; RAX /= 10, RDX = remainder
    add dl, '0'         ; แปลง digit เป็น ASCII
    mov [rbx], dl       ; เก็บ digit
    dec rbx
    jmp .convert_loop
    
.done:
    inc rbx             ; ชี้ไปยัง first digit
    pop r8
    pop rbx
    
    mov rdi, rbx        ; ส่ง pointer กลับ
    ret

_start:
    ; พิมพ์ header
    lea rdi, [rel msg_header]
    call print_str
    
    ; ตั้งค่า registers
    mov rax, 42         ; RAX = 42
    mov rbx, 100        ; RBX = 100
    
    ; พิมพ์ "RAX = 42"
    lea rdi, [rel msg_rax]
    call print_str
    
    ; แปลง RAX = 42 เป็น string และพิมพ์
    push rax            ; บันทึก RAX
    lea rdi, [rel num_buf]
    call int_to_str     ; RDI = address ของ string result
    call print_str
    
    lea rdi, [rel newline]
    call print_str
    pop rax             ; คืนค่า RAX
    
    ; พิมพ์ "RBX = 100"
    lea rdi, [rel msg_rbx]
    call print_str
    
    push rax
    mov rax, rbx        ; ย้าย RBX ไป RAX สำหรับ conversion
    lea rdi, [rel num_buf]
    call int_to_str
    call print_str
    lea rdi, [rel newline]
    call print_str
    pop rax
    
    ; คำนวณ sum
    add rax, rbx        ; RAX = RAX + RBX = 42 + 100 = 142
    push rax
    
    ; พิมพ์ "Sum = 142"
    lea rdi, [rel msg_sum]
    call print_str
    
    lea rdi, [rel num_buf]
    call int_to_str
    call print_str
    lea rdi, [rel newline]
    call print_str
    pop rax
    
    ; Exit
    mov rax, 60         ; syscall: exit
    xor rdi, rdi        ; exit code 0
    syscall
```

**การ Compile และ Run:**
```bash
nasm -f elf64 register_demo.asm -o register_demo.o
ld register_demo.o -o register_demo
./register_demo
```

**Expected Output:**
```
Register Demo Program
RAX = 42
RBX = 100
Sum = 142
```

---

### Example 2: FLAGS Register Demonstration

```nasm
; file: flags_demo.asm
; แสดงการทำงานของ FLAGS register
; Compile: nasm -f elf64 flags_demo.asm -o flags_demo.o
; Link:    ld flags_demo.o -o flags_demo
; Run:     ./flags_demo
; Expected output:
;   FLAGS Demo
;   CF test: Carry detected!
;   ZF test: Zero detected!
;   SF test: Negative detected!
;   OF test: Overflow detected!

section .data
    msg_header  db "FLAGS Demo", 10, 0
    msg_cf      db "CF test: Carry detected!", 10, 0
    msg_cf_no   db "CF test: No carry", 10, 0
    msg_zf      db "ZF test: Zero detected!", 10, 0
    msg_zf_no   db "ZF test: Non-zero", 10, 0
    msg_sf      db "SF test: Negative detected!", 10, 0
    msg_sf_no   db "SF test: Positive", 10, 0
    msg_of      db "OF test: Overflow detected!", 10, 0
    msg_of_no   db "OF test: No overflow", 10, 0

section .text
    global _start

; ฟังก์ชัน helper: พิมพ์ string
; Input: RSI = address, RDX = length
print_msg:
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    syscall
    ret

_start:
    ; พิมพ์ header
    mov rsi, msg_header
    mov rdx, 11         ; "FLAGS Demo\n" = 11 chars
    call print_msg
    
    ; ===== Test CF (Carry Flag) =====
    mov al, 0xFF        ; AL = 255 (max unsigned 8-bit)
    add al, 1           ; 255 + 1 = 256 → overflow! CF = 1
    
    jc .cf_set          ; JC = Jump if Carry (CF = 1)
    mov rsi, msg_cf_no
    mov rdx, 18
    call print_msg
    jmp .test_zf
.cf_set:
    mov rsi, msg_cf
    mov rdx, 25
    call print_msg
    
    ; ===== Test ZF (Zero Flag) =====
.test_zf:
    mov eax, 100
    sub eax, 100        ; 100 - 100 = 0, ZF = 1
    
    jz .zf_set          ; JZ = Jump if Zero (ZF = 1)
    mov rsi, msg_zf_no
    mov rdx, 18
    call print_msg
    jmp .test_sf
.zf_set:
    mov rsi, msg_zf
    mov rdx, 24
    call print_msg
    
    ; ===== Test SF (Sign Flag) =====
.test_sf:
    mov eax, 5
    sub eax, 10         ; 5 - 10 = -5, SF = 1 (negative result)
    
    js .sf_set          ; JS = Jump if Sign (SF = 1)
    mov rsi, msg_sf_no
    mov rdx, 18
    call print_msg
    jmp .test_of
.sf_set:
    mov rsi, msg_sf
    mov rdx, 27
    call print_msg
    
    ; ===== Test OF (Overflow Flag) =====
.test_of:
    mov al, 127         ; AL = 0x7F (max positive signed 8-bit)
    add al, 1           ; 127 + 1 = 128 = -128 ใน signed! OF = 1
    
    jo .of_set          ; JO = Jump if Overflow (OF = 1)
    mov rsi, msg_of_no
    mov rdx, 20
    call print_msg
    jmp .done
.of_set:
    mov rsi, msg_of
    mov rdx, 28
    call print_msg
    
.done:
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และ Run:**
```bash
nasm -f elf64 flags_demo.asm -o flags_demo.o
ld flags_demo.o -o flags_demo
./flags_demo
```

**Expected Output:**
```
FLAGS Demo
CF test: Carry detected!
ZF test: Zero detected!
SF test: Negative detected!
OF test: Overflow detected!
```

---

### Example 3: Stack Frame และ Register Convention

```nasm
; file: stack_frame.asm
; แสดง stack frame และ calling convention
; Compile: nasm -f elf64 stack_frame.asm -o stack_frame.o
; Link:    ld stack_frame.o -o stack_frame
; Run:     ./stack_frame
; Expected output:
;   sum(1, 2, 3) = 6
;   max(10, 20) = 20
;   factorial(5) = 120

section .data
    newline     db 10

section .bss
    result_buf  resb 32

section .text
    global _start

; ============================================
; ฟังก์ชัน sum: บวกสาม integers
; Input:  RDI = a, RSI = b, RDX = c
; Output: RAX = a + b + c
; ============================================
sum:
    ; Standard prologue
    push rbp
    mov rbp, rsp
    
    ; ไม่ต้องใช้ callee-saved registers ที่นี่
    ; parameters อยู่ใน RDI, RSI, RDX
    
    mov rax, rdi        ; RAX = a
    add rax, rsi        ; RAX += b
    add rax, rdx        ; RAX += c
    
    ; Standard epilogue
    pop rbp
    ret

; ============================================
; ฟังก์ชัน max: หาค่ามากกว่า
; Input:  RDI = a, RSI = b
; Output: RAX = max(a, b)
; ============================================
max:
    push rbp
    mov rbp, rsp
    
    mov rax, rdi        ; RAX = a (assume a is max)
    cmp rdi, rsi        ; เปรียบเทียบ a กับ b
    jge .done           ; ถ้า a >= b ไปที่ done
    mov rax, rsi        ; ถ้า b > a, RAX = b
    
.done:
    pop rbp
    ret

; ============================================
; ฟังก์ชัน factorial: คำนวณ n!
; Input:  RDI = n
; Output: RAX = n!
; ============================================
factorial:
    push rbp
    mov rbp, rsp
    push rbx            ; callee-saved! เราจะใช้ RBX
    
    mov rbx, rdi        ; RBX = n (ใช้ callee-saved เพราะจะ call ตัวเอง)
    
    cmp rbx, 1          ; base case: n <= 1
    jle .base_case
    
    ; recursive case: n! = n * (n-1)!
    lea rdi, [rbx-1]    ; RDI = n-1
    call factorial      ; RAX = (n-1)!
    imul rax, rbx       ; RAX = n * (n-1)!
    jmp .done
    
.base_case:
    mov rax, 1          ; 0! = 1! = 1
    
.done:
    pop rbx             ; restore callee-saved
    pop rbp
    ret

; ============================================
; ฟังก์ชัน print_number: พิมพ์ integer
; Input: RAX = number to print
; ============================================
print_number:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13
    
    mov r12, rax        ; บันทึก number
    
    ; แปลง integer เป็น string
    lea r13, [rel result_buf]
    add r13, 30         ; ชี้ไปท้าย buffer
    mov byte [r13], 0   ; null terminator
    
    mov rbx, r12        ; RBX = number
    test rbx, rbx
    jnz .convert
    mov byte [r13-1], '0'
    dec r13
    jmp .print
    
.convert:
    test rbx, rbx
    jz .print
    
    xor rdx, rdx
    mov rax, rbx
    mov rcx, 10
    div rcx             ; RAX = quotient, RDX = remainder
    mov rbx, rax
    add dl, '0'
    dec r13
    mov [r13], dl
    jmp .convert
    
.print:
    ; หาความยาว string
    mov rsi, r13
    mov rdx, 0
.count:
    cmp byte [r13 + rdx], 0
    je .do_print
    inc rdx
    jmp .count
    
.do_print:
    mov rax, 1
    mov rdi, 1
    syscall
    
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret

; ============================================
; ฟังก์ชัน print_str: พิมพ์ string
; Input: RSI = address, RDX = length
; ============================================
print_str:
    mov rax, 1
    mov rdi, 1
    syscall
    ret

_start:
    ; Test sum(1, 2, 3)
    mov rdi, 1          ; a = 1
    mov rsi, 2          ; b = 2
    mov rdx, 3          ; c = 3
    call sum            ; RAX = 6
    
    push rax            ; บันทึกผลลัพธ์
    mov rsi, sum_msg
    mov rdx, sum_msg_len
    call print_str
    pop rax
    call print_number
    
    ; พิมพ์ newline
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall
    
    ; Test max(10, 20)
    mov rdi, 10
    mov rsi, 20
    call max            ; RAX = 20
    
    push rax
    mov rsi, max_msg
    mov rdx, max_msg_len
    call print_str
    pop rax
    call print_number
    
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall
    
    ; Test factorial(5)
    mov rdi, 5
    call factorial      ; RAX = 120
    
    push rax
    mov rsi, fact_msg
    mov rdx, fact_msg_len
    call print_str
    pop rax
    call print_number
    
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall
    
    ; Exit
    mov rax, 60
    xor rdi, rdi
    syscall

section .data
    sum_msg     db "sum(1, 2, 3) = "
    sum_msg_len equ $ - sum_msg
    max_msg     db "max(10, 20) = "
    max_msg_len equ $ - max_msg
    fact_msg    db "factorial(5) = "
    fact_msg_len equ $ - fact_msg
```

**การ Compile และ Run:**
```bash
nasm -f elf64 stack_frame.asm -o stack_frame.o
ld stack_frame.o -o stack_frame
./stack_frame
```

**Expected Output:**
```
sum(1, 2, 3) = 6
max(10, 20) = 20
factorial(5) = 120
```

---

### Example 4: Register Aliasing Visualization

```nasm
; file: register_aliasing.asm
; แสดงการทำงานของ register aliasing
; Compile: nasm -f elf64 register_aliasing.asm -o register_aliasing.o
; Link:    ld register_aliasing.o -o register_aliasing
; Expected output:
;   RAX aliasing test:
;   After mov rax, 0x123456789ABCDEF0:
;   AL  = 0xF0
;   AH  = 0xDE
;   AX  = 0xDEF0
;   EAX = 0x9ABCDEF0
;   RAX = 0x123456789ABCDEF0
;   EAX write zeros upper 32 bits!

section .data
    header_msg  db "RAX aliasing test:", 10, 0
    al_msg      db "AL  = 0xF0", 10, 0
    ah_msg      db "AH  = 0xDE", 10, 0
    ax_msg      db "AX  = 0xDEF0", 10, 0
    eax_msg     db "EAX = 0x9ABCDEF0", 10, 0
    rax_msg     db "RAX = 0x123456789ABCDEF0", 10, 0
    eax_zero_msg db "EAX write zeros upper 32 bits!", 10, 0

section .text
    global _start

; พิมพ์ null-terminated string ใน RDI
print_string:
    push rdi
    push rcx
    mov rcx, 0
.len_loop:
    cmp byte [rdi + rcx], 0
    je .do_write
    inc rcx
    jmp .len_loop
.do_write:
    mov rax, 1
    mov rsi, rdi
    mov rdx, rcx
    mov rdi, 1
    pop rcx
    pop rdi
    push rdi
    push rcx
    syscall
    pop rcx
    pop rdi
    ret

_start:
    ; พิมพ์ header
    lea rdi, [rel header_msg]
    call print_string
    
    ; ตั้งค่า RAX ด้วยค่าที่แต่ละส่วนชัดเจน
    mov rax, 0x123456789ABCDEF0
    ; ตอนนี้:
    ; RAX = 0x123456789ABCDEF0
    ; EAX = 0x9ABCDEF0 (32 bits ล่าง)
    ; AX  = 0xDEF0   (16 bits ล่าง)
    ; AH  = 0xDE     (bits 15-8)
    ; AL  = 0xF0     (bits 7-0)
    
    ; ตรวจสอบโดยพิมพ์ข้อความที่เตรียมไว้
    ; (ในโปรแกรมจริงจะต้องแปลงค่าเป็น hex string)
    lea rdi, [rel al_msg]
    call print_string
    
    lea rdi, [rel ah_msg]
    call print_string
    
    lea rdi, [rel ax_msg]
    call print_string
    
    lea rdi, [rel eax_msg]
    call print_string
    
    lea rdi, [rel rax_msg]
    call print_string
    
    ; แสดง side effect ของการเขียน EAX:
    ; เมื่อเขียน EAX, bits 63-32 ของ RAX จะถูก zero!
    mov rax, 0x123456789ABCDEF0   ; RAX มีค่า 64-bit
    mov eax, 0x12345678           ; เขียนแค่ 32 bits ล่าง
    ; ตอนนี้ RAX = 0x0000000012345678 ไม่ใช่ 0x123456789ABCDEF0!
    ; bits 63-32 ถูก zero automatically
    
    lea rdi, [rel eax_zero_msg]
    call print_string
    
    ; ===== แสดงว่า AX/AH/AL ไม่ zero upper bits =====
    mov rax, 0x1111111111111111   ; ตั้งค่าทุก bit
    mov ax, 0x2222                ; เขียนแค่ 16 bits ล่าง
    ; RAX = 0x1111111111112222 (upper bits ไม่เปลี่ยน!)
    ; แต่ EAX write จะ zero upper 32 bits
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และ Run:**
```bash
nasm -f elf64 register_aliasing.asm -o register_aliasing.o
ld register_aliasing.o -o register_aliasing
./register_aliasing
```

**Expected Output:**
```
RAX aliasing test:
AL  = 0xF0
AH  = 0xDE
AX  = 0xDEF0
EAX = 0x9ABCDEF0
RAX = 0x123456789ABCDEF0
EAX write zeros upper 32 bits!
```

---

### Example 5: R8-R15 Extended Registers

```nasm
; file: extended_regs.asm
; แสดงการใช้ R8-R15 registers
; Compile: nasm -f elf64 extended_regs.asm -o extended_regs.o
; Link:    ld extended_regs.o -o extended_regs
; Expected output:
;   Extended Register Demo
;   R8=10 R9=20 R10=30 R11=40
;   R12=50 R13=60 R14=70 R15=80
;   Sum of all = 360

section .data
    msg_start   db "Extended Register Demo", 10, 0
    msg_r8_11   db "R8=10 R9=20 R10=30 R11=40", 10, 0
    msg_r12_15  db "R12=50 R13=60 R14=70 R15=80", 10, 0
    msg_sum_pre db "Sum of all = ", 0
    newline     db 10

section .bss
    num_buf resb 32

section .text
    global _start

; ============================================
; print_str: พิมพ์ null-terminated string
; Input: RDI = string address
; ============================================
print_str:
    push r12            ; ใช้ callee-saved register
    push r13
    mov r12, rdi        ; R12 = string pointer
    xor r13, r13        ; R13 = length = 0
.find_len:
    cmp byte [r12 + r13], 0
    je .write
    inc r13
    jmp .find_len
.write:
    mov rax, 1
    mov rdi, 1
    mov rsi, r12
    mov rdx, r13
    syscall
    pop r13
    pop r12
    ret

; ============================================
; print_int: พิมพ์ integer
; Input: RAX = number
; ============================================
print_int:
    push rbx
    push r12
    
    lea r12, [rel num_buf]
    add r12, 30
    mov byte [r12], 0
    
    test rax, rax
    jnz .convert
    dec r12
    mov byte [r12], '0'
    jmp .do_print
    
.convert:
    mov rbx, 10
.loop:
    test rax, rax
    jz .do_print
    xor rdx, rdx
    div rbx
    add dl, '0'
    dec r12
    mov [r12], dl
    jmp .loop
    
.do_print:
    ; หา length
    mov rsi, r12
    xor rdx, rdx
.len:
    cmp byte [r12 + rdx], 0
    je .print
    inc rdx
    jmp .len
.print:
    mov rax, 1
    mov rdi, 1
    syscall
    
    pop r12
    pop rbx
    ret

_start:
    ; ต้อง push callee-saved registers ที่จะใช้
    push r12
    push r13
    push r14
    push r15
    
    ; พิมพ์ header
    lea rdi, [rel msg_start]
    call print_str
    
    ; ตั้งค่า R8-R15
    mov r8,  10         ; R8  = 10
    mov r9,  20         ; R9  = 20
    mov r10, 30         ; R10 = 30
    mov r11, 40         ; R11 = 40
    mov r12, 50         ; R12 = 50 (callee-saved แต่ main ใช้ได้)
    mov r13, 60         ; R13 = 60
    mov r14, 70         ; R14 = 70
    mov r15, 80         ; R15 = 80
    
    ; พิมพ์ R8-R11
    lea rdi, [rel msg_r8_11]
    call print_str
    
    ; พิมพ์ R12-R15
    lea rdi, [rel msg_r12_15]
    call print_str
    
    ; คำนวณผลรวมทุก registers
    ; R8 + R9 + R10 + R11 + R12 + R13 + R14 + R15
    mov rax, r8
    add rax, r9
    add rax, r10
    add rax, r11
    add rax, r12
    add rax, r13
    add rax, r14
    add rax, r15
    ; RAX = 10+20+30+40+50+60+70+80 = 360
    
    push rax            ; บันทึก sum
    
    ; พิมพ์ "Sum of all = "
    lea rdi, [rel msg_sum_pre]
    call print_str
    
    pop rax
    call print_int      ; พิมพ์ 360
    
    ; พิมพ์ newline
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall
    
    pop r15
    pop r14
    pop r13
    pop r12
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และ Run:**
```bash
nasm -f elf64 extended_regs.asm -o extended_regs.o
ld extended_regs.o -o extended_regs
./extended_regs
```

**Expected Output:**
```
Extended Register Demo
R8=10 R9=20 R10=30 R11=40
R12=50 R13=60 R14=70 R15=80
Sum of all = 360
```

---

### Example 6: XMM Registers และ SSE Operations

```nasm
; file: xmm_demo.asm
; แสดง XMM registers สำหรับ floating-point operations
; Compile: nasm -f elf64 xmm_demo.asm -o xmm_demo.o
; Link:    ld xmm_demo.o -o xmm_demo
; Expected output:
;   XMM SIMD Demo
;   Float add complete
;   Integer SIMD complete

section .data
    ; ค่า float 32-bit สี่ตัว (4 x float packed)
    float_a     dd 1.0, 2.0, 3.0, 4.0
    float_b     dd 5.0, 6.0, 7.0, 8.0
    
    ; ค่า integer 32-bit สี่ตัว
    int_a       dd 10, 20, 30, 40
    int_b       dd 1,  2,  3,  4
    
    header_msg  db "XMM SIMD Demo", 10, 0
    float_msg   db "Float add complete", 10, 0
    int_msg     db "Integer SIMD complete", 10, 0

section .bss
    float_result    resd 4      ; 4 floats สำหรับผลลัพธ์
    int_result      resd 4      ; 4 ints สำหรับผลลัพธ์

section .text
    global _start

print_str:
    push r12
    push r13
    mov r12, rdi
    xor r13, r13
.loop:
    cmp byte [r12 + r13], 0
    je .write
    inc r13
    jmp .loop
.write:
    mov rax, 1
    mov rdi, 1
    mov rsi, r12
    mov rdx, r13
    syscall
    pop r13
    pop r12
    ret

_start:
    lea rdi, [rel header_msg]
    call print_str
    
    ; ===== XMM Float Operations (SSE) =====
    ; โหลด 4 floats เข้า XMM0 (128-bit = 4 x 32-bit float)
    movups xmm0, [rel float_a]    ; XMM0 = {4.0, 3.0, 2.0, 1.0}
    movups xmm1, [rel float_b]    ; XMM1 = {8.0, 7.0, 6.0, 5.0}
    
    ; บวก 4 floats พร้อมกัน (SIMD!)
    addps xmm0, xmm1              ; XMM0 = {12.0, 10.0, 8.0, 6.0}
    
    ; เก็บผลลัพธ์
    movups [rel float_result], xmm0
    ; float_result = {6.0, 8.0, 10.0, 12.0}
    
    lea rdi, [rel float_msg]
    call print_str
    
    ; ===== XMM Integer Operations (SSE2) =====
    ; โหลด 4 integers เข้า XMM0
    movdqu xmm0, [rel int_a]      ; XMM0 = {40, 30, 20, 10}
    movdqu xmm1, [rel int_b]      ; XMM1 = {4, 3, 2, 1}
    
    ; บวก 4 integers พร้อมกัน
    paddd xmm0, xmm1              ; XMM0 = {44, 33, 22, 11}
    
    ; เก็บผลลัพธ์
    movdqu [rel int_result], xmm0
    ; int_result = {11, 22, 33, 44}
    
    lea rdi, [rel int_msg]
    call print_str
    
    ; คืนค่า FPU state (vzeroupper ป้องกัน AVX/SSE penalty)
    ; vzeroupper    ; uncomment ถ้าใช้ AVX
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และ Run:**
```bash
nasm -f elf64 xmm_demo.asm -o xmm_demo.o
ld xmm_demo.o -o xmm_demo
./xmm_demo
```

**Expected Output:**
```
XMM SIMD Demo
Float add complete
Integer SIMD complete
```

---

## ส่วนที่ 9: Register File Visualization

### 9.1 ภาพรวม Complete Register File (64-bit x86)

```
┌─────────────────────────────────────────────────────────────────┐
│                  x86-64 Register File                           │
├─────────────────────────────────────────────────────────────────┤
│  General Purpose Registers (Integer)                            │
│                                                                 │
│  63        32 31     16 15   8 7    0                           │
│  ┌──────────┬─────────┬───────┬──────┐                          │
│  │  Hi RAX  │   EAX   │  AH   │  AL  │  ← Accumulator          │
│  │  Hi RBX  │   EBX   │  BH   │  BL  │  ← Base                 │
│  │  Hi RCX  │   ECX   │  CH   │  CL  │  ← Counter              │
│  │  Hi RDX  │   EDX   │  DH   │  DL  │  ← Data                 │
│  │  Hi RSI  │   ESI   │   SIL (8-bit)│  ← Source Index         │
│  │  Hi RDI  │   EDI   │   DIL (8-bit)│  ← Dest Index           │
│  │  Hi RSP  │   ESP   │   SPL (8-bit)│  ← Stack Pointer        │
│  │  Hi RBP  │   EBP   │   BPL (8-bit)│  ← Base Pointer         │
│  │  Hi R8   │   R8D   │  R8W  │  R8B │  ← (64-bit only)        │
│  │  Hi R9   │   R9D   │  R9W  │  R9B │                          │
│  │  Hi R10  │   R10D  │  R10W │ R10B │                          │
│  │  Hi R11  │   R11D  │  R11W │ R11B │                          │
│  │  Hi R12  │   R12D  │  R12W │ R12B │                          │
│  │  Hi R13  │   R13D  │  R13W │ R13B │                          │
│  │  Hi R14  │   R14D  │  R14W │ R14B │                          │
│  │  Hi R15  │   R15D  │  R15W │ R15B │                          │
│  └──────────┴─────────┴───────┴──────┘                          │
├─────────────────────────────────────────────────────────────────┤
│  Special Registers                                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    RIP (Instruction Pointer)              │  │
│  │                    RFLAGS (Flags Register)                │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  Segment Registers (16-bit each)                                │
│  CS  DS  ES  FS  GS  SS                                         │
├─────────────────────────────────────────────────────────────────┤
│  SIMD Registers                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           ZMM0 (512-bit)                                │    │
│  │  [Upper 256]  │  YMM0 (256-bit)                        │    │
│  │               │  [Upper 128] │ XMM0 (128-bit)          │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ZMM0-ZMM31, YMM0-YMM31, XMM0-XMM31 (AVX-512)                 │
│  YMM0-YMM15, XMM0-XMM15 (AVX2/AVX)                             │
│  XMM0-XMM15 (SSE)                                               │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 RFLAGS Register Layout

```
RFLAGS (64-bit):
Bit: 63   22 21 20 19 18 17 16 15 14 13 12 11 10 9  8  7  6  5  4  3  2  1  0
     ┌────┐  ┌─┬─┬─┬─┬─┬─┬─┬─┬──┬─┬──┬─┬──┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
     │ 0  │  │I│V│V│A│V│R│0│N│IO│O│D│I│T│S│Z│0│A│0│P│1│C│
     └────┘  │D│I│I│C│M│F│ │T│PL│F│F│F│F│F│F│ │F│ │F│ │F│
             └─┴─┴─┴─┴─┴─┴─┴─┴──┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
                                                   ↑     ↑
                                                   ZF    CF
```

---

## Common Mistakes and Pitfalls

### ข้อผิดพลาดที่พบบ่อย

#### 1. ลืม clear RDX ก่อน Division

```nasm
; ผิด! - RDX อาจมีขยะ
mov rax, 100
div rbx         ; DIVISION ERROR ถ้า RDX ไม่ใช่ 0!

; ถูก - ต้อง clear RDX ก่อน unsigned division
mov rax, 100
xor rdx, rdx    ; RDX = 0
div rbx

; ถูก - สำหรับ signed division ใช้ cdq/cqo แทน
mov eax, -100
cdq             ; sign-extend EAX → EDX:EAX
idiv ebx

; 64-bit signed division
mov rax, -100
cqo             ; sign-extend RAX → RDX:RAX
idiv rbx
```

#### 2. ไม่เข้าใจ EAX Zero-Extension

```nasm
; สับสน!
mov rax, 0xFFFFFFFFFFFFFFFF  ; RAX มีทุก bit เป็น 1
mov eax, 0                    ; ตั้งใจจะ clear แค่ lower 32 bits
; แต่ผลคือ RAX = 0x0000000000000000 !!!
; เพราะการเขียน EAX จะ zero upper 32 bits ของ RAX เสมอ

; ถ้าต้องการ clear แค่ lower 16 bits โดยไม่เปลี่ยน upper:
mov ax, 0       ; zero bits 15-0, bits 63-16 ไม่เปลี่ยน
```

#### 3. ลืม Align Stack ก่อน CALL

```nasm
; System V ABI กำหนดว่า RSP ต้องเป็น 16-byte aligned ก่อน CALL
; CALL จะ push return address (8 bytes) ทำให้ RSP ลดลง 8

_start:
    ; RSP เริ่มต้น aligned (ปกติเป็น 16-byte aligned)
    ; CALL push 8 bytes → RSP ไม่ aligned
    
    ; ถ้าจะเรียก C functions หรือ library ต้อง align:
    sub rsp, 8      ; จัดให้ RSP เป็น 16-byte aligned
    call some_func
    add rsp, 8
    
    ; หรือ push dummy value:
    push rax        ; ทำให้ RSP - 8 → aligned
    call some_func
    pop rax
```

#### 4. การใช้ AH/BH/CH/DH ผิด

```nasm
; ปัญหา: AH/BH/CH/DH ไม่ accessible เมื่อมี REX prefix!
; ถ้าใช้ R8-R15, RSP, RBP, RSI, RDI พร้อมกับ AH/BH/CH/DH
; NASM จะ error

; ผิด (ถ้าใช้ R8 หรือ registers อื่นที่ต้อง REX):
; mov ah, r8b     ; Error! ไม่สามารถใช้ AH กับ REX prefix

; ถูก - ใช้ AL แทน AH หรือ shift:
mov al, [some_data]    ; อ่าน byte
shl eax, 8            ; shift ไปยัง AH position

; หรือใช้ movzx:
movzx eax, byte [some_data]   ; zero-extend byte to 32-bit
```

#### 5. Direction Flag (DF) ค้างอยู่

```nasm
; ปัญหา: ใช้ STD แล้วลืม CLD
my_function:
    std             ; ตั้ง DF สำหรับ backward string ops
    rep movsb
    ; ลืม CLD ก่อน ret!
    ret             ; DF = 1 → function ที่เรียกอาจทำงานผิดพลาด!

; ถูก:
my_function:
    push rflags_save
    std
    rep movsb
    cld             ; reset DF ก่อน return
    ret

; หรือ save/restore:
my_function:
    pushfq          ; บันทึก flags รวมถึง DF
    std
    rep movsb
    popfq           ; คืนค่า flags เดิม
    ret
```

#### 6. การใช้ LOOP instruction กับ function calls

```nasm
; ปัญหา: LOOP ใช้ RCX เป็น counter
; ถ้า call function ภายใน loop และ function เปลี่ยน RCX จะเสีย!

; ผิด:
    mov rcx, 10
loop_bad:
    call some_function  ; some_function อาจเปลี่ยน RCX!
    loop loop_bad       ; RCX อาจไม่ถูกต้อง

; ถูก: save/restore RCX หรือใช้วิธีอื่น
    mov rcx, 10
loop_good:
    push rcx            ; บันทึก counter
    call some_function
    pop rcx             ; คืนค่า counter
    dec rcx
    jnz loop_good
```

#### 7. Caller vs Callee Save Confusion

```nasm
; ผิด: ใช้ caller-saved register หลัง CALL โดยไม่ save ก่อน
my_func:
    mov rax, 42         ; ตั้งค่า rax
    call other_func     ; other_func อาจเปลี่ยน RAX!
    ; rax ตอนนี้อาจไม่ใช่ 42 อีกต่อไป

; ถูก: ถ้าต้องการค่า rax หลัง call ต้อง save ก่อน
my_func:
    mov rax, 42
    push rax            ; save ก่อน call
    call other_func
    pop rax             ; restore หลัง call
    ; rax = 42 แน่นอน

; หรือใช้ callee-saved register:
my_func:
    push rbx
    mov rbx, 42         ; ใช้ RBX แทน (callee-saved)
    call other_func     ; other_func ต้อง preserve RBX
    ; rbx ยังเป็น 42 (guaranteed)
    pop rbx
```

---

## Advanced Techniques

### เทคนิคขั้นสูง

#### 1. Register Renaming และ Out-of-Order Execution

CPU ทันสมัย (Intel/AMD) มีกลไก Register Renaming ที่ทำให้การเขียน instruction สามารถ optimize ได้:

```nasm
; CPU จะ rename physical registers เพื่อ break dependency chains
; ตัวอย่าง: ทำ xor reg,reg เพื่อ clear register (ดีกว่า mov reg,0)

; mov eax, 0     ; 5 bytes, ต้อง read EAX ก่อน (dependency!)
xor eax, eax    ; 2 bytes, break dependency chain, CPU รู้ว่า result = 0
                ; CPU เห็นว่า xor reg,reg → result always 0 → dependency break

; เทคนิค: ใช้ xor สำหรับ register clearing
xor eax, eax    ; RAX = 0 (ดีที่สุด)
xor edi, edi    ; RDI = 0
xor esi, esi    ; RSI = 0
```

#### 2. Partial Register Updates และ False Dependencies

```nasm
; ปัญหา False Dependency กับ partial register writes:
; (Intel CPUs pre-Ivy Bridge โดยเฉพาะ)

; Slow path: AH อาจสร้าง false dependency
movzx eax, byte [mem]   ; ดีกว่า mov al, [mem] เพราะ zero-extends

; การอ่าน/เขียน 8/16-bit sub-registers อาจสร้าง false dependency
; บน CPU บางรุ่น เพราะต้องรวม partial result กับ register เดิม

; ใช้ movzx แทน mov al เมื่อไม่ต้องการ upper bits:
movzx eax, byte [memory]     ; clear upper, no false dep
movzx eax, word [memory]     ; clear upper, no false dep
mov al, [memory]             ; อาจมี false dep บน old CPUs

; Intel เรียกว่า "partial register stall" บน older CPUs
```

#### 3. LAHF/SAHF - Load/Store AH from/to FLAGS

```nasm
; LAHF: Load AH ← FLAGS (bits 7,6,4,2,0)
; SAHF: Store FLAGS ← AH

; ใช้ประโยชน์ใน: copy flags เป็น byte value
lahf            ; AH = SF:ZF:0:AF:0:PF:1:CF (bits จาก FLAGS)
mov [flags_save], ah    ; บันทึก flags

; Restore:
mov ah, [flags_save]
sahf            ; FLAGS ← AH

; ตัวอย่าง: เปรียบเทียบ 2 items แล้ว return flag ใน byte
compare_items:
    cmp rdi, rsi    ; ตั้ง flags
    lahf            ; AH มีค่า flags
    mov al, ah      ; AL = flags byte
    ret             ; return flags ใน AL
```

#### 4. การใช้ LEA สำหรับ Arithmetic

```nasm
; LEA ไม่ได้ใช้แค่สำหรับ load address!
; LEA คำนวณ address โดยไม่ touch FLAGS และ bypass memory

; คูณ 3:
lea rax, [rax + rax*2]    ; rax = rax + rax*2 = rax*3

; คูณ 5:
lea rax, [rax + rax*4]    ; rax = rax*5

; คูณ 9:
lea rax, [rax + rax*8]    ; rax = rax*9

; บวกและคูณ:
lea rax, [rbx + rcx*4 + 100]  ; rax = rbx + rcx*4 + 100

; เร็วกว่า:
imul rax, 3    ; ต้องใช้ memory operand หรือ immediate
; เพราะ LEA เป็น single-cycle ใน pipeline
```

#### 5. Zero/Sign Extension Techniques

```nasm
; Zero Extension:
movzx eax, al       ; zero-extend AL to EAX (bits 31-8 = 0)
movzx eax, ax       ; zero-extend AX to EAX (bits 31-16 = 0)
movzx rax, eax      ; 64-bit: ใช้ mov eax,eax แทนได้เพราะ auto zero-extend

; Sign Extension:
movsx eax, al       ; sign-extend AL to EAX
movsx eax, ax       ; sign-extend AX to EAX
movsx rax, eax      ; sign-extend EAX to RAX
movsxd rax, eax     ; sign-extend dword to qword (MOVSXD)

; CBW/CWDE/CDQE: extend AX → sign-extend
cbw                 ; AL → AX (sign-extend)
cwde                ; AX → EAX (sign-extend)
cdqe                ; EAX → RAX (sign-extend)

; CDQ/CQO: extend สำหรับ division
cdq                 ; sign-extend EAX → EDX:EAX (สำหรับ IDIV)
cqo                 ; sign-extend RAX → RDX:RAX (สำหรับ 64-bit IDIV)
```

#### 6. Rotating ผ่าน CF

```nasm
; RCL/RCR: Rotate through Carry
; ใช้สำหรับ multi-word shifts

; Example: shift 128-bit value (RDX:RAX) left 1 bit
shl rax, 1          ; shift RAX left, MSB → CF
rcl rdx, 1          ; shift RDX left, CF → LSB

; 128-bit right shift:
shr rdx, 1          ; shift RDX right, LSB → CF
rcr rax, 1          ; shift RAX right, CF → MSB
```

#### 7. CMOVcc - Conditional Move

```nasm
; CMOVcc: conditional move ไม่มี branch → ไม่มี branch misprediction

; ตัวอย่าง: absolute value
mov rax, rdi        ; rax = n
neg rax             ; rax = -n
cmovs rax, rdi      ; ถ้า SF=1 (ผลลัพธ์ neg เป็น negative หมายความว่า n เป็น negative)
                    ; rax = rdi (original n)
                    ; ถ้า SF=0 (n เป็น positive) rax = -n ← ผิด ต้องแก้ logic

; ถูกต้อง:
mov rax, rdi        ; rax = n
test rdi, rdi       ; ตั้ง flags ตาม n
jns .positive       ; ถ้า n >= 0 ข้ามไป
neg rax             ; rax = -n (เพื่อให้เป็น positive)
.positive:
; ใช้ CMOVcc:
mov rax, rdi
neg rdi             ; rdi = -n (ชั่วคราว)
cmovs rax, rdi      ; ถ้า n < 0 (SF หลัง neg), rax = -n
; แต่ logic ซับซ้อน ใช้ branch อาจชัดกว่าในกรณีนี้

; ตัวอย่างที่ดีกว่า: max(a, b)
mov rax, rdi        ; rax = a
cmp rdi, rsi        ; a - b
cmovl rax, rsi      ; ถ้า a < b, rax = b
; rax = max(a, b)
```

---

## Exercises (แบบฝึกหัด)

### Exercise 1: Register Exploration

**โจทย์**: เขียนโปรแกรมที่แสดง value ของทุก registers ใน hex format

**เงื่อนไข**:
- แสดง RAX, RBX, RCX, RDX, RSI, RDI, RSP, RBP
- แสดง R8-R15
- แสดง RFLAGS
- ใช้ format: "RAX = 0x0000000000000000"

**Hint**:
```nasm
; ใช้ pushfq เพื่ออ่าน RFLAGS
pushfq
pop rax
; rax มีค่า RFLAGS

; สำหรับ hex conversion:
; ใช้ nibble-by-nibble conversion
; nibble = 4 bits = 1 hex digit
; 64-bit = 16 nibbles

hex_chars db "0123456789ABCDEF"
; loop 16 ครั้ง
; ทำ (value >> 60) & 0xF เพื่อเอา hex digit แรก
; ใช้ index ไปยัง hex_chars
; shift left 4 bits แล้ว loop ต่อ
```

**โครงสร้างที่ต้องเขียน**:
```nasm
section .text
    global _start

; convert RAX to hex string ใน RDI
to_hex:
    ; TODO: แปลง RAX เป็น 16-character hex string ใน buffer [RDI]
    ret

_start:
    ; TODO: อ่านและแสดง registers ทุกตัว
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

### Exercise 2: FLAGS Manipulation

**โจทย์**: เขียนฟังก์ชัน `check_flags` ที่:
1. รับค่า integer ใน RDI
2. บวก integer นั้นกับ RSI
3. ตรวจสอบว่า: CF set? ZF set? SF set? OF set?
4. return ค่า bit mask ใน RAX โดย:
   - bit 0 = CF
   - bit 1 = ZF
   - bit 2 = SF
   - bit 3 = OF

**Expected behavior**:
```
check_flags(0xFF, 1) → CF=1, ZF=0, SF=0, OF=0 → return 0b0001 = 1
check_flags(0, 0)    → CF=0, ZF=1, SF=0, OF=0 → return 0b0010 = 2
check_flags(5, -10)  → CF=1, ZF=0, SF=1, OF=0 → return 0b0101 = 5
```

**Hint**:
```nasm
; หลัง add operation ใช้ setcc instructions:
; setc al    → AL = 1 ถ้า CF=1, else 0
; setz bl    → BL = 1 ถ้า ZF=1, else 0
; sets cl    → CL = 1 ถ้า SF=1, else 0
; seto dl    → DL = 1 ถ้า OF=1, else 0

; แล้ว combine เป็น mask:
; movzx rax, al    ; rax = CF
; movzx rbx, bl    ; rbx = ZF
; shl rbx, 1       ; rbx = ZF << 1
; or rax, rbx      ; rax |= ZF bit
; ...
```

---

### Exercise 3: Calling Convention Practice

**โจทย์**: เขียนฟังก์ชัน C-compatible ที่:
```c
long dot_product(long *a, long *b, int n);
// คำนวณ dot product: a[0]*b[0] + a[1]*b[1] + ... + a[n-1]*b[n-1]
```

**Calling convention**: Linux x86-64 (System V AMD64 ABI)
- RDI = pointer to array a
- RSI = pointer to array b
- RDX = n (number of elements)
- Return in RAX

**Constraints**:
- ต้อง preserve callee-saved registers ที่ใช้
- ต้อง handle n=0 correctly (return 0)

**Hint**:
```nasm
dot_product:
    push rbp
    mov rbp, rsp
    push rbx        ; บันทึก callee-saved ที่จะใช้
    
    xor rax, rax    ; result = 0
    test rdx, rdx   ; ตรวจ n = 0
    jz .done
    
    xor rcx, rcx    ; counter i = 0
.loop:
    ; TODO: คำนวณ a[i] * b[i] และบวกเข้า RAX
    ; ระวัง: imul ผลลัพธ์ 64-bit * 64-bit = 128-bit ใน RDX:RAX
    ; ใช้ imul rbx, [rdi + rcx*8] แทน (ผลลัพธ์ 64-bit เท่านั้น)
    
.done:
    pop rbx
    pop rbp
    ret
```

---

### Exercise 4: SIMD Register Basics

**โจทย์**: เขียนฟังก์ชัน `add_arrays` ที่บวก arrays สองชุดด้วย SSE:
```c
void add_arrays(float *dst, float *src_a, float *src_b, int count);
// dst[i] = src_a[i] + src_b[i] สำหรับ i = 0 ถึง count-1
```

**Requirements**:
- ต้องใช้ XMM registers และ addps instruction
- Process 4 floats ต่อ iteration (SSE = 128 bits = 4 floats)
- Handle count ที่ไม่ใช่ multiple of 4

**Hint**:
```nasm
; Loop หลัก: process 4 floats ต่อครั้ง
; movups xmm0, [rsi]    ; โหลด 4 floats จาก src_a
; movups xmm1, [rdx]    ; โหลด 4 floats จาก src_b
; addps xmm0, xmm1      ; บวก 4 floats พร้อมกัน
; movups [rdi], xmm0    ; เก็บผลลัพธ์

; ตรวจสอบ alignment: movups = unaligned, movaps = aligned (16-byte)
; aligned ต้องการ address ที่หาร 16 ลงตัว แต่เร็วกว่า
```

---

### Exercise 5: Register-Based State Machine

**โจทย์**: เขียน state machine ที่ parse string และนับจำนวน:
- `words`: ลำดับอักขระที่ไม่ใช่ whitespace
- `spaces`: จำนวน whitespace sequences
- `total_chars`: จำนวน characters ทั้งหมด

**สภาวะ (states)**:
- State 0: IN_SPACE (กำลังอ่าน whitespace หรือยังไม่เริ่ม)
- State 1: IN_WORD (กำลังอ่าน word)

**ใช้ registers สำหรับ state**:
- R12 = current state
- R13 = word count
- R14 = space count
- R15 = total char count

**Input**: RDI = string pointer

**Expected**:
```
"hello world foo" → words=3, spaces=2, chars=15
"  test  " → words=1, spaces=3, chars=8
```

---

## Summary

### สรุปสิ่งสำคัญที่เรียนใน Part 011

#### Register Categories:

| ประเภท | Registers | หมายเหตุ |
|--------|-----------|---------|
| General Purpose | RAX, RBX, RCX, RDX, RSI, RDI, RSP, RBP | ใช้งานทั่วไป |
| Extended (64-bit) | R8-R15 | เพิ่มใน x86-64 ต้อง REX prefix |
| Flags | RFLAGS | เก็บผลลัพธ์จาก operations |
| Instruction Pointer | RIP | ชี้ instruction ถัดไป |
| Segment | CS, DS, ES, FS, GS, SS | Memory segmentation |
| SIMD | XMM0-15, YMM0-15, ZMM0-31 | Vector/SIMD operations |

#### Register Aliasing:
- RAX (64) → EAX (32) → AX (16) → AH (8 high) / AL (8 low)
- การเขียน EAX จะ **zero upper 32 bits** ของ RAX อัตโนมัติ
- การเขียน AX/AH/AL **ไม่** zero upper bits

#### RFLAGS สำคัญ:
| Flag | Bit | เมื่อไหร่ Set |
|------|-----|--------------|
| CF | 0 | Unsigned overflow/borrow |
| PF | 2 | จำนวน 1-bits เป็นเลขคู่ |
| AF | 4 | BCD carry จาก nibble |
| ZF | 6 | ผลลัพธ์เป็น 0 |
| SF | 7 | ผลลัพธ์เป็น negative |
| OF | 11 | Signed overflow |
| DF | 10 | ควบคุม string direction |

#### Calling Convention (Linux x86-64):
- **Parameters**: RDI, RSI, RDX, RCX, R8, R9
- **Return**: RAX (integers), XMM0 (floats)
- **Caller-saved**: RAX, RCX, RDX, RSI, RDI, R8-R11
- **Callee-saved**: RBX, RBP, R12-R15

#### REX Prefix:
- 0x40-0x4F: 1-byte prefix ก่อน instruction ใน 64-bit mode
- REX.W=1: ใช้ 64-bit operand size
- REX.R/X/B=1: ขยาย register เป็น R8-R15

#### XMM/YMM/ZMM:
- XMM: 128-bit (SSE), 4 floats หรือ 2 doubles
- YMM: 256-bit (AVX), 8 floats หรือ 4 doubles
- ZMM: 512-bit (AVX-512), 16 floats หรือ 8 doubles

---

### Quick Reference Card

```
; General Purpose Register Usage (Linux x86-64)
; System Calls:
;   RAX = syscall number
;   RDI = arg1, RSI = arg2, RDX = arg3
;   R10 = arg4, R8 = arg5, R9 = arg6
;   Return value in RAX

; Common Syscall Numbers (Linux):
;   1  = write(fd, buf, count)
;   0  = read(fd, buf, count)
;   2  = open(filename, flags, mode)
;   3  = close(fd)
;   9  = mmap(...)
;   60 = exit(code)
;   231 = exit_group(code)

; Flag-setting Instructions:
;   ADD, SUB, AND, OR, XOR → CF, ZF, SF, OF, PF, AF
;   CMP, TEST              → CF, ZF, SF, OF, PF, AF (ไม่เปลี่ยนค่า dest)
;   INC, DEC               → ZF, SF, OF, PF, AF (ไม่เปลี่ยน CF!)
;   SHL, SHR, SAR          → CF, ZF, SF, OF, PF
;   MUL, IMUL              → CF, OF
;   DIV, IDIV              → undefined

; Conditional Jump Summary:
;   JE/JZ   ZF=1          เท่ากัน / เป็น 0
;   JNE/JNZ ZF=0          ไม่เท่ากัน / ไม่ใช่ 0
;   JG/JNLE ZF=0 AND SF=OF  > (signed)
;   JGE/JNL SF=OF           >= (signed)
;   JL/JNGE SF≠OF           < (signed)
;   JLE/JNG ZF=1 OR SF≠OF  <= (signed)
;   JA/JNBE CF=0 AND ZF=0  > (unsigned)
;   JAE/JNB CF=0            >= (unsigned)
;   JB/JNAE CF=1            < (unsigned)
;   JBE/JNA CF=1 OR ZF=1   <= (unsigned)
;   JC      CF=1           มี carry
;   JO      OF=1           มี overflow
;   JS      SF=1           เป็น negative
```

---

### ต่อไปใน Part 012

ใน Part ถัดไปเราจะเรียนเรื่อง:
- **Memory Addressing Modes** อย่างละเอียด
- Direct, indirect, indexed, scaled-index addressing
- RIP-relative addressing ใน 64-bit
- Memory operand size specifiers (BYTE, WORD, DWORD, QWORD)
- การทำงานกับ arrays และ structs ใน memory

---

*Part 011 เสร็จสมบูรณ์ - x86 Registers อย่างละเอียด*
*ไฟล์ถัดไป: part-012-memory-addressing.md*
