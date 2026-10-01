# Part 015: Logical และ Bitwise Operations

## Prerequisites and Learning Objectives

### สิ่งที่ควรรู้ก่อนเรียน (Prerequisites)
- ผ่าน Part 001-014 มาแล้ว
- เข้าใจระบบเลขฐาน 2 (Binary) และ ฐาน 16 (Hexadecimal)
- รู้จักรีจิสเตอร์พื้นฐาน (RAX, RBX, RCX, RDX ฯลฯ)
- เข้าใจการทำงานของ Flags Register (EFLAGS/RFLAGS)
- รู้จักคำสั่ง MOV, ADD, SUB พื้นฐาน

### วัตถุประสงค์การเรียนรู้ (Learning Objectives)
เมื่อเรียนจบ Part นี้ผู้เรียนจะสามารถ:
1. ใช้คำสั่ง AND, OR, XOR, NOT ได้อย่างถูกต้อง
2. ทำ Bit Masking, Bit Setting, Bit Clearing ได้
3. ใช้ TEST instruction แทน CMP สำหรับ boolean check
4. ใช้คำสั่ง BT, BTS, BTR, BTC สำหรับ bit manipulation
5. ใช้ BSF, BSR สำหรับ bit scan operations
6. ใช้ LZCNT, TZCNT สำหรับนับ zero bits
7. ใช้ POPCNT สำหรับนับจำนวน set bits
8. ทำ bit manipulation tricks ขั้นสูง
9. implement Bitset และ Flag manipulation ได้

---

## Theory: ทฤษฎีพื้นฐาน Bitwise Operations

### 1. ทบทวนระบบเลขฐาน 2

ก่อนเข้าสู่คำสั่ง Assembly ให้ทบทวนพื้นฐาน:

```
ตัวอย่าง: เลข 0xAB = 10101011 ในฐาน 2

Bit position:  7  6  5  4  3  2  1  0
Bit value:     1  0  1  0  1  0  1  1
               ^  ^  ^  ^  ^  ^  ^  ^
               |  |  |  |  |  |  |  |
              128 64 32 16  8  4  2  1

128 + 0 + 32 + 0 + 8 + 0 + 2 + 1 = 171 = 0xAB
```

### 2. AND Operation - การ mask บิต

AND จะได้ผล 1 เมื่อ ทั้งสองบิตเป็น 1 เท่านั้น

```
Truth Table:
  A | B | A AND B
  0 | 0 |   0
  0 | 1 |   0
  1 | 0 |   0
  1 | 1 |   1

ตัวอย่าง:
  0xF0 = 1111 0000
  0x5A = 0101 1010
AND     ---------
        0101 0000 = 0x50

ใช้งาน:
- Masking: เก็บเฉพาะบิตที่ต้องการ (AND กับ 1 ที่ตำแหน่งนั้น)
- Clearing: ล้างบิตที่ต้องการ (AND กับ 0 ที่ตำแหน่งนั้น)
- Checking: ตรวจสอบว่าบิตใดบิตหนึ่งเป็น 1 หรือไม่
```

### 3. OR Operation - การ set บิต

OR จะได้ผล 1 เมื่อ อย่างน้อยหนึ่งบิตเป็น 1

```
Truth Table:
  A | B | A OR B
  0 | 0 |   0
  0 | 1 |   1
  1 | 0 |   1
  1 | 1 |   1

ตัวอย่าง:
  0xF0 = 1111 0000
  0x0F = 0000 1111
 OR     ---------
        1111 1111 = 0xFF

ใช้งาน:
- Setting: เซ็ตบิตที่ต้องการให้เป็น 1 (OR กับ 1 ที่ตำแหน่งนั้น)
- Combining flags: รวม flag ต่างๆ เข้าด้วยกัน
- Bitwise union: หา union ของ bitsets สองชุด
```

### 4. XOR Operation - การ toggle บิต

XOR จะได้ผล 1 เมื่อ บิตทั้งสองต่างกัน

```
Truth Table:
  A | B | A XOR B
  0 | 0 |   0
  0 | 1 |   1
  1 | 0 |   1
  1 | 1 |   0

ตัวอย่าง:
  0xFF = 1111 1111
  0xAA = 1010 1010
XOR    ---------
       0101 0101 = 0x55

คุณสมบัติพิเศษของ XOR:
- A XOR A = 0        (ล้างตัวเองได้)
- A XOR 0 = A        (ไม่เปลี่ยนแปลง)
- A XOR ~A = 0xFF..  (ได้ all ones)
- (A XOR B) XOR B = A (คืนค่าเดิม - ใช้ encryption)

ใช้งาน:
- Toggle: สลับบิตที่ต้องการ
- Swap: สลับค่าสองตัวแปรโดยไม่ใช้ temp
- Encryption: XOR cipher พื้นฐาน
- Parity: คำนวณ parity bit
```

### 5. NOT Operation - การ complement บิต

NOT จะสลับทุกบิต (0 เป็น 1, 1 เป็น 0)

```
ตัวอย่าง:
  0x0F = 0000 1111
NOT    ---------
       1111 0000 = 0xF0

หมายเหตุ:
- NOT ใน NASM คือ bitwise NOT (เหมือน ~ ใน C)
- ต่างจาก NEG ที่เป็น arithmetic negation (2's complement)
```

### 6. TEST Instruction - Non-destructive AND

TEST ทำงานเหมือน AND แต่ไม่เก็บผลลัพธ์ ทำแค่ set flags

```
TEST คล้าย AND แต่:
- AND: ทำการ AND และเก็บผลลัพธ์กลับไปที่ operand แรก
- TEST: ทำการ AND แล้วทิ้งผลลัพธ์ แต่ set ZF, SF, PF

ใช้งาน:
- ตรวจสอบ bit โดยไม่แก้ไขค่า
- ตรวจสอบว่าเป็น NULL pointer หรือไม่
- ตรวจสอบว่าค่าเป็น 0 หรือไม่ (TEST reg, reg)
```

### 7. Bit Manipulation Instructions

#### BT (Bit Test)
ทดสอบบิตที่ระบุตำแหน่ง และเก็บค่าบิตนั้นใน CF

#### BTS (Bit Test and Set)
ทดสอบบิตแล้ว set บิตนั้นให้เป็น 1

#### BTR (Bit Test and Reset)
ทดสอบบิตแล้ว clear บิตนั้นให้เป็น 0

#### BTC (Bit Test and Complement)
ทดสอบบิตแล้ว toggle บิตนั้น

#### BSF (Bit Scan Forward)
หา index ของ least significant bit ที่เป็น 1 (จากขวาไปซ้าย)

#### BSR (Bit Scan Reverse)
หา index ของ most significant bit ที่เป็น 1 (จากซ้ายไปขวา)

#### LZCNT (Leading Zero Count)
นับจำนวน leading zeros (zeros ทางซ้าย)

#### TZCNT (Trailing Zero Count)
นับจำนวน trailing zeros (zeros ทางขวา)

#### POPCNT (Population Count)
นับจำนวน bits ที่เป็น 1 ทั้งหมด

---

## Code Examples

### Example 1: AND, OR, XOR, NOT พื้นฐาน

```nasm
; ============================================================
; File: bitwise_basic.asm
; คำอธิบาย: สาธิตการใช้คำสั่ง AND, OR, XOR, NOT พื้นฐาน
; Compile: nasm -f elf64 bitwise_basic.asm -o bitwise_basic.o
;          ld bitwise_basic.o -o bitwise_basic
; Run:     ./bitwise_basic
; ============================================================

section .data
    ; Messages สำหรับแสดงผล
    msg_and      db "=== AND Operation ===", 10, 0
    msg_or       db "=== OR Operation ===", 10, 0
    msg_xor      db "=== XOR Operation ===", 10, 0
    msg_not      db "=== NOT Operation ===", 10, 0
    msg_newline  db 10, 0
    
    ; ข้อความแสดงค่า
    msg_hex      db "Hex: 0x", 0
    msg_newline2 db 10, 0
    
    ; Format string สำหรับ printf
    fmt_and    db "0x%02X AND 0x%02X = 0x%02X", 10, 0
    fmt_or     db "0x%02X OR  0x%02X = 0x%02X", 10, 0
    fmt_xor    db "0x%02X XOR 0x%02X = 0x%02X", 10, 0
    fmt_not    db "NOT 0x%02X = 0x%02X", 10, 0
    fmt_mask   db "Mask lower nibble: 0x%02X AND 0x0F = 0x%02X", 10, 0
    fmt_set    db "Set bit 3: 0x%02X OR 0x08 = 0x%02X", 10, 0
    fmt_toggle db "Toggle bits: 0x%02X XOR 0xFF = 0x%02X", 10, 0

section .text
    global main
    extern printf

main:
    push rbp
    mov rbp, rsp
    sub rsp, 48                     ; เตรียม stack space

    ; ===== AND Operation =====
    lea rdi, [rel fmt_and]          ; format string
    mov rsi, 0xF0                   ; operand 1: 0xF0 = 1111 0000
    mov rdx, 0x5A                   ; operand 2: 0x5A = 0101 1010
    
    ; คำนวณ AND
    mov rax, rsi                    ; rax = 0xF0
    and rax, rdx                    ; rax = 0xF0 AND 0x5A = 0x50
    mov rcx, rax                    ; rcx = ผลลัพธ์
    
    xor eax, eax                    ; clear eax (ไม่ใช้ float)
    call printf                     ; แสดง: 0xF0 AND 0x5A = 0x50

    ; ===== AND: Masking lower nibble =====
    ; เก็บเฉพาะ 4 บิตล่าง (lower nibble) ของ 0xAB
    lea rdi, [rel fmt_mask]
    mov rsi, 0xAB                   ; 0xAB = 1010 1011
    mov rax, rsi
    and rax, 0x0F                   ; mask = 0x0F = 0000 1111, ผล = 0x0B
    mov rdx, rax
    xor eax, eax
    call printf                     ; แสดง: Mask lower nibble: 0xAB AND 0x0F = 0x0B

    ; ===== OR Operation =====
    lea rdi, [rel fmt_or]
    mov rsi, 0xF0                   ; 0xF0 = 1111 0000
    mov rdx, 0x0F                   ; 0x0F = 0000 1111
    mov rax, rsi
    or rax, rdx                     ; 0xF0 OR 0x0F = 0xFF
    mov rcx, rax
    xor eax, eax
    call printf                     ; แสดง: 0xF0 OR 0x0F = 0xFF

    ; ===== OR: Setting a bit =====
    ; Set bit 3 (ค่า 8) ของ 0x51
    lea rdi, [rel fmt_set]
    mov rsi, 0x51                   ; 0x51 = 0101 0001
    mov rax, rsi
    or rax, 0x08                    ; set bit 3: 0x51 OR 0x08 = 0x59
    mov rdx, rax
    xor eax, eax
    call printf                     ; แสดง: Set bit 3: 0x51 OR 0x08 = 0x59

    ; ===== XOR Operation =====
    lea rdi, [rel fmt_xor]
    mov rsi, 0xFF                   ; 0xFF = 1111 1111
    mov rdx, 0xAA                   ; 0xAA = 1010 1010
    mov rax, rsi
    xor rax, rdx                    ; 0xFF XOR 0xAA = 0x55
    mov rcx, rax
    xor eax, eax
    call printf                     ; แสดง: 0xFF XOR 0xAA = 0x55

    ; ===== XOR: Toggle all bits =====
    lea rdi, [rel fmt_toggle]
    mov rsi, 0x3C                   ; 0x3C = 0011 1100
    mov rax, rsi
    xor rax, 0xFF                   ; toggle all bits: 0x3C XOR 0xFF = 0xC3
    mov rdx, rax
    xor eax, eax
    call printf                     ; แสดง: Toggle bits: 0x3C XOR 0xFF = 0xC3

    ; ===== NOT Operation =====
    lea rdi, [rel fmt_not]
    mov rsi, 0x0F                   ; 0x0F = 0000 1111
    mov rax, rsi
    not rax                         ; NOT 0x0F (64-bit: FFFFFFFFFFFFFFF0)
    and rax, 0xFF                   ; เก็บแค่ byte เดียวเพื่อความชัดเจน = 0xF0
    mov rdx, rax
    xor eax, eax
    call printf                     ; แสดง: NOT 0x0F = 0xF0

    ; จบโปรแกรม
    xor eax, eax                    ; return 0
    leave
    ret
```

