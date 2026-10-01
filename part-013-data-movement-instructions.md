# Part 013: Data Movement Instructions (คำสั่งย้ายข้อมูล)

## Prerequisites and Learning Objectives (ความต้องการเบื้องต้นและวัตถุประสงค์การเรียนรู้)

### Prerequisites (ความต้องการเบื้องต้น)
- เข้าใจ Register Architecture (x86/x86-64) จาก Part 001-005
- เข้าใจ Memory Addressing Modes จาก Part 010
- เข้าใจ Stack Operations จาก Part 011
- สามารถเขียน NASM syntax ขั้นพื้นฐานได้

### Learning Objectives (วัตถุประสงค์การเรียนรู้)
หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ใช้คำสั่ง MOV ได้ทุกรูปแบบ (register-to-register, memory-to-register, immediate)
2. ใช้ MOVZX และ MOVSX สำหรับการขยายข้อมูล
3. ใช้ MOVSXD ใน 64-bit mode
4. ใช้ LEA สำหรับการคำนวณ Address และ arithmetic
5. ใช้ XCHG และ BSWAP สำหรับการสลับข้อมูล
6. ใช้ CMOVcc สำหรับ Conditional Move แบบไม่ใช้ branch
7. เข้าใจ PUSH/POP, PUSHA/POPA และ PUSHAD/POPAD
8. ใช้ LAHF/SAHF สำหรับจัดการ Flags
9. เขียน memcpy ด้วย Assembly

---

## Theory (ทฤษฎี)

### 1. ภาพรวมของ Data Movement (Overview)

Data Movement Instructions คือคำสั่งที่ใช้ในการ:
- **คัดลอกข้อมูล** จากที่หนึ่งไปยังอีกที่หนึ่ง
- **โหลดข้อมูล** จาก Memory เข้า Register
- **เก็บข้อมูล** จาก Register ลง Memory
- **จัดการ Stack** สำหรับการเรียกใช้ฟังก์ชัน
- **ปรับขนาดข้อมูล** (Zero-extend, Sign-extend)

```
Source                          Destination
┌─────────────────────────────────────────────┐
│  Register  ──────────────►  Register        │
│  Memory    ──────────────►  Register        │
│  Register  ──────────────►  Memory          │
│  Immediate ──────────────►  Register        │
│  Immediate ──────────────►  Memory          │
└─────────────────────────────────────────────┘

หมายเหตุ: Memory ─► Memory โดยตรงไม่ได้ (ยกเว้น MOVS string instruction)
```

### 2. MOV Instruction - รูปแบบทั้งหมด

คำสั่ง `MOV` คือคำสั่งพื้นฐานที่ใช้บ่อยที่สุดใน x86 Assembly

**Syntax:**
```
MOV destination, source
```

**กฎสำคัญของ MOV:**
1. ทั้ง source และ destination ต้องมีขนาดเท่ากัน
2. ไม่สามารถ move จาก memory ไป memory โดยตรง
3. ไม่สามารถ move immediate value ไป segment register โดยตรง
4. Segment register ต้องใช้ general-purpose register เป็นตัวกลาง

**ตัวอย่างรูปแบบ MOV:**
```
; MOV reg, reg  (register to register)
MOV eax, ebx          ; copy ebx ไปยัง eax (32-bit)
MOV rax, rbx          ; copy rbx ไปยัง rax (64-bit)
MOV al, bl            ; copy bl ไปยัง al (8-bit)

; MOV reg, mem  (memory to register)
MOV eax, [ebx]        ; โหลดค่า 32-bit จาก address ที่ ebx ชี้
MOV rax, [rsp+8]      ; โหลดค่าจาก stack + offset
MOV al, [var]         ; โหลด byte จาก memory variable

; MOV mem, reg  (register to memory)
MOV [ebx], eax        ; เก็บ eax ที่ address ที่ ebx ชี้
MOV [var], ecx        ; เก็บค่าลง memory variable
MOV [rsp-4], edx      ; เก็บลง local variable บน stack

; MOV reg, imm  (immediate to register)
MOV eax, 42           ; โหลดค่า 42 ลง eax
MOV rax, 0xDEADBEEF   ; โหลด immediate 64-bit
MOV al, 0xFF          ; โหลด byte immediate

; MOV mem, imm  (immediate to memory)
MOV DWORD [var], 100  ; เก็บ immediate ลง memory (ต้องระบุขนาด)
MOV BYTE [ptr], 0     ; เก็บ 0 ลง memory byte
MOV QWORD [rsp], -1   ; เก็บ immediate ลง stack
```

### 3. MOVZX - Zero Extension (การขยายแบบ Zero)

`MOVZX` (Move with Zero-Extension) ใช้เมื่อต้องการ copy ค่าจาก register/memory ขนาดเล็กไปยัง register ขนาดใหญ่ โดย **เติม 0 ในส่วนที่เหลือ**

```
การทำงาน:
เช่น al = 0xAB (8-bit unsigned: 171)

MOVZX eax, al
ก่อน: al = 0xAB
หลัง: eax = 0x000000AB  (upper bits เป็น 0 ทั้งหมด)
```

**เหมาะสำหรับ:** ข้อมูลที่ไม่มี sign (unsigned) เช่น ASCII, byte counters

### 4. MOVSX - Sign Extension (การขยายแบบ Sign)

`MOVSX` (Move with Sign-Extension) ใช้เมื่อต้องการ copy ค่า signed จาก register/memory ขนาดเล็กไปยัง register ขนาดใหญ่ โดย **ขยาย sign bit ออกไป**

```
การทำงาน:
เช่น al = 0xFE (8-bit signed: -2)
     MSB = 1 (negative)

MOVSX eax, al
ก่อน: al = 0xFE
หลัง: eax = 0xFFFFFFFE  (ขยาย sign bit ออก = ยังเป็น -2)

ถ้า al = 0x7E (8-bit signed: 126)
     MSB = 0 (positive)

MOVSX eax, al
หลัง: eax = 0x0000007E  (MSB=0, เติม 0 เหมือน MOVZX)
```

**เหมาะสำหรับ:** ข้อมูลที่มี sign (signed) เช่น signed integers, temperature values

### 5. MOVSXD - Sign Extension 64-bit

`MOVSXD` (Move with Sign-Extension Double to Quad) ใช้ใน 64-bit mode เพื่อ sign-extend จาก 32-bit ไปเป็น 64-bit

```
MOVSXD rax, eax       ; sign-extend eax (32-bit) เป็น rax (64-bit)
MOVSXD rbx, [mem32]   ; sign-extend จาก memory 32-bit เป็น rbx

หมายเหตุ: ใน 64-bit mode, การ assign ค่าให้ 32-bit register จะ zero-extend เป็น 64-bit อัตโนมัติ
เช่น: MOV eax, 1    →  rax = 0x0000000000000001

ดังนั้น MOVSXD จำเป็นสำหรับ signed 32-bit ที่เป็นลบเท่านั้น
เช่น: eax = -1 (0xFFFFFFFF)
MOV eax, -1           → rax = 0x00000000FFFFFFFF  (WRONG! ถ้าต้องการ -1 ใน 64-bit)
MOVSXD rax, eax       → rax = 0xFFFFFFFFFFFFFFFF  (CORRECT: -1 ใน 64-bit)
```

### 6. LEA - Load Effective Address

`LEA` (Load Effective Address) คำนวณ address แล้วเก็บผลลัพธ์ใน register โดย **ไม่ access memory จริง**

```
LEA reg, [expression]

ตัวอย่าง:
LEA eax, [ebx + ecx*4 + 8]
→ eax = ebx + ecx*4 + 8   (แค่คำนวณ ไม่ได้อ่าน memory)

เทียบกับ MOV:
MOV eax, [ebx + ecx*4 + 8]
→ eax = memory[ebx + ecx*4 + 8]  (อ่านค่าจาก memory จริงๆ)
```

**การใช้ LEA เป็น fast arithmetic:**
```
; แทน ADD + MUL ด้วย LEA เดียว
LEA eax, [eax + eax*2]      ; eax = eax * 3   (เร็วกว่า IMUL)
LEA eax, [eax + eax*4]      ; eax = eax * 5
LEA eax, [eax*8]            ; eax = eax * 8   (shift left 3)
LEA rax, [rbx + 16]         ; pointer arithmetic
LEA rdi, [rip + label]      ; RIP-relative addressing (Position Independent Code)
```

### 7. XCHG - Exchange (สลับค่า)

`XCHG` สลับค่าระหว่าง source และ destination

```
XCHG reg, reg    ; สลับค่าสอง registers
XCHG reg, mem    ; สลับระหว่าง register และ memory (atomic!)
XCHG mem, reg    ; เหมือนบรรทัดบน

หมายเหตุสำคัญ: XCHG กับ memory มี implicit LOCK prefix = atomic operation
เหมาะสำหรับ spinlock, semaphore implementation
```

**ตัวอย่างการ swap โดยไม่ใช้ temp variable:**
```
; swap eax และ ebx โดยไม่ใช้ temp
XCHG eax, ebx

; หรือใช้ XOR สลับ (แต่ XCHG สั้นกว่า)
XOR eax, ebx
XOR ebx, eax
XOR eax, ebx
```

### 8. BSWAP - Byte Swap (สลับ Byte)

`BSWAP` สลับลำดับ bytes ใน register เพื่อแปลงระหว่าง Little-Endian และ Big-Endian

