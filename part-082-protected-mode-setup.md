# Part 082: Protected Mode Setup

## บทนำ (Introduction)

Protected Mode คือโหมดการทำงานหลักของโปรเซสเซอร์ x86 ที่ถูกเปิดใช้งานตั้งแต่ Intel 80286 และยังคงเป็นพื้นฐานของระบบปฏิบัติการสมัยใหม่ ในบทนี้เราจะเรียนรู้วิธีการเปลี่ยนจาก Real Mode ไปสู่ Protected Mode พร้อมกับทำความเข้าใจโครงสร้างหน่วยความจำและกลไกการป้องกันที่ Protected Mode มีให้

---

## 1. ข้อจำกัดของ Real Mode (Real Mode Limitations)

### 1.1 สถาปัตยกรรมของ Real Mode

Real Mode คือโหมดที่โปรเซสเซอร์เริ่มต้นทำงานเมื่อเปิดเครื่อง มันถูกออกแบบมาให้เข้ากันได้กับ Intel 8086 ที่เปิดตัวในปี 1978

```
Real Mode Memory Map:
┌─────────────────────────────────┐ 0x100000 (1MB)
│      BIOS ROM & Extensions      │
├─────────────────────────────────┤ 0xC0000
│         Video Memory            │
├─────────────────────────────────┤ 0xA0000
│      Upper Memory Area          │
├─────────────────────────────────┤ 0x9FFFF
│                                 │
│     Conventional Memory         │
│         (640 KB)                │
│                                 │
├─────────────────────────────────┤ 0x00500
│         BIOS Data Area          │
├─────────────────────────────────┤ 0x00400
│   Interrupt Vector Table (IVT)  │
└─────────────────────────────────┘ 0x00000
```

### 1.2 ข้อจำกัดหลักของ Real Mode

**1. ขนาดหน่วยความจำจำกัดที่ 1MB**

Real Mode ใช้ Segmented Addressing แบบ 20 บิต:
```
Physical Address = Segment × 16 + Offset
                 = (16-bit Segment × 0x10) + 16-bit Offset
```

ตัวอย่างการคำนวณ:
```nasm
; Segment = 0x1000, Offset = 0x0500
; Physical = 0x1000 * 0x10 + 0x0500
;          = 0x10000 + 0x0500
;          = 0x10500

mov ax, 0x1000
mov ds, ax
mov bx, 0x0500
mov al, [ds:bx]    ; อ่านจาก Physical Address 0x10500
```

ที่อยู่สูงสุดที่เข้าถึงได้:
```
Segment = 0xFFFF, Offset = 0xFFFF
Physical = 0xFFFF0 + 0xFFFF = 0x10FFEF
```
แต่ bus address มีแค่ 20 บิต ทำให้ wrap around กลับมาที่ 0x0FFEF (A20 Line issue)

**2. โค้ด 16 บิตเท่านั้น**

```nasm
; Real Mode - ใช้ 16-bit registers เท่านั้น
mov ax, 0x1234      ; 16-bit operation
mov bx, ax          ; ไม่สามารถใช้ EAX, EBX ได้โดยตรง
                    ; (ต้อง use prefix 0x66)
```

**3. ไม่มีการป้องกันหน่วยความจำ (No Memory Protection)**

โปรแกรมใดก็สามารถเขียนทับหน่วยความจำใดก็ได้:
```nasm
; อันตราย! เขียนทับ IVT ได้โดยตรง
mov ax, 0
mov ds, ax
mov word [0x00], 0xDEAD    ; เขียนทับ Interrupt Vector 0!
```

**4. Single Tasking โดยธรรมชาติ**
- ไม่มี privilege levels
- ไม่มี virtual memory
- โปรแกรมทุกตัวทำงานที่ privilege level เดียวกัน
- ไม่สามารถ isolate processes ได้

**5. ไม่มี Hardware Multitasking Support**
- ไม่มี TSS (Task State Segment)
- Context switching ต้องทำเองทั้งหมด

---

## 2. ข้อดีของ Protected Mode (Protected Mode Advantages)

### 2.1 การเข้าถึงหน่วยความจำ 4GB

Protected Mode ใช้ Linear Address 32 บิต:
```
Linear Address Space: 0x00000000 - 0xFFFFFFFF (4 Gigabytes)
```

```
Protected Mode Memory Model:
┌─────────────────────────────────┐ 0xFFFFFFFF (4GB)
│                                 │
│        Kernel Space             │
│     (Usually 1GB or 2GB)        │
│                                 │
├─────────────────────────────────┤ 0xC0000000 (3GB typical)
│                                 │
│        User Space               │
│     (Usually 2GB or 3GB)        │
│                                 │
│                                 │
└─────────────────────────────────┘ 0x00000000
```

### 2.2 โค้ด 32 บิต (32-bit Code)

```nasm
[BITS 32]
; สามารถใช้ 32-bit registers ได้เต็มที่
mov eax, 0x12345678    ; 32-bit immediate
mov ebx, 0xDEADBEEF
add eax, ebx
push eax               ; stack operations ด้วย 32-bit
pop ecx
```

### 2.3 Virtual Memory

```
Virtual Memory Concept:
Process A sees:          Process B sees:
0x00000000               0x00000000
    ↓                        ↓
[Page Table A]           [Page Table B]
    ↓                        ↓
Physical RAM             Physical RAM
```

กลไก Paging ช่วยให้:
- แต่ละ process เห็น address space ของตัวเอง
- หน่วยความจำจริงอาจอยู่คนละที่กับ virtual address
- สามารถ swap pages ไปยัง disk ได้
- Memory mapping files

### 2.4 Protection Rings

```
Ring Architecture:
┌─────────────────────────────────┐
│  Ring 0: Kernel (Most Trusted)  │  ← OS Kernel ทำงานที่นี่
├─────────────────────────────────┤
│  Ring 1: (Rarely used)          │  ← Device Drivers บางครั้ง
├─────────────────────────────────┤
│  Ring 2: (Rarely used)          │  ← Device Drivers บางครั้ง
├─────────────────────────────────┤
│  Ring 3: User (Least Trusted)   │  ← User Programs ทำงานที่นี่
└─────────────────────────────────┘
```

ระบบปฏิบัติการส่วนใหญ่ใช้แค่ Ring 0 (kernel) และ Ring 3 (user):

```
Linux/Windows Ring Usage:
Ring 0: Kernel code, device drivers (trusted)
Ring 3: User applications (untrusted)

Privilege Level Stored in:
- CS register bits 0-1 (CPL - Current Privilege Level)
- Descriptor DPL field
- Selector RPL field
```

### 2.5 ประโยชน์อื่นๆ

| คุณสมบัติ | Real Mode | Protected Mode |
|-----------|-----------|----------------|
| Address Space | 1MB | 4GB |
| Register Width | 16-bit | 32-bit (64-bit ใน Long Mode) |
| Memory Protection | ไม่มี | มี |
| Virtual Memory | ไม่มี | มี (with Paging) |
| Hardware Multitasking | ไม่มี | มี (TSS) |
| Privilege Levels | 1 | 4 |
| Descriptor Tables | ไม่มี | GDT, LDT, IDT |

---

## 3. ขั้นตอนการเข้า Protected Mode

### 3.1 ภาพรวมขั้นตอน (Overview)

```
Real Mode Boot Process → Protected Mode Transition:

1. BIOS POST → Load Bootloader at 0x7C00
2. Bootloader ใน Real Mode
3. ─── Transition Steps ───
   a. Disable Interrupts (CLI)
   b. Load GDT (LGDT)
   c. Set PE bit in CR0
   d. Far Jump to flush pipeline
   e. Reload Segment Registers
4. Now in Protected Mode!
5. Setup Stack
6. Call C kernel or continue in ASM
```

### 3.2 ขั้นตอนที่ 1: Disable Interrupts (CLI)

ก่อนเข้า Protected Mode ต้องปิด interrupts เพราะ:
- Real Mode interrupt handlers ที่ BIOS ตั้งค่าไว้ใน IVT จะไม่ทำงานใน Protected Mode
- Protected Mode ใช้ IDT (Interrupt Descriptor Table) แทน IVT
- การ interrupt ระหว่าง transition จะทำให้ระบบ crash

```nasm
; ขั้นตอนที่ 1: ปิด Interrupts
cli                    ; Clear Interrupt Flag (IF = 0)
                       ; ทำให้ CPU ไม่รับ maskable interrupts
```

**หมายเหตุ:** NMI (Non-Maskable Interrupt) ยังไม่ถูกปิดด้วย CLI แต่สำหรับการสอนนี้เราจะไม่จัดการกับ NMI

### 3.3 ขั้นตอนที่ 2: Load GDT (LGDT)

GDT (Global Descriptor Table) คือตาราง descriptor ที่ CPU ใช้อ้างอิง segment ต่างๆ:

```nasm
; โครงสร้าง GDT Descriptor (สำหรับ LGDT instruction)
gdt_descriptor:
    dw gdt_end - gdt_start - 1    ; ขนาด GDT - 1 (16-bit limit)
    dd gdt_start                   ; ที่อยู่เริ่มต้นของ GDT (32-bit base)

; โหลด GDT
lgdt [gdt_descriptor]             ; บอก CPU ว่า GDT อยู่ที่ไหน
```

### 3.4 ขั้นตอนที่ 3: Set CR0 PE Bit

CR0 (Control Register 0) มี flag สำคัญหลายตัว:

```
CR0 Register Bit Layout:
Bit 31: PG  - Paging Enable
Bit 30: CD  - Cache Disable
Bit 29: NW  - Not Write-through
Bit 18: AM  - Alignment Mask
Bit 16: WP  - Write Protect
Bit  5: NE  - Numeric Error
Bit  4: ET  - Extension Type
Bit  3: TS  - Task Switched
Bit  2: EM  - Emulation
Bit  1: MP  - Monitor Coprocessor
Bit  0: PE  - Protection Enable  ← เราต้องตั้งค่า bit นี้!
```

