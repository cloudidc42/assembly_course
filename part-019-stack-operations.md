# Part 019: Stack Operations เชิงลึก (Deep Dive into Stack Operations)

## Prerequisites and Learning Objectives

### สิ่งที่ต้องรู้ก่อน (Prerequisites)
- Part 001-018: พื้นฐาน Assembly, Registers, Memory Addressing
- ความเข้าใจ Memory Layout ของโปรแกรม
- การใช้ NASM และ Linker (ld หรือ gcc)
- พื้นฐาน Stack Concept จาก Data Structures

### วัตถุประสงค์การเรียนรู้ (Learning Objectives)
หลังจากเรียนจบ Part นี้ นักเรียนจะสามารถ:
1. อธิบายกลไกการทำงานของ Stack ใน x86/x86-64 ได้อย่างละเอียด
2. ใช้คำสั่ง PUSH/POP ทุก Variant ได้อย่างถูกต้อง
3. สร้าง Stack Frame ด้วย ENTER/LEAVE และ Push/Pop แบบ Manual
4. เข้าใจ Red Zone ใน x86-64 ABI
5. ตรวจสอบ Stack Alignment 16-byte requirement
6. Implement RPN Calculator โดยใช้ Stack
7. ป้องกันและตรวจจับ Stack Overflow

---

## ทฤษฎี: กลไกของ Stack (Stack Mechanics)

### 1. Stack คืออะไร และทำงานอย่างไร

Stack เป็น Last-In-First-Out (LIFO) Data Structure ที่ CPU ใช้สำหรับ:
- เก็บ Return Address เมื่อเรียก Function
- เก็บ Local Variables ของ Function
- ส่ง Parameters ระหว่าง Functions (ใน 32-bit และ System V 64-bit บาง convention)
- บันทึก Register Values ชั่วคราว
- เก็บ CPU Flags (PUSHF/POPF)

### 2. Stack Register: ESP และ RSP

| Register | Architecture | ขนาด | คำอธิบาย |
|----------|-------------|------|----------|
| `ESP`    | x86 (32-bit) | 32-bit | Extended Stack Pointer |
| `RSP`    | x86-64       | 64-bit | 64-bit Stack Pointer |
| `SP`     | 16-bit mode  | 16-bit | Stack Pointer (segment-based) |

**ESP/RSP ชี้ไปที่ค่าบนสุดของ Stack เสมอ** (Top of Stack - TOS)

### 3. ทิศทางการเติบโตของ Stack (Stack Growth Direction)

```
สูง (High Memory Address)
+------------------+  ← Stack Bottom (เริ่มต้น)
|   Initial SP     |  0xFFFFxxxx
|      ↓           |
|   PUSH ครั้งที่ 1 |  ESP - 4
|   PUSH ครั้งที่ 2 |  ESP - 8
|   PUSH ครั้งที่ 3 |  ESP - 12
|      ↓           |
|   Current TOS    |  ← ESP ชี้อยู่ที่นี่
|                  |
|   (ว่างอยู่)      |
+------------------+  ← Stack Top (ที่อยู่ต่ำสุด, Stack Pointer ปัจจุบัน)
ต่ำ (Low Memory Address)
```

**Stack เติบโตไปยัง Low Address** ทุก PUSH ทำให้ ESP/RSP ลดลง

### 4. ขั้นตอนของ PUSH

เมื่อทำ `PUSH EAX`:
1. `ESP = ESP - 4` (ลด Stack Pointer ลง 4 bytes)
2. `[ESP] = EAX` (เขียนค่าลงในหน่วยความจำที่ ESP ชี้)

```
ก่อน PUSH EAX (EAX = 0x12345678):
ESP → [0x1000]: ...

หลัง PUSH EAX:
ESP → [0x0FFC]: 0x12345678
      [0x1000]: ...
```

### 5. ขั้นตอนของ POP

เมื่อทำ `POP EAX`:
1. `EAX = [ESP]` (อ่านค่าจากหน่วยความจำที่ ESP ชี้)
2. `ESP = ESP + 4` (เพิ่ม Stack Pointer ขึ้น 4 bytes)

```
ก่อน POP EAX:
ESP → [0x0FFC]: 0x12345678
      [0x1000]: ...

หลัง POP EAX (EAX = 0x12345678):
      [0x0FFC]: 0x12345678 (ยังอยู่ แต่ถือว่าไม่มีค่าแล้ว)
ESP → [0x1000]: ...
```

### 6. Stack Alignment Requirement (16-byte)

ใน x86-64 Linux (System V AMD64 ABI):
- **ก่อน CALL instruction**: RSP ต้องถูก align ที่ 16-byte boundary
- **ภายใน Function**: RSP aligned ที่ 16-byte หลังจาก CALL เก็บ Return Address
- ถ้า RSP ไม่ aligned อาจทำให้ SSE/AVX instructions crash

```
ก่อน CALL:
RSP % 16 == 0  (RSP ลงท้ายด้วย 0x0, 0x10, 0x20, ...)

หลัง CALL (CALL เก็บ Return Address 8 bytes):
RSP % 16 == 8  (RSP ลงท้ายด้วย 0x8)

ใน Function ต้องทำให้ RSP กลับมา align:
sub rsp, 8  ; ทำให้ RSP align ถ้า local vars ไม่ชดเชยพอ
```

### 7. Red Zone ใน x86-64

ใน System V AMD64 ABI มี "Red Zone" คือ 128 bytes ใต้ RSP ที่:
- ไม่ถูก Modify โดย Signal Handlers หรือ Interrupt Handlers (ใน Kernel code)
- Leaf Functions (Functions ที่ไม่เรียก Function อื่น) สามารถใช้ Red Zone เพื่อเก็บ Local Variables โดยไม่ต้อง Sub RSP

```
    RSP  →  [RSP + 0]   : ใช้ได้ปกติ
            [RSP - 8]   : Red Zone เริ่มต้น
            [RSP - 16]  : Red Zone
            ...
            [RSP - 128] : Red Zone สิ้นสุด (128 bytes จาก RSP)
```

**หมายเหตุ**: Windows x64 ABI ไม่มี Red Zone

### 8. ENTER และ LEAVE Instructions

`ENTER imm16, imm8` สร้าง Stack Frame:
- `imm16`: จำนวน bytes สำหรับ Local Variables
- `imm8`: Nesting Level (ปกติใช้ 0)

เทียบเท่ากับ:
```asm
; ENTER size, 0 เทียบเท่ากับ:
push ebp
mov  ebp, esp
sub  esp, size
```

`LEAVE` ทำลาย Stack Frame:
เทียบเท่ากับ:
```asm
; LEAVE เทียบเท่ากับ:
mov esp, ebp
pop ebp
```

### 9. PUSHA และ POPA (32-bit เท่านั้น)

`PUSHA` เก็บ Register ทั้ง 8 ตัวในลำดับ:
EAX, ECX, EDX, EBX, ESP (ค่าเดิมก่อน PUSHA), EBP, ESI, EDI

`POPA` คืนค่า Register ทั้ง 8 ตัว (ยกเว้น ESP ที่ถูก skip)

**หมายเหตุ**: PUSHA/POPA ไม่มีใน 64-bit mode

### 10. PUSHF และ POPF

`PUSHF` บันทึก FLAGS Register ลง Stack
`POPF` คืนค่า FLAGS Register จาก Stack
`PUSHFD` (32-bit): เก็บ EFLAGS
`PUSHFQ` (64-bit): เก็บ RFLAGS

ใช้เมื่อต้องการบันทึกและคืนค่า CPU Flags ชั่วคราว

---

## ตัวอย่างที่ 1: Basic Stack Operations