```
BSWAP eax   ; สลับ 4 bytes ใน eax
BSWAP rax   ; สลับ 8 bytes ใน rax

ตัวอย่าง:
eax = 0x01020304

BSWAP eax

eax = 0x04030201

การใช้งาน: Network programming (network byte order = Big-Endian)
```

**Byte Order:**
```
Little-Endian (x86 native):   01 02 03 04  (LSB ที่ address ต่ำ)
Big-Endian (Network):         04 03 02 01  (MSB ที่ address ต่ำ)

BSWAP แปลงระหว่างสองแบบ
```

### 9. MOVBE - Move Byte with Byte Reversal

`MOVBE` (Available on newer CPUs: Haswell+) คือ load/store พร้อมกับ byte swap ในคำสั่งเดียว

```
MOVBE reg, mem    ; load จาก memory แล้ว byte-swap ใส่ register
MOVBE mem, reg    ; byte-swap แล้ว store ลง memory

เหมือนทำ: MOV + BSWAP ในคำสั่งเดียว
เหมาะสำหรับ: Network protocol parsing, Big-Endian data structures
```

### 10. CMOVcc - Conditional Move (การย้ายแบบมีเงื่อนไข)

`CMOVcc` ทำ MOV เฉพาะเมื่อ condition เป็นจริง เป็น **branchless** alternative ที่ดีกว่า `jcc + mov`

```
CMOVE  dst, src    ; Move if Equal (ZF=1)
CMOVNE dst, src    ; Move if Not Equal (ZF=0)
CMOVL  dst, src    ; Move if Less (signed: SF≠OF)
CMOVG  dst, src    ; Move if Greater (signed: ZF=0 AND SF=OF)
CMOVLE dst, src    ; Move if Less or Equal (signed: ZF=1 OR SF≠OF)
CMOVGE dst, src    ; Move if Greater or Equal (signed: SF=OF)
CMOVB  dst, src    ; Move if Below (unsigned: CF=1)
CMOVA  dst, src    ; Move if Above (unsigned: CF=0 AND ZF=0)
CMOVS  dst, src    ; Move if Sign (SF=1, negative)
CMOVNS dst, src    ; Move if Not Sign (SF=0, positive)
CMOVO  dst, src    ; Move if Overflow (OF=1)
CMOVNO dst, src    ; Move if Not Overflow (OF=0)
CMOVC  dst, src    ; Move if Carry (CF=1)
CMOVNC dst, src    ; Move if Not Carry (CF=0)
CMOVP  dst, src    ; Move if Parity (PF=1)
CMOVNP dst, src    ; Move if Not Parity (PF=0)
CMOVZ  dst, src    ; Move if Zero (ZF=1, same as CMOVE)
CMOVNZ dst, src    ; Move if Not Zero (ZF=0, same as CMOVNE)
```

**ทำไม CMOVcc ดีกว่า branch:**
```
; แบบใช้ branch (ช้าเพราะ Branch Misprediction)
CMP eax, ebx
JGE skip
MOV eax, ebx     ; เอาค่าที่น้อยกว่า
skip:

; แบบใช้ CMOVcc (ไม่มี branch, ไม่มี misprediction penalty)
CMP eax, ebx
CMOVGE eax, ebx  ; ถ้า eax >= ebx, copy ebx ไป eax
; ผลลัพธ์เหมือนกัน: eax = min(eax, ebx)
```

### 11. PUSH/POP Review

**PUSH** กดค่าลง stack (SP ลดลงก่อน แล้ว store):
```
PUSH reg/mem/imm

การทำงาน (32-bit):
1. ESP = ESP - 4
2. [ESP] = operand

การทำงาน (64-bit):
1. RSP = RSP - 8
2. [RSP] = operand
```

**POP** ดึงค่าจาก stack (load แล้ว SP เพิ่มขึ้น):
```
POP reg/mem

การทำงาน (32-bit):
1. operand = [ESP]
2. ESP = ESP + 4

การทำงาน (64-bit):
1. operand = [RSP]
2. RSP = RSP + 8
```

### 12. PUSHA/POPA และ PUSHAD/POPAD (32-bit เท่านั้น)

`PUSHA` (Push All 16-bit registers) กด AX, CX, DX, BX, SP, BP, SI, DI ลง stack
`POPA`  (Pop All 16-bit registers) ดึงกลับ (DI, SI, BP, SP, BX, DX, CX, AX)
`PUSHAD` (Push All 32-bit registers) กด EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
`POPAD` (Pop All 32-bit registers) ดึงกลับ

```
ลำดับ PUSHAD:
ESP ก่อน push  │ ─────────── │ ← ESP เดิม
               │     EAX     │
               │     ECX     │
               │     EDX     │
               │     EBX     │
               │   ESP_old   │ (ค่า ESP ก่อน PUSHAD)
               │     EBP     │
               │     ESI     │
               │     EDI     │ ← ESP ใหม่

หมายเหตุ: PUSHA/PUSHAD ไม่มีใน 64-bit mode (deprecated)
ใน 64-bit ต้อง PUSH แต่ละ register เอง
```

### 13. LAHF/SAHF - Load/Store AH with Flags

`LAHF` (Load AH from Flags) คัดลอก lower 8 bits ของ EFLAGS ไปยัง AH
`SAHF` (Store AH into Flags) คัดลอก AH กลับเข้า lower 8 bits ของ EFLAGS

```
Flags ที่ LAHF/SAHF จัดการ:
Bit 7: SF (Sign Flag)
Bit 6: ZF (Zero Flag)
Bit 4: AF (Adjust Flag)
Bit 2: PF (Parity Flag)
Bit 0: CF (Carry Flag)

การใช้งาน:
LAHF            ; AH = SF ZF 0 AF 0 PF 1 CF
; เก็บ flags ไว้ใน AH
SAHF            ; คืน flags จาก AH กลับ EFLAGS
```

**ประโยชน์:** บันทึก/คืน condition flags ชั่วคราวโดยไม่ใช้ stack

---

## Code Examples (ตัวอย่างโปรแกรม)

### Example 1: MOV Variants Showcase

```nasm
; ไฟล์: mov_variants.asm
; คำอธิบาย: แสดงการใช้ MOV รูปแบบต่างๆ
; Compile: nasm -f elf64 mov_variants.asm -o mov_variants.o
;          ld mov_variants.o -o mov_variants
; Run: ./mov_variants
; Expected output:
;   reg-to-reg: 42
;   mem-to-reg: 100
;   immediate: 255
;   computed address: 200

section .data
    ; ข้อมูลใน memory
    value1  dd 100          ; 32-bit integer = 100
    value2  db 0xFF         ; byte = 255
    array   dd 10, 20, 30, 40, 50  ; array of 32-bit ints
    
    ; Format strings สำหรับ printf
    fmt_rr  db "reg-to-reg: %d", 10, 0
    fmt_mr  db "mem-to-reg: %d", 10, 0
    fmt_im  db "immediate: %d", 10, 0
    fmt_ca  db "computed address: %d", 10, 0
    fmt_arr db "array[%d] = %d", 10, 0

section .bss
    result  resd 1          ; พื้นที่สำรองสำหรับผลลัพธ์

section .text
    extern printf
    global main

main:
    ; ── Prologue ──────────────────────────────────────────
    push rbp
    mov  rbp, rsp
    sub  rsp, 32            ; สร้าง shadow space / align stack

    ; ──────────────────────────────────────────────────────
    ; 1. MOV reg, reg  (register to register)
    ; ──────────────────────────────────────────────────────
    mov  eax, 42            ; load immediate 42 ลง eax
    mov  ebx, eax           ; copy eax ─► ebx  (reg to reg)
    ; ebx ตอนนี้ = 42

    ; เรียก printf เพื่อแสดงผล
    lea  rdi, [rel fmt_rr]  ; argument 1: format string
    mov  esi, ebx           ; argument 2: value (ebx = 42)
    xor  eax, eax           ; AL = 0 (ไม่มี vector args)
    call printf

    ; ──────────────────────────────────────────────────────
    ; 2. MOV reg, mem  (memory to register)
    ; ──────────────────────────────────────────────────────
    mov  eax, [value1]      ; โหลด 32-bit จาก memory variable value1 = 100
    ; eax ตอนนี้ = 100

    lea  rdi, [rel fmt_mr]
    mov  esi, eax           ; argument 2: 100
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; 3. MOV reg, imm  (immediate to register)
    ; ──────────────────────────────────────────────────────
    movzx eax, byte [value2] ; โหลด byte 0xFF และ zero-extend เป็น 32-bit
    ; eax = 0x000000FF = 255

    lea  rdi, [rel fmt_im]
    mov  esi, eax           ; argument 2: 255
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; 4. MOV กับ computed address (base + index * scale + disp)
    ; ──────────────────────────────────────────────────────
    ; เข้าถึง array[4] = 50
    lea  rbx, [rel array]   ; rbx = base address ของ array
    mov  ecx, 4             ; index = 4
    mov  eax, [rbx + rcx*4] ; array[4] = 50 (แต่ละ element คือ 4 bytes)
    ; eax = 50

    ; ──────────────────────────────────────────────────────
    ; 5. MOV mem, reg  (register to memory)
    ; ──────────────────────────────────────────────────────
    imul eax, eax, 4        ; eax = 50 * 4 = 200
    mov  [result], eax      ; เก็บ 200 ลงใน result variable
    mov  eax, [result]      ; โหลดกลับมาเพื่อยืนยัน

    lea  rdi, [rel fmt_ca]
    mov  esi, eax
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; 6. แสดง Array ทั้งหมด
    ; ──────────────────────────────────────────────────────
    xor  ecx, ecx           ; ecx = index = 0
    lea  rbx, [rel array]   ; rbx = base of array

.loop:
    cmp  ecx, 5             ; วนจนครบ 5 elements
    jge  .done

    mov  edx, [rbx + rcx*4] ; edx = array[ecx]
    
    lea  rdi, [rel fmt_arr]
    mov  esi, ecx           ; argument 2: index
    ; edx ใส่ใน rdx แต่ rdx ต้องเป็น 3rd arg
    ; System V AMD64 ABI: rdi, rsi, rdx, rcx, r8, r9
    mov  edx, [rbx + rcx*4] ; argument 3: value
    push rcx                ; บันทึก rcx ก่อน call (caller-saved)
    push rbx
    xor  eax, eax
    call printf
    pop  rbx
    pop  rcx

    inc  ecx
    jmp  .loop

.done:
    ; ── Epilogue ──────────────────────────────────────────
    mov  eax, 0
    leave
    ret
```

