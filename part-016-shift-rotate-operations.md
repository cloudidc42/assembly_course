# Part 016: Shift และ Rotate Operations

## Prerequisites and Learning Objectives

### สิ่งที่ต้องรู้ก่อน (Prerequisites)
- ความเข้าใจเรื่อง Binary และ Hexadecimal (Part 001-003)
- การใช้งาน Registers เบื้องต้น (Part 004-006)
- คำสั่ง Arithmetic พื้นฐาน (Part 010-012)
- Flags Register และการทำงาน (Part 013)
- Bitwise Operations (Part 015)

### วัตถุประสงค์การเรียนรู้ (Learning Objectives)
หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
1. เข้าใจความแตกต่างระหว่าง Logical Shift และ Arithmetic Shift
2. ใช้คำสั่ง SHL, SAL, SHR, SAR ได้อย่างถูกต้อง
3. ใช้คำสั่ง ROL, ROR, RCL, RCR สำหรับการหมุน bits
4. ใช้ SHLD/SHRD สำหรับ Double Precision Shift
5. นำ Shift Operations มาใช้แทนการคูณ/หารด้วยเลขยกกำลัง 2
6. ทำ Bit Field Extraction และ Packing/Unpacking Data
7. คำนวณ CRC และ Fast Modulo ด้วย Shifts
8. แปลง Hex Digits โดยใช้ Shift Operations

---

## Theory Part 1: ความรู้พื้นฐานเรื่อง Shift Operations

### Shift คืออะไร?

Shift Operation คือการเลื่อนตำแหน่ง bits ภายใน register ไปทางซ้ายหรือขวา
เปรียบได้กับการคูณหรือหารด้วย 2 ยกกำลัง n

```
ตัวอย่าง: SHL (Shift Left) 1 bit
ก่อน: 0000 1010  (10 decimal)
หลัง: 0001 0100  (20 decimal) = 10 × 2 = 20

ตัวอย่าง: SHR (Shift Right) 1 bit
ก่อน: 0001 0100  (20 decimal)
หลัง: 0000 1010  (10 decimal) = 20 ÷ 2 = 10
```

### ประเภทของ Shift Operations

| คำสั่ง | ชื่อเต็ม | การทำงาน |
|--------|----------|----------|
| SHL | Shift Left Logical | เลื่อนซ้าย เติม 0 ด้านขวา |
| SAL | Shift Arithmetic Left | เหมือน SHL (identical) |
| SHR | Shift Right Logical | เลื่อนขวา เติม 0 ด้านซ้าย |
| SAR | Shift Arithmetic Right | เลื่อนขวา เติม Sign Bit |
| ROL | Rotate Left | หมุนซ้าย bit หลุดกลับมาด้านขวา |
| ROR | Rotate Right | หมุนขวา bit หลุดกลับมาด้านซ้าย |
| RCL | Rotate Left through Carry | หมุนซ้ายผ่าน Carry Flag |
| RCR | Rotate Right through Carry | หมุนขวาผ่าน Carry Flag |
| SHLD | Shift Left Double | Shift ซ้ายสองค่าพร้อมกัน |
| SHRD | Shift Right Double | Shift ขวาสองค่าพร้อมกัน |

---

## Theory Part 2: Logical vs Arithmetic Shift

### Logical Shift (SHL/SHR)
- **SHL**: เลื่อน bits ไปทางซ้าย n ตำแหน่ง
  - bit ที่หลุดออกไปเข้า Carry Flag
  - ด้านขวาเติมด้วย 0
- **SHR**: เลื่อน bits ไปทางขวา n ตำแหน่ง
  - bit ที่หลุดออกไปเข้า Carry Flag
  - ด้านซ้ายเติมด้วย 0

```
SHL ตัวอย่าง:
CF ← [1011 0110] ← 0
     1011 0110  (182)
     0110 1100  (108)  << SHL 1

SHR ตัวอย่าง:
0 → [1011 0110] → CF
     1011 0110  (182)
     0101 1011  (91)   >> SHR 1
```

### Arithmetic Shift (SAR)
- **SAL**: เหมือน SHL ทุกประการ (same opcode)
- **SAR**: เลื่อนขวาแต่รักษา Sign Bit (MSB)
  - ใช้สำหรับหารจำนวนที่มีเครื่องหมาย (signed division)
  - เติม bit เดียวกับ MSB ด้านซ้าย

```
SAR ตัวอย่าง (จำนวนลบ):
     1011 0110  (-74 ใน 2's complement)
     1101 1011  (-37)  >> SAR 1  (เติม 1 เพราะ MSB = 1)

SHR ตัวอย่าง (จำนวนลบ แต่ผลผิด):
     1011 0110  (-74)
     0101 1011  (91)   >> SHR 1  (เติม 0 เสมอ = ผลเป็น unsigned)
```

**สำคัญ**: ใช้ SAR สำหรับ signed number, SHR สำหรับ unsigned number

---

## Theory Part 3: Rotate Operations

### ROL (Rotate Left)
bit ที่หลุดออกจากซ้ายกลับมาที่ขวา และเข้า Carry Flag ด้วย

```
ROL AX, 1:
CF ← [MSB ... LSB] ← MSB (wrap around)

ก่อน:  CF=0  AX = 1011 0110
หลัง:  CF=1  AX = 0110 1101
       (bit 7 หลุด → ไป bit 0 และ CF)
```

### ROR (Rotate Right)
bit ที่หลุดออกจากขวากลับมาที่ซ้าย และเข้า Carry Flag ด้วย

```
ROR AX, 1:
MSB (wrap around) → [MSB ... LSB] → CF

ก่อน:  CF=0  AX = 1011 0110
หลัง:  CF=0  AX = 0101 1011
       (bit 0 = 0 หลุด → ไป bit 7 และ CF)
```

### RCL (Rotate through Carry Left)
CF เข้าร่วมการ rotate ด้วย ทำให้มี 9 bits (8-bit register + CF)

```
RCL AX, 1:
CF_old → [MSB ... LSB] ← CF_old
MSB_old → CF

ก่อน:  CF=1  AX = 1011 0110
หลัง:  CF=1  AX = 0110 1101
       (CF=1 เข้า bit 0, bit 7=1 ออกไป CF)
```

### RCR (Rotate through Carry Right)
```
RCR AX, 1:
CF_old → [MSB ... LSB] → CF_old
         LSB_old → CF

ก่อน:  CF=0  AX = 1011 0110
หลัง:  CF=0  AX = 0101 1011
       (CF=0 เข้า bit 7, bit 0=0 ออกไป CF)
```

---

## Theory Part 4: SHLD และ SHRD

### SHLD (Shift Left Double)
ใช้เลื่อนค่าสองตัวพร้อมกัน โดย bits ที่หลุดจาก destination มาจาก source

```
SHLD dest, src, count
; dest เลื่อนซ้าย count bits
; bits ที่หลุดออกจาก dest ด้านขวา มาจาก MSB ของ src
; src ไม่เปลี่ยน

ตัวอย่าง:
EAX = 0000 0000 1111 1111 (0x000000FF)
EBX = 1100 0000 0000 0000 (0xC0000000)
SHLD EAX, EBX, 2
EAX = 0000 0011 1111 1111  (EAX เลื่อน 2 ซ้าย + 2 MSB ของ EBX)
```

### SHRD (Shift Right Double)
```
SHRD dest, src, count
; dest เลื่อนขวา count bits
; bits ที่หลุดออกจาก dest ด้านซ้าย มาจาก LSB ของ src

ตัวอย่าง:
EAX = 0000 0000 1111 1111 (0x000000FF)
EBX = 0000 0000 0000 0011 (0x00000003)
SHRD EAX, EBX, 2
EAX = 1100 0000 0011 1111  (EAX เลื่อน 2 ขวา + 2 LSB ของ EBX เข้า MSB)
```

---

## Theory Part 5: Flag Effects

### Flags ที่ได้รับผลกระทบ

| คำสั่ง | CF | OF | SF | ZF | PF |
|--------|----|----|----|----|-----|
| SHL n=1 | bit สุดท้ายที่หลุด | CF xor MSB ใหม่ | * | * | * |
| SHR n=1 | bit สุดท้ายที่หลุด | MSB เดิม | * | * | * |
| SAR n=1 | bit สุดท้ายที่หลุด | 0 (always) | * | * | * |
| ROL | MSB ก่อนหมุน | CF xor MSB ใหม่ | - | - | - |
| ROR | LSB ก่อนหมุน | XOR สอง MSBs | - | - | - |
| RCL | MSB ก่อนหมุน | CF xor MSB ใหม่ | - | - | - |
| RCR | LSB ก่อนหมุน | CF xor MSB ใหม่ | - | - | - |

หมายเหตุ: * = ได้รับผลกระทบ, - = ไม่เปลี่ยน, CF = Carry Flag

---

## Code Example 1: Basic Shift Operations