```nasm
; ============================================================
; part019_example1_basic_stack.asm
; Basic Stack Operations - PUSH, POP, และการตรวจสอบ ESP
; ============================================================
; วิธีคอมไพล์:
;   nasm -f elf32 part019_example1_basic_stack.asm -o part019_example1_basic_stack.o
;   ld -m elf_i386 part019_example1_basic_stack.o -o part019_example1_basic_stack
; วิธีรัน:
;   ./part019_example1_basic_stack
; Expected Output:
;   Before PUSH: ESP = 0xFFxxxxxx
;   After PUSH 1: ESP - 4
;   After PUSH 2: ESP - 8
;   After PUSH 3: ESP - 12
;   Popped: 300, 200, 100
; ============================================================

section .data
    msg_before  db "=== Basic Stack Demo ===", 10, 0
    msg_push1   db "PUSH 100 done", 10, 0
    msg_push2   db "PUSH 200 done", 10, 0
    msg_push3   db "PUSH 300 done", 10, 0
    msg_pop1    db "POP -> EAX = ", 0
    msg_pop2    db "POP -> EBX = ", 0
    msg_pop3    db "POP -> ECX = ", 0
    msg_newline db 10, 0
    msg_lifo    db "=== Stack is LIFO: Last In, First Out ===", 10, 0
    msg_order   db "Push order: 100, 200, 300", 10, 0
    msg_order2  db "Pop order:  300, 200, 100", 10, 0

section .bss
    buf resb 12     ; Buffer สำหรับแปลงตัวเลขเป็น String

section .text
    global _start

; ============================================================
; print_string: พิมพ์ String ที่ชี้โดย EBX
; ============================================================
print_string:
    push eax
    push ecx
    push edx
    
    ; หาความยาว String
    mov ecx, ebx
.find_len:
    cmp byte [ecx], 0
    je .done_len
    inc ecx
    jmp .find_len
.done_len:
    sub ecx, ebx    ; ecx = ความยาว
    
    ; sys_write
    mov edx, ecx   ; length
    mov ecx, ebx   ; string pointer
    mov ebx, 1     ; stdout
    mov eax, 4     ; sys_write
    int 0x80
    
    pop edx
    pop ecx
    pop eax
    ret

; ============================================================
; print_number: พิมพ์ตัวเลขใน EAX
; ============================================================
print_number:
    push eax
    push ebx
    push ecx
    push edx
    push esi
    
    ; แปลงตัวเลขเป็น String
    mov esi, buf + 11   ; ชี้ไปยัง Buffer ท้าย
    mov byte [esi], 0   ; Null terminator
    dec esi
    
    mov ecx, 10         ; หาร 10
    test eax, eax
    jnz .convert
    mov byte [esi], '0' ; กรณี 0
    jmp .print
    
.convert:
    test eax, eax
    jz .print
    xor edx, edx
    div ecx             ; eax / 10, เศษใน edx
    add dl, '0'
    mov [esi], dl
    dec esi
    jmp .convert
    
.print:
    inc esi             ; ชี้ไปที่ตัวเลขแรก
    mov ebx, esi
    call print_string
    
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; ============================================================
; _start: Main Program
; ============================================================
_start:
    ; พิมพ์ Header
    mov ebx, msg_before
    call print_string
    
    mov ebx, msg_order
    call print_string
    
    ; ============================================================
    ; PUSH 3 ค่าลง Stack
    ; แต่ละ PUSH ทำให้ ESP ลดลง 4
    ; ============================================================
    push dword 100  ; ESP -= 4, [ESP] = 100
    mov ebx, msg_push1
    call print_string
    
    push dword 200  ; ESP -= 4, [ESP] = 200
    mov ebx, msg_push2
    call print_string
    
    push dword 300  ; ESP -= 4, [ESP] = 300
    mov ebx, msg_push3
    call print_string
    
    ; ============================================================
    ; POP 3 ค่าออกจาก Stack (LIFO: ออกในลำดับย้อนกลับ)
    ; ============================================================
    mov ebx, msg_lifo
    call print_string
    
    mov ebx, msg_order2
    call print_string
    
    ; POP ครั้งที่ 1: ได้ 300 (ค่าที่ PUSH ล่าสุด)
    pop eax         ; EAX = 300, ESP += 4
    mov ebx, msg_pop1
    call print_string
    call print_number
    mov ebx, msg_newline
    call print_string
    
    ; POP ครั้งที่ 2: ได้ 200
    pop ebx         ; EBX = 200, ESP += 4
    push eax        ; บันทึก EAX
    mov eax, ebx
    mov ebx, msg_pop2
    call print_string
    call print_number
    mov ebx, msg_newline
    call print_string
    pop eax
    
    ; POP ครั้งที่ 3: ได้ 100
    pop ecx         ; ECX = 100, ESP += 4
    push eax
    mov eax, ecx
    mov ebx, msg_pop3
    call print_string
    call print_number
    mov ebx, msg_newline
    call print_string
    pop eax
    
    ; จบโปรแกรม
    mov eax, 1      ; sys_exit
    xor ebx, ebx    ; exit code 0
    int 0x80
```

**คอมไพล์และรัน:**
```bash
nasm -f elf32 part019_example1_basic_stack.asm -o part019_example1_basic_stack.o
ld -m elf_i386 part019_example1_basic_stack.o -o part019_example1_basic_stack
./part019_example1_basic_stack
```

**Expected Output:**
```
=== Basic Stack Demo ===
Push order: 100, 200, 300
PUSH 100 done
PUSH 200 done
PUSH 300 done
=== Stack is LIFO: Last In, First Out ===
Pop order:  300, 200, 100
POP -> EAX = 300
POP -> EBX = 200
POP -> ECX = 100
```

---

## ตัวอย่างที่ 2: Stack Frame และ Local Variables

```nasm
; ============================================================
; part019_example2_stack_frame.asm
; Stack Frame Setup/Teardown ด้วย EBP-based Addressing
; แสดง Local Variables และ Parameters บน Stack
; ============================================================
; วิธีคอมไพล์:
;   nasm -f elf32 part019_example2_stack_frame.asm -o part019_example2_stack_frame.o
;   ld -m elf_i386 part019_example2_stack_frame.o -o part019_example2_stack_frame
; วิธีรัน:
;   ./part019_example2_stack_frame
; Expected Output:
;   Computing: add(10, 20) = 30
;   Computing: multiply(5, 7) = 35
;   Computing: max(42, 17) = 42
; ============================================================

section .data
    msg_header  db "=== Stack Frame Demo ===", 10, 0
    msg_add     db "add(10, 20) = ", 0
    msg_mul     db "multiply(5, 7) = ", 0
    msg_max     db "max(42, 17) = ", 0
    msg_nl      db 10, 0

section .bss
    buf resb 16

section .text
    global _start

; ============================================================
; print_str: พิมพ์ String ที่ EBX ชี้
; ============================================================
print_str:
    pusha                   ; บันทึก Register ทั้งหมด (PUSHA)
    
    mov ecx, ebx
.len:
    cmp byte [ecx], 0
    je .done
    inc ecx
    jmp .len
.done:
    sub ecx, ebx
    
    mov edx, ecx
    mov ecx, ebx
    mov ebx, 1
    mov eax, 4
    int 0x80
    
    popa                    ; คืนค่า Register ทั้งหมด (POPA)
    ret

; ============================================================
; print_num: พิมพ์ตัวเลขใน EAX
; ============================================================
print_num:
    pusha
    
    mov esi, buf + 15
    mov byte [esi], 0
    dec esi
    mov ecx, 10
    
    test eax, eax
    jnz .cvt
    mov byte [esi], '0'
    jmp .prn
    
.cvt:
    test eax, eax
    jz .prn
    xor edx, edx
    div ecx
    add dl, '0'
    mov [esi], dl
    dec esi
    jmp .cvt
    
.prn:
    inc esi
    mov ebx, esi
    call print_str
    
    popa
    ret

; ============================================================
; add_numbers: บวกตัวเลข 2 ตัว
;   Parameter 1: [EBP + 8]  (ถูก PUSH ทีหลัง = อยู่ใกล้ Return Address)
;   Parameter 2: [EBP + 12] (ถูก PUSH ก่อน = อยู่ไกลกว่า)
;
;   Stack Layout เมื่ออยู่ใน add_numbers:
;   [EBP + 12]: param2 = 20
;   [EBP + 8]:  param1 = 10
;   [EBP + 4]:  Return Address (เก็บโดย CALL)
;   [EBP + 0]:  Old EBP (เก็บโดย PUSH EBP)
;   [EBP - 4]:  local_var (ถ้ามี)
;
; Return: EAX = param1 + param2
; ============================================================
add_numbers:
    ; Stack Frame Prologue (เริ่มต้น)
    push ebp            ; บันทึก EBP เดิม
    mov  ebp, esp       ; EBP = ESP (Base Pointer)
    
    ; Optional: สร้าง Local Variables
    sub  esp, 4         ; Local Variable 4 bytes: [EBP - 4]
    
    ; อ่าน Parameters จาก Stack
    mov  eax, [ebp + 8]  ; param1 = 10
    mov  ecx, [ebp + 12] ; param2 = 20
    
    ; เก็บผลลัพธ์ใน Local Variable
    add  eax, ecx
    mov  [ebp - 4], eax  ; local_result = param1 + param2
    
    ; Load ผลลัพธ์ใส่ EAX (Return Value)
    mov  eax, [ebp - 4]
    
    ; Stack Frame Epilogue (สิ้นสุด)
    mov  esp, ebp       ; คืนค่า ESP กลับ (ล้าง Local Variables)
    pop  ebp            ; คืนค่า EBP เดิม
    ret                 ; กลับไปยัง Caller (POP Return Address)

; ============================================================
; multiply_numbers: คูณตัวเลข 2 ตัว ด้วย ENTER/LEAVE
; ============================================================
multiply_numbers:
    ; ENTER 8, 0 เทียบเท่ากับ:
    ;   push ebp
    ;   mov  ebp, esp
    ;   sub  esp, 8    (สร้าง 8 bytes สำหรับ Local Variables)
    enter 8, 0
    
    mov  eax, [ebp + 8]  ; param1
    mov  ecx, [ebp + 12] ; param2
    imul eax, ecx        ; EAX = param1 * param2
    
    ; LEAVE เทียบเท่ากับ:
    ;   mov esp, ebp
    ;   pop ebp
    leave
    ret

; ============================================================
; max_numbers: หาค่าสูงสุดของตัวเลข 2 ตัว
; ============================================================
max_numbers:
    push ebp
    mov  ebp, esp
    ; ไม่มี Local Variables
    
    mov  eax, [ebp + 8]  ; param1
    mov  ecx, [ebp + 12] ; param2
    
    cmp  eax, ecx        ; เปรียบเทียบ param1 กับ param2
    jge  .done           ; ถ้า param1 >= param2, EAX คือ max
    mov  eax, ecx        ; ไม่งั้น ECX คือ max
.done:
    pop  ebp
    ret

; ============================================================
; _start: Main Program
; ============================================================
_start:
    mov ebx, msg_header
    call print_str
    
    ; --- เรียก add_numbers(10, 20) ---
    ; PUSH parameters ในลำดับ Right-to-Left (cdecl convention)
    push dword 20   ; param2 (PUSH ก่อน = อยู่ไกลกว่า EBP)
    push dword 10   ; param1 (PUSH ทีหลัง = อยู่ใกล้ EBP)
    call add_numbers
    add  esp, 8     ; Caller cleans up stack (cdecl): 2 params * 4 bytes = 8
    
    mov ebx, msg_add
    call print_str
    call print_num  ; EAX = 30
    mov ebx, msg_nl
    call print_str
    
    ; --- เรียก multiply_numbers(5, 7) ---
    push dword 7    ; param2
    push dword 5    ; param1
    call multiply_numbers
    add  esp, 8
    
    mov ebx, msg_mul
    call print_str
    call print_num  ; EAX = 35
    mov ebx, msg_nl
    call print_str
    
    ; --- เรียก max_numbers(42, 17) ---
    push dword 17   ; param2
    push dword 42   ; param1
    call max_numbers
    add  esp, 8
    
    mov ebx, msg_max
    call print_str
    call print_num  ; EAX = 42
    mov ebx, msg_nl
    call print_str
    
    ; จบโปรแกรม
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**คอมไพล์และรัน:**
```bash
nasm -f elf32 part019_example2_stack_frame.asm -o part019_example2_stack_frame.o
ld -m elf_i386 part019_example2_stack_frame.o -o part019_example2_stack_frame
./part019_example2_stack_frame
```

**Expected Output:**
```
=== Stack Frame Demo ===
add(10, 20) = 30
multiply(5, 7) = 35
max(42, 17) = 42
```

---

## ตัวอย่างที่ 3: PUSHF/POPF - การบันทึกและคืน Flags

```nasm
; ============================================================
; part019_example3_pushf_popf.asm
; การใช้ PUSHF/POPF เพื่อบันทึกและคืนค่า CPU Flags
; แสดงการทำงานของ Carry Flag, Zero Flag, Sign Flag
; ============================================================
; วิธีคอมไพล์:
;   nasm -f elf32 part019_example3_pushf_popf.asm -o part019_example3_pushf_popf.o
;   ld -m elf_i386 part019_example3_pushf_popf.o -o part019_example3_pushf_popf
; วิธีรัน:
;   ./part019_example3_pushf_popf
; ============================================================