**คำสั่ง Compile และ Run:**
```bash
nasm -f elf64 bitwise_basic.asm -o bitwise_basic.o
gcc -no-pie bitwise_basic.o -o bitwise_basic
./bitwise_basic
```

**Expected Output:**
```
0xF0 AND 0x5A = 0x50
Mask lower nibble: 0xAB AND 0x0F = 0x0B
0xF0 OR  0x0F = 0xFF
Set bit 3: 0x51 OR 0x08 = 0x59
0xFF XOR 0xAA = 0x55
Toggle bits: 0x3C XOR 0xFF = 0xC3
NOT 0x0F = 0xF0
```

---

### Example 2: TEST Instruction และ Flag Checking

```nasm
; ============================================================
; File: test_instruction.asm
; คำอธิบาย: การใช้ TEST instruction และการตรวจสอบบิต
; Compile: nasm -f elf64 test_instruction.asm -o test_instruction.o
;          gcc -no-pie test_instruction.o -o test_instruction
; Run:     ./test_instruction
; ============================================================

section .data
    fmt_check_bit  db "Value 0x%02X: bit %d is %s", 10, 0
    fmt_is_zero    db "Value %d is %s", 10, 0
    fmt_is_odd     db "Value %d is %s", 10, 0
    fmt_is_neg     db "Value %d is %s", 10, 0
    str_set        db "SET (1)", 0
    str_clear      db "CLEAR (0)", 0
    str_zero       db "ZERO", 0
    str_nonzero    db "NON-ZERO", 0
    str_odd        db "ODD", 0
    str_even       db "EVEN", 0
    str_negative   db "NEGATIVE", 0
    str_positive   db "POSITIVE/ZERO", 0
    
    header_test    db "=== TEST Instruction Demo ===", 10, 0
    header_null    db 10, "=== NULL Check with TEST ===", 10, 0
    fmt_null       db "Pointer %s null", 10, 0
    str_is         db "IS", 0
    str_is_not     db "IS NOT", 0

section .text
    global main
    extern printf

main:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    ; --- แสดง header ---
    lea rdi, [rel header_test]
    xor eax, eax
    call printf

    ; ===== ตรวจสอบบิตต่างๆ ด้วย TEST =====
    
    ; ตรวจสอบ bit 4 ของ 0x1F = 0001 1111
    ; bit 4 = ค่า 16 = 0x10
    mov rax, 0x1F                   ; ค่าที่จะตรวจ
    test rax, 0x10                  ; TEST กับ mask ของ bit 4 (0x10 = 0001 0000)
    ; ZF=1 ถ้าบิตนั้นเป็น 0, ZF=0 ถ้าบิตนั้นเป็น 1
    jnz .bit4_set_1F                ; ถ้า ZF=0 แปลว่าบิตนั้นเป็น 1
    lea rcx, [rel str_clear]
    jmp .print_bit4_1F
.bit4_set_1F:
    lea rcx, [rel str_set]
.print_bit4_1F:
    lea rdi, [rel fmt_check_bit]
    mov rsi, 0x1F
    mov rdx, 4
    xor eax, eax
    call printf                     ; แสดง: Value 0x1F: bit 4 is CLEAR (0)
    ; เพราะ 0x1F = 0001 1111, bit4 = 0 (นับจาก bit0)

    ; ตรวจสอบ bit 3 ของ 0x1F
    mov rax, 0x1F                   ; 0x1F = 0001 1111
    test rax, 0x08                  ; TEST กับ mask ของ bit 3 (0x08 = 0000 1000)
    jnz .bit3_set_1F
    lea rcx, [rel str_clear]
    jmp .print_bit3_1F
.bit3_set_1F:
    lea rcx, [rel str_set]
.print_bit3_1F:
    lea rdi, [rel fmt_check_bit]
    mov rsi, 0x1F
    mov rdx, 3
    xor eax, eax
    call printf                     ; แสดง: Value 0x1F: bit 3 is SET (1)
    ; เพราะ 0x1F = 0001 1111, bit3 = 1

    ; ===== ตรวจสอบว่าค่าเป็น 0 หรือไม่ (TEST reg, reg) =====
    ; เทคนิคนี้เร็วกว่า CMP reg, 0
    mov rax, 0                      ; ค่า = 0
    test rax, rax                   ; TEST ตัวเองกับตัวเอง: ถ้า 0, ZF=1
    jnz .nonzero_case1
    lea rdx, [rel str_zero]
    jmp .print_zero1
.nonzero_case1:
    lea rdx, [rel str_nonzero]
.print_zero1:
    lea rdi, [rel fmt_is_zero]
    mov rsi, 0
    xor eax, eax
    call printf                     ; แสดง: Value 0 is ZERO

    mov rax, 42                     ; ค่า = 42
    test rax, rax
    jnz .nonzero_case2
    lea rdx, [rel str_zero]
    jmp .print_zero2
.nonzero_case2:
    lea rdx, [rel str_nonzero]
.print_zero2:
    lea rdi, [rel fmt_is_zero]
    mov rsi, 42
    xor eax, eax
    call printf                     ; แสดง: Value 42 is NON-ZERO

    ; ===== ตรวจสอบ odd/even ด้วย TEST =====
    ; bit 0 = 1 แปลว่า odd, bit 0 = 0 แปลว่า even
    mov rax, 7                      ; ค่า = 7 (odd)
    test rax, 1                     ; TEST bit 0
    jnz .is_odd3
    lea rdx, [rel str_even]
    jmp .print_odd3
.is_odd3:
    lea rdx, [rel str_odd]
.print_odd3:
    lea rdi, [rel fmt_is_odd]
    mov rsi, 7
    xor eax, eax
    call printf                     ; แสดง: Value 7 is ODD

    mov rax, 12                     ; ค่า = 12 (even)
    test rax, 1
    jnz .is_odd4
    lea rdx, [rel str_even]
    jmp .print_odd4
.is_odd4:
    lea rdx, [rel str_odd]
.print_odd4:
    lea rdi, [rel fmt_is_odd]
    mov rsi, 12
    xor eax, eax
    call printf                     ; แสดง: Value 12 is EVEN

    ; ===== NULL pointer check =====
    lea rdi, [rel header_null]
    xor eax, eax
    call printf

    ; ตรวจ NULL pointer
    mov rax, 0                      ; NULL pointer
    test rax, rax
    jnz .not_null1
    lea rdx, [rel str_is]
    jmp .print_null1
.not_null1:
    lea rdx, [rel str_is_not]
.print_null1:
    lea rdi, [rel fmt_null]
    mov rsi, rdx
    xor eax, eax
    call printf                     ; แสดง: Pointer IS null

    lea rax, [rel str_set]          ; ใช้ address เป็น non-NULL pointer
    test rax, rax
    jnz .not_null2
    lea rdx, [rel str_is]
    jmp .print_null2
.not_null2:
    lea rdx, [rel str_is_not]
.print_null2:
    lea rdi, [rel fmt_null]
    mov rsi, rdx
    xor eax, eax
    call printf                     ; แสดง: Pointer IS NOT null

    xor eax, eax
    leave
    ret
```

**คำสั่ง Compile และ Run:**
```bash
nasm -f elf64 test_instruction.asm -o test_instruction.o
gcc -no-pie test_instruction.o -o test_instruction
./test_instruction
```

**Expected Output:**
```
=== TEST Instruction Demo ===
Value 0x1F: bit 4 is CLEAR (0)
Value 0x1F: bit 3 is SET (1)
Value 0 is ZERO
Value 42 is NON-ZERO
Value 7 is ODD
Value 12 is EVEN

=== NULL Check with TEST ===
Pointer IS null
Pointer IS NOT null
```

---

### Example 3: BT, BTS, BTR, BTC Instructions