```nasm
; ขั้นตอนที่ 3: เปิด Protection Enable
mov eax, cr0           ; อ่านค่า CR0
or eax, 0x1            ; ตั้ง bit 0 (PE = Protection Enable)
mov cr0, eax           ; เขียนกลับไปยัง CR0
; ณ จุดนี้ CPU อยู่ใน Protected Mode แต่ยังต้องทำต่อ!
```

**คำเตือน:** หลังจาก set PE bit แล้ว CPU ยังใช้ segment cache เก่าอยู่ ต้อง far jump เพื่อ flush

### 3.5 ขั้นตอนที่ 4: Far Jump เพื่อ Flush Pipeline

Far Jump (JMP far) คำสั่ง jump ที่เปลี่ยนทั้ง CS (Code Segment) และ IP (Instruction Pointer):

```nasm
; ขั้นตอนที่ 4: Far Jump ไปยัง 32-bit code
; Format: JMP segment_selector:offset
jmp 0x08:protected_mode_entry
; 0x08 คือ Code Segment Selector (index 1 ใน GDT, RPL=0)
; protected_mode_entry คือ label ในโค้ด 32-bit
```

**ทำไมต้อง Far Jump?**
1. Flush the prefetch queue - CPU บางรุ่นดึงคำสั่งล่วงหน้า การ flush ทำให้แน่ใจว่า CPU จะ decode ในโหมดใหม่
2. Load new CS descriptor - CS register ถูก reload จาก GDT
3. Ensures pipeline is in PM state - ทำให้แน่ใจว่า CPU ทำงานใน PM

### 3.6 ขั้นตอนที่ 5: Reload Segment Registers

หลังจาก far jump แล้ว CS ถูก reload แล้ว แต่ DS, ES, SS, FS, GS ยังเป็นค่าเก่า:

```nasm
[BITS 32]
protected_mode_entry:
    ; ขั้นตอนที่ 5: Reload Segment Registers
    mov ax, 0x10       ; Data Segment Selector (index 2 ใน GDT)
    mov ds, ax         ; Data Segment
    mov es, ax         ; Extra Segment
    mov ss, ax         ; Stack Segment
    mov fs, ax         ; FS (สำหรับ TLS ใน Linux)
    mov gs, ax         ; GS (สำหรับ TLS ใน Linux)
    
    ; ตั้งค่า Stack Pointer
    mov esp, 0x90000   ; Stack ที่ด้านบนของ Conventional Memory
    
    ; เราอยู่ใน Protected Mode แล้ว!
```

---

## 4. Global Descriptor Table (GDT)

### 4.1 GDT คืออะไร

GDT คือตารางที่เก็บ Segment Descriptors ซึ่งบอก CPU เกี่ยวกับ:
- ที่อยู่เริ่มต้นของ segment (Base Address)
- ขนาดของ segment (Limit)
- สิทธิ์การเข้าถึง (Access Rights)
- คุณสมบัติต่างๆ (Flags)

### 4.2 รูปแบบ Segment Descriptor (8 bytes)

```
Segment Descriptor Format (64 bits = 8 bytes):

Byte 7: Base[31:24]
Byte 6: Flags[7:4] | Limit[19:16]
Byte 5: Access Byte
Byte 4: Base[23:16]
Byte 3: Base[15:8]
Byte 2: Base[7:0]    ← Wait, this is confusing, let's draw it properly
Byte 1: Limit[15:8]
Byte 0: Limit[7:0]

More accurately:
Bits 63-56: Base[31:24]
Bits 55-52: Flags (G, D/B, L, AVL)
Bits 51-48: Limit[19:16]
Bits 47-40: Access Byte (P, DPL, S, Type)
Bits 39-32: Base[23:16]
Bits 31-16: Base[15:0]
Bits 15-0:  Limit[15:0]
```

### 4.3 Access Byte รายละเอียด

```
Access Byte (Bit 7 to 0):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ P │ DPL   │ S │   TYPE        │
│   │[6:5]  │   │[3:0]          │
└───┴───┴───┴───┴───┴───┴───┴───┘

P   (bit 7): Present - 1 ถ้า segment ใช้งานอยู่
DPL (bits 6-5): Descriptor Privilege Level (0=kernel, 3=user)
S   (bit 4): Descriptor Type
             0 = System Descriptor (TSS, LDT, Gate)
             1 = Code or Data Descriptor
TYPE (bits 3-0): Depends on S bit
  For Code (S=1):
    bit 3: 1 (code segment)
    bit 2: C - Conforming
    bit 1: R - Readable
    bit 0: A - Accessed
  For Data (S=1):
    bit 3: 0 (data segment)
    bit 2: E - Expand-down
    bit 1: W - Writable
    bit 0: A - Accessed
```

ตัวอย่าง Access Byte:
```
Code Segment, Ring 0, Readable:
P=1, DPL=00, S=1, Type=1010
= 1001 1010 = 0x9A

Data Segment, Ring 0, Writable:
P=1, DPL=00, S=1, Type=0010
= 1001 0010 = 0x92

Code Segment, Ring 3, Readable:
P=1, DPL=11, S=1, Type=1010
= 1111 1010 = 0xFA

Data Segment, Ring 3, Writable:
P=1, DPL=11, S=1, Type=0010
= 1111 0010 = 0xF2
```

### 4.4 Flags Nibble รายละเอียด

```
Flags (4 bits, upper nibble of byte 6):
┌───┬───┬───┬───┐
│ G │D/B│ L │AVL│
└───┴───┴───┴───┘

G   (bit 3): Granularity
             0 = Limit in bytes (max 1MB with 20-bit limit)
             1 = Limit in 4KB pages (max 4GB with 20-bit limit)
D/B (bit 2): Default Operation Size / Big
             For Code: 0=16-bit, 1=32-bit
             For Data/Stack: 0=16-bit SP, 1=32-bit ESP
L   (bit 1): Long mode (64-bit)
             1 = 64-bit code segment (for Long Mode)
AVL (bit 0): Available for system software (ignored by CPU)
```

### 4.5 Minimum GDT: Null + Code + Data

GDT ต้องมีอย่างน้อย 3 descriptors:

```nasm
; GDT สำหรับ 32-bit Protected Mode (Flat Memory Model)
gdt_start:

; Descriptor 0: Null Descriptor (required by CPU spec)
; CPU โหลด segment descriptor นี้เมื่อ segment register = 0
; ถ้า CPU พยายามใช้งาน segment ที่ point ไปยัง null descriptor
; จะเกิด General Protection Fault
gdt_null:
    dd 0x00000000    ; 4 bytes แรก = 0
    dd 0x00000000    ; 4 bytes หลัง = 0

; Descriptor 1: Code Segment (selector = 0x08)
; Index 1, TI=0 (GDT), RPL=0
; Selector calculation: (index << 3) | TI | RPL
;                     = (1 << 3) | 0 | 0 = 0x08
gdt_code:
    dw 0xFFFF        ; Limit[15:0] = 0xFFFF
    dw 0x0000        ; Base[15:0] = 0x0000
    db 0x00          ; Base[23:16] = 0x00
    db 10011010b     ; Access: P=1, DPL=0, S=1, Type=1010 (code, readable)
    db 11001111b     ; Flags+Limit[19:16]: G=1, D=1, L=0, AVL=0, Limit[19:16]=0xF
    db 0x00          ; Base[31:24] = 0x00

; Descriptor 2: Data Segment (selector = 0x10)
; Index 2, TI=0 (GDT), RPL=0
; Selector = (2 << 3) | 0 | 0 = 0x10
gdt_data:
    dw 0xFFFF        ; Limit[15:0] = 0xFFFF
    dw 0x0000        ; Base[15:0] = 0x0000
    db 0x00          ; Base[23:16] = 0x00
    db 10010010b     ; Access: P=1, DPL=0, S=1, Type=0010 (data, writable)
    db 11001111b     ; Flags+Limit[19:16]: G=1, B=1, L=0, AVL=0, Limit[19:16]=0xF
    db 0x00          ; Base[31:24] = 0x00

gdt_end:

; GDT Descriptor (สำหรับ LGDT instruction)
gdt_descriptor:
    dw gdt_end - gdt_start - 1    ; GDT size - 1
    dd gdt_start                   ; Linear address ของ GDT
```

**การคำนวณ Segment Selectors:**
```
Selector Format:
Bits 15-3: Index ใน GDT หรือ LDT
Bit 2:     TI (Table Indicator) - 0=GDT, 1=LDT
Bits 1-0:  RPL (Requested Privilege Level)

Null Descriptor:  Index=0, TI=0, RPL=0 → Selector = 0x0000
Code Descriptor:  Index=1, TI=0, RPL=0 → Selector = 0x0008
Data Descriptor:  Index=2, TI=0, RPL=0 → Selector = 0x0010
```

### 4.6 การสร้าง Descriptor ด้วย Macro

```nasm
; Macro สำหรับสร้าง GDT Descriptor
%macro GDT_ENTRY 4
    ; %1 = base, %2 = limit, %3 = access, %4 = flags
    dw (%2 & 0xFFFF)               ; limit low
    dw (%1 & 0xFFFF)               ; base low
    db ((%1 >> 16) & 0xFF)         ; base middle
    db %3                           ; access byte
    db ((%4 << 4) | ((%2 >> 16) & 0x0F))  ; flags + limit high
    db ((%1 >> 24) & 0xFF)         ; base high
%endmacro

; ใช้งาน:
gdt_start:
    GDT_ENTRY 0, 0, 0, 0                  ; Null descriptor
    GDT_ENTRY 0, 0xFFFFF, 0x9A, 0xC      ; Code: base=0, limit=4GB, ring0
    GDT_ENTRY 0, 0xFFFFF, 0x92, 0xC      ; Data: base=0, limit=4GB, ring0
gdt_end:

; Flags 0xC = 1100b = G=1, D/B=1, L=0, AVL=0
; Limit 0xFFFFF with G=1 → 0xFFFFF * 4096 = 0xFFFFFFFF = 4GB
```