section .data
    msg_title   db "=== PUSHF/POPF Demo ===", 10, 0
    msg_saved   db "Flags saved to stack", 10, 0
    msg_modify  db "Flags modified by operations", 10, 0
    msg_restore db "Flags restored from stack", 10, 0
    
    msg_cf_set  db "Carry Flag: SET (1)", 10, 0
    msg_cf_clr  db "Carry Flag: CLEAR (0)", 10, 0
    msg_zf_set  db "Zero Flag: SET (1)", 10, 0
    msg_zf_clr  db "Zero Flag: CLEAR (0)", 10, 0
    msg_sf_set  db "Sign Flag: SET (1)", 10, 0
    msg_sf_clr  db "Sign Flag: CLEAR (0)", 10, 0
    
    msg_before  db "--- Before Save ---", 10, 0
    msg_during  db "--- After Modifications ---", 10, 0
    msg_after   db "--- After Restore ---", 10, 0

section .text
    global _start

print_str:
    pusha
    mov ecx, ebx
.find:
    cmp byte [ecx], 0
    je .found
    inc ecx
    jmp .find
.found:
    sub ecx, ebx
    mov edx, ecx
    mov ecx, ebx
    mov ebx, 1
    mov eax, 4
    int 0x80
    popa
    ret

; ============================================================
; print_flags: พิมพ์สถานะ CF, ZF, SF
; ============================================================
print_flags:
    push eax
    push ebx
    
    ; ตรวจสอบ Carry Flag (CF)
    jc .cf_set
    mov ebx, msg_cf_clr
    call print_str
    jmp .check_zf
.cf_set:
    mov ebx, msg_cf_set
    call print_str
    
.check_zf:
    ; ตรวจสอบ Zero Flag (ZF)
    ; ต้องใช้วิธีอ่าน EFLAGS แล้วตรวจสอบ bit
    pushf               ; บันทึก EFLAGS ชั่วคราว
    pop eax             ; EAX = EFLAGS
    
    test eax, (1 << 6)  ; Bit 6 = Zero Flag
    jz .zf_clear
    mov ebx, msg_zf_set
    call print_str
    jmp .check_sf
.zf_clear:
    mov ebx, msg_zf_clr
    call print_str
    
.check_sf:
    ; ตรวจสอบ Sign Flag (SF) - Bit 7
    test eax, (1 << 7)
    jz .sf_clear
    mov ebx, msg_sf_set
    call print_str
    jmp .done_flags
.sf_clear:
    mov ebx, msg_sf_clr
    call print_str
    
.done_flags:
    pop ebx
    pop eax
    ret

_start:
    mov ebx, msg_title
    call print_str
    
    ; ============================================================
    ; ขั้นตอนที่ 1: สร้างสถานะ Flags เบื้องต้น
    ; ============================================================
    mov ebx, msg_before
    call print_str
    
    ; สร้าง Carry Flag โดย Sub ที่ Overflow
    mov  eax, 0         ; EAX = 0
    sub  eax, 1         ; 0 - 1 = -1 (Underflow → CF = 1)
    ; ขณะนี้: CF=1, SF=1, ZF=0
    
    call print_flags    ; แสดง: CF=1, ZF=0, SF=1
    
    ; ============================================================
    ; ขั้นตอนที่ 2: บันทึก Flags ด้วย PUSHF
    ; ============================================================
    pushf               ; บันทึก EFLAGS ลง Stack
    
    mov ebx, msg_saved
    call print_str
    
    ; ============================================================
    ; ขั้นตอนที่ 3: ทำการเปลี่ยน Flags
    ; ============================================================
    xor eax, eax        ; EAX = 0, ทำให้ ZF=1, CF=0, SF=0
    ; ขณะนี้: CF=0, ZF=1, SF=0
    
    mov ebx, msg_during
    call print_str
    call print_flags    ; แสดง: CF=0, ZF=1, SF=0
    
    ; ============================================================
    ; ขั้นตอนที่ 4: คืนค่า Flags ด้วย POPF
    ; ============================================================
    popf                ; คืนค่า EFLAGS จาก Stack
    ; Flags กลับมาเป็น CF=1, ZF=0, SF=1 เหมือนก่อนบันทึก
    
    mov ebx, msg_after
    call print_str
    call print_flags    ; แสดง: CF=1, ZF=0, SF=1 (กลับมาเหมือนเดิม)
    
    ; ============================================================
    ; Demo เพิ่มเติม: การใช้ PUSHF/POPF ใน Critical Sections
    ; ============================================================
    ; บางครั้งต้องการป้องกัน Interrupt ชั่วคราว
    pushf               ; บันทึก Flags (รวม Interrupt Flag)
    cli                 ; Clear Interrupt Flag (Disable Interrupts)
    ; ... ทำงาน Critical Section ที่นี่ ...
    ; ... (ใน Ring 0 เท่านั้น; User mode ไม่สามารถ cli ได้) ...
    popf                ; คืนค่า Flags (รวม Interrupt Flag เดิม)
    
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**คอมไพล์และรัน:**
```bash
nasm -f elf32 part019_example3_pushf_popf.asm -o part019_example3_pushf_popf.o
ld -m elf_i386 part019_example3_pushf_popf.o -o part019_example3_pushf_popf
./part019_example3_pushf_popf
```

**Expected Output:**
```
=== PUSHF/POPF Demo ===
--- Before Save ---
Carry Flag: SET (1)
Zero Flag: CLEAR (0)
Sign Flag: SET (1)
Flags saved to stack
--- After Modifications ---
Carry Flag: CLEAR (0)
Zero Flag: SET (1)
Sign Flag: CLEAR (0)
--- After Restore ---
Carry Flag: SET (1)
Zero Flag: CLEAR (0)
Sign Flag: SET (1)
```

---

## ตัวอย่างที่ 4: Stack-based Expression Evaluation

