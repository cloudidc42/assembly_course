# Part 020: String Instructions (REP prefix)

## Prerequisites and Learning Objectives

### Prerequisites (สิ่งที่ต้องรู้ก่อน)
- ความเข้าใจพื้นฐานเกี่ยวกับ registers (EAX, EBX, ECX, EDX, ESI, EDI)
- การทำงานกับ memory addressing
- Loop instructions (LOOP, JMP, conditional jumps)
- Stack operations (PUSH, POP)
- ผ่าน Part 001-019 มาแล้ว

### Learning Objectives (จุดประสงค์การเรียนรู้)

หลังจากจบ Part นี้ นักเรียนจะสามารถ:
1. เข้าใจ Direction Flag (DF) และการควบคุมทิศทางการประมวลผล string
2. ใช้คำสั่ง MOVS, CMPS, SCAS, LODS, STOS ได้อย่างถูกต้อง
3. ใช้ prefix REP, REPE/REPZ, REPNE/REPNZ เพื่อทำ loop อัตโนมัติ
4. เขียน string functions พื้นฐาน: strcpy, strlen, strcmp, memset, memcmp, memchr
5. เปรียบเทียบประสิทธิภาพระหว่าง string instructions กับ loop แบบ manual

---

## Theory (ทฤษฎีและแนวคิด)

### 1. String Instructions คืออะไร?

String Instructions เป็นคำสั่ง Assembly พิเศษที่ออกแบบมาเพื่อจัดการกับข้อมูลขนาดใหญ่ใน memory อย่างมีประสิทธิภาพ คำสั่งเหล่านี้ทำงานร่วมกับ registers ที่กำหนดไว้โดยเฉพาะ:

```
Registers ที่ใช้กับ String Instructions:
┌─────────────┬──────────────────────────────────────────────┐
│   Register  │              หน้าที่                          │
├─────────────┼──────────────────────────────────────────────┤
│ ESI (SI)    │ Source Index - ชี้ไปที่ข้อมูลต้นทาง           │
│ EDI (DI)    │ Destination Index - ชี้ไปที่ข้อมูลปลายทาง    │
│ ECX (CX)    │ Counter - นับจำนวนครั้งที่ทำซ้ำ              │
│ EAX (AL/AX)│ Accumulator - ใช้กับ LODS และ STOS            │
│ FLAGS (DF)  │ Direction Flag - กำหนดทิศทางการเดิน           │
└─────────────┴──────────────────────────────────────────────┘
```

หลักการทำงาน:
- แต่ละคำสั่งทำการ **อ่านหรือเขียน** ข้อมูล 1 หน่วย (byte/word/dword/qword)
- หลังจากดำเนินการ จะ **อัปเดต ESI และ/หรือ EDI** โดยอัตโนมัติ
- ทิศทางการอัปเดตควบคุมด้วย **Direction Flag (DF)**

---

### 2. Direction Flag (DF) - CLD และ STD

Direction Flag คือ bit ใน FLAGS register ที่ควบคุมทิศทางการเดินของ ESI และ EDI

```
┌─────────────────────────────────────────────────────────────┐
│                    Direction Flag                           │
├─────────────────────────────────────────────────────────────┤
│  DF = 0 (CLD - Clear Direction Flag)                       │
│  → ESI และ EDI จะ INCREMENT (บวก) ทุกครั้งที่ดำเนินการ    │
│  → ประมวลผลจากต้นไปปลาย (forward direction)               │
│                                                             │
│  DF = 1 (STD - Set Direction Flag)                         │
│  → ESI และ EDI จะ DECREMENT (ลบ) ทุกครั้งที่ดำเนินการ    │
│  → ประมวลผลจากปลายไปต้น (backward direction)              │
└─────────────────────────────────────────────────────────────┘
```

จำนวน bytes ที่ ESI/EDI เปลี่ยนแปลง:
```
MOVSB/STOSB/LODSB/SCASB/CMPSB  → ±1 byte
MOVSW/STOSW/LODSW/SCASW/CMPSW  → ±2 bytes  
MOVSD/STOSD/LODSD/SCASD/CMPSD  → ±4 bytes
MOVSQ/STOSQ/LODSQ/SCASQ/CMPSQ  → ±8 bytes (64-bit mode)
```

**หลักสำคัญ**: ในกรณีปกติ ให้ใช้ `CLD` ก่อนเสมอ เพื่อให้แน่ใจว่า DF = 0

---

### 3. MOVS - Memory Copy Instructions

MOVS ย้ายข้อมูลจาก `[ESI]` ไปยัง `[EDI]` แล้วอัปเดต ESI และ EDI

```
MOVSB - ย้าย 1 byte  จาก [ESI] ไป [EDI]
MOVSW - ย้าย 2 bytes จาก [ESI] ไป [EDI]
MOVSD - ย้าย 4 bytes จาก [ESI] ไป [EDI]
MOVSQ - ย้าย 8 bytes จาก [ESI] ไป [EDI] (64-bit เท่านั้น)
```

การทำงาน:
```
; MOVSB ทำงานเทียบเท่ากับ:
mov al, [esi]    ; อ่าน 1 byte จาก source
mov [edi], al    ; เขียน 1 byte ไปยัง destination
inc esi          ; (ถ้า DF=0) หรือ dec esi (ถ้า DF=1)
inc edi          ; (ถ้า DF=0) หรือ dec edi (ถ้า DF=1)
```

---

### 4. CMPS - Memory Compare Instructions

CMPS เปรียบเทียบข้อมูลที่ `[ESI]` กับ `[EDI]` แล้วตั้งค่า FLAGS โดยไม่เปลี่ยนแปลงข้อมูล

```
CMPSB - เปรียบเทียบ 1 byte  ที่ [ESI] กับ [EDI]
CMPSW - เปรียบเทียบ 2 bytes ที่ [ESI] กับ [EDI]
CMPSD - เปรียบเทียบ 4 bytes ที่ [ESI] กับ [EDI]
CMPSQ - เปรียบเทียบ 8 bytes ที่ [ESI] กับ [EDI] (64-bit)
```

การทำงาน:
```
; CMPSB ทำงานเทียบเท่ากับ:
mov al, [esi]    ; อ่าน byte จาก source
cmp al, [edi]    ; เปรียบเทียบกับ destination (ตั้ง FLAGS)
inc esi          ; อัปเดต ESI
inc edi          ; อัปเดต EDI
```

---

### 5. SCAS - Scan Memory Instructions

SCAS เปรียบเทียบ register accumulator กับ `[EDI]` เพื่อค้นหาค่าเฉพาะใน memory

```
SCASB - เปรียบเทียบ AL  กับ [EDI], อัปเดต EDI
SCASW - เปรียบเทียบ AX  กับ [EDI], อัปเดต EDI
SCASD - เปรียบเทียบ EAX กับ [EDI], อัปเดต EDI
SCASQ - เปรียบเทียบ RAX กับ [EDI], อัปเดต EDI (64-bit)
```

**หมายเหตุ**: SCAS ใช้ **EDI** เป็น pointer ไม่ใช่ ESI

การทำงาน:
```
; SCASB ทำงานเทียบเท่ากับ:
cmp al, [edi]    ; เปรียบเทียบ AL กับ byte ที่ EDI
inc edi          ; อัปเดต EDI
```

---

### 6. LODS - Load from String

LODS โหลดค่าจาก `[ESI]` เข้า accumulator register แล้วอัปเดต ESI

```
LODSB - โหลด [ESI] เข้า AL,  อัปเดต ESI
LODSW - โหลด [ESI] เข้า AX,  อัปเดต ESI
LODSD - โหลด [ESI] เข้า EAX, อัปเดต ESI
LODSQ - โหลด [ESI] เข้า RAX, อัปเดต ESI (64-bit)
```

การทำงาน:
```
; LODSB ทำงานเทียบเท่ากับ:
mov al, [esi]    ; โหลด byte จาก source
inc esi          ; อัปเดต ESI
```

LODS มักใช้ในลูปที่ต้องการประมวลผลข้อมูลทีละ element โดยไม่ใช้ REP

---

### 7. STOS - Store to String

STOS เขียนค่าจาก accumulator register ไปยัง `[EDI]` แล้วอัปเดต EDI

```
STOSB - เขียน AL  ไป [EDI], อัปเดต EDI
STOSW - เขียน AX  ไป [EDI], อัปเดต EDI
STOSD - เขียน EAX ไป [EDI], อัปเดต EDI
STOSQ - เขียน RAX ไป [EDI], อัปเดต EDI (64-bit)
```

การทำงาน:
```
; STOSB ทำงานเทียบเท่ากับ:
mov [edi], al    ; เขียน AL ไปยัง destination
inc edi          ; อัปเดต EDI
```