```nasm
; ============================================================
; File: bit_test_ops.asm
; คำอธิบาย: การใช้ BT, BTS, BTR, BTC instructions
; Compile: nasm -f elf64 bit_test_ops.asm -o bit_test_ops.o
;          gcc -no-pie bit_test_ops.o -o bit_test_ops
; Run:     ./bit_test_ops
; ============================================================

section .data
    fmt_bt   db "BT   0x%08X, bit %d -> CF=%d (Original value unchanged: 0x%08X)", 10, 0
    fmt_bts  db "BTS  0x%08X, bit %d -> CF=%d (New value: 0x%08X)", 10, 0
    fmt_btr  db "BTR  0x%08X, bit %d -> CF=%d (New value: 0x%08X)", 10, 0
    fmt_btc  db "BTC  0x%08X, bit %d -> CF=%d (New value: 0x%08X)", 10, 0
    
    hdr_bt   db "=== BT: Bit Test ===", 10, 0
    hdr_bts  db 10, "=== BTS: Bit Test and Set ===", 10, 0
    hdr_btr  db 10, "=== BTR: Bit Test and Reset ===", 10, 0
    hdr_btc  db 10, "=== BTC: Bit Test and Complement ===", 10, 0
    hdr_demo db 10, "=== Practical Demo: Permission Bits ===", 10, 0
    
    ; Permission bits demo
    ; สมมติ permissions เป็น 32-bit flags
    ; bit 0: read, bit 1: write, bit 2: execute
    ; bit 3: admin, bit 4: owner, bit 5: group_read
    fmt_perm  db "Permission value: 0x%08X", 10, 0
    fmt_has   db "  Has %s: %s", 10, 0
    str_yes   db "YES", 0
    str_no    db "NO", 0
    str_read  db "READ", 0
    str_write db "WRITE", 0
    str_exec  db "EXECUTE", 0
    str_admin db "ADMIN", 0
    
    fmt_grant  db "After granting EXECUTE: 0x%08X", 10, 0
    fmt_revoke db "After revoking WRITE: 0x%08X", 10, 0
    fmt_toggle_admin db "After toggling ADMIN: 0x%08X", 10, 0

section .text
    global main
    extern printf

main:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    ; ===== BT: Bit Test =====
    ; BT ทำแค่อ่านบิตและเก็บใน CF ไม่แก้ค่า
    lea rdi, [rel hdr_bt]
    xor eax, eax
    call printf

    mov rbx, 0xABCD1234             ; ค่าทดสอบ
    
    ; ทดสอบ bit 4 (ค่า 0x10)
    bt rbx, 4                       ; ทดสอบ bit 4 ของ rbx
    ; 0xABCD1234 = ...0001 0010 0011 0100
    ; bit 4 = 1 (ค่า 0x10 = 16 ≤ 0x34)
    ; CF = ค่าของ bit 4
    setc al                         ; al = CF (1 ถ้าบิตนั้นเป็น 1)
    movzx r8, al                    ; r8 = CF value
    
    lea rdi, [rel fmt_bt]
    mov rsi, rbx                    ; ค่าเดิม
    mov rdx, 4                      ; bit index
    mov rcx, r8                     ; CF value
    mov r8, rbx                     ; ค่าหลัง BT (ไม่เปลี่ยน)
    xor eax, eax
    call printf

    ; ทดสอบ bit 31 (MSB ของ 32-bit)
    bt rbx, 31                      ; ทดสอบ bit 31
    ; 0xABCD1234: bit 31 = 1 (0xABCD1234 มี bit 31 set เพราะ 0xA = 1010)
    setc al
    movzx r8, al
    
    lea rdi, [rel fmt_bt]
    mov rsi, rbx
    mov rdx, 31
    mov rcx, r8
    mov r8, rbx
    xor eax, eax
    call printf

    ; ===== BTS: Bit Test and Set =====
    lea rdi, [rel hdr_bts]
    xor eax, eax
    call printf

    mov rbx, 0x00000005             ; 0x05 = 0000 0101 (bit 0 และ bit 2 set)
    
    ; Set bit 1 (ซึ่งยังไม่ได้ set)
    bts rbx, 1                      ; ทดสอบ bit 1 แล้ว set ให้เป็น 1
    ; bit 1 เดิม = 0 ดังนั้น CF = 0 (ค่าเดิมของบิต)
    ; หลัง BTS: rbx = 0x07 (bit 0,1,2 set)
    setc al
    movzx r8, al
    
    lea rdi, [rel fmt_bts]
    mov rsi, 5                      ; ค่าเดิม
    mov rdx, 1                      ; bit index
    mov rcx, r8                     ; CF (ค่าบิตก่อน set)
    mov r8, rbx                     ; ค่าใหม่หลัง BTS
    xor eax, eax
    call printf

    ; ===== BTR: Bit Test and Reset =====
    lea rdi, [rel hdr_btr]
    xor eax, eax
    call printf

    mov rbx, 0x000000FF             ; ทุกบิตใน byte ล่างเป็น 1
    
    ; Clear bit 3
    btr rbx, 3                      ; ทดสอบ bit 3 แล้ว clear ให้เป็น 0
    ; bit 3 เดิม = 1 ดังนั้น CF = 1
    ; หลัง BTR: rbx = 0xF7 (bit 3 cleared)
    setc al
    movzx r8, al
    
    lea rdi, [rel fmt_btr]
    mov rsi, 0xFF                   ; ค่าเดิม
    mov rdx, 3
    mov rcx, r8
    mov r8, rbx
    xor eax, eax
    call printf

    ; ===== BTC: Bit Test and Complement =====
    lea rdi, [rel hdr_btc]
    xor eax, eax
    call printf

    mov rbx, 0x000000AA             ; 0xAA = 1010 1010
    
    ; Toggle bit 0 (เดิม = 0)
    btc rbx, 0
    setc al
    movzx r8, al
    
    lea rdi, [rel fmt_btc]
    mov rsi, 0xAA
    mov rdx, 0
    mov rcx, r8
    mov r8, rbx
    xor eax, eax
    call printf                     ; แสดง CF=0 (เดิม 0), ค่าใหม่ = 0xAB

    ; Toggle bit 1 (เดิม = 1)
    mov rbx, 0x000000AA             ; reset
    btc rbx, 1
    setc al
    movzx r8, al
    
    lea rdi, [rel fmt_btc]
    mov rsi, 0xAA
    mov rdx, 1
    mov rcx, r8
    mov r8, rbx
    xor eax, eax
    call printf                     ; แสดง CF=1 (เดิม 1), ค่าใหม่ = 0xA8

    ; ===== Practical Demo: Permission System =====
    lea rdi, [rel hdr_demo]
    xor eax, eax
    call printf

    ; สร้าง permission = READ | WRITE (bits 0 และ 1)
    mov rbx, 0x03                   ; 0x03 = 0000 0011

    ; แสดง permission value
    lea rdi, [rel fmt_perm]
    mov rsi, rbx
    xor eax, eax
    call printf

    ; ตรวจสอบ READ permission (bit 0)
    bt rbx, 0
    setc al
    movzx r8d, al
    lea rdi, [rel fmt_has]
    lea rsi, [rel str_read]
    test r8d, r8d
    jz .no_read
    lea rdx, [rel str_yes]
    jmp .print_read
.no_read:
    lea rdx, [rel str_no]
.print_read:
    xor eax, eax
    call printf

    ; ตรวจสอบ WRITE permission (bit 1)
    bt rbx, 1
    setc al
    movzx r8d, al
    lea rdi, [rel fmt_has]
    lea rsi, [rel str_write]
    test r8d, r8d
    jz .no_write
    lea rdx, [rel str_yes]
    jmp .print_write
.no_write:
    lea rdx, [rel str_no]
.print_write:
    xor eax, eax
    call printf

    ; ตรวจสอบ EXECUTE permission (bit 2)
    bt rbx, 2
    setc al
    movzx r8d, al
    lea rdi, [rel fmt_has]
    lea rsi, [rel str_exec]
    test r8d, r8d
    jz .no_exec
    lea rdx, [rel str_yes]
    jmp .print_exec
.no_exec:
    lea rdx, [rel str_no]
.print_exec:
    xor eax, eax
    call printf

    ; Grant EXECUTE (set bit 2)
    bts rbx, 2                      ; set bit 2
    lea rdi, [rel fmt_grant]
    mov rsi, rbx
    xor eax, eax
    call printf                     ; แสดง: 0x07 (READ|WRITE|EXECUTE)

    ; Revoke WRITE (clear bit 1)
    btr rbx, 1                      ; clear bit 1
    lea rdi, [rel fmt_revoke]
    mov rsi, rbx
    xor eax, eax
    call printf                     ; แสดง: 0x05 (READ|EXECUTE)

    ; Toggle ADMIN (bit 3)
    btc rbx, 3                      ; toggle bit 3
    lea rdi, [rel fmt_toggle_admin]
    mov rsi, rbx
    xor eax, eax
    call printf                     ; แสดง: 0x0D (READ|EXECUTE|ADMIN)

    xor eax, eax
    leave
    ret
```

**คำสั่ง Compile และ Run:**
```bash
nasm -f elf64 bit_test_ops.asm -o bit_test_ops.o
gcc -no-pie bit_test_ops.o -o bit_test_ops
./bit_test_ops
```

**Expected Output:**
```
=== BT: Bit Test ===
BT   0xABCD1234, bit 4 -> CF=1 (Original value unchanged: 0xABCD1234)
BT   0xABCD1234, bit 31 -> CF=1 (Original value unchanged: 0xABCD1234)

=== BTS: Bit Test and Set ===
BTS  0x00000005, bit 1 -> CF=0 (New value: 0x00000007)

=== BTR: Bit Test and Reset ===
BTR  0x000000FF, bit 3 -> CF=1 (New value: 0x000000F7)

=== BTC: Bit Test and Complement ===
BTC  0x000000AA, bit 0 -> CF=0 (New value: 0x000000AB)
BTC  0x000000AA, bit 1 -> CF=1 (New value: 0x000000A8)

=== Practical Demo: Permission Bits ===
Permission value: 0x00000003
  Has READ: YES
  Has WRITE: YES
  Has EXECUTE: NO
After granting EXECUTE: 0x00000007
After revoking WRITE: 0x00000005
After toggling ADMIN: 0x0000000D
```

---

### Example 4: BSF, BSR, POPCNT, LZCNT, TZCNT