```nasm
; ===================================================
; ไฟล์: shift_basic.asm
; คำอธิบาย: แสดงการทำงานของ SHL, SHR, SAR พื้นฐาน
; ===================================================
; การ Compile:
;   nasm -f elf64 shift_basic.asm -o shift_basic.o
;   ld shift_basic.o -o shift_basic
; การรัน:
;   ./shift_basic
; ===================================================

section .data
    ; ข้อความแสดงผล
    msg_shl     db "=== SHL (Shift Left) ===" , 10, 0
    msg_shr     db "=== SHR (Shift Right) ===", 10, 0
    msg_sar     db "=== SAR (Arithmetic Right) ===", 10, 0
    msg_orig    db "Original value: ", 0
    msg_after   db "After shift:    ", 0
    newline     db 10, 0
    
    ; ค่าทดสอบ
    val_pos     dq 20          ; ค่าบวก 20 (0x14)
    val_neg     dq -20         ; ค่าลบ -20

section .bss
    buffer      resb 32        ; buffer สำหรับแปลงตัวเลข

section .text
    global _start

; ======================================
; Subroutine: print_string
; Input: RSI = address of string
; ======================================
print_string:
    push rax
    push rdx
    push rdi
    
    ; หา length ของ string
    mov rdx, 0
.count_loop:
    cmp byte [rsi + rdx], 0   ; ตรวจสอบ null terminator
    je .done_count
    inc rdx
    jmp .count_loop
    
.done_count:
    mov rax, 1                 ; sys_write
    mov rdi, 1                 ; stdout
    syscall
    
    pop rdi
    pop rdx
    pop rax
    ret

; ======================================
; Subroutine: print_decimal
; Input: RAX = ค่าที่ต้องการแสดง (signed)
; ======================================
print_decimal:
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    
    lea rsi, [buffer + 31]     ; ชี้ไปที่ท้าย buffer
    mov byte [rsi], 0          ; null terminator
    
    ; ตรวจสอบว่าเป็นลบหรือไม่
    mov rbx, 0                 ; flag: 0 = บวก, 1 = ลบ
    test rax, rax
    jns .positive              ; ถ้าไม่ใช่ลบ ข้ามไป
    neg rax                    ; เปลี่ยนเป็นบวก
    mov rbx, 1                 ; ตั้ง flag ว่าลบ
    
.positive:
    mov rcx, 10                ; หารด้วย 10

.convert_loop:
    dec rsi                    ; เลื่อน pointer ไปซ้าย
    xor rdx, rdx               ; เคลียร์ RDX ก่อนหาร
    div rcx                    ; RAX = quotient, RDX = remainder
    add dl, '0'                ; แปลงเป็น ASCII
    mov [rsi], dl              ; เก็บ digit
    test rax, rax              ; ตรวจสอบว่า RAX = 0 หรือยัง
    jnz .convert_loop          ; ถ้าไม่ใช่ 0 วนต่อ
    
    ; เติมเครื่องหมายลบถ้าจำเป็น
    test rbx, rbx
    jz .print_it
    dec rsi
    mov byte [rsi], '-'
    
.print_it:
    ; คำนวณ length
    lea rdx, [buffer + 31]
    sub rdx, rsi               ; length = end - current
    
    mov rax, 1                 ; sys_write
    mov rdi, 1                 ; stdout
    syscall
    
    ; ขึ้นบรรทัดใหม่
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel newline]
    mov rdx, 1
    syscall
    
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

_start:
    ; ==========================================
    ; ทดสอบ SHL - Shift Left Logical
    ; ==========================================
    lea rsi, [rel msg_shl]
    call print_string
    
    ; แสดงค่าเดิม
    lea rsi, [rel msg_orig]
    call print_string
    mov rax, [rel val_pos]     ; โหลด 20
    call print_decimal
    
    ; SHL 1 bit = คูณ 2
    mov rax, [rel val_pos]     ; โหลด 20 อีกครั้ง
    shl rax, 1                 ; เลื่อนซ้าย 1 bit = 20 * 2 = 40
    lea rsi, [rel msg_after]
    call print_string
    call print_decimal         ; ควรได้ 40
    
    ; SHL 2 bits = คูณ 4
    lea rsi, [rel msg_orig]
    call print_string
    mov rax, [rel val_pos]
    call print_decimal
    
    mov rax, [rel val_pos]     ; โหลด 20
    shl rax, 2                 ; เลื่อนซ้าย 2 bits = 20 * 4 = 80
    lea rsi, [rel msg_after]
    call print_string
    call print_decimal         ; ควรได้ 80
    
    ; ==========================================
    ; ทดสอบ SHR - Shift Right Logical
    ; ==========================================
    lea rsi, [rel msg_shr]
    call print_string
    
    ; SHR 1 bit = หาร 2 (unsigned)
    lea rsi, [rel msg_orig]
    call print_string
    mov rax, [rel val_pos]
    call print_decimal
    
    mov rax, [rel val_pos]     ; โหลด 20
    shr rax, 1                 ; เลื่อนขวา 1 bit = 20 / 2 = 10
    lea rsi, [rel msg_after]
    call print_string
    call print_decimal         ; ควรได้ 10
    
    ; ==========================================
    ; ทดสอบ SAR - Arithmetic Right Shift
    ; (สำหรับจำนวนลบ)
    ; ==========================================
    lea rsi, [rel msg_sar]
    call print_string
    
    ; SAR กับค่าลบ
    lea rsi, [rel msg_orig]
    call print_string
    mov rax, [rel val_neg]     ; โหลด -20
    call print_decimal
    
    mov rax, [rel val_neg]     ; โหลด -20
    sar rax, 1                 ; เลื่อนขวา 1 bit = -20 / 2 = -10
    lea rsi, [rel msg_after]
    call print_string
    call print_decimal         ; ควรได้ -10
    
    ; ทดสอบ SHR กับค่าลบ (ผลผิด!)
    mov rax, [rel val_neg]     ; โหลด -20
    shr rax, 1                 ; เลื่อนขวา 1 bit แบบ logical (ผลผิด)
    lea rsi, [rel msg_after]
    call print_string
    call print_decimal         ; จะได้ค่าบวกขนาดใหญ่ (ไม่ใช่ -10)
    
    ; จบโปรแกรม
    mov rax, 60               ; sys_exit
    xor rdi, rdi              ; exit code 0
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 shift_basic.asm -o shift_basic.o
ld shift_basic.o -o shift_basic
./shift_basic
```

**Expected Output:**
```
=== SHL (Shift Left) ===
Original value: 20
After shift:    40
Original value: 20
After shift:    80
=== SHR (Shift Right) ===
Original value: 20
After shift:    10
=== SAR (Arithmetic Right) ===
Original value: -20
After shift:    -10
After shift:    9223372036854775806
```

---

## Code Example 2: Rotate Operations

```nasm
; ===================================================
; ไฟล์: rotate_ops.asm
; คำอธิบาย: แสดงการทำงานของ ROL, ROR, RCL, RCR
; ===================================================
; การ Compile:
;   nasm -f elf64 rotate_ops.asm -o rotate_ops.o
;   ld rotate_ops.o -o rotate_ops
; การรัน:
;   ./rotate_ops
; ===================================================

section .data
    header      db "=== Rotate Operations Demo ===", 10, 0
    msg_rol     db "ROL result: ", 0
    msg_ror     db "ROR result: ", 0
    msg_rcl     db "RCL result: ", 0
    msg_rcr     db "RCR result: ", 0
    msg_hex     db "Hex: 0x", 0
    newline     db 10, 0
    
    ; ค่าทดสอบ
    test_val    db 0xB6        ; 1011 0110 in binary

section .bss
    hex_buf     resb 20

section .text
    global _start

; ======================================
; Subroutine: print_str_len
; Input: RSI = string, RDX = length
; ======================================
print_str_len:
    mov rax, 1
    mov rdi, 1
    syscall
    ret

; ======================================
; Subroutine: print_cstring
; Input: RSI = null-terminated string
; ======================================
print_cstring:
    push rcx
    xor rcx, rcx
.loop:
    cmp byte [rsi + rcx], 0
    je .done
    inc rcx
    jmp .loop
.done:
    mov rdx, rcx
    call print_str_len
    pop rcx
    ret

; ======================================
; Subroutine: print_byte_hex
; Input: AL = byte to print as hex
; ======================================
print_byte_hex:
    push rax
    push rbx
    push rcx
    push rsi
    
    lea rsi, [rel hex_buf]
    
    ; แปลง nibble สูง
    mov bl, al
    shr bl, 4                  ; เลื่อนขวา 4 bits เพื่อได้ nibble สูง
    and bl, 0x0F               ; mask เอาแค่ 4 bits ล่าง
    cmp bl, 10
    jl .digit1                 ; ถ้า < 10 ใช้ตัวเลข
    add bl, 'A' - 10           ; ถ้า >= 10 ใช้ตัวอักษร A-F
    jmp .store1
.digit1:
    add bl, '0'
.store1:
    mov [rsi], bl
    
    ; แปลง nibble ต่ำ
    mov bl, al
    and bl, 0x0F               ; mask เอาแค่ 4 bits ล่าง
    cmp bl, 10
    jl .digit2
    add bl, 'A' - 10
    jmp .store2
.digit2:
    add bl, '0'
.store2:
    mov [rsi + 1], bl
    mov byte [rsi + 2], 10     ; newline
    
    mov rdx, 3                 ; พิมพ์ 3 chars (2 hex + newline)
    call print_str_len
    
    pop rsi
    pop rcx
    pop rbx
    pop rax
    ret

; ======================================
; Subroutine: print_byte_binary
; Input: AL = byte to print as binary
; ======================================
print_byte_binary:
    push rax
    push rbx
    push rcx
    push rsi
    
    lea rsi, [rel hex_buf]
    mov rcx, 8                 ; 8 bits
    
.bit_loop:
    mov bl, al
    shl bl, 1                  ; เลื่อนซ้าย เพื่อให้ bit สูงสุดเข้า CF
    ; ใช้ setc เพื่อรับ CF
    ; แต่ใน 64-bit ต้องระวัง
    rol al, 1                  ; หมุนซ้าย เพื่อให้ MSB เข้า LSB
    and al, 1                  ; เอาแค่ LSB
    add al, '0'                ; แปลงเป็น ASCII
    ; *** ต้องใช้ตัวแปรแยก ***
    
    pop rsi
    pop rcx
    pop rbx
    pop rax
    ret

_start:
    ; พิมพ์ header
    lea rsi, [rel header]
    call print_cstring
    
    ; ==========================================
    ; ทดสอบ ROL - Rotate Left
    ; ==========================================
    lea rsi, [rel msg_rol]
    call print_cstring
    lea rsi, [rel msg_hex]
    call print_cstring
    
    movzx eax, byte [rel test_val]  ; โหลด 0xB6 = 1011 0110
    rol al, 1                        ; หมุนซ้าย 1 bit
    ; 1011 0110 → 0110 1101 = 0x6D
    call print_byte_hex
    
    ; ==========================================
    ; ทดสอบ ROR - Rotate Right
    ; ==========================================
    lea rsi, [rel msg_ror]
    call print_cstring
    lea rsi, [rel msg_hex]
    call print_cstring
    
    movzx eax, byte [rel test_val]  ; โหลด 0xB6 = 1011 0110
    ror al, 1                        ; หมุนขวา 1 bit
    ; 1011 0110 → 0101 1011 = 0x5B
    call print_byte_hex
    
    ; ==========================================
    ; ทดสอบ RCL - Rotate Left through Carry
    ; ==========================================
    lea rsi, [rel msg_rcl]
    call print_cstring
    lea rsi, [rel msg_hex]
    call print_cstring
    
    movzx eax, byte [rel test_val]  ; โหลด 0xB6
    stc                              ; ตั้ง CF = 1
    rcl al, 1                        ; หมุนซ้ายผ่าน CF
    ; CF=1, 1011 0110 → 0110 1101 (1 เข้า bit0, bit7=1 ออก CF)
    call print_byte_hex
    
    ; ==========================================
    ; ทดสอบ RCR - Rotate Right through Carry
    ; ==========================================
    lea rsi, [rel msg_rcr]
    call print_cstring
    lea rsi, [rel msg_hex]
    call print_cstring
    
    movzx eax, byte [rel test_val]  ; โหลด 0xB6
    clc                              ; เคลียร์ CF = 0
    rcr al, 1                        ; หมุนขวาผ่าน CF
    ; CF=0, 1011 0110 → 0101 1011 (0 เข้า bit7, bit0=0 ออก CF)
    call print_byte_hex
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 rotate_ops.asm -o rotate_ops.o
ld rotate_ops.o -o rotate_ops
./rotate_ops
```