STOS ใช้บ่อยมากสำหรับการ initialize memory (เช่น memset)

---

### 8. REP Prefix - Unconditional Repeat

REP ทำให้ string instruction ทำซ้ำจนกว่า ECX จะเป็น 0

```
REP คำสั่ง:
  1. ถ้า ECX = 0 → หยุด
  2. ทำคำสั่ง
  3. ECX = ECX - 1
  4. กลับไปขั้นตอน 1
```

ตัวอย่าง:
```nasm
mov ecx, 100     ; ทำซ้ำ 100 ครั้ง
rep movsb        ; copy 100 bytes จาก [ESI] ไป [EDI]
```

REP ใช้ได้กับ: MOVS, STOS, LODS (แม้ว่า REP LODS จะไม่ค่อยมีประโยชน์)

---

### 9. REPE/REPZ และ REPNE/REPNZ

**REPE (REPeat while Equal) หรือ REPZ (REPeat while Zero)**:
```
ทำซ้ำในขณะที่: ECX != 0 AND ZF = 1 (ค่าที่เปรียบเทียบเท่ากัน)
```

**REPNE (REPeat while Not Equal) หรือ REPNZ (REPeat while Not Zero)**:
```
ทำซ้ำในขณะที่: ECX != 0 AND ZF = 0 (ค่าที่เปรียบเทียบไม่เท่ากัน)
```

ตัวอย่างการใช้:
```nasm
; REPE CMPSB - เปรียบเทียบ string จนกว่าจะพบความแตกต่าง
repe cmpsb       ; ทำซ้ำในขณะที่ bytes เท่ากันและ ECX != 0

; REPNE SCASB - ค้นหา null terminator ใน string
mov al, 0        ; ค้นหา null byte
repne scasb      ; สแกนจนกว่าจะพบ 0
```

---

## Code Examples (ตัวอย่างโปรแกรม)

---

### Example 1: Basic String Operations (การดำเนินการ String พื้นฐาน)

```nasm
; ===================================================================
; Part 020 - Example 1: Basic String Instructions Demo
; ไฟล์: example1_basic_string.asm
; แสดงการทำงานพื้นฐานของ String Instructions ทุกประเภท
; ===================================================================

section .data
    ; ข้อมูลต้นทาง
    source_str  db "Hello, Assembly World!", 0
    src_len     equ $ - source_str - 1   ; ความยาว string ไม่รวม null

    ; ข้อความสำหรับแสดงผล
    msg_movs    db "MOVSB result: ", 0
    msg_scas    db "SCASB strlen: ", 0
    msg_stos    db "STOSB fill:   ", 0
    msg_lods    db "LODSB count:  ", 0
    msg_nl      db 10, 0
    
    fmt_str     db "%s%s", 10, 0
    fmt_int     db "%s%d", 10, 0

section .bss
    ; บัฟเฟอร์สำหรับเก็บผลลัพธ์
    dest_buffer resb 64    ; destination สำหรับ MOVSB
    fill_buffer resb 32    ; buffer สำหรับ STOSB

section .text
    global main
    extern printf

main:
    push ebp
    mov ebp, esp
    sub esp, 16          ; จอง local variables
    push ebx             ; save registers ตาม calling convention
    push esi
    push edi

    ; =========================================================
    ; Demo 1: MOVSB - copy string
    ; =========================================================
    ; ตั้งค่า registers สำหรับ MOVSB
    cld                          ; DF = 0, เดินหน้า
    lea esi, [source_str]        ; ESI = ที่อยู่ source
    lea edi, [dest_buffer]       ; EDI = ที่อยู่ destination
    mov ecx, src_len + 1         ; copy รวม null terminator

    rep movsb                    ; copy ECX bytes จาก [ESI] ไป [EDI]
    ; หลังจาก rep movsb:
    ; - ESI ชี้ไปหลัง source string
    ; - EDI ชี้ไปหลัง destination string
    ; - ECX = 0

    ; แสดงผลลัพธ์
    push dest_buffer
    push msg_movs
    push fmt_str
    call printf
    add esp, 12

    ; =========================================================
    ; Demo 2: SCASB - scan for null terminator (วัดความยาว string)
    ; =========================================================
    cld
    lea edi, [source_str]        ; EDI = ที่อยู่ string
    mov ecx, 0xFFFFFFFF          ; maximum count
    xor al, al                   ; AL = 0 (null byte ที่ค้นหา)

    repne scasb                  ; สแกนจนกว่าจะพบ 0
    ; หลังจาก repne scasb:
    ; - EDI ชี้ไปหลัง null byte
    ; - ECX ถูกลดลงทุกครั้ง

    ; คำนวณความยาว: เริ่มต้น ECX = 0xFFFFFFFF
    ; ทำซ้ำ (len+1) ครั้ง → ECX = 0xFFFFFFFF - (len+1)
    ; ดังนั้น len = 0xFFFFFFFF - ECX - 1
    not ecx                      ; ECX = ~ECX = 0xFFFFFFFF - ECX
    dec ecx                      ; ลบ null terminator ออก
    ; ตอนนี้ ECX = length ของ string

    push ecx
    push msg_scas
    push fmt_int
    call printf
    add esp, 12

    ; =========================================================
    ; Demo 3: STOSB - fill buffer ด้วย pattern
    ; =========================================================
    cld
    lea edi, [fill_buffer]       ; EDI = ที่อยู่ buffer
    mov ecx, 32                  ; เติม 32 bytes
    mov al, 0x41                 ; AL = 'A' (ASCII 65)

    rep stosb                    ; เติม 'A' ทุก byte ใน buffer

    ; แสดง 8 bytes แรกของ fill buffer
    mov byte [fill_buffer + 8], 0   ; null terminate สำหรับแสดงผล
    push fill_buffer
    push msg_stos
    push fmt_str
    call printf
    add esp, 12

    ; =========================================================
    ; Demo 4: LODSB + ประมวลผลทีละ byte (นับตัวอักษร)
    ; =========================================================
    cld
    lea esi, [source_str]        ; ESI = ที่อยู่ string
    xor ecx, ecx                 ; ECX = 0 (counter)
    xor ebx, ebx                 ; EBX = 0 (letter count)

count_loop:
    lodsb                        ; AL = [ESI], ESI++
    test al, al                  ; ตรวจสอบ null terminator
    jz count_done                ; ถ้า AL = 0 จบ

    ; ตรวจสอบว่าเป็นตัวอักษรหรือไม่
    cmp al, 'A'
    jb not_letter
    cmp al, 'z'
    ja not_letter
    inc ebx                      ; นับตัวอักษร

not_letter:
    inc ecx                      ; นับทุก character
    jmp count_loop

count_done:
    ; แสดงจำนวนตัวอักษร
    push ebx
    push msg_lods
    push fmt_int
    call printf
    add esp, 12

    ; =========================================================
    ; Cleanup
    ; =========================================================
    pop edi
    pop esi
    pop ebx
    mov esp, ebp
    pop ebp
    xor eax, eax                 ; return 0
    ret
```

**คำอธิบาย**:
- Demo 1: `REP MOVSB` copy string ทั้งหมด
- Demo 2: `REPNE SCASB` ค้นหา null byte เพื่อวัดความยาว
- Demo 3: `REP STOSB` เติม buffer ด้วย 'A'
- Demo 4: `LODSB` ในลูปเพื่อประมวลผลทีละ byte

**คำสั่ง Compile และ Run**:
```bash
nasm -f elf32 example1_basic_string.asm -o example1.o
gcc -m32 example1.o -o example1
./example1
```

**Expected Output**:
```
MOVSB result: Hello, Assembly World!
SCASB strlen: 22
STOSB fill:   AAAAAAAA
LODSB count:  20
```

---

### Example 2: Custom strlen, strcpy, strcmp Implementation