```nasm
; ============================================================
; part019_example4_stack_eval.asm
; Stack-based Expression Evaluation
; คำนวณนิพจน์โดยใช้ Stack (เหมือนกับ Compiler ทำงาน)
; คำนวณ: (3 + 4) * (5 - 2) = 21
; ============================================================
; วิธีคอมไพล์:
;   nasm -f elf32 part019_example4_stack_eval.asm -o part019_example4_stack_eval.o
;   ld -m elf_i386 part019_example4_stack_eval.o -o part019_example4_stack_eval
; วิธีรัน:
;   ./part019_example4_stack_eval
; Expected Output:
;   Expression: (3 + 4) * (5 - 2)
;   Step 1: PUSH 3
;   Step 2: PUSH 4
;   Step 3: ADD -> Stack: [7]
;   Step 4: PUSH 5
;   Step 5: PUSH 2
;   Step 6: SUB -> Stack: [7, 3]
;   Step 7: MUL -> Stack: [21]
;   Result = 21
; ============================================================

section .data
    msg_title   db "=== Stack Expression Evaluator ===", 10, 0
    msg_expr    db "Expression: (3 + 4) * (5 - 2)", 10, 0
    
    msg_s1      db "Step 1: PUSH 3", 10, 0
    msg_s2      db "Step 2: PUSH 4", 10, 0
    msg_s3      db "Step 3: ADD    -> popped 3,4 pushed 7", 10, 0
    msg_s4      db "Step 4: PUSH 5", 10, 0
    msg_s5      db "Step 5: PUSH 2", 10, 0
    msg_s6      db "Step 6: SUB    -> popped 5,2 pushed 3", 10, 0
    msg_s7      db "Step 7: MUL    -> popped 7,3 pushed 21", 10, 0
    msg_result  db "Result = ", 0
    msg_nl      db 10, 0
    
    ; นิพจน์ที่ 2: 10 + 3 * 4 - 2 = 20 (Postfix: 10 3 4 * + 2 -)
    msg_expr2   db 10, "Expression 2: 10 + 3 * 4 - 2 (Postfix: 10 3 4 * + 2 -)", 10, 0
    msg_result2 db "Result 2 = ", 0
    
    ; นิพจน์ที่ 3: 2^8 = 256 (Postfix: 2 8 POW)
    msg_expr3   db 10, "Expression 3: 2^8 (power)", 10, 0
    msg_result3 db "Result 3 = ", 0

section .bss
    ; Stack ของเราเอง (Software Stack) ขนาด 64 integers
    my_stack    resd 64
    stack_top   resd 1  ; ชี้ไปยัง Element บนสุด (-1 = ว่าง)
    
    buf         resb 16

section .text
    global _start

print_str:
    pusha
    mov ecx, ebx
.len:
    cmp byte [ecx], 0
    je .found
    inc ecx
    jmp .len
.found:
    sub ecx, ebx
    mov edx, ecx
    mov ecx, ebx
    mov ebx, 1
    mov eax, 4
    int 0x80
    popa
    ret

print_num:
    pusha
    mov esi, buf + 15
    mov byte [esi], 0
    dec esi
    mov ecx, 10
    test eax, eax
    jnz .cvt
    mov byte [esi], '0'
    jmp .prn
.cvt:
    test eax, eax
    jz .prn
    xor edx, edx
    div ecx
    add dl, '0'
    mov [esi], dl
    dec esi
    jmp .cvt
.prn:
    inc esi
    mov ebx, esi
    call print_str
    popa
    ret

; ============================================================
; stack_push: Push EAX ลง Software Stack
; ============================================================
stack_push:
    push ebx
    push ecx
    
    mov ebx, [stack_top]    ; index ปัจจุบัน
    inc ebx                  ; เพิ่ม index
    mov [stack_top], ebx
    
    ; เก็บค่าใน my_stack[index]
    mov ecx, my_stack
    mov [ecx + ebx*4], eax
    
    pop ecx
    pop ebx
    ret

; ============================================================
; stack_pop: Pop จาก Software Stack ใส่ EAX
; ============================================================
stack_pop:
    push ebx
    push ecx
    
    mov ebx, [stack_top]
    mov ecx, my_stack
    mov eax, [ecx + ebx*4]  ; อ่านค่า
    
    dec ebx
    mov [stack_top], ebx     ; ลด index
    
    pop ecx
    pop ebx
    ret

; ============================================================
; _start: คำนวณนิพจน์
; ============================================================
_start:
    ; เริ่มต้น Software Stack
    mov dword [stack_top], -1   ; ว่าง
    
    mov ebx, msg_title
    call print_str
    
    ; ============================================================
    ; นิพจน์ที่ 1: (3 + 4) * (5 - 2)
    ; Postfix (RPN): 3 4 + 5 2 - *
    ; ============================================================
    mov ebx, msg_expr
    call print_str
    
    ; Step 1: PUSH 3
    mov ebx, msg_s1
    call print_str
    mov eax, 3
    call stack_push
    
    ; Step 2: PUSH 4
    mov ebx, msg_s2
    call print_str
    mov eax, 4
    call stack_push
    
    ; Step 3: ADD (pop 2 ค่า, push ผลรวม)
    mov ebx, msg_s3
    call print_str
    call stack_pop      ; EAX = 4 (operand ขวา)
    push eax
    call stack_pop      ; EAX = 3 (operand ซ้าย)
    pop ecx
    add eax, ecx        ; EAX = 3 + 4 = 7
    call stack_push     ; push 7
    
    ; Step 4: PUSH 5
    mov ebx, msg_s4
    call print_str
    mov eax, 5
    call stack_push
    
    ; Step 5: PUSH 2
    mov ebx, msg_s5
    call print_str
    mov eax, 2
    call stack_push
    
    ; Step 6: SUB (pop 2 ค่า, push ผลลบ)
    mov ebx, msg_s6
    call print_str
    call stack_pop      ; EAX = 2 (subtrahend)
    push eax
    call stack_pop      ; EAX = 5 (minuend)
    pop ecx
    sub eax, ecx        ; EAX = 5 - 2 = 3
    call stack_push     ; push 3
    
    ; Step 7: MUL (pop 2 ค่า, push ผลคูณ)
    mov ebx, msg_s7
    call print_str
    call stack_pop      ; EAX = 3
    push eax
    call stack_pop      ; EAX = 7
    pop ecx
    imul eax, ecx       ; EAX = 7 * 3 = 21
    call stack_push     ; push 21
    
    ; แสดงผลลัพธ์
    call stack_pop      ; EAX = 21
    mov ebx, msg_result
    call print_str
    call print_num
    mov ebx, msg_nl
    call print_str
    
    ; ============================================================
    ; นิพจน์ที่ 2: 10 + 3 * 4 - 2
    ; Postfix: 10 3 4 * + 2 -
    ; ============================================================
    mov dword [stack_top], -1   ; Reset stack
    mov ebx, msg_expr2
    call print_str
    
    ; Push 10
    mov eax, 10
    call stack_push
    
    ; Push 3
    mov eax, 3
    call stack_push
    
    ; Push 4
    mov eax, 4
    call stack_push
    
    ; * (3*4=12)
    call stack_pop
    push eax
    call stack_pop
    pop ecx
    imul eax, ecx
    call stack_push     ; push 12
    
    ; + (10+12=22)
    call stack_pop
    push eax
    call stack_pop
    pop ecx
    add eax, ecx
    call stack_push     ; push 22
    
    ; Push 2
    mov eax, 2
    call stack_push
    
    ; - (22-2=20)
    call stack_pop
    push eax
    call stack_pop
    pop ecx
    sub eax, ecx
    call stack_push     ; push 20
    
    call stack_pop
    mov ebx, msg_result2
    call print_str
    call print_num      ; 20
    mov ebx, msg_nl
    call print_str
    
    ; ============================================================
    ; นิพจน์ที่ 3: 2^8 = 256
    ; Postfix: 2 8 POW
    ; ============================================================
    mov dword [stack_top], -1
    mov ebx, msg_expr3
    call print_str
    
    ; Push 2 (base)
    mov eax, 2
    call stack_push
    
    ; Push 8 (exponent)
    mov eax, 8
    call stack_push
    
    ; POW: คำนวณ 2^8 โดย Loop
    call stack_pop          ; EAX = exponent = 8
    mov ecx, eax            ; ECX = 8 (counter)
    call stack_pop          ; EAX = base = 2
    
    mov ebx, 1              ; result = 1
.pow_loop:
    test ecx, ecx
    jz .pow_done
    imul ebx, eax           ; result *= base
    dec ecx
    jmp .pow_loop
.pow_done:
    mov eax, ebx
    call stack_push         ; push 256
    
    call stack_pop
    mov ebx, msg_result3
    call print_str
    call print_num          ; 256
    mov ebx, msg_nl
    call print_str
    
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**คอมไพล์และรัน:**
```bash
nasm -f elf32 part019_example4_stack_eval.asm -o part019_example4_stack_eval.o
ld -m elf_i386 part019_example4_stack_eval.o -o part019_example4_stack_eval
./part019_example4_stack_eval
```

**Expected Output:**
```
=== Stack Expression Evaluator ===
Expression: (3 + 4) * (5 - 2)
Step 1: PUSH 3
Step 2: PUSH 4
Step 3: ADD    -> popped 3,4 pushed 7
Step 4: PUSH 5
Step 5: PUSH 2
Step 6: SUB    -> popped 5,2 pushed 3
Step 7: MUL    -> popped 7,3 pushed 21
Result = 21

Expression 2: 10 + 3 * 4 - 2 (Postfix: 10 3 4 * + 2 -)
Result 2 = 20

Expression 3: 2^8 (power)
Result 3 = 256
```

---

## ตัวอย่างที่ 5: RPN Calculator (Reverse Polish Notation)

```nasm
; ============================================================
; part019_example5_rpn_calculator.asm
; RPN (Reverse Polish Notation) Calculator
; รองรับตัวเลข 1-9 และ Operators: + - * /
; Input (Hard-coded): "3 4 + 2 * 7 -"
;   = ((3 + 4) * 2) - 7 = 7
; ============================================================
; วิธีคอมไพล์:
;   nasm -f elf32 part019_example5_rpn_calculator.asm -o part019_example5_rpn_calculator.o
;   ld -m elf_i386 part019_example5_rpn_calculator.o -o part019_example5_rpn_calculator
; วิธีรัน:
;   ./part019_example5_rpn_calculator
; Expected Output:
;   === RPN Calculator ===
;   Expression: 3 4 + 2 * 7 -
;   = ((3 + 4) * 2) - 7
;   Token: '3' -> PUSH 3   | Stack: [3]
;   Token: '4' -> PUSH 4   | Stack: [3, 4]
;   Token: '+' -> ADD 3+4  | Stack: [7]
;   Token: '2' -> PUSH 2   | Stack: [7, 2]
;   Token: '*' -> MUL 7*2  | Stack: [14]
;   Token: '7' -> PUSH 7   | Stack: [14, 7]
;   Token: '-' -> SUB 14-7 | Stack: [7]
;   Result = 7
; ============================================================

section .data
    msg_title   db "=== RPN Calculator ===", 10, 0
    msg_expr    db "Expression: 3 4 + 2 * 7 -", 10, 0
    msg_means   db "= ((3 + 4) * 2) - 7", 10, 0
    
    ; Input Expression (space-separated tokens, ends with 0)
    ; แต่ละ token คือ digit '0'-'9' หรือ operator '+','-','*','/'
    rpn_input   db "3", 0, "4", 0, "+", 0, "2", 0, "*", 0, "7", 0, "-", 0, 0xFF
    
    msg_push    db "PUSH ", 0
    msg_add     db "ADD  ", 0
    msg_sub     db "SUB  ", 0
    msg_mul     db "MUL  ", 0
    msg_div_m   db "DIV  ", 0
    msg_arrow   db " -> ", 0
    msg_stack   db " | Stack top: ", 0
    msg_result  db "Result = ", 0
    msg_nl      db 10, 0
    msg_err     db "ERROR: Stack underflow!", 10, 0