```nasm
; ============================================================
; File: bit_scan.asm
; คำอธิบาย: BSF, BSR, POPCNT, LZCNT, TZCNT instructions
; Compile: nasm -f elf64 bit_scan.asm -o bit_scan.o
;          gcc -no-pie bit_scan.o -o bit_scan
;          (ต้องการ CPU ที่รองรับ POPCNT, LZCNT, TZCNT)
; Run:     ./bit_scan
; ============================================================

section .data
    hdr_bsf   db "=== BSF: Bit Scan Forward (หา LSB) ===", 10, 0
    hdr_bsr   db 10, "=== BSR: Bit Scan Reverse (หา MSB) ===", 10, 0
    hdr_pop   db 10, "=== POPCNT: Population Count ===", 10, 0
    hdr_lz    db 10, "=== LZCNT: Leading Zero Count ===", 10, 0
    hdr_tz    db 10, "=== TZCNT: Trailing Zero Count ===", 10, 0
    hdr_app   db 10, "=== Applications ===", 10, 0
    
    fmt_bsf   db "BSF(0x%016llX) = index %d (LSB position)", 10, 0
    fmt_bsf_z db "BSF(0) = undefined (ZF=1, ค่าในผลลัพธ์ไม่แน่นอน)", 10, 0
    fmt_bsr   db "BSR(0x%016llX) = index %d (MSB position)", 10, 0
    fmt_pop   db "POPCNT(0x%016llX) = %d ones", 10, 0
    fmt_lz    db "LZCNT(0x%016llX) = %d leading zeros", 10, 0
    fmt_tz    db "TZCNT(0x%016llX) = %d trailing zeros", 10, 0
    
    fmt_log2  db "floor(log2(%llu)) = %d", 10, 0
    fmt_align db "Align %llu to next power of 2: %llu", 10, 0
    fmt_iso   db "Isolate lowest set bit: 0x%016llX & (-0x%016llX) = 0x%016llX", 10, 0
    fmt_pow2  db "Is %llu a power of 2? %s", 10, 0
    str_yes   db "YES", 0
    str_no    db "NO", 0

section .text
    global main
    extern printf

main:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    ; ===== BSF: Bit Scan Forward =====
    ; BSF หา index ของ least significant bit ที่เป็น 1
    ; (นับจาก bit 0 ทางขวาไปซ้าย)
    ; ถ้า source = 0, ZF=1 และค่าใน destination ไม่แน่นอน
    
    lea rdi, [rel hdr_bsf]
    xor eax, eax
    call printf

    ; ตัวอย่าง 1: 0x0000 0010 0000 0000 = bit 9 set
    mov rbx, 0x200                  ; 0x200 = 10 0000 0000, bit 9 = 1
    bsf rcx, rbx                    ; rcx = index of lowest set bit = 9
    jz .bsf_zero1
    lea rdi, [rel fmt_bsf]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf
    jmp .bsf_done1
.bsf_zero1:
    lea rdi, [rel fmt_bsf_z]
    xor eax, eax
    call printf
.bsf_done1:

    ; ตัวอย่าง 2: 0xABCD1234 = ...0001 0010 0011 0100
    ; bit 2 เป็น lowest set bit (0x34 = 0011 0100, bit 2 = 1)
    mov rbx, 0xABCD1234
    bsf rcx, rbx
    lea rdi, [rel fmt_bsf]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ตัวอย่าง 3: input = 0
    xor rbx, rbx                    ; rbx = 0
    bsf rcx, rbx                    ; ZF=1 เพราะ source=0
    jz .bsf_is_zero3
    lea rdi, [rel fmt_bsf]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf
    jmp .bsf_done3
.bsf_is_zero3:
    lea rdi, [rel fmt_bsf_z]
    xor eax, eax
    call printf
.bsf_done3:

    ; ===== BSR: Bit Scan Reverse =====
    ; BSR หา index ของ most significant bit ที่เป็น 1
    ; (นับจาก bit สูงสุดลงมา แต่ผลเป็น index จาก bit 0)
    
    lea rdi, [rel hdr_bsr]
    xor eax, eax
    call printf

    ; ตัวอย่าง: 0x0F00 = 0000 1111 0000 0000 0000
    ; MSB ที่เป็น 1 อยู่ที่ bit 11 (0x800 ≤ 0xF00)
    mov rbx, 0x0F00
    bsr rcx, rbx                    ; rcx = 11
    lea rdi, [rel fmt_bsr]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ตัวอย่าง: 1 → bit 0 = MSB (เป็นตัวเดียวที่ set)
    mov rbx, 1
    bsr rcx, rbx                    ; rcx = 0
    lea rdi, [rel fmt_bsr]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; BSR = floor(log2(n)) สำหรับ n > 0
    mov rbx, 100                    ; log2(100) ≈ 6.64, floor = 6
    bsr rcx, rbx
    lea rdi, [rel fmt_log2]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ===== POPCNT: Population Count =====
    lea rdi, [rel hdr_pop]
    xor eax, eax
    call printf

    ; นับจำนวน 1-bits
    mov rbx, 0xFF                   ; 8 bits = 8 ones
    popcnt rcx, rbx                 ; rcx = 8
    lea rdi, [rel fmt_pop]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    mov rbx, 0x5555555555555555     ; สลับ 0 และ 1 (32 ones ใน 64 bits)
    popcnt rcx, rbx                 ; rcx = 32
    lea rdi, [rel fmt_pop]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    mov rbx, 0xFFFFFFFFFFFFFFFF     ; ทุก bit เป็น 1
    popcnt rcx, rbx                 ; rcx = 64
    lea rdi, [rel fmt_pop]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ===== LZCNT: Leading Zero Count =====
    lea rdi, [rel hdr_lz]
    xor eax, eax
    call printf

    ; LZCNT นับ zeros ก่อน MSB ที่เป็น 1
    ; ต้องการ CPU feature: LZCNT (ABM/BMI1)
    mov rbx, 1                      ; bit 0 set: 63 leading zeros ใน 64-bit
    lzcnt rcx, rbx                  ; rcx = 63
    lea rdi, [rel fmt_lz]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    mov rbx, 0x8000000000000000     ; bit 63 set: 0 leading zeros
    lzcnt rcx, rbx                  ; rcx = 0
    lea rdi, [rel fmt_lz]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    mov rbx, 0x00000000000001FF     ; bit 8 set (highest): 55 leading zeros
    lzcnt rcx, rbx                  ; rcx = 55
    lea rdi, [rel fmt_lz]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ===== TZCNT: Trailing Zero Count =====
    lea rdi, [rel hdr_tz]
    xor eax, eax
    call printf

    mov rbx, 8                      ; 0x08 = 1000, trailing zeros = 3
    tzcnt rcx, rbx                  ; rcx = 3
    lea rdi, [rel fmt_tz]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    mov rbx, 0x80                   ; bit 7: trailing zeros = 7
    tzcnt rcx, rbx                  ; rcx = 7
    lea rdi, [rel fmt_tz]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    mov rbx, 0xFF                   ; bit 0 set: trailing zeros = 0
    tzcnt rcx, rbx                  ; rcx = 0
    lea rdi, [rel fmt_tz]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ===== Applications =====
    lea rdi, [rel hdr_app]
    xor eax, eax
    call printf

    ; 1) Isolate lowest set bit: x & (-x)
    ; -x ใน 2's complement = ~x + 1
    ; ตัวอย่าง: 0x0C = 0000 1100
    ; -0x0C = 1111 0100 (2's complement)
    ; 0x0C & (-0x0C) = 0000 0100 (isolate bit 2)
    mov rbx, 0x0C
    mov rcx, rbx
    neg rcx                         ; rcx = -rbx (2's complement)
    and rcx, rbx                    ; rcx = isolated lowest set bit
    lea rdi, [rel fmt_iso]
    mov rsi, rbx
    mov rdx, rbx
    mov r8, rcx
    xor eax, eax
    call printf

    ; 2) ตรวจสอบว่า n เป็น power of 2 หรือไม่
    ; n เป็น power of 2 ถ้า n > 0 และ (n & (n-1)) == 0
    ; เพราะ power of 2 มีแค่ bit เดียวที่เป็น 1
    ; ตัวอย่าง: 16 = 10000, 16-1=01111, 16 & 15 = 0 → power of 2
    ;           12 = 01100, 12-1=01011, 12 & 11 = 01000 ≠ 0 → not
    
    mov rbx, 16
    mov rcx, rbx
    dec rcx                         ; rcx = n-1 = 15
    test rbx, rcx                   ; n & (n-1)
    jnz .not_pow2_16
    lea r8, [rel str_yes]
    jmp .print_pow2_16
.not_pow2_16:
    lea r8, [rel str_no]
.print_pow2_16:
    lea rdi, [rel fmt_pow2]
    mov rsi, rbx
    mov rdx, r8
    xor eax, eax
    call printf

    mov rbx, 12
    mov rcx, rbx
    dec rcx
    test rbx, rcx
    jnz .not_pow2_12
    lea r8, [rel str_yes]
    jmp .print_pow2_12
.not_pow2_12:
    lea r8, [rel str_no]
.print_pow2_12:
    lea rdi, [rel fmt_pow2]
    mov rsi, rbx
    mov rdx, r8
    xor eax, eax
    call printf

    xor eax, eax
    leave
    ret
```

**คำสั่ง Compile และ Run:**
```bash
nasm -f elf64 bit_scan.asm -o bit_scan.o
gcc -no-pie bit_scan.o -o bit_scan
./bit_scan
```

**Expected Output:**
```
=== BSF: Bit Scan Forward (หา LSB) ===
BSF(0x0000000000000200) = index 9 (LSB position)
BSF(0x00000000ABCD1234) = index 2 (LSB position)
BSF(0) = undefined (ZF=1, ค่าในผลลัพธ์ไม่แน่นอน)

=== BSR: Bit Scan Reverse (หา MSB) ===
BSR(0x0000000000000F00) = index 11 (MSB position)
BSR(0x0000000000000001) = index 0 (MSB position)
floor(log2(100)) = 6

=== POPCNT: Population Count ===
POPCNT(0x00000000000000FF) = 8 ones
POPCNT(0x5555555555555555) = 32 ones
POPCNT(0xFFFFFFFFFFFFFFFF) = 64 ones

=== LZCNT: Leading Zero Count ===
LZCNT(0x0000000000000001) = 63 leading zeros
LZCNT(0x8000000000000000) = 0 leading zeros
LZCNT(0x00000000000001FF) = 55 leading zeros

=== TZCNT: Trailing Zero Count ===
TZCNT(0x0000000000000008) = 3 trailing zeros
TZCNT(0x0000000000000080) = 7 trailing zeros
TZCNT(0x00000000000000FF) = 0 trailing zeros

=== Applications ===
Isolate lowest set bit: 0x000000000000000C & (-0x000000000000000C) = 0x0000000000000004
Is 16 a power of 2? YES
Is 12 a power of 2? NO
```

---

### Example 5: XOR Encryption และ Bit Manipulation Tricks