```nasm
; ===================================================================
; Part 020 - Example 2: String Functions with REP Instructions
; ไฟล์: example2_string_functions.asm
; การ implement string functions พื้นฐานโดยใช้ string instructions
; ===================================================================

section .data
    ; test data
    str1        db "Hello World", 0
    str2        db "Hello World", 0
    str3        db "Hello Assembly", 0
    empty_str   db 0

    ; format strings สำหรับแสดงผล
    fmt_len     db "strlen('%s') = %d", 10, 0
    fmt_cpy     db "strcpy result: '%s'", 10, 0
    fmt_cmp_eq  db "strcmp('%s', '%s') = %d (equal)", 10, 0
    fmt_cmp_ne  db "strcmp('%s', '%s') = %d (not equal)", 10, 0
    fmt_sep     db "-----------------------------------", 10, 0

section .bss
    copy_buffer resb 64    ; buffer สำหรับ strcpy

section .text
    global main
    extern printf

; ===================================================================
; ฟังก์ชัน my_strlen - วัดความยาว string
; Input:  EDI = pointer to string
; Output: EAX = length (ไม่รวม null terminator)
; ทำลาย: ECX, EDI, FLAGS
; ===================================================================
my_strlen:
    push edi                     ; save EDI
    cld                          ; DF = 0, เดินหน้า
    xor eax, eax                 ; EAX = 0

    mov ecx, 0xFFFFFFFF          ; maximum possible length
    xor al, al                   ; AL = 0 (null byte ที่ค้นหา)
    repne scasb                  ; สแกนจนพบ null

    ; คำนวณความยาว
    ; ECX เริ่มต้น = 0xFFFFFFFF
    ; ทำซ้ำ (len+1) ครั้ง
    ; ECX = 0xFFFFFFFF - (len+1)
    not ecx                      ; ECX = len+1
    dec ecx                      ; ECX = len
    mov eax, ecx                 ; return value

    pop edi                      ; restore EDI
    ret

; ===================================================================
; ฟังก์ชัน my_strcpy - copy string
; Input:  EDI = destination pointer
;         ESI = source pointer  
; Output: EAX = destination pointer (ค่าเดิมของ EDI)
; ทำลาย: ECX, ESI, EDI, FLAGS
; ===================================================================
my_strcpy:
    push edi                     ; save original destination
    cld                          ; DF = 0, เดินหน้า

    ; หาความยาวของ source string ก่อน
    push edi                     ; save current EDI
    push esi
    mov edi, esi                 ; EDI = source (สำหรับ strlen)
    call my_strlen               ; EAX = strlen(source)
    pop esi
    pop edi                      ; restore destination

    ; copy รวม null terminator (+1)
    inc eax                      ; เพิ่ม 1 สำหรับ null terminator
    mov ecx, eax                 ; ECX = bytes to copy

    rep movsb                    ; copy ECX bytes จาก [ESI] ไป [EDI]

    pop eax                      ; EAX = original destination pointer
    ret

; ===================================================================
; ฟังก์ชัน my_strcmp - เปรียบเทียบ strings
; Input:  ESI = pointer to string1
;         EDI = pointer to string2
; Output: EAX = 0 ถ้าเท่ากัน
;               < 0 ถ้า str1 < str2
;               > 0 ถ้า str1 > str2
; ทำลาย: ECX, ESI, EDI, FLAGS
; ===================================================================
my_strcmp:
    cld                          ; DF = 0, เดินหน้า
    mov ecx, 0xFFFFFFFF          ; maximum count

strcmp_loop:
    ; โหลด byte จากทั้งสอง strings
    mov al, [esi]                ; AL = byte จาก string1
    mov bl, [edi]                ; BL = byte จาก string2

    ; ตรวจสอบ null terminators
    test al, al                  ; ตรวจสอบ string1 จบหรือยัง
    jz strcmp_end
    test bl, bl                  ; ตรวจสอบ string2 จบหรือยัง
    jz strcmp_end

    ; เปรียบเทียบ bytes
    cmp al, bl
    jne strcmp_diff              ; ถ้าต่างกัน ออกจาก loop

    ; bytes เท่ากัน ไปยัง byte ถัดไป
    inc esi
    inc edi
    loop strcmp_loop

strcmp_diff:
    ; คำนวณ return value
    movzx eax, al                ; EAX = byte จาก string1
    movzx ebx, bl                ; EBX = byte จาก string2
    sub eax, ebx                 ; EAX = str1[i] - str2[i]
    ret

strcmp_end:
    ; หนึ่งหรือทั้งสอง strings จบแล้ว
    movzx eax, al
    movzx ebx, bl
    sub eax, ebx
    ret

; ===================================================================
; ฟังก์ชัน main
; ===================================================================
main:
    push ebp
    mov ebp, esp
    push ebx
    push esi
    push edi

    ; ------- Test strlen -------
    ; strlen("Hello World")
    lea edi, [str1]
    call my_strlen               ; EAX = 11

    push eax
    push str1
    push fmt_len
    call printf
    add esp, 12

    ; strlen("")
    lea edi, [empty_str]
    call my_strlen               ; EAX = 0

    push eax
    push empty_str
    push fmt_len
    call printf
    add esp, 12

    push fmt_sep
    call printf
    add esp, 4

    ; ------- Test strcpy -------
    lea edi, [copy_buffer]       ; destination
    lea esi, [str1]              ; source
    call my_strcpy               ; copy str1 -> copy_buffer

    push copy_buffer
    push fmt_cpy
    call printf
    add esp, 8

    push fmt_sep
    call printf
    add esp, 4

    ; ------- Test strcmp -------
    ; เปรียบเทียบ str1 กับ str2 (เท่ากัน)
    lea esi, [str1]
    lea edi, [str2]
    call my_strcmp               ; EAX = 0

    push eax
    push str2
    push str1
    push fmt_cmp_eq
    call printf
    add esp, 16

    ; เปรียบเทียบ str1 กับ str3 (ต่างกัน)
    lea esi, [str1]
    lea edi, [str3]
    call my_strcmp               ; EAX != 0

    push eax
    push str3
    push str1
    push fmt_cmp_ne
    call printf
    add esp, 16

    pop edi
    pop esi
    pop ebx
    mov esp, ebp
    pop ebp
    xor eax, eax
    ret
```

**คำสั่ง Compile และ Run**:
```bash
nasm -f elf32 example2_string_functions.asm -o example2.o
gcc -m32 example2.o -o example2
./example2
```

**Expected Output**:
```
strlen('Hello World') = 11
strlen('') = 0
-----------------------------------
strcpy result: 'Hello World'
-----------------------------------
strcmp('Hello World', 'Hello World') = 0 (equal)
strcmp('Hello World', 'Hello Assembly') = 13 (not equal)
```

---

### Example 3: memset, memcpy, memcmp Implementation

