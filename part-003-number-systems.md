# Part 003: ระบบตัวเลขและการแทนข้อมูล (Number Systems and Data Representation)

**เวลาที่ใช้เรียน:** 4-6 ชั่วโมง  
**ระดับ:** พื้นฐาน-กลาง  
**ความต่อเนื่อง:** ต่อจาก Part 002 (สถาปัตยกรรมและ Registers)

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:

- แปลงตัวเลขระหว่างระบบ Binary, Octal, Decimal, Hexadecimal ได้อย่างคล่องแคล่ว
- อธิบายและใช้งาน Two's Complement สำหรับจำนวนเต็มที่มีเครื่องหมาย
- เข้าใจ IEEE 754 floating-point representation และข้อจำกัดของมัน
- ทำ Bitwise operations (AND, OR, XOR, NOT, Shifts) ใน Assembly
- เขียน code แปลงระบบตัวเลขใน NASM x86/x86-64
- หลีกเลี่ยง overflow/underflow bugs ที่พบบ่อยใน low-level programming
- ทำงานกับ ASCII และ Unicode characters ใน Assembly
- ใช้ bit manipulation tricks เพื่อเพิ่มประสิทธิภาพ code

---

## 1. Binary System (ระบบเลขฐาน 2)

### 1.1 ทฤษฎีพื้นฐาน

ระบบ Binary เป็นรากฐานของการทำงานของคอมพิวเตอร์ทุกเครื่อง ทรานซิสเตอร์ในชิป CPU มีสถานะเพียง 2 อย่าง คือ เปิด (1) และ ปิด (0) ดังนั้น Binary จึงเป็นภาษาธรรมชาติของฮาร์ดแวร์

**ค่าตำแหน่ง (Positional Value) ของ Binary:**

```
ตำแหน่ง:  7    6    5    4    3    2    1    0
ค่า:      128   64   32   16    8    4    2    1
          2^7  2^6  2^5  2^4  2^3  2^2  2^1  2^0
```

**ตัวอย่างการแปลง Binary → Decimal:**
```
10110101 (binary)
= 1×2^7 + 0×2^6 + 1×2^5 + 1×2^4 + 0×2^3 + 1×2^2 + 0×2^1 + 1×2^0
= 128    +  0   +  32   +  16   +   0   +   4   +   0   +   1
= 181 (decimal)
```

**ตัวอย่างการแปลง Decimal → Binary:**
```
181 ÷ 2 = 90 เศษ 1   ← LSB (Least Significant Bit)
 90 ÷ 2 = 45 เศษ 0
 45 ÷ 2 = 22 เศษ 1
 22 ÷ 2 = 11 เศษ 0
 11 ÷ 2 =  5 เศษ 1
  5 ÷ 2 =  2 เศษ 1
  2 ÷ 2 =  1 เศษ 0
  1 ÷ 2 =  0 เศษ 1   ← MSB (Most Significant Bit)

อ่านจากล่างขึ้นบน: 10110101
```

### 1.2 ขนาดข้อมูล (Data Sizes)

| ชื่อ | ขนาด | ช่วงค่า (Unsigned) | ช่วงค่า (Signed) |
|------|------|-------------------|-----------------|
| Bit | 1 bit | 0-1 | - |
| Nibble | 4 bits | 0-15 | -8 ถึง 7 |
| Byte | 8 bits | 0-255 | -128 ถึง 127 |
| Word | 16 bits | 0-65,535 | -32,768 ถึง 32,767 |
| Dword | 32 bits | 0-4,294,967,295 | -2,147,483,648 ถึง 2,147,483,647 |
| Qword | 64 bits | 0-18,446,744,073,709,551,615 | ±9.2×10^18 |

---

## 2. Hexadecimal System (ระบบเลขฐาน 16)

### 2.1 ทำไม Hex ถึงสำคัญใน Assembly

Hexadecimal เป็น "ภาษากลาง" ระหว่างมนุษย์และ binary เพราะ 1 ตัวเลข hex = 4 bits (1 nibble) พอดี ทำให้แปลงระหว่าง hex และ binary ได้ง่ายมาก

**ตารางแปลง Hex ↔ Binary ↔ Decimal:**

| Hex | Binary | Decimal |
|-----|--------|---------|
| 0   | 0000   | 0       |
| 1   | 0001   | 1       |
| 2   | 0010   | 2       |
| 3   | 0011   | 3       |
| 4   | 0100   | 4       |
| 5   | 0101   | 5       |
| 6   | 0110   | 6       |
| 7   | 0111   | 7       |
| 8   | 1000   | 8       |
| 9   | 1001   | 9       |
| A   | 1010   | 10      |
| B   | 1011   | 11      |
| C   | 1100   | 12      |
| D   | 1101   | 13      |
| E   | 1110   | 14      |
| F   | 1111   | 15      |

**ตัวอย่าง: Memory Address 0xDEADBEEF**
```
D    E    A    D    B    E    E    F
1101 1110 1010 1101 1011 1110 1110 1111

= 11011110101011011011111011101111 (binary)
= 3,735,928,559 (decimal)
```

### 2.2 การเขียน Hex ใน NASM

```nasm
; การเขียน Hexadecimal ใน NASM syntax
mov eax, 0xFF        ; 255 decimal
mov ebx, 0xDEAD      ; 57005 decimal
mov ecx, 0x1000      ; 4096 decimal (ขนาด 1 page)
mov edx, 0xFFFFFFFF  ; 4294967295 (ค่าสูงสุด 32-bit unsigned)

; การเขียน Binary ใน NASM (ใช้ prefix 0b)
mov al, 0b10110101   ; = 0xB5 = 181 decimal

; การเขียน Octal ใน NASM (ใช้ prefix 0o หรือ 0q)
mov al, 0o377        ; = 0xFF = 255 decimal
```

---

## 3. Octal System (ระบบเลขฐาน 8)

### 3.1 การใช้ Octal ใน Assembly และ Unix

Octal พบบ่อยใน Unix/Linux permissions และบางครั้งใน embedded systems เพราะ 3 bits = 1 octal digit

```
Binary:  000 001 010 011 100 101 110 111
Octal:    0   1   2   3   4   5   6   7
```

**ตัวอย่าง Unix Permission:**
```
chmod 755 file.sh
= 111 101 101 (binary)
= rwx r-x r-x

7 = 111 = rwx (owner: read, write, execute)
5 = 101 = r-x (group: read, execute)
5 = 101 = r-x (others: read, execute)
```

**ตัวอย่าง Code ใช้ Octal สำหรับ System Call:**
```nasm
; Linux system call: open file with permissions 0644
; O_CREAT | O_WRONLY = 0o101 = 65
section .data
    filename db 'output.txt', 0

section .text
    global _start

_start:
    ; sys_open(filename, O_WRONLY|O_CREAT|O_TRUNC, 0644)
    mov rax, 2          ; syscall number: open
    mov rdi, filename   ; pointer to filename
    mov rsi, 577        ; 0o1101 = O_WRONLY|O_CREAT|O_TRUNC
    mov rdx, 420        ; 0o644 = permission bits
    syscall
```

---

## 4. BCD (Binary Coded Decimal)

### 4.1 อธิบาย BCD