---

## 5. LGDT Instruction และ GDT Descriptor Structure

### 5.1 LGDT Instruction

```nasm
; LGDT instruction syntax:
lgdt [memory_location]

; memory_location ชี้ไปยัง structure 6 bytes:
; ┌─────────────────┬──────────────────────────┐
; │  Limit (2 bytes)│  Base Address (4 bytes)  │
; └─────────────────┴──────────────────────────┘
; Byte 0-1: GDT Size - 1 (ขนาดสูงสุด = 65535 bytes = 8191 descriptors)
; Byte 2-5: Linear Base Address ของ GDT
```

### 5.2 สร้าง GDT Descriptor Structure

```nasm
; วิธีที่ 1: ใช้ dw และ dd แยกกัน
gdt_descriptor:
    dw gdt_end - gdt_start - 1    ; 16-bit limit
    dd gdt_start                   ; 32-bit base

; วิธีที่ 2: ใช้ struct (ถ้า NASM version รองรับ)
struc GDT_DESCRIPTOR
    .limit  resw 1
    .base   resd 1
endstruc

; วิธีที่ 3: คำนวณ offset ด้วยตัวเอง
GDT_DESC_LIMIT equ 0
GDT_DESC_BASE  equ 2

gdt_descriptor:
    dw 0    ; placeholder สำหรับ limit
    dd 0    ; placeholder สำหรับ base
```

### 5.3 ตัวอย่างการโหลด GDT ใน Real Mode

```nasm
[BITS 16]
[ORG 0x7C00]

setup_gdt:
    ; คำนวณที่อยู่จริงถ้าโค้ดถูกโหลดที่ 0x7C00
    ; GDT อาจอยู่หลัง bootloader code
    
    ; วิธีที่ 1: ใช้ linear address โดยตรง (ถ้าอยู่ใน first 64KB)
    lgdt [gdt_descriptor]
    
    ; วิธีที่ 2: คำนวณ linear address ถ้า segment ≠ 0
    ; (สำหรับกรณีที่ CS ≠ 0)
    xor eax, eax
    mov ax, cs
    shl eax, 4              ; segment * 16
    add eax, gdt_descriptor ; บวก offset
    ; ตอนนี้ EAX = linear address ของ gdt_descriptor
    ; แต่ LGDT ต้องการ address ใน [memory]
    ; ดังนั้นต้องทำให้ gdt_descriptor.base ถูกต้องก่อน
```

### 5.4 แก้ไข GDT Base Address แบบ Dynamic

```nasm
[BITS 16]
[ORG 0x7C00]

fix_gdt_base:
    ; คำนวณ physical address ของ gdt_start
    xor eax, eax
    mov ax, cs
    shl eax, 4              ; CS * 16 = segment base
    add eax, gdt_start      ; เพิ่ม offset
    mov [gdt_descriptor + 2], eax    ; เขียน base address ลงใน GDT descriptor
    
    lgdt [gdt_descriptor]
    ret
```

---

## 6. Far Jump เพื่อ Flush Pipeline

### 6.1 ทำไม Far Jump จึงจำเป็น

หลังจาก set PE bit แล้ว CPU ยังอยู่ในสถานะ "quasi-protected mode":
- instruction prefetch queue ยังมีคำสั่ง real mode อยู่
- CS descriptor cache ยังเป็นค่าเก่า
- CPU ต้องการ serialize execution

```nasm
; หลังจาก set CR0.PE:
mov cr0, eax           ; เปิด Protected Mode

; Far Jump - เปลี่ยน CS และ flush pipeline
; format: jmp selector:offset
jmp 0x08:pm_entry      ; 0x08 = Code Segment selector
                        ; pm_entry = label ใน 32-bit code section
```

### 6.2 วิธีการ Encode Far Jump ใน NASM

```nasm
; วิธีที่ 1: ใช้ NASM syntax โดยตรง
jmp 0x08:protected_mode_entry

; วิธีที่ 2: ใช้ db/dw สร้าง opcode เอง
; Opcode สำหรับ far jump = 0xEA
; ตามด้วย 4-byte offset แล้ว 2-byte segment
db 0xEA
dd protected_mode_entry    ; 32-bit offset
dw 0x08                    ; segment selector

; วิธีที่ 3: ใช้ o32 prefix (สำหรับ 16-bit code)
[BITS 16]
jmp dword 0x08:protected_mode_entry    ; o32 far jump
```

### 6.3 หลังจาก Far Jump

```nasm
[BITS 32]
protected_mode_entry:
    ; ตอนนี้ CS = 0x08 (Code Segment)
    ; CPU ทำงานใน 32-bit Protected Mode เต็มรูปแบบ
    
    ; CS descriptor cache ถูก reload จาก GDT แล้ว
    ; DPL ของ CS = 0 (Ring 0)
    ; Default operation size = 32-bit
    
    ; ต้อง reload segment registers อื่นๆ
    mov ax, 0x10       ; Data Segment
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; Setup Stack
    mov esp, stack_top
    
    ; พร้อมแล้ว!
```

---

## 7. Flat Memory Model

### 7.1 Flat Memory Model คืออะไร

Flat Memory Model คือ model ที่ทุก segment มี:
- Base = 0x00000000 (เริ่มต้นที่ byte แรก)
- Limit = 0xFFFFFFFF (ครอบคลุม 4GB ทั้งหมด)

ทำให้ Linear Address = Physical Address (เมื่อไม่ใช้ Paging)

```
Flat Memory Model:
┌──────────────────────────────────────┐
│ Code Segment: Base=0, Limit=4GB-1    │
│ Data Segment: Base=0, Limit=4GB-1    │
│ Stack Segment: Base=0, Limit=4GB-1   │
└──────────────────────────────────────┘
     ↕ All map to the same 4GB space
┌──────────────────────────────────────┐
│         Linear Address Space         │
│        0x00000000 - 0xFFFFFFFF       │
└──────────────────────────────────────┘
```

### 7.2 GDT สำหรับ Flat Memory Model

```nasm
gdt_start:

; Null Descriptor
dq 0

; Code Segment Descriptor
; Base = 0x00000000, Limit = 0xFFFFF (with G=1 → 4GB)
; Access = 0x9A (Present, Ring 0, Code, Readable)
; Flags = 0xCF (Granularity=1, 32-bit, Limit high=0xF)
code_seg:
    dw 0xFFFF        ; Limit[15:0]
    dw 0x0000        ; Base[15:0]
    db 0x00          ; Base[23:16]
    db 0x9A          ; Access Byte = 10011010b
    db 0xCF          ; Flags[7:4] + Limit[19:16] = 1100 1111
    db 0x00          ; Base[31:24]

; Data Segment Descriptor  
; Base = 0x00000000, Limit = 0xFFFFF (with G=1 → 4GB)
; Access = 0x92 (Present, Ring 0, Data, Writable)
data_seg:
    dw 0xFFFF        ; Limit[15:0]
    dw 0x0000        ; Base[15:0]
    db 0x00          ; Base[23:16]
    db 0x92          ; Access Byte = 10010010b
    db 0xCF          ; Flags[7:4] + Limit[19:16] = 1100 1111
    db 0x00          ; Base[31:24]

gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1
    dd gdt_start
```

### 7.3 ความหมายของ Flags Byte 0xCF

```
0xCF = 1100 1111b

Bits 7-4 (Flags nibble): 1100
    G   = 1: Granularity = 4KB pages
    D/B = 1: 32-bit default operation size
    L   = 0: Not 64-bit mode
    AVL = 0: Not used

Bits 3-0 (Limit high): 1111 = 0xF
    Upper 4 bits ของ 20-bit limit

Limit = 0x0F_FFFF (20 bits)
With G=1: Actual Limit = 0x0F_FFFF * 4096 + 4095
        = 0xFFFF_FFFF = 4GB - 1
```

---

## 8. Segment Registers ใน Protected Mode

### 8.1 Selector vs Descriptor

ใน Protected Mode segment register ไม่ได้เก็บ base address แต่เก็บ **Selector**:

```
Segment Register (16-bit):
Bits 15-3: Index (13 bits → สูงสุด 8192 entries ใน GDT/LDT)
Bit 2:     TI - Table Indicator (0=GDT, 1=LDT)
Bits 1-0:  RPL - Requested Privilege Level (0-3)

Examples:
0x0000 = Index 0, GDT, Ring 0 (Null selector)
0x0008 = Index 1, GDT, Ring 0 (Code selector)
0x0010 = Index 2, GDT, Ring 0 (Data selector)
0x001B = Index 3, GDT, Ring 3 (Code selector, user)
0x0023 = Index 4, GDT, Ring 3 (Data selector, user)
```

### 8.2 การ Load Segment Registers

```nasm
[BITS 32]

; หลังเข้า Protected Mode:
; CS ถูก load โดย far jump แล้ว
; ต้อง reload segment registers อื่นๆ ด้วย

; Data segments ทั้งหมด ใช้ selector 0x10 (ใน flat model)
mov ax, 0x10       ; Data segment selector
mov ds, ax         ; Data Segment
mov es, ax         ; Extra Segment
mov fs, ax         ; F Segment
mov gs, ax         ; G Segment
mov ss, ax         ; Stack Segment

; ตั้งค่า ESP สำหรับ stack
mov esp, 0x00090000    ; Stack pointer (ลง conventional memory)
```

### 8.3 Segment Register Caching

CPU เก็บ segment descriptor ไว้ใน "invisible" portion ของ segment register:

```
Segment Register (visible + hidden):
┌──────────────────┬────────────────────────────────────────────┐
│   Visible (16)   │            Hidden Cache (96 bits)          │
├──────────────────┼─────────────┬───────────────┬─────────────┤
│    Selector      │   Base (32) │  Limit (32)   │  Access (32)│
└──────────────────┴─────────────┴───────────────┴─────────────┘
```