```nasm
; ===================================================================
; Part 020 - Example 3: Memory Functions
; ไฟล์: example3_memory_functions.asm
; การ implement memory functions โดยใช้ string instructions
; ===================================================================

section .data
    src_data    dd 1, 2, 3, 4, 5, 6, 7, 8   ; 8 integers = 32 bytes
    src_len     equ 32

    ; ข้อมูลสำหรับ memcmp test
    data_a      db "ABCDEFGH", 0
    data_b      db "ABCDEFGH", 0
    data_c      db "ABCXEFGH", 0

    ; format strings
    fmt_memset  db "memset  buffer[0..7]: %d %d %d %d %d %d %d %d", 10, 0
    fmt_memcpy  db "memcpy  buffer[0..7]: %d %d %d %d %d %d %d %d", 10, 0
    fmt_cmp_eq  db "memcmp(data_a, data_b, 8) = %d (should be 0)", 10, 0
    fmt_cmp_ne  db "memcmp(data_a, data_c, 8) = %d (should not be 0)", 10, 0
    fmt_chr     db "memchr found 'D' at offset: %d", 10, 0
    fmt_sep     db "==========================================", 10, 0

section .bss
    buffer1     resd 8     ; 32 bytes buffer (8 dwords)
    buffer2     resd 8     ; 32 bytes buffer

section .text
    global main
    extern printf

; ===================================================================
; my_memset - เติม memory ด้วยค่าที่กำหนด
; Input:  EDI = destination pointer
;         AL  = fill value (byte)
;         ECX = number of bytes
; Output: EAX = destination pointer
; ทำลาย: ECX, EDI
; ===================================================================
my_memset:
    push edi                     ; save original destination
    cld                          ; DF = 0
    rep stosb                    ; เติม ECX bytes ด้วย AL
    pop eax                      ; EAX = original destination
    ret

; ===================================================================
; my_memset_dword - เติม memory ด้วย dword value (เร็วกว่า memset ปกติ)
; Input:  EDI = destination pointer
;         EAX = fill value (dword)
;         ECX = number of dwords
; Output: EAX = destination pointer
; ===================================================================
my_memset_dword:
    push edi
    cld
    rep stosd                    ; เติม ECX dwords ด้วย EAX
    pop eax
    ret

; ===================================================================
; my_memcpy - copy memory
; Input:  EDI = destination pointer
;         ESI = source pointer
;         ECX = number of bytes
; Output: EAX = destination pointer
; ===================================================================
my_memcpy:
    push edi                     ; save original destination
    cld
    rep movsb                    ; copy ECX bytes
    pop eax
    ret

; ===================================================================
; my_memcpy_optimized - copy memory แบบ optimized
; Copy เป็น dwords ก่อน แล้วค่อย copy bytes ที่เหลือ
; Input:  EDI = destination pointer
;         ESI = source pointer
;         ECX = number of bytes
; ===================================================================
my_memcpy_optimized:
    push edi
    cld

    ; Copy เป็น dwords (4 bytes ต่อครั้ง)
    mov eax, ecx
    shr ecx, 2                   ; ECX = ECX / 4 (จำนวน dwords)
    rep movsd                    ; copy dwords

    ; Copy bytes ที่เหลือ
    mov ecx, eax
    and ecx, 3                   ; ECX = ECX mod 4 (bytes ที่เหลือ)
    rep movsb                    ; copy bytes ที่เหลือ

    pop eax
    ret

; ===================================================================
; my_memcmp - เปรียบเทียบ memory blocks
; Input:  ESI = pointer to block1
;         EDI = pointer to block2
;         ECX = number of bytes
; Output: EAX = 0 ถ้าเท่ากัน, != 0 ถ้าต่างกัน
; ===================================================================
my_memcmp:
    cld
    repe cmpsb                   ; เปรียบเทียบจนกว่าจะพบความแตกต่าง
                                 ; หรือ ECX = 0

    ; ตรวจสอบผลลัพธ์
    ; ถ้า ZF = 1 → ทุก byte เท่ากัน (ECX = 0)
    ; ถ้า ZF = 0 → พบความแตกต่าง
    jz memcmp_equal

    ; พบความแตกต่าง: คำนวณ return value
    movzx eax, byte [esi - 1]    ; byte สุดท้ายที่อ่าน (ESI ถูก increment แล้ว)
    movzx ebx, byte [edi - 1]
    sub eax, ebx
    ret

memcmp_equal:
    xor eax, eax                 ; return 0
    ret

; ===================================================================
; my_memchr - ค้นหา byte ใน memory block
; Input:  EDI = pointer to memory block
;         AL  = byte ที่ค้นหา
;         ECX = number of bytes to search
; Output: EAX = pointer to found byte, หรือ 0 ถ้าไม่พบ
; ===================================================================
my_memchr:
    cld
    push edi                     ; save original pointer

    repne scasb                  ; สแกนจนพบ byte หรือ ECX = 0

    jnz memchr_not_found         ; ZF = 0 → ไม่พบ

    ; พบ: EDI ชี้หลัง found byte
    lea eax, [edi - 1]           ; EAX = pointer to found byte
    pop edi                      ; discard saved pointer
    ret

memchr_not_found:
    xor eax, eax                 ; return NULL
    pop edi
    ret

; ===================================================================
; main function
; ===================================================================
main:
    push ebp
    mov ebp, esp
    push ebx
    push esi
    push edi

    ; ------- Test memset -------
    ; เติม buffer1 ด้วย 0x42 ('B')
    lea edi, [buffer1]
    mov al, 0x42                 ; fill value = 'B' = 66
    mov ecx, 32                  ; 32 bytes
    call my_memset

    ; แสดงผล buffer1 (แสดงเป็น decimal ASCII)
    push dword [buffer1 + 28]
    push dword [buffer1 + 24]
    push dword [buffer1 + 20]
    push dword [buffer1 + 16]
    push dword [buffer1 + 12]
    push dword [buffer1 + 8]
    push dword [buffer1 + 4]
    push dword [buffer1]
    push fmt_memset
    call printf
    add esp, 36

    ; ------- Test memcpy -------
    ; copy src_data ไปยัง buffer2
    lea edi, [buffer2]
    lea esi, [src_data]
    mov ecx, src_len
    call my_memcpy_optimized

    ; แสดงผล buffer2
    push dword [buffer2 + 28]
    push dword [buffer2 + 24]
    push dword [buffer2 + 20]
    push dword [buffer2 + 16]
    push dword [buffer2 + 12]
    push dword [buffer2 + 8]
    push dword [buffer2 + 4]
    push dword [buffer2]
    push fmt_memcpy
    call printf
    add esp, 36

    push fmt_sep
    call printf
    add esp, 4

    ; ------- Test memcmp -------
    ; เปรียบเทียบ data_a กับ data_b (เท่ากัน)
    lea esi, [data_a]
    lea edi, [data_b]
    mov ecx, 8
    call my_memcmp

    push eax
    push fmt_cmp_eq
    call printf
    add esp, 8

    ; เปรียบเทียบ data_a กับ data_c (ต่างกันที่ตำแหน่ง 3)
    lea esi, [data_a]
    lea edi, [data_c]
    mov ecx, 8
    call my_memcmp

    push eax
    push fmt_cmp_ne
    call printf
    add esp, 8

    push fmt_sep
    call printf
    add esp, 4

    ; ------- Test memchr -------
    ; ค้นหา 'D' ใน data_a
    lea edi, [data_a]
    mov al, 'D'                  ; ค้นหา 'D'
    mov ecx, 8
    call my_memchr

    ; คำนวณ offset
    lea ebx, [data_a]
    sub eax, ebx                 ; offset = found_ptr - base_ptr

    push eax
    push fmt_chr
    call printf
    add esp, 8

    pop edi
    pop esi
    pop ebx
    mov esp, ebp
    pop ebp
    xor eax, eax
    ret
```

**คำสั่ง Compile และ Run**:
```bash
nasm -f elf32 example3_memory_functions.asm -o example3.o
gcc -m32 example3.o -o example3
./example3
```

**Expected Output**:
```
memset  buffer[0..7]: 66 66 66 66 66 66 66 66
memcpy  buffer[0..7]: 1 2 3 4 5 6 7 8
==========================================
memcmp(data_a, data_b, 8) = 0 (should be 0)
memcmp(data_a, data_c, 8) = -20 (should not be 0)
==========================================
memchr found 'D' at offset: 3
```

---

### Example 4: Direction Flag Demo (Forward vs Backward)

```nasm
; ===================================================================
; Part 020 - Example 4: Direction Flag (CLD vs STD)
; ไฟล์: example4_direction_flag.asm
; แสดงความแตกต่างระหว่าง forward copy และ backward copy
; ===================================================================

section .data
    ; ข้อมูลทดสอบ
    test_data   db "ABCDEFGHIJ", 0
    test_len    equ 10

    ; ข้อความแสดงผล
    msg_fwd     db "Forward copy:  ", 0
    msg_bwd     db "Backward copy: ", 0
    msg_overlap db "Overlap copy:  ", 0
    msg_src     db "Source:        ", 0
    msg_nl      db 10, 0
    fmt_buf     db "%s%s", 10, 0

    ; สำหรับ overlap test - copy overlapping region ไปข้างหน้า
    ; ต้องใช้ backward copy (STD) เพื่อหลีกเลี่ยง overwrite
    overlap_info db "=== Overlap Copy Demo ===", 10, 0
    overlap_msg1 db "Shifting 'ABCDE' right by 2 positions:", 10, 0
    before_msg   db "Before: ", 0
    after_msg    db "After:  ", 0

section .bss
    fwd_buffer  resb 16    ; forward copy destination
    bwd_buffer  resb 16    ; backward copy destination
    ovlp_buffer resb 16    ; overlap test buffer

section .text
    global main
    extern printf, memset

main:
    push ebp
    mov ebp, esp
    push esi
    push edi

    ; =========================================================
    ; Clear buffers
    ; =========================================================
    push 0
    push 16
    push fwd_buffer
    call memset
    add esp, 12

    push 0
    push 16
    push bwd_buffer
    call memset
    add esp, 12

    ; =========================================================
    ; Demo 1: Forward Copy (CLD)
    ; =========================================================
    cld                          ; DF = 0 → increment

    lea esi, [test_data]         ; source
    lea edi, [fwd_buffer]        ; destination
    mov ecx, test_len            ; ECX = 10

    rep movsb                    ; copy forward

    ; แสดงผล forward copy
    push fwd_buffer
    push msg_fwd
    push fmt_buf
    call printf
    add esp, 12

    ; =========================================================
    ; Demo 2: Backward Copy (STD)
    ; สำคัญ: ต้องตั้ง ESI/EDI ชี้ไปที่ปลายของ data
    ; =========================================================
    std                          ; DF = 1 → decrement

    ; ESI ชี้ไปที่ byte สุดท้ายของ source
    lea esi, [test_data + test_len - 1]
    ; EDI ชี้ไปที่ byte สุดท้ายของ destination
    lea edi, [bwd_buffer + test_len - 1]
    mov ecx, test_len

    rep movsb                    ; copy backward (จากปลายไปต้น)

    cld                          ; กลับมา DF = 0 (สำคัญ! อย่าลืม)

    ; แสดงผล backward copy
    push bwd_buffer
    push msg_bwd
    push fmt_buf
    call printf
    add esp, 12

    ; =========================================================
    ; Demo 3: Overlapping Copy
    ; ปัญหา: copy "ABCDE" ไปข้างหน้า 2 positions
    ; ถ้าใช้ forward copy: A B C D E _ _ _ _ _
    ;                      ↗ ↗ ↗ ↗ ↗
    ;                  _ _ A B C D E _ _ _
    ; แต่ถ้า dst overlap กับ src ต้องระวัง!
    ; =========================================================
    push overlap_info
    call printf
    add esp, 4

    push overlap_msg1
    call printf
    add esp, 4

    ; ตั้งค่า overlap_buffer ด้วย "ABCDE....."
    lea edi, [ovlp_buffer]
    mov byte [edi + 0], 'A'
    mov byte [edi + 1], 'B'
    mov byte [edi + 2], 'C'
    mov byte [edi + 3], 'D'
    mov byte [edi + 4], 'E'
    mov byte [edi + 5], '.'
    mov byte [edi + 6], '.'
    mov byte [edi + 7], '.'
    mov byte [edi + 8], '.'
    mov byte [edi + 9], '.'
    mov byte [edi + 10], 0

    ; แสดง before
    push ovlp_buffer
    push before_msg
    push fmt_buf
    call printf
    add esp, 12

    ; ต้องการ shift "ABCDE" ไปข้างหน้า 2 positions
    ; dst = ovlp_buffer + 2, src = ovlp_buffer, len = 5
    ; เนื่องจาก dst > src และ regions overlap ต้องใช้ backward copy

    std                          ; DF = 1 → backward

    lea esi, [ovlp_buffer + 4]   ; ปลาย source (E)
    lea edi, [ovlp_buffer + 6]   ; ปลาย destination (G position)
    mov ecx, 5

    rep movsb                    ; copy backward

    cld                          ; กลับมา DF = 0
    mov byte [ovlp_buffer + 10], 0  ; null terminate

    ; แสดง after
    push ovlp_buffer
    push after_msg
    push fmt_buf
    call printf
    add esp, 12

    pop edi
    pop esi
    mov esp, ebp
    pop ebp
    xor eax, eax
    ret
```