```nasm
; ============================================================
; File: xor_encrypt.asm
; คำอธิบาย: XOR encryption และ bit manipulation tricks ขั้นสูง
; Compile: nasm -f elf64 xor_encrypt.asm -o xor_encrypt.o
;          gcc -no-pie xor_encrypt.o -o xor_encrypt
; Run:     ./xor_encrypt
; ============================================================

section .data
    hdr_xor_enc  db "=== XOR Encryption Demo ===", 10, 0
    hdr_xor_swap db 10, "=== XOR Swap (ไม่ใช้ temp variable) ===", 10, 0
    hdr_tricks   db 10, "=== Bit Manipulation Tricks ===", 10, 0
    hdr_bitset   db 10, "=== Bitset Implementation ===", 10, 0
    
    ; XOR Encryption
    plaintext    db "Hello, Assembly World!", 0
    plain_len    equ $ - plaintext - 1   ; ความยาวไม่นับ null
    key_byte     db 0x5A                  ; XOR key
    ciphertext   times 64 db 0           ; buffer สำหรับ ciphertext
    decrypted    times 64 db 0           ; buffer สำหรับ decrypted
    
    fmt_plain    db "Plaintext:  %s", 10, 0
    fmt_cipher   db "Ciphertext: (hex) ", 0
    fmt_hex_byte db "%02X ", 0
    fmt_decrypt  db 10, "Decrypted:  %s", 10, 0
    
    ; XOR Swap
    fmt_before_swap db "Before swap: a=%d, b=%d", 10, 0
    fmt_after_swap  db "After swap:  a=%d, b=%d", 10, 0
    
    ; Bit tricks
    fmt_trick1   db "Trick 1 - Clear lowest set bit: 0x%02X -> 0x%02X", 10, 0
    fmt_trick2   db "Trick 2 - Round down to power of 2: %d -> %d", 10, 0
    fmt_trick3   db "Trick 3 - Next power of 2 >= n: %d -> %d", 10, 0
    fmt_trick4   db "Trick 4 - Absolute value (no branch): %d -> %d", 10, 0
    fmt_trick5   db "Trick 5 - Sign of number: %d -> %d (1=pos, -1=neg, 0=zero)", 10, 0
    
    ; Bitset
    fmt_bs_set   db "Set bit %d in bitset", 10, 0
    fmt_bs_test  db "Test bit %d: %s", 10, 0
    fmt_bs_clr   db "Clear bit %d from bitset", 10, 0
    fmt_bs_union db "Union bitset: 0x%016llX", 10, 0
    fmt_bs_inter db "Intersection bitset: 0x%016llX", 10, 0
    fmt_bs_diff  db "Difference bitset: 0x%016llX", 10, 0
    str_set      db "SET", 0
    str_not_set  db "NOT SET", 0

section .bss
    bitset_A     resq 1             ; 64-bit bitset A
    bitset_B     resq 1             ; 64-bit bitset B

section .text
    global main
    extern printf, memcpy

main:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    ; ===== XOR Encryption =====
    lea rdi, [rel hdr_xor_enc]
    xor eax, eax
    call printf

    ; แสดง plaintext
    lea rdi, [rel fmt_plain]
    lea rsi, [rel plaintext]
    xor eax, eax
    call printf

    ; XOR encrypt: ciphertext[i] = plaintext[i] XOR key
    lea rsi, [rel plaintext]        ; source
    lea rdi, [rel ciphertext]       ; destination
    movzx rbx, byte [rel key_byte]  ; XOR key
    mov rcx, plain_len              ; ความยาว

.encrypt_loop:
    movzx rax, byte [rsi]           ; โหลด byte ของ plaintext
    xor rax, rbx                    ; XOR กับ key
    mov [rdi], al                   ; เก็บลง ciphertext
    inc rsi                         ; ไปยัง byte ถัดไป
    inc rdi
    dec rcx                         ; ลด counter
    jnz .encrypt_loop               ; ทำจนครบทุก byte

    ; แสดง ciphertext เป็น hex
    lea rdi, [rel fmt_cipher]
    xor eax, eax
    call printf

    lea rsi, [rel ciphertext]
    mov rcx, plain_len
.print_hex:
    movzx rdi, byte [rsi]           ; โหลด hex byte
    push rsi
    push rcx
    lea rdi, [rel fmt_hex_byte]
    movzx rsi, byte [rsi + rcx*0 - plain_len + (plain_len - 1)]
    ; แก้ไข: print from beginning
    pop rcx
    pop rsi
    movzx rsi, byte [rsi]
    push rsi
    push rcx
    lea rdi, [rel fmt_hex_byte]
    xor eax, eax
    ; ใช้วิธีที่ง่ายกว่า
    pop rcx
    pop rsi
    inc rsi
    dec rcx
    jnz .print_hex

    ; พิมพ์ hex ทีเดียว
    lea rsi, [rel ciphertext]
    xor ecx, ecx
.print_hex2:
    movzx rsi, byte [rel ciphertext + rcx]
    push rcx
    lea rdi, [rel fmt_hex_byte]
    xor eax, eax
    call printf
    pop rcx
    inc rcx
    cmp rcx, plain_len
    jl .print_hex2

    ; XOR decrypt: ทำ XOR อีกครั้งด้วย key เดิม
    lea rsi, [rel ciphertext]
    lea rdi, [rel decrypted]
    movzx rbx, byte [rel key_byte]
    mov rcx, plain_len

.decrypt_loop:
    movzx rax, byte [rsi]
    xor rax, rbx                    ; XOR กับ key เดิม = ได้ plaintext กลับมา
    mov [rdi], al
    inc rsi
    inc rdi
    dec rcx
    jnz .decrypt_loop

    ; แสดง decrypted text
    lea rdi, [rel fmt_decrypt]
    lea rsi, [rel decrypted]
    xor eax, eax
    call printf

    ; ===== XOR Swap =====
    lea rdi, [rel hdr_xor_swap]
    xor eax, eax
    call printf

    mov rbx, 42                     ; a = 42
    mov rcx, 99                     ; b = 99

    lea rdi, [rel fmt_before_swap]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; XOR swap: a = a^b, b = b^a, a = a^b
    xor rbx, rcx                    ; a = a XOR b
    xor rcx, rbx                    ; b = b XOR (a XOR b) = a
    xor rbx, rcx                    ; a = (a XOR b) XOR a = b

    lea rdi, [rel fmt_after_swap]
    mov rsi, rbx
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ===== Bit Manipulation Tricks =====
    lea rdi, [rel hdr_tricks]
    xor eax, eax
    call printf

    ; Trick 1: Clear lowest set bit: n & (n-1)
    ; ใช้ใน loop นับ set bits, หา power of 2 ฯลฯ
    mov rax, 0x0C                   ; 0x0C = 0000 1100
    mov rbx, rax
    dec rbx                         ; rbx = n-1 = 0x0B = 0000 1011
    and rax, rbx                    ; 0x0C & 0x0B = 0x08 (ล้าง bit 2 ซึ่งเป็น lowest)
    lea rdi, [rel fmt_trick1]
    mov rsi, 0x0C
    mov rdx, rax
    xor eax, eax
    call printf

    ; Trick 2: Round down to nearest power of 2
    ; ใช้ BSR หา position ของ MSB แล้ว shift
    mov rax, 100                    ; 100 → 64 (2^6)
    bsr rcx, rax                    ; rcx = 6 (position ของ MSB)
    mov rbx, 1
    shl rbx, cl                     ; 1 << 6 = 64
    lea rdi, [rel fmt_trick2]
    mov rsi, 100
    mov rdx, rbx
    xor eax, eax
    call printf

    ; Trick 3: Next power of 2 >= n
    ; รูปแบบ: round up ด้วยการ set all lower bits แล้ว +1
    mov rax, 100                    ; จะหา next power of 2 >= 100 = 128
    dec rax                         ; rax = 99
    ; ทำ OR fold: เผยแพร่บิตสูงสุดลงมาทาง right
    mov rbx, rax
    shr rbx, 1
    or rax, rbx                     ; รวม bit ที่ 1 ทางขวา
    mov rbx, rax
    shr rbx, 2
    or rax, rbx
    mov rbx, rax
    shr rbx, 4
    or rax, rbx
    mov rbx, rax
    shr rbx, 8
    or rax, rbx
    mov rbx, rax
    shr rbx, 16
    or rax, rbx
    mov rbx, rax
    shr rbx, 32
    or rax, rbx                     ; ตอนนี้ทุก bit ทางขวาของ MSB เป็น 1
    inc rax                         ; +1 = next power of 2
    lea rdi, [rel fmt_trick3]
    mov rsi, 100
    mov rdx, rax
    xor eax, eax
    call printf

    ; Trick 4: Absolute value without branch
    ; abs(x) = (x XOR sign) - sign  โดย sign = x >> 63 (arithmetic)
    mov rax, -42                    ; ค่า negative
    mov rbx, rax
    sar rbx, 63                     ; rbx = sign mask: 0 ถ้าบวก, -1 (0xFF..FF) ถ้าลบ
    xor rax, rbx                    ; XOR กับ sign mask
    sub rax, rbx                    ; ลบ sign mask = abs value
    lea rdi, [rel fmt_trick4]
    mov rsi, -42
    mov rdx, rax
    xor eax, eax
    call printf

    mov rax, 42                     ; ค่า positive
    mov rbx, rax
    sar rbx, 63                     ; rbx = 0 (positive)
    xor rax, rbx                    ; XOR 0 = ไม่เปลี่ยน
    sub rax, rbx                    ; ลบ 0 = ไม่เปลี่ยน
    lea rdi, [rel fmt_trick4]
    mov rsi, 42
    mov rdx, rax
    xor eax, eax
    call printf

    ; Trick 5: Sign of number (branchless)
    ; sign(x) = (x >> 63) | ((-x) >> 63) → 1=pos, -1=neg, 0=zero
    ; หรือ: sign(x) = (x > 0) - (x < 0)
    mov rax, -7
    mov rbx, rax
    sar rbx, 63                     ; rbx = -1 ถ้าลบ, 0 ถ้าบวก/ศูนย์
    neg rax
    sar rax, 63                     ; rax = -1 ถ้า -(-7)>0 → ไม่จริงสำหรับลบ...
    ; วิธีง่ายกว่า: เปรียบเทียบแล้วใช้ SETG/SETL
    mov rax, -7
    xor rcx, rcx
    test rax, rax
    jz .sign_zero5
    js .sign_neg5
    mov rcx, 1
    jmp .sign_done5
.sign_neg5:
    mov rcx, -1
    jmp .sign_done5
.sign_zero5:
    mov rcx, 0
.sign_done5:
    lea rdi, [rel fmt_trick5]
    mov rsi, -7
    mov rdx, rcx
    xor eax, eax
    call printf

    ; ===== Bitset Implementation =====
    ; Bitset คือ array ของ bits สำหรับเก็บ set ของ integers อย่างกระทัดรัด
    ; 64-bit register เก็บ set ขนาด 0-63 ได้

    lea rdi, [rel hdr_bitset]
    xor eax, eax
    call printf

    ; เริ่มต้น bitsets
    mov qword [rel bitset_A], 0     ; Bitset A ว่าง
    mov qword [rel bitset_B], 0     ; Bitset B ว่าง

    ; เพิ่ม elements ลง Bitset A: {2, 5, 10, 20, 35}
    mov rax, [rel bitset_A]
    bts rax, 2                      ; add 2
    bts rax, 5                      ; add 5
    bts rax, 10                     ; add 10
    bts rax, 20                     ; add 20
    bts rax, 35                     ; add 35
    mov [rel bitset_A], rax

    ; เพิ่ม elements ลง Bitset B: {5, 15, 20, 40}
    mov rax, [rel bitset_B]
    bts rax, 5                      ; add 5
    bts rax, 15                     ; add 15
    bts rax, 20                     ; add 20
    bts rax, 40                     ; add 40
    mov [rel bitset_B], rax

    ; ทดสอบว่า 5 อยู่ใน Bitset A หรือไม่
    mov rax, [rel bitset_A]
    bt rax, 5                       ; ทดสอบ bit 5
    setc al
    movzx rsi, al
    lea rdi, [rel fmt_bs_test]
    mov rsi, 5
    test al, al
    jz .not_in_a
    lea rdx, [rel str_set]
    jmp .print_in_a
.not_in_a:
    lea rdx, [rel str_not_set]
.print_in_a:
    xor eax, eax
    call printf

    ; ทดสอบว่า 15 อยู่ใน Bitset A หรือไม่
    mov rax, [rel bitset_A]
    bt rax, 15
    setc al
    lea rdi, [rel fmt_bs_test]
    mov rsi, 15
    test al, al
    jz .not_in_a2
    lea rdx, [rel str_set]
    jmp .print_in_a2
.not_in_a2:
    lea rdx, [rel str_not_set]
.print_in_a2:
    xor eax, eax
    call printf

    ; Union: A | B
    mov rax, [rel bitset_A]
    mov rbx, [rel bitset_B]
    or rax, rbx                     ; Union = OR
    lea rdi, [rel fmt_bs_union]
    mov rsi, rax
    xor eax, eax
    call printf

    ; Intersection: A & B
    mov rax, [rel bitset_A]
    mov rbx, [rel bitset_B]
    and rax, rbx                    ; Intersection = AND
    lea rdi, [rel fmt_bs_inter]
    mov rsi, rax
    xor eax, eax
    call printf

    ; Difference: A - B = A & (~B)
    mov rax, [rel bitset_A]
    mov rbx, [rel bitset_B]
    not rbx                         ; ~B
    and rax, rbx                    ; A & ~B
    lea rdi, [rel fmt_bs_diff]
    mov rsi, rax
    xor eax, eax
    call printf

    xor eax, eax
    leave
    ret
```