BCD เก็บตัวเลข decimal แต่ละหลักใน 4 bits แยกกัน มีประโยชน์เมื่อต้องการแสดงผลตัวเลขโดยไม่ต้องแปลง

```
Decimal 93 ใน BCD:
9        3
1001   0011
```

**เปรียบเทียบ BCD กับ Binary:**
```
Decimal: 99
Binary:  01100011  (1 byte = 0-255)
BCD:     10011001  (ใช้ 2 nibbles สำหรับ 2 หลัก = 0-99 เท่านั้น)
```

### 4.2 ตัวอย่าง BCD ใน x86 Assembly

x86 มี instruction พิเศษสำหรับ BCD:
- `DAA` (Decimal Adjust for Addition)
- `DAS` (Decimal Adjust for Subtraction)
- `AAA` (ASCII Adjust for Addition)
- `AAS` (ASCII Adjust for Subtraction)
- `AAM` (ASCII Adjust for Multiplication)
- `AAD` (ASCII Adjust for Division)

```nasm
; ตัวอย่าง BCD Addition ใน x86 (32-bit)
; บวก BCD 39 + 28 = 67
section .text
    global _start

_start:
    mov al, 0x39        ; BCD สำหรับ 39 (3=0x3, 9=0x9)
    add al, 0x28        ; บวก BCD 28
    ; al = 0x61 (ยังไม่ถูก BCD)
    daa                 ; ปรับ al ให้เป็น BCD ที่ถูกต้อง
    ; al = 0x67 (BCD สำหรับ 67)
    ; DAA ตรวจสอบ: ถ้า lower nibble > 9 หรือ AF set → บวก 6
    ; ถ้า upper nibble > 9 หรือ CF set → บวก 0x60

    ; แสดงผล: exit
    mov eax, 1          ; syscall: exit
    xor ebx, ebx        ; exit code 0
    int 0x80
```

---

## 5. Two's Complement (จำนวนเต็มที่มีเครื่องหมาย)

### 5.1 ทำไมต้องใช้ Two's Complement?

Two's Complement เป็น standard วิธีเก็บจำนวนเต็มลบในคอมพิวเตอร์ เพราะ:
1. มี zero เพียงค่าเดียว (ไม่มี +0 และ -0)
2. การบวก/ลบทำได้โดยใช้ hardware เดิม ไม่ต้องเพิ่ม circuit พิเศษ

### 5.2 วิธีแปลงเป็น Two's Complement

**วิธี 1: Invert แล้วบวก 1**
```
แปลง -5 เป็น 8-bit two's complement:

1. เริ่มจาก 5 (positive)  → 00000101
2. Invert ทุก bit (NOT)   → 11111010
3. บวก 1                  → 11111011

ดังนั้น -5 = 11111011 ใน 8-bit two's complement
```

**วิธี 2: คำนวณตรง**
```
-5 = 2^8 - 5 = 256 - 5 = 251 = 0xFB = 11111011
```

**ตรวจสอบ:**
```
11111011
= -128 + 64 + 32 + 16 + 0 + 8 + 2 + 1
= -128 + 123
= -5 ✓

(bit 7 มีค่า -128 ใน signed interpretation)
```

### 5.3 ตาราง Two's Complement 8-bit

| Binary | Unsigned | Signed |
|--------|----------|--------|
| 01111111 | 127 | +127 |
| 01111110 | 126 | +126 |
| 00000001 | 1 | +1 |
| 00000000 | 0 | 0 |
| 11111111 | 255 | -1 |
| 11111110 | 254 | -2 |
| 10000001 | 129 | -127 |
| 10000000 | 128 | -128 |

### 5.4 Code ตัวอย่าง: การทำงานกับ Signed Numbers

```nasm
; ไฟล์: signed_numbers.asm
; compile: nasm -f elf64 signed_numbers.asm -o signed_numbers.o
;          ld signed_numbers.o -o signed_numbers

section .data
    msg_pos db 'Number is positive', 0xA, 0
    msg_neg db 'Number is negative', 0xA, 0
    msg_zero db 'Number is zero', 0xA, 0
    len_pos equ $ - msg_pos
    len_neg equ $ - msg_neg
    len_zero equ $ - msg_zero

section .text
    global _start

_start:
    ; ทดสอบ signed number -42
    mov rax, -42            ; โหลด -42 (two's complement)
    ; rax = 0xFFFFFFFFFFFFFFD6

    ; วิธีที่ 1: ใช้ TEST instruction
    test rax, rax           ; AND rax กับตัวเอง (ไม่เปลี่ยนค่า แต่ set flags)
    js negative_handler     ; jump ถ้า Sign Flag set (negative)
    jz zero_handler         ; jump ถ้า Zero Flag set

    ; positive
    mov rax, 1              ; syscall: write
    mov rdi, 1              ; fd: stdout
    mov rsi, msg_pos        ; message
    mov rdx, len_pos        ; length
    syscall
    jmp done

negative_handler:
    ; วิธีที่ 2: NEG instruction (negate = เปลี่ยนเครื่องหมาย)
    ; neg rax              ; rax = 42 (ค่า absolute)
    
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_neg
    mov rdx, len_neg
    syscall
    jmp done

zero_handler:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_zero
    mov rdx, len_zero
    syscall

done:
    mov rax, 60             ; syscall: exit
    xor rdi, rdi            ; exit code 0
    syscall
```

---

## 6. Overflow และ Underflow

### 6.1 ความหมายของ Overflow

Overflow เกิดเมื่อผลลัพธ์ของการคำนวณใหญ่เกินกว่าที่ register จะเก็บได้

**สำหรับ Unsigned:**
```
255 + 1 = 256 → ไม่จุใน 8 bits → Overflow (CF=1)
Carry Flag (CF) ถูก set

  11111111  (255)
+ 00000001  (1)
----------
100000000  ← bit ที่ 8 ล้นออกมา
  00000000  (0) ← ค่าที่เหลือใน 8 bits
```

**สำหรับ Signed:**
```
+127 + 1 = +128 → ไม่จุใน 8-bit signed → Overflow (OF=1)
Overflow Flag (OF) ถูก set

  01111111  (+127)
+ 00000001  (+1)
----------
  10000000  (-128) ← ค่าผิด! บวก positive แต่ได้ negative
```

### 6.2 วิธีตรวจจับ Overflow

```nasm
; ไฟล์: overflow_check.asm
; แสดงการตรวจ overflow ใน x86-64

section .data
    msg_overflow db 'Overflow detected!', 0xA, 0
    msg_ok       db 'No overflow', 0xA, 0

section .text
    global _start

_start:
    ; ===== Test 1: Unsigned Overflow =====
    mov al, 255         ; al = 0xFF (max unsigned byte)
    add al, 1           ; เพิ่ม 1 → overflow!
    ; al = 0 (ล้น), CF = 1

    jc unsigned_overflow ; jump if Carry Flag set
    jmp test2

unsigned_overflow:
    ; จัดการ unsigned overflow ที่นี่
    ; ตัวอย่าง: ใช้ register ใหญ่กว่า
    movzx eax, al       ; zero-extend al เป็น eax
    ; ตอนนี้ eax = 0 ซึ่งผิด

    ; วิธีที่ถูกต้อง: ใช้ register ใหญ่กว่าตั้งแต่แรก
    mov eax, 255
    add eax, 1          ; eax = 256, no overflow in 32-bit

test2:
    ; ===== Test 2: Signed Overflow =====
    mov al, 127         ; al = 0x7F (max positive signed byte)
    add al, 1           ; เพิ่ม 1 → signed overflow!
    ; al = -128 (0x80), OF = 1

    jo signed_overflow  ; jump if Overflow Flag set
    jmp no_overflow

signed_overflow:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_overflow
    mov rdx, 18
    syscall
    jmp done

no_overflow:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_ok
    mov rdx, 11
    syscall

done:
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 6.3 Saturating Arithmetic (ป้องกัน Overflow)

```nasm
; Saturating Add: ถ้า overflow → clamp ไว้ที่ค่าสูงสุด
; saturating_add(al, bl) → al
saturating_add_byte:
    add al, bl          ; บวกปกติ
    jnc .done           ; ถ้าไม่ carry → ok
    mov al, 0xFF        ; ถ้า carry → set เป็น 255 (max)