**คำสั่ง Compile และ Run**:
```bash
nasm -f elf32 example4_direction_flag.asm -o example4.o
gcc -m32 example4.o -o example4
./example4
```

**Expected Output**:
```
Forward copy:  ABCDEFGHIJ
Backward copy: ABCDEFGHIJ
=== Overlap Copy Demo ===
Shifting 'ABCDE' right by 2 positions:
Before: ABCDE.....
After:  ABABCDE...
```

---

### Example 5: Performance Comparison (String Instructions vs Manual Loop)

```nasm
; ===================================================================
; Part 020 - Example 5: Performance Comparison
; ไฟล์: example5_performance.asm
; เปรียบเทียบความเร็ว: REP MOVSD vs manual loop
; ===================================================================

section .data
    BUFFER_SIZE equ 1024 * 1024    ; 1 MB buffer

    ; format strings
    fmt_header  db "Performance Test: Copy %d bytes", 10, 0
    fmt_manual  db "Manual loop (byte): ", 0
    fmt_repb    db "REP MOVSB:          ", 0
    fmt_repd    db "REP MOVSD:          ", 0
    fmt_cycles  db "%llu cycles", 10, 0
    fmt_sep     db "------------------------", 10, 0
    fmt_winner  db "MOVSD is ~%dx faster than manual byte loop", 10, 0

section .bss
    src_buf     resb BUFFER_SIZE
    dst_buf     resb BUFFER_SIZE

section .text
    global main
    extern printf

; ===================================================================
; get_tsc - อ่าน CPU timestamp counter (RDTSC instruction)
; Output: EDX:EAX = 64-bit cycle count
; ===================================================================
get_tsc:
    rdtsc        ; Read Time-Stamp Counter → EDX:EAX
    ret

; ===================================================================
; copy_manual - copy แบบ manual loop (byte ต่อ byte)
; Input:  EDI = destination
;         ESI = source
;         ECX = byte count
; ===================================================================
copy_manual:
    test ecx, ecx
    jz copy_manual_done

manual_loop:
    mov al, [esi]
    mov [edi], al
    inc esi
    inc edi
    dec ecx
    jnz manual_loop

copy_manual_done:
    ret

; ===================================================================
; copy_rep_movsb - copy โดยใช้ REP MOVSB
; Input:  EDI = destination
;         ESI = source
;         ECX = byte count
; ===================================================================
copy_rep_movsb:
    cld
    rep movsb
    ret

; ===================================================================
; copy_rep_movsd - copy โดยใช้ REP MOVSD (optimized)
; Input:  EDI = destination
;         ESI = source
;         ECX = byte count
; ===================================================================
copy_rep_movsd:
    cld
    push ecx

    ; copy dwords ก่อน
    shr ecx, 2
    rep movsd

    ; copy bytes ที่เหลือ
    pop ecx
    and ecx, 3
    rep movsb
    ret

; ===================================================================
; main
; ===================================================================
main:
    push ebp
    mov ebp, esp
    sub esp, 32          ; local variables: 2x 64-bit timestamps + overhead
    push ebx
    push esi
    push edi

    ; แสดง header
    push BUFFER_SIZE
    push fmt_header
    call printf
    add esp, 8

    push fmt_sep
    call printf
    add esp, 4

    ; =========================================================
    ; Test 1: Manual loop (byte per byte)
    ; =========================================================
    push fmt_manual
    call printf
    add esp, 4

    ; เริ่มจับเวลา
    call get_tsc
    mov [ebp - 8], eax           ; low 32 bits ของ start time
    mov [ebp - 4], edx           ; high 32 bits ของ start time

    lea esi, [src_buf]
    lea edi, [dst_buf]
    mov ecx, BUFFER_SIZE
    call copy_manual

    ; หยุดจับเวลา
    call get_tsc
    ; EAX = low, EDX = high ของ end time
    sub eax, [ebp - 8]           ; elapsed low
    sbb edx, [ebp - 4]           ; elapsed high

    ; แสดง cycles
    push edx
    push eax
    push fmt_cycles
    call printf
    add esp, 12

    ; เก็บ manual cycles สำหรับเปรียบเทียบ
    mov [ebp - 16], eax          ; manual_cycles low
    mov [ebp - 12], edx          ; manual_cycles high

    ; =========================================================
    ; Test 2: REP MOVSB
    ; =========================================================
    push fmt_repb
    call printf
    add esp, 4

    call get_tsc
    mov [ebp - 8], eax
    mov [ebp - 4], edx

    lea esi, [src_buf]
    lea edi, [dst_buf]
    mov ecx, BUFFER_SIZE
    call copy_rep_movsb

    call get_tsc
    sub eax, [ebp - 8]
    sbb edx, [ebp - 4]

    push edx
    push eax
    push fmt_cycles
    call printf
    add esp, 12

    ; =========================================================
    ; Test 3: REP MOVSD (fastest)
    ; =========================================================
    push fmt_repd
    call printf
    add esp, 4

    call get_tsc
    mov [ebp - 8], eax
    mov [ebp - 4], edx

    lea esi, [src_buf]
    lea edi, [dst_buf]
    mov ecx, BUFFER_SIZE
    call copy_rep_movsd

    call get_tsc
    sub eax, [ebp - 8]
    sbb edx, [ebp - 4]

    ; เก็บ MOVSD cycles
    mov [ebp - 24], eax
    mov [ebp - 20], edx

    push edx
    push eax
    push fmt_cycles
    call printf
    add esp, 12

    push fmt_sep
    call printf
    add esp, 4

    ; คำนวณ speedup (rough estimate ใช้เฉพาะ low 32 bits)
    mov eax, [ebp - 16]          ; manual_cycles
    mov ecx, [ebp - 24]          ; movsd_cycles
    xor edx, edx
    div ecx                      ; EAX = manual / movsd

    push eax
    push fmt_winner
    call printf
    add esp, 8

    pop edi
    pop esi
    pop ebx
    mov esp, ebp
    pop ebp
    xor eax, eax
    ret
```

**คำสั่ง Compile และ Run**:
```bash
nasm -f elf32 example5_performance.asm -o example5.o
gcc -m32 example5.o -o example5
./example5
```

**Expected Output** (ตัวเลขจะแตกต่างกันตามเครื่อง):
```
Performance Test: Copy 1048576 bytes
------------------------
Manual loop (byte): 8234561 cycles
REP MOVSB:          2145632 cycles
REP MOVSD:          987234 cycles
------------------------
MOVSD is ~8x faster than manual byte loop
```