เมื่อโหลด selector ใหม่ CPU จะ:
1. อ่าน descriptor จาก GDT
2. validate descriptor
3. บันทึก base, limit, access ลงใน hidden cache
4. หลังจากนี้ memory accesses ใช้ค่าจาก cache ไม่ต้องอ่าน GDT ทุกครั้ง

---

## 9. Stack Setup ใน Protected Mode

### 9.1 Stack ใน Protected Mode

```nasm
[BITS 32]

; Stack grows downward
; กำหนด stack ที่ด้านบนของ conventional memory

STACK_TOP equ 0x00090000    ; ด้านบนของ conventional memory

setup_stack:
    mov ax, 0x10        ; Stack Segment = Data Segment
    mov ss, ax          ; โหลด SS ก่อน
    mov esp, STACK_TOP  ; ตั้ง Stack Pointer
    ; ebp ยังไม่จำเป็นต้องตั้งในตอนนี้
    xor ebp, ebp        ; เคลียร์ EBP เพื่อความสะอาด
```

### 9.2 Stack Layout

```
Memory Layout หลัง setup:

0x00090000  ← ESP (Stack Top - empty stack)
0x0008FFFC  ← หลัง push ครั้งแรก (ESP -= 4)
0x0008FFF8  ← หลัง push ครั้งที่สอง
...

0x00080000  ← โค้ดของเรา (boot sector + จาก disk)
...
0x00007E00  ← หลัง Boot Sector
0x00007C00  ← Boot Sector (512 bytes)
...
0x00000500  ← BIOS Data Area
0x00000400  ← IVT (ใน Real Mode) / IDT (ใน PM)
0x00000000
```

### 9.3 การใช้งาน Stack

```nasm
[BITS 32]

; Push และ Pop
push eax              ; ESP -= 4, [ESP] = EAX
pop ebx               ; EBX = [ESP], ESP += 4

; Function call
call my_function      ; push EIP, jump
; ใน function:
push ebp              ; บันทึก base pointer
mov ebp, esp          ; ตั้ง frame
sub esp, 16           ; allocate local variables
; ... code ...
mov esp, ebp          ; restore stack
pop ebp
ret                   ; pop EIP, jump back

; PUSHA/POPA - save/restore all registers
pusha                 ; push EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
; ...
popa                  ; restore all (ตรงข้าม)
```

---

## 10. Hello World ใน Protected Mode - VGA Text Mode

### 10.1 VGA Text Buffer

ใน Protected Mode เราไม่ใช้ BIOS interrupt แต่เขียนตรงไปยัง VGA text buffer:

```
VGA Text Buffer:
Physical Address: 0x000B8000
Size: 80 columns × 25 rows = 2000 characters
Each character = 2 bytes:
  Byte 0: ASCII character code
  Byte 1: Attribute byte
  
Attribute Byte Format:
Bits 7: Blink (or bright background on some hardware)
Bits 6-4: Background color (0-7)
Bits 3-0: Foreground color (0-15)

Color Codes:
0 = Black          8 = Dark Gray
1 = Blue           9 = Light Blue
2 = Green          A = Light Green
3 = Cyan           B = Light Cyan
4 = Red            C = Light Red
5 = Magenta        D = Light Magenta
6 = Brown          E = Yellow
7 = Light Gray     F = White
```

### 10.2 ตัวอย่าง: เขียนตัวอักษรที่ VGA

```nasm
[BITS 32]

VGA_BASE equ 0xB8000    ; VGA text buffer base address
VGA_COLS equ 80         ; จำนวนคอลัมน์
VGA_ROWS equ 25         ; จำนวนแถว

; เขียนอักขระเดี่ยวที่ตำแหน่ง (col, row)
; Input: AL = character, AH = attribute, BL = col, BH = row
write_char:
    push eax
    push ebx
    push edi
    
    ; คำนวณ offset = (row * 80 + col) * 2
    xor edi, edi
    movzx edi, bh      ; row
    imul edi, VGA_COLS ; row * 80
    movzx ebx, bl      ; col
    add edi, ebx       ; + col
    shl edi, 1         ; * 2 (เพราะแต่ละ cell = 2 bytes)
    add edi, VGA_BASE  ; + base address
    
    ; เขียนอักขระและ attribute
    mov [edi], al      ; character
    mov [edi+1], ah    ; attribute
    
    pop edi
    pop ebx
    pop eax
    ret

; ตัวอย่างการใช้งาน:
; เขียน 'H' สีเขียวบนพื้นดำที่ position (0,0)
mov al, 'H'         ; character
mov ah, 0x0A        ; attribute: black background, light green foreground
mov bl, 0           ; column 0
mov bh, 0           ; row 0
call write_char
```

### 10.3 เขียน String ที่ VGA

```nasm
[BITS 32]

; write_string: เขียน null-terminated string
; Input: ESI = pointer to string, BL = start col, BH = row, AH = attribute
; Modifies: ESI, BL
write_string:
    push eax
    push ebx
    
.loop:
    lodsb              ; AL = [ESI], ESI++
    test al, al        ; ตรวจสอบ null terminator
    jz .done
    
    call write_char    ; เขียนอักขระ
    inc bl             ; ไปคอลัมน์ถัดไป
    
    ; ตรวจสอบว่า wrap ไปแถวถัดไปหรือเปล่า
    cmp bl, VGA_COLS
    jl .loop
    
    ; ถึงจุดสิ้นสุดของแถว - ขึ้นแถวใหม่
    xor bl, bl         ; col = 0
    inc bh             ; row++
    
    ; ตรวจสอบว่าเต็มหน้าจอหรือเปล่า
    cmp bh, VGA_ROWS
    jge .scroll        ; scroll ถ้าเต็ม
    
    jmp .loop

.scroll:
    call scroll_screen
    dec bh             ; ใช้แถวล่าง
    jmp .loop

.done:
    pop ebx
    pop eax
    ret
```

### 10.4 ล้างหน้าจอ (Clear Screen)

```nasm
[BITS 32]

; clear_screen: ล้างหน้าจอทั้งหมด
; เติมด้วย space สีขาวบนพื้นดำ
clear_screen:
    push eax
    push ecx
    push edi
    
    mov edi, VGA_BASE
    mov ah, 0x07       ; attribute: white on black
    mov al, ' '        ; space character
    mov ecx, VGA_COLS * VGA_ROWS    ; จำนวน characters
    
.loop:
    mov [edi], ax      ; เขียน char + attribute
    add edi, 2         ; ไปตำแหน่งถัดไป
    loop .loop
    
    pop edi
    pop ecx
    pop eax
    ret
```

### 10.5 Scrolling

```nasm
[BITS 32]

; scroll_screen: เลื่อนหน้าจอขึ้น 1 แถว
scroll_screen:
    push eax
    push ecx
    push esi
    push edi
    
    ; ย้ายแถว 1-24 ขึ้นมาเป็นแถว 0-23
    mov esi, VGA_BASE + (VGA_COLS * 2)   ; source: แถว 1
    mov edi, VGA_BASE                     ; dest: แถว 0
    mov ecx, VGA_COLS * (VGA_ROWS - 1)   ; จำนวน words
    
    ; copy แบบ word (2 bytes ต่อครั้ง)
    rep movsw          ; [EDI] = [ESI], ESI+=2, EDI+=2, ECX--
    
    ; ล้างแถวล่างสุด
    mov edi, VGA_BASE + (VGA_COLS * (VGA_ROWS - 1) * 2)
    mov ax, 0x0720     ; space + white-on-black
    mov ecx, VGA_COLS
    rep stosw          ; [EDI] = AX, EDI+=2, ECX--
    
    pop edi
    pop esi
    pop ecx
    pop eax
    ret
```

### 10.6 Cursor Movement

```nasm
[BITS 32]

; VGA Hardware Cursor I/O Ports
VGA_CTRL_PORT equ 0x3D4
VGA_DATA_PORT equ 0x3D5

; set_cursor: ย้าย hardware cursor ไปตำแหน่ง (col, row)
; Input: BL = col, BH = row
set_cursor:
    push eax
    push edx
    
    ; คำนวณ linear position
    xor eax, eax
    movzx eax, bh      ; row
    imul eax, VGA_COLS ; row * 80
    movzx ebx, bl      ; col  
    add eax, ebx       ; + col = linear position
    
    ; ส่งค่า cursor position ไปยัง VGA controller
    ; High byte
    mov dx, VGA_CTRL_PORT
    mov al, 0x0E       ; Cursor High register
    out dx, al
    
    mov dx, VGA_DATA_PORT
    mov eax, eax
    shr eax, 8         ; high byte
    out dx, al
    
    ; Low byte
    mov dx, VGA_CTRL_PORT
    mov al, 0x0F       ; Cursor Low register
    out dx, al
    
    mov dx, VGA_DATA_PORT
    ; EAX ยังมี low byte อยู่ (ส่วน low 8 bits)
    ; แต่ต้องคำนวณใหม่
    xor eax, eax
    movzx eax, bh
    imul eax, VGA_COLS
    movzx ebx, bl
    add eax, ebx
    out dx, al         ; low byte
    
    pop edx
    pop eax
    ret
```

---

## 11. การกลับสู่ Real Mode (Back to Real Mode)

### 11.1 ทำไมต้องกลับ Real Mode

บางครั้งต้องการเรียก BIOS functions จาก Protected Mode kernel:
- อ่าน disk ผ่าน BIOS
- ตรวจสอบ hardware
- APM power management

### 11.2 ขั้นตอนการกลับ Real Mode