**Expected Output:**
```
=== Rotate Operations Demo ===
ROL result: Hex: 0x6D
ROR result: Hex: 0x5B
RCL result: Hex: 0x6D
RCR result: Hex: 0x5B
```

---

## Code Example 3: Multiplication and Division by Powers of 2

```nasm
; ===================================================
; ไฟล์: power2_math.asm
; คำอธิบาย: การคูณและหารด้วยเลขยกกำลัง 2 ด้วย Shift
; ===================================================
; การ Compile:
;   nasm -f elf64 power2_math.asm -o power2_math.o
;   ld power2_math.o -o power2_math
; การรัน:
;   ./power2_math
; ===================================================

section .data
    ; ข้อความแสดงผล
    title       db "=== Power of 2 Arithmetic via Shifts ===", 10, 0
    lf          db 10, 0
    
    msg_x1      db "x * 1   = ", 0    ; x << 0 (ไม่เปลี่ยน)
    msg_x2      db "x * 2   = ", 0    ; x << 1
    msg_x4      db "x * 4   = ", 0    ; x << 2
    msg_x8      db "x * 8   = ", 0    ; x << 3
    msg_x16     db "x * 16  = ", 0    ; x << 4
    msg_x32     db "x * 32  = ", 0    ; x << 5
    msg_x64     db "x * 64  = ", 0    ; x << 6
    msg_d2      db "x / 2   = ", 0    ; x >> 1
    msg_d4      db "x / 4   = ", 0    ; x >> 2
    msg_d8      db "x / 8   = ", 0    ; x >> 3
    
    ; คำอธิบาย trick สำหรับการคูณด้วยเลขอื่น
    msg_trick   db "=== Multiply Tricks ===", 10, 0
    msg_x3      db "x * 3  (x + x*2) = ", 0
    msg_x5      db "x * 5  (x + x*4) = ", 0
    msg_x6      db "x * 6  (x*2 + x*4) = ", 0
    msg_x7      db "x * 7  (x*8 - x) = ", 0
    msg_x10     db "x * 10 (x*2 + x*8) = ", 0
    
    input_val   dq 100         ; ค่า x = 100

section .bss
    num_buf     resb 32

section .text
    global _start

; Macro สำหรับแสดงผล (ใช้แบบ inline เพื่อความง่าย)
; ======================================
; print_cstring: RSI = null-terminated string
; ======================================
print_cstring:
    push rax
    push rcx
    push rdx
    push rdi
    xor rcx, rcx
.len_loop:
    cmp byte [rsi + rcx], 0
    je .len_done
    inc rcx
    jmp .len_loop
.len_done:
    mov rdx, rcx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rax
    ret

; ======================================
; print_uint64: RAX = ค่าที่แสดง (unsigned)
; ======================================
print_uint64:
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    
    lea rsi, [rel num_buf]
    add rsi, 30                ; ชี้ไปที่ท้าย buffer
    mov byte [rsi], 0          ; null terminator
    mov rcx, 10                ; base 10
    
.convert:
    dec rsi
    xor rdx, rdx
    div rcx                    ; RAX = quotient, RDX = remainder
    add dl, '0'
    mov [rsi], dl
    test rax, rax
    jnz .convert
    
    ; คำนวณ length
    lea rdx, [rel num_buf]
    add rdx, 31
    sub rdx, rsi
    
    mov rax, 1
    mov rdi, 1
    syscall
    
    ; พิมพ์ newline
    push rsi
    lea rsi, [rel lf]
    call print_cstring
    pop rsi
    
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

_start:
    lea rsi, [rel title]
    call print_cstring
    
    ; โหลดค่า x
    mov rbx, [rel input_val]  ; rbx = 100
    
    ; ==========================================
    ; การคูณด้วยเลขยกกำลัง 2
    ; ==========================================
    
    ; x * 1 (ไม่ต้องทำอะไร)
    lea rsi, [rel msg_x1]
    call print_cstring
    mov rax, rbx              ; rax = 100
    call print_uint64
    
    ; x * 2 = SHL 1
    lea rsi, [rel msg_x2]
    call print_cstring
    mov rax, rbx
    shl rax, 1                ; 100 << 1 = 200
    call print_uint64
    
    ; x * 4 = SHL 2
    lea rsi, [rel msg_x4]
    call print_cstring
    mov rax, rbx
    shl rax, 2                ; 100 << 2 = 400
    call print_uint64
    
    ; x * 8 = SHL 3
    lea rsi, [rel msg_x8]
    call print_cstring
    mov rax, rbx
    shl rax, 3                ; 100 << 3 = 800
    call print_uint64
    
    ; x * 16 = SHL 4
    lea rsi, [rel msg_x16]
    call print_cstring
    mov rax, rbx
    shl rax, 4                ; 100 << 4 = 1600
    call print_uint64
    
    ; x * 32 = SHL 5
    lea rsi, [rel msg_x32]
    call print_cstring
    mov rax, rbx
    shl rax, 5                ; 100 << 5 = 3200
    call print_uint64
    
    ; ==========================================
    ; การหารด้วยเลขยกกำลัง 2
    ; ==========================================
    
    ; x / 2 = SHR 1
    lea rsi, [rel msg_d2]
    call print_cstring
    mov rax, rbx
    shr rax, 1                ; 100 >> 1 = 50
    call print_uint64
    
    ; x / 4 = SHR 2
    lea rsi, [rel msg_d4]
    call print_cstring
    mov rax, rbx
    shr rax, 2                ; 100 >> 2 = 25
    call print_uint64
    
    ; x / 8 = SHR 3
    lea rsi, [rel msg_d8]
    call print_cstring
    mov rax, rbx
    shr rax, 3                ; 100 >> 3 = 12 (truncation)
    call print_uint64
    
    ; ==========================================
    ; Multiplication Tricks (คูณด้วยเลขที่ไม่ใช่ 2^n)
    ; ==========================================
    lea rsi, [rel msg_trick]
    call print_cstring
    
    ; x * 3 = x + x*2 = x + (x << 1)
    lea rsi, [rel msg_x3]
    call print_cstring
    mov rax, rbx              ; rax = x = 100
    mov rcx, rbx              ; rcx = x
    shl rcx, 1                ; rcx = x * 2 = 200
    add rax, rcx              ; rax = x + x*2 = 300
    call print_uint64
    
    ; x * 5 = x + x*4 = x + (x << 2)
    lea rsi, [rel msg_x5]
    call print_cstring
    mov rax, rbx              ; rax = x = 100
    lea rax, [rax + rax*4]    ; LEA trick: rax = x + x*4 = 5x = 500
    call print_uint64
    
    ; x * 6 = x*2 + x*4 = (x << 1) + (x << 2)
    lea rsi, [rel msg_x6]
    call print_cstring
    mov rax, rbx
    shl rax, 1                ; rax = x*2 = 200
    mov rcx, rbx
    shl rcx, 2                ; rcx = x*4 = 400
    add rax, rcx              ; rax = x*2 + x*4 = 600
    call print_uint64
    
    ; x * 7 = x*8 - x = (x << 3) - x
    lea rsi, [rel msg_x7]
    call print_cstring
    mov rax, rbx
    shl rax, 3                ; rax = x*8 = 800
    sub rax, rbx              ; rax = x*8 - x = 700
    call print_uint64
    
    ; x * 10 = x*2 + x*8 = (x << 1) + (x << 3)
    lea rsi, [rel msg_x10]
    call print_cstring
    mov rax, rbx
    shl rax, 1                ; rax = x*2 = 200
    mov rcx, rbx
    shl rcx, 3                ; rcx = x*8 = 800
    add rax, rcx              ; rax = x*2 + x*8 = 1000
    call print_uint64
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 power2_math.asm -o power2_math.o
ld power2_math.o -o power2_math
./power2_math
```