.done:
    ret
```

---

## 7. Floating-Point: IEEE 754

### 7.1 โครงสร้าง IEEE 754 Single Precision (32-bit)

```
Bit 31    Bits 30-23      Bits 22-0
  S    EEEEEEEE    MMMMMMMMMMMMMMMMMMMMMMM
Sign   Exponent        Mantissa (Fraction)
(1b)    (8b)              (23b)
```

**สูตร:**
```
Value = (-1)^S × 1.Mantissa × 2^(Exponent - 127)
```

**ตัวอย่าง: เลข 3.14**
```
3.14 ≈ 3.14159...

1. แปลง 3 เป็น binary: 11
2. แปลง 0.14 เป็น binary:
   0.14 × 2 = 0.28 → 0
   0.28 × 2 = 0.56 → 0
   0.56 × 2 = 1.12 → 1
   0.12 × 2 = 0.24 → 0
   0.24 × 2 = 0.48 → 0
   0.48 × 2 = 0.96 → 0
   0.96 × 2 = 1.92 → 1
   ...
   ≈ 0.00100011...

3. รวม: 11.00100011...
4. Normalize: 1.100100011... × 2^1

S = 0 (positive)
Exponent = 1 + 127 = 128 = 10000000
Mantissa = 10010001111010111000011...

IEEE 754: 0 10000000 10010001111010111000011
= 0x40490FDB
```

### 7.2 Special Values ใน IEEE 754

| Value | Sign | Exponent | Mantissa |
|-------|------|----------|----------|
| +0 | 0 | 00000000 | 000...000 |
| -0 | 1 | 00000000 | 000...000 |
| +∞ | 0 | 11111111 | 000...000 |
| -∞ | 1 | 11111111 | 000...000 |
| NaN | × | 11111111 | non-zero |
| Denormal | × | 00000000 | non-zero |

### 7.3 Code ตัวอย่าง: Float ใน x86 Assembly

```nasm
; ไฟล์: float_demo.asm
; ใช้ SSE2 สำหรับ floating point
; compile: nasm -f elf64 float_demo.asm -o float_demo.o
;          ld float_demo.o -o float_demo

section .data
    ; IEEE 754 single precision constants
    pi_f32    dd 3.14159265     ; float (32-bit)
    e_f32     dd 2.71828182
    two_pi    dd 6.28318530

    ; Double precision
    pi_f64    dq 3.14159265358979  ; double (64-bit)

section .text
    global _start

_start:
    ; โหลด float เข้า XMM register
    movss xmm0, [pi_f32]    ; โหลด float 32-bit เข้า xmm0
    movss xmm1, [e_f32]     ; โหลด float 32-bit เข้า xmm1

    ; บวก float สองตัว
    addss xmm2, xmm0        ; xmm2 = 0 + pi ≈ 3.14159
    addss xmm2, xmm1        ; xmm2 = pi + e ≈ 5.85987

    ; คูณ float
    movss xmm3, [pi_f32]
    mulss xmm3, xmm3        ; xmm3 = pi^2 ≈ 9.8696

    ; หาร float
    movss xmm4, [two_pi]
    movss xmm5, [e_f32]
    divss xmm4, xmm5        ; xmm4 = 2π/e ≈ 2.309

    ; Double precision operations
    movsd xmm6, [pi_f64]    ; โหลด double 64-bit
    sqrtsd xmm7, xmm6       ; xmm7 = √π ≈ 1.7724

    ; Comparison: ใช้ UCOMISS (Unordered Compare Scalar Single-precision)
    movss xmm8, [pi_f32]
    movss xmm9, [e_f32]
    ucomiss xmm8, xmm9      ; เปรียบเทียบ pi กับ e
    ja pi_greater           ; jump if above (pi > e)
    jb e_greater            ; jump if below
    je equal                ; jump if equal

pi_greater:
    ; pi > e (ถูกต้อง!)
    jmp done

e_greater:
    jmp done

equal:
    jmp done

done:
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 7.4 Double Precision (64-bit)

```
Bit 63    Bits 62-52         Bits 51-0
  S    EEEEEEEEEEE    MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
Sign    Exponent (11b)              Mantissa (52b)
```

```
Precision comparison:
Single (32-bit): ~7 decimal digits
Double (64-bit): ~15-16 decimal digits
Extended (80-bit, x87): ~18-19 decimal digits
```

---

## 8. Fixed-Point Arithmetic

### 8.1 ทำไมต้องใช้ Fixed-Point?

Fixed-point ใช้เมื่อ:
- ไม่มี FPU hardware
- ต้องการความเร็วสูงกว่า float
- ต้องการ deterministic results (เช่น เกม, crypto)

### 8.2 Q Format

**Q16.16 Format:** 16 bits สำหรับส่วนจำนวนเต็ม, 16 bits สำหรับทศนิยม

```
Bit 31-16: Integer part
Bit 15-0:  Fractional part

ค่า = (stored_integer) / 2^16 = (stored_integer) / 65536
```

**ตัวอย่าง:**
```
เก็บ 3.14 ใน Q16.16:
3.14 × 65536 = 205,887 ≈ 205,887
stored = 0x0003_23D7

เมื่อแสดงผล: 205887 / 65536 ≈ 3.1400146
```

### 8.3 Code ตัวอย่าง: Fixed-Point