```nasm
[BITS 32]

; ขั้นตอนการกลับ Real Mode:
; 1. Load 16-bit data segments
; 2. Load 16-bit code segment (far jump)
; 3. Disable protection (CR0.PE = 0)
; 4. Far jump to real mode
; 5. Load real mode segments
; 6. Enable interrupts (STI)

switch_to_real_mode:
    ; 1. โหลด 16-bit compatible data segments
    ; ต้องมี descriptor ใน GDT ที่มี limit ≤ 0xFFFF
    mov ax, 0x28       ; 16-bit data segment selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; 2. Far jump ไปยัง 16-bit code segment
    jmp 0x20:real_mode_entry_16    ; 0x20 = 16-bit code segment selector
```

```nasm
[BITS 16]

real_mode_entry_16:
    ; 3. ปิด Protected Mode
    mov eax, cr0
    and eax, 0xFFFFFFFE    ; clear PE bit
    mov cr0, eax
    
    ; 4. Far jump ไปยัง real mode code
    ; หลังจากนี้ CS = real mode segment
    jmp 0x0000:real_mode_code
    
real_mode_code:
    ; 5. Load real mode segments
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00      ; restore stack (ปรับตามความต้องการ)
    
    ; 6. เปิด interrupts
    sti
    
    ; ตอนนี้อยู่ใน Real Mode แล้ว!
    ; สามารถเรียก BIOS interrupts ได้
```

### 11.3 GDT ที่รองรับ Real Mode Transition

```nasm
gdt_start:
    dq 0                    ; Null descriptor (0x00)
    
    ; 32-bit Code Segment (0x08)
    dw 0xFFFF, 0x0000
    db 0x00, 0x9A, 0xCF, 0x00
    
    ; 32-bit Data Segment (0x10)
    dw 0xFFFF, 0x0000
    db 0x00, 0x92, 0xCF, 0x00
    
    ; 16-bit Code Segment for Real Mode transition (0x18)
    ; Base=0, Limit=0xFFFF, 16-bit, Ring 0
    dw 0xFFFF, 0x0000
    db 0x00, 0x9A, 0x0F, 0x00  ; Flags=0x0 → no G, 16-bit
    
    ; 16-bit Data Segment for Real Mode transition (0x20)  
    ; Base=0, Limit=0xFFFF, 16-bit, Ring 0
    dw 0xFFFF, 0x0000
    db 0x00, 0x92, 0x0F, 0x00  ; Flags=0x0 → no G, 16-bit

gdt_end:
```

---

## 12. โปรแกรม Complete Example

### 12.1 boot.asm - ตัวอย่างสมบูรณ์

```nasm
;============================================================
; boot.asm - Protected Mode Demo Bootloader
; Build: nasm -f bin boot.asm -o boot.bin
; Test:  qemu-system-i386 -drive format=raw,file=boot.bin
;============================================================

[BITS 16]
[ORG 0x7C00]

;------------------------------------------------------------
; Entry Point - Real Mode Start
;------------------------------------------------------------
start:
    ; ตั้งค่า segment registers ใน Real Mode
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00         ; Stack เติบโตลงมาจาก 0x7C00
    
    ; แสดงข้อความว่ากำลังเข้า Protected Mode
    mov si, msg_entering
    call print_string_rm   ; ใช้ BIOS interrupt ยังได้อยู่
    
    ; ปิด Interrupts
    cli
    
    ; โหลด GDT
    lgdt [gdt_descriptor]
    
    ; เปิด Protected Mode
    mov eax, cr0
    or eax, 0x1
    mov cr0, eax
    
    ; Far Jump ไปยัง 32-bit Protected Mode code
    jmp CODE_SEG:init_pm

;------------------------------------------------------------
; Real Mode Print String (ใช้ BIOS INT 10h)
; Input: SI = pointer to null-terminated string
;------------------------------------------------------------
print_string_rm:
    mov ah, 0x0E       ; BIOS teletype function
    mov bh, 0x00       ; page 0
.loop:
    lodsb              ; AL = [SI++]
    test al, al
    jz .done
    int 0x10           ; BIOS interrupt
    jmp .loop
.done:
    ret

;------------------------------------------------------------
; Real Mode Messages
;------------------------------------------------------------
msg_entering db 'Entering Protected Mode...', 13, 10, 0

;------------------------------------------------------------
; Global Descriptor Table
;------------------------------------------------------------
gdt_start:
    ; Null Descriptor
    dq 0x0000000000000000

    ; Code Segment: Base=0, Limit=4GB, 32-bit, Ring 0
    ; Access: P=1, DPL=0, S=1, Type=1010 → 0x9A
    ; Flags: G=1, D=1 → 0xC, Limit[19:16]=0xF → 0xCF
gdt_code:
    dw 0xFFFF          ; Limit[15:0]
    dw 0x0000          ; Base[15:0]
    db 0x00            ; Base[23:16]
    db 0x9A            ; Access Byte
    db 0xCF            ; Flags | Limit[19:16]
    db 0x00            ; Base[31:24]

    ; Data Segment: Base=0, Limit=4GB, 32-bit, Ring 0
    ; Access: P=1, DPL=0, S=1, Type=0010 → 0x92
gdt_data:
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0x92
    db 0xCF
    db 0x00

gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1     ; GDT limit
    dd gdt_start                    ; GDT base

; Segment Selectors
CODE_SEG equ gdt_code - gdt_start  ; = 0x08
DATA_SEG equ gdt_data - gdt_start  ; = 0x10

;------------------------------------------------------------
; Protected Mode Initialization
;------------------------------------------------------------
[BITS 32]

init_pm:
    ; Reload segment registers
    mov ax, DATA_SEG
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; Setup Stack ที่ 0x90000
    mov esp, 0x90000
    
    ; เรียก main function
    call pm_main
    
    ; ถ้า main return (ไม่ควรเกิดขึ้น)
    jmp $

;------------------------------------------------------------
; Protected Mode Main
;------------------------------------------------------------
pm_main:
    push ebp
    mov ebp, esp
    
    ; ล้างหน้าจอ
    call clear_screen
    
    ; แสดงข้อความต้อนรับ
    push dword 0x1F        ; attribute: blue background, white text
    push dword 0            ; row 0
    push dword 0            ; col 0
    push dword msg_welcome
    call print_colored_string
    add esp, 16
    
    ; แสดงข้อความอื่นๆ
    push dword 0x0A        ; attribute: black background, green text
    push dword 2            ; row 2
    push dword 0            ; col 0
    push dword msg_success
    call print_colored_string
    add esp, 16
    
    push dword 0x0E        ; attribute: black background, yellow text
    push dword 4            ; row 4
    push dword 0
    push dword msg_info
    call print_colored_string
    add esp, 16
    
    ; วน loop ไม่รู้จบ
    jmp $
    
    pop ebp
    ret

;------------------------------------------------------------
; Protected Mode Messages
;------------------------------------------------------------
msg_welcome  db '=== Welcome to Protected Mode! ===', 0
msg_success  db 'Successfully switched from Real Mode to Protected Mode!', 0
msg_info     db '32-bit code running, 4GB address space available.', 0

;------------------------------------------------------------
; VGA Text Mode Functions
;------------------------------------------------------------

VGA_BASE equ 0xB8000
VGA_COLS equ 80
VGA_ROWS equ 25

; clear_screen: ล้างหน้าจอทั้งหมด
clear_screen:
    push eax
    push ecx
    push edi
    
    mov edi, VGA_BASE
    mov eax, 0x07200720    ; 2 spaces with white-on-black attribute
    mov ecx, (VGA_COLS * VGA_ROWS) / 2  ; จำนวน dwords
    rep stosd              ; เติมด้วย dword ทีละตัว
    
    pop edi
    pop ecx
    pop eax
    ret

; print_colored_string: พิมพ์ string ด้วยสีที่กำหนด
; Stack frame: [esp+4]=string ptr, [esp+8]=col, [esp+12]=row, [esp+16]=attr
print_colored_string:
    push ebp
    mov ebp, esp
    push eax
    push ebx
    push esi
    push edi
    
    mov esi, [ebp+8]       ; string pointer
    mov ebx, [ebp+12]      ; col
    mov ecx, [ebp+16]      ; row
    mov ah, [ebp+20]       ; attribute byte
    
.loop:
    lodsb                  ; AL = [ESI++]
    test al, al
    jz .done
    
    ; คำนวณ VGA address
    mov edi, ecx           ; row
    imul edi, VGA_COLS     ; row * 80
    add edi, ebx           ; + col
    shl edi, 1             ; * 2
    add edi, VGA_BASE      ; + base
    
    ; เขียน char + attribute
    mov [edi], al
    mov [edi+1], ah
    
    inc ebx                ; col++
    jmp .loop
    
.done:
    pop edi
    pop esi
    pop ebx
    pop eax
    pop ebp
    ret

;------------------------------------------------------------
; Boot Signature
;------------------------------------------------------------
times 510 - ($ - $$) db 0    ; Pad to 510 bytes
dw 0xAA55                     ; Boot signature
```

---

## 13. ตัวอย่างขั้นสูง - พิมพ์ Text สีต่างๆ

### 13.1 demo_colors.asm