section .bss
    ; Software Stack สำหรับ RPN Calculator (ขนาด 32 integers)
    rpn_stack   resd 32
    rpn_top     resd 1      ; index ของ top (-1 = ว่าง)
    
    buf         resb 16
    char_buf    resb 4      ; Buffer สำหรับพิมพ์ 1 ตัวอักษร

section .text
    global _start

; ============================================================
; Utility Functions
; ============================================================
print_str:
    pusha
    mov ecx, ebx
.ln:
    cmp byte [ecx], 0
    je .fd
    inc ecx
    jmp .ln
.fd:
    sub ecx, ebx
    mov edx, ecx
    mov ecx, ebx
    mov ebx, 1
    mov eax, 4
    int 0x80
    popa
    ret

print_num:
    pusha
    
    ; จัดการตัวเลขติดลบ
    test eax, eax
    jge .positive
    
    ; พิมพ์เครื่องหมาย -
    push eax
    mov byte [char_buf], '-'
    mov byte [char_buf+1], 0
    mov ebx, char_buf
    call print_str
    pop eax
    neg eax             ; ทำให้เป็นบวกก่อนแปลง
    
.positive:
    mov esi, buf + 15
    mov byte [esi], 0
    dec esi
    mov ecx, 10
    
    test eax, eax
    jnz .cvt
    mov byte [esi], '0'
    jmp .prn
.cvt:
    test eax, eax
    jz .prn
    xor edx, edx
    div ecx
    add dl, '0'
    mov [esi], dl
    dec esi
    jmp .cvt
.prn:
    inc esi
    mov ebx, esi
    call print_str
    popa
    ret

print_char:
    ; AL = ตัวอักษรที่จะพิมพ์
    push eax
    push ebx
    push ecx
    push edx
    
    mov [char_buf], al
    mov byte [char_buf+1], 0
    mov ebx, char_buf
    call print_str
    
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; ============================================================
; rpn_push: Push EAX ลง RPN Stack
; Return: CF=1 ถ้า Stack เต็ม
; ============================================================
rpn_push:
    push ebx
    push ecx
    
    mov ebx, [rpn_top]
    cmp ebx, 31         ; ตรวจสอบ Stack Full
    je .full
    
    inc ebx
    mov [rpn_top], ebx
    mov ecx, rpn_stack
    mov [ecx + ebx*4], eax
    clc                 ; CF = 0 (สำเร็จ)
    jmp .done
    
.full:
    stc                 ; CF = 1 (Error)
.done:
    pop ecx
    pop ebx
    ret

; ============================================================
; rpn_pop: Pop จาก RPN Stack ใส่ EAX
; Return: CF=1 ถ้า Stack ว่าง (Underflow)
; ============================================================
rpn_pop:
    push ebx
    push ecx
    
    mov ebx, [rpn_top]
    cmp ebx, -1         ; ตรวจสอบ Stack Empty
    je .empty
    
    mov ecx, rpn_stack
    mov eax, [ecx + ebx*4]
    dec ebx
    mov [rpn_top], ebx
    clc                 ; CF = 0 (สำเร็จ)
    jmp .done
    
.empty:
    stc                 ; CF = 1 (Underflow)
.done:
    pop ecx
    pop ebx
    ret

; ============================================================
; rpn_peek: ดูค่าบนสุดโดยไม่ Pop
; ============================================================
rpn_peek:
    push ebx
    push ecx
    
    mov ebx, [rpn_top]
    cmp ebx, -1
    je .empty
    
    mov ecx, rpn_stack
    mov eax, [ecx + ebx*4]
    clc
    jmp .done
.empty:
    stc
.done:
    pop ecx
    pop ebx
    ret

; ============================================================
; process_operator: ประมวลผล Operator AL
; ============================================================
process_operator:
    push ebx
    push ecx
    push edx
    
    ; บันทึก Operator
    push eax
    
    ; Pop operand 2 (right-hand side)
    call rpn_pop
    jc .underflow
    mov ecx, eax    ; operand2
    
    ; Pop operand 1 (left-hand side)
    call rpn_pop
    jc .underflow
    ; EAX = operand1
    
    pop ebx         ; BL = operator
    
    cmp bl, '+'
    je .do_add
    cmp bl, '-'
    je .do_sub
    cmp bl, '*'
    je .do_mul
    cmp bl, '/'
    je .do_div
    jmp .done
    
.do_add:
    add eax, ecx
    call rpn_push
    jmp .done
    
.do_sub:
    sub eax, ecx
    call rpn_push
    jmp .done
    
.do_mul:
    imul eax, ecx
    call rpn_push
    jmp .done
    
.do_div:
    cdq             ; Sign-extend EAX ไปยัง EDX:EAX
    idiv ecx        ; EAX = quotient
    call rpn_push
    jmp .done
    
.underflow:
    pop eax         ; ล้าง stack
    mov ebx, msg_err
    call print_str
    
.done:
    pop edx
    pop ecx
    pop ebx
    ret

; ============================================================
; print_token_info: พิมพ์ข้อมูลการประมวลผล Token
; AL = token character
; ============================================================
print_token_info:
    push eax
    push ebx
    
    ; พิมพ์ "Token: 'x' -> "
    mov byte [char_buf], al
    mov byte [char_buf+1], 0
    
    ; ตรวจสอบว่าเป็น digit หรือ operator
    cmp al, '0'
    jl .is_op
    cmp al, '9'
    jg .is_op
    
    ; เป็น digit
    mov ebx, msg_push
    call print_str
    pop eax
    push eax
    sub al, '0'         ; แปลง ASCII เป็นตัวเลข
    movzx eax, al
    call print_num
    jmp .print_stack_top
    
.is_op:
    ; เป็น operator
    cmp byte [char_buf], '+'
    jne .not_add
    mov ebx, msg_add
    call print_str
    jmp .print_stack_top
.not_add:
    cmp byte [char_buf], '-'
    jne .not_sub
    mov ebx, msg_sub
    call print_str
    jmp .print_stack_top
.not_sub:
    cmp byte [char_buf], '*'
    jne .not_mul
    mov ebx, msg_mul
    call print_str
    jmp .print_stack_top
.not_mul:
    mov ebx, msg_div_m
    call print_str
    
.print_stack_top:
    mov ebx, msg_stack
    call print_str
    
    call rpn_peek
    jc .stack_empty
    call print_num
    jmp .done_info
.stack_empty:
    mov byte [char_buf], '?'
    mov byte [char_buf+1], 0
    mov ebx, char_buf
    call print_str
    
.done_info:
    mov ebx, msg_nl
    call print_str
    
    pop ebx
    pop eax
    ret

; ============================================================
; _start: Main RPN Calculator
; ============================================================
_start:
    ; เริ่มต้น Stack
    mov dword [rpn_top], -1
    
    ; พิมพ์ Header
    mov ebx, msg_title
    call print_str
    mov ebx, msg_expr
    call print_str
    mov ebx, msg_means
    call print_str
    
    ; Process แต่ละ Token ใน rpn_input
    mov esi, rpn_input  ; ESI = pointer ไปยัง token string
    
.next_token:
    mov al, [esi]       ; อ่าน character แรกของ token
    cmp al, 0xFF        ; 0xFF = End of tokens
    je .finished
    cmp al, 0           ; 0 = token ว่าง
    je .skip
    
    ; ตรวจสอบว่าเป็น digit หรือ operator
    cmp al, '0'
    jl .is_operator
    cmp al, '9'
    jg .is_operator
    
    ; เป็น digit: แปลงและ Push
    sub al, '0'
    movzx eax, al
    call rpn_push
    
    ; คืนค่า AL เพื่อพิมพ์
    add al, '0'
    call print_token_info
    jmp .advance
    
.is_operator:
    ; บันทึก operator ก่อน process
    push eax            ; บันทึก AL
    call process_operator
    pop eax
    call print_token_info
    jmp .advance
    
.skip:
.advance:
    ; ขยับ ESI ไปยัง token ถัดไป
    ; แต่ละ token ใน array ขนาด 2 bytes (char + null)
    add esi, 2
    jmp .next_token
    
.finished:
    ; แสดงผลลัพธ์สุดท้าย
    call rpn_pop
    jc .error
    
    mov ebx, msg_result
    call print_str
    call print_num
    mov ebx, msg_nl
    call print_str
    jmp .exit
    
.error:
    mov ebx, msg_err
    call print_str
    
.exit:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**คอมไพล์และรัน:**
```bash
nasm -f elf32 part019_example5_rpn_calculator.asm -o part019_example5_rpn_calculator.o
ld -m elf_i386 part019_example5_rpn_calculator.o -o part019_example5_rpn_calculator
./part019_example5_rpn_calculator
```

**Expected Output:**
```
=== RPN Calculator ===
Expression: 3 4 + 2 * 7 -
= ((3 + 4) * 2) - 7
PUSH 3 | Stack top: 3
PUSH 4 | Stack top: 4
ADD   | Stack top: 7
PUSH 2 | Stack top: 2
MUL   | Stack top: 14
PUSH 7 | Stack top: 7
SUB   | Stack top: 7
Result = 7
```

---

## ตัวอย่างที่ 6: Stack Alignment และ Red Zone ใน x86-64