```nasm
; ไฟล์: fixed_point.asm
; Q16.16 fixed-point arithmetic

section .data
    ; เก็บค่า 3.14 ใน Q16.16
    ; 3.14 * 65536 = 205887.04 ≈ 205887 = 0x000323D7
    pi_fixed  dd 205887

    ; เก็บค่า 2.0 ใน Q16.16
    ; 2.0 * 65536 = 131072 = 0x00020000
    two_fixed dd 131072

section .text
    global _start

_start:
    ; ===== Fixed-Point Addition (ทำได้ตรงๆ) =====
    mov eax, [pi_fixed]     ; eax = pi (3.14 * 65536)
    mov ebx, [two_fixed]    ; ebx = 2.0 * 65536
    add eax, ebx            ; eax = 5.14 * 65536
    ; ผลลัพธ์ = 205887 + 131072 = 336959

    ; ===== Fixed-Point Multiplication (ต้อง shift) =====
    ; a * b ใน Q16.16: (a * b) >> 16
    mov eax, [pi_fixed]     ; eax = pi * 65536
    mov ebx, [pi_fixed]     ; ebx = pi * 65536
    imul ebx                ; edx:eax = (pi * 65536)^2
    ; ผลลัพธ์อยู่ใน 64-bit edx:eax
    shrd eax, edx, 16       ; shift right 16 bits
    sar edx, 16             ; arithmetic shift
    ; eax = pi^2 * 65536 ≈ 648736

    ; ===== แปลงกลับเป็น integer part =====
    mov eax, 648736         ; pi^2 * 65536
    sar eax, 16             ; shift right 16 → integer part
    ; eax = 9 (integer part of pi^2 ≈ 9.8696)

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 9. ASCII และ Unicode ใน Assembly

### 9.1 ASCII Table

ASCII ใช้ 7 bits (0-127) เพื่อเข้ารหัสตัวอักษร

```
0-31:   Control characters (TAB=9, LF=10, CR=13)
32:     Space
48-57:  '0'-'9' (0x30-0x39)
65-90:  'A'-'Z' (0x41-0x5A)
97-122: 'a'-'z' (0x61-0x7A)
```

**เคล็ดลับสำคัญ:**
```
'A' = 0x41 = 65
'a' = 0x61 = 97
'a' - 'A' = 32 = 0x20

ดังนั้น: การแปลง uppercase → lowercase คือการ OR กับ 0x20
         การแปลง lowercase → uppercase คือการ AND กับ 0xDF
```

### 9.2 Code ตัวอย่าง: Case Conversion

```nasm
; ไฟล์: case_convert.asm
; แปลงข้อความ uppercase ↔ lowercase
; compile: nasm -f elf64 case_convert.asm -o case_convert.o
;          ld case_convert.o -o case_convert
; ./case_convert (แสดง hello world จาก HELLO WORLD)

section .data
    text    db 'HELLO WORLD', 0xA  ; ข้อความต้นฉบับ
    length  equ $ - text

section .bss
    lower   resb 12                ; buffer สำหรับ lowercase

section .text
    global _start

_start:
    ; แปลง uppercase → lowercase
    mov rcx, length - 1    ; จำนวนตัวอักษร (ไม่นับ newline)
    lea rsi, [text]        ; pointer ไปยัง source
    lea rdi, [lower]       ; pointer ไปยัง destination

convert_loop:
    cmp rcx, 0             ; ตรวจสอบว่าครบแล้วหรือยัง
    jle write_output       ; ถ้าครบ → ไปเขียน output

    mov al, [rsi]          ; โหลดตัวอักษร 1 ตัว
    cmp al, 0xA            ; ตรวจสอบ newline
    je copy_char           ; ถ้า newline → copy ตรงๆ

    cmp al, 'A'            ; ตรวจสอบว่าเป็น uppercase หรือไม่
    jl copy_char           ; ถ้าน้อยกว่า 'A' → ไม่ใช่ uppercase
    cmp al, 'Z'
    jg copy_char           ; ถ้ามากกว่า 'Z' → ไม่ใช่ uppercase

    or al, 0x20            ; แปลงเป็น lowercase (OR กับ 0x20)

copy_char:
    mov [rdi], al          ; เขียนไปยัง destination
    inc rsi                ; เลื่อน source pointer
    inc rdi                ; เลื่อน destination pointer
    dec rcx                ; ลด counter
    jmp convert_loop

write_output:
    ; เพิ่ม newline
    mov byte [rdi], 0xA

    ; เขียน output ด้วย sys_write
    mov rax, 1             ; syscall: write
    mov rdi, 1             ; fd: stdout
    lea rsi, [lower]       ; ข้อความที่แปลงแล้ว
    mov rdx, length        ; ความยาว
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

### 9.3 Unicode และ UTF-8

UTF-8 ใช้ 1-4 bytes ต่อ character:
```
U+0000   ถึง U+007F:   0xxxxxxx                 (1 byte, compatible ASCII)
U+0080   ถึง U+07FF:   110xxxxx 10xxxxxx         (2 bytes)
U+0800   ถึง U+FFFF:   1110xxxx 10xxxxxx 10xxxxxx (3 bytes)
U+10000  ถึง U+10FFFF: 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx (4 bytes)
```

**ตัวอย่าง: ตัวอักษร 'ก' (U+0E01)**
```
U+0E01 = 0000 1110 0000 0001

UTF-8 encoding (3 bytes):
1110xxxx 10xxxxxx 10xxxxxx
  0000     111000    000001

= 11100000 10111000 10000001
= 0xE0 0xB8 0x81
```

---

## 10. Bitwise Operations

### 10.1 AND Operation

AND ใช้สำหรับ:
- Masking (ดึงบาง bits)
- Clearing bits
- ตรวจสอบ bits

```
Truth Table:
A AND B = C
0 AND 0 = 0
0 AND 1 = 0
1 AND 0 = 0
1 AND 1 = 1

ตัวอย่าง: ดึงเฉพาะ lower nibble
  10110101  (0xB5 = 181)
& 00001111  (0x0F = mask)
----------
  00000101  (0x05 = 5) ← lower nibble
```

### 10.2 OR Operation

OR ใช้สำหรับ:
- Setting bits
- Combining bit fields

```
Truth Table:
A OR B = C
0 OR 0 = 0
0 OR 1 = 1
1 OR 0 = 1
1 OR 1 = 1

ตัวอย่าง: Set bit 3
  10100000  (0xA0)
| 00001000  (0x08 = bit 3 mask)
----------
  10101000  (0xA8) ← bit 3 ถูก set
```

### 10.3 XOR Operation

XOR ใช้สำหรับ:
- Toggle bits
- Zero register อย่างมีประสิทธิภาพ (xor eax, eax)
- Simple encryption
- Swap ค่าโดยไม่ใช้ temp variable

```
Truth Table:
A XOR B = C
0 XOR 0 = 0
0 XOR 1 = 1
1 XOR 0 = 1
1 XOR 1 = 0

เคล็ดลับ: A XOR A = 0 (ใช้ zero register)
          A XOR 0 = A (ไม่เปลี่ยน)

XOR Swap:
mov eax, 5      ; eax = 5
mov ebx, 3      ; ebx = 3
xor eax, ebx    ; eax = 5 XOR 3 = 6
xor ebx, eax    ; ebx = 3 XOR 6 = 5
xor eax, ebx    ; eax = 6 XOR 5 = 3
; ตอนนี้ eax=3, ebx=5 (swap สำเร็จ)
```

### 10.4 NOT Operation

```
NOT A = complement of A

NOT 10110101 = 01001010
NOT 0xFF = 0x00
NOT 0x00 = 0xFF
```

### 10.5 Shift Operations

```
SHL (Shift Left Logical):  เพิ่มค่า ×2 ต่อ 1 bit shift
SHR (Shift Right Logical): ลดค่า ÷2 ต่อ 1 bit shift (unsigned)
SAR (Shift Right Arithmetic): ลดค่า ÷2 (signed, ขยาย sign bit)
ROL (Rotate Left):  หมุน bits ไปซ้าย
ROR (Rotate Right): หมุน bits ไปขวา

ตัวอย่าง SHL:
00000101 SHL 1 = 00001010  (5 × 2 = 10)
00000101 SHL 2 = 00010100  (5 × 4 = 20)
00000101 SHL 3 = 00101000  (5 × 8 = 40)

ตัวอย่าง SAR (signed):
10001000 SAR 1 = 11000100  (-120 ÷ 2 = -60)
หมายเหตุ: SAR ขยาย sign bit (1) ไปทางซ้าย
```