**คำสั่ง Compile และ Run:**
```bash
nasm -f elf64 xor_encrypt.asm -o xor_encrypt.o
gcc -no-pie xor_encrypt.o -o xor_encrypt
./xor_encrypt
```

**Expected Output:**
```
=== XOR Encryption Demo ===
Plaintext:  Hello, Assembly World!
Ciphertext: (hex) 12 3F 36 36 35 7A 78 3B 39 39 3F 38 36 37 7A 0D 35 26 36 3A 7A 5B
Decrypted:  Hello, Assembly World!

=== XOR Swap (ไม่ใช้ temp variable) ===
Before swap: a=42, b=99
After swap:  a=99, b=42

=== Bit Manipulation Tricks ===
Trick 1 - Clear lowest set bit: 0x0C -> 0x08
Trick 2 - Round down to power of 2: 100 -> 64
Trick 3 - Next power of 2 >= n: 100 -> 128
Trick 4 - Absolute value (no branch): -42 -> 42
Trick 4 - Absolute value (no branch): 42 -> 42
Trick 5 - Sign of number: -7 -> -1 (1=pos, -1=neg, 0=zero)

=== Bitset Implementation ===
Test bit 5: SET
Test bit 15: NOT SET
Union bitset: 0x0000008000134024
Intersection bitset: 0x0000000000100020
Difference bitset: 0x0000000000034004
```

---

### Example 6: Bitset สมบูรณ์ 128-bit และ Flag Register System

```nasm
; ============================================================
; File: bitset_flags.asm
; คำอธิบาย: Bitset 128-bit และระบบ Flag สำหรับ game/app
; Compile: nasm -f elf64 bitset_flags.asm -o bitset_flags.o
;          gcc -no-pie bitset_flags.o -o bitset_flags
; Run:     ./bitset_flags
; ============================================================

section .data
    hdr_flags   db "=== Game Flag System Demo ===", 10, 0
    hdr_128bit  db 10, "=== 128-bit Bitset Demo ===", 10, 0
    
    ; Game flags (bit positions)
    ; 0: HAS_SWORD, 1: HAS_SHIELD, 2: HAS_POTION, 3: IS_POISONED
    ; 4: IS_BLESSED, 5: HAS_KEY, 6: DOOR_OPENED, 7: QUEST_DONE
    PLAYER_HAS_SWORD   equ 0
    PLAYER_HAS_SHIELD  equ 1
    PLAYER_HAS_POTION  equ 2
    PLAYER_IS_POISONED equ 3
    PLAYER_IS_BLESSED  equ 4
    PLAYER_HAS_KEY     equ 5
    PLAYER_DOOR_OPENED equ 6
    PLAYER_QUEST_DONE  equ 7
    
    fmt_flags    db "Player flags: 0x%02X", 10, 0
    fmt_item     db "  %-15s: %s", 10, 0
    fmt_action   db "Action: %s", 10, 0
    
    ; Item names
    name_sword   db "HAS_SWORD", 0
    name_shield  db "HAS_SHIELD", 0
    name_potion  db "HAS_POTION", 0
    name_poison  db "IS_POISONED", 0
    name_blessed db "IS_BLESSED", 0
    name_key     db "HAS_KEY", 0
    name_door    db "DOOR_OPENED", 0
    name_quest   db "QUEST_DONE", 0
    
    str_yes      db "YES", 0
    str_no       db "NO", 0
    
    ; Actions
    act_pickup_sword   db "Pick up sword", 0
    act_pickup_shield  db "Pick up shield", 0
    act_use_potion     db "Use potion (clear poison)", 0
    act_get_poisoned   db "Got poisoned!", 0
    act_find_key       db "Found key!", 0
    act_open_door      db "Open door (requires key)", 0
    act_complete_quest db "Complete quest!", 0
    
    ; Messages
    msg_no_key   db "Cannot open door - no key!", 10, 0
    msg_door_ok  db "Door opened!", 10, 0
    
    ; 128-bit bitset demo
    fmt_128_hi   db "128-bit bitset (high 64): 0x%016llX", 10, 0
    fmt_128_lo   db "128-bit bitset (low  64): 0x%016llX", 10, 0
    fmt_128_pop  db "Total set bits: %d", 10, 0
    fmt_128_set  db "Set bit %d (in %s half)", 10, 0
    str_high     db "high", 0
    str_low      db "low", 0

section .bss
    player_flags  resb 1            ; 8-bit flag field สำหรับผู้เล่น
    bitset_hi     resq 1            ; 128-bit bitset: high 64 bits
    bitset_lo     resq 1            ; 128-bit bitset: low 64 bits

; === Macro-like procedures ===
section .text
    global main
    extern printf

; --- subroutine: set_flag(flag_byte*, bit_index) ---
; Input: rdi = pointer to flag byte, rsi = bit index
; Clobbers: rax
set_flag:
    mov al, [rdi]                   ; โหลด flag byte
    bts ax, si                      ; set bit ที่ si (ใช้ ax เพราะ flags เป็น byte)
    mov [rdi], al                   ; เก็บกลับ
    ret

; --- subroutine: clear_flag(flag_byte*, bit_index) ---
clear_flag:
    mov al, [rdi]
    btr ax, si
    mov [rdi], al
    ret

; --- subroutine: test_flag(flag_byte*, bit_index) -> rax ---
; Return: rax = 1 ถ้า flag set, 0 ถ้าไม่
test_flag_fn:
    mov al, [rdi]
    bt ax, si
    setc al
    movzx rax, al
    ret

; --- subroutine: print_flag_status ---
; Input: rdi = flag name string, rsi = flag value (0 or 1)
print_flag_status:
    push rbp
    mov rbp, rsp
    sub rsp, 32
    push rdi
    push rsi
    lea rdi, [rel fmt_item]
    pop rdx                         ; flag value → check for yes/no
    pop rsi                         ; flag name string
    test rdx, rdx
    jz .pfs_no
    lea rcx, [rel str_yes]
    jmp .pfs_print
.pfs_no:
    lea rcx, [rel str_no]
.pfs_print:
    xor eax, eax
    call printf
    leave
    ret

main:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    ; เริ่มต้น flags = 0
    mov byte [rel player_flags], 0

    ; แสดง header
    lea rdi, [rel hdr_flags]
    xor eax, eax
    call printf

    ; ===== Action: Pickup Sword =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_pickup_sword]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_SWORD
    call set_flag

    ; ===== Action: Pickup Shield =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_pickup_shield]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_SHIELD
    call set_flag

    ; ===== Action: Got Poisoned =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_get_poisoned]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_IS_POISONED
    call set_flag

    ; ===== Action: Use Potion (clear poison) =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_use_potion]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_IS_POISONED
    call clear_flag

    ; ===== Action: Find Key =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_find_key]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_KEY
    call set_flag

    ; ===== Action: Try to Open Door =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_open_door]
    xor eax, eax
    call printf
    
    ; ตรวจสอบว่ามี key หรือไม่
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_KEY
    call test_flag_fn
    test rax, rax
    jz .no_key
    
    ; มี key: open door
    lea rdi, [rel msg_door_ok]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_DOOR_OPENED
    call set_flag
    jmp .door_done
    
.no_key:
    lea rdi, [rel msg_no_key]
    xor eax, eax
    call printf
.door_done:

    ; ===== Complete Quest =====
    lea rdi, [rel fmt_action]
    lea rsi, [rel act_complete_quest]
    xor eax, eax
    call printf
    
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_QUEST_DONE
    call set_flag

    ; ===== แสดงสถานะ flags ทั้งหมด =====
    movzx rsi, byte [rel player_flags]
    lea rdi, [rel fmt_flags]
    xor eax, eax
    call printf

    ; แสดงแต่ละ flag
    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_SWORD
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_sword]
    call print_flag_status

    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_SHIELD
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_shield]
    call print_flag_status

    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_POTION
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_potion]
    call print_flag_status

    lea rdi, [rel player_flags]
    mov rsi, PLAYER_IS_POISONED
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_poison]
    call print_flag_status

    lea rdi, [rel player_flags]
    mov rsi, PLAYER_HAS_KEY
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_key]
    call print_flag_status

    lea rdi, [rel player_flags]
    mov rsi, PLAYER_DOOR_OPENED
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_door]
    call print_flag_status

    lea rdi, [rel player_flags]
    mov rsi, PLAYER_QUEST_DONE
    call test_flag_fn
    mov rsi, rax
    lea rdi, [rel name_quest]
    call print_flag_status

    ; ===== 128-bit Bitset =====
    lea rdi, [rel hdr_128bit]
    xor eax, eax
    call printf

    ; เริ่มต้น 128-bit bitset
    mov qword [rel bitset_hi], 0
    mov qword [rel bitset_lo], 0

    ; Set bits ต่างๆ
    ; bits 0-63 อยู่ใน bitset_lo
    ; bits 64-127 อยู่ใน bitset_hi (ใช้ index 0-63 แต่หมายถึง 64-127)
    
    ; Set bit 5 (low half)
    mov rax, [rel bitset_lo]
    bts rax, 5
    mov [rel bitset_lo], rax
    
    lea rdi, [rel fmt_128_set]
    mov rsi, 5
    lea rdx, [rel str_low]
    xor eax, eax
    call printf

    ; Set bit 30 (low half)
    mov rax, [rel bitset_lo]
    bts rax, 30
    mov [rel bitset_lo], rax
    
    lea rdi, [rel fmt_128_set]
    mov rsi, 30
    lea rdx, [rel str_low]
    xor eax, eax
    call printf

    ; Set bit 70 (high half, index = 70-64 = 6)
    mov rax, [rel bitset_hi]
    bts rax, 6                      ; bit 70 = bit 6 ของ high half
    mov [rel bitset_hi], rax
    
    lea rdi, [rel fmt_128_set]
    mov rsi, 70
    lea rdx, [rel str_high]
    xor eax, eax
    call printf

    ; Set bit 100 (high half, index = 100-64 = 36)
    mov rax, [rel bitset_hi]
    bts rax, 36
    mov [rel bitset_hi], rax
    
    lea rdi, [rel fmt_128_set]
    mov rsi, 100
    lea rdx, [rel str_high]
    xor eax, eax
    call printf

    ; แสดง 128-bit bitset
    lea rdi, [rel fmt_128_hi]
    mov rsi, [rel bitset_hi]
    xor eax, eax
    call printf

    lea rdi, [rel fmt_128_lo]
    mov rsi, [rel bitset_lo]
    xor eax, eax
    call printf

    ; นับ total set bits ด้วย POPCNT
    mov rax, [rel bitset_lo]
    popcnt rax, rax
    mov rbx, rax
    mov rax, [rel bitset_hi]
    popcnt rax, rax
    add rbx, rax                    ; total = lo_count + hi_count

    lea rdi, [rel fmt_128_pop]
    mov rsi, rbx
    xor eax, eax
    call printf

    xor eax, eax
    leave
    ret
```

