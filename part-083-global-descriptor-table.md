# Part 083: Global Descriptor Table (GDT) Deep Dive

## บทนำ (Introduction)

Global Descriptor Table (GDT) คือโครงสร้างข้อมูลพื้นฐานที่สุดของ x86 Protected Mode และ Long Mode  
ก่อนที่ CPU จะเข้าสู่ Protected Mode ได้ เราต้องตั้งค่า GDT ให้ถูกต้องก่อน  
Part นี้จะอธิบายทุกอย่างเกี่ยวกับ GDT ตั้งแต่ต้นจนจบ รวมถึง TSS และ Privilege Rings

**สิ่งที่จะได้เรียนรู้:**
- โครงสร้างของ GDT และ Descriptor แต่ละชนิด
- Null Descriptor ทำไมต้องมี
- GDTR Register และ LGDT/SGDT Instructions
- Access Byte และ Flags Nibble แต่ละ bit หมายความว่าอะไร
- 64-bit Descriptors (Long Mode)
- Flat Memory Model
- Privilege Rings (Ring 0-3)
- Ring Transitions: Call Gates, SYSCALL/SYSRET
- Task State Segment (TSS)
- การสร้าง Complete GDT สำหรับ OS kernel

---

## 1. ทำไมต้องมี GDT?

ใน Real Mode (16-bit) การ address memory ทำโดย `segment:offset`  
segment register เก็บ paragraph address (คูณ 16 เพื่อได้ physical address)

```
Physical Address = Segment * 16 + Offset
ตัวอย่าง: CS=0x1000, IP=0x0200 → Physical = 0x10200
```

**ปัญหาของ Real Mode:**
- เข้าถึงได้แค่ 1MB (20-bit address)
- ไม่มี Memory Protection — โปรแกรมใด ๆ เขียนทับหน่วยความจำใดก็ได้
- ไม่มี Privilege Levels — โปรแกรม user ใช้ instruction ที่อันตรายได้

**Protected Mode แก้ปัญหาด้วย:**
- Segment Descriptors ที่เก็บ base, limit, และ access rights
- Privilege Levels (DPL/CPL) แยก kernel กับ user
- Paging เพิ่มเติม (ระดับที่สอง)

ใน Protected Mode segment register (CS, DS, ES, FS, GS, SS) ไม่ได้เก็บ address โดยตรง  
แต่เก็บ **Selector** ซึ่งเป็น index เข้าไปใน GDT หรือ LDT

---

## 2. โครงสร้างของ GDT

GDT คือ array ของ **8-byte entries** เรียกว่า **Segment Descriptors**

```
GDT Layout:
┌─────────────────────────────────────────┐
│ Index 0: Null Descriptor (8 bytes = 0)  │  ← ต้องเป็น 0 เสมอ
├─────────────────────────────────────────┤
│ Index 1: Kernel Code Segment            │
├─────────────────────────────────────────┤
│ Index 2: Kernel Data Segment            │
├─────────────────────────────────────────┤
│ Index 3: User Code Segment              │
├─────────────────────────────────────────┤
│ Index 4: User Data Segment              │
├─────────────────────────────────────────┤
│ Index 5: TSS Descriptor                 │
├─────────────────────────────────────────┤
│ ...                                     │
└─────────────────────────────────────────┘
```

**Selector Format (16 bits):**
```
Bits 15-3: Index ใน GDT (13 bits → สูงสุด 8191 entries)
Bit    2:  TI (Table Indicator): 0=GDT, 1=LDT
Bits  1-0: RPL (Requested Privilege Level): 0-3
```

ตัวอย่าง:
```
0x0008 = 0000 0000 0000 1 0 00b
         Index=1, TI=0(GDT), RPL=0 → Kernel Code Selector

0x0010 = 0000 0000 0001 0 0 00b
         Index=2, TI=0(GDT), RPL=0 → Kernel Data Selector

0x001B = 0000 0000 0001 1 0 11b
         Index=3, TI=0(GDT), RPL=3 → User Code Selector (Ring 3)
```

---

## 3. Null Descriptor

Entry แรก (Index 0) ของ GDT **ต้องเป็น 0 ทั้งหมด 8 bytes**

```nasm
; Null Descriptor — 8 bytes ของ zeros
gdt_null:
    dq 0x0000000000000000
```