**Expected Output:**
```
=== Power of 2 Arithmetic via Shifts ===
x * 1   = 100
x * 2   = 200
x * 4   = 400
x * 8   = 800
x * 16  = 1600
x * 32  = 3200
x / 2   = 50
x / 4   = 25
x / 8   = 12
=== Multiply Tricks ===
x * 3  (x + x*2) = 300
x * 5  (x + x*4) = 500
x * 6  (x*2 + x*4) = 600
x * 7  (x*8 - x) = 700
x * 10 (x*2 + x*8) = 1000
```

---

## Code Example 4: Bit Field Extraction and Packing

```nasm
; ===================================================
; ไฟล์: bitfield.asm
; คำอธิบาย: Bit Field Extraction และ Packing/Unpacking
;           ตัวอย่างเช่น การแยก/รวม RGB color components
; ===================================================
; การ Compile:
;   nasm -f elf64 bitfield.asm -o bitfield.o
;   ld bitfield.o -o bitfield
; การรัน:
;   ./bitfield
; ===================================================

section .data
    title       db "=== Bit Field Operations ===", 10, 0
    lf          db 10, 0
    space       db " ", 0
    
    ; ตัวอย่าง: RGB Color Packing
    ; Format: 0x00RRGGBB (32-bit)
    ; R = bits 23-16, G = bits 15-8, B = bits 7-0
    msg_rgb     db "=== RGB Color (0x00RRGGBB) ===", 10, 0
    msg_color   db "Packed Color: 0x", 0
    msg_red     db "Red   = ", 0
    msg_green   db "Green = ", 0
    msg_blue    db "Blue  = ", 0
    msg_repack  db "Repacked = 0x", 0
    
    ; ตัวอย่าง: IP Address Packing
    msg_ip      db "=== IP Address Packing ===", 10, 0
    msg_packed  db "Packed IP: ", 0
    msg_oct1    db "Octet 1: ", 0
    msg_oct2    db "Octet 2: ", 0
    msg_oct3    db "Octet 3: ", 0
    msg_oct4    db "Octet 4: ", 0
    
    ; ตัวอย่าง: Bit Field Extraction ทั่วไป
    msg_bf      db "=== General Bit Field ===", 10, 0
    msg_word    db "Word value: ", 0
    msg_b0      db "Bits 0-3:   ", 0
    msg_b1      db "Bits 4-7:   ", 0
    msg_b2      db "Bits 8-11:  ", 0
    msg_b3      db "Bits 12-15: ", 0
    
    ; ค่าทดสอบ
    rgb_color   dd 0x00FF8040  ; R=255, G=128, B=64
    ip_packed   dd 0xC0A80101  ; 192.168.1.1
    test_word   dw 0xABCD      ; 1010 1011 1100 1101

section .bss
    hex_str     resb 12
    num_str     resb 20

section .text
    global _start

; ======================================
; print_cstring: RSI = null-terminated
; ======================================
print_cstring:
    push rax
    push rcx
    push rdx
    push rdi
    xor rcx, rcx
.loop:
    cmp byte [rsi + rcx], 0
    je .done
    inc rcx
    jmp .loop
.done:
    mov rdx, rcx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rax
    ret

; ======================================
; print_hex32: EAX = 32-bit value to print as hex
; ======================================
print_hex32:
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    
    lea rsi, [rel hex_str]
    mov rcx, 8                 ; 8 hex digits สำหรับ 32-bit
    
.hex_loop:
    rol eax, 4                 ; หมุน 4 bits เข้ามา
    mov bl, al
    and bl, 0x0F               ; เอาแค่ nibble ล่าง
    cmp bl, 10
    jl .use_digit
    add bl, 'A' - 10
    jmp .store
.use_digit:
    add bl, '0'
.store:
    mov [rsi], bl
    inc rsi
    dec rcx
    jnz .hex_loop
    
    mov byte [rsi], 10         ; newline
    
    lea rsi, [rel hex_str]
    mov rdx, 9                 ; 8 + newline
    mov rax, 1
    mov rdi, 1
    syscall
    
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    ret

; ======================================
; print_uint32: EAX = 32-bit unsigned
; ======================================
print_uint32:
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    
    lea rsi, [rel num_str]
    add rsi, 18
    mov byte [rsi], 0
    mov byte [rsi+1], 10        ; newline
    mov rcx, 10
    
.conv:
    dec rsi
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rsi], dl
    test eax, eax
    jnz .conv
    
    lea rdx, [rel num_str]
    add rdx, 19
    sub rdx, rsi
    
    mov rax, 1
    mov rdi, 1
    syscall
    
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

_start:
    lea rsi, [rel title]
    call print_cstring
    
    ; ==========================================
    ; RGB Color Extraction
    ; ==========================================
    lea rsi, [rel msg_rgb]
    call print_cstring
    
    ; แสดง packed color
    lea rsi, [rel msg_color]
    call print_cstring
    mov eax, [rel rgb_color]   ; โหลด 0x00FF8040
    call print_hex32
    
    ; Extract Red (bits 23-16): SHR 16, AND 0xFF
    lea rsi, [rel msg_red]
    call print_cstring
    mov eax, [rel rgb_color]
    shr eax, 16                ; เลื่อนขวา 16 bits
    and eax, 0xFF              ; เอาแค่ byte ล่าง = R = 255
    call print_uint32
    
    ; Extract Green (bits 15-8): SHR 8, AND 0xFF
    lea rsi, [rel msg_green]
    call print_cstring
    mov eax, [rel rgb_color]
    shr eax, 8                 ; เลื่อนขวา 8 bits
    and eax, 0xFF              ; เอาแค่ byte ล่าง = G = 128
    call print_uint32
    
    ; Extract Blue (bits 7-0): AND 0xFF เลย
    lea rsi, [rel msg_blue]
    call print_cstring
    mov eax, [rel rgb_color]
    and eax, 0xFF              ; เอาแค่ byte ล่าง = B = 64
    call print_uint32
    
    ; ==========================================
    ; Repack RGB (แสดงวิธีสร้าง packed color ใหม่)
    ; ==========================================
    lea rsi, [rel msg_repack]
    call print_cstring
    
    ; สมมติว่า R=255 G=128 B=64
    mov eax, 255               ; R = 255
    shl eax, 8                 ; เลื่อนซ้าย 8 bits ไปตำแหน่ง Green
    or  eax, 128               ; เพิ่ม G = 128
    shl eax, 8                 ; เลื่อนซ้าย 8 bits อีกไปตำแหน่ง Blue
    or  eax, 64                ; เพิ่ม B = 64
    ; ตอนนี้ EAX = 0x00FF8040
    call print_hex32
    
    ; ==========================================
    ; IP Address Extraction
    ; ==========================================
    lea rsi, [rel msg_ip]
    call print_cstring
    
    ; แสดง packed IP
    lea rsi, [rel msg_packed]
    call print_cstring
    mov eax, [rel ip_packed]   ; 0xC0A80101 = 192.168.1.1
    call print_hex32
    
    ; Octet 1 (MSB): SHR 24
    lea rsi, [rel msg_oct1]
    call print_cstring
    mov eax, [rel ip_packed]
    shr eax, 24                ; 0xC0 = 192
    call print_uint32
    
    ; Octet 2: SHR 16, AND 0xFF
    lea rsi, [rel msg_oct2]
    call print_cstring
    mov eax, [rel ip_packed]
    shr eax, 16
    and eax, 0xFF              ; 0xA8 = 168
    call print_uint32
    
    ; Octet 3: SHR 8, AND 0xFF
    lea rsi, [rel msg_oct3]
    call print_cstring
    mov eax, [rel ip_packed]
    shr eax, 8
    and eax, 0xFF              ; 0x01 = 1
    call print_uint32
    
    ; Octet 4 (LSB): AND 0xFF
    lea rsi, [rel msg_oct4]
    call print_cstring
    mov eax, [rel ip_packed]
    and eax, 0xFF              ; 0x01 = 1
    call print_uint32
    
    ; ==========================================
    ; General Bit Field Extraction
    ; ==========================================
    lea rsi, [rel msg_bf]
    call print_cstring
    
    ; แสดง test word
    lea rsi, [rel msg_word]
    call print_cstring
    movzx eax, word [rel test_word]  ; 0xABCD
    call print_hex32
    
    ; Bits 0-3 (nibble 0)
    lea rsi, [rel msg_b0]
    call print_cstring
    movzx eax, word [rel test_word]
    and eax, 0x000F            ; mask bits 0-3 = 0xD = 13
    call print_uint32
    
    ; Bits 4-7 (nibble 1)
    lea rsi, [rel msg_b1]
    call print_cstring
    movzx eax, word [rel test_word]
    shr eax, 4                 ; เลื่อนขวา 4
    and eax, 0x000F            ; mask = 0xC = 12
    call print_uint32
    
    ; Bits 8-11 (nibble 2)
    lea rsi, [rel msg_b2]
    call print_cstring
    movzx eax, word [rel test_word]
    shr eax, 8                 ; เลื่อนขวา 8
    and eax, 0x000F            ; mask = 0xB = 11
    call print_uint32
    
    ; Bits 12-15 (nibble 3)
    lea rsi, [rel msg_b3]
    call print_cstring
    movzx eax, word [rel test_word]
    shr eax, 12                ; เลื่อนขวา 12
    and eax, 0x000F            ; mask = 0xA = 10
    call print_uint32
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 bitfield.asm -o bitfield.o
ld bitfield.o -o bitfield
./bitfield
```

