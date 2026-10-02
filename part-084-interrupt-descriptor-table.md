# Part 084: Interrupt Descriptor Table (IDT) and PIC

## บทนำ (Introduction)

ใน Protected Mode การจัดการ interrupts และ exceptions ทำผ่าน **Interrupt Descriptor Table (IDT)**
ซึ่งแตกต่างจาก Real Mode ที่ใช้ Interrupt Vector Table (IVT) ที่ address 0x0000:0x0000

IDT เป็น array ของ **descriptors** (gate descriptors) ที่บอก CPU ว่าเมื่อเกิด interrupt/exception
จะต้องกระโดดไป handler ที่ไหน และ privilege level เท่าไร

```
Real Mode IVT (1KB at 0x00000):     Protected Mode IDT (anywhere in memory):
┌──────────────────────────┐        ┌──────────────────────────────────────┐
│ Vector 0: CS:IP (4 bytes)│        │ Descriptor 0: Gate (8 bytes)         │
│ Vector 1: CS:IP (4 bytes)│        │ Descriptor 1: Gate (8 bytes)         │
│ Vector 2: CS:IP (4 bytes)│        │ ...                                  │
│ ...                      │        │ Descriptor 255: Gate (8 bytes)       │
│ Vector 255: CS:IP        │        └──────────────────────────────────────┘
└──────────────────────────┘                ↑ IDTR register ชี้มาที่นี่
```

---

## 1. ประเภทของ Exception (Exception Types)

CPU แบ่ง exceptions ออกเป็น 3 ประเภทหลัก ตามพฤติกรรมการกู้คืน:

### 1.1 Fault

**Fault** คือ exception ที่สามารถแก้ไขได้ (correctable)