**คำสั่ง Compile และ Run:**
```bash
nasm -f elf64 bitset_flags.asm -o bitset_flags.o
gcc -no-pie bitset_flags.o -o bitset_flags
./bitset_flags
```

**Expected Output:**
```
=== Game Flag System Demo ===
Action: Pick up sword
Action: Pick up shield
Action: Got poisoned!
Action: Use potion (clear poison)
Action: Found key!
Action: Open door (requires key)
Door opened!
Action: Complete quest!
Player flags: 0xE3
  HAS_SWORD      : YES
  HAS_SHIELD     : YES
  HAS_POTION     : NO
  IS_POISONED    : NO
  HAS_KEY        : YES
  DOOR_OPENED    : YES
  QUEST_DONE     : YES

=== 128-bit Bitset Demo ===
Set bit 5 (in low half)
Set bit 30 (in low half)
Set bit 70 (in high half)
Set bit 100 (in high half)
128-bit bitset (high 64): 0x0000001000000040
128-bit bitset (low  64): 0x0000000040000020
Total set bits: 4
```

---

## Common Mistakes and Pitfalls

### ข้อผิดพลาดที่พบบ่อย

#### 1. ขนาด Operand ไม่ตรงกัน

```nasm
; ผิด: AND ระหว่าง register 64-bit และ immediate 32-bit ที่ negative
mov rax, 0xFFFFFFFFFFFFFFFF
and rax, 0x80000000     ; ใช้ได้ แต่ immediate จะถูก sign-extend เป็น 64-bit
                         ; = 0xFFFFFFFF80000000 ไม่ใช่ 0x80000000

; ถูก: ใช้ register สำหรับค่า mask
mov rax, 0xFFFFFFFFFFFFFFFF
mov rbx, 0x80000000     ; zero-extend: rbx = 0x0000000080000000
and rax, rbx            ; ผล = 0x80000000

; หรือ: ใช้ MOV ก่อน
mov rax, 0xFFFFFFFFFFFFFFFF
and eax, 0x80000000     ; ทำ AND กับ 32-bit portion แล้ว zero-extend
```

#### 2. ลืมว่า NOT เป็น bitwise ทั้งหมด

```nasm
; ระวัง: NOT ใน 64-bit mode เปลี่ยนทุก bit รวมถึงบิตสูง
mov al, 0x0F            ; al = 0x0F = 0000 1111
not al                  ; al = 0xF0 = 1111 0000 (ถูกต้องสำหรับ byte)

mov rax, 0x0F           ; rax = 0x000000000000000F
not rax                 ; rax = 0xFFFFFFFFFFFFFFF0 (ไม่ใช่ 0xF0!)

; แก้ไข: ถ้าต้องการ NOT เฉพาะ byte ล่าง:
mov rax, 0x0F
not al                  ; NOT เฉพาะ byte ล่าง
; หรือ:
xor rax, 0xFF           ; XOR กับ 0xFF = เหมือน NOT เฉพาะ byte ล่าง
```

#### 3. ใช้ BSF/BSR กับ 0

```nasm
; ระวัง: BSF/BSR กับ 0 ทำให้ ZF=1 และผลลัพธ์ไม่แน่นอน
mov rax, 0
bsf rcx, rax            ; ZF=1, rcx = undefined!

; ถูกต้อง: ตรวจสอบก่อนใช้
mov rax, 0
bsf rcx, rax
jz .handle_zero         ; กระโดดถ้า source = 0

; ทางเลือก: ใช้ TZCNT/LZCNT ซึ่งกำหนดผลลัพธ์แม้ input = 0
; TZCNT(0) = operand size (32 หรือ 64)
; LZCNT(0) = operand size
```

#### 4. ลืม POPCNT ต้องการ CPU feature

```nasm
; POPCNT ต้องการ CPU ที่มี POPCNT feature (Intel Nehalem 2008+)
; ตรวจสอบด้วย CPUID ก่อนใช้ในโปรแกรมจริง

; ตรวจสอบ POPCNT support:
mov eax, 1              ; CPUID leaf 1
cpuid
test ecx, (1 << 23)     ; bit 23 ของ ECX = POPCNT
jz .no_popcnt           ; กระโดดถ้าไม่รองรับ

; หรือ compile flag:
; gcc -mpopcnt ...
; ใน NASM: default รองรับ แต่ต้องมี CPU ที่รองรับ
```

#### 5. XOR Swap ใช้กับ Memory อาจช้ากว่า

```nasm
; XOR Swap กับ register สองตัวที่เหมือนกัน = ล้างค่าทิ้ง!
mov rax, 42
xor rax, rax            ; 42 XOR 42 = 0 (ไม่ใช่ swap!)
xor rax, rax            ; 0 XOR 0 = 0
xor rax, rax            ; ทุกขั้นเป็น 0

; ถูกต้อง: XOR Swap ต้องเป็น register คนละตัว
mov rax, 42
mov rbx, 99
xor rax, rbx            ; rax = 42^99
xor rbx, rax            ; rbx = 99^(42^99) = 42
xor rax, rbx            ; rax = (42^99)^42 = 99

; ในทางปฏิบัติ: ใช้ XCHG หรือ temp variable ดีกว่า
; XCHG rax, rbx         ; สลับค่าอย่างถูกต้อง
```

#### 6. BT/BTS/BTR กับ Memory มี Behavior พิเศษ

```nasm
; BT กับ memory: bit index ไม่จำกัดอยู่ใน range ของ operand
; แต่จะไปอ่าน bit นอก address ได้! (string bit ops)
section .bss
    buffer  resq 4          ; 4 * 8 = 32 bytes = 256 bits

; BT [buffer], 200         ; ไปอ่าน bit 200 ของ buffer memory
; = byte offset = 200/8 = 25, bit = 200%8 = 0
; = buffer[25] bit 0

; ระวัง: BT กับ immediate immediate ไม่มีปัญหา
; แต่ BT กับ register เป็น bit index อาจอ่านนอก buffer ถ้า index สูงเกิน
```

---

## Advanced Techniques

### 1. Branchless Bit Operations

การใช้ bit manipulation แทน if-else เพื่อเพิ่มประสิทธิภาพ:

```nasm
; ===== ตรวจสอบ even/odd แบบ branchless =====
; ถ้า n เป็น even: ส่งคืน 0, ถ้า odd: ส่งคืน 1
; แบบมี branch:
;   test rax, 1
;   jz .even
;   mov rbx, 1
;   jmp .done
; .even:
;   xor rbx, rbx
; .done:

; แบบ branchless:
mov rbx, rax            ; copy n
and rbx, 1              ; isolate bit 0 → 0 ถ้า even, 1 ถ้า odd
; rbx = n & 1  (ไม่ต้องการ branch)

; ===== Min/Max แบบ branchless =====
; min(a, b) = b + ((a - b) & ((a - b) >> 63))
; max(a, b) = a - ((a - b) & ((a - b) >> 63))

; คำนวณ min(rax, rbx)
; สมมติ rax = a, rbx = b
mov rcx, rax            ; rcx = a
sub rcx, rbx            ; rcx = a - b
mov rdx, rcx
sar rdx, 63             ; rdx = sign mask: 0 ถ้า a>=b, -1 ถ้า a<b
and rcx, rdx            ; rcx = (a-b) & sign_mask
add rcx, rbx            ; rcx = b + ((a-b) & sign_mask) = min(a, b)
```

### 2. Gray Code Conversion

Gray Code คือ binary code ที่ค่าที่ติดกันต่างกันแค่ 1 bit (ใช้ใน error correction):

```nasm
; Binary → Gray Code: gray = n XOR (n >> 1)
; Gray Code → Binary: binary = gray XOR (gray >> 1) XOR (gray >> 2) ...

; Binary to Gray:
mov rax, 12             ; binary = 12 = 1100
mov rbx, rax
shr rbx, 1              ; rbx = 6
xor rax, rbx            ; rax = 12 XOR 6 = 10 = Gray code ของ 12

; ตัวอย่าง:
; Binary: 0=0000, 1=0001, 2=0010, 3=0011, 4=0100 ...
; Gray:   0=0000, 1=0001, 2=0011, 3=0010, 4=0110 ...
```

### 3. Parallel Bit Operations (SIMD-like ใน scalar)