---

## 11. Code ครบชุด: Bitwise Operations

```nasm
; ไฟล์: bitwise_ops.asm
; ตัวอย่างการใช้งาน bitwise operations ทั้งหมด
; compile: nasm -f elf64 bitwise_ops.asm -o bitwise_ops.o
;          ld bitwise_ops.o -o bitwise_ops

section .data
    ; ข้อความสำหรับแสดงผล
    nl      db 0xA          ; newline

section .text
    global _start

_start:
    ; ===== AND Operations =====

    ; 1. ตรวจสอบ even/odd (AND กับ 1)
    mov eax, 42             ; ตัวเลขที่ต้องการตรวจสอบ
    and eax, 1              ; AND กับ mask 0x01
    ; ถ้า eax = 0 → even, ถ้า eax = 1 → odd

    ; 2. ดึง lower byte จาก 32-bit
    mov eax, 0xABCD1234
    and eax, 0xFF           ; eax = 0x34 = 52

    ; 3. Align address (round down ไป multiple of 16)
    mov rax, 0x12345678ABCD ; address
    and rax, -16            ; AND กับ 0xFFFF...FFF0
    ; rax = 0x12345678ABC0  (aligned ไป 16 bytes)
    ; NOTE: -16 ใน two's complement = 0xFFFFFFFFFFFFFFF0

    ; ===== OR Operations =====

    ; 4. Set specific bit
    mov eax, 0b10100000     ; eax = 0xA0
    or eax, 1 << 3          ; set bit 3 (value = 8)
    ; eax = 0b10101000 = 0xA8

    ; 5. ใส่ไว้ใน flags register (เช่น set sign flag manually via)
    mov eax, 0              ; เตรียม flag byte
    or eax, 0x80            ; set bit 7 (sign flag equivalent)

    ; ===== XOR Operations =====

    ; 6. Zero register (วิธีที่เร็วที่สุด)
    xor eax, eax            ; eax = 0 (1 instruction, เร็วกว่า mov eax, 0)
    xor rbx, rbx            ; rbx = 0

    ; 7. Toggle bit
    mov eax, 0b11110000
    xor eax, 1 << 4         ; toggle bit 4
    ; eax = 0b11100000

    ; 8. Simple XOR encryption
    mov al, 'H'             ; 'H' = 0x48
    xor al, 0x20            ; XOR กับ key 0x20
    ; al = 0x68 = 'h' (encrypted? จริงๆคือแค่ case change)
    xor al, 0x20            ; XOR อีกครั้งเพื่อ decrypt
    ; al = 0x48 = 'H' อีกครั้ง

    ; ===== NOT Operation =====

    ; 9. Complement
    mov eax, 0xAAAAAAAA     ; alternating bits
    not eax                 ; eax = 0x55555555 (bits สลับ)

    ; ===== SHIFT Operations =====

    ; 10. คูณด้วย powers of 2 (เร็วกว่า MUL)
    mov eax, 7
    shl eax, 3              ; eax = 7 * 8 = 56 (shift left 3 = ×2^3 = ×8)

    ; 11. หารด้วย powers of 2
    mov eax, 100
    shr eax, 2              ; eax = 100 / 4 = 25 (shift right 2 = ÷2^2 = ÷4)

    ; 12. Signed divide by 2 (arithmetic shift)
    mov eax, -100           ; -100 two's complement
    sar eax, 1              ; eax = -50 (signed divide by 2)
    ; SHR จะให้ผลผิด! ต้องใช้ SAR กับ signed numbers

    ; 13. Extract bit field (bits 8-11)
    mov eax, 0xABCD         ; ตัวอย่างค่า
    shr eax, 8              ; เลื่อน bits 8-11 ไปที่ bits 0-3
    and eax, 0xF            ; mask เอาเฉพาะ 4 bits ล่าง
    ; eax = 0xC = 12

    ; 14. ROL (Rotate Left)
    mov al, 0b11000001      ; 0xC1
    rol al, 1               ; al = 0b10000011 = 0x83
    ; bit ที่ล้นออกไปทางซ้าย กลับมาทางขวา

    ; 15. ROR (Rotate Right)
    mov al, 0b10000011      ; 0x83
    ror al, 1               ; al = 0b11000001 = 0xC1

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 12. Bit Manipulation Tricks

### 12.1 เทคนิคที่ใช้บ่อยใน Assembly

```nasm
; ไฟล์: bit_tricks.asm
; รวมเทคนิค bit manipulation ที่มีประโยชน์

section .text
    global _start