**คำอธิบาย:**
- `mov ebx, eax` — copy register to register
- `mov eax, [value1]` — load จาก memory (ใช้ square brackets)
- `movzx eax, byte [value2]` — load byte แล้ว zero-extend
- `mov [rbx + rcx*4]` — indexed memory access
- `mov [result], eax` — store to memory

---

### Example 2: MOVZX, MOVSX, MOVSXD Demo

```nasm
; ไฟล์: extend_demo.asm
; คำอธิบาย: แสดงความแตกต่างระหว่าง MOVZX, MOVSX, MOVSXD
; Compile: nasm -f elf64 extend_demo.asm -o extend_demo.o && ld extend_demo.o -o extend_demo
;          (ใช้ libc: nasm -f elf64 extend_demo.asm && gcc -no-pie -o extend_demo extend_demo.o)
; Expected output:
;   === Byte Extension Demo ===
;   byte value: 0xFF (255 unsigned, -1 signed)
;   MOVZX result: 255 (0x000000FF)
;   MOVSX result: -1  (0xFFFFFFFF)
;   === 16-bit Extension Demo ===
;   word value: 0x8000 (-32768 signed)
;   MOVZX result: 32768
;   MOVSX result: -32768
;   === 32-to-64 bit MOVSXD ===
;   dword: -100 → qword via MOVSXD: -100

section .data
    header1 db "=== Byte Extension Demo ===", 10, 0
    header2 db "=== 16-bit Extension Demo ===", 10, 0
    header3 db "=== 32-to-64 bit MOVSXD ===", 10, 0
    
    fmt_byte_in   db "byte value: 0x%02X (%u unsigned, %d signed)", 10, 0
    fmt_movzx_b   db "MOVZX result: %u (0x%08X)", 10, 0
    fmt_movsx_b   db "MOVSX result: %d (0x%08X)", 10, 0
    fmt_word_in   db "word value: 0x%04X (%d signed)", 10, 0
    fmt_movzx_w   db "MOVZX result: %u", 10, 0
    fmt_movsx_w   db "MOVSX result: %d", 10, 0
    fmt_movsxd    db "dword: %d -> qword via MOVSXD: %ld", 10, 0
    
    byte_val  db 0xFF       ; -1 ใน signed, 255 ใน unsigned
    word_val  dw 0x8000     ; -32768 ใน signed, 32768 ใน unsigned
    dword_val dd -100       ; -100 ใน signed 32-bit

section .text
    extern printf
    global main

main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 32

    ; ──────────────────────────────────────────────────────
    ; Section 1: Byte Extension
    ; ──────────────────────────────────────────────────────
    lea  rdi, [rel header1]
    xor  eax, eax
    call printf

    ; แสดงค่า byte ดิบ
    lea  rdi, [rel fmt_byte_in]
    movzx esi, byte [byte_val]  ; unsigned value ใน rsi
    movsx edx, byte [byte_val]  ; signed value ใน rdx
    mov  ecx, esi               ; copy unsigned อีกครั้งสำหรับ %02X
    xor  eax, eax
    call printf

    ; MOVZX: zero-extend 0xFF → 0x000000FF = 255
    movzx eax, byte [byte_val]  ; eax = 0x000000FF
    
    lea  rdi, [rel fmt_movzx_b]
    mov  esi, eax               ; %u: unsigned = 255
    mov  edx, eax               ; %08X: hex = 0x000000FF
    xor  eax, eax
    call printf

    ; MOVSX: sign-extend 0xFF → 0xFFFFFFFF = -1
    movsx eax, byte [byte_val]  ; eax = 0xFFFFFFFF
    
    lea  rdi, [rel fmt_movsx_b]
    mov  esi, eax               ; %d: signed = -1
    mov  edx, eax               ; %08X: hex = 0xFFFFFFFF
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; Section 2: Word (16-bit) Extension
    ; ──────────────────────────────────────────────────────
    lea  rdi, [rel header2]
    xor  eax, eax
    call printf

    ; แสดงค่า word ดิบ
    lea  rdi, [rel fmt_word_in]
    movzx esi, word [word_val]  ; 0x8000 = 32768 unsigned
    movsx edx, word [word_val]  ; 0x8000 = -32768 signed
    xor  eax, eax
    call printf

    ; MOVZX 16→32: 0x8000 → 0x00008000 = 32768
    movzx eax, word [word_val]
    lea  rdi, [rel fmt_movzx_w]
    mov  esi, eax
    xor  eax, eax
    call printf

    ; MOVSX 16→32: 0x8000 → 0xFFFF8000 = -32768
    movsx eax, word [word_val]
    lea  rdi, [rel fmt_movsx_w]
    mov  esi, eax
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; Section 3: MOVSXD (32→64 bit)
    ; ──────────────────────────────────────────────────────
    lea  rdi, [rel header3]
    xor  eax, eax
    call printf

    mov  eax, [dword_val]   ; eax = -100 (0xFFFFFF9C)
    
    ; ผิด: MOV + 32-bit dest → zero extends เป็น 64-bit
    ; rax = 0x00000000FFFFFF9C ≠ -100 ใน 64-bit

    ; ถูก: MOVSXD → sign extends เป็น 64-bit
    movsxd rax, dword [dword_val]   ; rax = 0xFFFFFFFFFFFFFF9C = -100

    lea  rdi, [rel fmt_movsxd]
    mov  esi, [dword_val]   ; 32-bit: -100
    mov  rsi, rax           ; 64-bit รอรับ: -100
    ; แก้: System V AMD64: rdi, rsi, rdx ... 
    ; rsi สำหรับ %d (32-bit), rdx สำหรับ %ld (64-bit)
    mov  edx, [dword_val]   ; first arg after format: signed 32
    mov  rcx, rax           ; second arg: signed 64
    ; อาจ error ใน calling convention - ใช้ format แยก
    
    ; วิธีง่ายกว่า: แสดงทีละบรรทัด
    lea  rdi, [rel fmt_movsxd]
    movsxd rsi, dword [dword_val]   ; rsi = sign-extended -100 (64-bit)
    mov  edx, esi                   ; rdx = value ที่ 2
    mov  rdx, rsi                   ; ใช้ 64-bit สำหรับ %ld
    xor  eax, eax
    call printf

    xor  eax, eax
    leave
    ret
```

---

### Example 3: LEA - Arithmetic Tricks