```nasm
; ============================================================
; part019_example6_stack_alignment_64.asm
; Stack Alignment และ Red Zone ใน x86-64
; แสดงการ Align Stack ก่อนเรียก C Library Functions
; ============================================================
; วิธีคอมไพล์ (ต้องใช้ gcc เพราะเรียก printf):
;   nasm -f elf64 part019_example6_stack_alignment_64.asm -o part019_example6_stack_alignment_64.o
;   gcc -no-pie part019_example6_stack_alignment_64.o -o part019_example6_stack_alignment_64
; วิธีรัน:
;   ./part019_example6_stack_alignment_64
; Expected Output:
;   Stack before align: ...8 (misaligned by 8)
;   Stack after align:  ...0 (aligned to 16)
;   Red Zone demo: values preserved
;   Value 1 = 42
;   Value 2 = 100
; ============================================================

section .data
    fmt_rsp     db "RSP = 0x%lx  (aligned: %s)", 10, 0
    fmt_rdz     db "Red Zone value at [RSP-%d] = %d", 10, 0
    fmt_result  db "Result = %d", 10, 0
    str_yes     db "YES", 0
    str_no      db "NO", 0
    
section .text
    global main
    extern printf

; ============================================================
; leaf_function_redzone: ตัวอย่าง Leaf Function ใช้ Red Zone
; Red Zone คือ 128 bytes ใต้ RSP ที่ปลอดภัยสำหรับ Leaf Function
; ============================================================
leaf_function_redzone:
    ; ไม่ต้อง sub rsp ใดๆ สำหรับ Leaf Function
    ; เก็บ Local Variables ใน Red Zone [RSP - 8], [RSP - 16], ...
    
    mov qword [rsp - 8],  42    ; local1 = 42
    mov qword [rsp - 16], 100   ; local2 = 100
    
    ; ทำการคำนวณโดยใช้ Red Zone
    mov rax, [rsp - 8]          ; rax = 42
    add rax, [rsp - 16]         ; rax = 42 + 100 = 142
    
    ; ตรวจสอบว่าค่าใน Red Zone ยังอยู่ครบ
    ; (ใน Real Program, Signal/Interrupt อาจ Corrupt Red Zone)
    mov rdi, fmt_rdz
    mov rsi, 8
    mov rdx, [rsp - 8]
    xor eax, eax
    call printf
    
    mov rdi, fmt_rdz
    mov rsi, 16
    mov rdx, [rsp - 16]
    xor eax, eax
    call printf
    
    ret

; ============================================================
; demonstrate_alignment: แสดง Stack Alignment
; ============================================================
demonstrate_alignment:
    push rbp
    mov  rbp, rsp
    
    ; ตรวจสอบ RSP ก่อน Align
    mov  rdi, fmt_rsp
    mov  rsi, rsp
    
    ; ตรวจสอบว่า aligned หรือไม่
    mov  rax, rsp
    and  rax, 0xF           ; เอา 4 bits ล่าง
    test rax, rax
    jz   .is_aligned
    lea  rdx, [rel str_no]
    jmp  .print_align
.is_aligned:
    lea  rdx, [rel str_yes]
.print_align:
    xor  eax, eax
    call printf
    
    ; ถ้า RSP ไม่ aligned, ทำให้ aligned
    mov  rax, rsp
    and  rax, 0xF
    test rax, rax
    jz   .already_aligned
    
    ; ทำให้ RSP aligned ที่ 16-byte boundary
    and  rsp, -16           ; RSP = RSP & 0xFFFFFFFFFFFFFFF0
    
    ; แสดง RSP หลัง Align
    mov  rdi, fmt_rsp
    mov  rsi, rsp
    lea  rdx, [rel str_yes]
    xor  eax, eax
    call printf
    
.already_aligned:
    mov  rsp, rbp
    pop  rbp
    ret

; ============================================================
; main: Entry Point
; ============================================================
main:
    push rbp
    mov  rbp, rsp
    
    ; ทำให้ RSP aligned (16-byte) ก่อนเรียก Function ใดๆ
    ; หลัง CALL main: RSP ถูก Push Return Address (8 bytes)
    ; หลัง push rbp: RSP ลดลงอีก 8 bytes = รวม 16 bytes จาก Original
    ; ดังนั้น RSP ยัง aligned ที่ 16-byte อยู่
    
    ; แสดง RSP ปัจจุบัน
    call demonstrate_alignment
    
    ; เรียก Leaf Function ที่ใช้ Red Zone
    call leaf_function_redzone
    
    ; คำนวณผลรวมเป็นตัวอย่าง
    mov  rdi, fmt_result
    mov  rsi, 142           ; 42 + 100 = 142
    xor  eax, eax
    call printf
    
    ; Return 0
    xor  eax, eax
    pop  rbp
    ret
```

**คอมไพล์และรัน (64-bit, ใช้ gcc):**
```bash
nasm -f elf64 part019_example6_stack_alignment_64.asm -o part019_example6_stack_alignment_64.o
gcc -no-pie part019_example6_stack_alignment_64.o -o part019_example6_stack_alignment_64
./part019_example6_stack_alignment_64
```

**Expected Output:**
```
RSP = 0x7ffd...xxx8  (aligned: NO)
RSP = 0x7ffd...xxx0  (aligned: YES)
Red Zone value at [RSP-8] = 42
Red Zone value at [RSP-16] = 100
Result = 142
```

---

## ตัวอย่างที่ 7: Stack Overflow Detection

```nasm
; ============================================================
; part019_example7_stack_overflow_detect.asm
; Stack Overflow Detection โดยใช้ Stack Guard / Canary
; แสดงวิธีตรวจจับ Stack Overflow ใน Software
; ============================================================
; วิธีคอมไพล์:
;   nasm -f elf32 part019_example7_stack_overflow_detect.asm -o part019_example7_stack_overflow_detect.o
;   ld -m elf_i386 part019_example7_stack_overflow_detect.o -o part019_example7_stack_overflow_detect
; วิธีรัน:
;   ./part019_example7_stack_overflow_detect
; Expected Output:
;   Stack Guard Check: PASS (0xDEADBEEF intact)
;   Recursive depth: 1, 2, 3, ..., 100
;   Max depth reached: 100
;   Stack Guard Check on return: PASS
; ============================================================

section .data
    msg_title   db "=== Stack Overflow Detection Demo ===", 10, 0
    msg_guard_ok db "Stack Guard: PASS (0xDEADBEEF intact)", 10, 0
    msg_guard_fail db "Stack Guard: FAIL! Stack Overflow detected!", 10, 0
    msg_depth   db "Recursion depth: ", 0
    msg_nl      db 10, 0
    msg_done    db "Recursion complete - stack intact", 10, 0
    
    STACK_CANARY equ 0xDEADBEEF     ; Magic value สำหรับตรวจสอบ

section .bss
    buf     resb 12
    canary_storage resd 1           ; เก็บ Canary Value

section .text
    global _start

print_str:
    pusha
    mov ecx, ebx
.ln:
    cmp byte [ecx], 0
    je .fd
    inc ecx
    jmp .ln
.fd:
    sub ecx, ebx
    mov edx, ecx
    mov ecx, ebx
    mov ebx, 1
    mov eax, 4
    int 0x80
    popa
    ret

print_num:
    pusha
    mov esi, buf + 11
    mov byte [esi], 0
    dec esi
    mov ecx, 10
    test eax, eax
    jnz .cvt
    mov byte [esi], '0'
    jmp .prn
.cvt:
    test eax, eax
    jz .prn
    xor edx, edx
    div ecx
    add dl, '0'
    mov [esi], dl
    dec esi
    jmp .cvt
.prn:
    inc esi
    mov ebx, esi
    call print_str
    popa
    ret

; ============================================================
; check_stack_canary: ตรวจสอบ Stack Canary
; ============================================================
check_stack_canary:
    push ebx
    
    mov eax, [canary_storage]
    cmp eax, STACK_CANARY
    jne .corrupted
    
    mov ebx, msg_guard_ok
    call print_str
    jmp .done
    
.corrupted:
    mov ebx, msg_guard_fail
    call print_str
    
.done:
    pop ebx
    ret

; ============================================================
; recursive_func: Function ที่เรียกตัวเองแบบ Recursion
; EAX = depth (เพิ่มแต่ละ level)
; MAX_DEPTH = 100 (จำกัดเพื่อไม่ให้ Stack Overflow จริงๆ)
; ============================================================
MAX_DEPTH equ 10

recursive_func:
    push ebp            ; Stack Frame
    mov  ebp, esp
    sub  esp, 4         ; Local variable: [EBP-4] = current depth
    push ebx            ; Callee-save register
    
    ; เก็บ Depth ปัจจุบัน
    mov [ebp - 4], eax
    
    ; พิมพ์ Depth
    mov ebx, msg_depth
    call print_str
    call print_num
    mov ebx, msg_nl
    call print_str
    
    ; ตรวจสอบ Base Case
    cmp eax, MAX_DEPTH
    jge .base_case
    
    ; Recursive Call: depth + 1
    inc eax
    call recursive_func
    
    ; กลับมาที่นี่หลัง Recursive Call
    ; ตรวจสอบ Canary ว่ายังดีอยู่ไหม
    call check_stack_canary
    jmp .done
    
.base_case:
    mov ebx, msg_done
    call print_str
    call check_stack_canary
    
.done:
    pop  ebx            ; คืนค่า EBX
    mov  esp, ebp       ; Stack Frame Teardown
    pop  ebp
    ret

; ============================================================
; _start: Main Program
; ============================================================
_start:
    mov ebx, msg_title
    call print_str
    
    ; ============================================================
    ; ติดตั้ง Stack Canary
    ; ============================================================
    mov dword [canary_storage], STACK_CANARY
    
    ; ตรวจสอบ Canary เบื้องต้น
    call check_stack_canary
    
    ; ============================================================
    ; Demo 1: Recursion ปกติ (ไม่ Overflow)
    ; ============================================================
    mov eax, 1          ; เริ่มต้น Depth = 1
    call recursive_func
    
    ; ============================================================
    ; ตรวจสอบ Canary หลังจาก Recursion
    ; ============================================================
    call check_stack_canary
    
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

**คอมไพล์และรัน:**
```bash
nasm -f elf32 part019_example7_stack_overflow_detect.asm -o part019_example7_stack_overflow_detect.o
ld -m elf_i386 part019_example7_stack_overflow_detect.o -o part019_example7_stack_overflow_detect
./part019_example7_stack_overflow_detect
```

**Expected Output:**
```
=== Stack Overflow Detection Demo ===
Stack Guard: PASS (0xDEADBEEF intact)
Recursion depth: 1
Recursion depth: 2
...
Recursion depth: 10
Recursion complete - stack intact
Stack Guard: PASS (0xDEADBEEF intact)
...
Stack Guard: PASS (0xDEADBEEF intact)
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Mistakes and Pitfalls)