_start:

    ; ===== Trick 1: ตรวจสอบ Power of 2 =====
    ; n เป็น power of 2 ถ้า (n AND (n-1)) = 0
    mov eax, 64             ; 64 = 2^6
    mov ebx, eax
    dec ebx                 ; ebx = 63
    and eax, ebx            ; eax = 64 AND 63 = 0 → power of 2!

    mov eax, 65             ; 65 ไม่ใช่ power of 2
    mov ebx, eax
    dec ebx                 ; ebx = 64
    and eax, ebx            ; eax = 65 AND 64 = 64 ≠ 0 → not power of 2

    ; ===== Trick 2: หา lowest set bit =====
    ; isolate_lowest_bit = n AND (-n) = n AND (~n + 1)
    mov eax, 0b10110100     ; 180
    mov ebx, eax
    neg ebx                 ; ebx = -180 (two's complement)
    and eax, ebx            ; eax = bit ที่ต่ำสุดที่เป็น 1
    ; eax = 0b00000100 = 4 (bit 2)

    ; ===== Trick 3: Clear lowest set bit =====
    mov eax, 0b10110100
    mov ebx, eax
    dec ebx                 ; ebx = 0b10110011
    and eax, ebx            ; eax = 0b10110000 (bit ต่ำสุดถูก clear)

    ; ===== Trick 4: นับจำนวน set bits (Population Count) =====
    ; ใช้ POPCNT instruction (SSE4.2+)
    mov eax, 0xFF00FF00     ; 16 set bits
    popcnt eax, eax         ; eax = 16

    ; ===== Trick 5: ตรวจสอบ single bit =====
    mov eax, 0b10110100
    bt eax, 4               ; test bit 4
    jc bit4_set             ; jump ถ้า bit 4 = 1 (CF set)
    jmp bit4_clear

bit4_set:
    ; bit 4 = 1
    jmp continue_tricks

bit4_clear:
    ; bit 4 = 0
continue_tricks:

    ; ===== Trick 6: Set bit ด้วย BTS =====
    mov eax, 0b10110100
    bts eax, 0              ; Set bit 0 (old value → CF)
    ; eax = 0b10110101

    ; ===== Trick 7: Clear bit ด้วย BTR =====
    mov eax, 0b10110101
    btr eax, 2              ; Clear bit 2 (old value → CF)
    ; eax = 0b10110001

    ; ===== Trick 8: Toggle bit ด้วย BTC =====
    mov eax, 0b10110001
    btc eax, 3              ; Toggle bit 3
    ; eax = 0b10111001

    ; ===== Trick 9: หา MSB (Most Significant Set Bit) =====
    mov eax, 0b00101100     ; MSB อยู่ที่ bit 5
    bsr eax, eax            ; Bit Scan Reverse: eax = 5
    ; ถ้า eax = 0 ตั้งแต่แรก → ZF set

    ; ===== Trick 10: หา LSB (Least Significant Set Bit) =====
    mov eax, 0b00101100     ; LSB อยู่ที่ bit 2
    bsf eax, eax            ; Bit Scan Forward: eax = 2

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 13. Code แปลงระบบตัวเลข (Complete Converter)

```nasm
; ไฟล์: number_converter.asm
; แปลง decimal → hex string และแสดงผล
; compile: nasm -f elf64 number_converter.asm -o number_converter.o
;          ld number_converter.o -o number_converter
; Expected output:
;   Dec: 255
;   Hex: 0xFF
;   Bin: 11111111

section .data
    msg_dec  db 'Dec: ', 0
    msg_hex  db 'Hex: 0x', 0
    msg_bin  db 'Bin: ', 0
    newline  db 0xA, 0

section .bss
    hex_buf  resb 17        ; "0x" + 16 hex digits + null
    dec_buf  resb 21        ; max 20 decimal digits + null
    bin_buf  resb 65        ; max 64 binary digits + null

section .text
    global _start

; ฟังก์ชัน: print_string (rdi = pointer to string, rdx = length)
print_string:
    mov rax, 1              ; syscall: write
    mov rsi, rdi            ; รับ pointer จาก rdi
    syscall
    ret

; ฟังก์ชัน: uint64_to_hex (rax = number, rdi = buffer → fills buffer)
; returns: rdx = number of hex digits written
uint64_to_hex:
    push rbx
    push rcx
    push r8

    mov rcx, 16             ; สูงสุด 16 hex digits
    lea r8, [rdi + rcx]     ; ชี้ไปท้าย buffer
    dec r8
    mov byte [r8], 0        ; null terminator

    mov rbx, rax            ; เก็บค่าไว้ใน rbx

.hex_loop:
    mov rax, rbx
    and rax, 0xF            ; ดึง 4 bits ล่าง
    cmp rax, 9
    jle .digit              ; 0-9
    add rax, 'A' - 10       ; A-F
    jmp .store_digit
.digit:
    add rax, '0'
.store_digit:
    dec r8
    mov [r8], al            ; เก็บ hex digit
    shr rbx, 4              ; เลื่อน 4 bits
    dec rcx
    jnz .hex_loop

    mov rdi, r8             ; pointer ไปยังผลลัพธ์
    mov rdx, 16             ; ความยาว
    pop r8
    pop rcx
    pop rbx
    ret

; ฟังก์ชัน: uint64_to_decimal (rax = number, rdi = buffer)
; returns: rdx = length of string
uint64_to_decimal:
    push rbx
    push rcx
    push r8
    push r9

    test rax, rax
    jnz .not_zero

    ; กรณีพิเศษ: 0
    mov byte [rdi], '0'
    mov byte [rdi+1], 0
    mov rdx, 1
    jmp .dec_done

.not_zero:
    mov r8, rdi             ; เก็บ start pointer
    lea r9, [rdi + 20]      ; end of buffer
    mov byte [r9], 0        ; null terminator
    mov rbx, r9

.div_loop:
    test rax, rax
    jz .reverse_done
    xor rdx, rdx
    mov rcx, 10
    div rcx                 ; rax = rax/10, rdx = rax%10
    add dl, '0'             ; แปลงเป็น ASCII
    dec rbx
    mov [rbx], dl           ; เก็บ digit
    jmp .div_loop

.reverse_done:
    ; copy result to beginning of buffer
    mov rsi, rbx
    mov rdi, r8
.copy_loop:
    mov al, [rsi]
    mov [rdi], al
    inc rsi
    inc rdi
    cmp rsi, r9
    jle .copy_loop
    mov byte [rdi], 0

    ; คำนวณความยาว
    mov rdx, r9
    sub rdx, rbx            ; length = end - start

.dec_done:
    pop r9
    pop r8
    pop rcx
    pop rbx
    ret

_start:
    ; แปลง 255 ทุก format
    mov rax, 255

    ; แสดง "Dec: "
    mov rdi, msg_dec
    mov rdx, 5
    call print_string

    ; แปลงและแสดง decimal
    lea rdi, [dec_buf]
    call uint64_to_decimal
    push rdx
    mov rdi, dec_buf
    pop rdx
    call print_string

    ; newline
    mov rdi, newline
    mov rdx, 1
    call print_string

    ; แสดง "Hex: 0x"
    mov rdi, msg_hex
    mov rdx, 7
    call print_string

    ; แปลงและแสดง hex
    mov rax, 255
    lea rdi, [hex_buf]
    call uint64_to_hex
    call print_string

    ; newline
    mov rdi, newline
    mov rdx, 1
    call print_string

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 14. ARM Assembly: ระบบตัวเลข

### 14.1 ตัวอย่าง Bitwise ใน ARM (GAS syntax)

```asm
@ ไฟล์: arm_bitwise.s
@ ARM Cortex-A bitwise operations ใน GAS syntax
@ compile: as -o arm_bitwise.o arm_bitwise.s
@          ld -o arm_bitwise arm_bitwise.o

.section .data
result_msg:
    .ascii "Result calculated\n"
    .set msg_len, . - result_msg

.section .text
.global _start

_start:
    @ ===== AND =====
    MOV r0, #0xAB          @ r0 = 0xAB = 10101011
    MOV r1, #0x0F          @ r1 = 0x0F = mask (lower nibble)
    AND r2, r0, r1         @ r2 = r0 AND r1 = 0x0B = lower nibble

    @ ===== ORR (OR) =====
    MOV r0, #0x50          @ r0 = 0101 0000
    ORR r0, r0, #0x08      @ r0 = r0 OR 0x08 = 0101 1000 (set bit 3)

    @ ===== EOR (XOR) =====
    MOV r0, #0xFF
    EOR r0, r0, #0xFF      @ r0 = 0 (XOR กับตัวเอง = 0)

    @ ===== MVN (NOT) =====
    MOV r0, #0xAA          @ r0 = 1010 1010
    MVN r1, r0             @ r1 = NOT r0 = 0101 0101 = 0x55

    @ ===== Shift Operations =====
    @ ARM inline shifting (เป็น feature พิเศษของ ARM!)
    MOV r0, #1
    MOV r1, r0, LSL #4     @ r1 = r0 << 4 = 16 (ด้วย inline shift)

    MOV r0, #32
    MOV r1, r0, LSR #2     @ r1 = r0 >> 2 = 8 (logical shift right)

    MOV r0, #-32           @ r0 = -32 (signed)
    MOV r1, r0, ASR #2     @ r1 = r0 >> 2 = -8 (arithmetic shift, sign-extend)

    @ ===== BIC (Bit Clear) =====
    @ BIC r_dest, r_src, mask → clears bits where mask=1
    MOV r0, #0xFF
    BIC r0, r0, #0x0F      @ clear lower nibble → r0 = 0xF0

    @ ===== Test bit (ANDS + check Z flag) =====
    MOV r0, #0b10110100
    ANDS r1, r0, #(1 << 4) @ test bit 4 (set flags)
    BNE bit4_is_set         @ branch if not zero → bit 4 = 1
    B   bit4_is_clear

bit4_is_set:
    @ bit 4 = 1
    B exit

bit4_is_clear:
    @ bit 4 = 0

exit:
    @ Exit syscall (ARM Linux)
    MOV r7, #1             @ syscall: exit
    MOV r0, #0             @ exit code 0
    SWI 0                  @ software interrupt
```

---

## 15. ข้อผิดพลาดที่พบบ่อย (Common Mistakes)

### 15.1 ข้อผิดพลาด 1: ใช้ SHR กับ Signed Numbers

```nasm
; ผิด!
mov eax, -100           ; eax = 0xFFFFFF9C
shr eax, 1              ; eax = 0x7FFFFCE = ค่าบวกใหญ่มาก! ผิด!

; ถูกต้อง!
mov eax, -100
sar eax, 1              ; eax = -50 (sign-extended correctly)
```

### 15.2 ข้อผิดพลาด 2: MOVZX vs MOVSX

```nasm
; ถ้า al = 0xFF = -1 (signed) หรือ 255 (unsigned)

movzx eax, al           ; Zero-extend: eax = 0x000000FF = 255
movsx eax, al           ; Sign-extend: eax = 0xFFFFFFFF = -1

; ใช้ MOVZX สำหรับ unsigned, MOVSX สำหรับ signed
```

### 15.3 ข้อผิดพลาด 3: Float Comparison

```nasm
; ผิด! ไม่สามารถใช้ CMP กับ float ได้
; cmp xmm0, xmm1      ; ERROR: invalid instruction

; ถูกต้อง! ใช้ UCOMISS/UCOMISD
ucomiss xmm0, xmm1      ; compare single precision
ja greater              ; unsigned jump ใช้กับ float comparison!
; หมายเหตุ: ต้องใช้ unsigned jumps (ja, jb, je) ไม่ใช่ signed (jg, jl)

; ระวัง NaN! UCOMISS set PF (parity flag) เมื่อ NaN
jp nan_result           ; jump if parity flag (NaN detected)
```

### 15.4 ข้อผิดพลาด 4: Byte ↔ Word Confusion

```nasm
; ผิด: ขนาดไม่ตรงกัน
mov eax, [rbx]         ; โหลด 4 bytes
cmp al, 255            ; เปรียบเทียบแค่ 1 byte (al = lower byte ของ eax)
; อาจจะ work แต่สับสน

; ถูกต้อง: ชัดเจน
movzx eax, byte [rbx]  ; โหลด 1 byte, zero-extend
cmp eax, 255           ; เปรียบเทียบ 4 bytes (consistent size)
```

### 15.5 ข้อผิดพลาด 5: Overflow ใน Intermediate Calculations

```nasm
; ผิด: overflow ระหว่างการคำนวณ
; ต้องการคำนวณ (a * b) + c ที่ a=100000, b=100000, c=0
mov eax, 100000
imul eax, 100000        ; eax = 10,000,000,000 → OVERFLOW! (เกิน 32-bit)

; ถูกต้อง: ใช้ 64-bit
mov rax, 100000
imul rax, 100000        ; rax = 10,000,000,000 → ไม่ overflow ใน 64-bit
```

---

## 16. สรุปย่อ: Bit Manipulation Quick Reference

```
+------------------+------------------------+---------------------------+
| Operation        | Code                   | ผลลัพธ์                   |
+------------------+------------------------+---------------------------+
| Set bit N        | or eax, 1 << N         | bit N = 1                 |
| Clear bit N      | and eax, ~(1 << N)     | bit N = 0                 |
| Toggle bit N     | xor eax, 1 << N        | bit N flip                |
| Test bit N       | test eax, 1 << N       | ZF=0 ถ้า set              |
| Zero register    | xor eax, eax           | eax = 0 (เร็วกว่า mov)   |
| Sign extend byte | movsx eax, al          | eax = signed(al)          |
| Zero extend byte | movzx eax, al          | eax = unsigned(al)        |
| Abs value        | neg eax; cmovl eax,... | |eax|                      |
| Swap a, b        | xchg eax, ebx          | a↔b                       |
| Multiply by 2^N  | shl eax, N             | eax *= 2^N                |
| Divide by 2^N    | shr eax, N (unsigned)  | eax /= 2^N                |
|                  | sar eax, N (signed)    |                           |
| Lower nibble     | and eax, 0xF           | bits 0-3                  |
| Upper nibble     | and eax, 0xF0 → shr 4  | bits 4-7                  |
| Align to 16      | and rax, -16           | round down to 16 bytes    |
| Power of 2 check | and eax,(eax-1); test  | ZF=1 ถ้า power of 2      |
+------------------+------------------------+---------------------------+
```

---

## 17. แบบฝึกหัด (Exercises)

### แบบฝึกหัดระดับง่าย (Easy)

**ข้อ 1:** แปลงตัวเลขต่อไปนี้ด้วยมือ (ไม่ใช้เครื่องคิดเลข)
```
a) 0b11001010  → Decimal และ Hex
b) 0xA3        → Decimal และ Binary
c) 203         → Binary และ Hex
d) 0o377       → Decimal และ Hex
```

**ข้อ 2:** หา Two's Complement ของ:
```
a) -1   ใน 8-bit
b) -127 ใน 8-bit
c) -1   ใน 16-bit
d) -256 ใน 32-bit
```

**ข้อ 3:** ทำ Bitwise Operations ต่อไปนี้:
```
a) 0b10110101 AND 0b11110000 = ?
b) 0b10110101 OR  0b00001111 = ?
c) 0b10110101 XOR 0b11111111 = ?
d) NOT 0b10110101 (8-bit)    = ?
```

**ข้อ 4:** ค่า Decimal ของแต่ละ shift คืออะไร?
```
a) 7 SHL 1 = ?
b) 7 SHL 2 = ?
c) 64 SHR 3 = ?
d) -16 SAR 2 = ?
```

**ข้อ 5:** เขียน NASM code ที่ตรวจสอบว่า eax เป็นเลขคู่หรือเลขคี่ แล้ว jump ไป label ที่เหมาะสม

### แบบฝึกหัดระดับกลาง (Medium)

**ข้อ 6:** เขียน function ใน NASM ที่รับ byte ใน al แล้วนับจำนวน 1-bits (population count) โดยไม่ใช้ POPCNT instruction
```
Expected: count_bits(0b10110101) = 5
```

**ข้อ 7:** เขียน code ที่ใช้ bit tricks แปลง ASCII uppercase เป็น lowercase และในทางกลับกัน โดยใช้เพียง 1 instruction ต่อการแปลงแต่ละทิศทาง

**ข้อ 8:** เขียน function ที่ reverse bits ใน byte
```
Input:  0b10110001 (0xB1)
Output: 0b10001101 (0x8D)
```

**ข้อ 9:** ทำความเข้าใจ floating-point precision:
```
เขียน code ที่:
a) โหลด 0.1 เป็น float
b) บวก 0.1 สิบครั้ง
c) เปรียบเทียบกับ 1.0
d) อธิบายว่าทำไม result อาจไม่เท่ากับ 1.0 พอดี
```

**ข้อ 10:** เขียน code ที่ใช้ fixed-point Q8.8 (8 bit integer, 8 bit fraction) คำนวณ:
```
2.5 × 3.75 = ?
แสดงผลลัพธ์ทั้ง fixed-point representation และค่า decimal จริง
```

### แบบฝึกหัดระดับยาก (Hard)

**ข้อ 11:** เขียน function ที่แปลง 32-bit integer เป็น string ทศนิยม โดยไม่ใช้ printf หรือ library ใดๆ
```
Expected: int_to_str(12345, buffer) → "12345"
```

**ข้อ 12:** เขียน function ที่รับ null-terminated hex string แล้วแปลงเป็น integer
```
Expected: hex_to_int("DEADBEEF") → 0xDEADBEEF = 3735928559
```

**ข้อ 13:** Implement ฟังก์ชัน `isqrt` (integer square root) ที่ไม่ใช้ FPU
```
Expected: isqrt(25) = 5, isqrt(100) = 10, isqrt(2) = 1
hint: ใช้ bit manipulation method
```

**ข้อ 14:** เขียน code ที่ตรวจสอบว่า float value เป็น:
- NaN
- +infinity หรือ -infinity
- Denormalized (subnormal)
- Normal number
โดยทำเองไม่ใช้ library

**ข้อ 15:** Implement BCD multiplication:
```
คูณ BCD 99 × 99 = 9801
ต้องทำเป็น 4-digit BCD result
```

### แบบฝึกหัดระดับ Expert

**ข้อ 16:** เขียน fast reciprocal approximation (1/x) ใช้ bit magic:
```c
// Quake III fast inverse square root เป็น reference
float Q_rsqrt(float number) {
    long i = *(long*)&number;
    i = 0x5f3759df - (i >> 1);
    float y = *(float*)&i;
    return y * (1.5f - 0.5f*number*y*y);
}
```
แปลง algorithm นี้เป็น NASM x86-64 ด้วย SSE2

**ข้อ 17:** เขียน UTF-8 encoder ที่รับ Unicode code point (U+0000 ถึง U+10FFFF) แล้วแปลงเป็น UTF-8 byte sequence

**ข้อ 18:** Implement LCG (Linear Congruential Generator) pseudo-random number generator:
```
next = (a * current + c) mod m
ใช้: a = 1664525, c = 1013904223, m = 2^32
```
เขียนให้ไม่ใช้ DIV instruction (ใช้ AND กับ mask แทน)

**ข้อ 19:** เขียน function ที่ตรวจสอบ parity:
- Even parity: จำนวน 1-bits เป็นเลขคู่
- Odd parity: จำนวน 1-bits เป็นเลขคี่
ใช้ XOR chain technique

**ข้อ 20:** Implement Gray Code converter:
```
Binary to Gray: gray = binary XOR (binary >> 1)
Gray to Binary: binary = ??? (ทำกลับยากกว่า ต้องใช้ loop)
```

---

## 18. เฉลยแบบฝึกหัดบางข้อ

### เฉลยข้อ 1:
```
a) 0b11001010 = 202 decimal = 0xCA hex
b) 0xA3 = 163 decimal = 10100011 binary
c) 203 = 11001011 binary = 0xCB hex
d) 0o377 = 255 decimal = 0xFF hex
```

### เฉลยข้อ 2:
```
a) -1 ใน 8-bit = 0xFF = 11111111
b) -127 ใน 8-bit = 0x81 = 10000001
c) -1 ใน 16-bit = 0xFFFF = 1111111111111111
d) -256 ใน 32-bit = 0xFFFFFF00
```

### เฉลยข้อ 6: นับ 1-bits ใน byte
```nasm
; count_bits: นับจำนวน 1-bits ใน al
; input: al = byte to count
; output: al = count of 1-bits
; destroys: ah, cl