**Expected Output:**
```
=== Bit Field Operations ===
=== RGB Color (0x00RRGGBB) ===
Packed Color: 0x00FF8040
Red   = 255
Green = 128
Blue  = 64
Repacked = 0x00FF8040
=== IP Address Packing ===
Packed IP: C0A80101
Octet 1: 192
Octet 2: 168
Octet 3: 1
Octet 4: 1
=== General Bit Field ===
Word value: 0000ABCD
Bits 0-3:   13
Bits 4-7:   12
Bits 8-11:  11
Bits 12-15: 10
```

---

## Code Example 5: Hex Digit Conversion with Shifts

```nasm
; ===================================================
; ไฟล์: hex_convert.asm
; คำอธิบาย: แปลงค่าตัวเลขเป็น Hex String และกลับกัน
;           โดยใช้ Shift Operations
; ===================================================
; การ Compile:
;   nasm -f elf64 hex_convert.asm -o hex_convert.o
;   ld hex_convert.o -o hex_convert
; การรัน:
;   ./hex_convert
; ===================================================

section .data
    title       db "=== Hex Conversion with Shifts ===", 10, 0
    lf          db 10, 0
    prefix      db "0x", 0
    msg_bin     db "Binary to Hex: ", 0
    msg_dec     db "Decimal to Hex: ", 0
    msg_parse   db "Hex to Decimal: 0xDEAD = ", 0
    msg_nibbles db "Nibbles: ", 0
    colon       db ":", 0
    
    ; Lookup table สำหรับแปลง nibble เป็น hex char
    hex_table   db "0123456789ABCDEF"
    
    test_num    dq 0x1A2B3C4D  ; ตัวเลขทดสอบ

section .bss
    hex_out     resb 20
    result_buf  resb 32

section .text
    global _start

print_cstring:
    push rax
    push rcx
    push rdx
    push rdi
    xor rcx, rcx
.loop:
    cmp byte [rsi + rcx], 0
    je .done
    inc rcx
    jmp .loop
.done:
    mov rdx, rcx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rax
    ret

print_uint64:
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    lea rsi, [rel result_buf]
    add rsi, 30
    mov byte [rsi], 0
    mov byte [rsi+1], 10
    mov rcx, 10
.cv:
    dec rsi
    xor rdx, rdx
    div rcx
    add dl, '0'
    mov [rsi], dl
    test rax, rax
    jnz .cv
    lea rdx, [rel result_buf]
    add rdx, 31
    sub rdx, rsi
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    ret

; ======================================
; num_to_hex64: RAX = number → hex string ที่ RDI
; Output: 16 hex chars + null at RDI
; ======================================
num_to_hex64:
    push rax
    push rbx
    push rcx
    push rsi
    
    lea rsi, [rel hex_table]
    mov rcx, 16                ; 16 nibbles สำหรับ 64-bit
    
.loop:
    rol rax, 4                 ; หมุน 4 bits เข้ามา (MSN ก่อน)
    movzx rbx, al
    and rbx, 0x0F              ; เอาแค่ nibble ล่าง
    mov bl, [rsi + rbx]        ; lookup hex character
    mov [rdi], bl              ; เก็บลงใน output buffer
    inc rdi
    dec rcx
    jnz .loop
    
    mov byte [rdi], 0          ; null terminator
    
    pop rsi
    pop rcx
    pop rbx
    pop rax
    ret

; ======================================
; byte_to_hex: AL = byte → 2 hex chars ที่ RDI
; ======================================
byte_to_hex:
    push rax
    push rbx
    push rsi
    
    lea rsi, [rel hex_table]
    
    ; nibble สูง (bits 7-4)
    mov bl, al
    shr bl, 4                  ; เลื่อนขวา 4 bits
    and bl, 0x0F
    mov bl, [rsi + rbx]
    mov [rdi], bl
    inc rdi
    
    ; nibble ต่ำ (bits 3-0)
    mov bl, al
    and bl, 0x0F               ; mask 4 bits ล่าง
    mov bl, [rsi + rbx]
    mov [rdi], bl
    inc rdi
    
    pop rsi
    pop rbx
    pop rax
    ret

; ======================================
; hex_to_nibble: AL = hex char → nibble ใน AL
; Returns: AL = nibble value (0-15)
;          CF = 1 หากผิดพลาด
; ======================================
hex_to_nibble:
    ; '0'-'9' → 0-9
    cmp al, '0'
    jl .invalid
    cmp al, '9'
    jle .digit
    
    ; 'A'-'F' → 10-15
    cmp al, 'A'
    jl .try_lower
    cmp al, 'F'
    jg .try_lower
    sub al, 'A' - 10
    clc
    ret
    
.try_lower:
    ; 'a'-'f' → 10-15
    cmp al, 'a'
    jl .invalid
    cmp al, 'f'
    jg .invalid
    sub al, 'a' - 10
    clc
    ret
    
.digit:
    sub al, '0'
    clc
    ret
    
.invalid:
    stc                        ; set carry flag = error
    ret

; ======================================
; hex_str_to_uint: RSI = hex string → RAX
; ======================================
hex_str_to_uint:
    push rbx
    push rcx
    push rsi
    
    xor rax, rax               ; ล้างผลลัพธ์
    
.parse_loop:
    movzx rbx, byte [rsi]      ; โหลด char ถัดไป
    test bl, bl
    jz .done                   ; null terminator = จบ
    
    mov cl, bl
    call hex_to_nibble         ; แปลง char ใน AL
    jc .done                   ; ถ้า error ออก
    
    shl rax, 4                 ; เลื่อนผลลัพธ์ซ้าย 4 bits (× 16)
    or rax, rcx                ; เพิ่ม nibble ใหม่
    
    inc rsi
    jmp .parse_loop
    
.done:
    pop rsi
    pop rcx
    pop rbx
    ret

_start:
    lea rsi, [rel title]
    call print_cstring
    
    ; ==========================================
    ; แปลงตัวเลขเป็น Hex String
    ; ==========================================
    lea rsi, [rel msg_dec]
    call print_cstring
    
    lea rsi, [rel prefix]
    call print_cstring
    
    ; แปลง 0x1A2B3C4D เป็น hex string
    mov rax, [rel test_num]    ; โหลดค่าทดสอบ
    lea rdi, [rel hex_out]
    call num_to_hex64
    
    lea rsi, [rel hex_out]
    call print_cstring
    
    lea rsi, [rel lf]
    call print_cstring
    
    ; ==========================================
    ; แสดง Nibble ทีละ nibble
    ; ==========================================
    lea rsi, [rel msg_nibbles]
    call print_cstring
    
    mov rax, [rel test_num]    ; 0x1A2B3C4D
    lea rsi, [rel hex_table]
    
    ; nibble 7 (MSN) = 0x0 = 0
    mov rbx, rax
    shr rbx, 28                ; เลื่อนขวา 28 bits
    and rbx, 0x0F              ; mask
    mov bl, [rsi + rbx]
    mov [rel hex_out], bl
    mov byte [rel hex_out+1], ' '
    mov byte [rel hex_out+2], 0
    lea rsi, [rel hex_out]
    call print_cstring
    
    ; ==========================================
    ; Parse Hex String กลับเป็นตัวเลข
    ; ==========================================
    lea rsi, [rel msg_parse]
    call print_cstring
    
    ; สร้าง hex string "DEAD" ใน hex_out
    mov byte [rel hex_out+0], 'D'
    mov byte [rel hex_out+1], 'E'
    mov byte [rel hex_out+2], 'A'
    mov byte [rel hex_out+3], 'D'
    mov byte [rel hex_out+4], 0
    
    lea rsi, [rel hex_out]
    call hex_str_to_uint       ; แปลง "DEAD" → RAX = 57005
    call print_uint64
    
    ; ==========================================
    ; Fast Modulo ด้วย Shift
    ; ==========================================
    lea rsi, [rel lf]
    call print_cstring
    
    ; x mod 2^n = x AND (2^n - 1)
    ; ตัวอย่าง: 100 mod 16 = 100 AND 15
    ; (แทน DIV instruction ที่ช้ากว่า)
    mov rax, 100               ; x = 100
    and rax, 15                ; x mod 16 = 4 (เพราะ 100 = 6*16 + 4)
    call print_uint64          ; ควรได้ 4
    
    ; 255 mod 32 = 255 AND 31
    mov rax, 255               ; x = 255
    and rax, 31                ; x mod 32 = 31 (255 = 7*32 + 31)
    call print_uint64          ; ควรได้ 31
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 hex_convert.asm -o hex_convert.o
ld hex_convert.o -o hex_convert
./hex_convert
```