**เหตุผล:**
- ถ้า segment register ถูก load ด้วย selector 0 แล้วมีการ access memory → CPU จะ generate General Protection Fault (#GP)
- เป็น safety net ป้องกัน uninitialized segment registers
- CPU spec บังคับให้ entry 0 เป็น null

ในทางปฏิบัติ:
```nasm
; เมื่อ boot ขึ้นมา
mov ax, 0
mov ds, ax    ; ถ้าใช้ DS=0 ใน protected mode → #GP!
```

---

## 4. Descriptor Format (8 Bytes)

นี่คือโครงสร้างที่ซับซ้อนที่สุด — Intel ออกแบบให้แปลกเพื่อ backward compatibility กับ 286

```
8-byte Segment Descriptor Layout:

Byte: 7        6        5        4        3        2        1        0
      ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
      │Base    │Flags+  │Access  │Base    │Base    │Limit   │Limit   │Limit   │
      │[31:24] │Limit   │Byte    │[23:16] │[15:0]  │[15:8]  │[15:8]  │[7:0]   │
      │        │[19:16] │        │        │(2 bytes)│       │        │        │
      └────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
         7        6        5        4       3,2      ...      1        0
```

อธิบายเป็น field:
```
Bits  7:0   (Byte 0)     = Limit[7:0]
Bits 15:8   (Byte 1)     = Limit[15:8]
Bits 31:16  (Bytes 2-3)  = Base[15:0]
Bits 39:32  (Byte 4)     = Base[23:16]
Bits 47:40  (Byte 5)     = Access Byte
Bits 51:48  (Byte 6 low) = Limit[19:16]
Bits 55:52  (Byte 6 high)= Flags Nibble
Bits 63:56  (Byte 7)     = Base[31:24]
```

**ทำไม Intel ออกแบบแบบนี้?**  
ใน 286 มี base 24-bit และ limit 16-bit เท่านั้น  
ตอน 386 เพิ่มเป็น 32-bit base จึงต้องแทรก bits เพิ่มโดยไม่ทำลาย compatibility

---

## 5. การประกอบ Descriptor ด้วย NASM Macro

```nasm
; Macro สำหรับสร้าง GDT Descriptor
; Parameters: base, limit, access, flags
%macro GDT_ENTRY 4
    ; %1 = base (32-bit)
    ; %2 = limit (20-bit, 0xFFFFF สำหรับ 4GB)
    ; %3 = access byte
    ; %4 = flags nibble (4-bit)
    dw (%2 & 0xFFFF)                    ; Limit[15:0]
    dw (%1 & 0xFFFF)                    ; Base[15:0]
    db ((%1 >> 16) & 0xFF)              ; Base[23:16]
    db %3                               ; Access Byte
    db ((%4 << 4) | ((%2 >> 16) & 0xF)); Flags + Limit[19:16]
    db ((%1 >> 24) & 0xFF)              ; Base[31:24]
%endmacro

; ตัวอย่างการใช้:
gdt_start:
    GDT_ENTRY 0, 0, 0, 0                ; Null Descriptor
    GDT_ENTRY 0, 0xFFFFF, 0x9A, 0xC    ; Kernel Code (32-bit)
    GDT_ENTRY 0, 0xFFFFF, 0x92, 0xC    ; Kernel Data (32-bit)
gdt_end:
```

---

## 6. Access Byte (Byte 5) — อธิบายทุก Bit

```
Access Byte: 8 bits

Bit 7: P   — Present
Bit 6: DPL[1] — Descriptor Privilege Level (bit 1)
Bit 5: DPL[0] — Descriptor Privilege Level (bit 0)
Bit 4: S   — Descriptor Type (0=System, 1=Code/Data)
Bit 3: Type[3] — ความหมายต่างกันสำหรับ Code/Data
Bit 2: Type[2] — ความหมายต่างกัน
Bit 1: Type[1] — ความหมายต่างกัน
Bit 0: A   — Accessed (CPU set เมื่อ segment ถูกใช้)
```

### Bit 7: P (Present)
```
P=1: Descriptor valid อยู่ใน memory
P=0: Segment ไม่ได้ present → load selector นี้จะ trigger Segment Not Present (#NP)
     ใช้สำหรับ swapping segments ออก disk (หายากมากใน modern OS)
```

### Bits 6-5: DPL (Descriptor Privilege Level)
```
DPL=00 (0): Ring 0 — Kernel mode, สิทธิ์สูงสุด
DPL=01 (1): Ring 1 — (ไม่ค่อยใช้ใน modern OS)
DPL=10 (2): Ring 2 — (ไม่ค่อยใช้ใน modern OS)
DPL=11 (3): Ring 3 — User mode, สิทธิ์ต่ำสุด
```

### Bit 4: S (Descriptor Type)
```
S=0: System Descriptor (TSS, Call Gate, LDT, etc.)
S=1: Code or Data Segment Descriptor
```

### Bits 3-0: Type (เมื่อ S=1)

**สำหรับ Data Segment (Type[3]=0):**
```
Bit 3: 0 (Data)
Bit 2: E — Expand-down (0=normal, 1=expand-down เช่น stack)
Bit 1: W — Writable (0=read-only, 1=writable)
Bit 0: A — Accessed

ตัวอย่าง:
Type = 0010b = 0x2 → Data, expand-up, writable, not accessed
Type = 0011b = 0x3 → Data, expand-up, writable, accessed
```

**สำหรับ Code Segment (Type[3]=1):**
```
Bit 3: 1 (Code)
Bit 2: C — Conforming
         0 = Non-conforming: code ใน ring น้อยกว่า DPL ไม่สามารถ execute ได้
         1 = Conforming: code สามารถถูกเรียกจาก ring ที่สูงกว่า (equal/lower privilege)
Bit 1: R — Readable (0=execute-only, 1=execute+read)
Bit 0: A — Accessed

ตัวอย่าง:
Type = 1000b = 0x8 → Code, non-conforming, execute-only
Type = 1010b = 0xA → Code, non-conforming, readable
Type = 1100b = 0xC → Code, conforming, execute-only
```

**ตารางสรุป Access Byte ที่ใช้บ่อย:**
```
0x9A = 1001 1010b = P=1, DPL=00, S=1, Type=1010 → Kernel Code (read/exec)
0x92 = 1001 0010b = P=1, DPL=00, S=1, Type=0010 → Kernel Data (read/write)
0xFA = 1111 1010b = P=1, DPL=11, S=1, Type=1010 → User Code (read/exec)
0xF2 = 1111 0010b = P=1, DPL=11, S=1, Type=0010 → User Data (read/write)
```

---

## 7. Flags Nibble (Byte 6 High) — 4 Bits

```
Bits: [G][D/B][L][AVL]

Bit 3 (G):   Granularity
             0 = Limit ใน bytes (สูงสุด 1MB)
             1 = Limit ใน 4KB pages (สูงสุด 4GB)

Bit 2 (D/B): Default operation size / Big
             สำหรับ Code segment: D=1 → 32-bit default operation size
             สำหรับ Code segment: D=0 → 16-bit default operation size
             สำหรับ Data/Stack: B=1 → 32-bit stack pointer (ESP)
             สำหรับ Data/Stack: B=0 → 16-bit stack pointer (SP)

Bit 1 (L):   Long mode bit
             L=1 → 64-bit Code Segment (Long Mode)
             L=0 → 32-bit or 16-bit
             ถ้า L=1 ต้อง D=0

Bit 0 (AVL): Available for OS use (CPU ไม่ใช้)
```

**ตัวอย่าง Flags:**
```
Flags = 0xC = 1100b → G=1 (4KB granularity), D/B=1 (32-bit), L=0, AVL=0
         สำหรับ 32-bit Protected Mode ปกติ

Flags = 0xA = 1010b → G=1 (4KB granularity), D/B=0, L=1 (64-bit!), AVL=0
         สำหรับ 64-bit Long Mode code segment

Flags = 0x0 = 0000b → G=0 (byte granularity), D/B=0 (16-bit)
         สำหรับ Real Mode หรือ 16-bit protected mode
```

---

## 8. GDT Register (GDTR)

CPU มี special register ชื่อ **GDTR** (GDT Register) ที่เก็บ:
- **Base**: Linear address (32-bit) หรือ virtual address (64-bit) ของ GDT
- **Limit**: ขนาด GDT ลบ 1 (16-bit)

```
GDTR Format:
┌──────────────────────────────────────────┬──────────────────┐
│           Base (32 or 64 bits)           │  Limit (16 bits) │
└──────────────────────────────────────────┴──────────────────┘
```

**Limit calculation:**
```
GDT มี N descriptors (แต่ละ 8 bytes)
Limit = N * 8 - 1

ตัวอย่าง: GDT มี 6 entries
Limit = 6 * 8 - 1 = 47 = 0x2F
```

### LGDT Instruction

```nasm
; Load GDT Register
; Operand: 6-byte memory location [limit (2), base (4)]

; สร้าง GDTR descriptor ใน memory
gdt_descriptor:
    dw gdt_end - gdt_start - 1    ; limit
    dd gdt_start                   ; base (32-bit)

; Load GDT
lgdt [gdt_descriptor]
```

### SGDT Instruction

```nasm
; Store GDT Register — อ่านค่าปัจจุบันของ GDTR
sgdt [my_gdtr_buffer]    ; บันทึก GDTR ลง memory
```

**Note:** LGDT เป็น privileged instruction ต้องรัน Ring 0  
SGDT สามารถรันได้จาก Ring 3 (แต่ OS modern มักปิดด้วย SMEP/UMIP)

---

## 9. LGDT ใน 16-bit Real Mode (Bootstrap)

เมื่อ boot เราอยู่ใน Real Mode (16-bit) แต่ต้องการเข้า Protected Mode

```nasm
bits 16
org 0x7C00        ; Bootloader load address

start:
    cli           ; Disable interrupts
    
    ; Load GDT
    lgdt [gdt_descriptor]
    
    ; Enable Protected Mode: set bit 0 (PE) ใน CR0
    mov eax, cr0
    or eax, 0x1
    mov cr0, eax
    
    ; Far jump เพื่อ flush pipeline และ load CS
    jmp 0x08:protected_mode_start    ; 0x08 = Kernel Code Selector
    
bits 32
protected_mode_start:
    ; ตอนนี้อยู่ใน 32-bit Protected Mode แล้ว
    mov ax, 0x10    ; Kernel Data Selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; ตั้งค่า stack
    mov esp, 0x90000
    
    ; ต่อไป...
    jmp $

; ==================== GDT ====================
align 8
gdt_start:
    ; Null Descriptor
    dq 0x0000000000000000

    ; Kernel Code Segment (Ring 0, 32-bit)
    ; Base=0, Limit=4GB, Access=0x9A, Flags=0xC
    dw 0xFFFF        ; Limit[15:0]
    dw 0x0000        ; Base[15:0]
    db 0x00          ; Base[23:16]
    db 0x9A          ; Access: P=1, DPL=0, S=1, Type=0xA (code, readable)
    db 0xCF          ; Flags=0xC + Limit[19:16]=0xF
    db 0x00          ; Base[31:24]

    ; Kernel Data Segment (Ring 0, 32-bit)
    ; Base=0, Limit=4GB, Access=0x92, Flags=0xC
    dw 0xFFFF        ; Limit[15:0]
    dw 0x0000        ; Base[15:0]
    db 0x00          ; Base[23:16]
    db 0x92          ; Access: P=1, DPL=0, S=1, Type=0x2 (data, writable)
    db 0xCF          ; Flags=0xC + Limit[19:16]=0xF
    db 0x00          ; Base[31:24]
gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1    ; Limit
    dd gdt_start                   ; Base

; Boot signature
times 510-($-$$) db 0
dw 0xAA55
```

---

## 10. Flat Memory Model

**Flat Model** หมายความว่า: code และ data segments มี base=0 และ limit=4GB  
ผลคือ linear address = effective address (ไม่มี segmentation จริง ๆ)

OS modern เช่น Linux, Windows ใช้ flat model เพราะง่ายกว่า  
และใช้ **Paging** แทนสำหรับ isolation ระหว่าง processes

```
Flat Model 32-bit:

Segment: Base    Limit    Access
Code:    0x00000000  0xFFFFF (x4KB=4GB)  Read/Execute
Data:    0x00000000  0xFFFFF (x4KB=4GB)  Read/Write

Linear Address = Segment Base + Offset = 0 + Offset = Offset
```

```nasm
; Complete Flat Model GDT (32-bit)
section .data
align 8

gdt32:
.null:
    dq 0x0000000000000000       ; Null descriptor

.kernel_code:                   ; Selector 0x08
    dw 0xFFFF                   ; Limit 0-15
    dw 0x0000                   ; Base 0-15
    db 0x00                     ; Base 16-23
    db 10011010b                ; P=1,DPL=0,S=1,Type=1010(code,readable)
    db 11001111b                ; G=1,D=1,L=0,AVL=0,Limit 16-19=0xF
    db 0x00                     ; Base 24-31

.kernel_data:                   ; Selector 0x10
    dw 0xFFFF                   ; Limit 0-15
    dw 0x0000                   ; Base 0-15
    db 0x00                     ; Base 16-23
    db 10010010b                ; P=1,DPL=0,S=1,Type=0010(data,writable)
    db 11001111b                ; G=1,B=1,L=0,AVL=0,Limit 16-19=0xF
    db 0x00                     ; Base 24-31

.user_code:                     ; Selector 0x1B (0x18 | RPL=3)
    dw 0xFFFF                   ; Limit 0-15
    dw 0x0000                   ; Base 0-15
    db 0x00                     ; Base 16-23
    db 11111010b                ; P=1,DPL=3,S=1,Type=1010(code,readable)
    db 11001111b                ; G=1,D=1,L=0,AVL=0,Limit 16-19=0xF
    db 0x00                     ; Base 24-31

.user_data:                     ; Selector 0x23 (0x20 | RPL=3)
    dw 0xFFFF                   ; Limit 0-15
    dw 0x0000                   ; Base 0-15
    db 0x00                     ; Base 16-23
    db 11110010b                ; P=1,DPL=3,S=1,Type=0010(data,writable)
    db 11001111b                ; G=1,B=1,L=0,AVL=0,Limit 16-19=0xF
    db 0x00                     ; Base 24-31

gdt32_end:

gdt32_descriptor:
    dw gdt32_end - gdt32 - 1   ; Limit
    dd gdt32                    ; Base (physical address)
```

---

## 11. 64-bit GDT (Long Mode)

ใน 64-bit Long Mode descriptors มีการเปลี่ยนแปลงสำคัญ:

1. **Base และ Limit ถูกละเว้น** สำหรับ CS, DS, ES, SS (เป็น flat 0..2^64 เสมอ)
2. **FS และ GS** ยังมี base ที่สามารถตั้งได้ผ่าน MSR (ใช้สำหรับ TLS)
3. **L bit = 1** บ่งบอกว่าเป็น 64-bit code segment
4. TSS descriptor ขยายเป็น **16 bytes** (2 entries)

```nasm
; 64-bit GDT
section .data
align 16

gdt64:
.null:
    dq 0x0000000000000000       ; Null descriptor

.kernel_code:                   ; Selector 0x08
    dw 0x0000                   ; Limit (ignored in 64-bit)
    dw 0x0000                   ; Base (ignored in 64-bit)
    db 0x00                     ; Base (ignored)
    db 10011010b                ; P=1,DPL=0,S=1,Type=1010
    db 00100000b                ; G=0,D=0,L=1(64-bit!),AVL=0,Limit=0
    db 0x00                     ; Base (ignored)

.kernel_data:                   ; Selector 0x10
    dw 0x0000
    dw 0x0000
    db 0x00
    db 10010010b                ; P=1,DPL=0,S=1,Type=0010
    db 00000000b                ; G=0,D=0,L=0,AVL=0
    db 0x00

.user_code32:                   ; Selector 0x18 (compat mode)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11111010b                ; P=1,DPL=3,S=1,Type=1010
    db 11001111b                ; G=1,D=1,L=0 (32-bit compat)
    db 0x00

.user_data:                     ; Selector 0x20 | RPL=3 = 0x23
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11110010b                ; P=1,DPL=3,S=1,Type=0010
    db 11001111b
    db 0x00

.user_code64:                   ; Selector 0x28 | RPL=3 = 0x2B
    dw 0x0000
    dw 0x0000
    db 0x00
    db 11111010b                ; P=1,DPL=3,S=1,Type=1010
    db 00100000b                ; L=1 (64-bit user code)
    db 0x00

; TSS Descriptor จะอยู่ที่นี่ (16 bytes)
.tss:
    dq 0                        ; ต้องตั้งค่า runtime
    dq 0                        ; Extension (high 64-bit of base)

gdt64_end:

gdt64_descriptor:
    dw gdt64_end - gdt64 - 1
    dq gdt64                    ; 64-bit base!
```

**ข้อสำคัญสำหรับ 64-bit:**
```nasm
; LGDT ใน 64-bit mode ใช้ 10-byte operand (2+8)
; ไม่ใช่ 6-byte เหมือน 32-bit
lgdt [gdt64_descriptor]

; Load segment registers หลัง LGDT
mov ax, 0x10        ; Kernel data selector
mov ds, ax
mov es, ax
mov fs, ax
mov gs, ax
mov ss, ax
; CS ต้อง load ด้วย far jump/return
```

---

## 12. Privilege Rings

x86 มี 4 privilege levels (Rings) แต่ OS ส่วนใหญ่ใช้แค่ Ring 0 และ Ring 3

```
Ring 0: Kernel Mode
  - สิทธิ์สูงสุด
  - สามารถใช้ privileged instructions ทั้งหมด
  - เข้าถึง hardware registers โดยตรง (CR0, CR3, etc.)
  - I/O ports

Ring 1, 2: ไม่ค่อยใช้
  - บางครั้งใช้สำหรับ device drivers หรือ hypervisor guests
  - Windows, Linux ข้ามไปเลย

Ring 3: User Mode
  - สิทธิ์ต่ำสุด
  - ไม่สามารถใช้ privileged instructions
  - ไม่สามารถเข้าถึง I/O ports โดยตรง (ถ้า IOPL < 3)
  - การ access memory ถูก control โดย page tables
```

### CPL, DPL, RPL

```
CPL (Current Privilege Level):
  - privilege level ของ code ที่กำลัง execute
  - เก็บใน bits 1:0 ของ CS register

DPL (Descriptor Privilege Level):
  - privilege level ที่ descriptor กำหนด
  - บอกว่าต้องมี privilege ระดับไหนถึงจะใช้ segment นี้ได้

RPL (Requested Privilege Level):
  - privilege level ที่ขอใน selector
  - bits 1:0 ของ segment selector
  - CPU ใช้ max(CPL, RPL) เปรียบเทียบกับ DPL
```

**กฎการ Access:**

Data Segment:
```
max(CPL, RPL) ≤ DPL
หรือ: ต้องมี privilege ≥ DPL (ตัวเลขน้อยกว่าหรือเท่ากัน)

ตัวอย่าง:
  CPL=0 (kernel) เข้า DPL=3 segment ได้ (0 ≤ 3)
  CPL=3 (user) เข้า DPL=0 segment ไม่ได้ (3 > 0 → #GP)
```

Code Segment (non-conforming):
```
CPL ต้องเท่ากับ DPL เท่านั้น
  ตัวอย่าง: จะ jump เข้า kernel code (DPL=0) จาก user (CPL=3) ไม่ได้โดยตรง
  ต้องผ่าน call gate หรือ syscall
```

---

## 13. Ring Transitions

### 13.1 SYSCALL/SYSRET (Fast System Call)

วิธีที่ใช้บ่อยที่สุดในระบบปฏิบัติการสมัยใหม่ (Linux, Windows x64)

```nasm
; SYSCALL ใช้ MSRs (Model Specific Registers)
; MSR_STAR (0xC0000081):
;   bits 63:48 = Sysret CS/SS selectors
;   bits 47:32 = Syscall CS/SS selectors
;
; MSR_LSTAR (0xC0000082): ที่อยู่ของ kernel syscall handler (64-bit)
; MSR_CSTAR (0xC0000083): ที่อยู่ (32-bit compat mode)
; MSR_SFMASK (0xC0000084): RFLAGS mask ตอน syscall

; Setup SYSCALL/SYSRET (ทำใน kernel initialization)
section .text
bits 64

setup_syscall:
    ; Enable SYSCALL/SYSRET: set SCE bit ใน EFER MSR
    mov ecx, 0xC0000080         ; IA32_EFER
    rdmsr
    or eax, (1 << 0)            ; SCE bit
    wrmsr
    
    ; Set STAR: Kernel CS=0x08, User CS ตอน sysret=0x1B (index 3 | RPL=3)
    mov ecx, 0xC0000081         ; MSR_STAR
    xor eax, eax
    mov edx, 0x00180008         ; bits 47:32=0x0008(kernel), bits 63:48=0x0018
    wrmsr
    
    ; Set LSTAR: syscall handler address
    mov ecx, 0xC0000082         ; MSR_LSTAR
    mov rax, syscall_handler
    mov rdx, rax
    shr rdx, 32
    wrmsr
    
    ; Set SFMASK: mask RFLAGS ตอน syscall (clear IF, DF)
    mov ecx, 0xC0000084         ; MSR_SFMASK
    mov eax, 0x00000200         ; Clear IF (interrupt flag)
    xor edx, edx
    wrmsr
    
    ret

; Syscall handler
syscall_handler:
    ; เมื่อเข้ามา:
    ; RCX = saved RIP (return address)
    ; R11 = saved RFLAGS
    ; RAX = syscall number
    ; RDI, RSI, RDX, R10, R8, R9 = arguments
    ; CS เปลี่ยนเป็น kernel code segment
    ; SS เปลี่ยนเป็น kernel data segment
    ; Stack ยัง user stack อยู่!
    
    ; ต้อง swap stack ทันที
    swapgs                      ; swap GS.base ระหว่าง kernel/user
    mov [gs:user_rsp], rsp      ; บันทึก user RSP
    mov rsp, [gs:kernel_rsp]    ; load kernel stack
    
    ; บันทึก registers
    push rcx                    ; saved RIP
    push r11                    ; saved RFLAGS
    
    ; dispatch syscall
    cmp rax, MAX_SYSCALL
    jae syscall_invalid
    call [syscall_table + rax*8]
    
    ; คืนค่า
    pop r11
    pop rcx
    
    ; คืน stack
    mov [gs:kernel_rsp], rsp
    mov rsp, [gs:user_rsp]
    swapgs
    
    sysretq                     ; return to user (64-bit)
    
syscall_invalid:
    mov rax, -1                 ; -ENOSYS
    jmp .return

; User code เรียก syscall
user_syscall_example:
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    mov rsi, msg
    mov rdx, msg_len
    syscall
    ret
```

### 13.2 INT/IRET (Legacy System Call / Interrupts)

```nasm
; INT เปลี่ยน privilege โดยผ่าน IDT (Interrupt Descriptor Table)
; ใช้ใน 32-bit Linux (int 0x80) และ exception handling

; ตอน INT เกิดขึ้น CPU ทำ:
; 1. ตรวจสอบ privilege: CPL ≤ DPL ของ IDT gate
; 2. Stack switch ถ้า CPL เปลี่ยน (ใช้ค่าจาก TSS)
; 3. Push SS, ESP (ถ้า privilege change), EFLAGS, CS, EIP
; 4. Load new CS:EIP จาก IDT entry
; 5. Clear IF ถ้าเป็น interrupt gate

; IRET คืนค่าทั้งหมด
iret_example:
    ; Stack ณ ตอน return (32-bit, privilege change):
    ; [ESP+16] SS    (user)
    ; [ESP+12] ESP   (user)
    ; [ESP+8]  EFLAGS
    ; [ESP+4]  CS
    ; [ESP+0]  EIP
    iret
```

### 13.3 Call Gates

```nasm
; Call Gate ใน GDT/LDT เป็นอีกวิธีหนึ่ง (เก่า, ไม่ค่อยใช้แล้ว)

; Call Gate Descriptor (8 bytes, S=0, Type=0100 for 32-bit):
; Bits 47:32 = Offset[15:0]
; Bits 63:48 = Offset[31:16]
; Bits 31:16 = Segment Selector
; Bits 12:8  = Parameter count (สำหรับ copy parameters จาก user stack)
; Bit  15:14 = DPL
; Bit  15    = P

; การใช้งาน:
; far call [call_gate_selector]:offset_ignored
; ถ้า DPL ของ gate ≥ CPL ก็สามารถเรียกได้
; CPU จะ switch ไป segment+offset ที่ gate กำหนด
```

---

## 14. Task State Segment (TSS)

TSS เป็น data structure ที่ Intel กำหนดสำหรับ hardware multitasking  
แม้ modern OS ไม่ใช้ hardware task switching แต่ยังต้องการ TSS สำหรับ:
1. **Stack switching**: เมื่อเกิด interrupt จาก Ring 3 → Ring 0 ต้องใช้ stack จาก TSS
2. **RSP0**: kernel stack pointer ที่ CPU จะ load เมื่อ interrupt เกิดใน user mode (64-bit)
3. **IST (Interrupt Stack Table)**: สำหรับ double fault, NMI ที่ต้องการ known-good stack

### TSS Structure (32-bit)

```nasm
struc tss32
    .prev_task    resw 1    ; Previous task link (hardware multitask)
    .reserved0    resw 1
    .esp0         resd 1    ; Ring 0 ESP (kernel stack)
    .ss0          resw 1    ; Ring 0 SS
    .reserved1    resw 1
    .esp1         resd 1    ; Ring 1 ESP
    .ss1          resw 1
    .reserved2    resw 1
    .esp2         resd 1    ; Ring 2 ESP
    .ss2          resw 1
    .reserved3    resw 1
    .cr3          resd 1    ; Page Directory Base
    .eip          resd 1    ; เมื่อใช้ hardware task switching
    .eflags       resd 1
    .eax          resd 1
    .ecx          resd 1
    .edx          resd 1
    .ebx          resd 1
    .esp          resd 1
    .ebp          resd 1
    .esi          resd 1
    .edi          resd 1
    .es           resw 1
    .reserved4    resw 1
    .cs           resw 1
    .reserved5    resw 1
    .ss           resw 1
    .reserved6    resw 1
    .ds           resw 1
    .reserved7    resw 1
    .fs           resw 1
    .reserved8    resw 1
    .gs           resw 1
    .reserved9    resw 1
    .ldt          resw 1    ; LDT Selector
    .reserved10   resw 1
    .debug_flag   resw 1    ; T bit for debug trap
    .iomap_base   resw 1    ; I/O Permission Bitmap offset
endstruc                    ; Total: 104 bytes minimum
```

### TSS Structure (64-bit)

```nasm
struc tss64
    .reserved0    resd 1    ; (was Prev Task Link in 32-bit)
    .rsp0         resq 1    ; Ring 0 Stack Pointer ← สำคัญมาก!
    .rsp1         resq 1    ; Ring 1 RSP
    .rsp2         resq 1    ; Ring 2 RSP
    .reserved1    resq 1
    .ist1         resq 1    ; Interrupt Stack Table 1 (double fault)
    .ist2         resq 1    ; IST 2
    .ist3         resq 1    ; IST 3
    .ist4         resq 1    ; IST 4
    .ist5         resq 1    ; IST 5
    .ist6         resq 1    ; IST 6
    .ist7         resq 1    ; IST 7
    .reserved2    resq 1
    .reserved3    resw 1
    .iomap_base   resw 1    ; I/O Permission Bitmap
endstruc                    ; Size: 104 bytes
```

### TSS ใน GDT

```nasm
; TSS Descriptor ใน GDT (32-bit): 8 bytes, System Descriptor (S=0)
; Type = 1001b (32-bit TSS, Available)
; Type = 1011b (32-bit TSS, Busy)

; สร้าง TSS descriptor:
%macro TSS_DESCRIPTOR32 1       ; %1 = address of TSS
    dw (tss32_size - 1)         ; Limit[15:0]
    dw (%1 & 0xFFFF)            ; Base[15:0]
    db ((%1 >> 16) & 0xFF)      ; Base[23:16]
    db 10001001b                ; P=1,DPL=0,S=0,Type=1001(TSS32,Avail)
    db (0 | ((tss32_size-1)>>16)&0xF)  ; Flags + Limit[19:16]
    db ((%1 >> 24) & 0xFF)      ; Base[31:24]
%endmacro

; TSS Descriptor ใน GDT (64-bit): 16 bytes! (2 consecutive GDT entries)
; เพราะ base address ต้องเป็น 64-bit
%macro TSS_DESCRIPTOR64 1       ; %1 = address of TSS
    dw (tss64_size - 1)         ; Limit[15:0]
    dw (%1 & 0xFFFF)            ; Base[15:0]
    db ((%1 >> 16) & 0xFF)      ; Base[23:16]
    db 10001001b                ; P=1,DPL=0,S=0,Type=1001(TSS64,Avail)
    db (0 | ((tss64_size-1)>>16)&0xF)
    db ((%1 >> 24) & 0xFF)      ; Base[31:24]
    dd ((%1 >> 32) & 0xFFFFFFFF); Base[63:32] — 32-bit upper!
    dd 0x00000000               ; Reserved + must be 0
%endmacro
```

### Loading TSS (LTR Instruction)

```nasm
; หลังจาก load GDT และอยู่ใน Protected/Long Mode
; ต้อง load TR (Task Register) ด้วย LTR

; LTR ใช้ selector ชี้ไปที่ TSS descriptor ใน GDT
ltr ax          ; ax = TSS selector เช่น 0x28

; ข้อสังเกต:
; - LTR เป็น privileged instruction (Ring 0 เท่านั้น)
; - LTR set TSS เป็น "Busy" ใน GDT
; - ทำเพียงครั้งเดียวตอน init เพียงพอ
; - ถ้าต้องการ per-CPU TSS ก็ทำแต่ละ CPU แยกกัน

; ตั้งค่า RSP0 ใน TSS ตอน context switch:
; เมื่อ switch จาก process A ไป process B
; ต้องอัพเดต TSS.rsp0 ให้ชี้ kernel stack ของ process B
mov [tss + tss64.rsp0], rsp_kernel_b
```

---

## 15. Complete GDT Setup สำหรับ 64-bit OS Kernel

```nasm
; ============================================================
; gdt64_complete.asm — Complete 64-bit GDT with TSS
; Build: nasm -f elf64 gdt64_complete.asm
; ============================================================
bits 64
section .data

; ==================== Selectors ====================
KERNEL_CODE_SEL  equ 0x08
KERNEL_DATA_SEL  equ 0x10
USER_CODE32_SEL  equ 0x18    ; 32-bit compatibility mode
USER_DATA_SEL    equ 0x20
USER_CODE64_SEL  equ 0x28    ; 64-bit user code
TSS_SEL          equ 0x30    ; TSS descriptor (16 bytes, takes 2 slots)

; ==================== TSS ====================
align 16
tss:
.reserved0:     dd 0
.rsp0:          dq 0        ; Kernel stack (filled at runtime)
.rsp1:          dq 0
.rsp2:          dq 0
.reserved1:     dq 0
.ist1:          dq 0        ; Double fault stack
.ist2:          dq 0
.ist3:          dq 0
.ist4:          dq 0
.ist5:          dq 0
.ist6:          dq 0
.ist7:          dq 0
.reserved2:     dq 0
.reserved3:     dw 0
.iomap_base:    dw (tss_end - tss)   ; No I/O bitmap → set past end
tss_end:
TSS_SIZE        equ tss_end - tss

; ==================== GDT ====================
align 16
gdt64:
.null:                              ; 0x00 — Null
    dq 0

.kernel_code:                       ; 0x08 — Kernel Code 64-bit
    dw 0x0000                       ; Limit (ignored)
    dw 0x0000                       ; Base (ignored)
    db 0x00                         ; Base (ignored)
    db 10011010b                    ; P=1,DPL=0,S=1,Type=1010
    db 00100000b                    ; G=0,D=0,L=1,AVL=0
    db 0x00                         ; Base (ignored)

.kernel_data:                       ; 0x10 — Kernel Data
    dw 0x0000
    dw 0x0000
    db 0x00
    db 10010010b                    ; P=1,DPL=0,S=1,Type=0010
    db 00000000b
    db 0x00

.user_code32:                       ; 0x18 — User Code 32-bit (compat)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11111010b                    ; P=1,DPL=3,S=1,Type=1010
    db 11001111b                    ; G=1,D=1,L=0
    db 0x00

.user_data:                         ; 0x20 — User Data (selector: 0x23)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11110010b                    ; P=1,DPL=3,S=1,Type=0010
    db 11001111b
    db 0x00

.user_code64:                       ; 0x28 — User Code 64-bit (selector: 0x2B)
    dw 0x0000
    dw 0x0000
    db 0x00
    db 11111010b                    ; P=1,DPL=3,S=1,Type=1010
    db 00100000b                    ; L=1!
    db 0x00

.tss_low:                           ; 0x30 — TSS Low (16-byte descriptor)
    dw TSS_SIZE - 1                 ; Limit[15:0]
    dw 0x0000                       ; Base[15:0] (filled at runtime)
    db 0x00                         ; Base[23:16] (filled at runtime)
    db 10001001b                    ; P=1,DPL=0,S=0,Type=1001(TSS64 Avail)
    db 0x00                         ; Flags + Limit[19:16]
    db 0x00                         ; Base[31:24] (filled at runtime)

.tss_high:                          ; 0x38 — TSS High
    dq 0x0000000000000000           ; Base[63:32] (filled at runtime)

gdt64_end:

gdtr64:
    dw gdt64_end - gdt64 - 1       ; Limit
    dq gdt64                        ; Base

; ==================== Code ====================
section .text

; gdt_init — Initialize and load 64-bit GDT with TSS
; Call once during kernel startup
global gdt_init
gdt_init:
    ; Fill TSS descriptor in GDT with address of TSS
    mov rax, tss
    
    ; TSS base[15:0] → gdt64.tss_low + 2
    mov word [gdt64.tss_low + 2], ax
    ; TSS base[23:16] → gdt64.tss_low + 4
    shr rax, 16
    mov byte [gdt64.tss_low + 4], al
    ; TSS base[31:24] → gdt64.tss_low + 7
    shr rax, 8
    mov byte [gdt64.tss_low + 7], al
    ; TSS base[63:32] → gdt64.tss_high + 0
    shr rax, 8
    mov dword [gdt64.tss_high + 0], eax
    
    ; Load GDT
    lgdt [gdtr64]
    
    ; Reload segment registers
    mov ax, KERNEL_DATA_SEL
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; Reload CS via far return
    pop rdi             ; save return address
    push qword KERNEL_CODE_SEL
    push rdi
    retfq               ; far return → loads CS and jumps
    
    ; (Execution continues here with new CS)
    ; Load TSS
    mov ax, TSS_SEL
    ltr ax
    
    ret

; tss_set_kernel_stack — Set RSP0 ใน TSS (call on every context switch)
; Parameters: rdi = kernel stack pointer (RSP0)
global tss_set_kernel_stack
tss_set_kernel_stack:
    mov [tss + 4], rdi      ; TSS.rsp0 ที่ offset 4
    ret

; gdt_load_user_segments — สำหรับ return ไป user mode
; ใช้ก่อน IRETQ/SYSRETQ
global gdt_load_user_segs
gdt_load_user_segs:
    mov ax, (USER_DATA_SEL | 3)     ; RPL=3
    mov ds, ax
    mov es, ax
    ; FS, GS ใช้สำหรับ TLS — ตั้งแยก
    ret
```

---

## 16. GDT Loading Function (ใน C Style Assembly)

```nasm
; ============================================================
; gdt_functions.asm
; Functions สำหรับจัดการ GDT จาก C code
; ============================================================
bits 64

section .text

; void gdt_flush(uint64_t gdtr_ptr)
; Load GDTR จาก pointer ที่ให้มา
global gdt_flush
gdt_flush:
    lgdt [rdi]          ; rdi = pointer to GDTR structure
    
    ; Reload segment registers
    mov ax, 0x10        ; Kernel data selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; Far return เพื่อ reload CS
    lea rax, [rel .done]
    push 0x08           ; Kernel code selector
    push rax
    retfq

.done:
    ret

; void tss_flush(uint16_t tss_selector)
global tss_flush
tss_flush:
    mov ax, di          ; tss selector ใน low 16 bits ของ rdi
    ltr ax
    ret

; uint16_t get_cs(void) — อ่านค่า CS ปัจจุบัน
global get_cs
get_cs:
    xor rax, rax
    mov ax, cs
    ret

; uint16_t get_ds(void)
global get_ds
get_ds:
    xor rax, rax
    mov ax, ds
    ret

; void set_fs_base(uint64_t base) — ตั้ง FS.base ผ่าน MSR
global set_fs_base
set_fs_base:
    mov ecx, 0xC0000100     ; FS_BASE MSR
    mov eax, edi
    mov edx, edi
    shr rdx, 32
    wrmsr
    ret

; void set_gs_base(uint64_t base) — ตั้ง GS.base ผ่าน MSR
global set_gs_base
set_gs_base:
    mov ecx, 0xC0000101     ; GS_BASE MSR
    mov eax, edi
    mov edx, edi
    shr rdx, 32
    wrmsr
    ret

; void swapgs_and_return(void) — สำหรับ syscall return
global swapgs_and_return
swapgs_and_return:
    swapgs
    ret
```

---

## 17. ตัวอย่าง: Bootloader ที่สมบูรณ์พร้อม Protected Mode

```nasm
; ============================================================
; bootloader.asm — Simple bootloader เข้าสู่ Protected Mode
; Build: nasm -f bin bootloader.asm -o bootloader.bin
; Test:  qemu-system-x86_64 -drive format=raw,file=bootloader.bin
; ============================================================
bits 16
org 0x7C00

section .text

_start:
    ; ปิด interrupts
    cli
    
    ; ตั้ง segment registers
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00          ; Stack ไว้ด้านล่าง bootloader
    
    ; แสดงข้อความ "Entering PM..."
    mov si, msg_pm
    call print_string
    
    ; Load GDT
    lgdt [gdt32_descriptor]
    
    ; Enable A20 line (ถ้าต้องการ > 1MB)
    in al, 0x92
    or al, 0x02
    out 0x92, al
    
    ; Enable Protected Mode
    mov eax, cr0
    or eax, 0x1
    mov cr0, eax
    
    ; Far jump ไป 32-bit Protected Mode
    jmp 0x08:pm32_entry

; ===== Real Mode Functions =====
print_string:
    lodsb
    test al, al
    jz .done
    mov ah, 0x0E
    int 0x10
    jmp print_string
.done:
    ret

; ===== Protected Mode Entry =====
bits 32
pm32_entry:
    ; Load data segments
    mov ax, 0x10            ; Kernel Data Selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; Setup stack
    mov esp, 0x00090000
    
    ; เขียนข้อความ "OK" ที่ video memory
    mov dword [0xB8000], 0x07200720  ; " " " "
    mov dword [0xB8000], 0x074F074B  ; "OK"
    
    ; Halt
    hlt
    jmp $

; ===== Data =====
bits 16

msg_pm: db "Entering Protected Mode...", 0x0D, 0x0A, 0

; ===== GDT =====
align 8
gdt32:
.null:
    dq 0x0000000000000000       ; Null

.kernel_code:                   ; Selector 0x08
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0x9A                     ; P=1,DPL=0,S=1,Type=0xA
    db 0xCF                     ; G=1,D=1,L=0 + Limit[19:16]=0xF
    db 0x00

.kernel_data:                   ; Selector 0x10
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0x92                     ; P=1,DPL=0,S=1,Type=0x2
    db 0xCF
    db 0x00

.user_code:                     ; Selector 0x1B (0x18|3)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0xFA                     ; P=1,DPL=3,S=1,Type=0xA
    db 0xCF
    db 0x00

.user_data:                     ; Selector 0x23 (0x20|3)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0xF2                     ; P=1,DPL=3,S=1,Type=0x2
    db 0xCF
    db 0x00
gdt32_end:

gdt32_descriptor:
    dw gdt32_end - gdt32 - 1
    dd gdt32

; ===== Boot Signature =====
times 510-($-$$) db 0
dw 0xAA55
```

---

## 18. ตัวอย่าง: Long Mode Setup (32-bit → 64-bit)

```nasm
; ============================================================
; long_mode_setup.asm
; เปลี่ยนจาก Protected Mode เป็น Long Mode
; ============================================================
bits 32

; สมมติว่าอยู่ใน 32-bit Protected Mode แล้ว

enter_long_mode:
    ; 1. ตรวจสอบ CPUID รองรับ Long Mode ไหม
    mov eax, 0x80000001
    cpuid
    test edx, (1 << 29)     ; LM bit
    jz no_long_mode
    
    ; 2. ตั้งค่า Page Tables (Identity mapping)
    call setup_page_tables
    
    ; 3. Load PML4 address ใน CR3
    mov eax, pml4_table
    mov cr3, eax
    
    ; 4. Enable PAE (Physical Address Extension)
    mov eax, cr4
    or eax, (1 << 5)        ; PAE bit
    mov cr4, eax
    
    ; 5. Enable Long Mode ใน EFER MSR
    mov ecx, 0xC0000080     ; IA32_EFER
    rdmsr
    or eax, (1 << 8)        ; LME (Long Mode Enable)
    wrmsr
    
    ; 6. Enable Paging (เปิด CR0.PG)
    mov eax, cr0
    or eax, (1 << 31)       ; PG bit
    ; พร้อมกับ PE bit ที่เปิดแล้ว
    mov cr0, eax
    
    ; ตอนนี้อยู่ใน "Compatibility Mode" (ยัง 32-bit)
    ; ต้อง load 64-bit GDT และ far jump เพื่อเข้า Long Mode จริง ๆ
    
    lgdt [gdtr64]
    jmp 0x08:long_mode_entry    ; 0x08 = 64-bit code segment

bits 64
long_mode_entry:
    ; ตอนนี้อยู่ใน 64-bit Long Mode แล้ว!
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; ต่อไป setup IDT, TSS, หรือ jump ไป kernel main
    jmp kernel_main

no_long_mode:
    hlt
    jmp $

; ===== Simple Identity Map Page Tables =====
bits 32
setup_page_tables:
    ; PML4 → PDPT → PD → PT
    ; Identity map 2MB แรก (ใช้ 2MB huge pages)
    
    ; PML4[0] → PDPT
    mov eax, pdpt_table
    or eax, 0x3              ; Present + Writable
    mov [pml4_table], eax
    mov dword [pml4_table + 4], 0
    
    ; PDPT[0] → PD
    mov eax, pd_table
    or eax, 0x3
    mov [pdpt_table], eax
    mov dword [pdpt_table + 4], 0
    
    ; PD[0] → 2MB huge page at 0x0
    mov dword [pd_table], 0x0 | 0x83   ; Present + Writable + Huge (PS bit)
    mov dword [pd_table + 4], 0
    
    ret

section .bss
align 4096
pml4_table: resb 4096
pdpt_table:  resb 4096
pd_table:    resb 4096

section .data
align 16
gdtr64:
    dw gdt64_end - gdt64 - 1
    dq gdt64

gdt64:
.null:  dq 0
.code:  ; 64-bit code
    dw 0, 0
    db 0
    db 0x9A
    db 0x20     ; L=1
    db 0
.data:
    dw 0, 0
    db 0
    db 0x92
    db 0
    db 0
gdt64_end:
```

---

## 19. Debugging GDT ปัญหาที่พบบ่อย

### 19.1 General Protection Fault (#GP, Vector 13)

```
สาเหตุที่พบบ่อย:
1. Load selector ที่ไม่ valid (เกิน limit ของ GDT)
2. Load null selector เข้า DS/ES/FS/GS แล้ว dereference
3. CPL > DPL ของ code segment ที่ต้องการ jump
4. CS หรือ SS ที่ไม่ compatible กับ current privilege level
5. Misaligned GDTR (ควร align 8)
```

### 19.2 Stack Fault (#SS, Vector 12)

```
สาเหตุ:
1. SS ไม่ match privilege level
2. Stack pointer out of segment limit
3. ตอน interrupt: TSS.RSP0 ไม่ valid
```

### 19.3 Invalid TSS (#TS, Vector 10)

```
สาเหตุ:
1. TSS descriptor ไม่ถูกต้อง
2. TSS ขนาดน้อยเกินไป
3. LTR ยังไม่ได้ทำก่อน ให้ interrupt เกิดขึ้น
```

### 19.4 ใช้ QEMU + GDB Debug

```bash
# เปิด QEMU พร้อม GDB server
qemu-system-x86_64 -drive format=raw,file=boot.bin \
    -s -S -nographic

# ใน terminal อีกอัน:
gdb
(gdb) target remote :1234
(gdb) set arch i386:x86-64
(gdb) b *0x7C00          # breakpoint ที่ bootloader start
(gdb) c

# อ่าน GDT ปัจจุบัน:
(gdb) monitor info registers    # ดู GDTR
(gdb) monitor xp/8bx 0x<gdt_addr>  # ดู GDT bytes
```

### 19.5 ตรวจสอบ Descriptor ด้วย Python

```python
#!/usr/bin/env python3
# decode_gdt.py — Decode GDT descriptor

def decode_descriptor(low64):
    """Decode an 8-byte GDT descriptor"""
    limit_lo = low64 & 0xFFFF
    base_lo  = (low64 >> 16) & 0xFFFF
    base_mid = (low64 >> 32) & 0xFF
    access   = (low64 >> 40) & 0xFF
    flags_limit_hi = (low64 >> 48) & 0xFF
    base_hi  = (low64 >> 56) & 0xFF
    
    limit_hi = (flags_limit_hi & 0x0F)
    flags    = (flags_limit_hi >> 4) & 0x0F
    
    limit = limit_lo | (limit_hi << 16)
    base  = base_lo | (base_mid << 16) | (base_hi << 24)
    
    P   = (access >> 7) & 1
    DPL = (access >> 5) & 3
    S   = (access >> 4) & 1
    typ = access & 0xF
    
    G   = (flags >> 3) & 1
    DB  = (flags >> 2) & 1
    L   = (flags >> 1) & 1
    AVL = flags & 1
    
    print(f"Base:   0x{base:08X}")
    print(f"Limit:  0x{limit:05X} ({'4KB pages' if G else 'bytes'})")
    print(f"Access: P={P} DPL={DPL} S={S} Type=0x{typ:X}")
    print(f"Flags:  G={G} D/B={DB} L={L} AVL={AVL}")
    
    if S == 1:
        if typ & 8:  # Code
            conforming = (typ >> 2) & 1
            readable   = (typ >> 1) & 1
            print(f"  Code Segment: Conforming={conforming} Readable={readable}")
        else:  # Data
            expand_down = (typ >> 2) & 1
            writable    = (typ >> 1) & 1
            print(f"  Data Segment: ExpandDown={expand_down} Writable={writable}")
    else:
        print(f"  System Descriptor: Type=0x{typ:X}")

# ตัวอย่าง: Kernel Code descriptor (0x9A bytes)
print("=== Kernel Code Segment ===")
# dw 0xFFFF, dw 0x0000, db 0x00, db 0x9A, db 0xCF, db 0x00
decode_descriptor(0x00CF9A000000FFFF)

print("\n=== Kernel Data Segment ===")
decode_descriptor(0x00CF92000000FFFF)

print("\n=== 64-bit Code Segment ===")
decode_descriptor(0x00209A0000000000)
```

---

## 20. Makefile สำหรับ Project

```makefile
# Makefile สำหรับ OS kernel with GDT
# ใช้กับ NASM 2.x + LD + QEMU

# Tools
NASM    := nasm
LD      := ld
QEMU    := qemu-system-x86_64

# Flags
NASMFLAGS_16  := -f bin
NASMFLAGS_32  := -f elf32
NASMFLAGS_64  := -f elf64 -g -F dwarf
LDFLAGS_32    := -m elf_i386 -T linker32.ld
LDFLAGS_64    := -m elf_x86_64 -T linker64.ld

# Targets
all: bootloader.bin kernel64.elf os.img

# 16-bit Bootloader
bootloader.bin: bootloader.asm gdt32.inc
	$(NASM) $(NASMFLAGS_16) $< -o $@

# 32-bit Protected Mode test
kernel32.o: kernel32.asm
	$(NASM) $(NASMFLAGS_32) $< -o $@

kernel32.elf: kernel32.o
	$(LD) $(LDFLAGS_32) $< -o $@

# 64-bit Long Mode kernel
gdt64.o: gdt64_complete.asm
	$(NASM) $(NASMFLAGS_64) $< -o $@

gdt_functions.o: gdt_functions.asm
	$(NASM) $(NASMFLAGS_64) $< -o $@

kernel64.elf: gdt64.o gdt_functions.o kernel_main.o
	$(LD) $(LDFLAGS_64) $^ -o $@

# Combined OS image (bootloader + kernel)
os.img: bootloader.bin kernel64.elf
	dd if=/dev/zero of=$@ bs=512 count=2880
	dd if=bootloader.bin of=$@ conv=notrunc
	# Copy kernel after bootloader
	dd if=kernel64.elf of=$@ bs=512 seek=1 conv=notrunc

# Test with QEMU
run: os.img
	$(QEMU) \
		-drive format=raw,file=$< \
		-m 256M \
		-serial stdio \
		-no-reboot \
		-no-shutdown

# Debug with GDB
debug: os.img
	$(QEMU) \
		-drive format=raw,file=$< \
		-m 256M \
		-s -S \
		-serial stdio \
		-no-reboot \
		-no-shutdown &
	sleep 0.5
	gdb -ex "target remote :1234" \
	    -ex "set arch i386:x86-64" \
	    -ex "symbol-file kernel64.elf"

# Test bootloader only
run-boot: bootloader.bin
	$(QEMU) -drive format=raw,file=$< -m 64M -no-reboot

# Clean
clean:
	rm -f *.o *.bin *.elf *.img

.PHONY: all run debug run-boot clean
```

---

## 21. QEMU Test Commands

```bash
# ============================================================
# คำสั่ง QEMU สำหรับทดสอบ GDT/Protected Mode
# ============================================================

# 1. รัน bootloader ธรรมดา
qemu-system-x86_64 -drive format=raw,file=bootloader.bin -nographic

# 2. รันพร้อม serial output
qemu-system-x86_64 \
    -drive format=raw,file=bootloader.bin \
    -serial stdio \
    -no-reboot \
    -no-shutdown

# 3. รันพร้อม debug (GDB)
qemu-system-x86_64 \
    -drive format=raw,file=bootloader.bin \
    -s -S \
    -m 256M \
    -serial stdio

# เปิด GDB อีก terminal:
gdb
(gdb) target remote localhost:1234
(gdb) set architecture i8086     # เริ่มต้นเป็น 16-bit
(gdb) break *0x7C00
(gdb) continue
# หลังจาก Protected Mode:
(gdb) set architecture i386:intel
(gdb) break *0x100000            # kernel entry

# 4. รัน kernel ELF โดยตรง (ถ้าใช้ multiboot)
qemu-system-x86_64 -kernel kernel64.elf -nographic

# 5. ดู log ของ CPU exceptions
qemu-system-x86_64 \
    -drive format=raw,file=bootloader.bin \
    -d int,cpu_reset \
    -no-reboot

# 6. QEMU Monitor commands (กด Ctrl+Alt+2 เพื่อเข้า monitor)
# info registers    — ดู CPU registers รวม GDTR, IDTR, TR
# info mem          — ดู memory map
# xp /8gx 0x1000   — ดู memory (8 quadwords ที่ address 0x1000)
# x /8i 0x7C00     — disassemble 8 instructions ที่ 0x7C00

# 7. Test 64-bit kernel
qemu-system-x86_64 \
    -drive format=raw,file=os.img \
    -m 512M \
    -cpu qemu64 \
    -serial stdio \
    -no-reboot \
    -no-shutdown \
    -d int,cpu_reset 2>&1 | grep -v "check_exception"
```

---

## 22. GDT สำหรับ SMP (Symmetric Multi-Processing)

ระบบที่มีหลาย CPU cores แต่ละ core ต้องการ TSS แยกกัน  
แต่ GDT สามารถ share กันได้ (read-only หลัง init)

```nasm
; ============================================================
; smp_gdt.asm — GDT Setup สำหรับ SMP
; ============================================================
bits 64

; Constants
MAX_CPUS        equ 8
TSS_SIZE        equ 104         ; 64-bit TSS size

section .data
align 16

; Per-CPU TSS array
tss_array:
    times (TSS_SIZE * MAX_CPUS) db 0

; Per-CPU kernel stacks (16KB each)
KERNEL_STACK_SIZE equ 0x4000
kernel_stacks:
    times (KERNEL_STACK_SIZE * MAX_CPUS) db 0

section .text

; gdt_init_cpu(int cpu_id)
; เรียกจากแต่ละ CPU core ตอน startup
global gdt_init_cpu
gdt_init_cpu:
    ; rdi = cpu_id
    
    ; คำนวณ address ของ TSS สำหรับ CPU นี้
    mov rax, rdi
    mov rcx, TSS_SIZE
    mul rcx
    add rax, tss_array      ; rax = &tss_array[cpu_id]
    
    ; คำนวณ kernel stack สำหรับ CPU นี้
    mov rbx, rdi
    mov rcx, KERNEL_STACK_SIZE
    imul rbx, rcx
    add rbx, kernel_stacks
    add rbx, KERNEL_STACK_SIZE  ; rbx = stack top

    ; ตั้งค่า TSS.RSP0
    mov [rax + 4], rbx      ; tss.rsp0 = kernel stack top
    
    ; อัพเดต TSS descriptor ใน GDT (ต้อง lock!)
    ; ใน real SMP ต้องใช้ per-CPU GDT หรือ lock
    call update_tss_descriptor  ; rdi=cpu_id, rax=tss_addr
    
    ; Load TR
    ; แต่ละ CPU ต้องการ TSS selector ต่างกัน
    ; หรือใช้ per-CPU GDT
    mov cx, TSS_SEL
    ltr cx
    
    ret

; ตัวเลือกที่ดีกว่า: Per-CPU GDT
; แต่ละ core มี GDT copy ของตัวเอง
; ต้องการ memory: GDT_SIZE * MAX_CPUS bytes
```

---

## 23. สรุป Access Byte และ Flags ทั้งหมด

```
=== Access Byte Cheatsheet ===

Value | Binary    | Meaning
------+-----------+------------------------------------------------
0x9A  | 1001 1010 | Kernel Code:  P=1, DPL=0, S=1, Type=0xA (exec/read)
0x92  | 1001 0010 | Kernel Data:  P=1, DPL=0, S=1, Type=0x2 (read/write)
0x96  | 1001 0110 | Kernel Stack: P=1, DPL=0, S=1, Type=0x6 (expand-down,write)
0xFA  | 1111 1010 | User Code:    P=1, DPL=3, S=1, Type=0xA
0xF2  | 1111 0010 | User Data:    P=1, DPL=3, S=1, Type=0x2
0x89  | 1000 1001 | TSS (Avail):  P=1, DPL=0, S=0, Type=0x9
0x8B  | 1000 1011 | TSS (Busy):   P=1, DPL=0, S=0, Type=0xB
0x85  | 1000 0101 | Task Gate:    P=1, DPL=0, S=0, Type=0x5
0x8C  | 1000 1100 | Call Gate 16b:P=1, DPL=0, S=0, Type=0xC
0x8E  | 1000 1110 | Intr Gate 32b:P=1, DPL=0, S=0, Type=0xE
0x8F  | 1000 1111 | Trap Gate 32b:P=1, DPL=0, S=0, Type=0xF

=== Flags Nibble Cheatsheet ===

Value | Binary | Meaning
------+--------+------------------------------------------
0xC   | 1100   | 32-bit segment (G=1/4KB, D=1/32-bit)
0xA   | 1010   | 64-bit code (G=1/4KB, L=1/long mode)
0x8   | 1000   | 64-bit data (G=1/4KB, D=0, L=0)
0x0   | 0000   | 16-bit byte-granular segment
0x4   | 0100   | 16-bit with 32-bit operation size

=== ค่าที่ใช้บ่อยสุด (Byte 6 = Flags<<4 | Limit[19:16]) ===
0xCF = 0xC<<4 | 0xF = G=1,D=1,L=0 + Limit[19:16]=F → 32-bit flat
0x20 = 0x2<<4 | 0x0 = G=0,D=0,L=1 + Limit[19:16]=0 → 64-bit code
0xAF = 0xA<<4 | 0xF = G=1,D=0,L=1 + Limit[19:16]=F → 64-bit (some uses)
```

---

## 24. Code Example: Verify GDT ด้วย ASSERT

```nasm
; ============================================================
; gdt_verify.asm — ตรวจสอบ GDT ตอน compile time
; ============================================================

; NASM compile-time assertions
%if (gdt_end - gdt_start) % 8 != 0
    %error "GDT size must be multiple of 8 bytes"
%endif

%if (gdt_end - gdt_start) > (8192 * 8)
    %error "GDT too large (max 8192 entries)"
%endif

; ตรวจสอบ alignment
%if gdt_start % 8 != 0
    %warning "GDT should be aligned to 8 bytes for performance"
%endif

; Runtime verification (ใน C style)
section .text
bits 64

; int verify_gdt_descriptor(uint64_t *descriptor, int expected_dpl)
; Returns 0 on success, -1 on failure
global verify_gdt_descriptor
verify_gdt_descriptor:
    ; rdi = pointer to 8-byte descriptor
    ; rsi = expected DPL
    
    mov rax, [rdi]
    
    ; ดึง P bit (bit 47)
    mov rcx, rax
    shr rcx, 47
    and rcx, 1
    test rcx, rcx
    jz .not_present
    
    ; ดึง DPL (bits 46:45)
    mov rcx, rax
    shr rcx, 45
    and rcx, 3
    cmp rcx, rsi
    jne .wrong_dpl
    
    ; ดึง S bit (bit 44)
    mov rcx, rax
    shr rcx, 44
    and rcx, 1
    ; ตรวจสอบ Type ต่อตาม S bit...
    
    xor rax, rax    ; Success
    ret

.not_present:
    mov rax, -1
    ret

.wrong_dpl:
    mov rax, -2
    ret
```

---

## 25. สรุปและ Key Points

**สิ่งที่ต้องจำ:**

1. **GDT entry 0 ต้องเป็น Null Descriptor** (8 bytes ของ 0)

2. **Selector = Index * 8 + TI + RPL**
   - Kernel Code: 0x08 (index 1, TI=0, RPL=0)
   - Kernel Data: 0x10 (index 2, TI=0, RPL=0)
   - User Code:   0x1B (index 3, TI=0, RPL=3)
   - User Data:   0x23 (index 4, TI=0, RPL=3)

3. **Descriptor Layout แปลก** — Intel แยก base/limit ออกเป็นหลาย field เพื่อ backward compatibility

4. **Access Byte = P + DPL + S + Type**
   - 0x9A = Kernel Code (32/64-bit)
   - 0x92 = Kernel Data
   - 0xFA = User Code
   - 0xF2 = User Data

5. **Flags = G + D/B + L + AVL**
   - 0xC (0xCF) = 32-bit flat
   - 0xA (0x20) = 64-bit code segment

6. **LGDT ต้องทำก่อน set CR0.PE**

7. **Far jump หลัง set CR0.PE** เพื่อ flush pipeline และ load CS ใหม่

8. **TSS ต้องมีก่อน interrupt ใด ๆ จาก Ring 3**
   - RSP0 = kernel stack pointer
   - LTR ต้องเรียกด้วย TSS selector

9. **64-bit TSS Descriptor = 16 bytes** (2 GDT entries)

10. **FS/GS base ตั้งได้ผ่าน MSR** สำหรับ TLS (Thread Local Storage)

```
Selectors ที่ใช้ใน Linux x86_64:
#define __KERNEL_CS  0x10   (GDT index 2, Ring 0)
#define __KERNEL_DS  0x18   (GDT index 3, Ring 0)
#define __USER_CS    0x33   (GDT index 6, Ring 3 = 0x30|3)
#define __USER_DS    0x2B   (GDT index 5, Ring 3 = 0x28|3)
```

---

## อ้างอิง (References)

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Volume 3A
  - Chapter 3: Protected-Mode Memory Management
  - Chapter 5: Protection
  - Chapter 7: Task Management
  
- AMD64 Architecture Programmer's Manual, Volume 2
  - Chapter 4: Segmentation
  
- OSDev Wiki: https://wiki.osdev.org/GDT
- OSDev Wiki: https://wiki.osdev.org/TSS

---

*Part 083 — Global Descriptor Table (GDT) Deep Dive*  
*Assembly Language Course — Thai/English Edition*