```nasm
; ไฟล์: lea_tricks.asm
; คำอธิบาย: การใช้ LEA สำหรับ arithmetic ที่เร็วกว่า MUL/ADD
; Compile: nasm -f elf64 lea_tricks.asm -o lea_tricks.o
;          gcc -no-pie -o lea_tricks lea_tricks.o
; Expected output:
;   LEA Arithmetic Tricks:
;   x * 2  = 20   (via LEA [x+x])
;   x * 3  = 30   (via LEA [x+x*2])
;   x * 4  = 40   (via LEA [x*4])
;   x * 5  = 50   (via LEA [x+x*4])
;   x * 9  = 90   (via LEA [x+x*8])
;   x + 7  = 17   (via LEA [x+7])
;   pointer math = 0x...

section .data
    title   db "LEA Arithmetic Tricks:", 10, 0
    fmt_mul db "x * %d = %d   (via LEA)", 10, 0
    fmt_add db "x + 7  = %d   (via LEA [x+7])", 10, 0
    fmt_ptr db "pointer math = %p", 10, 0
    x_val   dd 10               ; x = 10

section .text
    extern printf
    global main

main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16

    lea  rdi, [rel title]
    xor  eax, eax
    call printf

    ; โหลด x = 10
    mov  eax, [x_val]       ; eax = 10

    ; ─── x * 2 ────────────────────────────────────────────
    ; LEA ที่ใช้ addressing mode [reg + reg] = reg * 2
    lea  ecx, [eax + eax]   ; ecx = eax + eax = 10 + 10 = 20
    ; เร็วกว่า: IMUL ecx, eax, 2

    lea  rdi, [rel fmt_mul]
    mov  esi, 2
    mov  edx, ecx           ; 20
    xor  eax, eax
    call printf
    mov  eax, [x_val]       ; restore x

    ; ─── x * 3 ────────────────────────────────────────────
    ; LEA [reg + reg*2]: reg + 2*reg = 3*reg
    lea  ecx, [eax + eax*2] ; ecx = eax + 2*eax = 3*10 = 30

    lea  rdi, [rel fmt_mul]
    mov  esi, 3
    mov  edx, ecx           ; 30
    xor  eax, eax
    call printf
    mov  eax, [x_val]

    ; ─── x * 4 ────────────────────────────────────────────
    ; LEA [reg*4]: 4*reg (scale factor)
    lea  ecx, [eax*4]       ; ecx = 4 * eax = 40
    ; หมายเหตุ: [eax*4] ใช้ SIB byte กับ base=0

    lea  rdi, [rel fmt_mul]
    mov  esi, 4
    mov  edx, ecx
    xor  eax, eax
    call printf
    mov  eax, [x_val]

    ; ─── x * 5 ────────────────────────────────────────────
    ; LEA [reg + reg*4]: reg + 4*reg = 5*reg
    lea  ecx, [eax + eax*4] ; ecx = eax + 4*eax = 5*10 = 50

    lea  rdi, [rel fmt_mul]
    mov  esi, 5
    mov  edx, ecx
    xor  eax, eax
    call printf
    mov  eax, [x_val]

    ; ─── x * 9 ────────────────────────────────────────────
    ; LEA [reg + reg*8]: reg + 8*reg = 9*reg
    lea  ecx, [eax + eax*8] ; ecx = 9 * 10 = 90

    lea  rdi, [rel fmt_mul]
    mov  esi, 9
    mov  edx, ecx
    xor  eax, eax
    call printf
    mov  eax, [x_val]

    ; ─── x + 7 ────────────────────────────────────────────
    ; LEA กับ displacement: pointer arithmetic
    lea  ecx, [eax + 7]     ; ecx = eax + 7 = 17

    lea  rdi, [rel fmt_add]
    mov  esi, ecx
    xor  eax, eax
    call printf

    ; ─── Pointer Arithmetic ───────────────────────────────
    ; LEA ใช้คำนวณ pointer โดยไม่ modify base register
    lea  rbx, [rel x_val]   ; rbx = address ของ x_val
    lea  rcx, [rbx + 4]     ; rcx = address ของ element ถัดไป
    ; rbx ไม่ถูก modify!

    lea  rdi, [rel fmt_ptr]
    mov  rsi, rcx           ; pointer value
    xor  eax, eax
    call printf

    xor  eax, eax
    leave
    ret
```

**Expected Output:**
```
LEA Arithmetic Tricks:
x * 2  = 20   (via LEA)
x * 3  = 30   (via LEA)
x * 4  = 40   (via LEA)
x * 5  = 50   (via LEA)
x * 9  = 90   (via LEA)
x + 7  = 17   (via LEA [x+7])
pointer math = 0x[address]
```

---

### Example 4: XCHG, BSWAP, และ CMOVcc

```nasm
; ไฟล์: swap_cond_demo.asm
; คำอธิบาย: XCHG สำหรับ swap, BSWAP สำหรับ endian, CMOVcc สำหรับ branchless code
; Compile: nasm -f elf64 swap_cond_demo.asm -o swap_cond_demo.o
;          gcc -no-pie -o swap_cond_demo swap_cond_demo.o
; Expected output:
;   === XCHG Demo ===
;   Before swap: a=10, b=20
;   After  swap: a=20, b=10
;   === BSWAP Demo ===
;   Before: 0x01020304
;   After:  0x04030201
;   === CMOVcc Demo (branchless min/max) ===
;   min(30, 50) = 30
;   max(30, 50) = 50
;   abs(-42) = 42

section .data
    hdr_xchg    db "=== XCHG Demo ===", 10, 0
    hdr_bswap   db "=== BSWAP Demo ===", 10, 0
    hdr_cmov    db "=== CMOVcc Demo (branchless min/max) ===", 10, 0
    
    fmt_before  db "Before swap: a=%d, b=%d", 10, 0
    fmt_after   db "After  swap: a=%d, b=%d", 10, 0
    fmt_bswap_b db "Before: 0x%08X", 10, 0
    fmt_bswap_a db "After:  0x%08X", 10, 0
    fmt_min     db "min(%d, %d) = %d", 10, 0
    fmt_max     db "max(%d, %d) = %d", 10, 0
    fmt_abs     db "abs(%d) = %d", 10, 0

section .text
    extern printf
    global main

main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 32

    ; ──────────────────────────────────────────────────────
    ; XCHG Demo
    ; ──────────────────────────────────────────────────────
    lea  rdi, [rel hdr_xchg]
    xor  eax, eax
    call printf

    mov  eax, 10            ; a = 10
    mov  ebx, 20            ; b = 20

    ; แสดงก่อน swap
    lea  rdi, [rel fmt_before]
    mov  esi, eax           ; a
    mov  edx, ebx           ; b
    xor  eax, eax
    call printf

    ; swap โดยไม่ใช้ temp variable!
    mov  eax, 10
    mov  ebx, 20
    xchg eax, ebx           ; สลับ eax และ ebx ทันที
    ; ตอนนี้: eax=20, ebx=10

    ; แสดงหลัง swap
    lea  rdi, [rel fmt_after]
    mov  esi, eax           ; a = 20
    mov  edx, ebx           ; b = 10
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; BSWAP Demo - แปลง Little-Endian เป็น Big-Endian
    ; ──────────────────────────────────────────────────────
    lea  rdi, [rel hdr_bswap]
    xor  eax, eax
    call printf

    mov  eax, 0x01020304    ; Little-endian: 04 03 02 01 ใน memory

    ; แสดงก่อน
    push rax                ; บันทึก rax
    lea  rdi, [rel fmt_bswap_b]
    mov  esi, eax           ; 0x01020304
    xor  eax, eax
    call printf
    pop  rax

    bswap eax               ; สลับ byte: 0x04030201

    lea  rdi, [rel fmt_bswap_a]
    mov  esi, eax           ; 0x04030201
    xor  eax, eax
    call printf

    ; ──────────────────────────────────────────────────────
    ; CMOVcc Demo - Branchless min/max/abs
    ; ──────────────────────────────────────────────────────
    lea  rdi, [rel hdr_cmov]
    xor  eax, eax
    call printf

    ; ─── min(30, 50) ─────────────────────────────────────
    mov  eax, 30            ; eax = 30 (candidate 1)
    mov  ebx, 50            ; ebx = 50 (candidate 2)
    
    cmp  eax, ebx           ; เปรียบเทียบ eax vs ebx
    cmovg eax, ebx          ; ถ้า eax > ebx, copy ebx → eax
    ; ผล: eax = min(30, 50) = 30 ✓ (30 ไม่ > 50 ดังนั้น eax ยังเป็น 30)

    lea  rdi, [rel fmt_min]
    mov  esi, 30
    mov  edx, 50
    mov  ecx, eax           ; result
    xor  eax, eax
    call printf

    ; ─── max(30, 50) ─────────────────────────────────────
    mov  eax, 30            ; eax = 30
    mov  ebx, 50            ; ebx = 50
    
    cmp  eax, ebx
    cmovl eax, ebx          ; ถ้า eax < ebx, copy ebx → eax
    ; ผล: eax = max(30, 50) = 50 ✓ (30 < 50 ดังนั้น eax = 50)

    lea  rdi, [rel fmt_max]
    mov  esi, 30
    mov  edx, 50
    mov  ecx, eax
    xor  eax, eax
    call printf

    ; ─── abs(-42) ─────────────────────────────────────────
    ; Branchless absolute value
    mov  eax, -42           ; eax = -42

    mov  ebx, eax           ; ebx = -42 (copy)
    neg  ebx                ; ebx = 42  (negate)
    test eax, eax           ; set flags สำหรับ eax
    cmovs eax, ebx          ; ถ้า eax < 0 (SF=1), copy ebx (positive version)
    ; ผล: eax = 42 ✓

    lea  rdi, [rel fmt_abs]
    mov  esi, -42           ; original value
    mov  edx, eax           ; abs result
    xor  eax, eax
    call printf

    xor  eax, eax
    leave
    ret
```

---

### Example 5: LAHF/SAHF and PUSH/POP/PUSHAD