---

### Example 6: Advanced String Search (strstr-like)

```nasm
; ===================================================================
; Part 020 - Example 6: String Pattern Search
; ไฟล์: example6_strstr.asm
; การค้นหา pattern ใน string โดยใช้ string instructions
; ===================================================================

section .data
    haystack    db "The quick brown fox jumps over the lazy dog", 0
    hay_len     equ 43

    needle1     db "fox", 0
    needle2     db "cat", 0
    needle3     db "the", 0   ; case-sensitive: พบ "the" แต่ไม่ใช่ "The"

    fmt_found   db "Found '%s' at position %d", 10, 0
    fmt_notfnd  db "'%s' not found", 10, 0
    fmt_header  db "Searching in: '%s'", 10, 0

section .text
    global main
    extern printf, strlen

; ===================================================================
; my_strlen_edi - วัดความยาว string ที่ EDI ชี้อยู่
; Input:  EDI = pointer to string
; Output: EAX = length
; ===================================================================
my_strlen_edi:
    push ecx
    push edi
    cld
    mov ecx, 0xFFFFFFFF
    xor al, al
    repne scasb
    not ecx
    dec ecx
    mov eax, ecx
    pop edi
    pop ecx
    ret

; ===================================================================
; my_strstr - ค้นหา needle ใน haystack
; Input:  ESI = haystack pointer
;         EDI = needle pointer
; Output: EAX = pointer ถ้าพบ, 0 ถ้าไม่พบ
; ===================================================================
my_strstr:
    push ebp
    mov ebp, esp
    push ebx
    push esi
    push edi

    ; คำนวณความยาว needle
    call my_strlen_edi           ; EDI = needle
    mov ebx, eax                 ; EBX = needle_len
    test eax, eax                ; needle เป็น empty string?
    jz strstr_found_start        ; ถ้า empty → return haystack

    ; ถ้า needle_len = 0 → return ESI (haystack)
    ; ค้นหา char แรกของ needle ใน haystack ก่อน
    mov dl, [edi]                ; DL = needle[0]

strstr_outer:
    ; ตรวจสอบว่า haystack ยังเหลืออยู่หรือไม่
    cmp byte [esi], 0
    jz strstr_not_found

    ; ค้นหา needle[0] ใน haystack โดยใช้ SCASB
    push edi                     ; save needle pointer
    push esi                     ; save current haystack position
    mov edi, esi                 ; EDI = current haystack position
    cld
    mov al, dl                   ; AL = needle[0]

    ; scan ไปจนพบ needle[0] หรือ end of string
scan_first_char:
    mov al, [edi]
    test al, al
    jz strstr_restore_notfound
    cmp al, dl                   ; เปรียบเทียบกับ needle[0]
    je found_first_char
    inc edi
    jmp scan_first_char

found_first_char:
    ; พบ needle[0] ที่ EDI → เปรียบเทียบ needle ทั้งหมด
    pop esi                      ; restore (discard)
    pop edi                      ; restore needle

    push edi                     ; save needle pointer again
    push ebx                     ; save needle_len

    ; เปรียบเทียบทีละ byte
    mov ecx, ebx                 ; ECX = needle_len
    push edi
    ; EDI ปัจจุบันชี้ที่ needle, esi ชี้ที่ haystack position
    ; ต้องการเปรียบเทียบ [needle] กับ haystack ที่ found_first_char
    ; ใช้ CMPSB

    ; ... (simplified version using byte comparison)
    pop edi
    pop ebx
    pop edi

    ; คำนวณ match
    ; จะใช้ repe cmpsb เพื่อ compare
    pop esi
    push esi

    ; ค้นหา position ของ needle[0] อีกครั้งอย่างง่าย
    ; (simplified for clarity)

    ; ข้าม code นี้และใช้ approach ที่ง่ายกว่า:
    pop esi
    inc esi                      ; ลอง position ถัดไป
    jmp strstr_outer

strstr_restore_notfound:
    pop esi
    pop edi

strstr_not_found:
    xor eax, eax                 ; return NULL
    jmp strstr_exit

strstr_found_start:
    mov eax, esi                 ; return haystack pointer

strstr_exit:
    pop edi
    pop esi
    pop ebx
    pop ebp
    ret

; ===================================================================
; my_strstr_simple - version ที่ง่ายกว่า ใช้ REPE CMPSB
; Input:  ESI = haystack pointer
;         EDI = needle pointer
; Output: EAX = pointer ถ้าพบ, 0 ถ้าไม่พบ
; ===================================================================
my_strstr_simple:
    push ebp
    mov ebp, esp
    push ebx
    push esi
    push edi

    ; คำนวณ needle length
    push edi
    call my_strlen_edi
    pop edi
    mov ebx, eax                 ; EBX = needle_len

    test ebx, ebx
    jz .found                    ; empty needle → return haystack

.outer_loop:
    cmp byte [esi], 0            ; haystack จบหรือยัง
    jz .not_found

    ; เปรียบเทียบ needle กับ haystack ที่ position ปัจจุบัน
    push esi                     ; save haystack position
    push edi                     ; save needle position
    mov ecx, ebx                 ; ECX = needle_len
    cld
    repe cmpsb                   ; เปรียบเทียบ ECX bytes

    ; ตรวจสอบผลลัพธ์
    jnz .no_match                ; ไม่ match
    ; ECX = 0 และ ZF = 1 → match!
    pop edi                      ; discard saved needle
    pop eax                      ; EAX = matched haystack position
    jmp .exit

.no_match:
    pop edi                      ; restore needle
    pop esi                      ; restore haystack position
    inc esi                      ; ลอง position ถัดไป
    jmp .outer_loop

.not_found:
    xor eax, eax
    jmp .exit

.found:
    mov eax, esi

.exit:
    pop edi
    pop esi
    pop ebx
    pop ebp
    ret

; ===================================================================
; main
; ===================================================================
main:
    push ebp
    mov ebp, esp
    push esi
    push edi

    ; แสดง haystack
    push haystack
    push fmt_header
    call printf
    add esp, 8

    ; ค้นหา needle1 ("fox")
    lea esi, [haystack]
    lea edi, [needle1]
    call my_strstr_simple

    test eax, eax
    jz .not_found1

    lea ecx, [haystack]
    sub eax, ecx                 ; offset = result - base
    push eax
    push needle1
    push fmt_found
    call printf
    add esp, 12
    jmp .search2

.not_found1:
    push needle1
    push fmt_notfnd
    call printf
    add esp, 8

.search2:
    ; ค้นหา needle2 ("cat")
    lea esi, [haystack]
    lea edi, [needle2]
    call my_strstr_simple

    test eax, eax
    jz .not_found2

    lea ecx, [haystack]
    sub eax, ecx
    push eax
    push needle2
    push fmt_found
    call printf
    add esp, 12
    jmp .search3

.not_found2:
    push needle2
    push fmt_notfnd
    call printf
    add esp, 8

.search3:
    ; ค้นหา needle3 ("the") - case sensitive
    lea esi, [haystack]
    lea edi, [needle3]
    call my_strstr_simple

    test eax, eax
    jz .not_found3

    lea ecx, [haystack]
    sub eax, ecx
    push eax
    push needle3
    push fmt_found
    call printf
    add esp, 12
    jmp .done

.not_found3:
    push needle3
    push fmt_notfnd
    call printf
    add esp, 8

.done:
    pop edi
    pop esi
    mov esp, ebp
    pop ebp
    xor eax, eax
    ret
```

**คำสั่ง Compile และ Run**:
```bash
nasm -f elf32 example6_strstr.asm -o example6.o
gcc -m32 example6.o -o example6
./example6
```

**Expected Output**:
```
Searching in: 'The quick brown fox jumps over the lazy dog'
Found 'fox' at position 16
'cat' not found
Found 'the' at position 31
```

---

## Common Mistakes and Pitfalls (ข้อผิดพลาดที่พบบ่อย)

### 1. ลืมตั้งค่า Direction Flag

```nasm
; ❌ ผิด: ไม่ได้ set direction flag ก่อนใช้ string instruction
mov ecx, 100
rep movsb              ; ไม่รู้ว่า DF เป็น 0 หรือ 1!

; ✅ ถูก: ตั้งค่า DF ทุกครั้งก่อนใช้
cld                    ; DF = 0 (forward)
mov ecx, 100
rep movsb
```

**อธิบาย**: โปรแกรมอื่นหรือ system call อาจเปลี่ยน DF ได้ ดังนั้นต้องตั้งค่าทุกครั้ง

---

### 2. ลืม Restore DF หลังใช้ STD