**Expected Output:**
```
=== Hex Conversion with Shifts ===
Decimal to Hex: 0x000000001A2B3C4D
Nibbles: 0 
Hex to Decimal: 0xDEAD = 57005
4
31
```

---

## Code Example 6: CRC-8 Calculation with Shifts

```nasm
; ===================================================
; ไฟล์: crc8.asm
; คำอธิบาย: การคำนวณ CRC-8 โดยใช้ Shift Operations
;           CRC-8 polynomial: 0x07 (x^8 + x^2 + x + 1)
; ===================================================
; การ Compile:
;   nasm -f elf64 crc8.asm -o crc8.o
;   ld crc8.o -o crc8
; การรัน:
;   ./crc8
; ===================================================

section .data
    title       db "=== CRC-8 Calculation ===", 10, 0
    lf          db 10, 0
    msg_data    db "Data: Hello", 10, 0
    msg_crc     db "CRC-8 = 0x", 0
    
    ; ข้อมูลทดสอบ
    test_data   db "Hello", 0
    test_len    equ $ - test_data - 1  ; ความยาวของ "Hello"
    
    ; CRC-8 polynomial = 0x07
    crc_poly    equ 0x07

section .bss
    hex_buf     resb 8

section .text
    global _start

print_cstring:
    push rax
    push rcx
    push rdx
    push rdi
    xor rcx, rcx
.loop:
    cmp byte [rsi + rcx], 0
    je .done
    inc rcx
    jmp .loop
.done:
    mov rdx, rcx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rax
    ret

; ======================================
; Subroutine: crc8_update
; Input:  AL = current CRC value
;         BL = new data byte
; Output: AL = updated CRC
; ======================================
crc8_update:
    push rcx
    
    xor al, bl                 ; XOR data byte เข้า CRC
    mov rcx, 8                 ; วนลูป 8 ครั้ง (1 ครั้งต่อ 1 bit)
    
.bit_loop:
    ; ตรวจสอบ MSB ของ CRC
    test al, 0x80              ; ตรวจสอบ bit 7
    jz .no_xor                 ; ถ้า bit 7 = 0 ไม่ต้อง XOR polynomial
    
    shl al, 1                  ; เลื่อนซ้าย 1 bit
    xor al, crc_poly           ; XOR กับ polynomial
    jmp .next_bit
    
.no_xor:
    shl al, 1                  ; เลื่อนซ้าย 1 bit เท่านั้น
    
.next_bit:
    dec rcx
    jnz .bit_loop
    
    pop rcx
    ret

; ======================================
; Subroutine: crc8_compute
; Input:  RSI = data buffer, RCX = length
; Output: AL = CRC-8 value
; ======================================
crc8_compute:
    push rbx
    push rcx
    push rsi
    
    xor al, al                 ; เริ่มต้น CRC = 0
    
.data_loop:
    test rcx, rcx              ; ตรวจสอบว่า length = 0 หรือยัง
    jz .done
    
    movzx ebx, byte [rsi]      ; โหลด data byte
    call crc8_update           ; อัปเดต CRC
    
    inc rsi                    ; ไปยัง byte ถัดไป
    dec rcx                    ; ลด counter
    jmp .data_loop
    
.done:
    pop rsi
    pop rcx
    pop rbx
    ret

; ======================================
; print_byte_hex: AL = byte to print
; ======================================
print_byte_hex:
    push rax
    push rbx
    push rsi
    push rdx
    push rdi
    
    lea rsi, [rel hex_buf]
    
    ; nibble สูง
    mov bl, al
    shr bl, 4
    and bl, 0x0F
    cmp bl, 10
    jl .d1
    add bl, 'A' - 10
    jmp .s1
.d1:
    add bl, '0'
.s1:
    mov [rsi], bl
    
    ; nibble ต่ำ
    mov bl, al
    and bl, 0x0F
    cmp bl, 10
    jl .d2
    add bl, 'A' - 10
    jmp .s2
.d2:
    add bl, '0'
.s2:
    mov [rsi+1], bl
    mov byte [rsi+2], 10
    
    mov rdx, 3
    mov rax, 1
    mov rdi, 1
    syscall
    
    pop rdi
    pop rdx
    pop rsi
    pop rbx
    pop rax
    ret

_start:
    lea rsi, [rel title]
    call print_cstring
    
    ; แสดงข้อมูล
    lea rsi, [rel msg_data]
    call print_cstring
    
    ; คำนวณ CRC-8
    lea rsi, [rel test_data]
    mov rcx, test_len          ; ความยาว "Hello" = 5
    call crc8_compute          ; ผลลัพธ์ใน AL
    
    ; แสดงผล CRC
    lea rsi, [rel msg_crc]
    call print_cstring
    call print_byte_hex        ; แสดง CRC เป็น hex
    
    ; ==========================================
    ; ตัวอย่าง: CRC ของ single byte
    ; ==========================================
    lea rsi, [rel lf]
    call print_cstring
    
    ; CRC ของ byte 0xFF
    xor al, al                 ; เริ่ม CRC = 0
    mov bl, 0xFF               ; data = 0xFF
    call crc8_update
    
    lea rsi, [rel msg_crc]
    call print_cstring
    call print_byte_hex
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 crc8.asm -o crc8.o
ld crc8.o -o crc8
./crc8
```

**Expected Output:**
```
=== CRC-8 Calculation ===
Data: Hello
CRC-8 = 0x4A
(ตัวเลขอาจแตกต่างขึ้นกับ polynomial และ initial value)
```

---

## Code Example 7: Double Precision Shift (SHLD/SHRD)

```nasm
; ===================================================
; ไฟล์: double_shift.asm
; คำอธิบาย: การใช้ SHLD/SHRD สำหรับ 128-bit operations
; ===================================================
; การ Compile:
;   nasm -f elf64 double_shift.asm -o double_shift.o
;   ld double_shift.o -o double_shift
; การรัน:
;   ./double_shift
; ===================================================

section .data
    title       db "=== Double Precision Shift ===", 10, 0
    lf          db 10, 0
    msg_shld    db "SHLD result (high 64 bits): ", 0
    msg_shrd    db "SHRD result (low 64 bits):  ", 0
    msg_128hi   db "128-bit shift high: ", 0
    msg_128lo   db "128-bit shift low:  ", 0

section .bss
    hex_str     resb 20

section .text
    global _start

print_cstring:
    push rax
    push rcx
    push rdx
    push rdi
    xor rcx, rcx
.loop:
    cmp byte [rsi + rcx], 0
    je .done
    inc rcx
    jmp .loop
.done:
    mov rdx, rcx
    mov rax, 1
    mov rdi, 1
    syscall
    pop rdi
    pop rdx
    pop rcx
    pop rax
    ret

; print_hex64: RAX = 64-bit value
print_hex64:
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    
    lea rdi, [rel hex_str]
    mov rcx, 16
    lea rsi, [rel .hex_chars]
    
.loop:
    rol rax, 4
    movzx rbx, al
    and rbx, 0x0F
    mov bl, [rsi + rbx]
    mov [rdi], bl
    inc rdi
    dec rcx
    jnz .loop
    
    mov byte [rdi], 10
    
    lea rsi, [rel hex_str]
    mov rdx, 17
    mov rax, 1
    mov rdi, 1
    syscall
    
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    ret

.hex_chars db "0123456789ABCDEF"

_start:
    lea rsi, [rel title]
    call print_cstring
    
    ; ==========================================
    ; ทดสอบ SHLD - Shift Left Double
    ; ==========================================
    lea rsi, [rel msg_shld]
    call print_cstring
    
    ; RAX = 0x0000000000000001 (high part)
    ; RBX = 0x8000000000000000 (low part)
    ; SHLD RAX, RBX, 1 : เลื่อน RAX ซ้าย 1 bit + bit MSB ของ RBX เข้า LSB ของ RAX
    mov rax, 0x0000000000000001  ; ค่า destination
    mov rbx, 0x8000000000000000  ; ค่า source (bit 63 = 1)
    shld rax, rbx, 1             ; RAX เลื่อนซ้าย 1 + MSB ของ RBX เข้า
    ; ผล: RAX = 0x0000000000000003 (0...011)
    call print_hex64
    
    ; ==========================================
    ; ทดสอบ SHRD - Shift Right Double
    ; ==========================================
    lea rsi, [rel msg_shrd]
    call print_cstring
    
    ; SHRD RAX, RBX, 1 : เลื่อน RAX ขวา 1 bit + bit LSB ของ RBX เข้า MSB ของ RAX
    mov rax, 0x8000000000000000  ; ค่า destination
    mov rbx, 0x0000000000000001  ; ค่า source (bit 0 = 1)
    shrd rax, rbx, 1             ; RAX เลื่อนขวา 1 + LSB ของ RBX เข้า MSB
    ; ผล: RAX = 0xC000000000000000 (1100...0)
    call print_hex64
    
    ; ==========================================
    ; 128-bit Left Shift โดยใช้ SHLD
    ; ==========================================
    ; สมมติว่า RDX:RAX เป็น 128-bit number
    ; เราต้องการเลื่อนซ้าย 4 bits
    lea rsi, [rel msg_128hi]
    call print_cstring
    
    mov rax, 0x0123456789ABCDEF  ; low 64 bits
    mov rdx, 0x0000000000000000  ; high 64 bits
    
    ; เลื่อน RDX:RAX ซ้าย 4 bits
    shld rdx, rax, 4             ; เลื่อน RDX ซ้าย + 4 MSBs จาก RAX เข้า LSB ของ RDX
    shl rax, 4                   ; เลื่อน RAX ซ้าย
    
    mov rax, rdx                 ; แสดง high part
    call print_hex64
    
    lea rsi, [rel msg_128lo]
    call print_cstring
    mov rax, 0x0123456789ABCDEF0  ; แสดง low part (หลัง shift)
    call print_hex64
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

**การ Compile และรัน:**
```bash
nasm -f elf64 double_shift.asm -o double_shift.o
ld double_shift.o -o double_shift
./double_shift
```

**Expected Output:**
```
=== Double Precision Shift ===
SHLD result (high 64 bits): 0000000000000003
SHRD result (low 64 bits):  C000000000000000
128-bit shift high: 0000000001234567
128-bit shift low:  0123456789ABCDEF0
```

---

## Common Mistakes and Pitfalls

### 1. ใช้ SHR แทน SAR กับ Signed Numbers

```nasm
; ผิด: ใช้ SHR กับค่าลบ (ผลผิด)
mov rax, -100          ; -100 ใน 64-bit = 0xFFFFFFFFFFFFFF9C
shr rax, 1             ; ผล: 0x7FFFFFFFFFFFFFFFCE (ค่าบวกขนาดใหญ่!)