```nasm
; ไฟล์: flags_stack_demo.asm
; คำอธิบาย: LAHF/SAHF สำหรับ flags, PUSH/POP ต่างๆ
; Compile: nasm -f elf32 flags_stack_demo.asm -o flags_stack_demo.o
;          gcc -m32 -no-pie -o flags_stack_demo flags_stack_demo.o
; Run: ./flags_stack_demo
; Expected output:
;   === LAHF/SAHF Demo ===
;   After CMP 5,3: ZF=0, CF=0, SF=0
;   After CMP 3,5: ZF=0, CF=1, SF=1
;   After CMP 5,5: ZF=1, CF=0, SF=0
;   === PUSHAD/POPAD Demo ===
;   EAX before PUSHAD = 0xAABBCCDD
;   EAX after POPAD   = 0xAABBCCDD (restored)
;   === Stack Frame Demo ===
;   local1 = 100, local2 = 200

; NOTE: นี่คือ 32-bit code (PUSHAD/PUSHA ไม่มีใน 64-bit)
; ต้อง compile ด้วย -f elf32 และ link ด้วย -m32

section .data
    hdr1    db "=== LAHF/SAHF Demo ===", 10, 0
    hdr2    db "=== PUSHAD/POPAD Demo ===", 10, 0
    hdr3    db "=== Stack Frame Demo ===", 10, 0
    
    fmt_flags db "After CMP %d,%d: ZF=%d, CF=%d, SF=%d", 10, 0
    fmt_pushad_b db "EAX before PUSHAD = 0x%08X", 10, 0
    fmt_pushad_a db "EAX after POPAD   = 0x%08X (restored)", 10, 0
    fmt_local db "local1 = %d, local2 = %d", 10, 0

section .text
    extern printf
    global main

main:
    push ebp
    mov  ebp, esp
    sub  esp, 16            ; local variable space

    ; ──────────────────────────────────────────────────────
    ; LAHF/SAHF Demo
    ; ──────────────────────────────────────────────────────
    push hdr1
    call printf
    add  esp, 4

    ; Test 1: CMP 5, 3  (5 > 3: ZF=0, CF=0, SF=0)
    mov  eax, 5
    cmp  eax, 3             ; 5 - 3 = 2, ไม่ตรงเงื่อนไขใด
    lahf                    ; AH = lower 8 bits ของ EFLAGS
    ; Bit 7=SF, 6=ZF, 4=AF, 2=PF, 0=CF
    
    ; Extract flags จาก AH
    mov  bl, ah             ; บันทึก AH
    
    ; ZF = bit 6 ของ AH
    mov  cl, bl
    shr  cl, 6
    and  cl, 1              ; cl = ZF
    
    ; CF = bit 0 ของ AH  
    mov  dl, bl
    and  dl, 1              ; dl = CF
    
    ; SF = bit 7 ของ AH
    shr  bl, 7              ; bl = SF

    ; zero-extend สำหรับ printf
    movzx ecx, cl           ; ZF
    movzx edx, dl           ; CF
    movzx ebx, bl           ; SF

    push ebx                ; SF
    push edx                ; CF
    push ecx                ; ZF
    push dword 3            ; b
    push dword 5            ; a
    push fmt_flags
    call printf
    add  esp, 24

    ; ──────────────────────────────────────────────────────
    ; PUSHAD/POPAD Demo
    ; ──────────────────────────────────────────────────────
    push hdr2
    call printf
    add  esp, 4

    mov  eax, 0xAABBCCDD    ; set EAX ค่าพิเศษ

    push eax                ; บันทึกค่า eax สำหรับแสดง
    push fmt_pushad_b
    call printf
    add  esp, 4
    pop  eax                ; คืน eax = 0xAABBCCDD

    ; เก็บ all registers ลง stack
    pushad                  ; Push EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI

    ; เปลี่ยนค่า registers ทั้งหมด
    mov  eax, 0x11111111
    mov  ecx, 0x22222222
    mov  edx, 0x33333333
    mov  ebx, 0x44444444
    ; EAX ถูกเปลี่ยนแล้ว!

    ; คืนค่า registers ทั้งหมด
    popad                   ; Pop ทุกอย่างกลับ: EAX คืนเป็น 0xAABBCCDD!

    push eax
    push fmt_pushad_a
    call printf
    add  esp, 8

    ; ──────────────────────────────────────────────────────
    ; Stack Frame Demo: Local Variables ด้วย PUSH/POP
    ; ──────────────────────────────────────────────────────
    push hdr3
    call printf
    add  esp, 4

    ; สร้าง local variables บน stack
    push dword 100          ; local1 = 100  (ebp-4 เพราะ push ทำ esp-4)
    push dword 200          ; local2 = 200  (ebp-8)

    ; อ่าน local variables
    mov  eax, [esp]         ; local2 = 200 (ล่าสุดที่ push)
    mov  ebx, [esp+4]       ; local1 = 100

    push eax                ; local2
    push ebx                ; local1
    push fmt_local
    call printf
    add  esp, 12

    ; ล้าง local variables
    add  esp, 8             ; เอา 2 dword ออกจาก stack

    ; ── Epilogue ──
    xor  eax, eax
    mov  esp, ebp
    pop  ebp
    ret
```

---

### Example 6: Practical - memcpy Implementation

