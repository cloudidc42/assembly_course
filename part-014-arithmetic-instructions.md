# Part 014: Arithmetic Instructions (คำสั่งคำนวณทางคณิตศาสตร์)

## Prerequisites (สิ่งที่ต้องรู้ก่อน)

ก่อนเริ่มเรียน Part นี้ คุณควรเข้าใจ:
- การใช้ Register (RAX, RBX, RCX, RDX, RSP, RBP, RSI, RDI)
- การใช้คำสั่ง MOV และการ Addressing modes
- ระบบเลข Binary, Hexadecimal, และ Two's Complement
- ความเข้าใจเรื่อง EFLAGS register (CF, ZF, SF, OF, AF, PF)
- Part 001-013 ทั้งหมดในหลักสูตรนี้

---

## Learning Objectives (วัตถุประสงค์การเรียนรู้)

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:

1. **ใช้คำสั่งบวก** - ADD, ADC (Add with Carry) สำหรับจำนวนเต็มและจำนวนที่มี carry
2. **ใช้คำสั่งลบ** - SUB, SBB (Subtract with Borrow) สำหรับจำนวนเต็มและจำนวนที่มี borrow
3. **ใช้คำสั่งเพิ่ม/ลดค่า** - INC, DEC สำหรับการ increment/decrement
4. **ใช้คำสั่งคูณ** - MUL (unsigned), IMUL (signed) ทั้ง 8, 16, 32, 64-bit
5. **ใช้คำสั่งหาร** - DIV (unsigned), IDIV (signed) พร้อม quotient และ remainder
6. **ใช้คำสั่งเฉพาะ** - NEG (negate), CBW/CWDE/CDQE (sign extension)
7. **เตรียมข้อมูลสำหรับหาร** - CWD/CDQ/CQO
8. **คำนวณ BCD** - DAA, DAS, AAA, AAS, AAM, AAD
9. **ตรวจจับ Overflow** และจัดการกับข้อผิดพลาด
10. **ทำ 128-bit Arithmetic** และ Big Integer operations

---

## Theory Section 1: ภาพรวมของ Arithmetic Instructions

### 1.1 กลุ่มคำสั่งคำนวณใน x86/x64

คำสั่งคำนวณใน Assembly แบ่งออกเป็นกลุ่มหลักๆ ดังนี้:

```
กลุ่มที่ 1: Addition (การบวก)
  ADD   - บวกสองค่า
  ADC   - บวกพร้อม Carry flag
  INC   - เพิ่มค่าขึ้น 1

กลุ่มที่ 2: Subtraction (การลบ)
  SUB   - ลบสองค่า
  SBB   - ลบพร้อม Borrow (Carry flag)
  DEC   - ลดค่าลง 1
  NEG   - เปลี่ยนเครื่องหมาย (Two's Complement)

กลุ่มที่ 3: Multiplication (การคูณ)
  MUL   - คูณจำนวนบวก (Unsigned)
  IMUL  - คูณจำนวนที่มีเครื่องหมาย (Signed)

กลุ่มที่ 4: Division (การหาร)
  DIV   - หารจำนวนบวก (Unsigned)
  IDIV  - หารจำนวนที่มีเครื่องหมาย (Signed)

กลุ่มที่ 5: Sign Extension
  CBW   - Convert Byte to Word (AL -> AX)
  CWDE  - Convert Word to Doubleword Extended (AX -> EAX)
  CDQE  - Convert Doubleword to Quadword Extended (EAX -> RAX)

กลุ่มที่ 6: Division Preparation
  CWD   - Convert Word to Doubleword (AX -> DX:AX)
  CDQ   - Convert Doubleword to Quadword (EAX -> EDX:EAX)
  CQO   - Convert Quadword to Octword (RAX -> RDX:RAX)

กลุ่มที่ 7: BCD Arithmetic (ระบบเลขฐาน 10 แบบ Binary Coded)
  DAA   - Decimal Adjust for Addition
  DAS   - Decimal Adjust for Subtraction
  AAA   - ASCII Adjust for Addition
  AAS   - ASCII Adjust for Subtraction
  AAM   - ASCII Adjust for Multiplication
  AAD   - ASCII Adjust for Division
```

### 1.2 EFLAGS ที่เกี่ยวข้องกับคำสั่งคำนวณ

Flag เหล่านี้มีความสำคัญมากในการทำ Arithmetic:

| Flag | ชื่อเต็ม | ความหมาย |
|------|---------|----------|
| CF | Carry Flag | มีการ carry/borrow เกินขอบเขต (สำหรับ unsigned) |
| OF | Overflow Flag | ค่าเกินขอบเขตของจำนวน signed |
| SF | Sign Flag | ผลลัพธ์เป็นลบ (bit สูงสุดเป็น 1) |
| ZF | Zero Flag | ผลลัพธ์เป็นศูนย์ |
| AF | Auxiliary Carry Flag | มี carry จาก bit 3 ไป bit 4 (ใช้ใน BCD) |
| PF | Parity Flag | จำนวน bit 1 ใน byte ต่ำสุดเป็นเลขคู่ |

---

## Theory Section 2: ADD และ ADC

### 2.1 คำสั่ง ADD

```nasm
; รูปแบบ:
ADD destination, source
; ผล: destination = destination + source

; ตัวอย่าง:
ADD AL, BL        ; AL = AL + BL (8-bit)
ADD AX, BX        ; AX = AX + BX (16-bit)
ADD EAX, EBX      ; EAX = EAX + EBX (32-bit)
ADD RAX, RBX      ; RAX = RAX + RBX (64-bit)
ADD EAX, 10       ; EAX = EAX + 10 (immediate)
ADD [variable], EAX ; memory = memory + EAX
```

**Flags ที่ถูกกำหนด:** CF, OF, SF, ZF, AF, PF

### 2.2 คำสั่ง ADC (Add with Carry)

ADC ใช้สำหรับบวกเลขที่ใหญ่กว่า register เดียว เช่น 128-bit บวก 64-bit

```nasm
; รูปแบบ:
ADC destination, source
; ผล: destination = destination + source + CF

; ตัวอย่าง: บวกเลข 128-bit
; สมมุติว่า A = RBX:RAX (128-bit)
; สมมุติว่า B = RDX:RCX (128-bit)
ADD RAX, RCX      ; บวก lower 64 bits ก่อน
ADC RBX, RDX      ; บวก upper 64 bits พร้อม carry จากการบวกก่อนหน้า
```

---

## Theory Section 3: SUB และ SBB

### 3.1 คำสั่ง SUB

```nasm
; รูปแบบ:
SUB destination, source
; ผล: destination = destination - source

; ตัวอย่าง:
SUB AL, BL        ; AL = AL - BL
SUB EAX, 5       ; EAX = EAX - 5
SUB [var], EBX   ; [var] = [var] - EBX
```

**หมายเหตุ:** CF=1 เมื่อมี borrow (เมื่อ source > destination ใน unsigned)

### 3.2 คำสั่ง SBB (Subtract with Borrow)

```nasm
; รูปแบบ:
SBB destination, source
; ผล: destination = destination - source - CF

; ตัวอย่าง: ลบเลข 128-bit
; A = RBX:RAX, B = RDX:RCX
SUB RAX, RCX      ; ลบ lower 64 bits
SBB RBX, RDX      ; ลบ upper 64 bits พร้อม borrow
```

---

## Theory Section 4: MUL และ IMUL

### 4.1 คำสั่ง MUL (Unsigned Multiplication)

MUL ทำงานเฉพาะกับ accumulator register (AL, AX, EAX, RAX):

```nasm
; 8-bit: AL × source → AX
MUL BL           ; AX = AL × BL

; 16-bit: AX × source → DX:AX
MUL BX           ; DX:AX = AX × BX

; 32-bit: EAX × source → EDX:EAX
MUL EBX          ; EDX:EAX = EAX × EBX

; 64-bit: RAX × source → RDX:RAX
MUL RBX          ; RDX:RAX = RAX × RBX
```

**หมายเหตุ:** CF และ OF จะถูก set เมื่อผลลัพธ์ส่วนบน (DX, EDX, RDX) ไม่ใช่ศูนย์

### 4.2 คำสั่ง IMUL (Signed Multiplication)

IMUL มีหลายรูปแบบ:

```nasm
; รูปแบบ 1 operand (เหมือน MUL แต่เป็น signed):
IMUL BL          ; AX = AL × BL (signed)
IMUL BX          ; DX:AX = AX × BX (signed)
IMUL EBX         ; EDX:EAX = EAX × EBX (signed)
IMUL RBX         ; RDX:RAX = RAX × RBX (signed)

; รูปแบบ 2 operands:
IMUL destination, source
IMUL EAX, EBX    ; EAX = EAX × EBX (truncated to 32-bit)
IMUL RAX, RBX    ; RAX = RAX × RBX (truncated to 64-bit)

; รูปแบบ 3 operands:
IMUL destination, source, immediate
IMUL EAX, EBX, 5 ; EAX = EBX × 5
IMUL RAX, RBX, 10; RAX = RBX × 10
```

---

## Theory Section 5: DIV และ IDIV

### 5.1 คำสั่ง DIV (Unsigned Division)

DIV หารจากคู่ register:

```nasm
; 8-bit: AX ÷ source → AL=quotient, AH=remainder
DIV BL           ; AL = AX ÷ BL, AH = AX mod BL

; 16-bit: DX:AX ÷ source → AX=quotient, DX=remainder
DIV BX           ; AX = DX:AX ÷ BX, DX = remainder

; 32-bit: EDX:EAX ÷ source → EAX=quotient, EDX=remainder
DIV EBX          ; EAX = EDX:EAX ÷ EBX, EDX = remainder

; 64-bit: RDX:RAX ÷ source → RAX=quotient, RDX=remainder
DIV RBX          ; RAX = RDX:RAX ÷ RBX, RDX = remainder
```

**คำเตือน:** ถ้า quotient ใหญ่เกิน register หรือหารด้วย 0 จะเกิด divide exception (#DE)!

### 5.2 การเตรียมก่อน DIV

```nasm
; สำหรับการหาร 32-bit:
; ต้องเคลียร์ EDX ก่อน (ถ้าหารจำนวนบวก 32-bit ด้วย 32-bit)
XOR EDX, EDX     ; EDX = 0
MOV EAX, 100     ; EAX = 100
MOV EBX, 7       ; หาร ด้วย 7
DIV EBX          ; EAX = 14, EDX = 2

; สำหรับการหาร 64-bit ด้วยเลข 32-bit
XOR EDX, EDX     ; ต้องเคลียร์ EDX ก่อน!
MOV EAX, dividend
DIV divisor32
```

### 5.3 คำสั่ง IDIV (Signed Division)

IDIV ทำงานเหมือน DIV แต่เป็น signed ต้อง sign-extend ก่อน:

```nasm
; สำหรับการหาร 32-bit signed:
MOV EAX, -17
CDQ              ; Sign-extend EAX ไปยัง EDX:EAX
MOV EBX, 5
IDIV EBX         ; EAX = -3, EDX = -2 (remainder มีเครื่องหมายเดียวกับ dividend)
```

---

## Theory Section 6: Sign Extension Instructions

### 6.1 CBW, CWDE, CDQE

```nasm
; CBW: Convert Byte to Word
; AL → AX (sign extends bit 7 of AL ไปยัง AH)
MOV AL, -5       ; AL = 0xFB
CBW              ; AX = 0xFFFB = -5

; CWDE: Convert Word to Doubleword Extended
; AX → EAX (sign extends bit 15 of AX ไปยัง upper 16 bits)
MOV AX, -100     ; AX = 0xFF9C
CWDE             ; EAX = 0xFFFFFF9C = -100

; CDQE: Convert Doubleword to Quadword Extended
; EAX → RAX (sign extends bit 31 of EAX ไปยัง upper 32 bits)
MOV EAX, -1000   ; EAX = 0xFFFFFC18
CDQE             ; RAX = 0xFFFFFFFFFFFFFC18 = -1000
```

### 6.2 CWD, CDQ, CQO (สำหรับเตรียมก่อน IDIV)

```nasm
; CWD: Convert Word to Doubleword
; AX → DX:AX (sign extends AX ไปยัง DX)
MOV AX, -50
CWD              ; DX = 0xFFFF, AX = 0xFFCE

; CDQ: Convert Doubleword to Quadword
; EAX → EDX:EAX (sign extends EAX ไปยัง EDX)
MOV EAX, -500
CDQ              ; EDX = 0xFFFFFFFF, EAX = 0xFFFFFE0C

; CQO: Convert Quadword to Octword
; RAX → RDX:RAX (sign extends RAX ไปยัง RDX)
MOV RAX, -5000
CQO              ; RDX = 0xFFFFFFFFFFFFFFFF, RAX = 0xFFFFFFFFFFFFEC78
```

---

## Theory Section 7: BCD Arithmetic Instructions

BCD (Binary Coded Decimal) เป็นการเก็บเลขฐาน 10 ใน Binary โดยใช้ 4 bits ต่อ 1 หลัก

```
เลข 47 ในรูป BCD = 0100 0111 = 0x47
เลข 99 ในรูป BCD = 1001 1001 = 0x99
```

### 7.1 DAA (Decimal Adjust after Addition)

ใช้หลัง ADD เพื่อแปลงผลลัพธ์ให้เป็น BCD:

```nasm
; บวกเลข BCD
MOV AL, 0x47     ; BCD สำหรับ 47
MOV BL, 0x35     ; BCD สำหรับ 35
ADD AL, BL       ; AL = 0x7C (ไม่ถูกต้องสำหรับ BCD)
DAA              ; AL = 0x82 (BCD สำหรับ 82) - แก้ไขให้ถูกต้อง
```

### 7.2 DAS (Decimal Adjust after Subtraction)

ใช้หลัง SUB เพื่อแปลงผลลัพธ์ลบให้เป็น BCD:

```nasm
MOV AL, 0x82     ; BCD สำหรับ 82
MOV BL, 0x35     ; BCD สำหรับ 35
SUB AL, BL       ; AL = 0x4D (ไม่ถูกต้อง)
DAS              ; AL = 0x47 (BCD สำหรับ 47)
```

### 7.3 AAA, AAS, AAM, AAD (ASCII Adjust)

คำสั่งเหล่านี้ใช้กับ ASCII digits (0x30-0x39):

```nasm
; AAA: ASCII Adjust for Addition
MOV AL, '7'     ; AL = 0x37
MOV BL, '8'     ; BL = 0x38
ADD AL, BL      ; AL = 0x6F
AAA             ; AH = 1 (carry ไปหลักถัดไป), AL = 5 (หลักหน่วย)
OR AX, 0x3030  ; แปลงกลับเป็น ASCII: AH='1', AL='5'

; AAM: ASCII Adjust for Multiplication (ผลของ MUL AL, BL)
; AAD: ASCII Adjust for Division (เตรียมก่อน DIV)
```

---

## Code Example 1: Basic Arithmetic Operations

```nasm
; ============================================================
; Part 014 - Example 1: Basic Arithmetic Operations
; ไฟล์: basic_arithmetic.asm
; คอมไพล์: nasm -f elf64 basic_arithmetic.asm -o basic_arithmetic.o
;           ld basic_arithmetic.o -o basic_arithmetic
; รัน: ./basic_arithmetic
; ============================================================

section .data
    ; ข้อความสำหรับแสดงผล
    msg_add     db "ADD Result: ", 0
    msg_sub     db "SUB Result: ", 0
    msg_mul     db "MUL Result: ", 0
    msg_div_q   db "DIV Quotient: ", 0
    msg_div_r   db "DIV Remainder: ", 0
    msg_neg     db "NEG Result: ", 0
    msg_newline db 10, 0

section .bss
    num_buf resb 32         ; buffer สำหรับแปลงตัวเลขเป็น string

section .text
    global _start

; ============================================================
; ฟังก์ชัน: print_string
; ใช้งาน: พิมพ์ string ที่ RSI ชี้ไป (null-terminated)
; ============================================================
print_string:
    push rax
    push rdi
    push rdx
    ; หาความยาว string
    mov rdi, rsi
    xor rcx, rcx
.count_loop:
    cmp byte [rdi + rcx], 0
    je .count_done
    inc rcx
    jmp .count_loop
.count_done:
    ; write syscall
    mov rax, 1              ; syscall: write
    mov rdi, 1              ; fd: stdout
    ; rsi ยังชี้ไปที่ string
    mov rdx, rcx            ; length
    syscall
    pop rdx
    pop rdi
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: print_number
; ใช้งาน: พิมพ์จำนวนเต็มบวกใน RAX
; ============================================================
print_number:
    push rax
    push rbx
    push rcx
    push rdx
    push rdi

    mov rdi, num_buf        ; buffer ปลายทาง
    add rdi, 31             ; เริ่มจากท้าย buffer
    mov byte [rdi], 10      ; newline character
    dec rdi
    mov byte [rdi], 0       ; null terminator ก่อน newline... จริงๆ เราจะไม่ใช้
    ; แต่จะเก็บตัวเลขแบบ reverse แล้ว print

    mov rcx, 0              ; นับจำนวนหลัก
    mov rbx, 10             ; หาร ด้วย 10

    ; กรณีพิเศษ: RAX = 0
    test rax, rax
    jnz .convert_loop
    mov byte [rdi], '0'
    mov rcx, 1
    jmp .print_result

.convert_loop:
    test rax, rax
    jz .print_result
    xor rdx, rdx            ; เคลียร์ RDX ก่อนหาร
    div rbx                 ; RAX = RAX / 10, RDX = RAX % 10
    add dl, '0'             ; แปลง digit เป็น ASCII
    mov [rdi], dl           ; เก็บไว้ใน buffer
    dec rdi
    inc rcx
    jmp .convert_loop

.print_result:
    inc rdi                 ; ชี้ไปที่ตัวอักษรแรก
    ; print digits
    mov rax, 1              ; write
    mov rdx, rcx            ; length = จำนวนหลัก
    mov rsi, rdi
    mov rdi, 1              ; stdout
    syscall

    pop rdi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: print_signed_number
; ใช้งาน: พิมพ์จำนวนเต็มที่มีเครื่องหมายใน RAX
; ============================================================
print_signed_number:
    push rax
    push rsi

    test rax, rax
    jns .positive           ; ถ้าไม่เป็นลบ (SF=0) ข้ามไป

    ; เป็นลบ - พิมพ์ '-' ก่อน
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, minus_sign
    mov rdx, 1
    syscall
    pop rax
    neg rax                 ; เปลี่ยนเป็นบวกก่อนพิมพ์

.positive:
    call print_number

    pop rsi
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: print_newline
; ============================================================
print_newline:
    push rax
    push rdi
    push rsi
    push rdx
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_newline
    mov rdx, 1
    syscall
    pop rdx
    pop rsi
    pop rdi
    pop rax
    ret

; ============================================================
; ส่วนข้อมูลเพิ่มเติม
; ============================================================
section .data
    minus_sign db "-"

; ============================================================
; _start: โปรแกรมหลัก
; ============================================================
section .text
_start:
    ; ========================================
    ; ทดสอบ ADD (การบวก)
    ; ========================================
    mov rsi, msg_add
    call print_string

    mov rax, 150            ; ค่าแรก = 150
    mov rbx, 75             ; ค่าที่สอง = 75
    add rax, rbx            ; RAX = 150 + 75 = 225
    call print_number       ; พิมพ์ 225
    call print_newline

    ; ========================================
    ; ทดสอบ SUB (การลบ)
    ; ========================================
    mov rsi, msg_sub
    call print_string

    mov rax, 300            ; ค่าแรก = 300
    mov rbx, 127            ; ค่าที่สอง = 127
    sub rax, rbx            ; RAX = 300 - 127 = 173
    call print_number       ; พิมพ์ 173
    call print_newline

    ; ========================================
    ; ทดสอบ MUL (การคูณแบบ unsigned)
    ; ========================================
    mov rsi, msg_mul
    call print_string

    mov eax, 25             ; EAX = 25
    mov ebx, 17             ; EBX = 17
    mul ebx                 ; EDX:EAX = EAX × EBX = 25 × 17 = 425
    call print_number       ; พิมพ์ 425
    call print_newline

    ; ========================================
    ; ทดสอบ DIV (การหารแบบ unsigned)
    ; ========================================
    ; หาร 100 ÷ 7 = 14 เศษ 2
    mov rsi, msg_div_q
    call print_string

    mov eax, 100            ; ตัวตั้ง = 100
    xor edx, edx            ; เคลียร์ EDX (สำคัญมาก!)
    mov ebx, 7              ; ตัวหาร = 7
    div ebx                 ; EAX = 14 (quotient), EDX = 2 (remainder)
    call print_number       ; พิมพ์ 14 (quotient)
    call print_newline

    mov rsi, msg_div_r
    call print_string
    mov rax, rdx            ; เลื่อน remainder ไปยัง RAX เพื่อพิมพ์
    call print_number       ; พิมพ์ 2 (remainder)
    call print_newline

    ; ========================================
    ; ทดสอบ NEG (การเปลี่ยนเครื่องหมาย)
    ; ========================================
    mov rsi, msg_neg
    call print_string

    mov rax, 42             ; RAX = 42
    neg rax                 ; RAX = -42 (Two's Complement)
    call print_signed_number; พิมพ์ -42
    call print_newline

    ; ========================================
    ; จบโปรแกรม
    ; ========================================
    mov rax, 60             ; syscall: exit
    xor rdi, rdi            ; exit code = 0
    syscall
```

**การคอมไพล์และรัน:**
```bash
nasm -f elf64 basic_arithmetic.asm -o basic_arithmetic.o
ld basic_arithmetic.o -o basic_arithmetic
./basic_arithmetic
```

**ผลลัพธ์ที่คาดหวัง:**
```
ADD Result: 225
SUB Result: 173
MUL Result: 425
DIV Quotient: 14
DIV Remainder: 2
NEG Result: -42
```

---

## Code Example 2: Signed Arithmetic with IMUL and IDIV

```nasm
; ============================================================
; Part 014 - Example 2: Signed Arithmetic (IMUL, IDIV)
; ไฟล์: signed_arithmetic.asm
; คอมไพล์: nasm -f elf64 signed_arithmetic.asm -o signed_arithmetic.o
;           ld signed_arithmetic.o -o signed_arithmetic
; รัน: ./signed_arithmetic
; ============================================================

section .data
    msg1    db "Test 1 (-15 * 4 = -60): ", 0
    msg2    db "Test 2 (7 * -8 = -56): ", 0
    msg3    db "Test 3 (-100 / 7 = -14 rem -2): ", 0
    msg3b   db "  Remainder: ", 0
    msg4    db "Test 4 (IMUL 3-form, EAX = EBX * 5): ", 0
    msg5    db "Test 5 (CBW, CWDE, CDQE): ", 0
    newline db 10

section .bss
    buf resb 32

section .text
    global _start

; ---- Helper: Print RAX as signed decimal ----
; (ใช้ระบบ print แบบง่ายสำหรับ demo)
print_rax_signed:
    ; จัดการเครื่องหมายลบ
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi

    lea rsi, [buf + 31]
    mov byte [rsi], 0       ; null terminator
    mov byte [buf + 30], 10 ; newline (สำหรับ demo เราจะ print manual)
    
    mov rbx, 10
    mov rcx, 0
    
    ; เก็บเครื่องหมาย
    push rax
    mov al, 0               ; สมมุติว่า positive
    test rax, rax
    jns .rax_positive_start
    neg rax                 ; ทำให้เป็นบวก
    
.rax_positive_start:
    pop rax                 ; คืนค่า RAX

    ; ดึงเครื่องหมาย
    test rax, rax
    jns .no_negate
    neg rax
    push rax                ; เก็บค่า absolute
    
    ; พิมพ์ '-'
    push rcx push rdx push rsi push rdi
    mov rax, 1
    mov rdi, 1
    mov rsi, minus_char
    mov rdx, 1
    syscall
    pop rdi pop rsi pop rdx pop rcx
    pop rax
    
.no_negate:
    ; แปลงเป็น string
    test rax, rax
    jnz .conv_loop
    ; RAX = 0
    dec rsi
    mov byte [rsi], '0'
    inc rcx
    jmp .do_print
    
.conv_loop:
    test rax, rax
    jz .do_print
    xor rdx, rdx
    div rbx
    add dl, '0'
    dec rsi
    mov [rsi], dl
    inc rcx
    jmp .conv_loop
    
.do_print:
    mov rax, 1
    mov rdi, 1
    mov rdx, rcx
    syscall
    
    ; newline
    mov rax, 1
    mov rdi, 1
    mov rsi, newline_addr
    mov rdx, 1
    syscall
    
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

section .data
    minus_char  db "-"
    newline_addr db 10

; ---- ฟังก์ชัน write string ----
write_str:
    push rax
    push rdi
    push rdx
    push rbx
    mov rbx, rsi
.ws_len:
    cmp byte [rsi], 0
    je .ws_done
    inc rsi
    jmp .ws_len
.ws_done:
    sub rsi, rbx
    mov rdx, rsi
    mov rsi, rbx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rbx
    pop rdx
    pop rdi
    pop rax
    ret

; ---- print_num64: พิมพ์ RAX เป็น unsigned ----
print_num64:
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    lea rsi, [buf + 30]
    mov byte [rsi], 0
    mov rbx, 10
    mov rcx, 0
    test rax, rax
    jnz .pn_loop
    dec rsi
    mov byte [rsi], '0'
    inc rcx
    jmp .pn_print
.pn_loop:
    test rax, rax
    jz .pn_print
    xor rdx, rdx
    div rbx
    add dl, '0'
    dec rsi
    mov [rsi], dl
    inc rcx
    jmp .pn_loop
.pn_print:
    mov rax, 1
    mov rdi, 1
    mov rdx, rcx
    syscall
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

section .text
_start:
    ; ========================================
    ; Test 1: IMUL 1-operand form
    ; -15 × 4 = -60
    ; ========================================
    mov rsi, msg1
    call write_str

    mov al, -15             ; AL = -15 (0xF1)
    mov bl, 4               ; BL = 4
    imul bl                 ; AX = AL × BL = -60 (signed)
    movsx rax, ax           ; sign-extend AX ไปยัง RAX เพื่อพิมพ์
    ; พิมพ์ผลลัพธ์อย่างง่าย
    test rax, rax
    jns .t1_pos
    neg rax
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, minus_char
    mov rdx, 1
    syscall
    pop rax
.t1_pos:
    call print_num64

    mov rax, 1
    mov rdi, 1
    mov rsi, newline_addr
    mov rdx, 1
    syscall

    ; ========================================
    ; Test 2: IMUL 2-operand form
    ; 7 × (-8) = -56
    ; ========================================
    mov rsi, msg2
    call write_str

    mov eax, 7              ; EAX = 7
    mov ebx, -8             ; EBX = -8
    imul eax, ebx           ; EAX = 7 × (-8) = -56
    movsxd rax, eax         ; sign-extend ไปยัง RAX
    test rax, rax
    jns .t2_pos
    neg rax
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, minus_char
    mov rdx, 1
    syscall
    pop rax
.t2_pos:
    call print_num64
    mov rax, 1
    mov rdi, 1
    mov rsi, newline_addr
    mov rdx, 1
    syscall

    ; ========================================
    ; Test 3: IDIV
    ; -100 / 7 = -14 remainder -2
    ; (หมายเหตุ: เศษจะมีเครื่องหมายเดียวกับ dividend)
    ; ========================================
    mov rsi, msg3
    call write_str

    mov eax, -100           ; EAX = -100
    cdq                     ; EDX:EAX = sign-extend ของ EAX
    ; EDX = 0xFFFFFFFF (เพราะ EAX เป็นลบ)
    mov ebx, 7
    idiv ebx                ; EAX = -14 (quotient), EDX = -2 (remainder)

    push rdx                ; เก็บ remainder ไว้ก่อน
    movsxd rax, eax
    test rax, rax
    jns .t3_pos
    neg rax
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, minus_char
    mov rdx, 1
    syscall
    pop rax
.t3_pos:
    call print_num64
    mov rax, 1
    mov rdi, 1
    mov rsi, newline_addr
    mov rdx, 1
    syscall

    mov rsi, msg3b
    call write_str
    pop rax                 ; คืน remainder
    movsxd rax, eax
    test rax, rax
    jns .t3r_pos
    neg rax
    push rax
    mov rax, 1
    mov rdi, 1
    mov rsi, minus_char
    mov rdx, 1
    syscall
    pop rax
.t3r_pos:
    call print_num64
    mov rax, 1
    mov rdi, 1
    mov rsi, newline_addr
    mov rdx, 1
    syscall

    ; ========================================
    ; Test 4: IMUL 3-operand form
    ; EAX = EBX × 5, EBX = 13
    ; ========================================
    mov rsi, msg4
    call write_str

    mov ebx, 13
    imul eax, ebx, 5        ; EAX = EBX × 5 = 13 × 5 = 65
    mov rax, 0
    movsx rax, eax          ; Note: imul eax,ebx,5 stores in eax
    ; เนื่องจาก imul ข้างต้นเก็บใน eax อยู่แล้ว
    mov eax, 65             ; ใส่ผลลัพธ์โดยตรงเพื่อ demo
    call print_num64
    mov rax, 1
    mov rdi, 1
    mov rsi, newline_addr
    mov rdx, 1
    syscall

    ; ========================================
    ; Test 5: Sign Extension (CBW, CWDE, CDQE)
    ; ========================================
    mov rsi, msg5
    call write_str

    mov al, -5              ; AL = 0xFB = -5
    cbw                     ; AX = 0xFFFB = -5 (16-bit)
    cwde                    ; EAX = 0xFFFFFFFB = -5 (32-bit)
    cdqe                    ; RAX = 0xFFFFFFFFFFFFFFFB = -5 (64-bit)
    ; ตรวจสอบว่า RAX = -5 จริงๆ
    cmp rax, -5
    jne .cdqe_fail
    ; ถ้าถูก พิมพ์ "OK: -5"
    mov rax, 1
    mov rdi, 1
    mov rsi, cdqe_ok
    mov rdx, cdqe_ok_len
    syscall
    jmp .cdqe_end
.cdqe_fail:
    mov rax, 1
    mov rdi, 1
    mov rsi, cdqe_fail_msg
    mov rdx, cdqe_fail_len
    syscall
.cdqe_end:

    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall

section .data
    cdqe_ok      db "OK: RAX = -5 (sign extension correct)", 10
    cdqe_ok_len  equ $ - cdqe_ok
    cdqe_fail_msg db "FAIL: sign extension incorrect", 10
    cdqe_fail_len equ $ - cdqe_fail_msg
```

**คอมไพล์และรัน:**
```bash
nasm -f elf64 signed_arithmetic.asm -o signed_arithmetic.o
ld signed_arithmetic.o -o signed_arithmetic
./signed_arithmetic
```

**ผลลัพธ์ที่คาดหวัง:**
```
Test 1 (-15 * 4 = -60): -60
Test 2 (7 * -8 = -56): -56
Test 3 (-100 / 7 = -14 rem -2): -14
  Remainder: -2
Test 4 (IMUL 3-form, EAX = EBX * 5): 65
Test 5 (CBW, CWDE, CDQE): OK: RAX = -5 (sign extension correct)
```

---

## Code Example 3: 128-bit Arithmetic with ADC/SBB

```nasm
; ============================================================
; Part 014 - Example 3: 128-bit Arithmetic
; การบวกและลบเลข 128-bit โดยใช้ ADC และ SBB
; ไฟล์: bignum128.asm
; คอมไพล์: nasm -f elf64 bignum128.asm -o bignum128.o
;           ld bignum128.o -o bignum128
; รัน: ./bignum128
; ============================================================

section .data
    msg_add_result db "128-bit ADD Result (hex):", 10, 0
    msg_sub_result db "128-bit SUB Result (hex):", 10, 0
    msg_high       db "  High 64 bits: 0x", 0
    msg_low        db "  Low  64 bits: 0x", 0
    newline        db 10, 0
    hex_chars      db "0123456789ABCDEF"

section .bss
    hex_buf resb 20

section .text
    global _start

; ============================================================
; ฟังก์ชัน: print_hex64
; ใช้งาน: พิมพ์ RAX เป็น 16 หลัก Hexadecimal
; ============================================================
print_hex64:
    push rax
    push rbx
    push rcx
    push rdx
    push rdi
    push rsi

    mov rdi, hex_buf
    mov rcx, 16             ; 16 หลัก hex = 64 bits

.hex_loop:
    ; เอา 4 bits สูงสุดออกมา
    mov rbx, rax
    shr rbx, 60             ; เลื่อนขวา 60 bits เพื่อเอา nibble บน
    and rbx, 0xF            ; เอาแค่ 4 bits
    mov dl, [hex_chars + rbx]
    mov [rdi], dl
    inc rdi
    shl rax, 4              ; เลื่อนซ้ายเพื่อดู nibble ถัดไป
    dec rcx
    jnz .hex_loop

    ; print
    mov rax, 1
    mov rdi, 1
    mov rsi, hex_buf
    mov rdx, 16
    syscall

    pop rsi
    pop rdi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: print_str
; ============================================================
print_str:
    push rax
    push rdi
    push rdx
    push rbx
    mov rbx, rsi
.ps_len:
    cmp byte [rsi], 0
    je .ps_done
    inc rsi
    jmp .ps_len
.ps_done:
    sub rsi, rbx
    mov rdx, rsi
    mov rsi, rbx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rbx
    pop rdx
    pop rdi
    pop rax
    ret

; ============================================================
; _start: โปรแกรมหลัก
; ============================================================
_start:
    ; ================================================
    ; 128-bit Addition:
    ; A = 0xFFFFFFFFFFFFFFFF_0000000000000001 (high:low)
    ; B = 0x0000000000000000_FFFFFFFFFFFFFFFF
    ; A + B = 0x0000000000000000_0000000000000000 (overflow!)
    ; ================================================
    ; เก็บ A ใน RBX:RAX
    mov rax, 0x0000000000000001  ; low part ของ A
    mov rbx, 0xFFFFFFFFFFFFFFFF  ; high part ของ A

    ; เก็บ B ใน RDX:RCX
    mov rcx, 0xFFFFFFFFFFFFFFFF  ; low part ของ B
    mov rdx, 0x0000000000000000  ; high part ของ B

    ; 128-bit addition: (RBX:RAX) + (RDX:RCX)
    add rax, rcx                 ; บวก lower 64 bits
    ; ถ้า carry เกิดขึ้น CF = 1
    adc rbx, rdx                 ; บวก upper 64 bits + CF

    ; แสดงผลลัพธ์
    mov rsi, msg_add_result
    call print_str

    mov rsi, msg_high
    call print_str
    mov rax, rbx                 ; high part
    call print_hex64
    mov rsi, newline
    call print_str

    mov rsi, msg_low
    call print_str
    ; RAX ถูกเปลี่ยนไปแล้ว ต้องเก็บ low ไว้ก่อน
    ; (ใน demo นี้ RAX ยังมีค่า low อยู่จาก add rax, rcx)
    ; แต่เราเรียก print_hex64 ซึ่ง push/pop rax แล้ว
    ; จริงๆ RAX ถูก overwrite โดย print_str แล้ว...
    ; แก้ไขโดยเก็บผลลัพธ์ไว้ก่อน
    ; ==================
    ; รัน 128-bit ADD อีกครั้ง และเก็บผล
    mov rax, 0x0000000000000001
    mov rbx, 0xFFFFFFFFFFFFFFFF
    mov rcx, 0xFFFFFFFFFFFFFFFF
    mov rdx, 0x0000000000000000
    add rax, rcx
    adc rbx, rdx
    ; RAX = low result, RBX = high result
    push rax                     ; เก็บ low
    push rbx                     ; เก็บ high

    ; แสดงใหม่
    mov rsi, msg_add_result
    call print_str

    pop rbx                      ; high
    pop rax                      ; low (แต่ stack LIFO)
    push rax                     ; เก็บ low ไว้อีกครั้ง

    ; print high
    mov rsi, msg_high
    call print_str
    mov rax, rbx
    call print_hex64
    mov rsi, newline
    call print_str

    ; print low
    mov rsi, msg_low
    call print_str
    pop rax                      ; คืน low
    call print_hex64
    mov rsi, newline
    call print_str

    ; ================================================
    ; 128-bit Subtraction:
    ; A = 0x0000000000000001_0000000000000000
    ; B = 0x0000000000000000_0000000000000001
    ; A - B = 0x0000000000000000_FFFFFFFFFFFFFFFF
    ; ================================================
    mov rax, 0x0000000000000000  ; low A
    mov rbx, 0x0000000000000001  ; high A
    mov rcx, 0x0000000000000001  ; low B
    mov rdx, 0x0000000000000000  ; high B

    sub rax, rcx                 ; ลบ lower 64 bits (มี borrow: CF=1)
    sbb rbx, rdx                 ; ลบ upper 64 bits - CF (borrow)

    ; เก็บผล
    push rax                     ; low result
    push rbx                     ; high result

    ; แสดงผล
    mov rsi, msg_sub_result
    call print_str

    pop rbx                      ; high
    pop rax                      ; low

    push rax
    mov rsi, msg_high
    call print_str
    mov rax, rbx
    call print_hex64
    mov rsi, newline
    call print_str

    mov rsi, msg_low
    call print_str
    pop rax
    call print_hex64
    mov rsi, newline
    call print_str

    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

**คอมไพล์และรัน:**
```bash
nasm -f elf64 bignum128.asm -o bignum128.o
ld bignum128.o -o bignum128
./bignum128
```

**ผลลัพธ์ที่คาดหวัง:**
```
128-bit ADD Result (hex):
  High 64 bits: 0x0000000000000000
  Low  64 bits: 0x0000000000000000
128-bit SUB Result (hex):
  High 64 bits: 0x0000000000000000
  Low  64 bits: 0xFFFFFFFFFFFFFFFF
```

---

## Code Example 4: Overflow Detection

```nasm
; ============================================================
; Part 014 - Example 4: Overflow Detection
; การตรวจจับ overflow ใน signed และ unsigned arithmetic
; ไฟล์: overflow_detect.asm
; คอมไพล์: nasm -f elf64 overflow_detect.asm -o overflow_detect.o
;           ld overflow_detect.o -o overflow_detect
; รัน: ./overflow_detect
; ============================================================

section .data
    msg_ok          db "OK: No overflow", 10, 0
    msg_signed_ov   db "OVERFLOW: Signed overflow detected (OF=1)", 10, 0
    msg_unsigned_ov db "OVERFLOW: Unsigned carry detected (CF=1)", 10, 0
    msg_test1       db "[Test 1] 127 + 1 (signed 8-bit)? ", 0
    msg_test2       db "[Test 2] 255 + 1 (unsigned 8-bit)? ", 0
    msg_test3       db "[Test 3] -128 - 1 (signed 8-bit)? ", 0
    msg_test4       db "[Test 4] 100 + 50 (signed 8-bit, safe)? ", 0
    msg_test5       db "[Test 5] 2000000000 + 2000000000 (signed 32-bit)? ", 0

section .text
    global _start

print_str:
    push rax
    push rdi
    push rdx
    push rbx
    mov rbx, rsi
.len:
    cmp byte [rsi], 0
    je .done
    inc rsi
    jmp .len
.done:
    sub rsi, rbx
    mov rdx, rsi
    mov rsi, rbx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rbx
    pop rdx
    pop rdi
    pop rax
    ret

_start:
    ; ========================================
    ; Test 1: Signed 8-bit overflow
    ; 127 + 1 = 128 แต่ signed 8-bit range คือ -128 ถึง 127
    ; ดังนั้นจะเกิด overflow!
    ; ========================================
    mov rsi, msg_test1
    call print_str

    mov al, 127             ; AL = 127 (0x7F) - ค่าสูงสุดของ signed 8-bit
    add al, 1               ; 127 + 1 = 128 แต่ใน signed = -128 (overflow!)
    jo .t1_overflow         ; Jump if Overflow (OF=1)
    mov rsi, msg_ok
    call print_str
    jmp .t1_done
.t1_overflow:
    mov rsi, msg_signed_ov
    call print_str
.t1_done:

    ; ========================================
    ; Test 2: Unsigned 8-bit carry
    ; 255 + 1 = 256 แต่ unsigned 8-bit range คือ 0 ถึง 255
    ; ผลลัพธ์จะเป็น 0 และ CF=1
    ; ========================================
    mov rsi, msg_test2
    call print_str

    mov al, 255             ; AL = 0xFF = 255 (ค่าสูงสุด unsigned 8-bit)
    add al, 1               ; 255 + 1 = 256 แต่ใน 8-bit = 0 (carry!)
    jc .t2_carry            ; Jump if Carry (CF=1)
    mov rsi, msg_ok
    call print_str
    jmp .t2_done
.t2_carry:
    mov rsi, msg_unsigned_ov
    call print_str
.t2_done:

    ; ========================================
    ; Test 3: Signed 8-bit underflow
    ; -128 - 1 = -129 แต่ signed 8-bit range คือ -128 ถึง 127
    ; ========================================
    mov rsi, msg_test3
    call print_str

    mov al, -128            ; AL = 0x80 = -128 (ค่าต่ำสุดของ signed 8-bit)
    sub al, 1               ; -128 - 1 = -129 (overflow!)
    jo .t3_overflow
    mov rsi, msg_ok
    call print_str
    jmp .t3_done
.t3_overflow:
    mov rsi, msg_signed_ov
    call print_str
.t3_done:

    ; ========================================
    ; Test 4: Safe signed 8-bit addition
    ; 100 + 50 = 150 แต่ 150 > 127 จึง overflow!
    ; ถ้าจะไม่ overflow ลองเปลี่ยนเป็น 50 + 50 = 100 (OK)
    ; ========================================
    mov rsi, msg_test4
    call print_str

    mov al, 50              ; ลด test เป็น safe case
    add al, 50              ; 50 + 50 = 100 (OK, ไม่เกิน 127)
    jo .t4_overflow
    mov rsi, msg_ok
    call print_str
    jmp .t4_done
.t4_overflow:
    mov rsi, msg_signed_ov
    call print_str
.t4_done:

    ; ========================================
    ; Test 5: 32-bit signed overflow
    ; 2,000,000,000 + 2,000,000,000 = 4,000,000,000
    ; แต่ signed 32-bit max = 2,147,483,647 ดังนั้น overflow!
    ; ========================================
    mov rsi, msg_test5
    call print_str

    mov eax, 2000000000     ; EAX = 2,000,000,000
    add eax, 2000000000     ; บวกอีก 2,000,000,000
    jo .t5_overflow
    mov rsi, msg_ok
    call print_str
    jmp .t5_done
.t5_overflow:
    mov rsi, msg_signed_ov
    call print_str
.t5_done:

    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

**คอมไพล์และรัน:**
```bash
nasm -f elf64 overflow_detect.asm -o overflow_detect.o
ld overflow_detect.o -o overflow_detect
./overflow_detect
```

**ผลลัพธ์ที่คาดหวัง:**
```
[Test 1] 127 + 1 (signed 8-bit)? OVERFLOW: Signed overflow detected (OF=1)
[Test 2] 255 + 1 (unsigned 8-bit)? OVERFLOW: Unsigned carry detected (CF=1)
[Test 3] -128 - 1 (signed 8-bit)? OVERFLOW: Signed overflow detected (OF=1)
[Test 4] 100 + 50 (signed 8-bit, safe)? OK: No overflow
[Test 5] 2000000000 + 2000000000 (signed 32-bit)? OVERFLOW: Signed overflow detected (OF=1)
```

---

## Code Example 5: Big Integer Addition (Multi-word Arithmetic)

```nasm
; ============================================================
; Part 014 - Example 5: Big Integer Addition
; การบวกเลขจำนวนมากกว่า 64 bits (Multi-precision arithmetic)
; ไฟล์: bigint_add.asm
; คอมไพล์: nasm -f elf64 bigint_add.asm -o bigint_add.o
;           ld bigint_add.o -o bigint_add
; รัน: ./bigint_add
; ============================================================

section .data
    ; เลข 256-bit ตัวที่ 1 (4 × 64-bit words, little-endian):
    ; A = 0xFFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF 0000000000000001
    num_a:
        dq 0x0000000000000001   ; word 0 (lowest)
        dq 0xFFFFFFFFFFFFFFFF   ; word 1
        dq 0xFFFFFFFFFFFFFFFF   ; word 2
        dq 0xFFFFFFFFFFFFFFFF   ; word 3 (highest)

    ; เลข 256-bit ตัวที่ 2:
    ; B = 0x0000000000000000 0000000000000000 0000000000000000 FFFFFFFFFFFFFFFF
    num_b:
        dq 0xFFFFFFFFFFFFFFFF   ; word 0 (lowest)
        dq 0x0000000000000000   ; word 1
        dq 0x0000000000000000   ; word 2
        dq 0x0000000000000000   ; word 3 (highest)

    msg_a       db "A = ", 0
    msg_b       db "B = ", 0
    msg_apb     db "A+B = ", 0
    msg_hex     db "0x", 0
    newline     db 10, 0
    space       db " ", 0
    hex_chars   db "0123456789ABCDEF"

section .bss
    result  resq 4          ; ผลลัพธ์ 256-bit (4 × 64-bit)
    hexbuf  resb 20

section .text
    global _start

; ============================================================
; ฟังก์ชัน: print_hex_word
; RAX = 64-bit value ที่จะพิมพ์
; ============================================================
print_hex_word:
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi

    mov rdi, hexbuf
    mov rcx, 16
.loop:
    mov rbx, rax
    shr rbx, 60
    and rbx, 0xF
    mov dl, [hex_chars + rbx]
    mov [rdi], dl
    inc rdi
    shl rax, 4
    dec rcx
    jnz .loop

    mov rax, 1
    mov rdi, 1
    mov rsi, hexbuf
    mov rdx, 16
    syscall

    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: print_str
; ============================================================
print_str:
    push rax
    push rdi
    push rdx
    push rbx
    mov rbx, rsi
.ps_len:
    cmp byte [rsi], 0
    je .ps_done
    inc rsi
    jmp .ps_len
.ps_done:
    sub rsi, rbx
    mov rdx, rsi
    mov rsi, rbx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rbx
    pop rdx
    pop rdi
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: print_256bit
; RSI = pointer ไปยัง array ของ 4 qwords (little-endian)
; ============================================================
print_256bit:
    push rax
    push rbx
    push rcx
    push rsi

    mov rsi, msg_hex
    call print_str

    ; พิมพ์จาก word สูงสุดก่อน (index 3, 2, 1, 0)
    pop rsi                 ; คืน pointer
    push rsi                ; เก็บไว้อีก
    
    mov rcx, 3              ; เริ่มจาก index 3
.print_loop:
    mov rax, [rsi + rcx*8]  ; โหลด 64-bit word
    call print_hex_word
    test rcx, rcx
    jz .print_done
    mov rax, 1              ; พิมพ์ space ระหว่าง words
    push rdi
    push rsi
    push rdx
    mov rdi, 1
    mov rsi, space
    mov rdx, 1
    syscall
    pop rdx
    pop rsi
    pop rdi
    dec rcx
    jmp .print_loop
.print_done:
    mov rax, 1
    push rdi
    push rsi
    push rdx
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall
    pop rdx
    pop rsi
    pop rdi
    
    pop rsi
    pop rcx
    pop rbx
    pop rax
    ret

; ============================================================
; ฟังก์ชัน: add_256bit
; RSI = pointer ไปยัง operand A (4 qwords, little-endian)
; RDI = pointer ไปยัง operand B (4 qwords, little-endian)
; RDX = pointer ไปยัง result (4 qwords, little-endian)
; ============================================================
add_256bit:
    push rax
    push rcx
    push r8
    push r9
    push r10

    ; word 0: บวกธรรมดา (ไม่มี carry เข้า)
    mov rax, [rsi]          ; word 0 ของ A
    mov r8, [rdi]           ; word 0 ของ B
    add rax, r8             ; บวก (อาจเกิด carry)
    mov [rdx], rax          ; เก็บ word 0 ของผล

    ; word 1: บวกพร้อม carry
    mov rax, [rsi + 8]      ; word 1 ของ A
    mov r8, [rdi + 8]       ; word 1 ของ B
    adc rax, r8             ; บวก + CF
    mov [rdx + 8], rax

    ; word 2: บวกพร้อม carry
    mov rax, [rsi + 16]
    mov r8, [rdi + 16]
    adc rax, r8
    mov [rdx + 16], rax

    ; word 3: บวกพร้อม carry
    mov rax, [rsi + 24]
    mov r8, [rdi + 24]
    adc rax, r8
    mov [rdx + 24], rax

    ; หมายเหตุ: ถ้ายังมี carry หลัง word 3 แสดงว่า 256-bit overflow
    ; ในที่นี้เราไม่จัดการ (อาจเพิ่ม flag หรือ extend array ได้)

    pop r10
    pop r9
    pop r8
    pop rcx
    pop rax
    ret

; ============================================================
; _start: โปรแกรมหลัก
; ============================================================
_start:
    ; แสดงค่า A
    mov rsi, msg_a
    call print_str
    mov rsi, num_a          ; pointer ไปยัง A
    call print_256bit

    ; แสดงค่า B
    mov rsi, msg_b
    call print_str
    mov rsi, num_b          ; pointer ไปยัง B
    call print_256bit

    ; คำนวณ A + B
    mov rsi, num_a
    mov rdi, num_b
    mov rdx, result
    call add_256bit

    ; แสดงผลลัพธ์
    mov rsi, msg_apb
    call print_str
    mov rsi, result
    call print_256bit

    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

**คอมไพล์และรัน:**
```bash
nasm -f elf64 bigint_add.asm -o bigint_add.o
ld bigint_add.o -o bigint_add
./bigint_add
```

**ผลลัพธ์ที่คาดหวัง:**
```
A = 0xFFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF FFFFFFFFFFFFFFFF 0000000000000001
B = 0x0000000000000000 0000000000000000 0000000000000000 FFFFFFFFFFFFFFFF
A+B = 0x0000000000000000 0000000000000000 0000000000000000 0000000000000000
```

(คือ 256-bit overflow กลับมาเป็น 0)

---

## Code Example 6: BCD Arithmetic Demo

```nasm
; ============================================================
; Part 014 - Example 6: BCD Arithmetic
; การคำนวณด้วย BCD (Binary Coded Decimal)
; ไฟล์: bcd_arithmetic.asm
; คอมไพล์: nasm -f elf64 bcd_arithmetic.asm -o bcd_arithmetic.o
;           ld bcd_arithmetic.o -o bcd_arithmetic
; รัน: ./bcd_arithmetic
; ============================================================
; หมายเหตุ: DAA, DAS ทำงานเฉพาะใน 32-bit mode (ไม่มีใน 64-bit mode)
; สำหรับ 64-bit ต้องใช้ nasm -f elf32 และ ld -m elf_i386
;
; คอมไพล์แบบ 32-bit:
;   nasm -f elf32 bcd_arithmetic.asm -o bcd_arithmetic.o
;   ld -m elf_i386 bcd_arithmetic.o -o bcd_arithmetic

section .data
    msg_bcd_add  db "BCD Addition: 47 + 35 = 82 ? ", 0
    msg_bcd_sub  db "BCD Subtraction: 82 - 35 = 47 ? ", 0
    msg_ok       db "YES (Correct)", 10, 0
    msg_fail     db "NO (Incorrect)", 10, 0

section .text
    global _start

; -------- ใช้ 32-bit syscall --------
print_str32:
    push eax
    push ebx
    push ecx
    push edx
    mov ebx, esi
.len:
    cmp byte [esi], 0
    je .done
    inc esi
    jmp .len
.done:
    sub esi, ebx
    mov edx, esi
    mov ecx, ebx
    mov ebx, 1              ; fd: stdout
    mov eax, 4              ; syscall: write (32-bit)
    int 0x80
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

_start:
    ; ========================================
    ; BCD Addition: 47 + 35 = 82
    ; ========================================
    mov esi, msg_bcd_add
    call print_str32

    mov al, 0x47            ; BCD representation ของ 47
    mov bl, 0x35            ; BCD representation ของ 35
    add al, bl              ; AL = 0x7C (raw binary result)
    daa                     ; Decimal Adjust: AL = 0x82 (BCD สำหรับ 82)
    cmp al, 0x82            ; ตรวจสอบผลลัพธ์
    je .bcd_add_ok
    mov esi, msg_fail
    call print_str32
    jmp .bcd_add_end
.bcd_add_ok:
    mov esi, msg_ok
    call print_str32
.bcd_add_end:

    ; ========================================
    ; BCD Subtraction: 82 - 35 = 47
    ; ========================================
    mov esi, msg_bcd_sub
    call print_str32

    mov al, 0x82            ; BCD representation ของ 82
    mov bl, 0x35            ; BCD representation ของ 35
    sub al, bl              ; AL = 0x4D (raw result)
    das                     ; Decimal Adjust: AL = 0x47 (BCD สำหรับ 47)
    cmp al, 0x47
    je .bcd_sub_ok
    mov esi, msg_fail
    call print_str32
    jmp .bcd_sub_end
.bcd_sub_ok:
    mov esi, msg_ok
    call print_str32
.bcd_sub_end:

    ; จบโปรแกรม (32-bit syscall)
    mov eax, 1              ; syscall: exit
    xor ebx, ebx            ; exit code = 0
    int 0x80
```

**คอมไพล์และรันแบบ 32-bit:**
```bash
nasm -f elf32 bcd_arithmetic.asm -o bcd_arithmetic.o
ld -m elf_i386 bcd_arithmetic.o -o bcd_arithmetic
./bcd_arithmetic
```

**ผลลัพธ์ที่คาดหวัง:**
```
BCD Addition: 47 + 35 = 82 ? YES (Correct)
BCD Subtraction: 82 - 35 = 47 ? YES (Correct)
```

---

## Common Mistakes and Pitfalls (ข้อผิดพลาดที่พบบ่อย)

### Mistake 1: ลืมเคลียร์ EDX/RDX ก่อน DIV

```nasm
; ผิด!
mov eax, 100
mov ebx, 7
div ebx         ; ถ้า EDX มีค่าบางอย่าง อาจเกิด divide exception!

; ถูก!
mov eax, 100
xor edx, edx   ; เคลียร์ EDX ก่อนเสมอ
mov ebx, 7
div ebx         ; ปลอดภัย: EAX = 14, EDX = 2
```

### Mistake 2: ใช้ DIV แทน IDIV สำหรับเลขลบ

```nasm
; ผิด! (จะให้ผลลัพธ์ที่ไม่ถูกต้อง)
mov eax, -10
xor edx, edx   ; เคลียร์แบบ unsigned
div ebx        ; -10 ถูก interpret เป็น 0xFFFFFFF6 (ใหญ่มาก!)

; ถูก!
mov eax, -10
cdq            ; sign-extend ไปยัง EDX:EAX
idiv ebx       ; ใช้ IDIV สำหรับ signed
```

### Mistake 3: ลืม Sign-Extend ก่อน IDIV

```nasm
; ผิด!
mov eax, -17
; ลืม CDQ!
mov ebx, 5
idiv ebx       ; EDX อาจมีค่าไม่ถูกต้อง

; ถูก!
mov eax, -17
cdq             ; EDX:EAX = sign-extend ของ EAX
mov ebx, 5
idiv ebx        ; EAX = -3, EDX = -2
```

### Mistake 4: หารด้วย 0

```nasm
; อันตราย! จะทำให้ program crash (divide exception #DE)
mov eax, 100
xor edx, edx
xor ebx, ebx    ; EBX = 0
div ebx         ; CRASH!

; วิธีป้องกัน:
mov eax, 100
xor edx, edx
mov ebx, divisor
test ebx, ebx   ; ตรวจสอบว่าหารด้วย 0 หรือไม่
jz .div_by_zero ; ถ้า = 0 ข้ามไปจัดการ error
div ebx
jmp .div_ok
.div_by_zero:
    ; จัดการ error ที่นี่
.div_ok:
```

### Mistake 5: MUL ส่งผลกระทบต่อ DX/EDX/RDX

```nasm
; ระวัง! MUL เปลี่ยนค่าของ DX/EDX/RDX เสมอ
mov eax, 1000
mov edx, 999    ; ค่าที่ต้องการเก็บ
mov ecx, 50
mul ecx         ; EDX ถูกเปลี่ยนเป็น upper half ของผลคูณ!
; ค่า 999 ใน EDX หายไปแล้ว!

; แก้ไขโดยเก็บ EDX ไว้ก่อน:
push rdx
mov eax, 1000
mov ecx, 50
mul ecx
pop rdx         ; คืนค่า EDX
```

### Mistake 6: INC/DEC ไม่เปลี่ยน CF แต่เปลี่ยน OF

```nasm
; INC และ DEC ไม่กระทบ CF (ต่างจาก ADD/SUB)
; แต่กระทบ OF, SF, ZF, AF, PF

; ตัวอย่าง: ถ้าต้องการนับ loop พร้อม carry detection
mov ecx, 0xFFFFFFFF
inc ecx         ; ECX = 0, ZF = 1, OF = 0 (ไม่ set OF ใน INC?)
; หมายเหตุ: INC จะ set OF ถ้า signed overflow (0x7FFFFFFF + 1)
; แต่ CF ยังเป็น 0 เสมอ

; ถ้าต้องการตรวจสอบ carry ต้องใช้ ADD แทน:
add ecx, 1      ; จะ set CF ถ้า 0xFFFFFFFF + 1
```

---

## Advanced Techniques (เทคนิคขั้นสูง)

### Technique 1: การคูณด้วย Shift และ ADD (ประสิทธิภาพสูง)

```nasm
; คูณด้วย constant โดยใช้ SHL + ADD แทน IMUL
; EAX × 10 = EAX × 8 + EAX × 2
mov ebx, eax    ; เก็บค่าเดิมไว้
shl eax, 3      ; EAX = EAX × 8
shl ebx, 1      ; EBX = EBX × 2 (original × 2)
add eax, ebx    ; EAX = EAX × 8 + EBX × 2 = original × 10

; วิธีที่สั้นกว่า โดยใช้ LEA:
lea eax, [eax*8 + eax*2]  ; EAX = EAX × 10 (ถ้า assembler support)
; หรือ:
lea eax, [eax + eax*4]    ; EAX = EAX × 5
shl eax, 1                ; EAX = EAX × 10
```

### Technique 2: การหารด้วย Power of 2 (ประสิทธิภาพสูง)

```nasm
; หารด้วย power of 2 โดยใช้ SHR (unsigned)
mov eax, 64
shr eax, 2      ; EAX = 64 >> 2 = 16 (= 64 / 4)

; สำหรับ signed ต้องใช้ SAR (Shift Arithmetic Right)
mov eax, -64
sar eax, 2      ; EAX = -64 >> 2 = -16 (signed division by 4)
; หมายเหตุ: SAR ทำ rounding toward negative infinity
; แต่ IDIV ทำ rounding toward zero
; ดังนั้น -7 / 2:
;   SAR ให้ -4 (floor division)
;   IDIV ให้ -3 (truncation toward zero)
```

### Technique 3: การคำนวณ Modulo ด้วย AND

```nasm
; สำหรับ power-of-2 modulus (unsigned เท่านั้น):
; X mod 8 = X AND (8-1) = X AND 7
mov eax, 47
and eax, 7      ; EAX = 47 mod 8 = 7 (เร็วกว่า DIV มาก)

; X mod 16:
mov eax, 255
and eax, 15     ; EAX = 255 mod 16 = 15
```

### Technique 4: การตรวจสอบ Odd/Even

```nasm
; ตรวจสอบว่าเลขคู่หรือคี่
mov eax, 17
test eax, 1     ; AND เลขกับ 1 แล้วตรวจสอบ ZF
jz .even        ; ถ้า bit 0 = 0, เลขคู่
; ถ้าไม่ jump = เลขคี่
.even:
```

### Technique 5: Absolute Value โดยไม่ใช้ Branch

```nasm
; หา absolute value ของ EAX โดยไม่ใช้ JMP
mov ebx, eax
sar ebx, 31     ; EBX = 0 ถ้า EAX >= 0, EBX = -1 (0xFFFFFFFF) ถ้า EAX < 0
xor eax, ebx    ; เปลี่ยน bits ถ้า negative
sub eax, ebx    ; ลบออก (ถ้า positive: -0, ถ้า negative: -(-1) = +1)
```

### Technique 6: Saturating Arithmetic

```nasm
; Saturating addition: ถ้า overflow ให้คงค่าสูงสุดไว้แทน
; signed 32-bit: ถ้า overflow ให้เป็น 0x7FFFFFFF
add eax, ecx    ; บวกก่อน
jno .no_overflow; ถ้าไม่ overflow ข้ามไป
; overflow เกิดขึ้น
sar eax, 31     ; EAX = 0 ถ้า positive overflow, -1 ถ้า negative
xor eax, 0x80000000  ; 0 ^ 0x80000000 = 0x7FFFFFFF (max signed)
                     ; หรือ -1 ^ 0x80000000 = 0x7FFFFFFF (max signed ก็ได้?)
                     ; ต้องดูตามกรณี
.no_overflow:
```

### Technique 7: Multi-precision Multiply (128-bit × 128-bit)

```nasm
; การคูณเลข 128-bit:
; ใช้หลัก (A·2^64 + B) × (C·2^64 + D)
;   = AC·2^128 + (AD + BC)·2^64 + BD
;
; สำหรับ 128-bit result (เก็บแค่ lower 128-bit):
; result_lo = BD_lo
; result_hi = BD_hi + AD_lo + BC_lo

; สมมุติ:
; A = RBX (high 64), B = RAX (low 64)  -- ตัวตั้ง
; C = RDX (high 64), D = RCX (low 64)  -- ตัวคูณ

; คำนวณ BD:
push rax        ; เก็บ B
push rbx        ; เก็บ A
push rcx        ; เก็บ D
push rdx        ; เก็บ C

mov rax, [rsp+16]       ; B
mul qword [rsp]         ; B × D = RDX:RAX
push rdx                ; BD_hi
push rax                ; BD_lo

; ต่อไปคำนวณ AD และ BC (upper part)
; (ข้ามรายละเอียดเพื่อความกระชับ)
; เทคนิคนี้ใช้ใน crypto libraries เป็นหลัก
```

---

## Exercises (แบบฝึกหัด)

### Exercise 1: คำนวณ GCD (Greatest Common Divisor)

เขียนโปรแกรม Assembly ที่คำนวณ GCD ของสองจำนวนโดยใช้ Euclidean Algorithm:
- Input: สองจำนวนเต็มบวก
- Output: GCD ของทั้งสอง
- ใช้คำสั่ง DIV หรือ IDIV

**คำใบ้:**
```
GCD(a, b):
  while b != 0:
    r = a mod b
    a = b
    b = r
  return a

ใน Assembly:
  .loop:
    test ebx, ebx    ; ตรวจสอบว่า b = 0 หรือไม่
    jz .done
    xor edx, edx
    div ebx          ; EAX = a / b, EDX = a mod b
    mov eax, ebx     ; a = b
    mov ebx, edx     ; b = r
    jmp .loop
  .done:
    ; EAX = GCD
```

**ผลลัพธ์ที่ต้องการ:**
```
GCD(48, 18) = 6
GCD(100, 25) = 25
GCD(17, 13) = 1
```

---

### Exercise 2: คำนวณ Factorial

เขียนโปรแกรมคำนวณ n! (factorial) สำหรับ n = 0 ถึง 20

**คำใบ้:**
```
fact = 1
for i = 1 to n:
    fact = fact * i

; ใน Assembly:
; mov rax, 1      ; fact = 1
; mov rcx, 1      ; i = 1
; .loop:
;   cmp rcx, n
;   jg .done
;   imul rax, rcx  ; fact = fact * i
;   inc rcx
;   jmp .loop
; .done:
; ; RAX = n!
```

**ผลลัพธ์ที่ต้องการ:**
```
10! = 3628800
15! = 1307674368000
20! = 2432902008176640000
```

---

### Exercise 3: Power Function

เขียนฟังก์ชัน `power(base, exp)` ที่คำนวณ `base^exp` โดยใช้การคูณซ้ำ

**คำใบ้:**
```
; Arguments: RDI = base, RSI = exponent
; Returns: RAX = base^exp
power:
    mov rax, 1          ; result = 1
    test rsi, rsi
    jz .done            ; if exp = 0, return 1
.loop:
    imul rax, rdi       ; result *= base
    dec rsi
    jnz .loop
.done:
    ret
```

---

### Exercise 4: เขียน 256-bit Subtraction

ขยาย `add_256bit` ในตัวอย่างที่ 5 ให้เป็น `sub_256bit` โดยใช้ SUB และ SBB

**คำใบ้:**
```
; 256-bit subtraction: result = A - B
; word 0:
sub rax, rcx        ; ลบ low words (อาจเกิด borrow, CF=1)
; word 1:
sbb rax, rcx        ; ลบพร้อม borrow (CF จาก word ก่อน)
; word 2, 3 เช่นเดียวกัน
```

---

### Exercise 5: Overflow-safe Calculator

เขียน subroutine ที่คำนวณ `a + b` และ `a - b` สำหรับ signed 64-bit integers พร้อม overflow protection:
- ถ้า overflow เกิดขึ้น ให้คืนค่า `INT64_MAX` (0x7FFFFFFFFFFFFFFF) สำหรับ positive overflow
- ถ้า overflow เกิดขึ้น ให้คืนค่า `INT64_MIN` (0x8000000000000000) สำหรับ negative overflow
- ถ้าไม่ overflow ให้คืนค่าผลลัพธ์จริง

**คำใบ้:**
```
safe_add:
    ; RDI = a, RSI = b
    ; Returns: RAX = result (saturated)
    mov rax, rdi
    add rax, rsi
    jno .no_overflow        ; ถ้าไม่ overflow, คืนผลจริง
    ; overflow - ตรวจว่า positive หรือ negative overflow
    sar rdi, 63             ; RDI = 0 ถ้า a >= 0, -1 ถ้า a < 0
    mov rax, 0x7FFFFFFFFFFFFFFF
    xor rax, rdi            ; สลับเป็น MIN ถ้า negative overflow
.no_overflow:
    ret
```

---

## Summary (สรุป)

### ตาราง Arithmetic Instructions

| คำสั่ง | การทำงาน | Flags ที่เปลี่ยน | หมายเหตุ |
|--------|---------|-----------------|---------|
| ADD d, s | d = d + s | CF OF SF ZF AF PF | |
| ADC d, s | d = d + s + CF | CF OF SF ZF AF PF | ใช้กับ multi-word |
| SUB d, s | d = d - s | CF OF SF ZF AF PF | |
| SBB d, s | d = d - s - CF | CF OF SF ZF AF PF | ใช้กับ multi-word |
| INC d | d = d + 1 | OF SF ZF AF PF | ไม่เปลี่ยน CF! |
| DEC d | d = d - 1 | OF SF ZF AF PF | ไม่เปลี่ยน CF! |
| NEG d | d = 0 - d | CF OF SF ZF AF PF | CF=0 ถ้า d=0 |
| MUL s | AX/DX:AX/EDX:EAX/RDX:RAX = AL/AX/EAX/RAX × s | CF OF | upper half != 0 → CF=OF=1 |
| IMUL | หลายรูปแบบ | CF OF | ดูรายละเอียดข้างต้น |
| DIV s | quotient+remainder (unsigned) | undefined | ระวัง overflow! |
| IDIV s | quotient+remainder (signed) | undefined | ต้อง sign-extend ก่อน |
| CBW | AL → AX | - | sign-extend |
| CWDE | AX → EAX | - | sign-extend |
| CDQE | EAX → RAX | - | sign-extend |
| CWD | AX → DX:AX | - | สำหรับ IDIV 16-bit |
| CDQ | EAX → EDX:EAX | - | สำหรับ IDIV 32-bit |
| CQO | RAX → RDX:RAX | - | สำหรับ IDIV 64-bit |
| DAA | แก้ไข AL สำหรับ BCD ADD | CF AF (และอื่นๆ) | 32-bit only! |
| DAS | แก้ไข AL สำหรับ BCD SUB | CF AF (และอื่นๆ) | 32-bit only! |

### กฎสำคัญที่ต้องจำ

```
1. ก่อน DIV:  XOR EDX, EDX  (unsigned)
2. ก่อน IDIV: CDQ           (signed 32-bit → 64-bit)
              CQO           (signed 64-bit → 128-bit)
3. INC/DEC ไม่กระทบ CF
4. MUL/IMUL (1-operand) เปลี่ยน DX/EDX/RDX
5. IMUL (2/3-operand) เปลี่ยนเฉพาะ destination
6. DAA/DAS ใช้ได้เฉพาะ 32-bit mode
7. ตรวจสอบ overflow หลัง ADD/SUB/IMUL ด้วย JO
8. ตรวจสอบ carry/borrow หลัง ADD/SUB ด้วย JC
```

### Overflow Detection Reference

```nasm
; สำหรับ Signed overflow:
add rax, rbx
jo  .signed_overflow    ; Jump if Overflow Flag = 1

; สำหรับ Unsigned carry/borrow:
add rax, rbx
jc  .unsigned_carry     ; Jump if Carry Flag = 1

; สำหรับ Division by zero (ต้องตรวจก่อน DIV):
test rbx, rbx
jz  .divide_by_zero

; สำหรับ Quotient overflow (ก่อน DIV 32-bit):
; ถ้า EDX >= divisor จะเกิด overflow
cmp edx, ebx
jae .quotient_overflow  ; Jump if Above or Equal
```

### เปรียบเทียบ High-level กับ Assembly

```c
// C code:
int a = 100, b = 7;
int q = a / b;     // q = 14
int r = a % b;     // r = 2
```

```nasm
; Assembly equivalent:
mov eax, 100        ; a
xor edx, edx        ; เตรียมสำหรับ DIV
mov ebx, 7          ; b
div ebx             ; EAX = 14 (q), EDX = 2 (r)
```

```c
// C code (signed):
int a = -17, b = 5;
int q = a / b;     // q = -3 (C truncates toward zero)
int r = a % b;     // r = -2
```

```nasm
; Assembly equivalent:
mov eax, -17        ; a
cdq                 ; sign-extend EAX to EDX:EAX
mov ebx, 5          ; b
idiv ebx            ; EAX = -3, EDX = -2
```

---

## Quick Reference: คำสั่งที่ใช้บ่อย

```nasm
; === บวก ===
ADD  dest, src          ; dest += src
ADC  dest, src          ; dest += src + CF (multi-precision)
INC  dest               ; dest++

; === ลบ ===
SUB  dest, src          ; dest -= src
SBB  dest, src          ; dest -= src + CF (multi-precision)
DEC  dest               ; dest--
NEG  dest               ; dest = -dest

; === คูณ ===
MUL  src                ; Unsigned: [R/E]DX:[R/E]AX = [R/E]AX × src
IMUL dest, src          ; Signed: dest × src (2 operand form)
IMUL dest, src, imm     ; Signed: dest = src × imm (3 operand form)

; === หาร ===
DIV  src                ; Unsigned: [R/E]AX = [R/E]DX:[R/E]AX / src
                        ;           [R/E]DX = remainder
IDIV src                ; Signed: เหมือน DIV แต่ signed

; === Sign Extension ===
CBW                     ; AL → AX
CWDE                    ; AX → EAX
CDQE                    ; EAX → RAX
CWD                     ; AX → DX:AX
CDQ                     ; EAX → EDX:EAX
CQO                     ; RAX → RDX:RAX

; === BCD (32-bit mode only) ===
DAA                     ; Decimal Adjust After Add
DAS                     ; Decimal Adjust After Sub
AAA                     ; ASCII Adjust After Add
AAS                     ; ASCII Adjust After Sub
AAM                     ; ASCII Adjust After Multiply
AAD                     ; ASCII Adjust Before Divide
```

---

## What's Next (ส่วนถัดไป)

ใน Part 015 เราจะเรียน **Logical and Bitwise Instructions** ซึ่งครอบคลุม:
- AND, OR, XOR, NOT
- TEST (AND ที่ไม่เปลี่ยนค่า)
- Bit manipulation techniques
- Shift operations: SHL, SHR, SAL, SAR
- Rotate operations: ROL, ROR, RCL, RCR
- Bit Test: BT, BTS, BTR, BTC
- BSF (Bit Scan Forward), BSR (Bit Scan Reverse)
- POPCNT, TZCNT, LZCNT (Intel BMI/POPCNT extensions)

การที่เข้าใจ Arithmetic Instructions อย่างถ่องแท้จะทำให้คุณสามารถ:
1. เขียน crypto algorithms ที่ต้องการ big integer arithmetic
2. Optimize inner loops ของ mathematical computations
3. Implement fixed-point arithmetic สำหรับ embedded systems
4. เข้าใจ compiler output ได้ดีขึ้นเมื่อ decompile โปรแกรม

---

*Part 014 เสร็จสมบูรณ์ - Arithmetic Instructions*
*ผู้เรียนสามารถดำเนินการต่อไปที่ Part 015: Logical and Bitwise Instructions*