```nasm
; ❌ ผิด: ลืม CLD หลัง STD
std
mov ecx, 10
rep movsb
; DF ยังเป็น 1 อยู่ → code ถัดไปอาจทำงานผิดพลาด!

; ✅ ถูก: CLD เสมอหลัง STD
std
mov ecx, 10
rep movsb
cld                    ; กลับมา DF = 0 หลังเสร็จ
```

---

### 3. Register ผิดสำหรับ SCAS

```nasm
; ❌ ผิด: SCAS ใช้ EDI ไม่ใช่ ESI!
lea esi, [string]      ; ผิด! SCAS ต้องใช้ EDI
mov al, 0
repne scasb

; ✅ ถูก:
lea edi, [string]      ; SCAS ใช้ EDI
mov al, 0
repne scasb
```

---

### 4. ขนาดข้อมูลไม่ตรงกับ Register ที่ใช้

```nasm
; ❌ ผิด: ต้องการ copy byte แต่ใช้ MOVSD
mov ecx, 5             ; ต้องการ copy 5 bytes
rep movsd              ; แต่ copy 5 * 4 = 20 bytes!

; ✅ ถูก:
mov ecx, 5
rep movsb              ; copy 5 bytes ตามต้องการ

; หรือถ้าต้องการ copy 20 bytes ด้วย MOVSD:
mov ecx, 5             ; 5 dwords = 20 bytes
rep movsd
```

---

### 5. Overlapping Memory Regions

```nasm
; ❌ อันตราย: copy overlapping region ไปข้างหน้า (dst > src)
; src:  [A][B][C][D][E]
; dst:      [A][B][C][D][E]   (dst overlap กับ src)
; ถ้าใช้ forward copy:
; ขั้นที่ 1: A → B  → B ถูกเขียนทับ!
; ขั้นที่ 2: B (ซึ่งตอนนี้ = A) → C  → ผิดพลาด!

cld
lea esi, [buffer]
lea edi, [buffer + 2]
mov ecx, 5
rep movsb              ; ❌ ผลลัพธ์ผิดพลาด!

; ✅ ถูก: ใช้ backward copy (STD)
std
lea esi, [buffer + 4]          ; ปลาย source
lea edi, [buffer + 6]          ; ปลาย destination
mov ecx, 5
rep movsb
cld
```

---

### 6. ECX = 0 ทำให้ REP ไม่ทำงาน (โดยตั้งใจ)

```nasm
; หมายเหตุ: REP จะ check ECX ก่อนเริ่ม
; ถ้า ECX = 0 → ไม่ทำอะไรเลย (ไม่ใช่ bug แต่ต้องเข้าใจ)
xor ecx, ecx
rep movsb              ; ไม่ copy อะไรเลย (ECX = 0)
```

---

### 7. LODS/STOS ใช้ ESI/EDI ตามลำดับ

```nasm
; LODS ใช้ ESI (source)
; STOS ใช้ EDI (destination)

; ❌ ผิด: ใช้ register ผิด
lea edi, [src]         ; LODS ต้องการ ESI!
lodsb                  ; ผิด!

; ✅ ถูก:
lea esi, [src]         ; LODS ใช้ ESI
lodsb                  ; AL = [ESI], ESI++
```

---

## Advanced Techniques (เทคนิคขั้นสูง)

### Technique 1: Fast Memory Clear ด้วย STOSD

```nasm
; ===================================================================
; Fast memset(0) โดยใช้ STOSD แทน STOSB
; STOSD เร็วกว่า STOSB ประมาณ 4x (ในบาง CPU architectures)
; ===================================================================
fast_bzero:
    ; Input: EDI = pointer, ECX = byte count
    push ecx

    ; เคลียร์เป็น dwords ก่อน
    xor eax, eax                 ; EAX = 0
    shr ecx, 2                   ; ECX = dword count
    cld
    rep stosd                    ; เคลียร์ dwords

    ; เคลียร์ bytes ที่เหลือ
    pop ecx
    and ecx, 3                   ; ECX mod 4
    rep stosb                    ; เคลียร์ bytes ที่เหลือ
    ret
```

---

### Technique 2: CMPS กับ REP prefix combinations

```nasm
; ===================================================================
; เปรียบเทียบ strings สองตัวอย่างสมบูรณ์
; ===================================================================
compare_strings_full:
    ; Input: ESI = str1, EDI = str2
    ; Output: EAX = 0 (equal), -1 (str1 < str2), 1 (str1 > str2)

    cld
    ; หา max comparison length จาก strlen
    ; (ใน real code ควร limit ECX เพื่อความปลอดภัย)
    mov ecx, 256                 ; max comparison length

.cmp_loop:
    repe cmpsb                   ; เปรียบเทียบจนพบความแตกต่าง

    ; ตรวจสอบ null terminators
    jz .check_end                ; ZF=1 → ECX=0 (reached limit) หรือ bytes equal

    ; พบความแตกต่าง
    movzx eax, byte [esi - 1]
    movzx ecx, byte [edi - 1]
    sub eax, ecx
    jg .gt
    jl .lt
    xor eax, eax
    ret
.gt:
    mov eax, 1
    ret
.lt:
    mov eax, -1
    ret

.check_end:
    ; ตรวจสอบว่าทั้งคู่จบที่ null หรือ ECX หมด
    mov al, [esi - 1]
    mov bl, [edi - 1]
    test al, al
    jz .maybe_equal
    ; al != 0 → reached length limit, strings equal up to here
    xor eax, eax
    ret
.maybe_equal:
    test bl, bl
    jnz .lt                      ; str2 ยังมีต่อ → str1 < str2
    xor eax, eax                 ; ทั้งคู่จบที่ null → equal
    ret
```

---

### Technique 3: Vectorized String Operations (SIMD-like)

```nasm
; ===================================================================
; Copy และ Transform string พร้อมกัน (Uppercase conversion)
; ใช้ LODSB + ประมวลผล + STOSB
; ===================================================================
str_toupper:
    ; Input: ESI = source string, EDI = destination buffer
    ; Output: destination ถูก copy และแปลงเป็น uppercase
    cld
.convert_loop:
    lodsb                        ; AL = [ESI++]
    test al, al                  ; null terminator?
    jz .done

    ; แปลง lowercase เป็น uppercase
    cmp al, 'a'
    jb .not_lower
    cmp al, 'z'
    ja .not_lower
    sub al, 32                   ; 'a' - 'A' = 32

.not_lower:
    stosb                        ; [EDI++] = AL
    jmp .convert_loop

.done:
    stosb                        ; เขียน null terminator
    ret
```

---

### Technique 4: String Reverse ด้วย STD

```nasm
; ===================================================================
; Reverse string ใน-place โดยใช้ Direction Flag
; ===================================================================
str_reverse:
    ; Input: ESI = string pointer
    push esi
    push edi

    ; หาความยาว string
    lea edi, [esi]
    mov ecx, 0xFFFFFFFF
    cld
    xor al, al
    repne scasb
    sub edi, 2                   ; EDI = pointer to last char (before null)

    ; ตอนนี้: ESI = start, EDI = end
    ; swap bytes ทีละคู่จาก outside ไป inside
.reverse_loop:
    cmp esi, edi
    jae .done                    ; ถ้า ESI >= EDI เสร็จแล้ว

    ; swap [ESI] กับ [EDI]
    mov al, [esi]
    mov bl, [edi]
    mov [esi], bl
    mov [edi], al
    inc esi
    dec edi
    jmp .reverse_loop

.done:
    pop edi
    pop esi
    ret
```

---

### Technique 5: Efficient memcmp สำหรับขนาดต่างๆ

```nasm
; ===================================================================
; Optimized memcmp ที่ใช้ทั้ง CMPSD และ CMPSB
; ===================================================================
fast_memcmp:
    ; Input: ESI = ptr1, EDI = ptr2, ECX = size
    push ecx
    cld

    ; เปรียบเทียบเป็น dwords ก่อน
    mov eax, ecx
    shr ecx, 2                   ; dword count
    repe cmpsd                   ; เปรียบเทียบ dwords
    jne .dword_diff

    ; เปรียบเทียบ bytes ที่เหลือ
    pop ecx
    and ecx, 3
    repe cmpsb
    jne .byte_diff

    xor eax, eax                 ; equal
    ret

.dword_diff:
    ; ย้อนกลับ 4 bytes แล้วเปรียบเทียบ byte by byte
    sub esi, 4
    sub edi, 4
    mov ecx, 4
    repe cmpsb
    ; ... (same as byte_diff)

.byte_diff:
    movzx eax, byte [esi - 1]
    movzx ecx, byte [edi - 1]
    sub eax, ecx
    pop ecx                      ; clean stack (แต่ ecx value ไม่สำคัญแล้ว)
    ret
```