```nasm
;============================================================
; demo_colors.asm - สาธิต VGA สีต่างๆ ใน Protected Mode
;============================================================

[BITS 16]
[ORG 0x7C00]

start:
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00
    
    cli
    lgdt [gdt_desc]
    
    mov eax, cr0
    or eax, 1
    mov cr0, eax
    
    jmp 0x08:pm_start

;------------------------------------------------------------
; GDT
;------------------------------------------------------------
align 8
gdt:
    dq 0                            ; Null
    dq 0x00CF9A000000FFFF           ; Code: 0x08
    dq 0x00CF92000000FFFF           ; Data: 0x10

gdt_desc:
    dw $ - gdt - 1
    dd gdt

;------------------------------------------------------------
; Protected Mode Code
;------------------------------------------------------------
[BITS 32]

pm_start:
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov esp, 0x90000
    
    call clear_vga
    call demo_all_colors
    
    jmp $

;------------------------------------------------------------
; demo_all_colors: แสดงสีทุก combination
;------------------------------------------------------------
demo_all_colors:
    push ebp
    mov ebp, esp
    push eax
    push ebx
    push ecx
    push edx
    push edi
    
    ; แสดง title
    mov edi, VGA_BASE
    mov esi, title_str
    mov ah, 0x0F           ; white on black
    call write_str_at_edi
    
    ; loop ผ่านทุก foreground colors (0-15)
    xor ecx, ecx           ; row counter
    
.color_loop:
    cmp ecx, 16
    jge .done
    
    ; คำนวณ VGA position สำหรับแถว ecx+1
    mov edi, VGA_BASE
    add edi, VGA_COLS * 2  ; ข้าม title row
    mov eax, ecx
    imul eax, VGA_COLS * 2
    add edi, eax
    
    ; สร้าง attribute: background = ecx/2, foreground = ecx
    mov ah, cl             ; foreground = loop counter
    shl ah, 4              ; shift to background nibble
    or ah, cl              ; เพิ่ม foreground (เหมือน BG)
    xor ah, 0x07           ; สลับเพื่อให้เห็นชัด
    
    ; เขียน label
    mov esi, color_label
    call write_str_at_edi
    
    ; เขียน color number
    push eax
    push ecx
    call write_hex_byte    ; เขียนเลข hex ของ color
    pop ecx
    pop eax
    
    inc ecx
    jmp .color_loop
    
.done:
    pop edi
    pop edx
    pop ecx
    pop ebx
    pop eax
    pop ebp
    ret

;------------------------------------------------------------
; write_str_at_edi: เขียน string จาก ESI ไปยัง VGA[EDI]
; Input: ESI = string, AH = attribute, EDI = VGA address
; Modifies: ESI, EDI
;------------------------------------------------------------
write_str_at_edi:
.loop:
    lodsb
    test al, al
    jz .done
    mov [edi], ax      ; al=char, ah=attribute
    add edi, 2
    jmp .loop
.done:
    ret

;------------------------------------------------------------
; VGA Functions
;------------------------------------------------------------

VGA_BASE equ 0xB8000
VGA_COLS equ 80
VGA_ROWS equ 25

clear_vga:
    push eax
    push ecx
    push edi
    mov edi, VGA_BASE
    mov eax, 0x07200720
    mov ecx, VGA_COLS * VGA_ROWS / 2
    rep stosd
    pop edi
    pop ecx
    pop eax
    ret

write_hex_byte:
    ; เขียน CL เป็น 2-digit hex ที่ EDI
    ; (simplified version)
    push eax
    push ecx
    push edi
    mov al, cl
    shr al, 4
    call nibble_to_hex
    mov byte [edi], al
    mov byte [edi+1], 0x0F
    add edi, 2
    mov al, cl
    and al, 0x0F
    call nibble_to_hex
    mov byte [edi], al
    mov byte [edi+1], 0x0F
    add edi, 2
    pop edi
    pop ecx
    pop eax
    ret

nibble_to_hex:
    cmp al, 10
    jl .digit
    add al, 'A' - 10
    ret
.digit:
    add al, '0'
    ret

;------------------------------------------------------------
; Data
;------------------------------------------------------------
title_str   db 'VGA Color Demo - Protected Mode', 0
color_label db 'Color: ', 0

times 510 - ($ - $$) db 0
dw 0xAA55
```

---

## 14. การสร้างและทดสอบด้วย QEMU

### 14.1 Makefile

```makefile
# Makefile สำหรับ Protected Mode Examples

NASM = nasm
QEMU = qemu-system-i386

# Targets
.PHONY: all clean boot demo test-boot test-demo

all: boot.bin demo_colors.bin

# Build boot sector
boot.bin: boot.asm
	$(NASM) -f bin $< -o $@ -l boot.lst

# Build color demo
demo_colors.bin: demo_colors.asm
	$(NASM) -f bin $< -o $@

# Test boot sector in QEMU
test-boot: boot.bin
	$(QEMU) \
		-drive format=raw,file=boot.bin \
		-m 32 \
		-no-reboot \
		-no-shutdown \
		-display curses

# Test with SDL display (GUI)
test-boot-gui: boot.bin
	$(QEMU) \
		-drive format=raw,file=boot.bin \
		-m 32 \
		-no-reboot \
		-display sdl

# Debug with GDB
debug-boot: boot.bin
	$(QEMU) \
		-drive format=raw,file=boot.bin \
		-m 32 \
		-s -S \
		-no-reboot &
	gdb \
		-ex "target remote :1234" \
		-ex "set architecture i386" \
		-ex "break *0x7c00" \
		-ex "continue"

# Debug แบบ monitor
test-monitor: boot.bin
	$(QEMU) \
		-drive format=raw,file=boot.bin \
		-m 32 \
		-monitor stdio \
		-no-reboot

# ทดสอบ color demo
test-demo: demo_colors.bin
	$(QEMU) \
		-drive format=raw,file=demo_colors.bin \
		-m 32 \
		-no-reboot

clean:
	rm -f *.bin *.lst *.o

# สร้าง disk image ขนาด 1.44MB (Floppy)
floppy.img: boot.bin
	dd if=/dev/zero of=floppy.img bs=512 count=2880
	dd if=boot.bin of=floppy.img conv=notrunc

test-floppy: floppy.img
	$(QEMU) \
		-fda floppy.img \
		-m 32 \
		-no-reboot
```

### 14.2 QEMU Command Options ที่มีประโยชน์

```bash
# รันแบบพื้นฐาน
qemu-system-i386 -drive format=raw,file=boot.bin

# ปิด reboot อัตโนมัติ (เพื่อเห็น error messages)
qemu-system-i386 -drive format=raw,file=boot.bin -no-reboot -no-shutdown

# ใช้ RAM น้อยลง (32MB สำหรับ test)
qemu-system-i386 -drive format=raw,file=boot.bin -m 32

# ดู debug output
qemu-system-i386 -drive format=raw,file=boot.bin -d int,cpu_reset 2>&1 | head -50

# รันด้วย QEMU monitor ที่ stdio
qemu-system-i386 -drive format=raw,file=boot.bin -monitor stdio

# Debug กับ GDB (ใน terminal แรก)
qemu-system-i386 -drive format=raw,file=boot.bin -s -S

# GDB (ใน terminal ที่สอง)
gdb
(gdb) target remote :1234
(gdb) set architecture i386:intel
(gdb) break *0x7c00
(gdb) continue
(gdb) x/20i $pc         # ดู instructions รอบๆ PC
(gdb) info registers    # ดู registers ทั้งหมด
(gdb) x/4bx 0xb8000    # ดู VGA buffer
```

### 14.3 QEMU Monitor Commands

```
QEMU Monitor (กด Ctrl+Alt+2 เพื่อเข้า, Ctrl+Alt+1 เพื่อกลับ):

info registers        # แสดง CPU registers
info mem              # แสดง memory mapping
xp/10i 0x7c00        # disassemble 10 instructions ที่ 0x7c00
x/10x 0xb8000        # แสดง memory ที่ VGA buffer
x/10x 0x00           # แสดงหน่วยความจำที่ address 0
set var $eax = 0     # ตั้งค่า register
memsave 0 0x10000 mem.bin  # dump memory ไปยังไฟล์
quit                  # ออกจาก QEMU
```

---

## 15. Debugging Protected Mode Issues

### 15.1 ปัญหาที่พบบ่อย (Common Issues)

**ปัญหา 1: Triple Fault หลังเข้า Protected Mode**

```
สาเหตุ: GDT descriptor ไม่ถูกต้อง
แก้ไข:
1. ตรวจสอบ GDT descriptor format
2. ตรวจสอบว่า null descriptor เป็น 0 ทั้งหมด
3. ตรวจสอบ base address ของ GDT ว่าถูกต้อง

Debug:
qemu-system-i386 -drive format=raw,file=boot.bin \
    -d cpu_reset,int -no-reboot 2>&1
```

**ปัญหา 2: ไม่เห็นข้อความหลังเข้า PM**

```
สาเหตุ: segment registers ไม่ได้ถูก reload
แก้ไข:
mov ax, DATA_SEG   ; ต้อง reload ทันทีหลัง far jump
mov ds, ax
mov es, ax
mov ss, ax
...
```

**ปัญหา 3: Stack Overflow**

```
สาเหตุ: stack pointer ตั้งค่าผิด
แก้ไข:
mov esp, 0x90000   ; ตั้ง stack ให้ห่างจากโค้ด
; ตรวจสอบว่า stack ไม่ทับโค้ด
```

**ปัญหา 4: General Protection Fault**

```
สาเหตุ: เข้าถึง segment ที่ไม่อนุญาต
แก้ไข:
1. ตรวจสอบ DPL ของ descriptor
2. ตรวจสอบ CPL ของ code ที่รัน
3. ตรวจสอบ RPL ของ selector
```

### 15.2 การ Debug GDT

```nasm
; ใส่ breakpoint ก่อน LGDT เพื่อตรวจสอบ
[BITS 16]

debug_gdt:
    ; แสดงขนาด GDT
    mov ax, [gdt_descriptor]        ; limit
    ; พิมพ์ค่า...
    
    ; แสดง base address
    mov eax, [gdt_descriptor + 2]   ; base
    ; พิมพ์ค่า...
    
    ; ตรวจสอบว่า base ถูกต้อง
    ; ถ้า ORG 0x7C00 และ segment = 0:
    ; linear address ของ gdt_start ควร = 0x7C00 + offset
    
    lgdt [gdt_descriptor]
```

### 15.3 Verification Code