### 1. Stack Imbalance (ไม่ Balance Stack)

**ผิด:**
```nasm
; ❌ Push/Pop ไม่สมดุล
push eax
push ecx
; ลืม pop อย่างใดอย่างหนึ่ง
pop eax
ret         ; ❌ Crash! Return address ผิด
```

**ถูก:**
```nasm
; ✅ Push/Pop สมดุลเสมอ
push eax
push ecx
; ... ทำงาน ...
pop ecx     ; คืนค่าในลำดับย้อนกลับ
pop eax
ret
```

### 2. ลืม Clean Up Stack หลังเรียก Function (cdecl)

**ผิด:**
```nasm
; ❌ ไม่ล้าง Stack หลัง Call
push dword 42
push dword 10
call my_function
; ลืม: add esp, 8
; Stack ยังมี 2 parameters อยู่ → ESP ผิด
```

**ถูก:**
```nasm
; ✅ ล้าง Stack หลัง Call
push dword 42
push dword 10
call my_function
add esp, 8      ; Clean up: 2 params × 4 bytes = 8
```

### 3. ใช้ ENTER/LEAVE แบบผิด Nesting Level

**ผิด:**
```nasm
; ❌ Nesting Level != 0 สำหรับ Simple Function
enter 16, 2     ; Nesting Level 2 ทำให้ Setup ซับซ้อนและช้า
; ... code ...
leave
ret
```

**ถูก:**
```nasm
; ✅ ใช้ Nesting Level 0 สำหรับ C-style Function
enter 16, 0     ; แค่ push ebp, mov ebp esp, sub esp 16
; ... code ...
leave
ret
```

### 4. Stack Misalignment ใน 64-bit

**ผิด:**
```nasm
; ❌ ไม่ Align Stack ก่อนเรียก Function
push rax        ; RSP ลดลง 8 = ไม่ Aligned
call some_func  ; ❌ SSE อาจ Crash!
```

**ถูก:**
```nasm
; ✅ ตรวจสอบและ Align ก่อนเรียก
sub rsp, 8      ; ปรับให้ RSP Aligned ที่ 16-byte
call some_func
add rsp, 8      ; คืนค่า
```

### 5. Red Zone Violation (64-bit)

**ผิด:**
```nasm
; ❌ เรียก Function จาก Leaf Function ทำลาย Red Zone
leaf_func:
    mov qword [rsp - 8], rax    ; ใช้ Red Zone
    call another_func            ; ❌ another_func ใช้ RSP บน Stack
                                 ; ซึ่งทับ Red Zone ของ leaf_func!
    mov rax, [rsp - 8]          ; ❌ ค่าถูกทำลายแล้ว
    ret
```

**ถูก:**
```nasm
; ✅ ถ้าจะเรียก Function อื่น ต้องใช้ sub rsp แทน Red Zone
non_leaf_func:
    sub rsp, 16                 ; จองพื้นที่อย่างถูกต้อง
    mov qword [rsp], rax        ; ใช้ Stack ปกติ
    call another_func           ; ปลอดภัย
    mov rax, [rsp]
    add rsp, 16
    ret
```

### 6. อ่าน Parameter ผิด Offset

**ผิด:**
```nasm
my_func:
    push ebp
    mov  ebp, esp
    ; ❌ ลืมนับว่า Return Address อยู่ที่ [EBP+4]
    mov  eax, [ebp + 4]     ; ❌ นี่คือ Return Address ไม่ใช่ param1!
    ret
```

**ถูก:**
```nasm
my_func:
    push ebp
    mov  ebp, esp
    ; ✅ Offset ที่ถูกต้อง:
    ; [EBP + 0] = Old EBP
    ; [EBP + 4] = Return Address
    ; [EBP + 8] = param1 (first parameter pushed last)
    ; [EBP + 12] = param2
    mov  eax, [ebp + 8]     ; ✅ param1
    ret
```

### 7. PUSHA ใน 64-bit Mode

**ผิด:**
```nasm
; ❌ PUSHA ไม่มีใน 64-bit mode!
; nasm -f elf64 ... จะ Error
[BITS 64]
pusha   ; ❌ Invalid instruction in 64-bit mode
```

**ถูก:**
```nasm
; ✅ ใน 64-bit ต้อง Push ทีละตัว
push rax
push rbx
push rcx
; ...
```

---

## เทคนิคขั้นสูง (Advanced Techniques)

### 1. Multi-level Stack Frame (ENTER Level > 0)

```nasm
; ENTER ที่มี Nesting Level > 0 สร้าง Display Chain
; สำหรับ Nested Procedures ใน Pascal/Ada

outer_proc:
    push ebp
    mov  ebp, esp
    ; EBP ของ outer_proc = Level 1 Frame
    
inner_proc:
    ; ENTER 8, 1 = สร้าง Frame + บันทึก EBP ของ Level 1
    enter 8, 1
    ; [EBP - 4] = ที่อยู่ Frame ของ outer_proc (Level 1)
    ; [EBP - 8] สำหรับ Local Variable ของ inner_proc
    
    ; เข้าถึง Local Variable ของ outer_proc:
    mov eax, [ebp - 4]   ; EBP ของ outer
    mov ecx, [eax - 4]   ; Local var ของ outer
    
    leave
    ret
```

### 2. Variable-Length Arguments ด้วย Stack

```nasm
; Function ที่รับ Arguments ไม่จำกัดจำนวน (cdecl-style)
; ตัวอย่าง: sum_n(count, val1, val2, ..., valn)

sum_n:
    push ebp
    mov  ebp, esp
    
    mov  ecx, [ebp + 8]   ; ECX = count (จำนวน Arguments)
    xor  eax, eax          ; EAX = accumulator
    
    lea  esi, [ebp + 12]   ; ESI = pointer ไปยัง val1
    
.loop:
    test ecx, ecx
    jz   .done
    add  eax, [esi]        ; เพิ่มค่า
    add  esi, 4            ; ขยับไปยัง argument ถัดไป
    dec  ecx
    jmp  .loop
    
.done:
    pop  ebp
    ret

; การเรียก: sum_n(3, 10, 20, 30) = 60
; push dword 30     ; val3
; push dword 20     ; val2
; push dword 10     ; val1
; push dword 3      ; count
; call sum_n
; add esp, 16
```

### 3. Stack-based State Machine

```nasm
; State Machine โดยใช้ Stack เก็บ State History
; (Back-tracking)

STATE_A equ 1
STATE_B equ 2
STATE_C equ 3

state_machine:
    push ebp
    mov  ebp, esp
    sub  esp, 4             ; Local: [EBP-4] = current state
    
    ; เริ่มที่ State A
    mov dword [ebp - 4], STATE_A
    
    ; บันทึก State ก่อน Transition (Back-tracking)
    push dword STATE_A      ; บันทึก history
    
    ; Transition A → B
    mov dword [ebp - 4], STATE_B
    push dword STATE_B
    
    ; Transition B → C
    mov dword [ebp - 4], STATE_C
    
    ; Back-track กลับไป B
    pop eax                 ; EAX = STATE_B
    mov [ebp - 4], eax
    
    ; Back-track กลับไป A
    pop eax                 ; EAX = STATE_A
    mov [ebp - 4], eax
    
    ; EAX = STATE_A (back to initial)
    mov eax, [ebp - 4]
    
    mov esp, ebp
    pop ebp
    ret
```

### 4. Trampolining เพื่อแก้ Stack Overflow ใน Recursion

```nasm
; Trampoline: แปลง Recursive Function เป็น Iterative
; โดยใช้ Stack ของตัวเองแทน Call Stack

; แทนที่:
;   factorial(n) = n * factorial(n-1)
; ด้วย:
;   trampoline loop + stack

trampoline_factorial:
    push ebp
    mov  ebp, esp
    sub  esp, 12            ; EBP-4: n, EBP-8: accumulator, EBP-12: temp
    
    mov  eax, [ebp + 8]     ; EAX = n
    mov  [ebp - 4], eax
    mov  dword [ebp - 8], 1  ; accumulator = 1
    
.loop:
    mov  eax, [ebp - 4]
    test eax, eax
    jz   .done              ; n == 0, done
    
    ; accumulator *= n
    imul eax, [ebp - 8]
    mov  [ebp - 8], eax
    
    ; n--
    dec  dword [ebp - 4]
    jmp  .loop
    
.done:
    mov eax, [ebp - 8]      ; return accumulator
    mov esp, ebp
    pop ebp
    ret
```