```nasm
; ไฟล์: my_memcpy.asm
; คำอธิบาย: Implement memcpy ด้วย Assembly - ตัวอย่าง Data Movement จริงๆ
; ใช้ MOV ขนาดต่างๆ เพื่อประสิทธิภาพสูงสุด
;
; Compile: nasm -f elf64 my_memcpy.asm -o my_memcpy.o
;          gcc -no-pie -o my_memcpy my_memcpy.o
;
; Expected output:
;   Original: Hello, World! Assembly memcpy test.
;   Copy1 (basic):    Hello, World! Assembly memcpy test.
;   Copy2 (optimized): Hello, World! Assembly memcpy test.
;   Copy3 (16-byte):  Hello, World! Assembly memcpy test.
;   Copying integers: [10][20][30][40][50]
;   memcpy_int test passed!

section .data
    src_str     db "Hello, World! Assembly memcpy test.", 0
    src_len     equ $ - src_str - 1    ; length without null

    int_src     dd 10, 20, 30, 40, 50  ; 5 ints = 20 bytes
    
    msg_orig    db "Original: ", 0
    msg_copy1   db "Copy1 (basic):    ", 0
    msg_copy2   db "Copy2 (optimized): ", 0
    msg_copy3   db "Copy3 (16-byte):  ", 0
    msg_int     db "Copying integers: ", 0
    msg_passed  db "memcpy_int test passed!", 10, 0
    msg_failed  db "memcpy_int test FAILED!", 10, 0
    msg_newline db 10, 0
    
    fmt_int     db "[%d]", 0

section .bss
    dst1    resb 64         ; ปลายทาง buffer 1 (basic memcpy)
    dst2    resb 64         ; ปลายทาง buffer 2 (optimized)
    dst3    resb 64         ; ปลายทาง buffer 3 (16-byte aligned)
    int_dst resd 5          ; int array destination

section .text
    extern printf, write
    global main

; ─────────────────────────────────────────────────────────
; Function: memcpy_basic
; คัดลอก n bytes จาก src ไปยัง dst แบบ byte-by-byte
; Arguments: rdi = dst, rsi = src, rdx = n (count)
; Returns: rdi (pointer to dst)
; Clobbers: rax, rcx
; ─────────────────────────────────────────────────────────
memcpy_basic:
    push rbp
    mov  rbp, rsp

    ; บันทึก destination pointer สำหรับ return
    mov  rax, rdi           ; return value = dst

    ; ตรวจสอบ count = 0
    test rdx, rdx
    jz   .done_basic

    ; rcx = counter (count down จาก n ถึง 0)
    mov  rcx, rdx

.loop_basic:
    ; copy byte ทีละ byte
    mov  al, [rsi]          ; โหลด byte จาก source
    mov  [rdi], al          ; เก็บ byte ลง destination

    inc  rdi                ; dst pointer เดินหน้า
    inc  rsi                ; src pointer เดินหน้า
    dec  rcx                ; counter--
    jnz  .loop_basic        ; ถ้ายังไม่ถึง 0 ให้ loop ต่อ

.done_basic:
    pop  rbp
    ret

; ─────────────────────────────────────────────────────────
; Function: memcpy_optimized
; คัดลอก n bytes แบบ optimized:
;   - คัดลอก 8 bytes (QWORD) ทีละครั้งก่อน
;   - แล้วจัดการ remainder ทีละ byte
; Arguments: rdi = dst, rsi = src, rdx = n
; Returns: rdi (original dst pointer)
; ─────────────────────────────────────────────────────────
memcpy_optimized:
    push rbp
    mov  rbp, rsp
    push rbx                ; save callee-saved register

    mov  rbx, rdi           ; เก็บ original dst สำหรับ return
    mov  rcx, rdx           ; rcx = total byte count

    ; ── Phase 1: copy 8 bytes (QWORD) ต่อรอบ ─────────────
    mov  rax, rcx
    shr  rax, 3             ; rax = n / 8  (จำนวน QWORD ที่ copy ได้)
    jz   .small_bytes       ; ถ้า n < 8 ข้ามไป byte loop

.qword_loop:
    mov  r8, [rsi]          ; โหลด 8 bytes จาก source
    mov  [rdi], r8          ; เก็บ 8 bytes ลง destination
    add  rsi, 8             ; src += 8
    add  rdi, 8             ; dst += 8
    dec  rax
    jnz  .qword_loop

.small_bytes:
    ; ── Phase 2: copy remaining bytes (0-7) ──────────────
    and  rcx, 7             ; rcx = n % 8  (จำนวน bytes ที่เหลือ)
    jz   .done_opt          ; ถ้าไม่มี remainder ก็เสร็จ

.byte_loop:
    mov  al, [rsi]          ; copy ทีละ byte
    mov  [rdi], al
    inc  rsi
    inc  rdi
    dec  rcx
    jnz  .byte_loop

.done_opt:
    mov  rax, rbx           ; return original dst
    pop  rbx
    pop  rbp
    ret

; ─────────────────────────────────────────────────────────
; Function: memcpy_16byte
; คัดลอกแบบ 16 bytes ต่อรอบ (เหมือน SSE movdqu แต่ใช้ 2x QWORD)
; Arguments: rdi = dst, rsi = src, rdx = n
; ─────────────────────────────────────────────────────────
memcpy_16byte:
    push rbp
    mov  rbp, rsp
    push rbx
    push r12

    mov  rbx, rdi           ; บันทึก original dst
    mov  rcx, rdx

    ; ── Phase 1: copy 16 bytes ต่อรอบ ────────────────────
    mov  rax, rcx
    shr  rax, 4             ; rax = n / 16
    jz   .tail_16

.loop_16:
    ; โหลด/เก็บ 16 bytes ในครั้งเดียว (2 QWORD)
    mov  r8,  [rsi]         ; โหลด 8 bytes แรก
    mov  r9,  [rsi+8]       ; โหลด 8 bytes หลัง
    mov  [rdi],   r8        ; เก็บ 8 bytes แรก
    mov  [rdi+8], r9        ; เก็บ 8 bytes หลัง
    add  rsi, 16
    add  rdi, 16
    dec  rax
    jnz  .loop_16

.tail_16:
    ; ── Phase 2: copy remaining 0-15 bytes ───────────────
    and  rcx, 15            ; rcx = n % 16

    ; ถ้าเหลือ >= 8 bytes
    cmp  rcx, 8
    jl   .tail_bytes

    mov  r8, [rsi]          ; copy 8 bytes
    mov  [rdi], r8
    add  rsi, 8
    add  rdi, 8
    sub  rcx, 8

.tail_bytes:
    ; copy bytes ที่เหลือ (0-7)
    test rcx, rcx
    jz   .done_16

.byte_rem:
    mov  al, [rsi]
    mov  [rdi], al
    inc  rsi
    inc  rdi
    dec  rcx
    jnz  .byte_rem

.done_16:
    mov  rax, rbx
    pop  r12
    pop  rbx
    pop  rbp
    ret

; ─────────────────────────────────────────────────────────
; Main Program
; ─────────────────────────────────────────────────────────
main:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16

    ; ── แสดง Original ────────────────────────────────────
    lea  rdi, [rel msg_orig]
    xor  eax, eax
    call printf

    ; write(1, src_str, src_len) - แสดง original string
    mov  eax, 1             ; syscall: write
    mov  edi, 1             ; fd: stdout
    lea  rsi, [rel src_str]
    mov  edx, src_len
    syscall
    
    ; แสดง newline
    lea  rdi, [rel msg_newline]
    xor  eax, eax
    call printf

    ; ── Copy 1: Basic byte-by-byte ───────────────────────
    lea  rdi, [rel dst1]            ; dst
    lea  rsi, [rel src_str]         ; src
    mov  rdx, src_len               ; count (ไม่รวม null)
    call memcpy_basic
    mov  byte [dst1 + src_len], 0   ; เพิ่ม null terminator

    lea  rdi, [rel msg_copy1]
    xor  eax, eax
    call printf

    lea  rdi, [rel dst1]
    xor  eax, eax
    call printf
    
    lea  rdi, [rel msg_newline]
    xor  eax, eax
    call printf

    ; ── Copy 2: Optimized QWORD ──────────────────────────
    lea  rdi, [rel dst2]
    lea  rsi, [rel src_str]
    mov  rdx, src_len
    call memcpy_optimized
    mov  byte [dst2 + src_len], 0

    lea  rdi, [rel msg_copy2]
    xor  eax, eax
    call printf

    lea  rdi, [rel dst2]
    xor  eax, eax
    call printf

    lea  rdi, [rel msg_newline]
    xor  eax, eax
    call printf

    ; ── Copy 3: 16-byte chunks ───────────────────────────
    lea  rdi, [rel dst3]
    lea  rsi, [rel src_str]
    mov  rdx, src_len
    call memcpy_16byte
    mov  byte [dst3 + src_len], 0

    lea  rdi, [rel msg_copy3]
    xor  eax, eax
    call printf

    lea  rdi, [rel dst3]
    xor  eax, eax
    call printf

    lea  rdi, [rel msg_newline]
    xor  eax, eax
    call printf

    ; ── Integer Array Copy ───────────────────────────────
    lea  rdi, [rel int_dst]
    lea  rsi, [rel int_src]
    mov  rdx, 20            ; 5 * 4 bytes
    call memcpy_optimized

    lea  rdi, [rel msg_int]
    xor  eax, eax
    call printf

    ; แสดง copied integers
    xor  ecx, ecx
    lea  rbx, [rel int_dst]

.print_ints:
    cmp  ecx, 5
    jge  .done_print

    mov  edx, [rbx + rcx*4]
    push rcx
    push rbx
    lea  rdi, [rel fmt_int]
    mov  esi, edx
    xor  eax, eax
    call printf
    pop  rbx
    pop  rcx
    inc  ecx
    jmp  .print_ints

.done_print:
    lea  rdi, [rel msg_newline]
    xor  eax, eax
    call printf

    ; ── ตรวจสอบความถูกต้อง ──────────────────────────────
    ; เปรียบเทียบ int_dst กับ int_src
    xor  ecx, ecx
    lea  rbx, [rel int_src]
    lea  r12, [rel int_dst]

.verify:
    cmp  ecx, 5
    jge  .verify_pass

    mov  eax, [rbx + rcx*4]
    mov  edx, [r12 + rcx*4]
    cmp  eax, edx
    jne  .verify_fail
    inc  ecx
    jmp  .verify

.verify_pass:
    lea  rdi, [rel msg_passed]
    xor  eax, eax
    call printf
    jmp  .exit

.verify_fail:
    lea  rdi, [rel msg_failed]
    xor  eax, eax
    call printf

.exit:
    xor  eax, eax
    leave
    ret
```

**Compilation and Running:**
```bash
# Compile
nasm -f elf64 my_memcpy.asm -o my_memcpy.o
gcc -no-pie -o my_memcpy my_memcpy.o

# Run
./my_memcpy
```

**Expected Output:**
```
Original: Hello, World! Assembly memcpy test.
Copy1 (basic):    Hello, World! Assembly memcpy test.
Copy2 (optimized): Hello, World! Assembly memcpy test.
Copy3 (16-byte):  Hello, World! Assembly memcpy test.
Copying integers: [10][20][30][40][50]
memcpy_int test passed!
```

---

## Common Mistakes and Pitfalls (ข้อผิดพลาดที่พบบ่อย)

### Mistake 1: Memory-to-Memory MOV

```nasm
; ❌ ผิด: ไม่สามารถ MOV จาก memory ไป memory โดยตรง
MOV [dst], [src]            ; ERROR! nasm จะ error ทันที

; ✓ ถูก: ต้องใช้ register เป็นตัวกลาง
MOV rax, [src]              ; โหลดจาก source ก่อน
MOV [dst], rax              ; แล้วจึงเก็บลง destination
```

### Mistake 2: Size Mismatch

```nasm
; ❌ ผิด: ขนาดไม่ตรงกัน
MOV ax, [mem32]             ; ax คือ 16-bit แต่ mem32 คือ 32-bit

; ✓ ถูก: ขนาดต้องตรงกัน
MOV eax, [mem32]            ; eax (32-bit) = mem32 (32-bit)
MOV ax,  [mem16]            ; ax (16-bit) = mem16 (16-bit)
MOV al,  [mem8]             ; al (8-bit)  = mem8 (8-bit)
```

### Mistake 3: ลืม Size Specifier กับ Memory Immediate

```nasm
; ❌ ผิด: nasm ไม่รู้ขนาดของ memory operand
MOV [ptr], 42               ; error: operation size not specified

; ✓ ถูก: ต้องระบุขนาด
MOV BYTE  [ptr], 42         ; 1 byte
MOV WORD  [ptr], 42         ; 2 bytes
MOV DWORD [ptr], 42         ; 4 bytes
MOV QWORD [ptr], 42         ; 8 bytes
```

### Mistake 4: MOVZX/MOVSX Source Size

```nasm
; ❌ ผิด: MOVZX ต้องการ source ที่เล็กกว่า destination
MOVZX eax, ebx              ; ERROR! source ต้องเป็น 8 หรือ 16 bit

; ✓ ถูก:
MOVZX eax, bx               ; eax ← zero-extend bx (16-bit → 32-bit)
MOVZX eax, bl               ; eax ← zero-extend bl (8-bit → 32-bit)
MOVZX rax, eax              ; ไม่มี MOVZX 32→64! ใช้ MOV eax,eax แทน
; Note: MOV eax, eax จะ zero-extend เป็น rax อัตโนมัติใน 64-bit mode
```

### Mistake 5: XCHG กับ Memory = Atomic (ช้า!)

```nasm
; ⚠️ ระวัง: XCHG กับ memory เป็น ATOMIC (มี implicit LOCK)
; มีผลต่อ performance อย่างมาก!
XCHG eax, [mem]             ; SLOW! เทียบเท่า LOCK XCHG

; ถ้าไม่ต้องการ atomicity ใช้ MOV แทน
MOV  ebx, [mem]             ; โหลดก่อน
XCHG eax, ebx               ; swap registers (ไม่ lock)
MOV  [mem], ebx             ; เก็บกลับ
```

### Mistake 6: LEA กับ Memory Access

```nasm
; ❌ ความเข้าใจผิด: คิดว่า LEA อ่าน memory
LEA eax, [ebx]              ; eax = ebx  (ไม่ได้อ่าน memory จาก [ebx]!)
; ถ้าต้องการอ่าน memory ต้องใช้ MOV
MOV eax, [ebx]              ; eax = memory[ebx]

; ✓ ใช้ LEA อย่างถูกต้อง
LEA eax, [ebx + ecx*4 + 8]  ; eax = ebx + ecx*4 + 8  (แค่ arithmetic!)
```