---

## Exercises (แบบฝึกหัด)

### Exercise 1: Implement strnlen (Basic)

**โจทย์**: เขียนฟังก์ชัน `strnlen` ที่วัดความยาว string แต่ไม่เกิน `n` characters

**Signature**: 
```
; Input:  EDI = string pointer, ECX = max length
; Output: EAX = length (min of actual length and max)
```

**Hint**:
```nasm
; ใช้ REPNE SCASB แต่จำกัด ECX ไว้ก่อน
; หลังจาก repne scasb:
; ถ้าพบ null: ECX = max - (pos+1), ต้องคำนวณ pos
; ถ้าไม่พบ null (ECX=0): return max
```

**Expected behavior**:
```
strnlen("Hello", 10) = 5
strnlen("Hello", 3)  = 3
strnlen("", 10)      = 0
```

---

### Exercise 2: Implement strncpy (Intermediate)

**โจทย์**: เขียน `strncpy` ที่ copy ไม่เกิน `n` characters และ null-pad ถ้าสั้นกว่า

**Signature**:
```
; Input:  EDI = destination, ESI = source, ECX = max chars to copy
; Output: EAX = destination pointer
```

**Hint**:
```nasm
; ขั้นตอน:
; 1. copy จาก ESI ไป EDI จนกว่า null หรือ ECX หมด
; 2. ถ้า copy < n → เติม null bytes ที่เหลือ
; ใช้ LODSB + STOSB + STOSB loop
```

---

### Exercise 3: Count Occurrences of a Character (Intermediate)

**โจทย์**: เขียนฟังก์ชันนับจำนวนครั้งที่ character ปรากฏใน string

**Signature**:
```
; Input:  EDI = string pointer, AL = char to count
; Output: EAX = count
```

**Hint**:
```nasm
; ใช้ SCASB ในลูป (ไม่ใช้ REP prefix เพราะต้องนับ)
; หรือใช้ LODSB + เปรียบเทียบ
count_loop:
    lodsb
    test al, al
    jz done
    cmp al, [char_to_find]   ; เปรียบเทียบกับ target
    jne count_loop
    inc counter
    jmp count_loop
```

---

### Exercise 4: memmove (Advanced)

**โจทย์**: เขียน `memmove` ที่จัดการ overlapping regions ได้อย่างถูกต้อง

**Signature**:
```
; Input:  EDI = destination, ESI = source, ECX = byte count
; Output: EAX = destination pointer
```

**Hint**:
```nasm
; ตรรกะ:
; ถ้า dst < src หรือ dst >= src + count → forward copy (CLD)
; ถ้า dst > src → backward copy (STD)

; Check overlap:
cmp edi, esi
jb forward_copy      ; dst < src → safe forward
; ถ้า dst >= src → ต้องตรวจสอบ
mov eax, esi
add eax, ecx         ; eax = src + count
cmp edi, eax
jae forward_copy     ; dst >= src+count → no overlap
; ถ้าไม่ใช่กรณีข้างบน → backward copy
```

---

### Exercise 5: String Tokenizer (Advanced)

**โจทย์**: เขียน string tokenizer ที่แยก string ด้วย delimiter character

**Signature**:
```
; Input:  ESI = string (หรือ NULL สำหรับ continue), AL = delimiter
; Output: EAX = pointer to next token (null-terminated), 
;               หรือ 0 ถ้าไม่มี token แล้ว
```

**Hint**:
```nasm
; ใช้ static variable เก็บ position ปัจจุบัน
; ขั้นตอน:
; 1. ข้าม delimiters ที่อยู่ตอนต้น (ถ้ามี)
; 2. จำตำแหน่งเริ่มต้น token
; 3. สแกนหา delimiter หรือ null
; 4. ถ้าเจอ delimiter → เขียน null ทับ, เก็บ position ถัดไป
; 5. return pointer ไปยัง token
```

---

## Summary (สรุป)

### ตาราง String Instructions ทั้งหมด

```
┌──────────────────────────────────────────────────────────────────┐
│                    String Instruction Summary                    │
├─────────────┬────────────────┬──────────────┬────────────────────┤
│ Instruction │ Operation      │ Registers    │ Updates            │
├─────────────┼────────────────┼──────────────┼────────────────────┤
│ MOVSB/W/D/Q │ [EDI]=[ESI]    │ ESI, EDI     │ ESI±1,2,4,8 EDI±  │
│ CMPSB/W/D/Q │ [ESI]-[EDI]    │ ESI, EDI     │ ESI±n, EDI±n, FLAGS│
│ SCASB/W/D/Q │ AL/AX/EAX-[EDI]│ AL,EDI       │ EDI±n, FLAGS       │
│ LODSB/W/D/Q │ AL/AX/EAX=[ESI]│ ESI          │ ESI±n              │
│ STOSB/W/D/Q │ [EDI]=AL/AX/EAX│ AL,EDI       │ EDI±n              │
├─────────────┴────────────────┴──────────────┴────────────────────┤
│ n = 1 (B), 2 (W), 4 (D), 8 (Q)                                  │
│ ± = + ถ้า DF=0 (CLD), - ถ้า DF=1 (STD)                         │
└──────────────────────────────────────────────────────────────────┘
```

### ตาราง REP Prefixes

```
┌──────────────────────────────────────────────────────────────────┐
│                      REP Prefix Summary                          │
├──────────────────┬───────────────────────────────────────────────┤
│ Prefix           │ ทำซ้ำในขณะที่...                               │
├──────────────────┼───────────────────────────────────────────────┤
│ REP              │ ECX != 0                                      │
│ REPE / REPZ      │ ECX != 0 AND ZF = 1 (bytes เท่ากัน)          │
│ REPNE / REPNZ    │ ECX != 0 AND ZF = 0 (bytes ไม่เท่ากัน)       │
├──────────────────┴───────────────────────────────────────────────┤
│ ใช้ได้กับ: REP → MOVS, STOS, LODS                               │
│             REPE/REPNE → CMPS, SCAS                              │
└──────────────────────────────────────────────────────────────────┘
```

### Common Use Cases

```
┌──────────────────────────────────────────────────────────────────┐
│                    Common Use Cases                              │
├──────────────────────────┬───────────────────────────────────────┤
│ Task                     │ Instruction                           │
├──────────────────────────┼───────────────────────────────────────┤
│ memcpy / strcpy          │ REP MOVSB (หรือ MOVSD + MOVSB)       │
│ memset / bzero           │ REP STOSB (หรือ STOSD + STOSB)       │
│ strlen                   │ REPNE SCASB + คำนวณ                  │
│ strcmp / memcmp          │ REPE CMPSB                            │
│ strchr / memchr          │ REPNE SCASB                           │
│ Transform string         │ LODSB + process + STOSB loop          │
│ Overlap-safe memmove     │ STD + REP MOVSB (ถ้า dst > src)       │
└──────────────────────────┴───────────────────────────────────────┘
```

### Performance Tips

1. **ใช้ MOVSD/STOSD แทน MOVSB/STOSB** เมื่อ copy/fill ข้อมูลขนาดใหญ่
2. **จัด alignment** ของ memory ก่อนใช้ string instructions
3. **REP MOVSD** เร็วกว่า **manual loop** หลายเท่า
4. **ในโปรเซสเซอร์สมัยใหม่** REP MOVSB ถูก optimize โดย CPU (ERMSB - Enhanced REP MOVSB/STOSB)
5. **ตั้ง CLD เสมอ** ก่อน string instructions เพื่อความปลอดภัย

### Key Rules to Remember

```
1. CLD → forward  (ESI/EDI ++)
   STD → backward (ESI/EDI --)
   
2. SCAS ใช้ EDI (ไม่ใช่ ESI!)
   LODS ใช้ ESI
   STOS ใช้ EDI
   MOVS ใช้ทั้ง ESI (source) และ EDI (destination)
   CMPS ใช้ทั้ง ESI และ EDI

3. REP ใช้กับ: MOVS, STOS, LODS
   REPE/REPNE ใช้กับ: CMPS, SCAS

4. ECX เป็น counter สำหรับ REP prefix

5. Overlapping copy ที่ dst > src → ต้องใช้ backward copy (STD)
```

---

*Part 020 จบแล้ว - ต่อไปใน Part 021: Floating Point Instructions (FPU)*

**การเตรียมตัวสำหรับ Part ถัดไป**:
- ทบทวนการใช้ registers ESI, EDI, ECX
- ฝึก implement string functions ด้วยตัวเอง
- ลองเปรียบเทียบความเร็วระหว่าง string instructions กับ loop ปกติ