; ถูก: ใช้ SAR กับค่าลบ (รักษา sign)
mov rax, -100
sar rax, 1             ; ผล: -50 (ถูกต้อง!)
```

### 2. Shift Count ต้องไม่เกิน 31 (32-bit) หรือ 63 (64-bit)

```nasm
; ผิด: shift count > 63 สำหรับ 64-bit
; สิ่งที่เกิดขึ้น: hardware mask count เป็น count & 0x3F
mov cl, 64
shl rax, cl            ; เหมือนกับ SHL RAX, 0 (ไม่เลื่อนเลย!)

; ถูก: ตรวจสอบ count ก่อนเสมอ
mov cl, 1
shl rax, cl
```

### 3. Shift ด้วยค่า 0 vs ไม่ shift

```nasm
; SHL RAX, 0 จะไม่ส่งผลต่อค่าใน RAX
; แต่จะ update flags บางตัว (OF อาจเปลี่ยน)
; Avoid shifting by 0 ถ้าไม่ต้องการ
```

### 4. ROL/ROR กับค่า > 8 สำหรับ byte

```nasm
; ระวัง: ROL AL, 9 เหมือนกับ ROL AL, 1
; เพราะ 9 mod 8 = 1 (hardware mask)
```

### 5. ลืม Clear Register ก่อน Shift

```nasm
; อาจมีปัญหาถ้าเก่า bits ยังอยู่
; ผิด:
mov ah, 0xFF
shr ax, 8              ; AX = 0xFF** (** = ah เดิม)

; ถูก:
xor ax, ax
mov al, 0xFF
shr ax, 8              ; AX = 0x00FF → SHR → 0x0000FF >> 8 = 0
```

### 6. Overflow Flag ใน SAL/SHR

```nasm
; OF (Overflow Flag) ถูก define เฉพาะสำหรับ shift count = 1
; สำหรับ count > 1, OF = undefined (อย่าใช้)
mov al, 0x80
sar al, 1              ; CF = 0, OF = 1 (sign changed)
```

### 7. ลืมว่า CL register ใช้เป็น shift count

```nasm
; CL ใช้เป็น shift count variable
mov cl, 5
shl rax, cl            ; ใช้ CL เป็น count (ถูก)

; แต่ถ้าคุณใช้ CL สำหรับอย่างอื่นด้วย:
; ต้องระวัง! save/restore CL
push rcx
; ... ใช้ RCX สำหรับ loop ...
pop rcx
shl rax, cl
```

---

## Advanced Techniques

### Technique 1: Fast Division with Round Toward Negative Infinity

สำหรับ signed division ด้วย 2^n ให้ผลที่แตกต่างจาก SAR เล็กน้อย
เมื่อค่าลบ SAR ปัดขึ้นแต่ mathematical floor รอบลง

```nasm
; Fast floor division ด้วย 2^n สำหรับ signed:
; result = (x + (x >> 63) & (2^n - 1)) >> n
;          ^^^^^^^^^^^^^^^^^^^^^^^^^^^
;          correction term สำหรับค่าลบ

; ตัวอย่าง: x / 4 (arithmetic floor)
mov rax, -7            ; -7 / 4 = -1.75 → floor = -2

; วิธีที่ 1: ใช้ correction
mov rcx, rax
sar rcx, 63            ; rcx = -1 ถ้า x < 0, 0 ถ้า x >= 0
and rcx, 3             ; rcx = 3 ถ้า x < 0, 0 ถ้า x >= 0
add rax, rcx           ; เพิ่ม correction
sar rax, 2             ; หาร 4 = SHR 2
; ผล: -2 (correct floor division)
```

### Technique 2: Sign Extension โดยใช้ SAR

```nasm
; ขยาย 8-bit signed value เป็น 64-bit
movsx rax, byte [mem]  ; Intel instruction ที่ถูกต้อง