### Mistake 7: CMOVcc ต้องใช้กับ Register Source เท่านั้น (ส่วนใหญ่)

```nasm
; ⚠️ หมายเหตุ: CMOVcc สามารถใช้กับ memory source ได้ใน x86-64
; แต่บาง assembler/context อาจต้องระบุ size
CMOVGE eax, [mem32]         ; อาจต้องระบุ: CMOVGE eax, DWORD [mem32]

; ✓ safe approach: load ก่อน แล้วใช้ register
MOV  ebx, [mem32]
CMP  eax, 0
CMOVL eax, ebx
```

### Mistake 8: BSWAP กับ 16-bit (ไม่มี!)

```nasm
; ❌ ผิด: ไม่มี BSWAP สำหรับ 16-bit
BSWAP ax                    ; UNDEFINED BEHAVIOR! (ax เป็น lower 16 of eax)

; ✓ ถูก: ใช้ XCHG al, ah สำหรับ 16-bit byte swap
XCHG al, ah                 ; swap bytes ของ ax (2 bytes)

; BSWAP รองรับแค่ 32-bit และ 64-bit
BSWAP eax                   ; OK: swap 4 bytes ใน eax
BSWAP rax                   ; OK: swap 8 bytes ใน rax
```

### Mistake 9: PUSHAD/POPAD ไม่มีใน 64-bit

```nasm
; ❌ ผิด: PUSHAD/POPAD ไม่รองรับใน 64-bit mode
; nasm จะ error เมื่อ compile ด้วย -f elf64
PUSHAD                      ; ERROR ใน 64-bit!
POPAD

; ✓ ถูก: push/pop แต่ละ register เอง ใน 64-bit
push rax
push rcx
push rdx
push rbx
push rsi
push rdi
push r8
push r9
; ... ทำ work ...
pop  r9
pop  r8
pop  rdi
pop  rsi
pop  rbx
pop  rdx
pop  rcx
pop  rax
```

---

## Advanced Techniques (เทคนิคขั้นสูง)

### 1. REP MOVS - String Copy Instructions

สำหรับ memcpy ขนาดใหญ่ x86 มี string instruction พิเศษ:

```nasm
; REP MOVSB: copy RCX bytes จาก [RSI] ไปยัง [RDI]
; ทำงานแบบ hardware-accelerated loop

; Setup:
; RSI = source pointer
; RDI = destination pointer
; RCX = byte count

cld                         ; clear direction flag (DF=0, copy forward)
rep movsb                   ; copy RCX bytes, RSI++ RDI-- ทุกรอบ

; Variants:
; MOVSB = copy 1 byte per iteration
; MOVSW = copy 2 bytes per iteration  
; MOVSD = copy 4 bytes per iteration
; MOVSQ = copy 8 bytes per iteration (64-bit)

; ตัวอย่าง fast memcpy:
lea  rdi, [dst]
lea  rsi, [src]
mov  rcx, count
; copy 8 bytes ต่อรอบ (ถ้า count divisible by 8)
shr  rcx, 3                 ; rcx /= 8
rep  movsq                  ; copy QWORD (8-byte) chunks
; handle remainder...
mov  rcx, [count_saved]
and  rcx, 7
rep  movsb                  ; copy remaining bytes
```

### 2. MOVNTI - Non-Temporal Store

สำหรับการ copy ข้อมูลขนาดใหญ่ที่ไม่ต้องการ cache:

```nasm
; MOVNTI เขียนข้อมูลลง memory โดย bypass cache (write-combining)
; เหมาะสำหรับ streaming store ขนาดใหญ่

MOVNTI [rdi], rax           ; store rax ลง [rdi] แบบ non-temporal
MOVNTI [rdi+8], rbx
; ต้อง MFENCE หลังจาก MOVNTI series เสมอ
MFENCE                      ; memory fence: ensure writes complete
```

### 3. CMOV สำหรับ Sorting Network (Branchless Sort)

```nasm
; Branchless swap สำหรับ sorting network
; เปรียบเทียบและ swap ถ้าจำเป็น โดยไม่มี branch

; compare-and-swap: ensure a <= b
; input: eax = a, ebx = b
; output: eax = min(a,b), ebx = max(a,b)

compare_and_swap:
    mov  ecx, eax           ; ecx = a (backup)
    cmp  eax, ebx           ; set flags
    CMOVG eax, ebx          ; if a > b: eax = b (min)
    CMOVG ebx, ecx          ; if a > b: ebx = a (max)
    ret
    ; ไม่มี branch ใดเลย! CPU สามารถ execute แบบ speculative ได้
```

### 4. LEA สำหรับ Position-Independent Code (PIC)

```nasm
; ใน 64-bit code, ใช้ RIP-relative addressing สำหรับ PIC
; LEA + [rel label] = address ที่ relocatable

lea  rax, [rel data_label]  ; rax = RIP + offset_to_data_label
                             ; ทำงานถูกต้องไม่ว่า code จะโหลดที่ address ไหน

; เทียบกับ 32-bit (absolute address, ไม่ PIC):
mov  eax, data_label        ; eax = absolute address (ใช้ได้แค่ -f elf32)
```

### 5. XADD - Exchange and Add (Atomic Increment)

```nasm
; XADD: exchange แล้ว add — เป็น atomic operation
; เหมาะสำหรับ lock-free counter

; lock xadd [counter], eax   ; atomic: tmp = counter; counter += eax; eax = tmp
; ใช้ใน threading, reference counting

section .data
    counter dd 0

increment_counter:
    mov  eax, 1             ; increment ทีละ 1
    lock xadd [counter], eax ; atomic increment
    ; eax ตอนนี้ = ค่าเดิมก่อน increment (old value)
    ret
```

### 6. MOV กับ Segment Registers

```nasm
; การโหลด Segment Register (16-bit operation เท่านั้น!)
; ใช้บ่อยใน OS kernel / bootloader

mov  ax, 0x10               ; data segment selector
mov  ds, ax                 ; โหลด DS ต้องผ่าน general register!
mov  es, ax                 ; โหลด ES
mov  ss, ax                 ; โหลด SS (ระวัง: disable interrupt ชั่วคราว)

; ดึงค่า Segment Register
mov  ax, cs                 ; อ่านค่า CS ออกมา
mov  ax, ds                 ; อ่านค่า DS

; FS และ GS ใช้สำหรับ Thread Local Storage ใน modern OS:
mov  rax, [fs:0]            ; อ่าน TLS base pointer (Linux)
mov  rax, [gs:0x30]         ; Windows TEB (Thread Environment Block)
```

### 7. MOVBE สำหรับ Network Programming

```nasm
; MOVBE: Load/Store with byte reversal (x86 ↔ Network byte order)
; ต้องใช้ CPU ที่รองรับ (Haswell+)
; ตรวจสอบด้วย CPUID bit: MOVBE in ECX bit 22 (leaf 1)

; อ่าน 4-byte network packet field (big-endian) เข้า little-endian register:
movbe eax, [packet_field]   ; อ่านแล้ว byte-swap อัตโนมัติ
; เทียบเท่ากับ:
; mov eax, [packet_field]
; bswap eax

; เขียน little-endian ออก network (big-endian):
movbe [out_field], eax      ; byte-swap แล้วเขียน
```

---

## Exercises (แบบฝึกหัด)

### Exercise 1: Data Type Conversion
**โจทย์:** เขียนโปรแกรม Assembly ที่รับ array ของ `signed byte` (-128 ถึง 127) และแปลงทุกค่าเป็น `signed int` (32-bit) โดยใช้ MOVSX

**Hints:**
- ใช้ MOVSX เพื่อ sign-extend จาก 8-bit เป็น 32-bit
- Loop ผ่าน array ด้วย indexed addressing
- อย่าลืม sign-extend ค่าลบด้วย

```nasm
; Template:
section .data
    byte_arr    db -5, 127, -128, 0, 42, -1, 100, -100
    arr_len     equ 8
    
section .bss
    int_arr     resd 8          ; ผลลัพธ์ 32-bit array

section .text
global my_convert
my_convert:
    ; TODO: ใช้ MOVSX เพื่อ convert byte_arr → int_arr
    ; loop ecx จาก 0 ถึง arr_len-1
    ; movsx eax, byte [byte_arr + rcx]
    ; mov [int_arr + rcx*4], eax
    ret
```

---

### Exercise 2: Branchless Min/Max Array
**โจทย์:** เขียนฟังก์ชัน `find_min_max(array, length)` ที่หาค่า min และ max ของ array integers โดยใช้ CMOVcc (ห้ามใช้ branch conditions แบบ JMP)

**Hints:**
- Compare แต่ละ element กับ current min/max
- ใช้ CMOVL สำหรับ update min
- ใช้ CMOVG สำหรับ update max

```nasm
; Template:
; rdi = array pointer, rsi = length
; คืน: rax = min (lower 32 bits), rdx = max (upper 32... หรือแยก register)
find_min_max:
    mov  eax, [rdi]         ; min = array[0]
    mov  edx, [rdi]         ; max = array[0]
    mov  rcx, 1             ; start from index 1
    
.loop:
    cmp  rcx, rsi
    jge  .done
    
    mov  r8d, [rdi + rcx*4] ; current element
    ; TODO: ใช้ CMOVL, CMOVG
    
    inc  rcx
    jmp  .loop
.done:
    ret
```