```nasm
[BITS 32]

; หลังเข้า Protected Mode ให้ตรวจสอบ
verify_pm:
    ; ตรวจสอบ CR0.PE
    mov eax, cr0
    and eax, 1
    jz .not_pm          ; ถ้า PE=0 แสดงว่าไม่ได้อยู่ใน PM
    
    ; ตรวจสอบ CS selector
    ; (ไม่สามารถอ่าน CS โดยตรงใน PM แต่ทำได้ผ่าน str trick)
    
    ; เขียน test pattern ที่ VGA
    mov dword [0xB8000], 0x1F4F1F4B  ; 'KO' สีฟ้าขาว
    jmp .ok
    
.not_pm:
    ; เขียน error message
    mov dword [0xB8000], 0x4C524F4E  ; 'NORE' สีแดง
    
.ok:
    ret
```

---

## 16. ตัวอย่าง: boot.asm ฉบับสมบูรณ์พร้อม Error Handling

```nasm
;============================================================
; boot_complete.asm - Complete Protected Mode Bootloader
; Features:
;   - GDT setup with proper descriptors
;   - Error handling
;   - VGA output
;   - Colored text
;   - Scrolling
;============================================================

[BITS 16]
[ORG 0x7C00]

;------------------------------------------------------------
; Constants
;------------------------------------------------------------
CODE_SEG equ 0x08
DATA_SEG equ 0x10

VGA_BASE equ 0xB8000
VGA_COLS equ 80
VGA_ROWS equ 25

STACK_TOP equ 0x90000

; VGA Attribute Colors
BLACK    equ 0x00
BLUE     equ 0x01
GREEN    equ 0x02
CYAN     equ 0x03
RED      equ 0x04
MAGENTA  equ 0x05
BROWN    equ 0x06
LGRAY    equ 0x07
DGRAY    equ 0x08
LBLUE    equ 0x09
LGREEN   equ 0x0A
LCYAN    equ 0x0B
LRED     equ 0x0C
LMAGENTA equ 0x0D
YELLOW   equ 0x0E
WHITE    equ 0x0F

; ฟังก์ชัน helper สำหรับ attribute: (BG << 4) | FG
%define ATTR(bg, fg) ((bg << 4) | fg)

;------------------------------------------------------------
; Real Mode Entry
;------------------------------------------------------------
boot_entry:
    ; เริ่มต้น Real Mode environment
    cli                        ; ปิด interrupts ชั่วคราว
    
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7BF0             ; Stack เริ่มก่อน boot sector
    
    sti                        ; เปิด interrupts อีกครั้ง
    
    ; แสดงข้อความด้วย BIOS
    mov si, rm_hello_msg
    call rm_puts
    
    ; ปิด interrupts ก่อนเข้า PM
    cli
    
    ; โหลด GDT
    lgdt [gdt_ptr]
    
    ; เปิด Protected Mode
    mov eax, cr0
    or eax, 1
    mov cr0, eax
    
    ; Far jump → flush pipeline + load CS
    jmp CODE_SEG:pm_entry

;------------------------------------------------------------
; Real Mode: puts (print string)
; Input: SI = pointer to null-terminated string
;------------------------------------------------------------
rm_puts:
    push ax
    push bx
    mov ah, 0x0E
    mov bh, 0x00
.loop:
    lodsb
    or al, al
    jz .done
    int 0x10
    jmp .loop
.done:
    pop bx
    pop ax
    ret

;------------------------------------------------------------
; Real Mode Messages
;------------------------------------------------------------
rm_hello_msg db '[RM] Booting... entering Protected Mode', 13, 10, 0

;------------------------------------------------------------
; GDT
;------------------------------------------------------------
align 8
gdt_table:

gdt_null:           ; Descriptor 0: Null (required)
    dq 0

gdt_code:           ; Descriptor 1: 32-bit Code (selector 0x08)
    dw 0xFFFF       ; Limit[15:0]
    dw 0x0000       ; Base[15:0]
    db 0x00         ; Base[23:16]
    db 10011010b    ; P=1, DPL=0, S=1, Type=Code+Read
    db 11001111b    ; G=1, D=1, L=0, AVL=0, Limit[19:16]=0xF
    db 0x00         ; Base[31:24]

gdt_data:           ; Descriptor 2: 32-bit Data (selector 0x10)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 10010010b    ; P=1, DPL=0, S=1, Type=Data+Write
    db 11001111b
    db 0x00

gdt_end:

gdt_ptr:            ; GDT Descriptor (สำหรับ LGDT)
    dw gdt_end - gdt_table - 1
    dd gdt_table

;------------------------------------------------------------
; Protected Mode Code
;------------------------------------------------------------
[BITS 32]

pm_entry:
    ; Reload segment registers
    mov ax, DATA_SEG
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; Setup Stack
    mov esp, STACK_TOP
    xor ebp, ebp
    
    ; Call PM main
    call pm_main
    
    ; หยุดถ้า main return
.halt:
    hlt
    jmp .halt

;------------------------------------------------------------
; PM Main Function
;------------------------------------------------------------
pm_main:
    push ebp
    mov ebp, esp
    sub esp, 32        ; allocate local space
    
    ; ล้างหน้าจอ
    call vga_clear
    
    ; วาด header
    push ATTR(BLUE, WHITE)     ; สีฟ้า-ขาว
    push 0                      ; row
    push 0                      ; col
    push header_msg
    call vga_print
    add esp, 16
    
    ; เติม header ไปตลอดแถว
    push ATTR(BLUE, WHITE)
    push 0
    push 35
    push header_fill
    call vga_print
    add esp, 16
    
    ; แสดง success message
    push ATTR(BLACK, LGREEN)
    push 2
    push 2
    push success_msg
    call vga_print
    add esp, 16
    
    ; แสดง info messages
    push ATTR(BLACK, LCYAN)
    push 4
    push 2
    push info_msg1
    call vga_print
    add esp, 16
    
    push ATTR(BLACK, LCYAN)
    push 5
    push 2
    push info_msg2
    call vga_print
    add esp, 16
    
    push ATTR(BLACK, YELLOW)
    push 7
    push 2
    push info_msg3
    call vga_print
    add esp, 16
    
    push ATTR(BLACK, LGRAY)
    push 9
    push 2
    push halt_msg
    call vga_print
    add esp, 16
    
    ; ทดสอบ: เขียน hex value ที่มุมขวา
    push ATTR(BLACK, LMAGENTA)
    push 0
    push 76
    push test_label
    call vga_print
    add esp, 16
    
    mov esp, ebp
    pop ebp
    ret

;------------------------------------------------------------
; Data / Strings
;------------------------------------------------------------
header_msg  db '=== Protected Mode Demo ===', 0
header_fill db '==========', 0
success_msg db '[OK] Successfully entered 32-bit Protected Mode!', 0
info_msg1   db '[*]  Code Segment: CS = 0x0008 (Ring 0, 32-bit)', 0
info_msg2   db '[*]  Data Segment: DS = 0x0010 (Ring 0, 32-bit)', 0
info_msg3   db '[*]  Stack at 0x90000, 4GB address space active', 0
halt_msg    db '[>>] System halted. Press Reset to restart.', 0
test_label  db 'PM!', 0

;------------------------------------------------------------
; VGA Functions (Protected Mode)
;------------------------------------------------------------

; vga_clear: ล้างหน้าจอ
vga_clear:
    push eax
    push ecx
    push edi
    
    mov edi, VGA_BASE
    mov eax, 0x07200720        ; 2x ' ' with white-on-black
    mov ecx, (VGA_COLS * VGA_ROWS) / 2
    rep stosd
    
    pop edi
    pop ecx
    pop eax
    ret

; vga_print: พิมพ์ string ที่ตำแหน่งที่กำหนด
; Stack: [esp+4]=str, [esp+8]=col, [esp+12]=row, [esp+16]=attr
vga_print:
    push ebp
    mov ebp, esp
    push eax
    push ebx
    push esi
    push edi
    
    mov esi, [ebp+8]           ; string pointer
    mov ebx, [ebp+12]          ; col
    mov ecx, [ebp+16]          ; row
    movzx eax, byte [ebp+20]   ; attribute byte
    mov ah, al
    
.next_char:
    lodsb
    test al, al
    jz .done
    
    ; คำนวณ VGA memory address
    push eax
    mov eax, ecx               ; row
    imul eax, VGA_COLS         ; × 80
    add eax, ebx               ; + col
    shl eax, 1                 ; × 2
    add eax, VGA_BASE          ; + base
    mov edi, eax
    pop eax
    
    ; เขียน character + attribute
    mov [edi], al              ; character
    mov [edi+1], ah            ; attribute
    
    inc ebx                    ; col++
    cmp ebx, VGA_COLS          ; wrap check
    jl .next_char
    xor ebx, ebx
    inc ecx
    jmp .next_char
    
.done:
    pop edi
    pop esi
    pop ebx
    pop eax
    pop ebp
    ret

; vga_putchar: พิมพ์อักขระเดี่ยวที่ EDI
; Input: AL=char, AH=attr, EDI=VGA_ptr
vga_putchar:
    mov [edi], ax
    add edi, 2
    ret

;------------------------------------------------------------
; Boot Signature
;------------------------------------------------------------
times 510 - ($ - $$) db 0
dw 0xAA55
```

---

## 17. แบบฝึกหัด (Exercises)

### 17.1 แบบฝึกหัดพื้นฐาน

**แบบฝึกหัด 1:** แก้ไขโค้ดให้แสดงข้อความ 10 บรรทัดด้วยสีต่างกันทุกบรรทัด

**แบบฝึกหัด 2:** เพิ่ม GDT descriptor สำหรับ Ring 3 (User mode) แล้วคำนวณ selector

**แบบฝึกหัด 3:** เขียน function ที่ล้างเฉพาะบางแถวของหน้าจอ

**แบบฝึกหัด 4:** เพิ่มการแสดงค่าของ CR0 register ในรูปแบบ hex บนหน้าจอ

### 17.2 แบบฝึกหัดขั้นกลาง