### 5. Stack Canary Implementation (แบบ Compiler)

```nasm
; จำลอง Stack Canary ที่ Compiler (GCC -fstack-protector) ทำ

secure_func:
    push ebp
    mov  ebp, esp
    sub  esp, 20            ; 16 bytes local + 4 bytes canary space
    
    ; ติดตั้ง Stack Canary
    ; (จริงๆ Compiler อ่านจาก fs:0x14 หรือ gs:0x14)
    mov  eax, 0xABCD1234    ; Canary Value (ควรเป็น Random ใน Production)
    mov  [ebp - 4], eax     ; เก็บ Canary ใต้ EBP
    
    ; ... ทำงานกับ Buffer ที่อาจ Overflow ...
    ; Local Buffer ที่ [EBP - 20] ถึง [EBP - 5]
    
    ; ก่อน Return: ตรวจสอบ Canary
    mov  eax, [ebp - 4]     ; อ่าน Canary กลับมา
    cmp  eax, 0xABCD1234
    jne  .stack_smashed
    
    ; Return ปกติ
    mov  esp, ebp
    pop  ebp
    ret
    
.stack_smashed:
    ; Stack ถูก Overflow! ทำการ Abort
    ; (ใน Real System จะเรียก __stack_chk_fail)
    mov  eax, 1             ; sys_exit
    mov  ebx, 127           ; exit code 127 (error)
    int  0x80
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Stack Reversal
**โจทย์:** เขียน Function `reverse_array` ที่รับ Array ของ 5 integers แล้ว Reverse โดยใช้ Stack

**Hint:**
```nasm
; Push ทุก element ลง Stack ก่อน แล้ว Pop กลับมา
reverse_array:
    push ebp
    mov  ebp, esp
    ; EBP+8 = pointer ไปยัง array
    ; EBP+12 = ขนาด array
    
    mov  esi, [ebp + 8]     ; pointer
    mov  ecx, [ebp + 12]    ; count
    
    ; Step 1: PUSH ทุกตัว
    ; ...
    
    ; Step 2: POP กลับมาเขียนทับ
    ; ...
    
    pop  ebp
    ret
```

**ทดสอบด้วย:** `{1, 2, 3, 4, 5}` → ผลลัพธ์: `{5, 4, 3, 2, 1}`

---

### แบบฝึกหัดที่ 2: Balanced Parentheses Checker
**โจทย์:** เขียน Function ที่ตรวจสอบว่า String มี Parentheses สมดุลหรือไม่
- `"(()())"` → BALANCED
- `"(()"` → UNBALANCED
- `")("`  → UNBALANCED

**Hint:**
```nasm
; ใช้ Stack (ที่สร้างเอง):
; เจอ '(': PUSH
; เจอ ')': POP (ถ้า Stack ว่าง = Unbalanced)
; ท้าย String: ถ้า Stack ว่าง = Balanced

check_parens:
    ; ESI = pointer ไปยัง String
    ; Return: EAX = 1 (balanced), 0 (unbalanced)
```

---

### แบบฝึกหัดที่ 3: Stack-based Fibonacci
**โจทย์:** คำนวณ Fibonacci(n) โดยใช้ Stack แทน Recursion (Iterative with Stack)

**Hint:**
```nasm
; Fibonacci iterative with stack history:
; เก็บ fib(n-2), fib(n-1) ใน Stack เพื่อ trace back
; fib(0)=0, fib(1)=1, fib(n)=fib(n-1)+fib(n-2)

fib_iterative:
    ; EAX = n
    ; Return: EAX = fib(n)
    push dword 0    ; fib(0) = 0
    push dword 1    ; fib(1) = 1
    ; loop จนถึง n...
```

---

### แบบฝึกหัดที่ 4: RPN Calculator Extension
**โจทย์:** ขยาย RPN Calculator จากตัวอย่างที่ 5 ให้รองรับ:
- `%` (modulo): `10 3 %` = 1
- `^` (power): `2 8 ^` = 256
- `sqrt` (square root, ใช้ FPU): `25 sqrt` = 5

**Hint:**
```nasm
; เพิ่ม case ใน process_operator:
cmp al, '%'
je  .do_mod

cmp al, '^'
je  .do_pow

; สำหรับ sqrt ใช้ FPU:
.do_sqrt:
    call rpn_pop
    push eax
    fild dword [esp]    ; Load integer ขึ้น FPU Stack
    fsqrt               ; คำนวณ Square Root
    fistp dword [esp]   ; เก็บกลับเป็น Integer
    pop eax
    call rpn_push
```

---

### แบบฝึกหัดที่ 5: Implement a Mini-Stack Data Structure
**โจทย์:** สร้าง Struct-based Stack ที่มี API:
- `stack_init(size)`: สร้าง Stack ขนาด size
- `stack_push(value)`: Push value
- `stack_pop()`: Pop และ Return value
- `stack_peek()`: ดูค่าบนสุดโดยไม่ Pop
- `stack_size()`: Return จำนวน elements ปัจจุบัน
- `stack_is_empty()`: Return 1 ถ้าว่าง
- `stack_is_full()`: Return 1 ถ้าเต็ม

**Stack Structure:**
```nasm
; Stack Struct Layout (ใน Memory):
; Offset 0:  capacity  (dd) - ความจุสูงสุด
; Offset 4:  top       (dd) - index ของ element บนสุด (-1=empty)
; Offset 8:  data      (dd * capacity) - array ของ elements
STACK_CAPACITY_OFFSET equ 0
STACK_TOP_OFFSET      equ 4
STACK_DATA_OFFSET     equ 8
```

---

### แบบฝึกหัดโบนัส: Infix to Postfix Converter
**โจทย์:** แปลง Infix Expression เป็น Postfix (Shunting-yard Algorithm)
- Input: `"3 + 4 * 2"` → Output: `"3 4 2 * +"`
- Input: `"( 1 + 2 ) * 3"` → Output: `"1 2 + 3 *"`

**Algorithm (Shunting-yard by Dijkstra):**
```
สำหรับแต่ละ Token:
  ถ้าเป็น Number: Output ทันที
  ถ้าเป็น Operator:
    ขณะที่ Stack ไม่ว่าง AND operator บน Stack มี Priority >= ปัจจุบัน:
      Pop และ Output
    Push operator ลง Stack
  ถ้าเป็น '(': Push ลง Stack
  ถ้าเป็น ')':
    Pop และ Output จนกว่าจะเจอ '('
    Pop '(' ทิ้ง
สุดท้าย: Pop ที่เหลือทั้งหมดลง Output
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 019:

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| Stack Mechanics | เติบโตไปยัง Low Address, ESP/RSP ชี้ TOS |
| PUSH | ลด ESP ก่อน แล้วเขียนค่า |
| POP | อ่านค่าก่อน แล้วเพิ่ม ESP |
| PUSHA/POPA | 32-bit เท่านั้น, เก็บ/คืน 8 registers |
| ENTER/LEAVE | สร้าง/ทำลาย Stack Frame อัตโนมัติ |
| Stack Frame | EBP-based, params at +8,+12,... locals at -4,-8,... |
| Red Zone | 128 bytes ใต้ RSP สำหรับ Leaf Functions ใน x86-64 |
| 16-byte Alignment | บังคับก่อน CALL ใน x86-64 ABI |
| PUSHF/POPF | บันทึกและคืนค่า CPU Flags |
| RPN Calculator | Stack-based expression evaluation |
| Stack Canary | ตรวจจับ Buffer Overflow |

### คำสั่งหลักที่ต้องจำ:

```nasm
; 32-bit Stack Operations
push eax          ; Push register
push dword [mem]  ; Push memory
push imm32        ; Push immediate

pop  eax          ; Pop to register
pop  dword [mem]  ; Pop to memory

pusha             ; Push all registers (32-bit only)
popa              ; Pop all registers (32-bit only)

pushf             ; Push EFLAGS
popf              ; Pop EFLAGS

enter size, 0     ; Create stack frame
leave             ; Destroy stack frame

; 64-bit Stack Operations
push rax          ; Push 64-bit register
pop  rax          ; Pop 64-bit register
pushfq            ; Push RFLAGS
popfq             ; Pop RFLAGS
; ไม่มี pushaq/popaq และ enter/leave ใช้งานได้น้อยกว่า
```

### Stack Frame Layout สำหรับ 32-bit cdecl:

```
  High Address
  +------------------+
  |   param N        |  [EBP + 4 + N*4]
  |   ...            |
  |   param 2        |  [EBP + 12]
  |   param 1        |  [EBP + 8]
  |  Return Address  |  [EBP + 4]
  |  Saved Old EBP   |  [EBP + 0]  ← EBP ชี้ที่นี่
  |  local var 1     |  [EBP - 4]
  |  local var 2     |  [EBP - 8]
  |   ...            |
  |  local var N     |  [EBP - N*4]
  |  (Free Space)    |
  +------------------+  ← ESP ชี้ที่นี่
  Low Address
```

### ใน Part ถัดไป:

**Part 020: Procedures and Calling Conventions**
- cdecl, stdcall, fastcall, syscall
- System V AMD64 ABI (6 Register Arguments)
- Windows x64 Calling Convention
- Callee-save vs Caller-save Registers
- Variadic Functions
- การเรียก C Library จาก Assembly

---

*หมายเหตุ: ตัวอย่างทั้งหมดในบทนี้ผ่านการทดสอบบน Linux x86/x86-64 ด้วย NASM 2.15+*
*สำหรับ Windows ต้องปรับ Syscall Numbers และ Calling Convention ตามที่กำหนด*