---

### Exercise 3: BSWAP สำหรับ Network Protocol
**โจทย์:** เขียนโปรแกรมที่จำลองการรับ IPv4 header (partial) จาก network และแปลงค่า fields จาก Big-Endian เป็น Little-Endian โดยใช้ BSWAP

**IPv4 Header Fields ที่ต้องแปลง:**
- Total Length (2 bytes) - offset 2
- Identification (2 bytes) - offset 4
- Source IP (4 bytes) - offset 12
- Dest IP (4 bytes) - offset 16

**Hints:**
- BSWAP ทำงานกับ 32-bit และ 64-bit เท่านั้น
- สำหรับ 16-bit ใช้ XCHG al, ah หลังจาก MOVZX
- ลำดับ: โหลดด้วย MOVZX/MOV, แล้ว BSWAP หรือ XCHG

```nasm
; Template:
section .data
    ; Simulated network packet (big-endian)
    ;             VER  LEN  TOTLEN  ID      FLAGS  TTL  PROTO CHKSUM
    ipv4_header db 0x45,0x00,0x00,0x28, 0xAB,0xCD,0x40,0x00, 0x40,0x06,0x00,0x00
    ;             SRC_IP              DST_IP
                db 0xC0,0xA8,0x01,0x01, 0xC0,0xA8,0x01,0x02
    ; 192.168.1.1 → 192.168.1.2
```

---

### Exercise 4: Stack-Based Calculator
**โจทย์:** เขียน stack-based calculator ใน Assembly ที่รองรับการ PUSH ค่า, POP, ADD, SUB โดยใช้ PUSH/POP instructions จริง

**Features ที่ต้องมี:**
1. `push_val n` — push integer n ลง stack
2. `pop_val` — pop ค่าออกจาก stack
3. `add_top` — pop สองค่า บวกกัน แล้ว push ผลลัพธ์
4. `sub_top` — pop สองค่า ลบกัน แล้ว push ผลลัพธ์

**Hints:**
- ใช้ software stack (array) แยกจาก hardware stack
- ใช้ pointer register (rbx) ชี้ stack top
- ทดสอบด้วย sequence: push 10, push 20, add → result = 30

---

### Exercise 5: Efficient XCHG Sorting (Bubble Sort)
**โจทย์:** เขียน Bubble Sort สำหรับ integer array โดยใช้ XCHG สำหรับการ swap elements

**Hints:**
- ใช้ XCHG eax, ebx เพื่อ swap registers
- ใช้ MOV เพื่อ load จาก memory, XCHG เพื่อ swap, MOV เพื่อ store กลับ
- วน loop 2 ชั้น: outer loop จาก n-1 ลงมา, inner loop ตรวจ adjacent pairs

```nasm
; Template:
; rdi = array pointer, rsi = length
bubble_sort:
    push rbp
    mov  rbp, rsp
    
    ; outer loop: i จาก n-1 ถึง 1
    mov  r10, rsi           ; r10 = outer counter
    dec  r10
    
.outer:
    test r10, r10
    jz   .done
    
    ; inner loop: j จาก 0 ถึง i-1
    xor  r11, r11           ; j = 0
    
.inner:
    cmp  r11, r10
    jge  .next_outer
    
    ; โหลด array[j] และ array[j+1]
    mov  eax, [rdi + r11*4]
    mov  ebx, [rdi + r11*4 + 4]
    
    ; ถ้า array[j] > array[j+1] ให้ swap
    cmp  eax, ebx
    jle  .no_swap
    
    ; TODO: ใช้ XCHG eax, ebx แล้ว store กลับ
    
.no_swap:
    inc  r11
    jmp  .inner
    
.next_outer:
    dec  r10
    jmp  .outer
    
.done:
    pop  rbp
    ret
```

---

## Summary (สรุป)

### ตารางสรุป Data Movement Instructions

| Instruction | การทำงาน | ขนาดที่รองรับ | หมายเหตุ |
|------------|----------|--------------|---------|
| `MOV` | คัดลอกข้อมูล | 8/16/32/64-bit | พื้นฐานที่สุด |
| `MOVZX` | คัดลอก + zero-extend | src: 8/16-bit, dst: 16/32/64-bit | สำหรับ unsigned |
| `MOVSX` | คัดลอก + sign-extend | src: 8/16-bit, dst: 16/32/64-bit | สำหรับ signed |
| `MOVSXD` | sign-extend 32→64 bit | src: 32-bit, dst: 64-bit | 64-bit mode only |
| `LEA` | คำนวณ address | 16/32/64-bit result | ไม่ access memory |
| `XCHG` | สลับค่า | 8/16/32/64-bit | atomic ถ้า memory operand |
| `BSWAP` | สลับ byte order | 32/64-bit เท่านั้น | Little↔Big Endian |
| `MOVBE` | Load/Store + byte swap | 16/32/64-bit | ต้องการ Haswell+ |
| `CMOVcc` | MOV แบบมีเงื่อนไข | 16/32/64-bit | ไม่มี branch |
| `PUSH` | กดลง stack | 16/32/64-bit | SP ลดลงก่อน |
| `POP` | ดึงจาก stack | 16/32/64-bit | SP เพิ่มหลัง |
| `PUSHAD` | กด all regs | 32-bit | 32-bit mode only |
| `POPAD` | ดึง all regs | 32-bit | 32-bit mode only |
| `LAHF` | Load flags → AH | 8-bit | 5 flags เท่านั้น |
| `SAHF` | Store AH → flags | 8-bit | คืน flags จาก AH |

### Key Principles (หลักการสำคัญ)

1. **ขนาดต้องตรงกัน**: MOV source และ destination ต้องมีขนาดเท่ากัน
2. **Memory-to-Memory ไม่ได้**: ต้องผ่าน register เสมอ (ยกเว้น REP MOVS)
3. **ระบุ Size Specifier**: เมื่อ operand เป็น memory กับ immediate
4. **MOVZX vs MOVSX**: unsigned ใช้ MOVZX, signed ใช้ MOVSX
5. **LEA เป็น Arithmetic**: LEA ไม่ได้อ่าน memory จริง ใช้คำนวณ address ได้
6. **CMOVcc ดีกว่า Branch**: ช่วยหลีกเลี่ยง branch misprediction
7. **XCHG กับ Memory = Atomic**: ระวัง performance impact
8. **BSWAP เฉพาะ 32/64-bit**: 16-bit ใช้ XCHG al, ah แทน
9. **PUSHAD/POPAD เฉพาะ 32-bit**: 64-bit ต้อง push/pop ทีละ register
10. **LAHF/SAHF**: เร็วกว่า PUSHF/POPF สำหรับบันทึก flags

### รูปแบบ Addressing ที่ MOV รองรับ

```
MOV dst, src  โดยที่ dst/src อาจเป็น:

┌─────────────────────────────────────────────────┐
│ Register:    EAX, RBX, CL, AX, R8D, ...        │
│ Memory:      [ebx], [var], [rbx+rcx*4+8], ...  │
│ Immediate:   42, 0xFF, -100, 'A', ...           │
│                                                  │
│ ข้อจำกัด:                                        │
│  - ไม่มี mem,mem                                 │
│  - ไม่มี imm,reg (immediate เป็น dest ไม่ได้)   │
│  - ขนาด src == dst เสมอ                          │
└─────────────────────────────────────────────────┘
```

### Performance Tips

```
เรียงลำดับความเร็ว (เร็ว → ช้า):

1. MOV reg, reg         — 0 latency (register rename)
2. MOV reg, imm         — 1 cycle
3. LEA reg, [expr]      — 1-3 cycles (address calc)
4. CMOVcc reg, reg      — 1 cycle (ดีกว่า branch)
5. MOVZX/MOVSX reg, reg — 1 cycle
6. MOV reg, [mem]       — 3-4 cycles (cache hit L1)
7. MOV [mem], reg       — 3-4 cycles (write buffer)
8. XCHG reg, [mem]      — ~20 cycles (atomic lock)
9. MOV reg, [mem]       — 100+ cycles (cache miss)

สรุป: ใช้ registers ให้มากที่สุด, ลด memory access
      ใช้ CMOVcc แทน branch เมื่อเป็นไปได้
      ใช้ LEA แทน MUL/ADD สำหรับ multiply by 2,3,4,5,9
```

### Quick Reference - Compilation Commands

```bash
# 64-bit ELF (Linux)
nasm -f elf64 program.asm -o program.o
gcc -no-pie -o program program.o        # ใช้ libc (printf, etc.)
ld  -o program program.o                # standalone (ไม่ใช้ libc)

# 32-bit ELF (Linux)
nasm -f elf32 program.asm -o program.o
gcc -m32 -no-pie -o program program.o

# ดู disassembly
objdump -d -M intel program

# ดู symbol table
nm program

# Debug ด้วย GDB
gdb ./program
(gdb) break main
(gdb) run
(gdb) info registers      # ดู registers
(gdb) x/8xb &variable    # ดู memory
(gdb) stepi               # step ทีละ instruction
```

---

*Part 013 จบแล้ว — ใน Part 014 เราจะเรียนเรื่อง Arithmetic Instructions (ADD, SUB, MUL, DIV, IMUL, IDIV, NEG) และการจัดการ Overflow/Carry*