count_bits:
    xor ah, ah              ; ah = 0 (counter)
    mov cl, 8               ; วนลูป 8 ครั้ง (8 bits)

.loop:
    shr al, 1               ; shift right 1, ค่าที่หลุดไป CF
    adc ah, 0               ; ah += CF (เพิ่ม 1 ถ้า bit ที่หลุดเป็น 1)
    dec cl
    jnz .loop

    mov al, ah              ; ส่งผลลัพธ์กลับใน al
    ret
```

### เฉลยข้อ 7: Case conversion
```nasm
; Uppercase → Lowercase: OR กับ 0x20
or al, 0x20     ; 'A'(0x41) → 'a'(0x61)

; Lowercase → Uppercase: AND กับ 0xDF (= ~0x20)
and al, 0xDF    ; 'a'(0x61) → 'A'(0x41)
```

---

## 19. สรุปและ Key Takeaways

**สิ่งที่ต้องจำจาก Part นี้:**

1. **Hex = ภาษาของ Assembly programmer** - ต้องคล่องแคล่วในการแปลง
2. **Two's Complement** - วิธีที่คอมพิวเตอร์เก็บจำนวนลบ (invert + 1)
3. **Sign Flag vs Carry Flag** - CF สำหรับ unsigned overflow, OF สำหรับ signed overflow
4. **SAR vs SHR** - ใช้ SAR สำหรับ signed division, SHR สำหรับ unsigned เท่านั้น
5. **XOR eax, eax** = วิธีที่เร็วที่สุดในการ zero register
6. **IEEE 754** - Float ไม่ exact! อย่าใช้ == กับ float
7. **Bit manipulation** - ทำได้เร็วกว่า multiplication/division มาก
8. **MOVZX vs MOVSX** - ต้องเลือกให้ถูกต้องตาม signed/unsigned context

**คำสั่งสำคัญ:**
```
AND  - Mask, clear bits, test even/odd
OR   - Set bits, combine flags
XOR  - Toggle bits, zero register, swap
NOT  - Complement
SHL  - Multiply by power of 2
SHR  - Divide by power of 2 (unsigned)
SAR  - Divide by power of 2 (signed)
ROL/ROR - Rotate bits
BT/BTS/BTR/BTC - Bit test/set/reset/complement
BSF/BSR - Bit scan forward/reverse (find LSB/MSB)
POPCNT - Population count (count 1-bits)
MOVZX - Zero-extend (unsigned)
MOVSX - Sign-extend (signed)
```

---

## 20. แหล่งอ้างอิง (References)

1. **Intel Software Developer's Manual** - https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html
2. **AMD64 Architecture Programmer's Manual** - https://developer.amd.com/resources/developer-guides-manuals/
3. **IEEE 754 Standard** - https://ieeexplore.ieee.org/document/8766229
4. **Hacker's Delight (Book)** - Henry S. Warren - รวม bit manipulation tricks ที่ดีที่สุด
5. **NASM Documentation** - https://www.nasm.us/doc/
6. **ARM Architecture Reference Manual** - https://developer.arm.com/documentation/
7. **x86 Instruction Reference** - https://www.felixcloutier.com/x86/

---

## ต่อไป: Part 004

ใน **Part 004** เราจะเรียน **Memory Addressing Modes** อย่างละเอียด:
- Direct addressing
- Register indirect addressing
- Base + displacement
- Indexed addressing
- Scale factor
- Memory segmentation (x86)
- Virtual memory basics
- Stack operations และ Stack frame
- Heap และ dynamic memory ใน Assembly

---

*Part 003 | Assembly Course | เวลาที่ใช้เรียน: 4-6 ชั่วโมง*  
*ระดับ: พื้นฐาน-กลาง | Prerequisites: Part 001-002*