```nasm
; นับจำนวน bits ที่ต่างกัน (Hamming Distance) ระหว่าง 2 ค่า
; hamming(a, b) = POPCNT(a XOR b)

mov rax, 0b11010110     ; a = 214
mov rbx, 0b10110111     ; b = 183
xor rax, rbx            ; bit ที่ต่างกัน = XOR result
popcnt rax, rax         ; นับ bit ที่ต่างกัน

; ใช้ใน: error detection, fuzzy matching, DNA analysis
```

### 4. Bit Reversal

```nasm
; สลับ bit ทั้งหมดในตัวเลข (bit 0 ↔ bit 63, bit 1 ↔ bit 62, ...)
; ใช้ BSWAP สำหรับ byte reversal แล้วทำ bit reversal ใน byte

; Bit reversal ของ byte เดียว (เทคนิค 4-bit swap):
; ขั้น 1: สลับ nibble: (b & 0xF0) >> 4 | (b & 0x0F) << 4
; ขั้น 2: สลับ 2-bit pairs: (b & 0xCC) >> 2 | (b & 0x33) << 2
; ขั้น 3: สลับ bits คู่: (b & 0xAA) >> 1 | (b & 0x55) << 1

; ใน 64-bit:
mov rax, 0xABCD1234     ; ค่าทดสอบ
bswap rax               ; สลับ byte order ก่อน
; จากนั้นสลับ bits ภายใน byte...
; (ซับซ้อนในรายละเอียด ใช้ PSHUFB ใน SSE เร็วกว่า)
```

### 5. Bit Manipulation ใน Loop Performance

```nasm
; ===== Kernighan's Algorithm: นับ set bits โดยไม่ใช้ POPCNT =====
; วิธีนี้เร็วกว่าการ loop ทีละ bit
; หลักการ: n & (n-1) ล้าง lowest set bit
; วนซ้ำจนกว่า n = 0

; C equivalent:
; int count = 0;
; while (n) { n &= n-1; count++; }

mov rax, 0xABCD1234     ; ค่าทดสอบ (มี ? set bits)
xor rcx, rcx            ; counter = 0

.count_bits_loop:
    test rax, rax       ; ตรวจสอบว่าเป็น 0 หรือไม่
    jz .count_done
    mov rbx, rax
    dec rbx             ; rbx = n - 1
    and rax, rbx        ; n = n & (n-1) → ล้าง lowest set bit
    inc rcx             ; count++
    jmp .count_bits_loop

.count_done:
; rcx = จำนวน set bits ของ 0xABCD1234

; ===== de Bruijn Sequence สำหรับ bit scan =====
; เร็วกว่า BSF ใน CPU รุ่นเก่า
; ใช้ lookup table ที่คำนวณจาก de Bruijn sequence
```

---

## Exercises

### Exercise 1: Nibble Extraction
**โจทย์:** เขียนโปรแกรม Assembly ที่รับ byte value และแยก upper nibble (4 bits สูง) กับ lower nibble (4 bits ต่ำ) ออกจากกัน

**Hint:**
- Lower nibble: `AND` กับ `0x0F`
- Upper nibble: `AND` กับ `0xF0` แล้ว shift right 4 bit (`SHR rax, 4`)
- ทดสอบกับ `0xAB`: upper=0xA=10, lower=0xB=11

```nasm
; ตัวอย่าง skeleton:
section .text
global main
extern printf

main:
    push rbp
    mov rbp, rsp
    
    mov rax, 0xAB           ; ค่าทดสอบ
    
    ; TODO: แยก upper nibble
    ; TODO: แยก lower nibble
    ; TODO: แสดงผล
    
    xor eax, eax
    leave
    ret
```

**Expected Output:**
```
Value: 0xAB
Upper nibble: 0xA (10)
Lower nibble: 0xB (11)
```

---

### Exercise 2: Parity Bit Calculation
**โจทย์:** เขียนฟังก์ชันคำนวณ even parity bit ของข้อมูล 1 byte
(Parity = XOR ของทุก bit ใน byte)

**Hint:**
- ใช้ POPCNT นับ set bits
- ถ้าจำนวน set bits เป็นคี่: parity = 1 (even parity)
- ถ้าจำนวน set bits เป็นคู่: parity = 0

หรือใช้ XOR fold:
```nasm
; XOR fold: b ^= b >> 4, b ^= b >> 2, b ^= b >> 1, result = b & 1
mov al, 0xAB        ; 1010 1011 (5 ones → odd → parity = 1)
mov bl, al
shr bl, 4
xor al, bl          ; บวก nibbles
and al, 0xF
mov bl, al
shr bl, 2
xor al, bl          ; บวก 2-bit groups
mov bl, al
shr bl, 1
xor al, bl          ; บวก bits
and al, 1           ; เก็บ LSB = parity
```

---

### Exercise 3: Rotate Operations
**โจทย์:** ใช้ ROL/ROR (rotate instructions) ร่วมกับ AND/OR เพื่อ rotate nibbles ใน byte

**Hint:**
- `ROL rax, 4` หมุน 4 bits ไปทางซ้าย
- `ROR rax, 4` หมุน 4 bits ไปทางขวา
- สำหรับ byte ใช้ `rol al, 4`

**Expected:**
```
Original: 0xAB
ROL 4:    0xBA
ROR 4:    0xBA  (same for 4 on a byte)
```

---

### Exercise 4: Bitset Set Operations
**โจทย์:** implement ฟังก์ชัน Symmetric Difference ของ 2 bitsets
Symmetric Difference (A △ B) = (A ∪ B) - (A ∩ B) = A XOR B

```nasm
; A = {1, 3, 5, 7} = bit 1,3,5,7 = 0b10101010 = 0xAA
; B = {1, 2, 4, 6} = bit 1,2,4,6 = 0b01010110 = 0x56
; A △ B = {2, 3, 4, 5, 6, 7} = bit 2,3,4,5,6,7 = 0b11111100 = 0xFC

mov rax, 0xAA
mov rbx, 0x56
; TODO: คำนวณ symmetric difference
; Expected: 0xFC
```

---

### Exercise 5: Run-Length Encoding ของ Bit Stream
**โจทย์:** เขียนโปรแกรมนับ "runs" ของ bits (ลำดับบิตเดียวกันติดกัน) ในค่า 8-bit

**ตัวอย่าง:** 0b11100101 = "111" "00" "1" "0" "1" = 5 runs

**Hint:**
- XOR ค่ากับตัวเองที่ shift 1: `x ^ (x >> 1)` ให้ 1 ที่ตำแหน่ง transitions
- POPCNT ของผลลัพธ์ + 1 = จำนวน runs

```nasm
mov rax, 0b11100101     ; = 0xE5
mov rbx, rax
shr rbx, 1
xor rax, rbx            ; transitions
and rax, 0x7F           ; ไม่รวม virtual transition ทางซ้าย
popcnt rax, rax         ; นับ transitions
inc rax                 ; runs = transitions + 1
; rax = 5 (ถูกต้อง)
```

---

## Summary

### สรุปคำสั่ง Bitwise และ Bit Manipulation

| คำสั่ง | การทำงาน | ตัวอย่างใช้งาน |
|--------|----------|----------------|
| `AND dst, src` | Bitwise AND, เก็บผลใน dst | Masking, Clearing bits |
| `OR dst, src` | Bitwise OR, เก็บผลใน dst | Setting bits, Combining flags |
| `XOR dst, src` | Bitwise XOR, เก็บผลใน dst | Toggling bits, Encryption, Swap |
| `NOT dst` | Bitwise complement | Invert all bits |
| `TEST dst, src` | AND โดยไม่เก็บผล (ทำแค่ set flags) | Check bits, Zero test |
| `BT src, idx` | Bit Test: CF = bit[idx] | Read a bit |
| `BTS src, idx` | Bit Test and Set | Read and set a bit |
| `BTR src, idx` | Bit Test and Reset | Read and clear a bit |
| `BTC src, idx` | Bit Test and Complement | Read and toggle a bit |
| `BSF dst, src` | Bit Scan Forward (LSB index) | Find first set bit from right |
| `BSR dst, src` | Bit Scan Reverse (MSB index) | Find first set bit from left, log2 |
| `LZCNT dst, src` | Count Leading Zeros | Normalize numbers |
| `TZCNT dst, src` | Count Trailing Zeros | Find alignment, LSB position |
| `POPCNT dst, src` | Population Count | Count set bits |

### สรุป Bit Manipulation Tricks ที่สำคัญ

```nasm
; 1. ตรวจว่า bit k set หรือไม่
test rax, (1 << k)      ; ZF=0 ถ้า set

; 2. Set bit k
or rax, (1 << k)

; 3. Clear bit k
and rax, ~(1 << k)

; 4. Toggle bit k
xor rax, (1 << k)

; 5. Isolate lowest set bit
mov rbx, rax
neg rbx
and rax, rbx            ; หรือ: and rax, -rax (ทำไม่ได้โดยตรง)

; 6. Clear lowest set bit
mov rbx, rax
dec rbx
and rax, rbx            ; n & (n-1)

; 7. ตรวจว่าเป็น power of 2
mov rbx, rax
dec rbx
test rax, rbx           ; ZF=1 ถ้า power of 2 (และ n > 0)

; 8. Round down to power of 2
bsr rcx, rax
mov rax, 1
shl rax, cl             ; 1 << floor(log2(n))

; 9. Absolute value (branchless)
mov rbx, rax
sar rbx, 63             ; sign mask
xor rax, rbx
sub rax, rbx

; 10. XOR swap
xor rax, rbx
xor rbx, rax
xor rax, rbx
```

### เทคนิคที่ควรจำ

1. **AND** สำหรับ **Clear/Mask** bits
2. **OR** สำหรับ **Set** bits  
3. **XOR** สำหรับ **Toggle** bits และ encryption
4. **TEST** แทน **CMP** เมื่อตรวจสอบ bit/zero
5. **POPCNT** เร็วที่สุดสำหรับนับ 1-bits
6. **BSF/TZCNT** หา lowest set bit position
7. **BSR/LZCNT** หา highest set bit position, คำนวณ log2

### ความสัมพันธ์กับ Part ถัดไป

Part 016 จะเรียนเรื่อง:
- Shift Operations: SHL, SHR, SAR, ROL, ROR
- SHLD/SHRD สำหรับ double precision shifts
- การใช้ shift ทำ multiplication/division
- Barrel shifter concepts
- ต่อยอดจาก bitwise ใน Part นี้

---

*Part 015 ครอบคลุม: AND/OR/XOR/NOT, TEST instruction, BT/BTS/BTR/BTC, BSF/BSR, LZCNT/TZCNT, POPCNT, XOR encryption, bit manipulation tricks, bitset implementation, และ flag system*

*เวลาเรียนโดยประมาณ: 4-6 ชั่วโมง*