; แต่ถ้าไม่มี MOVSX:
movzx rax, byte [mem]  ; zero extend ก่อน
shl rax, 56            ; เลื่อน bit ไปที่ MSB
sar rax, 56            ; SAR 56 ครั้ง ขยาย sign กลับมา
```

### Technique 3: Isolate Lowest Set Bit

```nasm
; x & (-x) ให้ lowest set bit
; เปรียบเทียบ: ใช้กับ BSF (Bit Scan Forward)
mov rax, 0b1011_0100   ; = 0x84
mov rbx, rax
neg rbx                ; -rax (two's complement)
and rax, rbx           ; rax = lowest set bit only
; ผล: 0b0000_0100 = 4
```

### Technique 4: Popcount ด้วย Shifts

```nasm
; นับจำนวน bits ที่เป็น 1 (Hamming Weight)
; POPCNT instruction ดีกว่า แต่ถ้าต้องทำ manual:

count_ones:
    ; Input: RAX = value
    ; Output: RAX = count of 1 bits
    xor rcx, rcx           ; counter = 0
.loop:
    test rax, rax
    jz .done
    
    mov rbx, rax
    and rbx, 1             ; เช็ค LSB
    add rcx, rbx           ; เพิ่ม counter ถ้า LSB = 1
    
    shr rax, 1             ; เลื่อนขวา 1 bit
    jmp .loop
.done:
    mov rax, rcx
    ret
```

### Technique 5: Round Up to Next Power of 2

```nasm
; หา power of 2 ที่ >= n
; ใช้ BSR + SHL
next_power_of_2:
    ; Input: RAX = n (สมมติ n > 0)
    ; Output: RAX = smallest power of 2 >= n
    
    dec rax                ; n - 1 (เผื่อ n เป็น power of 2 อยู่แล้ว)
    bsr rcx, rax           ; หาตำแหน่ง bit สูงสุด
    mov rax, 1
    shl rax, cl            ; 2^(bit position + 1)
    shl rax, 1
    ret
```

### Technique 6: Barrel Shifter Emulation

```nasm
; Shift ที่ count อาจเป็น 0-63 (variable)
; แต่ต้องการ result แบบ circular/barrel

barrel_rotate_left:
    ; Input: RAX = value, RCX = count (0-63)
    ; Output: RAX = rotated value
    
    and rcx, 63            ; mask count เป็น 0-63
    rol rax, cl            ; ROL รองรับ CL ได้ตรงๆ
    ret
```

### Technique 7: Extracting Bit Range (BEXTR เหมือนกัน)

```nasm
; Extract bits [start, start+length-1] จาก value
; (BEXTR instruction ทำสิ่งเดียวกัน)

extract_bits:
    ; Input: RAX = value, RBX = start bit, RCX = length
    ; Output: RAX = extracted field
    
    shr rax, bl            ; เลื่อนขวา start bits
    mov rdx, 1
    shl rdx, cl            ; 2^length
    dec rdx                ; 2^length - 1 = mask
    and rax, rdx           ; mask ออก
    ret
```

---

## Exercises

### Exercise 1: Nibble Swap
**โจทย์**: เขียนฟังก์ชัน swap_nibbles ที่รับ byte ใน AL
และสลับ nibble สูงกับ nibble ต่ำ
ตัวอย่าง: 0xAB → 0xBA

**Hint**: ใช้ ROL หรือ ROR 4, หรือ SHL/SHR + OR

```nasm
; Template:
; Input: AL = byte
; Output: AL = byte with nibbles swapped
swap_nibbles:
    ; TODO: เขียน code ที่นี่
    ; Hint: ROL AL, 4 จะสลับ nibbles ได้โดยตรง
    ret
```

**เฉลย:**
```nasm
swap_nibbles:
    rol al, 4    ; หมุนซ้าย 4 bits = swap nibbles
    ret
; หรือ:
; ror al, 4    ; หมุนขวา 4 bits ก็ได้ผลเดียวกัน
```

---

### Exercise 2: Bit Reverse
**โจทย์**: เขียนฟังก์ชัน reverse_bits ที่รับค่า 8-bit ใน AL
และกลับลำดับ bits ทั้งหมด
ตัวอย่าง: 0b10110001 → 0b10001101

**Hint**: ใช้ loop วน 8 ครั้ง ดึง bit ออกทีละ bit แล้วใส่กลับ

```nasm
; Template:
; Input: AL = byte
; Output: AL = bit-reversed byte
reverse_bits:
    push rcx
    push rbx
    
    mov rcx, 8     ; วนลูป 8 ครั้ง
    xor bl, bl     ; BL = ผลลัพธ์
    
.loop:
    ; TODO: ดึง bit จาก AL แล้วใส่ใน BL
    ; Hint: ใช้ SHL AL, 1 เพื่อดึง bit เข้า CF
    ;       จากนั้น RCL BL, 1 เพื่อหมุน CF เข้า BL
    
    dec rcx
    jnz .loop
    
    mov al, bl
    pop rbx
    pop rcx
    ret
```

**เฉลย:**
```nasm
reverse_bits:
    push rcx
    push rbx
    
    mov rcx, 8
    xor bl, bl
    
.loop:
    shl al, 1      ; MSB ของ AL เข้า CF
    rcl bl, 1      ; CF เข้า LSB ของ BL (reverse order)
    dec rcx
    jnz .loop
    
    mov al, bl
    pop rbx
    pop rcx
    ret
```

---

### Exercise 3: Fast Modulo
**โจทย์**: เขียนโปรแกรมที่คำนวณ n mod 256, n mod 512 และ n mod 1024
สำหรับ n = 1000 โดยใช้ AND แทน DIV

**Hint**: n mod 2^k = n AND (2^k - 1)
- 256 = 2^8, mask = 0xFF
- 512 = 2^9, mask = 0x1FF
- 1024 = 2^10, mask = 0x3FF

```nasm
; ตัวอย่าง structure:
section .data
    n   dq 1000

section .text
_start:
    mov rax, [n]
    ; mod 256
    mov rbx, rax
    and rbx, 0xFF        ; 1000 mod 256 = 232
    
    ; mod 512
    mov rcx, rax
    and rcx, 0x1FF       ; 1000 mod 512 = 488
    
    ; mod 1024
    and rax, 0x3FF       ; 1000 mod 1024 = 1000 (1000 < 1024)
    
    ; TODO: แสดงผล rbx, rcx, rax
```

---

### Exercise 4: Pack/Unpack Date
**โจทย์**: เขียนโปรแกรม pack/unpack วันที่ใน format:
- Bits 15-9: Year (0-127, ปีนับจาก 2000)
- Bits 8-5: Month (1-12)
- Bits 4-0: Day (1-31)

ตัวอย่าง: 2024-10-15 → pack → unpack → 2024-10-15

```nasm
; Pack: ปี 24 (2024-2000), เดือน 10, วัน 15
; Year  = 24  = 0b0011000
; Month = 10  = 0b01010
; Day   = 15  = 0b01111
; Packed: 0b 0011000 01010 01111 = 0x306F

pack_date:
    ; Input: AL=day, AH=month, BX=year_offset
    ; Output: CX = packed date
    xor ecx, ecx
    
    mov cx, bx       ; year offset (bits 15-9)
    shl cx, 4        ; เลื่อนซ้าย 9 bits... (จริงๆ ต้อง shl 9)
    shl cx, 5
    or  cl, ah       ; เพิ่ม month
    shl cx, 5        ; เลื่อนซ้าย 5 bits
    or  cl, al       ; เพิ่ม day
    ret

unpack_date:
    ; Input: CX = packed date
    ; Output: AL=day, AH=month, BX=year_offset
    mov ax, cx
    and al, 0x1F     ; day = bits 4-0
    mov dl, al       ; save day
    
    shr cx, 5
    mov ah, cl
    and ah, 0x0F     ; month = bits 8-5
    
    shr cx, 4
    mov bx, cx       ; year = bits 15-9
    and bx, 0x7F
    
    mov al, dl       ; restore day
    ret
```

---

### Exercise 5: Counting Bits ด้วย Shift
**โจทย์**: เขียนฟังก์ชัน count_ones ที่นับจำนวน bits ที่เป็น 1
ใน register 64-bit โดยใช้ loop + SHR

ต้องการ: รับ input ใน RDI, ส่งผลลัพธ์กลับใน RAX

```nasm
; เฉลย:
count_ones:
    xor rax, rax       ; count = 0
    
.loop:
    test rdi, rdi      ; ตรวจสอบว่าค่า = 0 หรือยัง
    jz .done
    
    ; เช็ค LSB
    mov rcx, rdi
    and rcx, 1         ; เอา LSB
    add rax, rcx       ; เพิ่ม count
    
    shr rdi, 1         ; เลื่อนขวา 1 bit
    jmp .loop
    
.done:
    ret

; ตัวอย่างการเรียก:
;   mov rdi, 0b1011_0101_1100_1110  ; 8 ones
;   call count_ones
;   ; RAX = 8
```

---

## Summary

### สรุปคำสั่ง Shift และ Rotate

#### Shift Operations
| คำสั่ง | ตัวอย่าง | ผล | Flags |
|--------|----------|-----|-------|
| SHL dest, n | SHL AL, 2 | dest << n, fill 0 right | CF=last bit out, OF(n=1) |
| SAL dest, n | SAL EAX, 1 | same as SHL | same as SHL |
| SHR dest, n | SHR AX, 3 | dest >> n, fill 0 left | CF=last bit out, OF(n=1) |
| SAR dest, n | SAR RAX, 1 | dest >> n, fill sign bit | CF=last bit out, OF=0(n=1) |

#### Rotate Operations
| คำสั่ง | ตัวอย่าง | ผล | Flags |
|--------|----------|-----|-------|
| ROL dest, n | ROL AL, 1 | rotate left, MSB wraps to LSB and CF | CF=new LSB |
| ROR dest, n | ROR BX, 4 | rotate right, LSB wraps to MSB and CF | CF=new MSB |
| RCL dest, n | RCL DL, 1 | rotate left through CF (n+1 bits) | CF changes |
| RCR dest, n | RCR EBX, 2 | rotate right through CF (n+1 bits) | CF changes |

#### Double Shift
| คำสั่ง | ตัวอย่าง | ผล |
|--------|----------|----|
| SHLD dest, src, n | SHLD EAX, EBX, 4 | dest << n, fill from MSB of src |
| SHRD dest, src, n | SHRD EAX, EBX, 4 | dest >> n, fill from LSB of src |

### Use Cases สำคัญ

1. **Multiplication by 2^n**: `SHL reg, n` (เร็วกว่า MUL)
2. **Division by 2^n** (unsigned): `SHR reg, n`
3. **Division by 2^n** (signed): `SAR reg, n`
4. **Modulo 2^n**: `AND reg, (2^n - 1)`
5. **Bit Field Extract**: `SHR` + `AND mask`
6. **Bit Field Insert**: `AND` (clear field) + `SHL` + `OR`
7. **Byte Swap in Word**: `ROL reg16, 8` หรือ `XCHG AL, AH`
8. **Nibble Swap in Byte**: `ROL reg8, 4`
9. **Carry Chain**: `RCL`/`RCR` สำหรับ multi-precision arithmetic
10. **128-bit Shift**: `SHLD`/`SHRD` + `SHL`/`SHR`

### Performance Tips

```
; ลำดับความเร็ว (เร็วสุดไปช้าสุด):
; 1. SHL/SHR/SAR ด้วย immediate (1 clock)
; 2. SHL/SHR/SAR ด้วย CL (1 clock บน modern CPU)
; 3. IMUL สำหรับการคูณที่ไม่ใช่ power of 2 (3 clocks)
; 4. DIV สำหรับการหาร (20-30+ clocks)

; ใช้ SHL แทน MUL เสมอเมื่อ multiplier เป็น power of 2
; ใช้ AND แทน DIV/MOD เสมอเมื่อ divisor เป็น power of 2
```

### Signed vs Unsigned Summary

```
สำหรับ Shift:
- ค่า unsigned → SHR (logical right shift)
- ค่า signed  → SAR (arithmetic right shift)
- ค่า left shift → SHL (หรือ SAL, same encoding)

สำหรับ Division trick:
- ค่า unsigned: x / 4 = SHR x, 2
- ค่า signed:   x / 4 = SAR x, 2  (อาจปัดผิดสำหรับค่าลบ)
- Correct signed floor:
  x / 2^n = (x + (x<0 ? (2^n-1) : 0)) SAR n
```

---

## Part 016 Complete - Next Steps

Part ถัดไปที่แนะนำ:
- **Part 017**: String Operations (MOVS, STOS, LODS, SCAS, CMPS)
- **Part 018**: Procedures และ Stack Management
- **Part 019**: Calling Conventions (Linux x86-64 ABI)
- **Part 020**: File I/O System Calls

การ Shift และ Rotate Operations เป็นพื้นฐานสำคัญที่ใช้ใน:
- Compression algorithms
- Cryptography (AES, SHA ใช้ Rotate มาก)
- Network packet parsing
- Graphics/image processing
- Embedded systems/driver development
- Compiler optimization

ฝึกฝนจนชำนาญก่อนไปขั้นถัดไป!