- **CS:EIP ที่บันทึกไว้** ชี้ไปยัง **instruction ที่ก่อให้เกิด fault** (instruction ยังไม่ execute)
- หลังจาก handler แก้ปัญหาเรียบร้อย สามารถ IRET กลับไปลอง execute instruction นั้นใหม่ได้
- ตัวอย่าง: Page Fault (#PF) — handler load หน้า memory มาจาก disk แล้ว IRET กลับไป execute ใหม่

```
Instruction ที่ fault           Handler แก้ไข           Execute ใหม่
     │                               │                       │
     ▼                               ▼                       ▼
[MOV EAX, [addr]] ──fault──► [Page Fault Handler] ──IRET──► [MOV EAX, [addr]]
                                   (load page)                  (สำเร็จ)
```

### 1.2 Trap

**Trap** คือ exception ที่เกิดหลังจาก instruction execute เสร็จแล้ว

- **CS:EIP ที่บันทึกไว้** ชี้ไปยัง **instruction ถัดไป** (instruction ที่ trigger trap ทำงานเสร็จแล้ว)
- ใช้สำหรับ debugging (single-step, breakpoint)
- ตัวอย่าง: Breakpoint (#BP, INT 3), Overflow (#OF), Debug (#DB บางกรณี)

```
INT3 instruction               Handler จัดการ          Continue จาก instruction ถัดไป
     │                               │                       │
     ▼                               ▼                       ▼
[INT 3] ──trap──► [Breakpoint Handler] ──IRET──► [instruction หลัง INT 3]
```

### 1.3 Abort

**Abort** คือ exception ที่รุนแรงที่สุด ไม่สามารถกู้คืนได้

- ไม่สามารถ resume การทำงานได้
- ตัวอย่าง: Double Fault (#DF), Machine Check (#MC)
- มักต้องรีสตาร์ทระบบ

```
สรุปเปรียบเทียบ:
┌─────────┬──────────────────────────────────┬─────────────────────────────┐
│  Type   │ EIP saved ชี้ไปที่               │ สามารถ resume ได้หรือไม่    │
├─────────┼──────────────────────────────────┼─────────────────────────────┤
│ Fault   │ Instruction ที่ก่อให้เกิด fault  │ ได้ (re-execute instruction) │
│ Trap    │ Instruction ถัดไป               │ ได้ (continue)              │
│ Abort   │ ไม่แน่นอน                       │ ไม่ได้                       │
└─────────┴──────────────────────────────────┴─────────────────────────────┘
```

---

## 2. Exception Numbers 0-31 (CPU Exceptions)

Intel กำหนด interrupt vectors 0-31 ไว้สำหรับ CPU exceptions โดยเฉพาะ:

```
Vector  Mnemonic  ชื่อ                           Type    Error Code  ตัวอย่างสาเหตุ
───────────────────────────────────────────────────────────────────────────────────
  0     #DE       Divide Error                   Fault   No          DIV/IDIV ด้วย 0
  1     #DB       Debug                          F/T     No          DR breakpoint, TF set
  2     ---       NMI Interrupt                  ---     No          Hardware NMI
  3     #BP       Breakpoint                     Trap    No          INT 3
  4     #OF       Overflow                       Trap    No          INTO เมื่อ OF=1
  5     #BR       BOUND Range Exceeded           Fault   No          BOUND instruction
  6     #UD       Invalid Opcode                 Fault   No          opcode ไม่รู้จัก
  7     #NM       Device Not Available           Fault   No          FPU/MMX ยังไม่ init
  8     #DF       Double Fault                   Abort   Yes (0)     exception ซ้อน exception
  9     ---       Coprocessor Segment Overrun    Fault   No          (เก่า, ไม่ใช้แล้ว)
 10     #TS       Invalid TSS                    Fault   Yes         TSS descriptor ผิด
 11     #NP       Segment Not Present            Fault   Yes         segment ไม่ present
 12     #SS       Stack-Segment Fault            Fault   Yes         stack overflow/underflow
 13     #GP       General Protection             Fault   Yes         privilege violation
 14     #PF       Page Fault                     Fault   Yes         page ไม่ present/ไม่มีสิทธิ์
 15     ---       (สงวนไว้)                      ---     ---         ---
 16     #MF       x87 FPU Error                  Fault   No          FPU math error
 17     #AC       Alignment Check                Fault   Yes (0)     unaligned access (CR0.AM=1)
 18     #MC       Machine Check                  Abort   No          hardware error
 19     #XM/#XF  SIMD Floating-Point            Fault   No          SSE math error
 20     #VE       Virtualization                 Fault   No          EPT violation
 21     #CP       Control Protection             Fault   Yes         CET violation
 22-31  ---       (สงวนไว้)                      ---     ---         ---
```

### Error Code Format สำหรับ #PF (Page Fault):

```
Bit  ชื่อ  ความหมาย
─────────────────────────────────────────────
 0   P     0 = page ไม่ present, 1 = protection violation
 1   W     0 = read, 1 = write
 2   U     0 = supervisor, 1 = user mode
 3   R     0 = ปกติ, 1 = reserved bit set ใน page entry
 4   I     0 = data, 1 = instruction fetch
 5   PK    0 = ปกติ, 1 = protection key violation
 6   SS    0 = ปกติ, 1 = shadow stack access
```

---

## 3. Gate Types: Interrupt Gate vs Trap Gate

IDT มี gate หลายประเภท แต่ที่สำคัญที่สุดคือ:

### 3.1 Interrupt Gate (Type = 0xE หรือ 0b1110 ใน 32-bit)

- เมื่อ CPU เรียก handler ผ่าน interrupt gate **CPU จะ clear IF flag** (disable maskable interrupts)
- ทำให้ handler ทำงานโดยไม่ถูก interrupt อื่นขัดจังหวะ (atomic)
- เหมาะสำหรับ hardware interrupt handlers (IRQ handlers)
- ใช้ IRET เพื่อกลับ (IRET จะ restore EFLAGS รวมถึง IF กลับมา)

```
ก่อน: EFLAGS.IF = 1 (interrupts enabled)
CPU เรียก handler ผ่าน Interrupt Gate
ระหว่าง handler: EFLAGS.IF = 0 (interrupts disabled)
IRET: EFLAGS.IF = 1 กลับมา (restore จาก stack)
```

### 3.2 Trap Gate (Type = 0xF หรือ 0b1111 ใน 32-bit)

- เมื่อ CPU เรียก handler ผ่าน trap gate **IF flag ไม่เปลี่ยน**
- handler ทำงานพร้อม interrupts enabled (ถ้า IF เป็น 1 อยู่ก่อนแล้ว)
- เหมาะสำหรับ software exceptions และ system calls
- ใช้สำหรับ #BP, #OF, debug exceptions

```
สรุปความแตกต่าง:
┌──────────────────┬─────────────────────────────┬──────────────────────────┐
│    Gate Type     │ Effect on IF flag            │ เหมาะใช้กับ              │
├──────────────────┼─────────────────────────────┼──────────────────────────┤
│ Interrupt Gate   │ Clear IF (disable interrupts)│ Hardware IRQ handlers    │
│ Trap Gate        │ ไม่เปลี่ยน IF               │ Exceptions, System calls │
└──────────────────┴─────────────────────────────┴──────────────────────────┘
```

### 3.3 Task Gate (ไม่ค่อยใช้)

- สำหรับ hardware task switching (ใช้ TSS)
- มีประสิทธิภาพต่ำมาก, OS สมัยใหม่ไม่ใช้

---

## 4. IDT Descriptor Format (32-bit Protected Mode)

แต่ละ entry ใน IDT ขนาด **8 bytes** มี format ดังนี้:

```
Bits 63-48: Offset[31:16]  (upper 16 bits ของ handler address)
Bits 47-40: Type/Attributes
Bits 39-32: Reserved (ต้องเป็น 0)
Bits 31-16: Segment Selector  (เลือก code segment ของ handler)
Bits 15-0:  Offset[15:0]  (lower 16 bits ของ handler address)
```

### วิธีดู bit layout แบบละเอียด:

```
Byte 7 (MSB)  Byte 6          Byte 5          Byte 4
┌─────────────────────────────────────────────────────────┐
│        Offset[31:16]         │P│DPL│0│Type│  Reserved   │
└─────────────────────────────────────────────────────────┘

Byte 3          Byte 2          Byte 1          Byte 0 (LSB)
┌─────────────────────────────────────────────────────────┐
│         Segment Selector[15:0]         │  Offset[15:0]  │
└─────────────────────────────────────────────────────────┘
```

### Fields อธิบาย:

```
Field             Bits   ความหมาย
─────────────────────────────────────────────────────────────
Offset[15:0]       0-15  Lower 16 bits ของ handler address
Segment Selector  16-31  Code segment selector (เช่น 0x08 = kernel code)
Reserved          32-39  ต้องเป็น 0 ทั้งหมด
Type              40-43  Gate type (0xE=interrupt gate, 0xF=trap gate)
Bit 44            44     ต้องเป็น 0 (ไม่ใช่ storage segment)
DPL               45-46  Descriptor Privilege Level (0=kernel, 3=user)
P                 47     Present bit (1=valid descriptor)
Offset[31:16]     48-63  Upper 16 bits ของ handler address
```

### Type field values (32-bit):

```
Type Value  ความหมาย
0x5         Task Gate (32-bit)
0x6         Interrupt Gate (16-bit) — ไม่ค่อยใช้
0x7         Trap Gate (16-bit)     — ไม่ค่อยใช้
0xE         Interrupt Gate (32-bit) ← ใช้บ่อยที่สุด
0xF         Trap Gate (32-bit)     ← ใช้บ่อยที่สุด
```

### ตัวอย่าง NASM structure สำหรับ IDT entry:

```nasm
; IDT entry structure
struc idt_entry
    .offset_low   resw 1    ; bits 0-15 ของ handler address
    .selector     resw 1    ; code segment selector
    .reserved     resb 1    ; ต้องเป็น 0
    .type_attr    resb 1    ; P|DPL|0|Type
    .offset_high  resw 1    ; bits 16-31 ของ handler address
endstruc

; ค่า type_attr สำหรับ 32-bit interrupt gate, DPL=0, Present=1:
; P=1, DPL=00, 0, Type=1110
; = 1_00_0_1110 = 0x8E

; ค่า type_attr สำหรับ 32-bit trap gate, DPL=0, Present=1:
; P=1, DPL=00, 0, Type=1111
; = 1_00_0_1111 = 0x8F

; ค่า type_attr สำหรับ 32-bit trap gate, DPL=3 (user callable):
; P=1, DPL=11, 0, Type=1111
; = 1_11_0_1111 = 0xEF
```

---

## 5. IDTR Register และ LIDT Instruction

CPU ใช้ register พิเศษชื่อ **IDTR (IDT Register)** เพื่อเก็บ address และขนาดของ IDT

### IDTR Format (48 bits):

```
Bits 47-16: Base Address (32-bit linear address ของ IDT)
Bits 15-0:  Limit (ขนาดของ IDT - 1, หน่วย bytes)
```

```
   IDTR (6 bytes):
   ┌──────────────────────────────────────────┐
   │  Base[31:0] (4 bytes)  │  Limit (2 bytes) │
   └──────────────────────────────────────────┘
```

### การโหลด IDT ด้วย LIDT:

```nasm
; โครงสร้างที่ LIDT ต้องการ (เรียกว่า IDT descriptor หรือ pseudo-descriptor)
idtr:
    dw idt_end - idt - 1    ; limit = ขนาด IDT - 1
    dd idt                  ; base address ของ IDT

; โหลด IDT
lidt [idtr]
```

### ตัวอย่างสมบูรณ์:

```nasm
; กำหนดขนาด IDT (256 entries × 8 bytes = 2048 bytes)
IDT_ENTRIES equ 256

section .data
align 8
idt:
    times IDT_ENTRIES * 8 db 0  ; IDT เปล่าทั้งหมด

idtr:
    dw IDT_ENTRIES * 8 - 1      ; limit
    dd idt                      ; base

section .text
load_idt:
    lidt [idtr]
    ret
```

---

## 6. การ Setup IDT Entry

### Macro สำหรับสร้าง IDT entry:

```nasm
; Macro: ตั้งค่า IDT entry
; Parameters: vector, handler_addr, selector, type_attr
%macro SET_IDT_ENTRY 4
    mov eax, %3                     ; selector
    mov [idt + %1*8 + 2], ax        ; เขียน selector

    mov eax, %2                     ; handler address
    mov [idt + %1*8], ax            ; เขียน offset low
    shr eax, 16
    mov [idt + %1*8 + 6], ax        ; เขียน offset high

    mov byte [idt + %1*8 + 4], 0    ; reserved = 0
    mov byte [idt + %1*8 + 5], %4   ; type_attr
%endmacro

; ตัวอย่างการใช้:
; SET_IDT_ENTRY 0, divide_error_handler, 0x08, 0x8E
; SET_IDT_ENTRY 3, breakpoint_handler, 0x08, 0x8F
```

### ฟังก์ชัน C-style สำหรับตั้งค่า IDT entry (ใน assembly):

```nasm
; void set_idt_entry(uint8_t vector, uint32_t handler, uint16_t selector, uint8_t type_attr)
; Parameters: [esp+4]=vector, [esp+8]=handler, [esp+12]=selector, [esp+16]=type_attr
set_idt_entry:
    push ebp
    mov ebp, esp
    push ebx
    push edi

    movzx ebx, byte [ebp+8]         ; vector number
    lea edi, [idt + ebx*8]          ; pointer ไปยัง IDT entry

    mov eax, [ebp+12]               ; handler address
    mov [edi], ax                   ; offset low word
    shr eax, 16
    mov [edi+6], ax                 ; offset high word

    mov ax, [ebp+16]                ; selector
    mov [edi+2], ax

    mov byte [edi+4], 0             ; reserved
    mov al, [ebp+20]                ; type_attr
    mov [edi+5], al

    pop edi
    pop ebx
    pop ebp
    ret
```

---

## 7. Interrupt Stack Frame

เมื่อ CPU รับ interrupt/exception มันจะ **push ข้อมูลลง stack** โดยอัตโนมัติ
ก่อนที่จะ jump ไปยัง handler

### Stack Frame สำหรับ Exception ที่ไม่มี Error Code:

```
Stack เมื่อเข้า handler (ESP ชี้ไปที่ EIP):

Higher address
┌──────────┐
│    SS    │ ← pushed เฉพาะเมื่อเปลี่ยน privilege level (ring 0 → ring 3)
├──────────┤
│   ESP    │ ← pushed เฉพาะเมื่อเปลี่ยน privilege level
├──────────┤
│  EFLAGS  │ ← CPU push (สถานะ flags ก่อน interrupt)
├──────────┤
│    CS    │ ← CPU push (code segment ก่อน interrupt)
├──────────┤
│   EIP    │ ← CPU push (return address) ← ESP ชี้ที่นี่
└──────────┘
Lower address
```

### Stack Frame สำหรับ Exception ที่มี Error Code:

```
Higher address
┌──────────┐
│    SS    │ ← (เฉพาะเมื่อเปลี่ยน privilege level)
├──────────┤
│   ESP    │ ← (เฉพาะเมื่อเปลี่ยน privilege level)
├──────────┤
│  EFLAGS  │
├──────────┤
│    CS    │
├──────────┤
│   EIP    │
├──────────┤
│  ERROR   │ ← CPU push error code ← ESP ชี้ที่นี่
│   CODE   │
└──────────┘
Lower address
```

### IRET Instruction:

```nasm
; IRET pop ออกจาก stack ในลำดับนี้:
; 1. EIP
; 2. CS
; 3. EFLAGS
; (ถ้าเปลี่ยน privilege level):
; 4. ESP
; 5. SS

; สำคัญ: ถ้ามี error code ต้องเอาออกก่อน IRET
; เช่น:
my_handler_with_error:
    ; ... จัดการ ...
    add esp, 4      ; เอา error code ออก
    iret            ; pop EIP, CS, EFLAGS
```

---

## 8. Exception Handlers

### 8.1 #DE (Vector 0) — Divide Error

เกิดเมื่อ: DIV หรือ IDIV ด้วย 0, หรือผลลัพธ์ใหญ่เกิน register

```nasm
; Divide Error Handler (#DE, Vector 0)
; ไม่มี error code
; เป็น Fault: EIP ชี้ไปยัง instruction ที่ fault

divide_error_handler:
    ; บันทึก registers ที่จะใช้
    pushad

    ; แสดง error message
    mov esi, msg_divide_error
    call print_string

    ; ตัวเลือก 1: หยุดระบบ (kernel panic)
    cli
    hlt

    ; ตัวเลือก 2: ถ้าเป็น userspace process — kill process
    ; (ในระบบจริง)

    popad
    iret    ; ถ้าไม่ kill จะ loop วนซ้ำ (fault กลับมา execute DIV ใหม่)

section .data
msg_divide_error db "EXCEPTION: Divide by Zero (#DE)", 0x0A, 0
```

### 8.2 #BP (Vector 3) — Breakpoint

เกิดเมื่อ: execute INT 3 (opcode 0xCC)

```nasm
; Breakpoint Handler (#BP, Vector 3)
; ไม่มี error code
; เป็น Trap: EIP ชี้ไปยัง instruction ถัดจาก INT 3

breakpoint_handler:
    pushad

    ; แสดงสถานะ registers สำหรับ debugging
    ; (EIP, EFLAGS อยู่ใน stack frame)
    mov esi, msg_breakpoint
    call print_string

    ; แสดง EIP ที่ breakpoint เกิดขึ้น
    ; [esp + 36] = EIP (หลัง pushad ซึ่ง push 8 registers × 4 = 32 bytes)
    mov eax, [esp + 36]
    mov esi, msg_eip
    call print_string
    call print_hex

    popad
    iret    ; กลับไป instruction ถัดจาก INT 3

section .data
msg_breakpoint db "BREAKPOINT (#BP) hit!", 0x0A, 0
msg_eip db "EIP = 0x", 0
```

### 8.3 #UD (Vector 6) — Invalid Opcode

เกิดเมื่อ: CPU พบ opcode ที่ไม่รู้จัก หรือ instruction ที่ invalid ใน context นั้น

```nasm
; Invalid Opcode Handler (#UD, Vector 6)
; ไม่มี error code
; เป็น Fault: EIP ชี้ไปยัง invalid instruction

invalid_opcode_handler:
    pushad
    push ds
    push es
    push fs
    push gs

    mov esi, msg_invalid_opcode
    call print_string

    ; แสดง EIP ของ invalid instruction
    ; stack layout: gs, fs, es, ds, pushad(32), eip, cs, eflags
    ; [esp + 4*4 + 32] = EIP
    mov eax, [esp + 48]
    call print_hex

    pop gs
    pop fs
    pop es
    pop ds
    popad

    ; ตัวเลือก: skip ไปยัง instruction ถัดไป
    ; (ต้อง decode instruction ที่ invalid เพื่อรู้ขนาด)
    ; ง่ายกว่าคือ halt
    cli
    hlt

section .data
msg_invalid_opcode db "EXCEPTION: Invalid Opcode (#UD) at EIP=", 0
```

### 8.4 #GP (Vector 13) — General Protection Fault

เกิดเมื่อ: privilege violation, segment ผิด, instruction ที่ไม่อนุญาต

Error code format สำหรับ #GP:
```
Bits 15-3: Selector Index (ถ้า fault เกี่ยวข้องกับ descriptor)
Bit 2:     TI = 0 (GDT), 1 (LDT)
Bit 1:     IDT = 1 ถ้าเป็น IDT descriptor
Bit 0:     External = 1 ถ้าเป็น external interrupt
(ถ้า error code = 0 หมายความว่า fault ไม่ได้เกิดจาก specific descriptor)
```

```nasm
; General Protection Fault Handler (#GP, Vector 13)
; มี error code
; เป็น Fault

gp_fault_handler:
    ; Error code อยู่บน stack แล้ว (CPU push ไว้)
    ; Stack: [esp]=error_code, [esp+4]=eip, [esp+8]=cs, [esp+12]=eflags
    push ebp
    mov ebp, esp

    push eax
    push esi

    mov esi, msg_gp_fault
    call print_string

    ; แสดง error code
    mov eax, [ebp+4]        ; error code
    mov esi, msg_error_code
    call print_string
    call print_hex

    ; แสดง EIP
    mov eax, [ebp+8]        ; EIP
    mov esi, msg_at_eip
    call print_string
    call print_hex

    pop esi
    pop eax
    pop ebp

    ; ลบ error code ออกแล้ว halt (หรือ kill process ใน OS จริง)
    add esp, 4              ; ข้าม error code
    cli
    hlt

section .data
msg_gp_fault    db "EXCEPTION: General Protection Fault (#GP)", 0x0A, 0
msg_error_code  db "Error code: 0x", 0
msg_at_eip      db ", EIP=0x", 0
```

### 8.5 #PF (Vector 14) — Page Fault

เกิดเมื่อ: access memory page ที่ไม่ present หรือไม่มีสิทธิ์

CR2 register เก็บ **linear address** ที่ก่อให้เกิด page fault

```nasm
; Page Fault Handler (#PF, Vector 14)
; มี error code
; เป็น Fault: EIP ชี้ไปยัง instruction ที่ fault

page_fault_handler:
    ; Stack: [esp]=error_code, [esp+4]=eip, [esp+8]=cs, [esp+12]=eflags
    push eax
    push ebx
    push esi

    ; อ่าน fault address จาก CR2
    mov eax, cr2
    mov ebx, eax            ; เก็บ fault address

    ; อ่าน error code
    mov eax, [esp+12]       ; error code (หลัง push eax,ebx,esi)

    mov esi, msg_page_fault
    call print_string

    ; แสดง fault address
    mov eax, ebx
    mov esi, msg_fault_addr
    call print_string
    call print_hex

    ; วิเคราะห์ error code
    mov eax, [esp+12]
    test eax, 1             ; Bit 0: P flag
    jnz .protection_violation
    mov esi, msg_not_present
    jmp .show_type
.protection_violation:
    mov esi, msg_prot_violation
.show_type:
    call print_string

    ; ตรวจสอบ read/write
    test byte [esp+12], 2   ; Bit 1: W flag
    jz .was_read
    mov esi, msg_write
    jmp .show_rw
.was_read:
    mov esi, msg_read
.show_rw:
    call print_string

    ; ตรวจสอบ user/supervisor
    test byte [esp+12], 4   ; Bit 2: U flag
    jz .was_supervisor
    mov esi, msg_user
    jmp .show_mode
.was_supervisor:
    mov esi, msg_supervisor
.show_mode:
    call print_string

    pop esi
    pop ebx
    pop eax

    add esp, 4              ; ข้าม error code
    cli
    hlt

section .data
msg_page_fault      db "EXCEPTION: Page Fault (#PF)", 0x0A, 0
msg_fault_addr      db "Fault address (CR2) = 0x", 0
msg_not_present     db "Reason: Page not present", 0x0A, 0
msg_prot_violation  db "Reason: Protection violation", 0x0A, 0
msg_write           db "Access type: Write", 0x0A, 0
msg_read            db "Access type: Read", 0x0A, 0
msg_user            db "Mode: User", 0x0A, 0
msg_supervisor      db "Mode: Supervisor", 0x0A, 0
```

---

## 9. PIC (8259A Programmable Interrupt Controller)

**PIC (Programmable Interrupt Controller)** คือ chip ที่จัดการ hardware interrupts
ระบบ x86 PC ดั้งเดิมมี **PIC สองตัว** (master + slave) ต่อกันแบบ cascade:

```
CPU ←──── INT ────── Master PIC (8259A) ──── IRQ 0-7
                          │
                          │ (IRQ 2 = cascade)
                          ▼
                     Slave PIC (8259A) ──── IRQ 8-15
```

### IRQ Assignments มาตรฐาน:

```
IRQ   แหล่งที่มา (Hardware)
──────────────────────────────────────────────
IRQ0  PIT (Programmable Interval Timer) — system timer
IRQ1  Keyboard controller (PS/2)
IRQ2  Cascade (เชื่อมต่อกับ slave PIC) — ไม่ใช้โดยตรง
IRQ3  COM2 / COM4 (serial port)
IRQ4  COM1 / COM3 (serial port)
IRQ5  LPT2 / Sound card
IRQ6  Floppy disk controller
IRQ7  LPT1 (parallel port)
IRQ8  Real Time Clock (RTC)
IRQ9  ACPI / PCI (remapped จาก IRQ2 เดิม)
IRQ10 PCI / Network card
IRQ11 PCI / USB / SCSI
IRQ12 PS/2 Mouse
IRQ13 FPU / Math coprocessor
IRQ14 Primary ATA (IDE) controller
IRQ15 Secondary ATA (IDE) controller
```

### Port Addresses ของ PIC:

```
PIC            Command Port   Data Port
────────────────────────────────────────
Master PIC     0x20           0x21
Slave PIC      0xA0           0xA1
```

---

## 10. PIC Initialization: ICW Sequence

การ initialize PIC ต้องส่ง **Initialization Command Words (ICW)** ตามลำดับ:

### ขั้นตอน ICW1 → ICW2 → ICW3 → ICW4:

```
                Master PIC (port 0x20/0x21)    Slave PIC (port 0xA0/0xA1)
                ─────────────────────────────  ─────────────────────────────
ICW1 (start)  → port 0x20                   → port 0xA0
ICW2 (vector) → port 0x21                   → port 0xA1
ICW3 (cascade)→ port 0x21                   → port 0xA1
ICW4 (mode)   → port 0x21                   → port 0xA1
```

### ICW1 Format (ส่งไปยัง Command Port):

```
Bit 7-5: 0 (ignored ใน PC/AT)
Bit 4:   1 (ต้องเป็น 1 เสมอ — เป็นสัญญาณว่าเป็น ICW1)
Bit 3:   LTIM = 0 (edge triggered), 1 (level triggered)
Bit 2:   ADI  = 0 (call address interval = 8), 1 (= 4) — ไม่ใช้ใน 80x86
Bit 1:   SNGL = 0 (cascade mode — มี slave), 1 (single mode)
Bit 0:   IC4  = 1 (ต้องส่ง ICW4), 0 (ไม่ต้องส่ง ICW4)

ค่าทั่วไป: 0x11 = 0001_0001
  - Bit 4 = 1 (ICW1)
  - Bit 1 = 0 (cascade)
  - Bit 0 = 1 (need ICW4)
```

### ICW2 Format (ส่งไปยัง Data Port):

```
กำหนด base interrupt vector:
Bits 7-3: Vector offset (upper 5 bits ของ interrupt number ที่ต้องการ)
Bits 2-0: 0 (CPU กำหนดเอง ตาม IRQ number)

ตัวอย่าง:
  Master: ICW2 = 0x20 → IRQ0 = vector 0x20, IRQ1 = 0x21, ..., IRQ7 = 0x27
  Slave:  ICW2 = 0x28 → IRQ8 = vector 0x28, IRQ9 = 0x29, ..., IRQ15 = 0x2F
```

### ICW3 Format:

```
Master PIC (ส่งไปยัง Data Port 0x21):
  Bit pattern บอกว่า IRQ ไหนที่เชื่อมกับ slave
  ปกติ: Bit 2 = 1 → IRQ2 เชื่อมกับ slave
  ค่า: 0x04 = 0000_0100

Slave PIC (ส่งไปยัง Data Port 0xA1):
  Bits 2-0 = cascade identity (IRQ line ของ master ที่ slave เชื่อมอยู่)
  ค่า: 0x02 = 0000_0010 → slave เชื่อมกับ IRQ2 ของ master
```

### ICW4 Format (ส่งไปยัง Data Port):

```
Bit 7-5: 0
Bit 4:   SFNM = 0 (not special fully nested)
Bit 3-2: BUF  = 00 (not buffered mode)
Bit 1:   AEOI = 0 (manual EOI), 1 (auto EOI)
Bit 0:   uPM  = 1 (8086/88 mode), 0 (MCS-80/85 mode)

ค่าทั่วไป: 0x01 = 0000_0001
  - 8086 mode
  - Manual EOI
```

### โค้ด PIC Initialization สมบูรณ์:

```nasm
; =====================================================
; pic_init: Initialize Master and Slave PIC
; Remaps IRQ0-7 to vectors 0x20-0x27
;         IRQ8-15 to vectors 0x28-0x2F
; =====================================================

; PIC Port Definitions
PIC1_CMD    equ 0x20    ; Master PIC command port
PIC1_DATA   equ 0x21    ; Master PIC data port
PIC2_CMD    equ 0xA0    ; Slave PIC command port
PIC2_DATA   equ 0xA1    ; Slave PIC data port

; ICW1
ICW1_INIT   equ 0x10    ; Initialization bit
ICW1_ICW4   equ 0x01    ; ICW4 required
ICW1_VAL    equ ICW1_INIT | ICW1_ICW4  ; = 0x11

; ICW4
ICW4_8086   equ 0x01    ; 8086/88 mode

pic_init:
    push eax

    ; ── ICW1: เริ่ม initialization sequence ──────────────
    mov al, ICW1_VAL        ; 0x11
    out PIC1_CMD, al        ; ส่งไปยัง Master PIC command
    out PIC2_CMD, al        ; ส่งไปยัง Slave PIC command

    ; (รอ I/O — บาง system ต้องการ delay)
    ; ใช้ port 0x80 สำหรับ I/O delay (write ไปยัง unused port)
    xor al, al
    out 0x80, al            ; delay

    ; ── ICW2: ตั้ง vector offset ─────────────────────────
    mov al, 0x20            ; Master: IRQ0 → vector 0x20
    out PIC1_DATA, al
    xor al, al
    out 0x80, al            ; delay

    mov al, 0x28            ; Slave: IRQ8 → vector 0x28
    out PIC2_DATA, al
    xor al, al
    out 0x80, al            ; delay

    ; ── ICW3: Cascade configuration ──────────────────────
    mov al, 0x04            ; Master: IRQ2 เชื่อมกับ Slave (bit 2)
    out PIC1_DATA, al
    xor al, al
    out 0x80, al

    mov al, 0x02            ; Slave: cascade identity = 2 (IRQ2 ของ master)
    out PIC2_DATA, al
    xor al, al
    out 0x80, al

    ; ── ICW4: 8086 mode ──────────────────────────────────
    mov al, ICW4_8086       ; 0x01
    out PIC1_DATA, al
    xor al, al
    out 0x80, al

    out PIC2_DATA, al       ; al ยัง = 0x01 อยู่
    ; (ไม่ต้องรอแล้ว)

    ; ── ตั้งค่า mask: ปิด interrupts ทั้งหมดก่อน ────────
    ; (เปิดเฉพาะ IRQ ที่ต้องการในภายหลัง)
    mov al, 0xFF            ; mask ทุก IRQ ของ master
    out PIC1_DATA, al
    mov al, 0xFF            ; mask ทุก IRQ ของ slave
    out PIC2_DATA, al

    pop eax
    ret
```

---

## 11. OCW1: Interrupt Mask Register

หลังจาก initialize PIC แล้ว สามารถควบคุมว่า IRQ ไหนจะถูกส่งไปยัง CPU โดยการเขียน **mask** ไปยัง data port

### Mask Register:

```
Bit = 1: IRQ นั้น masked (ถูกบล็อก, ไม่ส่งไปยัง CPU)
Bit = 0: IRQ นั้น unmasked (เปิดให้ส่งไปยัง CPU)

ตัวอย่าง:
Master mask = 0b11111101 = 0xFD → เปิดเฉพาะ IRQ1 (keyboard)
Master mask = 0b11111110 = 0xFE → เปิดเฉพาะ IRQ0 (timer)
Master mask = 0b11111100 = 0xFC → เปิด IRQ0 และ IRQ1
Master mask = 0b00000000 = 0x00 → เปิด IRQ0-7 ทั้งหมด
Master mask = 0b11111111 = 0xFF → ปิด IRQ0-7 ทั้งหมด
```

### ฟังก์ชัน Masking/Unmasking:

```nasm
; =====================================================
; pic_mask_irq: Mask (disable) a specific IRQ
; Input: AL = IRQ number (0-15)
; =====================================================
pic_mask_irq:
    push eax
    push ebx
    push ecx
    push edx

    movzx ebx, al       ; IRQ number

    cmp al, 8
    jge .slave_irq

    ; Master PIC
    in al, PIC1_DATA    ; อ่าน mask ปัจจุบัน
    mov cl, bl          ; IRQ number
    mov edx, 1
    shl edx, cl         ; สร้าง bitmask
    or al, dl           ; set bit (mask IRQ)
    out PIC1_DATA, al   ; เขียนกลับ
    jmp .done

.slave_irq:
    sub bl, 8           ; IRQ8-15 → bit 0-7 ใน slave
    in al, PIC2_DATA    ; อ่าน slave mask
    mov cl, bl
    mov edx, 1
    shl edx, cl
    or al, dl
    out PIC2_DATA, al

.done:
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; =====================================================
; pic_unmask_irq: Unmask (enable) a specific IRQ
; Input: AL = IRQ number (0-15)
; =====================================================
pic_unmask_irq:
    push eax
    push ebx
    push ecx
    push edx

    movzx ebx, al

    cmp al, 8
    jge .slave_irq

    ; Master PIC
    in al, PIC1_DATA
    mov cl, bl
    mov edx, 1
    shl edx, cl
    not edx
    and al, dl          ; clear bit (unmask IRQ)
    out PIC1_DATA, al
    jmp .done

.slave_irq:
    sub bl, 8
    in al, PIC2_DATA
    mov cl, bl
    mov edx, 1
    shl edx, cl
    not edx
    and al, dl
    out PIC2_DATA, al

    ; ต้อง unmask IRQ2 ของ master ด้วย (cascade line)
    in al, PIC1_DATA
    and al, 0b11111011  ; clear bit 2
    out PIC1_DATA, al

.done:
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret
```

### ตัวอย่างการใช้:

```nasm
; เปิด IRQ0 (timer) และ IRQ1 (keyboard)
mov al, 0
call pic_unmask_irq     ; unmask IRQ0
mov al, 1
call pic_unmask_irq     ; unmask IRQ1
```

---

## 12. EOI (End of Interrupt)

หลังจาก IRQ handler ทำงานเสร็จแล้ว **ต้องส่ง EOI** (End of Interrupt) ไปยัง PIC
เพื่อบอกว่าได้จัดการ interrupt นั้นเสร็จแล้ว และ PIC สามารถส่ง interrupt ใหม่ได้

### OCW2: EOI Command:

```
ส่ง 0x20 ไปยัง Command Port ของ PIC = Non-Specific EOI
```

```nasm
PIC_EOI equ 0x20        ; End of Interrupt command

; สำหรับ IRQ0-7 (master PIC เท่านั้น):
pic_send_eoi_master:
    mov al, PIC_EOI
    out PIC1_CMD, al    ; ส่ง EOI ไปยัง master
    ret

; สำหรับ IRQ8-15 (ต้องส่ง EOI ไปทั้ง slave และ master):
pic_send_eoi_slave:
    mov al, PIC_EOI
    out PIC2_CMD, al    ; ส่ง EOI ไปยัง slave ก่อน
    out PIC1_CMD, al    ; แล้วส่งไปยัง master (cascade)
    ret

; ฟังก์ชัน generic:
; Input: AL = IRQ number (0-15)
pic_send_eoi:
    cmp al, 8
    jge .slave
    ; Master only
    mov al, PIC_EOI
    out PIC1_CMD, al
    ret
.slave:
    mov al, PIC_EOI
    out PIC2_CMD, al    ; slave ก่อน
    out PIC1_CMD, al    ; แล้ว master
    ret
```

> **สำคัญ**: ถ้าลืมส่ง EOI จะทำให้ interrupt นั้น (และ interrupt ที่มี priority ต่ำกว่า) ถูก block ตลอดไป!

---

## 13. Complete Timer Handler (IRQ0)

```nasm
; =====================================================
; Timer IRQ Handler (IRQ0, Vector 0x20)
; PIT (Programmable Interval Timer) ส่ง IRQ0 ทุก ~18.2 Hz
; (หรือปรับได้ตามที่ตั้งค่า PIT)
; =====================================================

section .data
timer_ticks dd 0            ; นับจำนวน timer interrupts
msg_tick    db "Tick!", 0x0A, 0

section .text

; ลงทะเบียน timer handler ใน IDT
; SET_IDT_ENTRY 0x20, timer_handler, 0x08, 0x8E

timer_handler:
    pushad              ; บันทึก general purpose registers
    push ds
    push es
    push fs
    push gs

    ; โหลด kernel data segment
    mov ax, 0x10        ; kernel data segment selector (GDT entry 2)
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    ; เพิ่มตัวนับ
    inc dword [timer_ticks]

    ; ทำงานที่ต้องการ (เช่น scheduler, timeout checks)
    ; ...

    ; ตัวอย่าง: แสดงข้อความทุก 18 ticks (~1 วินาที)
    mov eax, [timer_ticks]
    xor edx, edx
    mov ecx, 18
    div ecx             ; edx = remainder
    test edx, edx
    jnz .done

    mov esi, msg_tick
    call print_string

.done:
    ; ส่ง EOI ไปยัง Master PIC (IRQ0 มาจาก master)
    mov al, PIC_EOI
    out PIC1_CMD, al

    pop gs
    pop fs
    pop es
    pop ds
    popad
    iret                ; คืน EFLAGS, CS, EIP
```

---

## 14. Complete Keyboard Handler (IRQ1)

```nasm
; =====================================================
; Keyboard IRQ Handler (IRQ1, Vector 0x21)
; PS/2 keyboard controller ส่ง IRQ1 เมื่อมีการกดปุ่ม
; =====================================================

; Keyboard Controller Ports
KB_DATA_PORT    equ 0x60    ; อ่าน scancode ที่นี่
KB_STATUS_PORT  equ 0x64    ; status register

section .data
; Scancode to ASCII table (Set 1, US QWERTY, unshifted)
; Index = scancode, value = ASCII character (0 = no ASCII)
scancode_table:
    db 0,   27, '1', '2', '3', '4', '5', '6'   ; 0x00-0x07
    db '7', '8', '9', '0', '-', '=', 8,   9    ; 0x08-0x0F
    db 'q', 'w', 'e', 'r', 't', 'y', 'u', 'i'  ; 0x10-0x17
    db 'o', 'p', '[', ']', 13,  0,  'a', 's'   ; 0x18-0x1F
    db 'd', 'f', 'g', 'h', 'j', 'k', 'l', ';'  ; 0x20-0x27
    db 39,  '`', 0,  '\', 'z', 'x', 'c', 'v'  ; 0x28-0x2F
    db 'b', 'n', 'm', ',', '.', '/', 0,   '*'  ; 0x30-0x37
    db 0,   ' ', 0,   0,   0,   0,   0,   0    ; 0x38-0x3F
    ; ... (ย่อ)

keyboard_buffer:    times 256 db 0  ; circular buffer
kb_write_pos:       db 0
kb_read_pos:        db 0

section .text

keyboard_handler:
    pushad
    push ds
    push es

    mov ax, 0x10
    mov ds, ax
    mov es, ax

    ; อ่าน scancode จาก keyboard data port
    in al, KB_DATA_PORT

    ; ตรวจสอบว่าเป็น key release (bit 7 = 1) หรือ key press (bit 7 = 0)
    test al, 0x80
    jnz .key_release

    ; Key Press
    movzx eax, al       ; scancode

    ; แปลง scancode เป็น ASCII
    cmp eax, 58         ; scancode 0x3A = สูงสุดของ table เรา
    jge .special_key

    movzx ebx, byte [scancode_table + eax]  ; อ่าน ASCII
    test bl, bl
    jz .special_key     ; ถ้า 0 = ไม่มี ASCII ปกติ

    ; เก็บลง keyboard buffer
    movzx ecx, byte [kb_write_pos]
    mov [keyboard_buffer + ecx], bl
    inc cl
    ; and cl, 0xFF      ; modulo 256 (circular buffer)
    mov [kb_write_pos], cl

    jmp .done

.key_release:
    ; ทำเครื่องหมายว่า key นั้น release แล้ว
    ; (สำหรับ modifier keys เช่น Shift, Ctrl)
    jmp .done

.special_key:
    ; จัดการ special keys (arrow keys, function keys, etc.)
    ; ...

.done:
    ; ส่ง EOI ไปยัง Master PIC (IRQ1 มาจาก master)
    mov al, PIC_EOI
    out PIC1_CMD, al

    pop es
    pop ds
    popad
    iret

; =====================================================
; kb_getchar: รับอักขระจาก keyboard buffer
; Output: AL = ASCII character (0 ถ้า buffer ว่าง)
; =====================================================
kb_getchar:
    mov al, [kb_read_pos]
    cmp al, [kb_write_pos]
    je .buffer_empty    ; read_pos == write_pos → buffer ว่าง

    movzx eax, byte [kb_read_pos]
    mov al, [keyboard_buffer + eax]
    push eax
    movzx eax, byte [kb_read_pos]
    inc al
    mov [kb_read_pos], al
    pop eax
    ret

.buffer_empty:
    xor al, al
    ret
```

---

## 15. iret vs ret — ความแตกต่าง

### ret (Return from procedure):

```nasm
; ret ทำแค่:
; 1. Pop EIP จาก stack
; 2. Jump ไปที่ EIP

; Stack ก่อน ret:
; [esp] = return address (EIP)
; [esp+4] = ... (parameters, local vars)

my_function:
    ; ...
    ret         ; pop EIP แล้ว jump
```

### iret (Interrupt Return):

```nasm
; iret ทำ:
; 1. Pop EIP
; 2. Pop CS
; 3. Pop EFLAGS
; (ถ้าเปลี่ยน privilege level):
; 4. Pop ESP
; 5. Pop SS

; Stack ก่อน iret (ที่ CPU สร้างไว้):
; [esp]    = EIP
; [esp+4]  = CS
; [esp+8]  = EFLAGS
; (ถ้า privilege change):
; [esp+12] = ESP (old)
; [esp+16] = SS  (old)

interrupt_handler:
    ; ...
    iret        ; restore EIP, CS, EFLAGS (และอาจ ESP, SS)
```

### ทำไมต้องใช้ iret ไม่ใช่ ret:

```
ถ้าใช้ ret แทน iret ใน interrupt handler:
1. ret จะ pop แค่ EIP → EIP ถูกต้อง
2. แต่ stack ยังมี CS และ EFLAGS เหลืออยู่ → stack corruption!
3. EFLAGS ไม่ได้ถูก restore → IF flag อาจยังเป็น 0 (interrupts disabled)
4. CS ไม่ถูก restore → อาจทำให้ segment ผิด

ดังนั้น: interrupt handler ต้องใช้ iret เสมอ
         ยกเว้น task gate (ซึ่งใช้ task switch แทน)
```

---

## 16. โครงสร้างสมบูรณ์: IDT Setup พร้อม Exception และ IRQ Handlers

### ไฟล์หลัก: `kernel.asm`

```nasm
; =====================================================
; kernel.asm — 32-bit Protected Mode Kernel with IDT
; Build: nasm -f elf32 kernel.asm -o kernel.o
;        ld -m elf_i386 -T link.ld kernel.o -o kernel.bin
; =====================================================

[BITS 32]

; ─── Constants ───────────────────────────────────────
KERNEL_CS       equ 0x08        ; GDT kernel code segment
KERNEL_DS       equ 0x10        ; GDT kernel data segment
IDT_SIZE        equ 256
PIC1_CMD        equ 0x20
PIC1_DATA       equ 0x21
PIC2_CMD        equ 0xA0
PIC2_DATA       equ 0xA1
PIC_EOI         equ 0x20

; IDT gate type/attributes
INT_GATE_32     equ 0x8E        ; P=1, DPL=0, 32-bit interrupt gate
TRAP_GATE_32    equ 0x8F        ; P=1, DPL=0, 32-bit trap gate

; ─── Sections ────────────────────────────────────────
section .data
align 8

; IDT (256 entries × 8 bytes)
idt:
    times IDT_SIZE * 8 db 0

; IDTR (6 bytes)
idtr:
    dw IDT_SIZE * 8 - 1     ; limit
    dd idt                  ; base

; ─── Text Section ────────────────────────────────────
section .text
global kernel_main

kernel_main:
    ; ตั้งค่า segment registers
    mov ax, KERNEL_DS
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    mov esp, stack_top

    ; 1. Setup IDT entries
    call setup_idt

    ; 2. Initialize PIC (remap IRQs)
    call pic_init

    ; 3. Load IDT
    lidt [idtr]

    ; 4. Unmask IRQ0 (timer) and IRQ1 (keyboard)
    mov al, 0
    call pic_unmask_irq
    mov al, 1
    call pic_unmask_irq

    ; 5. Enable interrupts
    sti

    ; Main loop
.loop:
    hlt         ; รอ interrupt
    jmp .loop

; ─── Setup IDT Entries ────────────────────────────────
setup_idt:
    ; CPU Exceptions (vectors 0-31) — Trap gates
    mov eax, divide_error_handler
    call .set_trap_gate_0

    ; ... (ตั้งค่า exceptions อื่นๆ)

    ; IRQ handlers (vectors 0x20-0x2F) — Interrupt gates
    mov eax, timer_handler
    call .set_int_gate_0x20

    mov eax, keyboard_handler
    call .set_int_gate_0x21

    ret

; Helper: ตั้งค่า IDT entry
; Input: AL=vector, EBX=handler addr, CX=selector, DL=type_attr
.set_idt_entry_helper:
    push eax
    push ebx

    movzx eax, al
    lea ebx, [idt + eax*8]     ; pointer ไปยัง IDT entry

    ; เขียน offset low word
    mov eax, [esp-4]            ; handler addr (ต้องดึงจาก param)
    ; (simplified — ดูฟังก์ชัน set_idt_entry ด้านบน)

    pop ebx
    pop eax
    ret
```

### Stub handlers พร้อม error code:

```nasm
; =====================================================
; Exception stubs — ใช้ macro เพื่อสร้าง handlers
; =====================================================

; Macro สำหรับ exception ที่ไม่มี error code
%macro EXCEPTION_NOERR 2    ; %1=vector number, %2=handler name
exception_%1:
    push dword 0            ; dummy error code
    push dword %1           ; vector number
    jmp %2                  ; jump ไปยัง common handler
%endmacro

; Macro สำหรับ exception ที่มี error code
%macro EXCEPTION_ERR 2      ; %1=vector number, %2=handler name
exception_%1:
    ; error code อยู่บน stack แล้ว (CPU push ไว้)
    push dword %1           ; vector number
    jmp %2                  ; jump ไปยัง common handler
%endmacro

; สร้าง stubs
EXCEPTION_NOERR  0, common_exception_handler    ; #DE
EXCEPTION_NOERR  1, common_exception_handler    ; #DB
EXCEPTION_NOERR  2, common_exception_handler    ; NMI
EXCEPTION_NOERR  3, common_exception_handler    ; #BP
EXCEPTION_NOERR  4, common_exception_handler    ; #OF
EXCEPTION_NOERR  5, common_exception_handler    ; #BR
EXCEPTION_NOERR  6, common_exception_handler    ; #UD
EXCEPTION_NOERR  7, common_exception_handler    ; #NM
EXCEPTION_ERR    8, common_exception_handler    ; #DF
EXCEPTION_NOERR  9, common_exception_handler    ; Coprocessor
EXCEPTION_ERR   10, common_exception_handler    ; #TS
EXCEPTION_ERR   11, common_exception_handler    ; #NP
EXCEPTION_ERR   12, common_exception_handler    ; #SS
EXCEPTION_ERR   13, common_exception_handler    ; #GP
EXCEPTION_ERR   14, common_exception_handler    ; #PF

; =====================================================
; common_exception_handler
; Stack on entry:
;   [esp+0]  = vector number (pushed by stub)
;   [esp+4]  = error code (CPU push หรือ dummy 0)
;   [esp+8]  = EIP
;   [esp+12] = CS
;   [esp+16] = EFLAGS
; =====================================================
common_exception_handler:
    pushad          ; บันทึก EAX,ECX,EDX,EBX,ESP,EBP,ESI,EDI

    ; Stack layout หลัง pushad:
    ; [esp+0..31] = pushad data (8 regs × 4 bytes)
    ; [esp+32]    = vector
    ; [esp+36]    = error code
    ; [esp+40]    = EIP
    ; [esp+44]    = CS
    ; [esp+48]    = EFLAGS

    mov eax, [esp+32]   ; vector number
    mov ebx, [esp+36]   ; error code
    mov ecx, [esp+40]   ; EIP
    mov edx, [esp+48]   ; EFLAGS

    ; แสดงข้อมูล (เรียก print_exception_info)
    push edx            ; EFLAGS
    push ecx            ; EIP
    push ebx            ; error code
    push eax            ; vector
    call print_exception_info
    add esp, 16

    popad

    add esp, 8          ; เอา vector number และ error code ออก
    iret

; =====================================================
; print_exception_info
; Input (stack): [esp+4]=vector, [esp+8]=error_code,
;                [esp+12]=eip, [esp+16]=eflags
; =====================================================
section .data
exception_names:
    dd msg_ex0,  msg_ex1,  msg_ex2,  msg_ex3
    dd msg_ex4,  msg_ex5,  msg_ex6,  msg_ex7
    dd msg_ex8,  msg_ex9,  msg_ex10, msg_ex11
    dd msg_ex12, msg_ex13, msg_ex14, msg_ex15

msg_ex0  db "#DE Divide Error", 0
msg_ex1  db "#DB Debug", 0
msg_ex2  db "NMI", 0
msg_ex3  db "#BP Breakpoint", 0
msg_ex4  db "#OF Overflow", 0
msg_ex5  db "#BR BOUND Range Exceeded", 0
msg_ex6  db "#UD Invalid Opcode", 0
msg_ex7  db "#NM Device Not Available", 0
msg_ex8  db "#DF Double Fault", 0
msg_ex9  db "Coprocessor Segment Overrun", 0
msg_ex10 db "#TS Invalid TSS", 0
msg_ex11 db "#NP Segment Not Present", 0
msg_ex12 db "#SS Stack Fault", 0
msg_ex13 db "#GP General Protection", 0
msg_ex14 db "#PF Page Fault", 0
msg_ex15 db "(Reserved)", 0

section .text
print_exception_info:
    push ebp
    mov ebp, esp
    push eax
    push esi

    ; หัวข้อ
    mov esi, msg_exception_header
    call print_string

    ; ชื่อ exception
    mov eax, [ebp+8]        ; vector
    cmp eax, 15
    jg .unknown
    mov esi, [exception_names + eax*4]
    jmp .print_name
.unknown:
    mov esi, msg_unknown_exception
.print_name:
    call print_string

    ; EIP
    mov eax, [ebp+16]       ; EIP
    mov esi, msg_at_eip
    call print_string
    call print_hex_newline

    ; Error code
    mov eax, [ebp+12]       ; error_code
    test eax, eax
    jz .no_error_code
    mov esi, msg_error_code
    call print_string
    call print_hex_newline

.no_error_code:
    pop esi
    pop eax
    pop ebp
    ret

section .data
msg_exception_header    db 0x0A, "=== EXCEPTION! ===", 0x0A, 0
msg_unknown_exception   db "Unknown Exception", 0
msg_at_eip              db " at EIP=0x", 0
msg_error_code          db "Error Code: 0x", 0
```

---

## 17. Linker Script และ Makefile

### Linker Script (`link.ld`):

```ld
/* link.ld */
ENTRY(kernel_main)

SECTIONS {
    . = 0x100000;       /* โหลด kernel ที่ 1MB */

    .text : {
        *(.text)
    }

    .rodata : {
        *(.rodata)
        *(.rodata*)
    }

    .data : {
        *(.data)
    }

    .bss : {
        *(.bss)
        *(COMMON)
    }
}
```

### Makefile:

```makefile
# Makefile สำหรับ IDT/PIC kernel

CC      = nasm
LD      = ld
QEMU    = qemu-system-i386

NASM_FLAGS = -f elf32 -g -F dwarf
LD_FLAGS   = -m elf_i386 -T link.ld

SRCS = boot.asm kernel.asm idt.asm pic.asm handlers.asm
OBJS = $(SRCS:.asm=.o)

.PHONY: all clean run debug

all: kernel.bin

%.o: %.asm
	$(CC) $(NASM_FLAGS) $< -o $@

kernel.bin: $(OBJS)
	$(LD) $(LD_FLAGS) $(OBJS) -o $@

# สร้าง floppy image สำหรับ QEMU
floppy.img: boot.bin kernel.bin
	dd if=/dev/zero of=floppy.img bs=1024 count=1440
	dd if=boot.bin of=floppy.img conv=notrunc
	dd if=kernel.bin of=floppy.img bs=512 seek=1 conv=notrunc

# รันด้วย QEMU
run: floppy.img
	$(QEMU) -fda floppy.img -nographic

# รันพร้อม debug (รอ GDB ที่ port 1234)
debug: floppy.img
	$(QEMU) -fda floppy.img -s -S -nographic &
	gdb -ex "target remote :1234" \
	    -ex "symbol-file kernel.bin" \
	    -ex "break kernel_main" \
	    -ex "continue"

clean:
	rm -f *.o kernel.bin floppy.img
```

### QEMU Test Commands:

```bash
# Build และรัน
make all
make run

# รันโดยตรงด้วย kernel binary
qemu-system-i386 -kernel kernel.bin -nographic

# รันพร้อม debug output
qemu-system-i386 -kernel kernel.bin \
    -nographic \
    -d int,cpu_reset \
    2>&1 | head -100

# Monitor interrupts
qemu-system-i386 -kernel kernel.bin \
    -nographic \
    -d int \
    2>qemu_interrupts.log &

# รัน และดู VGA output
qemu-system-i386 -kernel kernel.bin

# ทดสอบกับ GRUB multiboot
qemu-system-i386 -cdrom os.iso -nographic
```

---

## 18. Interrupt Stub Generator (IRQ Handlers)

```nasm
; =====================================================
; IRQ Stub Generator — สร้าง stubs สำหรับ IRQ 0-15
; =====================================================

; Macro: สร้าง IRQ stub
%macro IRQ_STUB 1           ; %1 = IRQ number
irq%1_stub:
    push dword 0            ; dummy error code
    push dword (0x20 + %1)  ; interrupt vector (0x20+IRQ)
    jmp common_irq_handler
%endmacro

; สร้าง stubs ทั้งหมด
IRQ_STUB 0      ; timer
IRQ_STUB 1      ; keyboard
IRQ_STUB 2      ; cascade (ไม่ค่อยใช้)
IRQ_STUB 3      ; COM2
IRQ_STUB 4      ; COM1
IRQ_STUB 5      ; LPT2/Sound
IRQ_STUB 6      ; Floppy
IRQ_STUB 7      ; LPT1
IRQ_STUB 8      ; RTC
IRQ_STUB 9      ; PCI
IRQ_STUB 10     ; PCI
IRQ_STUB 11     ; PCI
IRQ_STUB 12     ; PS/2 Mouse
IRQ_STUB 13     ; FPU
IRQ_STUB 14     ; Primary ATA
IRQ_STUB 15     ; Secondary ATA

; =====================================================
; common_irq_handler
; Stack on entry (from stub):
;   [esp+0]  = vector number
;   [esp+4]  = dummy error code (0)
;   [esp+8]  = EIP
;   [esp+12] = CS
;   [esp+16] = EFLAGS
; =====================================================
section .data
; ตาราง IRQ handler functions
irq_handlers:
    times 16 dd default_irq_handler  ; ค่าเริ่มต้น: default handler

section .text

common_irq_handler:
    pushad

    ; อ่าน IRQ number จาก vector number
    mov eax, [esp+36]       ; vector (หลัง pushad = 8×4=32 bytes)
    sub eax, 0x20           ; IRQ = vector - 0x20

    ; เรียก handler ที่ลงทะเบียนไว้
    cmp eax, 15
    ja .send_eoi            ; ถ้า > 15 ข้ามไป

    mov eax, [irq_handlers + eax*4]
    test eax, eax
    jz .send_eoi
    call eax                ; เรียก handler

.send_eoi:
    ; ส่ง EOI
    mov eax, [esp+36]       ; vector
    sub eax, 0x20           ; IRQ number
    cmp eax, 8
    jl .master_eoi

    ; Slave + Master EOI
    mov al, PIC_EOI
    out PIC2_CMD, al
.master_eoi:
    mov al, PIC_EOI
    out PIC1_CMD, al

    popad
    add esp, 8              ; เอา vector + dummy error code ออก
    iret

; =====================================================
; ลงทะเบียน IRQ handler
; Input: AL = IRQ number, EBX = handler function address
; =====================================================
register_irq_handler:
    movzx eax, al
    cmp eax, 15
    ja .done
    mov [irq_handlers + eax*4], ebx
.done:
    ret

; Default IRQ handler (ไม่ทำอะไร แค่ส่ง EOI)
default_irq_handler:
    ret
```

---

## 19. ตัวอย่างการใช้งานสมบูรณ์: Keyboard + Timer

```nasm
; =====================================================
; demo.asm — Demo program ที่ใช้ keyboard + timer IRQ
; =====================================================

section .data
timer_count     dd 0
key_pressed     db 0

msg_welcome     db "IDT/PIC Demo Running...", 0x0A
                db "Press any key (ESC to quit)", 0x0A, 0
msg_timer       db "Timer: ", 0
msg_key         db "Key pressed: ", 0
msg_quit        db "Quitting...", 0x0A, 0

section .text
global kernel_main

kernel_main:
    ; Setup (ดูด้านบน)
    call setup_idt
    call pic_init
    lidt [idtr]

    ; ลงทะเบียน handlers จริง
    lea ebx, [my_timer_handler]
    mov al, 0
    call register_irq_handler

    lea ebx, [my_keyboard_handler]
    mov al, 1
    call register_irq_handler

    ; Unmask IRQ0 และ IRQ1
    mov al, 0
    call pic_unmask_irq
    mov al, 1
    call pic_unmask_irq

    sti

    mov esi, msg_welcome
    call print_string

    ; Main loop — รอ keyboard input
.main_loop:
    hlt                     ; รอ interrupt

    ; ตรวจสอบว่ามี key กดหรือไม่
    mov al, [key_pressed]
    test al, al
    jz .main_loop

    ; แสดงข้อมูล key
    mov esi, msg_key
    call print_string
    mov al, [key_pressed]
    call print_char

    ; เช็ค ESC (scancode 0x01 → ASCII 27)
    cmp byte [key_pressed], 27
    je .quit

    mov byte [key_pressed], 0   ; reset
    jmp .main_loop

.quit:
    mov esi, msg_quit
    call print_string
    cli
    hlt

; ─── My Timer Handler ─────────────────────────────
my_timer_handler:
    inc dword [timer_count]

    ; แสดงตัวนับทุก 100 ticks
    mov eax, [timer_count]
    xor edx, edx
    mov ecx, 100
    div ecx
    test edx, edx
    jnz .done

    mov esi, msg_timer
    call print_string
    mov eax, [timer_count]
    call print_dec
    mov al, 0x0A
    call print_char

.done:
    ret

; ─── My Keyboard Handler ──────────────────────────
my_keyboard_handler:
    in al, 0x60             ; อ่าน scancode

    test al, 0x80           ; bit 7 = key release
    jnz .release

    ; Key press — แปลง scancode เป็น ASCII
    cmp al, 1               ; ESC key
    je .esc_key

    cmp al, 58              ; สูงสุด
    jge .done

    movzx eax, al
    mov al, [scancode_table + eax]
    test al, al
    jz .done

    mov [key_pressed], al   ; บันทึก key
    jmp .done

.esc_key:
    mov byte [key_pressed], 27  ; ESC = ASCII 27

.release:
.done:
    ret
```

---

## 20. Spurious Interrupts

ปัญหาที่พบบ่อยกับ PIC: **Spurious interrupts** (IRQ7 สำหรับ master, IRQ15 สำหรับ slave)

เกิดขึ้นเมื่อ interrupt signal สั้นมากและหายไปก่อนที่ PIC จะตอบสนอง

```nasm
; วิธีตรวจสอบว่าเป็น spurious interrupt หรือไม่
; อ่าน In-Service Register (ISR) ของ PIC

PIC_READ_ISR equ 0x0B

; ตรวจสอบ spurious IRQ7
check_spurious_irq7:
    ; ส่งคำสั่งให้ PIC ส่ง ISR กลับมา
    mov al, PIC_READ_ISR
    out PIC1_CMD, al
    in al, PIC1_CMD         ; อ่าน ISR

    test al, 0x80           ; bit 7 = IRQ7 in service?
    jnz .real_irq7          ; ถ้า set = เป็น IRQ จริง

    ; Spurious IRQ7 — ไม่ต้องส่ง EOI!
    ret

.real_irq7:
    ; จัดการ IRQ7 จริง
    ; ...
    ; ส่ง EOI
    mov al, PIC_EOI
    out PIC1_CMD, al
    ret

; ตรวจสอบ spurious IRQ15
check_spurious_irq15:
    mov al, PIC_READ_ISR
    out PIC2_CMD, al
    in al, PIC2_CMD         ; อ่าน slave ISR

    test al, 0x80           ; bit 7 = IRQ15 in service?
    jnz .real_irq15

    ; Spurious IRQ15 — ส่ง EOI ไปยัง master เท่านั้น (ไม่ใช่ slave)
    mov al, PIC_EOI
    out PIC1_CMD, al        ; master EOI เท่านั้น
    ret

.real_irq15:
    ; จัดการ IRQ15 จริง
    ; ...
    mov al, PIC_EOI
    out PIC2_CMD, al        ; slave EOI
    out PIC1_CMD, al        ; master EOI
    ret
```

---

## 21. การตั้งค่า IDT Entry ทั้งหมด (สมบูรณ์)

```nasm
; =====================================================
; setup_all_idt_entries: ตั้งค่า IDT entries ทั้งหมด
; =====================================================
setup_all_idt_entries:
    push eax
    push ebx

    ; ─── Exception Handlers (vectors 0-21) ───────────
    ; Trap gates สำหรับ exceptions (DPL=0)
    mov al, 0
    mov ebx, exception_0
    mov cx, KERNEL_CS
    mov dl, TRAP_GATE_32
    call set_idt_entry_all

    mov al, 1
    mov ebx, exception_1
    call set_idt_entry_all

    mov al, 2
    mov ebx, exception_2
    call set_idt_entry_all

    ; Breakpoint: DPL=3 ให้ userspace เรียกได้
    mov al, 3
    mov ebx, exception_3
    mov dl, 0xEF        ; P=1, DPL=3, trap gate
    call set_idt_entry_all
    mov dl, TRAP_GATE_32    ; reset กลับ

    mov al, 4
    mov ebx, exception_4
    mov dl, TRAP_GATE_32
    call set_idt_entry_all

    mov al, 5
    mov ebx, exception_5
    call set_idt_entry_all

    mov al, 6
    mov ebx, exception_6
    call set_idt_entry_all

    mov al, 7
    mov ebx, exception_7
    call set_idt_entry_all

    mov al, 8
    mov ebx, exception_8
    call set_idt_entry_all

    mov al, 13
    mov ebx, exception_13
    call set_idt_entry_all

    mov al, 14
    mov ebx, exception_14
    call set_idt_entry_all

    ; ─── Hardware IRQ Handlers (vectors 0x20-0x2F) ───
    ; Interrupt gates สำหรับ IRQs (DPL=0, clears IF)
    mov dl, INT_GATE_32

    mov al, 0x20
    mov ebx, irq0_stub
    call set_idt_entry_all

    mov al, 0x21
    mov ebx, irq1_stub
    call set_idt_entry_all

    mov al, 0x22
    mov ebx, irq2_stub
    call set_idt_entry_all

    ; ... (ต่อสำหรับ IRQ3-15)

    mov al, 0x2E
    mov ebx, irq14_stub
    call set_idt_entry_all

    mov al, 0x2F
    mov ebx, irq15_stub
    call set_idt_entry_all

    pop ebx
    pop eax
    ret

; ─── Helper ─────────────────────────────────────────
; set_idt_entry_all
; Input: AL=vector, EBX=handler, CX=selector, DL=type_attr
set_idt_entry_all:
    push eax
    push ebx
    push edi

    movzx eax, al
    lea edi, [idt + eax*8]

    mov [edi], bx           ; offset low
    mov [edi+2], cx         ; selector
    mov byte [edi+4], 0     ; reserved
    mov [edi+5], dl         ; type_attr
    shr ebx, 16
    mov [edi+6], bx         ; offset high

    pop edi
    pop ebx
    pop eax
    ret
```

---

## 22. สรุปและ Checklist

### Checklist ในการ Setup IDT:

```
□ 1. สร้าง IDT array (256 × 8 bytes) ใน memory
□ 2. ตั้งค่า IDTR (base address + limit)
□ 3. เขียน exception handlers (vectors 0-21)
□ 4. Initialize PIC (ICW1 → ICW2 → ICW3 → ICW4)
□ 5. Remap IRQs (master → 0x20-0x27, slave → 0x28-0x2F)
□ 6. เขียน IRQ handlers (vectors 0x20-0x2F)
□ 7. ส่ง EOI ท้ายทุก IRQ handler
□ 8. Unmask เฉพาะ IRQ ที่ต้องการ
□ 9. LIDT (โหลด IDTR เข้า CPU)
□ 10. STI (เปิด maskable interrupts)
```

### ข้อผิดพลาดที่พบบ่อย:

```
ปัญหา                          สาเหตุ                    วิธีแก้
─────────────────────────────────────────────────────────────────────────
Triple fault ทันที              IDT ผิดหรือว่างเปล่า       ตรวจสอบ IDT entries
Interrupt loop                 ไม่ส่ง EOI                 เพิ่ม EOI ท้าย handler
Keyboard ไม่ตอบสนอง            IRQ1 masked                 Unmask IRQ1
Timer ไม่ทำงาน                 IRQ0 masked                 Unmask IRQ0
#GP ทุกครั้ง                   CS selector ผิด             ตรวจสอบ GDT/selector
Stack corruption               ไม่ pop error code          add esp, 4 ก่อน iret
Spurious interrupts            ไม่ตรวจ ISR                ตรวจ ISR ก่อน EOI
Page fault loop                fault handler ก็ fault      ตรวจสอบ stack ของ handler
```

### การ Debug ด้วย QEMU:

```bash
# แสดง interrupt log
qemu-system-i386 -kernel kernel.bin -d int 2>&1 | grep -E "^(SMM|IRQ|exception)"

# หยุดที่ interrupt แรก (ใน GDB)
# (gdb) watch $pc
# (gdb) set scheduler-locking on

# ดู IDTR ใน QEMU monitor (Ctrl+Alt+2):
# (qemu) info registers
# หรือ:
# (qemu) x/8gx 0xADDRESS_OF_IDT  — ดู IDT entries

# เปิด QEMU monitor พร้อมกัน
qemu-system-i386 -kernel kernel.bin -monitor stdio
```

---

## 23. Advanced: APIC (สำหรับอ่านเพิ่มเติม)

ระบบสมัยใหม่ใช้ **APIC (Advanced PIC)** แทน 8259A:

```
8259A PIC:                    APIC:
─────────────────────         ─────────────────────────────
16 IRQ lines เท่านั้น         255 interrupt vectors
ไม่รองรับ multi-CPU           รองรับ multi-CPU (SMP)
I/O port control              Memory-mapped registers
Priority แบบง่าย              Priority แบบ flexible

Components:
  Local APIC (LAPIC):   อยู่ใน CPU แต่ละตัว
  I/O APIC:             อยู่ใน chipset, รับ IRQ จาก hardware
```

การ disable 8259A และเปิดใช้ APIC:

```nasm
; Disable legacy PIC
mov al, 0xFF
out PIC1_DATA, al       ; mask all master IRQs
out PIC2_DATA, al       ; mask all slave IRQs

; Enable Local APIC (ผ่าน MSR)
mov ecx, 0x1B           ; IA32_APIC_BASE MSR
rdmsr
or eax, 0x800           ; APIC enable bit
wrmsr

; ... (ต้องการ setup เพิ่มเติมอีกมาก)
```

---

## บทสรุป

ใน Part นี้เราได้เรียนรู้:

1. **Exception Types**: fault (re-execute), trap (continue), abort (fatal)
2. **Exception vectors 0-31**: ทุกตัวถูกกำหนดโดย Intel
3. **IDT Entry Format**: 8 bytes ต่อ entry, มี offset, selector, type
4. **Gate Types**: interrupt gate (clear IF), trap gate (ไม่เปลี่ยน IF)
5. **IDTR และ LIDT**: วิธี load IDT เข้า CPU
6. **Exception Handlers**: #DE, #BP, #UD, #GP, #PF พร้อม error code handling
7. **Interrupt Stack Frame**: CPU push EIP, CS, EFLAGS (+ ESP, SS ถ้าเปลี่ยน privilege)
8. **iret vs ret**: ต้องใช้ iret เพื่อ restore EFLAGS
9. **8259A PIC**: ICW initialization sequence, remap IRQs
10. **EOI**: ส่ง 0x20 หลังจัดการ IRQ
11. **Timer (IRQ0) และ Keyboard (IRQ1)**: handlers สมบูรณ์
12. **Spurious Interrupts**: ตรวจสอบ ISR ก่อนส่ง EOI

### ต่อไปใน Part 085: Memory Management และ Paging

---

*จบ Part 084: Interrupt Descriptor Table (IDT) and PIC*