**แบบฝึกหัด 5:** เขียน function `vga_print_hex32` ที่รับ 32-bit value และพิมพ์เป็น hex 8 หลัก

```nasm
; โครงสร้างที่ต้องเขียน:
; vga_print_hex32:
;   Input: EAX = value, EDI = VGA address, AH = attribute
;   Output: พิมพ์ "0x" ตามด้วย 8 hex digits
```

**แบบฝึกหัด 6:** เพิ่ม function `vga_scroll_line` ที่เลื่อนหน้าจอขึ้น 1 แถวและล้างแถวล่างสุด

**แบบฝึกหัด 7:** สร้าง simple bootloader ที่:
1. อ่านข้อมูลจาก disk sector 2 (ใช้ BIOS INT 13h ใน real mode)
2. เข้า Protected Mode
3. แสดงข้อมูลที่อ่านมาบนหน้าจอ

### 17.3 แบบฝึกหัดขั้นสูง

**แบบฝึกหัด 8:** เพิ่ม IDT (Interrupt Descriptor Table) เบื้องต้น สำหรับจัดการ:
- Divide by Zero exception (#DE)
- General Protection Fault (#GP)
- Page Fault (#PF)

**แบบฝึกหัด 9:** เขียนระบบ task switching ง่ายๆ โดยใช้ TSS สองอัน สลับกันทุก 1000 iterations

**แบบฝึกหัด 10:** เพิ่ม user mode (Ring 3) segment descriptors และเขียนโค้ดที่ transition ไปยัง Ring 3

---

## 18. สรุป (Summary)

### 18.1 ขั้นตอนการเข้า Protected Mode - สรุปย่อ

```
1. CLI                          ; ปิด interrupts
2. LGDT [gdt_descriptor]        ; บอก CPU ตำแหน่ง GDT
3. MOV EAX, CR0                 ; อ่าน CR0
   OR EAX, 1                    ; ตั้ง PE bit
   MOV CR0, EAX                 ; เขียนกลับ
4. JMP CODE_SEG:pm_label        ; Far jump ไปยัง 32-bit code
5. [BITS 32]                    ; ตอนนี้อยู่ใน PM
   MOV AX, DATA_SEG             ; reload segment registers
   MOV DS, AX
   MOV ES, AX
   MOV SS, AX
   MOV ESP, stack_top            ; ตั้ง stack
```

### 18.2 GDT Minimum Requirements

```
GDT ต้องมีอย่างน้อย 3 entries:
Entry 0: Null Descriptor    (all zeros, required)
Entry 1: Code Descriptor    (selector = 0x08)
Entry 2: Data Descriptor    (selector = 0x10)

สำหรับ Flat 32-bit Memory Model:
- Base = 0x00000000
- Limit = 0xFFFFF (with G=1 → 4GB)
- Code Access = 0x9A
- Data Access = 0x92
- Flags = 0xCF
```

### 18.3 VGA Text Buffer Quick Reference

```
Address: 0xB8000
Format: char (1 byte) + attribute (1 byte) per character
Screen: 80 × 25 characters

Attribute: (background << 4) | foreground
Position: offset = (row * 80 + col) * 2

Common Attributes:
0x07 = white text on black background
0x0A = green text on black background
0x1F = white text on blue background
0x4F = white text on red background
```

### 18.4 Quick Checklist ก่อนทดสอบ

```
□ CLI ก่อน LGDT
□ GDT null descriptor = 0 ทั้งหมด
□ GDT base address ถูกต้อง (linear address)
□ Far jump ใช้ selector ที่ถูกต้อง (0x08)
□ [BITS 32] หลัง far jump label
□ Reload DS, ES, SS, FS, GS หลัง far jump
□ ตั้ง ESP ก่อนเรียก function
□ Boot signature 0xAA55 ที่ offset 510-511
```

---

## 19. อ้างอิงเพิ่มเติม (References)

### 19.1 Intel Manual References

- Intel® 64 and IA-32 Architectures Software Developer's Manual
  - Volume 3A: Chapter 3 (Protected-Mode Memory Management)
  - Volume 3A: Chapter 5 (Protection)
  - Volume 3A: Chapter 8 (Advanced Programmable Interrupt Controller)

### 19.2 GDT Descriptor Calculator

```python
#!/usr/bin/env python3
# gdt_calc.py - คำนวณ GDT Descriptor

def make_descriptor(base, limit, access, flags):
    """สร้าง 64-bit GDT Descriptor"""
    # ถ้า limit > 0xFFFFF ให้ตั้ง G bit และหาร limit ด้วย 0x1000
    if limit > 0xFFFFF:
        limit = (limit >> 12) & 0xFFFFF
        flags |= 0x8  # Set G bit
    
    desc = 0
    desc |= (limit & 0xFFFF)               # bits 15:0
    desc |= (base & 0xFFFF) << 16          # bits 31:16
    desc |= ((base >> 16) & 0xFF) << 32    # bits 39:32
    desc |= (access & 0xFF) << 40          # bits 47:40
    desc |= ((limit >> 16) & 0xF) << 48   # bits 51:48
    desc |= (flags & 0xF) << 52            # bits 55:52
    desc |= ((base >> 24) & 0xFF) << 56   # bits 63:56
    
    return desc

# ตัวอย่าง: Flat Code Segment
code_desc = make_descriptor(
    base   = 0x00000000,
    limit  = 0xFFFFFFFF,
    access = 0x9A,         # Ring 0, Code, Readable
    flags  = 0xC           # G=1, D=1
)
print(f"Code Descriptor: 0x{code_desc:016X}")

# แบ่งเป็น bytes (little-endian)
for i in range(8):
    byte_val = (code_desc >> (i * 8)) & 0xFF
    print(f"  Byte {i}: 0x{byte_val:02X}")
```

ผลลัพธ์:
```
Code Descriptor: 0x00CF9A000000FFFF
  Byte 0: 0xFF  (Limit[7:0])
  Byte 1: 0xFF  (Limit[15:8])
  Byte 2: 0x00  (Base[7:0])
  Byte 3: 0x00  (Base[15:8])
  Byte 4: 0x00  (Base[23:16])
  Byte 5: 0x9A  (Access)
  Byte 6: 0xCF  (Flags + Limit[19:16])
  Byte 7: 0x00  (Base[31:24])
```

### 19.3 Useful NASM Defines

```nasm
; ไฟล์ pm_defs.inc - ค่า constant ที่ใช้บ่อย

; Segment Selectors (Flat Memory Model)
%define PM_CODE_SEG     0x08
%define PM_DATA_SEG     0x10
%define PM_USER_CODE    0x1B    ; Ring 3 code
%define PM_USER_DATA    0x23    ; Ring 3 data

; VGA Constants
%define VGA_TEXTMODE    0xB8000
%define VGA_WIDTH       80
%define VGA_HEIGHT      25

; Color Attributes
%define VGA_BLACK       0x00
%define VGA_BLUE        0x01
%define VGA_GREEN       0x02
%define VGA_CYAN        0x03
%define VGA_RED         0x04
%define VGA_MAGENTA     0x05
%define VGA_BROWN       0x06
%define VGA_LGRAY       0x07
%define VGA_DGRAY       0x08
%define VGA_LBLUE       0x09
%define VGA_LGREEN      0x0A
%define VGA_LCYAN       0x0B
%define VGA_LRED        0x0C
%define VGA_LMAGENTA    0x0D
%define VGA_YELLOW      0x0E
%define VGA_WHITE       0x0F

; Attribute macro: VATTR(background, foreground)
%define VATTR(b,f)      (((b) << 4) | (f))

; Control Register bits
%define CR0_PE          0x00000001  ; Protection Enable
%define CR0_MP          0x00000002  ; Monitor Coprocessor
%define CR0_EM          0x00000004  ; Emulation
%define CR0_TS          0x00000008  ; Task Switched
%define CR0_WP          0x00010000  ; Write Protect
%define CR0_PG          0x80000000  ; Paging

; Descriptor Access Byte values
%define DA_CODE         0x9A    ; Code, Ring 0, Readable
%define DA_DATA         0x92    ; Data, Ring 0, Writable
%define DA_CODE_R3      0xFA    ; Code, Ring 3, Readable
%define DA_DATA_R3      0xF2    ; Data, Ring 3, Writable
%define DA_TSS          0x89    ; TSS (32-bit)

; Descriptor Flags nibble
%define DF_32BIT        0xC     ; G=1, D=1 (32-bit, 4KB granularity)
%define DF_16BIT        0x0     ; No G, no D (16-bit, byte granularity)
```

---

## บทสรุป (Conclusion)

ในบทนี้เราได้เรียนรู้:

1. **Real Mode Limitations** - ข้อจำกัดของ Real Mode: ขนาด 1MB, 16-bit, ไม่มี protection
2. **Protected Mode Advantages** - ข้อดีของ Protected Mode: 4GB, 32-bit, virtual memory, rings
3. **Transition Steps** - 5 ขั้นตอนสำคัญ: CLI → LGDT → Set PE → Far Jump → Reload Segments
4. **GDT Structure** - โครงสร้าง descriptor 8 bytes และ minimum GDT 3 entries
5. **LGDT Instruction** - GDT pointer structure และการโหลด
6. **Far Jump** - ทำไมจำเป็นและวิธี encode
7. **Flat Memory Model** - base=0, limit=4GB สำหรับทุก segment
8. **VGA Text Buffer** - เขียนข้อความที่ 0xB8000 โดยตรง
9. **Scrolling & Cursor** - จัดการหน้าจอใน Protected Mode
10. **Real Mode Return** - วิธีกลับมา Real Mode (สำหรับเรียก BIOS)

บทต่อไปจะเรียนรู้เรื่อง **IDT (Interrupt Descriptor Table)** และการจัดการ interrupts และ exceptions ใน Protected Mode

---

*จบ Part 082: Protected Mode Setup*

*ต่อไป → Part 083: Interrupt Descriptor Table (IDT)*
