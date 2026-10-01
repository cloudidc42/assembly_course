# Part 017: Comparison และ Jump Instructions

## สารบัญ
- [Prerequisites and Learning Objectives](#prerequisites)
- [ทฤษฎี: CMP Instruction และ EFLAGS](#cmp-eflags)
- [ทฤษฎี: TEST Instruction](#test-instruction)
- [ทฤษฎี: Jcc Instructions ทั้งหมด](#jcc-instructions)
- [ทฤษฎี: Signed vs Unsigned Comparison](#signed-unsigned)
- [ทฤษฎี: JMP Variations](#jmp-variations)
- [ทฤษฎี: Jump Tables](#jump-tables)
- [Code Examples](#code-examples)
- [Common Mistakes and Pitfalls](#common-mistakes)
- [Advanced Techniques](#advanced-techniques)
- [Exercises](#exercises)
- [Summary](#summary)

---

## Prerequisites and Learning Objectives {#prerequisites}

### ความรู้ที่ต้องมีก่อน

ก่อนเรียน Part นี้ ควรมีความเข้าใจใน:
- Registers (EAX, EBX, ECX, EDX, ESP, EBP, ESI, EDI) — Part 003
- Arithmetic Instructions (ADD, SUB, MUL, DIV) — Part 005-006
- EFLAGS Register เบื้องต้น — Part 004
- Basic program structure ด้วย NASM — Part 002

### วัตถุประสงค์การเรียนรู้ (Learning Objectives)

เมื่อเรียน Part นี้จบแล้ว ผู้เรียนจะสามารถ:

1. **อธิบาย** การทำงานของ CMP instruction และผลกระทบต่อ EFLAGS ได้
2. **ใช้งาน** TEST instruction สำหรับการตรวจสอบ bits ได้
3. **เลือกใช้** Jcc instruction ที่เหมาะสมจากกว่า 20 ตัวเลือก
4. **แยกแยะ** ความแตกต่างระหว่าง Signed และ Unsigned comparison
5. **ใช้งาน** JMP แบบ short, near, far และ indirect ได้
6. **สร้าง** Jump Table สำหรับ switch-case statement ได้
7. **แปลง** โครงสร้าง if/else, if/elif/else จาก C เป็น Assembly ได้
8. **ประเมิน** เมื่อไหร่ควรใช้ CMOV แทน Jump เพื่อประสิทธิภาพที่ดีกว่า

---

## ทฤษฎี: CMP Instruction และ EFLAGS {#cmp-eflags}

### EFLAGS Register — ทบทวน

EFLAGS เป็น register ขนาด 32 bits ที่เก็บสถานะของ CPU หลังจากการคำนวณแต่ละครั้ง

```
Bit Position: 31      11  10  9   8   7   6   5   4   3   2   1   0
                       |   |   |   |   |   |   |   |   |   |   |   |
Flag Name:            OF  DF  IF  TF  SF  ZF  --  AF  --  PF  --  CF
```

| Flag | ชื่อเต็ม | ความหมาย |
|------|---------|----------|
| CF | Carry Flag | มี carry หรือ borrow เกิดขึ้น |
| ZF | Zero Flag | ผลลัพธ์เป็น 0 |
| SF | Sign Flag | ผลลัพธ์เป็นลบ (MSB = 1) |
| OF | Overflow Flag | เกิด signed overflow |
| PF | Parity Flag | จำนวน 1-bits เป็นเลขคู่ |
| AF | Auxiliary Carry Flag | มี carry จาก bit 3 ไป bit 4 |

### CMP Instruction — หัวใจของการเปรียบเทียบ

**Syntax:** `CMP destination, source`

CMP ทำงานเหมือน SUB ทุกประการ **ยกเว้น** ไม่เก็บผลลัพธ์ มันเพียงแต่ set flags เท่านั้น

```
CMP A, B   <==>   A - B  (แต่ทิ้งผลลัพธ์, เก็บเฉพาะ flags)
```

#### ตัวอย่างการ set flags ด้วย CMP:

```
; กรณีที่ 1: A == B
MOV EAX, 5
CMP EAX, 5     ; 5 - 5 = 0  -->  ZF=1, CF=0, SF=0, OF=0

; กรณีที่ 2: A > B (unsigned)
MOV EAX, 10
CMP EAX, 5     ; 10 - 5 = 5 (บวก) -->  ZF=0, CF=0, SF=0, OF=0

; กรณีที่ 3: A < B (unsigned)
MOV EAX, 3
CMP EAX, 8     ; 3 - 8 = -5 (borrow) -->  ZF=0, CF=1, SF=1, OF=0

; กรณีที่ 4: A > B (signed, negative numbers)
MOV EAX, -1    ; 0xFFFFFFFF
CMP EAX, -5    ; -1 - (-5) = 4 (บวก) -->  ZF=0, CF=0, SF=0, OF=0

; กรณีที่ 5: Signed overflow
MOV EAX, 0x7FFFFFFF   ; INT_MAX = 2147483647
CMP EAX, -1           ; INT_MAX - (-1) จะ overflow  -->  OF=1
```

#### ตารางสรุป CMP Flags:

| เงื่อนไข | ZF | CF | SF | OF |
|---------|----|----|----|----|
| A == B | 1 | 0 | - | - |
| A != B | 0 | - | - | - |
| A < B (unsigned) | 0 | 1 | - | - |
| A > B (unsigned) | 0 | 0 | - | - |
| A <= B (unsigned) | ZF=1 หรือ CF=1 | - | - | - |
| A < B (signed) | 0 | - | SF != OF | |
| A > B (signed) | 0 | - | SF == OF | |

### CMP กับ Operand sizes ต่างๆ

```nasm
; 8-bit comparison
CMP AL, BL
CMP AL, 42
CMP BYTE [mem], 0

; 16-bit comparison  
CMP AX, BX
CMP AX, 1000
CMP WORD [mem], 0xFFFF

; 32-bit comparison (ที่ใช้บ่อยที่สุดใน x86)
CMP EAX, EBX
CMP ECX, 100
CMP DWORD [mem], 0

; 64-bit comparison (x86-64 mode)
CMP RAX, RBX
CMP RCX, 0xFFFFFFFFFFFFFFFF
```

---

## ทฤษฎี: TEST Instruction {#test-instruction}

### TEST คืออะไร?

TEST ทำงานเหมือน AND ทุกประการ **ยกเว้น** ไม่เก็บผลลัพธ์ ใช้สำหรับตรวจสอบ bits เฉพาะ

```
TEST A, B   <==>   A AND B  (แต่ทิ้งผลลัพธ์, เก็บเฉพาะ flags)
```

### การใช้งาน TEST ที่พบบ่อย

#### 1. ตรวจสอบว่า register เป็น 0 หรือไม่

```nasm
TEST EAX, EAX    ; EAX AND EAX = EAX
                 ; ถ้า EAX == 0  -->  ZF = 1
                 ; ถ้า EAX != 0  -->  ZF = 0

; เหมือนกับ CMP EAX, 0 แต่เร็วกว่าและสั้นกว่า (1 byte น้อยกว่า)
```

#### 2. ตรวจสอบ bit เฉพาะ

```nasm
; ตรวจสอบว่า bit 0 (LSB) เป็น 1 หรือไม่ (เลขคี่/คู่)
TEST EAX, 1      ; AND กับ mask 00000001
JNZ  is_odd      ; ถ้า ZF=0 แสดงว่า bit 0 เป็น 1 (เลขคี่)

; ตรวจสอบ bit 7 (ตรวจสอบ sign ของ byte)
TEST AL, 0x80
JNZ  is_negative

; ตรวจสอบว่า number หาร 4 ลงตัวหรือไม่ (bits 0 และ 1 เป็น 0)
TEST EAX, 0x03
JZ   divisible_by_4

; ตรวจสอบหลาย bits พร้อมกัน
TEST EAX, 0x0F   ; ตรวจสอบ lower nibble
```

#### 3. ความแตกต่างระหว่าง TEST และ CMP สำหรับกรณี = 0

```nasm
; วิธีที่ 1: ใช้ CMP (สร้าง opcode ใหญ่กว่า)
CMP EAX, 0
JE  zero_case

; วิธีที่ 2: ใช้ TEST (สร้าง opcode เล็กกว่า, เร็วกว่า)
TEST EAX, EAX
JZ   zero_case
```

---

## ทฤษฎี: Jcc Instructions ทั้งหมด {#jcc-instructions}

Jcc (Jump if condition) มีกว่า 20 variants แบ่งเป็นกลุ่มดังนี้:

### กลุ่ม 1: Zero-based (เปรียบเทียบทั้ง signed และ unsigned)

| Instruction | ชื่อเต็ม | เงื่อนไข | Flag |
|-------------|---------|---------|------|
| `JE` | Jump if Equal | A == B | ZF = 1 |
| `JZ` | Jump if Zero | result == 0 | ZF = 1 |
| `JNE` | Jump if Not Equal | A != B | ZF = 0 |
| `JNZ` | Jump if Not Zero | result != 0 | ZF = 0 |

หมายเหตุ: JE และ JZ เป็น alias เดียวกัน ใช้ opcode เดียวกัน

### กลุ่ม 2: Signed comparison (ใช้หลัง CMP กับค่า signed)

| Instruction | ชื่อเต็ม | เงื่อนไข | Flags |
|-------------|---------|---------|-------|
| `JL` | Jump if Less | A < B (signed) | SF ≠ OF |
| `JNGE` | Jump if Not Greater or Equal | A < B (signed) | SF ≠ OF |
| `JLE` | Jump if Less or Equal | A <= B (signed) | ZF=1 หรือ SF≠OF |
| `JNG` | Jump if Not Greater | A <= B (signed) | ZF=1 หรือ SF≠OF |
| `JG` | Jump if Greater | A > B (signed) | ZF=0 และ SF=OF |
| `JNLE` | Jump if Not Less or Equal | A > B (signed) | ZF=0 และ SF=OF |
| `JGE` | Jump if Greater or Equal | A >= B (signed) | SF = OF |
| `JNL` | Jump if Not Less | A >= B (signed) | SF = OF |

### กลุ่ม 3: Unsigned comparison (ใช้หลัง CMP กับค่า unsigned)

| Instruction | ชื่อเต็ม | เงื่อนไข | Flags |
|-------------|---------|---------|-------|
| `JB` | Jump if Below | A < B (unsigned) | CF = 1 |
| `JNAE` | Jump if Not Above or Equal | A < B (unsigned) | CF = 1 |
| `JC` | Jump if Carry | CF set | CF = 1 |
| `JBE` | Jump if Below or Equal | A <= B (unsigned) | CF=1 หรือ ZF=1 |
| `JNA` | Jump if Not Above | A <= B (unsigned) | CF=1 หรือ ZF=1 |
| `JA` | Jump if Above | A > B (unsigned) | CF=0 และ ZF=0 |
| `JNBE` | Jump if Not Below or Equal | A > B (unsigned) | CF=0 และ ZF=0 |
| `JAE` | Jump if Above or Equal | A >= B (unsigned) | CF = 0 |
| `JNB` | Jump if Not Below | A >= B (unsigned) | CF = 0 |
| `JNC` | Jump if No Carry | CF = 0 | CF = 0 |

### กลุ่ม 4: Sign/Overflow/Parity flags

| Instruction | ชื่อเต็ม | เงื่อนไข | Flag |
|-------------|---------|---------|------|
| `JS` | Jump if Sign | ผลลัพธ์เป็นลบ | SF = 1 |
| `JNS` | Jump if Not Sign | ผลลัพธ์เป็นบวก/ศูนย์ | SF = 0 |
| `JO` | Jump if Overflow | signed overflow เกิดขึ้น | OF = 1 |
| `JNO` | Jump if No Overflow | ไม่มี signed overflow | OF = 0 |
| `JP` | Jump if Parity | จำนวน 1-bits เป็นเลขคู่ | PF = 1 |
| `JPE` | Jump if Parity Even | เหมือน JP | PF = 1 |
| `JNP` | Jump if No Parity | จำนวน 1-bits เป็นเลขคี่ | PF = 0 |
| `JPO` | Jump if Parity Odd | เหมือน JNP | PF = 0 |

### กลุ่ม 5: CX/ECX/RCX-based loops

| Instruction | เงื่อนไข | หมายเหตุ |
|-------------|---------|---------|
| `JCXZ` | Jump if CX == 0 | ตรวจ CX (16-bit) |
| `JECXZ` | Jump if ECX == 0 | ตรวจ ECX (32-bit) |
| `JRCXZ` | Jump if RCX == 0 | ตรวจ RCX (64-bit) |

### Alias ทั้งหมด (Instructions ที่มี opcode เดียวกัน)

```
JE  = JZ       (ZF = 1)
JNE = JNZ      (ZF = 0)
JL  = JNGE     (SF ≠ OF)
JLE = JNG      (ZF=1 หรือ SF≠OF)
JG  = JNLE     (ZF=0 และ SF=OF)
JGE = JNL      (SF = OF)
JB  = JNAE = JC    (CF = 1)
JBE = JNA      (CF=1 หรือ ZF=1)
JA  = JNBE     (CF=0 และ ZF=0)
JAE = JNB = JNC    (CF = 0)
JP  = JPE      (PF = 1)
JNP = JPO      (PF = 0)
```

---

## ทฤษฎี: Signed vs Unsigned Comparison {#signed-unsigned}

### ทำไมต้องแยก Signed/Unsigned?

ตัวเลขชุดเดียวกันอาจมีความหมายต่างกันขึ้นอยู่กับว่าตีความแบบ signed หรือ unsigned:

```
Binary: 1111 1110  (0xFE = 254 bytes)

Unsigned interpretation: 254
Signed interpretation:   -2
```

### ตัวอย่างที่ชัดเจน

```nasm
MOV AL, 0xFF    ; = 255 (unsigned) หรือ -1 (signed)
MOV BL, 0x01    ; = 1

CMP AL, BL      ; 0xFF - 0x01

; Unsigned view: 255 > 1  --> JA (Jump if Above) จะ jump
; Signed view:   -1 < 1   --> JL (Jump if Less) จะ jump
```

### กฎการเลือก Jcc:

| ประเภทข้อมูล | น้อยกว่า | น้อยกว่าหรือเท่ากับ | มากกว่า | มากกว่าหรือเท่ากับ |
|-------------|---------|--------------|---------|----------------|
| **Signed** | JL | JLE | JG | JGE |
| **Unsigned** | JB | JBE | JA | JAE |
| **Either** | JE / JZ | - | JNE / JNZ | - |

### Memory Address เป็น Unsigned เสมอ

```nasm
; ที่อยู่หน่วยความจำเป็น unsigned เสมอ
; ห้ามใช้ JL/JG สำหรับเปรียบเทียบ addresses
CMP ESI, EDI    ; เปรียบเทียบ 2 pointers
JB  ptr1_lower  ; ใช้ JB ไม่ใช่ JL!
```

---

## ทฤษฎี: JMP Variations {#jmp-variations}

### 1. Short Jump (ระยะสั้น ±127 bytes)

```nasm
JMP short target    ; opcode: EB <offset>
                    ; offset: signed 8-bit (-128 ถึง +127)
                    ; เหมาะสำหรับ loop ขนาดเล็กหรือ skip code ใกล้ๆ
```

### 2. Near Jump (ระยะกลาง ±2GB)

```nasm
JMP near target     ; opcode: E9 <offset> (32-bit offset)
; หรือเพียง:
JMP target          ; NASM จะเลือก short/near โดยอัตโนมัติ
```

### 3. Far Jump (ข้าม Segment)

```nasm
JMP FAR [mem]       ; เปลี่ยนทั้ง CS และ EIP
                    ; ใช้ใน OS/kernel code เท่านั้น
                    ; ไม่ค่อยพบในโปรแกรม user-space
```

### 4. Indirect Jump ผ่าน Register

```nasm
JMP EAX             ; กระโดดไปที่ address ที่เก็บอยู่ใน EAX
JMP EBX             ; กระโดดไปที่ address ใน EBX
JMP ECX             ; กระโดดไปที่ address ใน ECX
```

### 5. Indirect Jump ผ่าน Memory

```nasm
JMP [EAX]           ; อ่าน address จาก memory ที่ EAX ชี้อยู่
JMP [label]         ; อ่าน address จาก memory label
JMP [table + EAX*4] ; Jump table lookup (32-bit pointers)
```

---

## ทฤษฎี: Jump Tables {#jump-tables}

Jump Table คือ array ของ function pointers ที่ใช้ implement switch-case statement อย่างมีประสิทธิภาพ

### C switch-case เทียบกับ Assembly

```c
// C code
switch (n) {
    case 0: func0(); break;
    case 1: func1(); break;
    case 2: func2(); break;
    case 3: func3(); break;
    default: default_func(); break;
}
```

```nasm
; Assembly equivalent
CMP ECX, 3          ; ตรวจสอบว่า n อยู่ใน range
JA  default_case    ; ถ้า n > 3 ไป default
JMP [jump_table + ECX*4]  ; index into jump table

section .data
jump_table:
    dd func0        ; case 0
    dd func1        ; case 1
    dd func2        ; case 2
    dd func3        ; case 3
```

---

## Code Examples {#code-examples}

---

### Example 1: CMP และ Basic Jumps — ตัวเปรียบเทียบตัวเลข

**ไฟล์:** `compare_basic.asm`

```nasm
; ============================================================
; compare_basic.asm - ตัวอย่าง CMP และ Jcc instructions เบื้องต้น
; แสดงการเปรียบเทียบตัวเลขและการใช้ conditional jumps
;
; วิธีคอมไพล์และรัน:
;   nasm -f elf32 compare_basic.asm -o compare_basic.o
;   ld -m elf_i386 compare_basic.o -o compare_basic
;   ./compare_basic
;
; Expected output:
;   10 > 5: YES
;   3 < 8: YES
;   7 == 7: YES
;   10 >= 10: YES
;   -5 < 3 (signed): YES
;   255 > 1 (unsigned): YES
; ============================================================

section .data
    ; ข้อความสำหรับแสดงผล
    msg_gt    db "10 > 5: YES", 10, 0
    msg_lt    db "3 < 8: YES", 10, 0
    msg_eq    db "7 == 7: YES", 10, 0
    msg_ge    db "10 >= 10: YES", 10, 0
    msg_slt   db "-5 < 3 (signed): YES", 10, 0
    msg_ugt   db "255 > 1 (unsigned): YES", 10, 0
    msg_no    db "[condition NOT met]", 10, 0

    ; ความยาวของแต่ละข้อความ
    len_gt    equ $ - msg_gt - 1
    len_lt    equ $ - msg_lt - 1
    len_eq    equ $ - msg_eq - 1
    len_ge    equ $ - msg_ge - 1
    len_slt   equ $ - msg_slt - 1
    len_ugt   equ $ - msg_ugt - 1

section .text
    global _start

; ============================================================
; Macro สำหรับเขียนข้อความ (เพื่อลดการซ้ำซ้อนของ code)
; ============================================================
%macro print_str 2
    mov eax, 4          ; sys_write
    mov ebx, 1          ; stdout
    mov ecx, %1         ; address ของ string
    mov edx, %2         ; ความยาว string
    int 0x80
%endmacro

_start:
    ; ============================================================
    ; Test 1: 10 > 5 (unsigned greater than)
    ; ============================================================
    mov eax, 10         ; โหลดค่า 10 เข้า EAX
    cmp eax, 5          ; เปรียบเทียบ 10 กับ 5 (คำนวณ 10 - 5)
                        ; ผล: ZF=0, CF=0 (ไม่มี borrow), SF=0
    ja  test1_yes       ; JA = Jump if Above (unsigned >)
                        ; เงื่อนไข: CF=0 และ ZF=0 --> jump!
    jmp test1_no

test1_yes:
    print_str msg_gt, 12        ; พิมพ์ "10 > 5: YES"
    jmp test2

test1_no:
    print_str msg_no, 19

    ; ============================================================
    ; Test 2: 3 < 8 (unsigned less than)
    ; ============================================================
test2:
    mov eax, 3          ; โหลดค่า 3 เข้า EAX
    cmp eax, 8          ; เปรียบเทียบ 3 กับ 8 (คำนวณ 3 - 8)
                        ; ผล: ZF=0, CF=1 (มี borrow!), SF=1
    jb  test2_yes       ; JB = Jump if Below (unsigned <)
                        ; เงื่อนไข: CF=1 --> jump!
    jmp test2_no

test2_yes:
    print_str msg_lt, 11
    jmp test3

test2_no:
    print_str msg_no, 19

    ; ============================================================
    ; Test 3: 7 == 7 (equality test)
    ; ============================================================
test3:
    mov eax, 7          ; โหลดค่า 7 เข้า EAX
    cmp eax, 7          ; เปรียบเทียบ 7 กับ 7 (คำนวณ 7 - 7 = 0)
                        ; ผล: ZF=1! (zero result)
    je  test3_yes       ; JE = Jump if Equal
                        ; เงื่อนไข: ZF=1 --> jump!
    jmp test3_no

test3_yes:
    print_str msg_eq, 11
    jmp test4

test3_no:
    print_str msg_no, 19

    ; ============================================================
    ; Test 4: 10 >= 10 (greater or equal)
    ; ============================================================
test4:
    mov eax, 10         ; โหลดค่า 10 เข้า EAX
    cmp eax, 10         ; เปรียบเทียบ 10 กับ 10 (คำนวณ 10 - 10 = 0)
                        ; ผล: ZF=1 (เท่ากัน)
    jae test4_yes       ; JAE = Jump if Above or Equal (unsigned >=)
                        ; เงื่อนไข: CF=0 (รวม ZF=1 กรณีเท่ากัน) --> jump!
    jmp test4_no

test4_yes:
    print_str msg_ge, 13
    jmp test5

test4_no:
    print_str msg_no, 19

    ; ============================================================
    ; Test 5: -5 < 3 (signed comparison)
    ; ============================================================
test5:
    mov eax, -5         ; โหลดค่า -5 เข้า EAX (0xFFFFFFFB)
    cmp eax, 3          ; เปรียบเทียบ -5 กับ 3
                        ; คำนวณ: -5 - 3 = -8
                        ; SF=1 (ผลลบ), OF=0 (ไม่มี overflow)
                        ; SF ≠ OF? 1 ≠ 0 = true
    jl  test5_yes       ; JL = Jump if Less (signed <)
                        ; เงื่อนไข: SF ≠ OF --> jump!
    jmp test5_no

test5_yes:
    print_str msg_slt, 20
    jmp test6

test5_no:
    print_str msg_no, 19

    ; ============================================================
    ; Test 6: 255 > 1 (unsigned, important: signed would give -1 < 1)
    ; ============================================================
test6:
    mov al, 0xFF        ; โหลด 0xFF = 255 (unsigned) หรือ -1 (signed)
    cmp al, 0x01        ; เปรียบเทียบกับ 1

    ; ถ้าใช้ JA (unsigned): 255 > 1 --> jump (ถูกต้อง!)
    ; ถ้าใช้ JG (signed): -1 > 1 --> ไม่ jump (ผิด!)
    ja  test6_yes_unsigned   ; ใช้ JA สำหรับ unsigned comparison

test6_yes_unsigned:
    print_str msg_ugt, 22
    jmp exit_program

test6_no:
    print_str msg_no, 19

exit_program:
    ; ออกจากโปรแกรม
    mov eax, 1          ; sys_exit
    xor ebx, ebx        ; exit code 0
    int 0x80
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 compare_basic.asm -o compare_basic.o
ld -m elf_i386 compare_basic.o -o compare_basic
./compare_basic
```

**Expected Output:**
```
10 > 5: YES
3 < 8: YES
7 == 7: YES
10 >= 10: YES
-5 < 3 (signed): YES
255 > 1 (unsigned): YES
```

---

### Example 2: TEST Instruction และ Bit Testing

**ไฟล์:** `bit_testing.asm`

```nasm
; ============================================================
; bit_testing.asm - การใช้ TEST instruction ตรวจสอบ bits
; แสดงการตรวจสอบคุณสมบัติต่างๆ ของตัวเลข
;
; วิธีคอมไพล์และรัน:
;   nasm -f elf32 bit_testing.asm -o bit_testing.o
;   ld -m elf_i386 bit_testing.o -o bit_testing
;   ./bit_testing
;
; Expected output:
;   Testing number: 42 (0x2A = 0010 1010)
;   Bit 0 (odd/even): EVEN
;   Bit 1: SET
;   Bit 3: SET
;   Is zero: NO
;   Testing number: 0
;   Is zero: YES
;   Testing number: 255 (0xFF)
;   Lower nibble nonzero: YES
;   Upper nibble nonzero: YES
; ============================================================

section .data
    ; ข้อความสำหรับแสดงผล
    header1     db "Testing number: 42 (0x2A = 0010 1010)", 10, 0
    msg_even    db "Bit 0 (odd/even): EVEN", 10, 0
    msg_odd     db "Bit 0 (odd/even): ODD", 10, 0
    msg_b1set   db "Bit 1: SET", 10, 0
    msg_b1clr   db "Bit 1: CLEAR", 10, 0
    msg_b3set   db "Bit 3: SET", 10, 0
    msg_b3clr   db "Bit 3: CLEAR", 10, 0
    msg_notzero db "Is zero: NO", 10, 0
    header2     db "Testing number: 0", 10, 0
    msg_zero    db "Is zero: YES", 10, 0
    header3     db "Testing number: 255 (0xFF)", 10, 0
    msg_lnib    db "Lower nibble nonzero: YES", 10, 0
    msg_llib    db "Lower nibble nonzero: NO", 10, 0
    msg_unib    db "Upper nibble nonzero: YES", 10, 0
    msg_ulib    db "Upper nibble nonzero: NO", 10, 0

section .text
    global _start

%macro print 2
    mov eax, 4
    mov ebx, 1
    mov ecx, %1
    mov edx, %2
    int 0x80
%endmacro

_start:
    ; ============================================================
    ; ส่วนที่ 1: ทดสอบตัวเลข 42 = 0x2A = 0010 1010 binary
    ; ============================================================
    print header1, 38

    mov eax, 42         ; โหลดตัวเลขที่จะทดสอบ

    ; --- ทดสอบ Bit 0: ตรวจสอบว่าเป็นเลขคี่หรือคู่ ---
    test eax, 1         ; AND EAX กับ mask 0000...0001
                        ; 42 = ...0010 1010
                        ; AND 0000 0001
                        ;   = 0000 0000  --> ZF = 1 (zero result = even!)
    jnz  is_odd_42      ; ถ้า ZF=0 แสดงว่า bit 0 เป็น 1 (เลขคี่)

is_even_42:
    print msg_even, 23  ; "Bit 0 (odd/even): EVEN"
    jmp test_bit1

is_odd_42:
    print msg_odd, 22
    jmp test_bit1

    ; --- ทดสอบ Bit 1 ---
test_bit1:
    test eax, 2         ; mask 0000...0010
                        ; 42 = ...0010 1010
                        ; AND 0000 0010
                        ;   = 0000 0010 --> ZF = 0 (nonzero = bit SET!)
    jz   bit1_clear

bit1_set:
    print msg_b1set, 11 ; "Bit 1: SET"
    jmp test_bit3

bit1_clear:
    print msg_b1clr, 13
    jmp test_bit3

    ; --- ทดสอบ Bit 3 ---
test_bit3:
    test eax, 8         ; mask 0000...1000 (bit 3)
                        ; 42 = ...0010 1010
                        ; AND 0000 1000
                        ;   = 0000 1000 --> ZF = 0 (bit SET!)
    jz   bit3_clear

bit3_set:
    print msg_b3set, 11 ; "Bit 3: SET"
    jmp test_zero_42

bit3_clear:
    print msg_b3clr, 13

    ; --- ทดสอบว่าเป็น Zero หรือไม่ ---
test_zero_42:
    test eax, eax       ; AND EAX กับตัวเอง
                        ; ถ้า EAX = 0: ผล = 0, ZF = 1
                        ; ถ้า EAX ≠ 0: ผล ≠ 0, ZF = 0
    jz   is_zero_42     ; ถ้า ZF=1 แสดงว่าเป็น 0

not_zero_42:
    print msg_notzero, 12   ; "Is zero: NO"
    jmp section2

is_zero_42:
    print msg_zero, 13

    ; ============================================================
    ; ส่วนที่ 2: ทดสอบตัวเลข 0
    ; ============================================================
section2:
    print header2, 18   ; "Testing number: 0"

    mov eax, 0          ; โหลดค่า 0

    test eax, eax       ; 0 AND 0 = 0  --> ZF = 1
    jz   is_zero_sec2

not_zero_sec2:
    print msg_notzero, 12
    jmp section3

is_zero_sec2:
    print msg_zero, 13  ; "Is zero: YES"

    ; ============================================================
    ; ส่วนที่ 3: ทดสอบ 255 = 0xFF สำหรับ nibble testing
    ; ============================================================
section3:
    print header3, 27   ; "Testing number: 255 (0xFF)"

    mov eax, 0xFF       ; = 1111 1111

    ; --- ทดสอบ Lower nibble (bits 0-3) ---
    test eax, 0x0F      ; mask: 0000 1111
                        ; 0xFF AND 0x0F = 0x0F (nonzero)
    jz   lower_zero

lower_nonzero:
    print msg_lnib, 25  ; "Lower nibble nonzero: YES"
    jmp test_upper

lower_zero:
    print msg_llib, 24

    ; --- ทดสอบ Upper nibble (bits 4-7) ---
test_upper:
    test eax, 0xF0      ; mask: 1111 0000
                        ; 0xFF AND 0xF0 = 0xF0 (nonzero)
    jz   upper_zero

upper_nonzero:
    print msg_unib, 25  ; "Upper nibble nonzero: YES"
    jmp program_exit

upper_zero:
    print msg_ulib, 24

program_exit:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 bit_testing.asm -o bit_testing.o
ld -m elf_i386 bit_testing.o -o bit_testing
./bit_testing
```

**Expected Output:**
```
Testing number: 42 (0x2A = 0010 1010)
Bit 0 (odd/even): EVEN
Bit 1: SET
Bit 3: SET
Is zero: NO
Testing number: 0
Is zero: YES
Testing number: 255 (0xFF)
Lower nibble nonzero: YES
Upper nibble nonzero: YES
```

---

### Example 3: If/Else และ If/Elif/Else Structures

**ไฟล์:** `if_else_demo.asm`

```nasm
; ============================================================
; if_else_demo.asm - การแปลง if/else และ if/elif/else เป็น Assembly
; แสดงรูปแบบการเขียน control flow ที่พบบ่อยในโปรแกรมจริง
;
; วิธีคอมไพล์และรัน:
;   nasm -f elf32 if_else_demo.asm -o if_else_demo.o
;   ld -m elf_i386 if_else_demo.o -o if_else_demo
;   ./if_else_demo
;
; Expected output:
;   Grade for score 85: B
;   Grade for score 55: F
;   Category for -10: Negative
;   Category for 0: Zero
;   Category for 100: Large Positive
; ============================================================

section .data
    ; Messages for grade output
    grade_a     db "Grade for score 85: A", 10, 0
    grade_b     db "Grade for score 85: B", 10, 0
    grade_c     db "Grade for score 85: C", 10, 0
    grade_d     db "Grade for score 85: D", 10, 0
    grade_f     db "Grade for score 85: F", 10, 0

    grade_a2    db "Grade for score 55: A", 10, 0
    grade_b2    db "Grade for score 55: B", 10, 0
    grade_c2    db "Grade for score 55: C", 10, 0
    grade_d2    db "Grade for score 55: D", 10, 0
    grade_f2    db "Grade for score 55: F", 10, 0

    ; Messages for category output
    cat_neg     db "Category for -10: Negative", 10, 0
    cat_zero    db "Category for 0: Zero", 10, 0
    cat_small   db "Category for 0: Small Positive", 10, 0
    cat_med     db "Category for 0: Medium Positive", 10, 0
    cat_large   db "Category for 100: Large Positive", 10, 0

section .text
    global _start

%macro print 2
    mov eax, 4
    mov ebx, 1
    mov ecx, %1
    mov edx, %2
    int 0x80
%endmacro

; ============================================================
; ฟังก์ชัน: classify_grade
; Input: EAX = score (0-100)
; Output: EAX = grade ('A','B','C','D','F')
;
; C equivalent:
;   if (score >= 90) return 'A';
;   else if (score >= 80) return 'B';
;   else if (score >= 70) return 'C';
;   else if (score >= 60) return 'D';
;   else return 'F';
; ============================================================
classify_grade:
    ; ตรวจสอบ A: score >= 90
    cmp eax, 90         ; เปรียบเทียบ score กับ 90
    jl  not_a           ; ถ้า score < 90 ไปตรวจ B
    mov eax, 'A'        ; score >= 90: return 'A'
    ret

not_a:
    ; ตรวจสอบ B: 80 <= score < 90
    cmp eax, 80         ; เปรียบเทียบ score กับ 80
    jl  not_b           ; ถ้า score < 80 ไปตรวจ C
    mov eax, 'B'        ; 80 <= score < 90: return 'B'
    ret

not_b:
    ; ตรวจสอบ C: 70 <= score < 80
    cmp eax, 70         ; เปรียบเทียบ score กับ 70
    jl  not_c
    mov eax, 'C'
    ret

not_c:
    ; ตรวจสอบ D: 60 <= score < 70
    cmp eax, 60         ; เปรียบเทียบ score กับ 60
    jl  not_d
    mov eax, 'D'
    ret

not_d:
    ; Default: F (score < 60)
    mov eax, 'F'
    ret

; ============================================================
; ฟังก์ชัน: classify_number
; Input: EAX = number (signed integer)
; Output: จะเรียกพิมพ์ผลโดยตรง
;
; C equivalent:
;   if (n < 0) print("Negative");
;   else if (n == 0) print("Zero");
;   else if (n < 50) print("Small Positive");
;   else if (n < 100) print("Medium Positive");
;   else print("Large Positive");
; ============================================================
classify_number:
    push ebx            ; บันทึก EBX (caller-saved register)
    mov ebx, eax        ; เก็บค่า input ไว้ใน EBX

    ; Branch 1: n < 0
    cmp ebx, 0          ; เปรียบเทียบกับ 0
    jge not_negative    ; ถ้า >= 0 ไปตรวจเงื่อนไขถัดไป
    print cat_neg, 27   ; พิมพ์ "Negative"
    jmp classify_done

not_negative:
    ; Branch 2: n == 0
    cmp ebx, 0
    jne not_zero        ; ถ้าไม่ใช่ 0 ไปตรวจเงื่อนไขถัดไป
    print cat_zero, 21
    jmp classify_done

not_zero:
    ; Branch 3: 0 < n < 50
    cmp ebx, 50
    jge not_small       ; ถ้า >= 50 ไปตรวจเงื่อนไขถัดไป
    print cat_small, 31
    jmp classify_done

not_small:
    ; Branch 4: 50 <= n < 100
    cmp ebx, 100
    jge not_medium      ; ถ้า >= 100 ไปตรวจเงื่อนไขถัดไป
    print cat_med, 32
    jmp classify_done

not_medium:
    ; Default: n >= 100
    print cat_large, 33

classify_done:
    pop ebx             ; คืนค่า EBX
    ret

_start:
    ; ============================================================
    ; ทดสอบ classify_grade กับ score = 85 (ควรได้ B)
    ; ============================================================
    mov eax, 85         ; score = 85
    call classify_grade  ; เรียก classify_grade, ผลลัพธ์อยู่ใน EAX

    ; ตรวจสอบว่าได้ grade อะไร และพิมพ์ข้อความที่เหมาะสม
    cmp eax, 'A'
    je  print_85_a
    cmp eax, 'B'
    je  print_85_b
    cmp eax, 'C'
    je  print_85_c
    cmp eax, 'D'
    je  print_85_d
    jmp print_85_f

print_85_a:
    print grade_a, 22
    jmp test_score55

print_85_b:
    print grade_b, 22
    jmp test_score55

print_85_c:
    print grade_c, 22
    jmp test_score55

print_85_d:
    print grade_d, 22
    jmp test_score55

print_85_f:
    print grade_f, 22

    ; ============================================================
    ; ทดสอบ classify_grade กับ score = 55 (ควรได้ F)
    ; ============================================================
test_score55:
    mov eax, 55         ; score = 55
    call classify_grade

    cmp eax, 'A'
    je  print_55_a
    cmp eax, 'B'
    je  print_55_b
    cmp eax, 'C'
    je  print_55_c
    cmp eax, 'D'
    je  print_55_d
    jmp print_55_f

print_55_a:
    print grade_a2, 22
    jmp test_categories

print_55_b:
    print grade_b2, 22
    jmp test_categories

print_55_c:
    print grade_c2, 22
    jmp test_categories

print_55_d:
    print grade_d2, 22
    jmp test_categories

print_55_f:
    print grade_f2, 22

    ; ============================================================
    ; ทดสอบ classify_number กับค่าต่างๆ
    ; ============================================================
test_categories:
    mov eax, -10        ; ทดสอบ Negative
    call classify_number

    mov eax, 0          ; ทดสอบ Zero
    call classify_number

    mov eax, 100        ; ทดสอบ Large Positive
    call classify_number

    ; ออกจากโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 if_else_demo.asm -o if_else_demo.o
ld -m elf_i386 if_else_demo.o -o if_else_demo
./if_else_demo
```

**Expected Output:**
```
Grade for score 85: B
Grade for score 55: F
Category for -10: Negative
Category for 0: Zero
Category for 100: Large Positive
```

---

### Example 4: Jump Table สำหรับ Switch-Case

**ไฟล์:** `jump_table.asm`

```nasm
; ============================================================
; jump_table.asm - การสร้างและใช้งาน Jump Table
; แสดงการ implement switch-case statement ด้วย jump table
; ซึ่งมีประสิทธิภาพ O(1) ไม่ว่าจะมีกี่ case ก็ตาม
;
; วิธีคอมไพล์และรัน:
;   nasm -f elf32 jump_table.asm -o jump_table.o
;   ld -m elf_i386 jump_table.o -o jump_table
;   ./jump_table
;
; Expected output:
;   Day 0: Sunday
;   Day 1: Monday
;   Day 2: Tuesday
;   Day 3: Wednesday
;   Day 4: Thursday
;   Day 5: Friday
;   Day 6: Saturday
;   Day 7: Invalid day
; ============================================================

section .data
    ; ข้อความสำหรับแต่ละวัน
    day_sun     db "Day 0: Sunday", 10
    day_sun_len equ $ - day_sun

    day_mon     db "Day 1: Monday", 10
    day_mon_len equ $ - day_mon

    day_tue     db "Day 2: Tuesday", 10
    day_tue_len equ $ - day_tue

    day_wed     db "Day 3: Wednesday", 10
    day_wed_len equ $ - day_wed

    day_thu     db "Day 4: Thursday", 10
    day_thu_len equ $ - day_thu

    day_fri     db "Day 5: Friday", 10
    day_fri_len equ $ - day_fri

    day_sat     db "Day 6: Saturday", 10
    day_sat_len equ $ - day_sat

    day_inv     db "Day 7: Invalid day", 10
    day_inv_len equ $ - day_inv

    ; ============================================================
    ; Jump Table: array ของ function pointers
    ; แต่ละ entry คือ address ของ handler สำหรับ case นั้น
    ; ============================================================
    jump_table:
        dd case_sunday      ; index 0
        dd case_monday      ; index 1
        dd case_tuesday     ; index 2
        dd case_wednesday   ; index 3
        dd case_thursday    ; index 4
        dd case_friday      ; index 5
        dd case_saturday    ; index 6

section .text
    global _start

; ============================================================
; ฟังก์ชัน: print_day
; Input: EAX = day number (0=Sunday, 6=Saturday)
; Output: พิมพ์ชื่อวัน
;
; C equivalent:
;   switch (day) {
;     case 0: puts("Sunday"); break;
;     case 1: puts("Monday"); break;
;     ... (6 cases)
;     default: puts("Invalid day"); break;
;   }
; ============================================================
print_day:
    ; ขั้นตอนที่ 1: Bounds checking - ตรวจสอบว่า input อยู่ใน range
    cmp eax, 6          ; เปรียบเทียบ day กับ maximum valid value (6)
    ja  default_case    ; ถ้า day > 6 (unsigned) ไป default case
                        ; ใช้ JA (unsigned) เพราะ day เป็น non-negative

    ; ขั้นตอนที่ 2: Table lookup - คำนวณ address จาก jump table
    ; jump_table + (day * 4) = address ของ pointer ที่ต้องการ
    ; (คูณด้วย 4 เพราะแต่ละ pointer ขนาด 4 bytes ใน 32-bit mode)
    jmp [jump_table + eax*4]    ; กระโดดไปตาม index ใน table
                                ; CPU จะ: อ่าน address จาก [jump_table + eax*4]
                                ; แล้ว jump ไปที่ address นั้น

    ; ============================================================
    ; Case handlers: แต่ละ case จะพิมพ์ข้อความและ return
    ; ============================================================
case_sunday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_sun
    mov edx, day_sun_len
    int 0x80
    ret

case_monday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_mon
    mov edx, day_mon_len
    int 0x80
    ret

case_tuesday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_tue
    mov edx, day_tue_len
    int 0x80
    ret

case_wednesday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_wed
    mov edx, day_wed_len
    int 0x80
    ret

case_thursday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_thu
    mov edx, day_thu_len
    int 0x80
    ret

case_friday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_fri
    mov edx, day_fri_len
    int 0x80
    ret

case_saturday:
    mov eax, 4
    mov ebx, 1
    mov ecx, day_sat
    mov edx, day_sat_len
    int 0x80
    ret

default_case:
    ; กรณีที่ day ไม่ถูกต้อง (out of range)
    mov eax, 4
    mov ebx, 1
    mov ecx, day_inv
    mov edx, day_inv_len
    int 0x80
    ret

_start:
    ; วนลูปทดสอบทุก day (0-7)
    mov esi, 0          ; ESI = counter (day number)

test_loop:
    cmp esi, 7          ; ทดสอบ 8 ค่า (0-7)
    ja  exit_main       ; ถ้า esi > 7 ออกจาก loop

    mov eax, esi        ; โหลด day number เข้า EAX
    call print_day      ; เรียกฟังก์ชัน print_day

    inc esi             ; เพิ่ม counter
    jmp test_loop       ; วนกลับ

exit_main:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 jump_table.asm -o jump_table.o
ld -m elf_i386 jump_table.o -o jump_table
./jump_table
```

**Expected Output:**
```
Day 0: Sunday
Day 1: Monday
Day 2: Tuesday
Day 3: Wednesday
Day 4: Thursday
Day 5: Friday
Day 6: Saturday
Day 7: Invalid day
```

---

### Example 5: Short-Circuit Evaluation และ Conditional Move

**ไฟล์:** `short_circuit_cmov.asm`

```nasm
; ============================================================
; short_circuit_cmov.asm - Short-circuit evaluation และ CMOV
; แสดงการ implement logical AND/OR แบบ short-circuit
; และการใช้ CMOVcc แทน conditional jump เพื่อประสิทธิภาพที่ดี
;
; วิธีคอมไพล์และรัน:
;   nasm -f elf32 short_circuit_cmov.asm -o short_circuit_cmov.o
;   ld -m elf_i386 short_circuit_cmov.o -o short_circuit_cmov
;   ./short_circuit_cmov
;
; Expected output:
;   Short-circuit AND test:
;   5 > 0 AND 10 > 5: TRUE
;   0 > 5 AND 10 > 5: FALSE (short-circuited)
;   Short-circuit OR test:
;   5 > 0 OR 0 > 5: TRUE (short-circuited)
;   0 > 5 OR 0 > 10: FALSE
;   CMOV demo:
;   max(10, 20) = 20
;   max(30, 15) = 30
;   abs(-42) = 42
;   abs(17) = 17
; ============================================================

section .data
    ; Short-circuit AND messages
    hdr_and     db "Short-circuit AND test:", 10, 0
    msg_and_tt  db "5 > 0 AND 10 > 5: TRUE", 10, 0
    msg_and_ff  db "0 > 5 AND 10 > 5: FALSE (short-circuited)", 10, 0

    ; Short-circuit OR messages
    hdr_or      db "Short-circuit OR test:", 10, 0
    msg_or_t    db "5 > 0 OR 0 > 5: TRUE (short-circuited)", 10, 0
    msg_or_f    db "0 > 5 OR 0 > 10: FALSE", 10, 0

    ; CMOV messages
    hdr_cmov    db "CMOV demo:", 10, 0
    msg_max1    db "max(10, 20) = 20", 10, 0
    msg_max2    db "max(30, 15) = 30", 10, 0
    msg_abs1    db "abs(-42) = 42", 10, 0
    msg_abs2    db "abs(17) = 17", 10, 0

section .text
    global _start

%macro print 2
    mov eax, 4
    mov ebx, 1
    mov ecx, %1
    mov edx, %2
    int 0x80
%endmacro

; ============================================================
; Short-Circuit AND evaluation
; C: if (a > 0 && b > c) { ... }
;
; หลักการ: ถ้าเงื่อนไขแรกเป็น FALSE ไม่ต้องตรวจเงื่อนไขที่สอง
; เพราะ FALSE AND anything = FALSE เสมอ
; ============================================================

; Test: 5 > 0 AND 10 > 5 (ทั้งคู่จริง)
and_test1:
    mov eax, 5          ; a = 5
    mov ebx, 0          ; threshold = 0
    mov ecx, 10         ; b = 10
    mov edx, 5          ; c = 5

    ; เงื่อนไขที่ 1: a > 0 (เงื่อนไขแรก)
    cmp eax, ebx        ; เปรียบเทียบ a กับ 0
    jle and1_false      ; ถ้า a <= 0 (เงื่อนไขแรก false)
                        ; SHORT-CIRCUIT: ข้ามไป false ทันที ไม่ตรวจเงื่อนไขที่ 2!

    ; เงื่อนไขที่ 2: b > c (เงื่อนไขที่สอง - ตรวจเฉพาะเมื่อเงื่อนไขแรก true)
    cmp ecx, edx        ; เปรียบเทียบ b กับ c
    jle and1_false      ; ถ้า b <= c --> false

    ; ถึงตรงนี้: ทั้งสองเงื่อนไขเป็น true
    print msg_and_tt, 23
    jmp and_test2

and1_false:
    print msg_and_ff, 42

and_test2:
    ; Test: 0 > 5 AND 10 > 5 (เงื่อนไขแรก false)
    mov eax, 0          ; a = 0
    mov ebx, 5          ; threshold = 5

    cmp eax, ebx        ; เปรียบเทียบ 0 กับ 5
    jle and2_false      ; 0 <= 5 --> false! short-circuit ทันที
                        ; เงื่อนไขที่สองจะไม่ถูกประเมินเลย

    ; ถ้ามาถึงตรงนี้: เงื่อนไขแรก true (จะไม่เกิดขึ้นในกรณีนี้)
    print msg_and_tt, 23
    jmp or_section

and2_false:
    print msg_and_ff, 42

    ; ============================================================
    ; Short-Circuit OR evaluation
    ; C: if (a > 0 || b > c) { ... }
    ;
    ; หลักการ: ถ้าเงื่อนไขแรกเป็น TRUE ไม่ต้องตรวจเงื่อนไขที่สอง
    ; เพราะ TRUE OR anything = TRUE เสมอ
    ; ============================================================
or_section:
    print hdr_or, 23

    ; Test: 5 > 0 OR 0 > 5 (เงื่อนไขแรก true)
    mov eax, 5          ; a = 5
    mov ebx, 0          ; threshold = 0

    cmp eax, ebx        ; เปรียบเทียบ a กับ 0
    jg  or1_true        ; ถ้า a > 0 --> true! short-circuit ทันที
                        ; เงื่อนไขที่สองจะไม่ถูกประเมินเลย

    ; เงื่อนไขที่สอง: 0 > 5 (จะถูกตรวจเฉพาะเมื่อเงื่อนไขแรก false)
    mov ecx, 0
    mov edx, 5
    cmp ecx, edx
    jg  or1_true        ; ถ้า 0 > 5 --> true (ไม่เกิดขึ้น)
    print msg_or_f, 23
    jmp or_test2

or1_true:
    print msg_or_t, 39

or_test2:
    ; Test: 0 > 5 OR 0 > 10 (ทั้งคู่ false)
    mov eax, 0
    mov ebx, 5

    cmp eax, ebx        ; 0 > 5? NO
    jg  or2_true        ; เงื่อนไขแรก false, ไปตรวจที่สอง

    mov ecx, 0
    mov edx, 10
    cmp ecx, edx        ; 0 > 10? NO
    jg  or2_true

    print msg_or_f, 23  ; ทั้งคู่ false
    jmp cmov_section

or2_true:
    print msg_or_t, 39

    ; ============================================================
    ; CMOVcc - Conditional Move (ไม่มี branch, ดีสำหรับ performance)
    ;
    ; ข้อดี: CPU ไม่ต้องทำ branch prediction
    ;        เหมาะสำหรับกรณีที่ branch มีโอกาสเกิดขึ้น 50/50
    ;
    ; ข้อเสีย: ต้องคำนวณทั้งสองค่าก่อนเสมอ
    ;          ไม่เหมาะถ้าการคำนวณค่าหนึ่งมี side effects หรือแพงมาก
    ; ============================================================
cmov_section:
    print hdr_cmov, 11

    ; --- max(a, b) function ด้วย CMOV ---
    ; C: result = (a > b) ? a : b;

    ; max(10, 20) = 20
    mov eax, 10         ; eax = a = 10
    mov ebx, 20         ; ebx = b = 20
    cmp eax, ebx        ; เปรียบเทียบ a กับ b
    cmovl eax, ebx      ; ถ้า a < b: EAX = EBX (เลือก b)
                        ; ถ้า a >= b: EAX ไม่เปลี่ยน (เลือก a)
    ; EAX ตอนนี้ = max(10, 20) = 20
    print msg_max1, 17

    ; max(30, 15) = 30
    mov eax, 30         ; eax = a = 30
    mov ebx, 15         ; ebx = b = 15
    cmp eax, ebx        ; 30 > 15
    cmovl eax, ebx      ; 30 < 15? NO --> EAX ไม่เปลี่ยน = 30
    ; EAX ตอนนี้ = max(30, 15) = 30
    print msg_max2, 17

    ; --- abs(x) function ด้วย CMOV ---
    ; C: result = (x < 0) ? -x : x;

    ; abs(-42)
    mov eax, -42        ; eax = x = -42
    mov ebx, eax        ; ebx = copy of x
    neg ebx             ; ebx = -x = 42
    cmp eax, 0          ; เปรียบเทียบ x กับ 0
    cmovl eax, ebx      ; ถ้า x < 0: EAX = -x = 42
                        ; ถ้า x >= 0: EAX ไม่เปลี่ยน = x
    ; EAX = abs(-42) = 42
    print msg_abs1, 14

    ; abs(17)
    mov eax, 17
    mov ebx, eax
    neg ebx             ; ebx = -17
    cmp eax, 0
    cmovl eax, ebx      ; 17 < 0? NO --> EAX ไม่เปลี่ยน = 17
    ; EAX = abs(17) = 17
    print msg_abs2, 13

    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 short_circuit_cmov.asm -o short_circuit_cmov.o
ld -m elf_i386 short_circuit_cmov.o -o short_circuit_cmov
./short_circuit_cmov
```

**Expected Output:**
```
Short-circuit AND test:
5 > 0 AND 10 > 5: TRUE
0 > 5 AND 10 > 5: FALSE (short-circuited)
Short-circuit OR test:
5 > 0 OR 0 > 5: TRUE (short-circuited)
0 > 5 OR 0 > 10: FALSE
CMOV demo:
max(10, 20) = 20
max(30, 15) = 30
abs(-42) = 42
abs(17) = 17
```

---

### Example 6: JCXZ/JECXZ และ Loop Control

**ไฟล์:** `loop_control.asm`

```nasm
; ============================================================
; loop_control.asm - การใช้ JECXZ และ loop control patterns
; แสดง pattern ต่างๆ สำหรับ loop ที่พบบ่อยใน Assembly
;
; วิธีคอมไพล์และรัน:
;   nasm -f elf32 loop_control.asm -o loop_control.o
;   ld -m elf_i386 loop_control.o -o loop_control
;   ./loop_control
;
; Expected output:
;   Sum 1..10 = 55
;   Factorial 5! = 120
;   First even in [3,7,4,9,2,8]: found=4
;   Empty array: no elements
; ============================================================

section .data
    msg_sum     db "Sum 1..10 = 55", 10, 0
    msg_fact    db "Factorial 5! = 120", 10, 0
    msg_found   db "First even in [3,7,4,9,2,8]: found=4", 10, 0
    msg_empty   db "Empty array: no elements", 10, 0
    msg_wrong   db "[WRONG RESULT]", 10, 0

    ; Array สำหรับทดสอบ
    test_array  dd 3, 7, 4, 9, 2, 8
    array_len   equ 6

section .text
    global _start

%macro print 2
    mov eax, 4
    mov ebx, 1
    mov ecx, %1
    mov edx, %2
    int 0x80
%endmacro

_start:
    ; ============================================================
    ; Test 1: คำนวณผลรวม 1+2+...+10 = 55
    ; ใช้ ECX เป็น counter, JECXZ เพื่อตรวจสอบ ECX=0
    ; ============================================================
    mov eax, 0          ; EAX = accumulator (ผลรวม)
    mov ecx, 10         ; ECX = counter (จาก 10 ลงมา)

sum_loop:
    jecxz   sum_done    ; ถ้า ECX == 0 ออกจาก loop
                        ; JECXZ ตรวจ ECX โดยตรง ไม่ affect flags
    add eax, ecx        ; ผลรวม += ECX
    dec ecx             ; ECX--
    jmp sum_loop        ; วนกลับ

sum_done:
    ; EAX ควรจะเป็น 55
    cmp eax, 55
    je  sum_correct
    print msg_wrong, 15
    jmp test2

sum_correct:
    print msg_sum, 15

    ; ============================================================
    ; Test 2: คำนวณ 5! = 120
    ; Pattern: multiply-loop ด้วย LOOP instruction
    ; ============================================================
test2:
    mov eax, 1          ; EAX = result (เริ่มต้น = 1)
    mov ecx, 5          ; ECX = 5 (จะคูณ 5 ครั้ง: 5*4*3*2*1)

factorial_loop:
    mul ecx             ; EAX = EAX * ECX (unsigned multiply)
                        ; ผล: 1*5=5, 5*4=20, 20*3=60, 60*2=120, 120*1=120
    loop factorial_loop ; ECX-- และ ถ้า ECX != 0 jump กลับ
                        ; (LOOP เหมือน DEC ECX + JNZ)

    ; EAX ควรจะเป็น 120
    cmp eax, 120
    je  fact_correct
    print msg_wrong, 15
    jmp test3

fact_correct:
    print msg_fact, 19

    ; ============================================================
    ; Test 3: หาตัวเลข even ตัวแรกใน array
    ; Pattern: linear search ด้วย JECXZ guard
    ; ============================================================
test3:
    mov ecx, array_len  ; ECX = ความยาว array
    mov esi, test_array ; ESI = pointer ไปยัง array

    ; ตรวจสอบ empty array ด้วย JECXZ ก่อนเข้า loop
    jecxz   empty_array ; ถ้า array ว่างเปล่า ข้ามไปทันที

search_loop:
    mov eax, [esi]      ; โหลดตัวเลขปัจจุบันจาก array
    test eax, 1         ; ตรวจ bit 0 (เลขคี่/คู่)
    jz   found_even     ; ถ้า bit 0 = 0 --> even number! found!

    add esi, 4          ; เลื่อน pointer ไปยัง element ถัดไป
    dec ecx             ; ECX--
    jecxz not_found     ; ถ้า ECX == 0 (ค้นหนมหมด array แล้ว) ไม่พบ
    jmp search_loop     ; วนกลับค้นต่อ

found_even:
    ; พบ even number แล้ว (EAX = 4)
    cmp eax, 4          ; ตรวจสอบว่าเป็น 4 (ตัวแรกที่พบ)
    je  search_correct
    print msg_wrong, 15
    jmp test4

search_correct:
    print msg_found, 38
    jmp test4

not_found:
    print msg_wrong, 15

    ; ============================================================
    ; Test 4: ทดสอบ JECXZ กับ empty array (ECX=0)
    ; ============================================================
test4:
    mov ecx, 0          ; จำลอง empty array

    jecxz   empty_array ; ECX=0 --> jump ไปทันที ไม่ต้องเข้า loop
    ; loop body จะไม่ถูก execute เลย
    print msg_wrong, 15
    jmp main_exit

empty_array:
    print msg_empty, 25

main_exit:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**วิธีคอมไพล์และรัน:**
```bash
nasm -f elf32 loop_control.asm -o loop_control.o
ld -m elf_i386 loop_control.o -o loop_control
./loop_control
```

**Expected Output:**
```
Sum 1..10 = 55
Factorial 5! = 120
First even in [3,7,4,9,2,8]: found=4
Empty array: no elements
```

---

## Common Mistakes and Pitfalls {#common-mistakes}

### ข้อผิดพลาดที่ 1: ใช้ Signed Jcc กับ Unsigned Data

```nasm
; ผิด: ใช้ JL/JG กับ addresses หรือ unsigned values
MOV EAX, 0xFFFFFFF0    ; = ค่า unsigned ขนาดใหญ่ หรือ -16 (signed)
CMP EAX, 0x10
JG  too_large           ; WRONG! ถ้าตีความแบบ signed, 0xFFFFFFF0 = -16 < 16

; ถูก: ใช้ JA/JB สำหรับ unsigned comparison
CMP EAX, 0x10
JA  too_large           ; CORRECT! unsigned 0xFFFFFFF0 > 0x10
```

### ข้อผิดพลาดที่ 2: ลืม bounds check ก่อน Jump Table

```nasm
; ผิด: ไม่ตรวจสอบ bounds ก่อนใช้ jump table
JMP [jump_table + EAX*4]    ; DANGEROUS! ถ้า EAX ใหญ่เกินไป

; ถูก: ตรวจสอบก่อนเสมอ
CMP EAX, max_index
JA  default_case            ; ออกไป default ถ้า out of range
JMP [jump_table + EAX*4]    ; ปลอดภัยแล้ว
```

### ข้อผิดพลาดที่ 3: Instruction ระหว่าง CMP และ Jcc ทำลาย Flags

```nasm
; ผิด: มี instruction ที่ affect flags อยู่ระหว่าง CMP กับ Jcc
CMP EAX, EBX
ADD ECX, 1          ; ADD เปลี่ยน flags!!! ZF, SF, OF เปลี่ยนตาม ADD
JE  equal_case      ; WRONG! flags ตอนนี้เป็นของ ADD ไม่ใช่ CMP

; ถูก: ไม่มี instruction ที่ affect flags ระหว่าง CMP กับ Jcc
CMP EAX, EBX
JE  equal_case      ; CORRECT: ทันทีหลัง CMP
; ทำ ADD ECX ที่นี่หลัง branch

; หรือถ้าต้องการ ADD ก่อน ให้บันทึก flag ด้วย PUSHF/POPF
CMP EAX, EBX
PUSHF               ; บันทึก flags
ADD ECX, 1
POPF                ; คืนค่า flags
JE  equal_case      ; ใช้ flags ที่บันทึกไว้
```

### ข้อผิดพลาดที่ 4: Infinite Loop จาก Jump ที่ไม่มี Exit Condition

```nasm
; ผิด: ไม่มีทางออกจาก loop
loop_start:
    INC EAX
    CMP EAX, 10
    JE  loop_done    ; จะ jump ออกเมื่อ EAX = 10
    JMP loop_start   ; กลับไป loop ใหม่ -- แต่ถ้าเริ่มด้วย EAX > 10?
                     ; Loop จะวนไปเรื่อยๆ จนล้น และกลับมาเป็น 10!
loop_done:

; ถูก: ใช้ JB หรือ JL เพื่อป้องกัน
loop_start:
    INC EAX
    CMP EAX, 10
    JB  loop_start   ; วนเฉพาะเมื่อ EAX < 10 (unsigned)
; ออกเมื่อ EAX >= 10
```

### ข้อผิดพลาดที่ 5: Short Jump Distance Overflow

```nasm
; ผิด: label อยู่ไกลเกิน 127 bytes แต่ใช้ short jump
JMP short far_away_label    ; Error! short jump range: -128 to +127 bytes

; ถูก: ใช้ near jump
JMP far_away_label          ; NASM จะเลือก short/near อัตโนมัติ

; หรือระบุ near อย่างชัดเจน
JMP near far_away_label
```

### ข้อผิดพลาดที่ 6: ลืมว่า TEST ไม่ set CF

```nasm
; ผิด: ใช้ JC หลัง TEST (TEST ไม่ set CF!)
TEST EAX, EBX
JC  carry_set        ; WRONG! TEST เซ็ต ZF, SF, PF เท่านั้น ไม่เซ็ต CF

; ถูก: ใช้ JNZ หลัง TEST
TEST EAX, EBX
JNZ bits_set         ; CORRECT: ตรวจ ZF แทน
```

### ข้อผิดพลาดที่ 7: ความสับสนระหว่าง JE/JZ และ JA/JG

```nasm
; JE และ JZ ใช้ทดแทนกันได้เสมอ (opcode เดียวกัน)
; แต่ JA (unsigned >) และ JG (signed >) ต่างกัน!

MOV AL, 0xFF       ; = 255 unsigned, = -1 signed
CMP AL, 0          ; 
JG  positive       ; ผิด! 0xFF ตีความ signed = -1 < 0 ไม่ jump
JA  nonzero        ; ถูก! 0xFF ตีความ unsigned = 255 > 0 jump
```

---

## Advanced Techniques {#advanced-techniques}

### เทคนิคที่ 1: Branchless Max/Min ด้วย CMOV

```nasm
; ============================================================
; Branchless maximum ด้วย CMOVcc
; เร็วกว่า conditional jump เพราะไม่มี branch misprediction
; ============================================================

; max(eax, ebx) --> result ใน EAX
; C: if (ebx > eax) eax = ebx;
branchless_max:
    cmp eax, ebx
    cmovl eax, ebx      ; CMOVL: move ถ้า less (signed)
    ret

; min(eax, ebx) --> result ใน EAX
branchless_min:
    cmp eax, ebx
    cmovg eax, ebx      ; CMOVG: move ถ้า greater (signed)
    ret

; Unsigned max
branchless_umax:
    cmp eax, ebx
    cmovb eax, ebx      ; CMOVB: move ถ้า below (unsigned)
    ret
```

### เทคนิคที่ 2: Branchless Absolute Value

```nasm
; ============================================================
; Branchless abs() โดยใช้ arithmetic shift trick
; เทคนิคนี้เร็วกว่า conditional branch มาก
; ============================================================
branchless_abs:
    ; Input: EAX = signed integer
    ; Output: EAX = |EAX|

    mov ebx, eax        ; EBX = copy ของ EAX
    sar ebx, 31         ; Arithmetic shift right 31 bits
                        ; ถ้า EAX >= 0: EBX = 0x00000000 (all zeros)
                        ; ถ้า EAX < 0:  EBX = 0xFFFFFFFF (all ones = -1)

    xor eax, ebx        ; ถ้า positive: EAX XOR 0 = EAX (ไม่เปลี่ยน)
                        ; ถ้า negative: EAX XOR 0xFFFFFFFF = ~EAX (bitwise NOT)

    sub eax, ebx        ; ถ้า positive: EAX - 0 = EAX (ไม่เปลี่ยน)
                        ; ถ้า negative: ~EAX - (-1) = ~EAX + 1 = -EAX

    ; ผลลัพธ์: |EAX| โดยไม่ใช้ branch เลย!
    ret
```

### เทคนิคที่ 3: Flag Manipulation โดยตรง

```nasm
; ============================================================
; การ set/clear/toggle flags โดยตรง
; ============================================================

; Set Carry Flag
STC                 ; CF = 1

; Clear Carry Flag
CLC                 ; CF = 0

; Toggle Carry Flag (ไม่มี instruction โดยตรง ต้องใช้ CMC)
CMC                 ; Complement CF (toggle)

; Clear Direction Flag (ใช้ก่อน string operations)
CLD                 ; DF = 0 (string ops ไปทิศทาง forward)

; Set Direction Flag
STD                 ; DF = 1 (string ops ไปทิศทาง backward)

; Push/Pop EFLAGS
PUSHF               ; push FLAGS register onto stack
POPF                ; pop FLAGS register from stack

PUSHFD              ; push EFLAGS (32-bit) onto stack
POPFD               ; pop EFLAGS (32-bit) from stack
```

### เทคนิคที่ 4: Computed Goto (Runtime-determined Jump)

```nasm
; ============================================================
; Computed Goto - กระโดดไปที่ address ที่คำนวณในเวลา runtime
; ใช้สำหรับ dispatch tables, interpreters, state machines
; ============================================================

; Virtual Machine Interpreter pattern
vm_dispatch:
    ; EBX = pointer ไปยัง opcode
    movzx eax, byte [ebx]   ; อ่าน opcode (0-255)
    inc ebx                  ; เลื่อน pointer ไปยัง instruction ถัดไป

    ; Dispatch table สำหรับ 256 opcodes
    jmp [vm_dispatch_table + eax*4]

vm_dispatch_table:
    dd op_nop        ; opcode 0x00
    dd op_add        ; opcode 0x01
    dd op_sub        ; opcode 0x02
    dd op_mul        ; opcode 0x03
    ; ... (สำหรับ opcodes 0-255)
    times 252 dd op_invalid   ; fill remaining with invalid handler
```

### เทคนิคที่ 5: Duff's Device (Loop Unrolling)

```nasm
; ============================================================
; Assembly version ของ Duff's Device
; เทคนิคนี้ลด loop overhead สำหรับ data copy
; ============================================================

; ต้องการ copy ECX bytes จาก ESI ไปยัง EDI
; ทำ loop unrolling 4x เพื่อลด branch
duffs_copy:
    push ecx
    push esi
    push edi

    ; คำนวณ remaining bytes (ECX mod 4)
    mov eax, ecx
    and eax, 3          ; EAX = ECX % 4 (เศษจากหารด้วย 4)
    shr ecx, 2          ; ECX = ECX / 4 (จำนวนรอบหลัก)

    ; Handle remaining 0-3 bytes ก่อน
    test eax, eax
    jz  main_copy_loop

    dec eax
    jz  copy_1_extra

    dec eax
    jz  copy_2_extra

    ; copy 3 extra bytes
    movsb
copy_2_extra:
    movsb
copy_1_extra:
    movsb

main_copy_loop:
    jecxz  copy_done    ; ถ้า ECX = 0 เสร็จแล้ว
    movsd               ; copy 4 bytes (DWORD) และ advance ESI/EDI
    loop  main_copy_loop

copy_done:
    pop edi
    pop esi
    pop ecx
    ret
```

### เทคนิคที่ 6: Conditional Move Matrix (CMOVcc ทั้งหมด)

```nasm
; ============================================================
; CMOVcc instructions ทั้งหมด (เหมือน Jcc แต่เป็น conditional move)
; ============================================================

; Signed comparisons
CMOVE   dest, src   ; Move if Equal          (ZF=1)
CMOVNE  dest, src   ; Move if Not Equal      (ZF=0)
CMOVL   dest, src   ; Move if Less           (SF≠OF)
CMOVLE  dest, src   ; Move if Less/Equal     (ZF=1 หรือ SF≠OF)
CMOVG   dest, src   ; Move if Greater        (ZF=0 และ SF=OF)
CMOVGE  dest, src   ; Move if Greater/Equal  (SF=OF)

; Unsigned comparisons
CMOVB   dest, src   ; Move if Below          (CF=1)
CMOVBE  dest, src   ; Move if Below/Equal    (CF=1 หรือ ZF=1)
CMOVA   dest, src   ; Move if Above          (CF=0 และ ZF=0)
CMOVAE  dest, src   ; Move if Above/Equal    (CF=0)

; Flag-based
CMOVS   dest, src   ; Move if Sign           (SF=1)
CMOVNS  dest, src   ; Move if Not Sign       (SF=0)
CMOVO   dest, src   ; Move if Overflow       (OF=1)
CMOVNO  dest, src   ; Move if No Overflow    (OF=0)
CMOVP   dest, src   ; Move if Parity         (PF=1)
CMOVNP  dest, src   ; Move if No Parity      (PF=0)

; หมายเหตุ: CMOVcc ใช้ได้เฉพาะกับ 16/32/64-bit registers
; ไม่สามารถใช้กับ 8-bit registers (AL, BL, ฯลฯ)
; ไม่สามารถใช้กับ immediate values
; source สามารถเป็น memory ได้: CMOVL EAX, [mem]
```

### เทคนิคที่ 7: Performance — เมื่อไหร่ควรใช้ CMOV vs Jump

```
CMOV ดีกว่า Jump เมื่อ:
1. Branch มีโอกาสเกิดขึ้น ~50% (branch predictor ทำนายได้ยาก)
2. Code ทั้งสองด้านของ branch สั้นและเร็ว
3. ต้องการ branchless code (เช่นใน crypto, where timing matters)
4. Hot path ที่ใช้ข้อมูล unpredictable (เช่น searching random data)

Jump ดีกว่า CMOV เมื่อ:
1. Branch ถูกทำนายได้ง่าย (เช่น loop: ไป 99 ครั้ง, ออก 1 ครั้ง)
2. Code ด้านหนึ่งของ branch หนักมาก (CMOV ต้องทำทั้งสอง)
3. Branch มี side effects (เช่น function call, memory access)
4. Branch มีโอกาสเกิดขึ้น >> 90% หรือ << 10%

Modern CPU Branch Prediction Success Rate:
- Loop termination: ~99%+ (CPU รู้ว่า loop จะวน)
- Pattern-based: ~90%+ (ถ้า branch ทำซ้ำ pattern)
- Data-dependent: ~50-60% (ขึ้นกับ data)
```

---

## Exercises {#exercises}

### Exercise 1: Sign Classifier (ระดับพื้นฐาน)

**โจทย์:** เขียนฟังก์ชัน `classify_sign` ที่รับ signed 32-bit integer ใน EAX และคืนค่า:
- 1 ถ้าเป็นบวก (> 0)
- 0 ถ้าเป็นศูนย์ (= 0)
- -1 ถ้าเป็นลบ (< 0)

**Hint:**
```nasm
classify_sign:
    ; ทดสอบ EAX กับตัวเอง
    test eax, eax
    ; ใช้ JS สำหรับ negative
    ; ใช้ JZ สำหรับ zero
    ; ส่วนที่เหลือคือ positive
```

**Expected behavior:**
```
classify_sign(5)   = 1
classify_sign(0)   = 0
classify_sign(-3)  = -1
classify_sign(-1)  = -1
classify_sign(100) = 1
```

---

### Exercise 2: Clamp Function (ระดับกลาง)

**โจทย์:** เขียนฟังก์ชัน `clamp` ที่รับค่า signed integer ใน EAX และ clamp ให้อยู่ใน range [low, high]

```
Input: EAX = value
       EBX = low
       ECX = high
Output: EAX = clamp(value, low, high)
         = low  ถ้า value < low
         = high ถ้า value > high
         = value ถ้า low <= value <= high
```

**Hint:** ใช้ combination ของ CMP และ CMOV

**Expected behavior:**
```
clamp(5, 0, 10)   = 5
clamp(-5, 0, 10)  = 0
clamp(15, 0, 10)  = 10
clamp(0, 0, 10)   = 0
clamp(10, 0, 10)  = 10
```

---

### Exercise 3: Linear Search with Early Exit (ระดับกลาง)

**โจทย์:** เขียนฟังก์ชัน `find_value` ที่ค้นหาค่าใน array ของ DWORD และคืน index ที่พบ (หรือ -1 ถ้าไม่พบ)

```
Input:  ESI = pointer to array
        ECX = array length
        EAX = value to find
Output: EAX = index (0-based) ถ้าพบ
             = -1 ถ้าไม่พบ
```

**Hint:**
```nasm
find_value:
    ; ใช้ EDX เป็น index counter
    ; ตรวจ JECXZ สำหรับ empty array
    ; ใช้ LOOP สำหรับวน
    ; ใช้ JE เมื่อ [ESI + EDX*4] == EAX
```

---

### Exercise 4: Optimized Bubble Sort Comparison (ระดับสูง)

**โจทย์:** เขียน function `compare_and_swap` ที่ swap สองค่าถ้าตัวแรกมากกว่าตัวที่สอง (สำหรับ ascending sort) โดยใช้ CMOV แทน conditional branch

```
Input:  EAX = ค่าแรก
        EBX = ค่าที่สอง
Output: EAX = min(original_eax, original_ebx)
        EBX = max(original_eax, original_ebx)
```

**Hint:**
```nasm
compare_and_swap:
    ; บันทึกค่าเดิมไว้
    mov ecx, eax        ; ECX = copy of EAX
    mov edx, ebx        ; EDX = copy of EBX
    ; ใช้ CMOV เพื่อเลือกค่าที่ถูกต้องสำหรับ EAX และ EBX
    ; โดยไม่ใช้ conditional jump เลย
```

---

### Exercise 5: Jump Table สำหรับ Calculator (ระดับสูง)

**โจทย์:** สร้าง simple calculator ที่รับ opcode ใน EAX, สองตัวเลขใน EBX และ ECX แล้วคืนผลลัพธ์ใน EAX โดยใช้ jump table

```
Opcode 0: ADD  --> EAX = EBX + ECX
Opcode 1: SUB  --> EAX = EBX - ECX
Opcode 2: MUL  --> EAX = EBX * ECX
Opcode 3: AND  --> EAX = EBX AND ECX
Opcode 4: OR   --> EAX = EBX OR ECX
Opcode 5: XOR  --> EAX = EBX XOR ECX
Other:    NOP  --> EAX = 0 (invalid opcode)
```

**Hint:**
```nasm
section .data
    calc_table:
        dd op_add, op_sub, op_mul, op_and, op_or, op_xor
    CALC_OPS equ 6

calculator:
    cmp eax, CALC_OPS
    jae invalid_op
    jmp [calc_table + eax*4]
```

---

### Exercise 6 (Bonus): Implement strcmp ด้วย CMOV

**โจทย์:** เขียน string comparison function `my_strcmp` ที่เปรียบเทียบสอง null-terminated strings

```
Input:  ESI = pointer to string1
        EDI = pointer to string2
Output: EAX = 0  ถ้า str1 == str2
             > 0 ถ้า str1 > str2 (ค่า byte ที่ต่างกันครั้งแรก)
             < 0 ถ้า str1 < str2
```

**Hint:**
```nasm
my_strcmp:
.loop:
    movzx eax, byte [esi]   ; โหลด byte จาก str1
    movzx ecx, byte [edi]   ; โหลด byte จาก str2
    inc esi
    inc edi
    cmp al, cl
    jne .differ             ; ถ้าต่างกัน ออก
    test al, al
    jnz .loop               ; ถ้ายังไม่ null ไปต่อ
    ; ถ้า null ทั้งคู่ --> เท่ากัน
    xor eax, eax
    ret
.differ:
    sub eax, ecx            ; คืนค่าผลต่าง
    ret
```

---

## Summary {#summary}

### สรุปสิ่งที่เรียนรู้ใน Part 017

#### CMP และ TEST

| Instruction | ทำงานเหมือน | เก็บผลลัพธ์? | Flags ที่ set |
|-------------|-----------|------------|--------------|
| `CMP a, b` | `a - b` | ไม่ | ZF, SF, OF, CF, AF, PF |
| `TEST a, b` | `a AND b` | ไม่ | ZF, SF, PF (CF=0, OF=0) |

#### การเลือก Jcc ที่ถูกต้อง

```
ข้อมูล Signed  --> ใช้ JL, JLE, JG, JGE
ข้อมูล Unsigned --> ใช้ JB, JBE, JA, JAE
ทั้งสองแบบ     --> ใช้ JE/JZ, JNE/JNZ
```

#### Jump Types

```
Short Jump:    ±127 bytes   (1 byte offset)
Near Jump:     ±2GB         (4 byte offset)
Far Jump:      ข้าม segment  (OS/kernel only)
Indirect Jump: JMP [reg/mem] (computed goto)
```

#### Jump Table vs Chain of CMP

| | Jump Table | Chain of CMP |
|--|-----------|--------------|
| Time complexity | O(1) | O(n) |
| เหมาะสำหรับ | switch/case, dispatch | if/elif ไม่กี่เงื่อนไข |
| ต้องการ | contiguous integer range | ไม่ต้องการ |
| Code ซับซ้อน | ใช่ (ต้องมี table) | ไม่ |

#### CMOV vs Branch Performance

```
ใช้ CMOV เมื่อ: branch มี 50/50 chance หรือ branchless code required
ใช้ Jump เมื่อ: branch predictable หรือ code ด้านหนึ่งหนักมาก
```

#### Quick Reference: Flag States หลัง CMP A, B

```
A == B:  ZF=1, CF=0
A != B:  ZF=0
A < B (unsigned):   CF=1
A > B (unsigned):   CF=0, ZF=0
A < B (signed):     SF ≠ OF
A > B (signed):     SF = OF, ZF=0
```

### Checklist ก่อนใช้ Comparison/Jump

- [ ] ข้อมูลเป็น signed หรือ unsigned? → เลือก JL/JG หรือ JB/JA
- [ ] ไม่มี instruction ที่ affect flags ระหว่าง CMP และ Jcc
- [ ] Jump table มี bounds check ก่อนเสมอ
- [ ] Short jump ไม่เกิน ±127 bytes (ปล่อยให้ NASM เลือกเอง)
- [ ] JECXZ ใช้สำหรับ guard empty arrays ก่อนเข้า loop
- [ ] CMOV ไม่สามารถใช้กับ 8-bit registers

---

### Part ถัดไป

**Part 018: Loops และ String Operations** จะครอบคลุม:
- LOOP, LOOPE, LOOPNE instructions
- REP, REPE, REPNE prefixes
- String instructions: MOVSB/W/D, CMPSB/W/D, SCASB/W/D, STOSB/W/D, LODSB/W/D
- การใช้ Direction Flag (CLD/STD)
- Performance ของ REP prefix
- Implementing memcpy, memset, strlen, strcpy ใน Assembly

---

*Part 017 of Assembly Programming Course — จาก Basic ถึง World-Class*
*ครอบคลุม: CMP, TEST, Jcc (20+ variants), JMP (short/near/far/indirect), Jump Tables, if/else patterns, short-circuit evaluation, CMOV*
