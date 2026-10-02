# Part 081: Bootloader Programming (MBR/BIOS)

## บทนำ (Introduction)

Bootloader คือโปรแกรมชิ้นแรกที่ CPU รันหลังจากเปิดเครื่อง มันทำหน้าที่เป็นสะพานเชื่อมระหว่าง BIOS และ Operating System การเขียน Bootloader ใน Assembly ถือเป็นการเขียนโปรแกรมในระดับต่ำที่สุดที่เป็นไปได้ เพราะทำงานโดยตรงกับฮาร์ดแวร์โดยไม่มี OS ช่วย

ในบทนี้เราจะเรียนรู้:
- กระบวนการ POST ของ BIOS
- โครงสร้าง MBR (Master Boot Record)
- การเขียน Bootloader แบบ Real Mode 16-bit
- การใช้ BIOS Interrupts
- การโหลด 2nd Stage Loader
- การทดสอบด้วย QEMU

---

## 1. BIOS POST Sequence (กระบวนการเริ่มต้น)

### 1.1 Power-On Self-Test (POST)

เมื่อกดปุ่มเปิดเครื่อง CPU จะเริ่มทำงานที่ address `0xFFFF0` (หรือ `0xFFFFFFF0` ใน 32-bit) ซึ่งเป็น Reset Vector โดย BIOS ROM อยู่ที่บริเวณนี้

```
POST Sequence:
┌─────────────────────────────────────────────────────┐
│ 1. CPU Reset → Execute BIOS ROM at 0xFFFF:0000      │
│ 2. POST: ตรวจสอบฮาร์ดแวร์                           │
│    - CPU Test                                        │
│    - Memory Test (Count & Check RAM)                 │
│    - I/O Devices (Keyboard, Video, Disk)             │
│    - PCI Bus Initialization                          │
│ 3. Initialize IVT (Interrupt Vector Table)           │
│    - ที่ address 0x0000:0x0000 ขนาด 1KB             │
│ 4. Initialize BIOS Data Area (BDA)                   │
│    - ที่ address 0x0040:0x0000 ขนาด 256 bytes        │
│ 5. หา Bootable Device ตาม Boot Order                 │
│ 6. โหลด MBR (512 bytes) จาก Sector 0                │
│ 7. ตรวจสอบ Boot Signature (0x55AA)                  │
│ 8. Jump to 0x0000:0x7C00                             │
└─────────────────────────────────────────────────────┘
```

### 1.2 Memory Map ตอน POST

```
Memory Layout หลัง BIOS POST:
┌──────────────────────────────────────┐
│ 0x00000 - 0x003FF │ IVT (1KB)        │
│ 0x00400 - 0x004FF │ BDA (256 bytes)  │
│ 0x00500 - 0x07BFF │ Free Area        │
│ 0x07C00 - 0x07DFF │ MBR/Bootloader   │
│ 0x07E00 - 0x9FBFF │ Free (Available) │
│ 0x9FC00 - 0x9FFFF │ EBDA             │
│ 0xA0000 - 0xBFFFF │ Video RAM        │
│ 0xC0000 - 0xC7FFF │ Video BIOS ROM   │
│ 0xF0000 - 0xFFFFF │ BIOS ROM         │
└──────────────────────────────────────┘
```

---

## 2. Master Boot Record (MBR) Structure

### 2.1 MBR Layout

MBR อยู่ที่ Sector 0 (LBA 0) ของ disk มีขนาด 512 bytes:

```
MBR Structure (512 bytes):
┌──────────────────────────────────────────────┐
│ Offset  │ Size    │ Description              │
├──────────────────────────────────────────────┤
│ 0x000   │ 446     │ Bootstrap Code           │
│ 0x1BE   │ 16      │ Partition Entry 1        │
│ 0x1CE   │ 16      │ Partition Entry 2        │
│ 0x1DE   │ 16      │ Partition Entry 3        │
│ 0x1EE   │ 16      │ Partition Entry 4        │
│ 0x1FE   │ 2       │ Boot Signature (0x55AA)  │
└──────────────────────────────────────────────┘
```

### 2.2 Partition Entry Structure

```
Partition Entry (16 bytes):
┌─────────────────────────────────────────┐
│ Offset │ Size │ Description             │
├─────────────────────────────────────────┤
│ 0x00   │ 1    │ Status (0x80=bootable)  │
│ 0x01   │ 3    │ CHS Start               │
│ 0x04   │ 1    │ Partition Type          │
│ 0x05   │ 3    │ CHS End                 │
│ 0x08   │ 4    │ LBA Start               │
│ 0x0C   │ 4    │ Partition Size (sectors)│
└─────────────────────────────────────────┘
```

### 2.3 Boot Signature

```nasm
; Boot Signature ต้องอยู่ที่ offset 510-511 ของ MBR
; BIOS จะตรวจสอบค่านี้ก่อน boot
; 0x55 ที่ offset 510, 0xAA ที่ offset 511

TIMES 510 - ($ - $$) DB 0   ; pad ด้วย zeros จนถึง offset 510
DW 0xAA55                    ; Boot signature (little-endian: 55 AA)
```

---

## 3. Real Mode Programming

### 3.1 Real Mode คืออะไร

เมื่อ BIOS jump ไปที่ `0x0000:0x7C00` CPU ทำงานใน **16-bit Real Mode**:

- **Address Space**: 20-bit = 1MB สูงสุด (0x00000 - 0xFFFFF)
- **Registers**: AX, BX, CX, DX, SI, DI, SP, BP (16-bit)
- **Segment Registers**: CS, DS, ES, SS (16-bit)
- **Physical Address**: Segment × 16 + Offset
- **Conventional RAM**: 640KB (0x00000 - 0x9FFFF)
- **ไม่มี Memory Protection**
- **ไม่มี Virtual Memory**

### 3.2 Segment:Offset Addressing

```
Physical Address = (Segment Register × 0x10) + Offset Register

ตัวอย่าง:
CS = 0x0000, IP = 0x7C00
Physical = 0x0000 × 0x10 + 0x7C00 = 0x07C00

DS = 0x07C0, SI = 0x0100
Physical = 0x07C0 × 0x10 + 0x0100 = 0x07D00
```

```nasm
; การตั้งค่า Segment Registers ที่ต้น bootloader
BITS 16
ORG 0x7C00

start:
    ; ปิด interrupts ชั่วคราว
    cli

    ; ตั้ง segment registers
    xor ax, ax
    mov ds, ax          ; DS = 0
    mov es, ax          ; ES = 0
    mov ss, ax          ; SS = 0
    mov sp, 0x7C00      ; Stack pointer (grows downward จาก 0x7C00)

    ; เปิด interrupts
    sti
```

### 3.3 Register Usage ใน Real Mode

```
16-bit General Purpose Registers:
┌────────────────────────────────────────────┐
│ AX │ AH (high 8-bit) │ AL (low 8-bit)      │
│ BX │ BH             │ BL                   │
│ CX │ CH             │ CL                   │
│ DX │ DH             │ DL                   │
├────────────────────────────────────────────┤
│ SI │ Source Index                           │
│ DI │ Destination Index                      │
│ SP │ Stack Pointer                          │
│ BP │ Base Pointer                           │
├────────────────────────────────────────────┤
│ Segment Registers:                         │
│ CS │ Code Segment                           │
│ DS │ Data Segment                           │
│ ES │ Extra Segment                          │
│ SS │ Stack Segment                          │
└────────────────────────────────────────────┘
```

---

## 4. BIOS Interrupt 10h (Video Services)

### 4.1 INT 10h Overview

BIOS Interrupt 10h ให้บริการด้าน Video/Display:

```
INT 10h Services:
┌───────────────────────────────────────────────────┐
│ AH=00h │ Set Video Mode                            │
│ AH=01h │ Set Cursor Shape                          │
│ AH=02h │ Set Cursor Position                       │
│ AH=03h │ Get Cursor Position                       │
│ AH=06h │ Scroll Window Up                          │
│ AH=07h │ Scroll Window Down                        │
│ AH=08h │ Read Character at Cursor                  │
│ AH=09h │ Write Character at Cursor (with attribute)│
│ AH=0Ah │ Write Character at Cursor (no attribute)  │
│ AH=0Bh │ Set Color Palette                         │
│ AH=0Eh │ Write Character in TTY Mode               │
│ AH=0Fh │ Get Current Video Mode                    │
│ AH=13h │ Write String                              │
└───────────────────────────────────────────────────┘
```

### 4.2 AH=00h: Set Video Mode

```nasm
; ตั้ง Video Mode
; Input: AL = mode number
; Mode 03h = 80×25 text mode (สี)
; Mode 13h = 320×200 graphics (256 colors)

set_video_mode:
    mov ah, 0x00
    mov al, 0x03        ; 80×25 text mode
    int 0x10
    ret
```

### 4.3 AH=0Eh: Print Character (TTY Mode)

```nasm
; พิมพ์ character ทีละตัว
; Input: AL = ASCII character
;        BH = Page number (ปกติ 0)
;        BL = Foreground color (graphic mode)

print_char:
    mov ah, 0x0E
    mov bh, 0x00        ; Page 0
    int 0x10
    ret

; ตัวอย่างการใช้งาน: พิมพ์ 'A'
example_print_a:
    mov al, 'A'
    call print_char
    ret
```

### 4.4 AH=0Eh: Print String Function

```nasm
; Function: print_string
; พิมพ์ null-terminated string
; Input: SI = pointer to string
; Clobbers: AX, BX, SI

print_string:
    push ax
    push bx
    push si

.loop:
    lodsb               ; AL = [DS:SI], SI++
    or al, al           ; ตรวจสอบว่า null หรือเปล่า
    jz .done            ; ถ้า zero จบ loop

    mov ah, 0x0E        ; TTY print
    mov bh, 0x00        ; Page 0
    int 0x10

    jmp .loop

.done:
    pop si
    pop bx
    pop ax
    ret
```

### 4.5 AH=13h: Print String (Advanced)

```nasm
; INT 10h AH=13h - Write String
; Input:
;   AL = Write mode (0-3)
;        0 = chars only, update cursor
;        1 = chars+attrs, update cursor
;        2 = chars only, don't update cursor
;        3 = chars+attrs, don't update cursor
;   BH = Page number
;   BL = Attribute (ถ้า AL=0 หรือ AL=2)
;   CX = String length
;   DH = Row (0-based)
;   DL = Column (0-based)
;   ES:BP = Pointer to string

print_string_advanced:
    ; ตั้งค่า parameters
    mov ah, 0x13
    mov al, 0x01        ; chars+attrs, update cursor
    mov bh, 0x00        ; Page 0
    mov bl, 0x07        ; Attribute: light grey on black
    mov cx, msg_len     ; string length
    mov dh, 5           ; Row 5
    mov dl, 10          ; Column 10
    ; ES:BP ชี้ไปที่ string (ES ต้องตั้งก่อน)
    push es
    push cs
    pop es
    lea bp, [msg]
    int 0x10
    pop es
    ret

msg:    db 'Hello from Bootloader!', 0
msg_len equ $ - msg - 1
```

### 4.6 Text Mode Color Attributes

```nasm
; Text Attribute Byte:
; Bits 7: Blink (หรือ Bright Background)
; Bits 6-4: Background Color (0-7)
; Bits 3: Bright Foreground
; Bits 2-0: Foreground Color (0-7)

; Colors:
; 0 = Black    4 = Red
; 1 = Blue     5 = Magenta
; 2 = Green    6 = Brown/Yellow
; 3 = Cyan     7 = Light Grey
; (เพิ่ม bit 3 = Bright version)

; ตัวอย่าง Attributes:
; 0x07 = White on Black
; 0x0F = Bright White on Black
; 0x1F = Bright White on Blue
; 0x4F = Bright White on Red
; 0x70 = Black on Light Grey (reversed)
```

---

## 5. BIOS Interrupt 13h (Disk Services)

### 5.1 INT 13h Overview

```
INT 13h Services:
┌─────────────────────────────────────────────────────┐
│ AH=00h │ Reset Disk System                          │
│ AH=01h │ Get Status of Last Operation               │
│ AH=02h │ Read Sectors (CHS)                         │
│ AH=03h │ Write Sectors (CHS)                        │
│ AH=04h │ Verify Sectors                             │
│ AH=05h │ Format Cylinder                            │
│ AH=08h │ Get Drive Parameters                       │
│ AH=15h │ Get Drive Type                             │
│ AH=41h │ Check for INT 13h Extensions               │
│ AH=42h │ Extended Read Sectors (LBA)                │
│ AH=43h │ Extended Write Sectors (LBA)               │
│ AH=48h │ Get Extended Drive Parameters              │
└─────────────────────────────────────────────────────┘
```

### 5.2 CHS Addressing

CHS = Cylinder, Head, Sector เป็นวิธีเก่าในการ address disk:

```
CHS Parameters:
┌────────────────────────────────────────────────────┐
│ Cylinder: 0 to 1023 (10 bits)                      │
│ Head:     0 to 255  (8 bits)                        │
│ Sector:   1 to 63   (6 bits, เริ่มจาก 1 ไม่ใช่ 0)  │
└────────────────────────────────────────────────────┘

LBA to CHS:
  Cylinder = LBA / (Heads × Sectors_per_Track)
  Head     = (LBA / Sectors_per_Track) % Heads
  Sector   = (LBA % Sectors_per_Track) + 1
```

### 5.3 AH=02h: Read Sectors (CHS)

```nasm
; INT 13h AH=02h - Read Disk Sectors
; Input:
;   AH = 02h
;   AL = Number of sectors to read (1-128)
;   CH = Cylinder number (bits 7-0)
;   CL = Sector number (bits 5-0) | Cylinder high bits (bits 7-6)
;   DH = Head number
;   DL = Drive number (0x80=first HDD, 0x00=first floppy)
;   ES:BX = Buffer address
; Output:
;   CF = 0 success, CF = 1 error
;   AH = Status code (0=success)
;   AL = Number of sectors read

read_sector_chs:
    mov ah, 0x02
    mov al, 1           ; อ่าน 1 sector
    mov ch, 0           ; Cylinder 0
    mov cl, 2           ; Sector 2 (sector 1 คือ MBR)
    mov dh, 0           ; Head 0
    mov dl, 0x80        ; First hard drive
    mov bx, 0x7E00      ; Buffer ที่ 0x7E00
    int 0x13

    jc disk_error       ; ถ้า CF set = error
    cmp al, 1           ; ตรวจสอบจำนวน sector ที่อ่านได้
    jne disk_error

    ret

disk_error:
    ; จัดการ error
    mov si, disk_err_msg
    call print_string
    jmp $               ; หยุด (infinite loop)

disk_err_msg: db 'Disk Error!', 0x0D, 0x0A, 0
```

### 5.4 AH=08h: Get Drive Parameters

```nasm
; INT 13h AH=08h - Get Drive Parameters
; Input:
;   AH = 08h
;   DL = Drive number
;   ES:DI = 0000:0000 (บาง BIOS ต้องการ)
; Output:
;   CF = 0 success, 1 error
;   BL = Drive type (floppy only)
;   CH = Max cylinder (bits 7-0)
;   CL = Max sector (bits 5-0) | Max cylinder high (bits 7-6)
;   DH = Max head number
;   DL = Number of drives

get_drive_params:
    push es
    push di
    xor di, di
    mov es, di          ; ES:DI = 0000:0000

    mov ah, 0x08
    mov dl, 0x80        ; First HDD
    int 0x13

    pop di
    pop es

    jc .error

    ; แยกค่า
    movzx ax, dh        ; Max head
    inc ax              ; จำนวน heads = max head + 1
    mov [num_heads], ax

    movzx ax, cl
    and ax, 0x3F        ; Mask bits 6-7
    mov [sectors_per_track], ax

    ; Cylinders = (CH | ((CL >> 6) << 8)) + 1
    movzx ax, ch
    movzx bx, cl
    shr bx, 6
    shl bx, 8
    or ax, bx
    inc ax
    mov [num_cylinders], ax

    ret
.error:
    stc
    ret

; ตัวแปรเก็บค่า
num_heads:          dw 0
sectors_per_track:  dw 0
num_cylinders:      dw 0
```

---

## 6. LBA Addressing และ INT 13h Extensions

### 6.1 LBA (Logical Block Addressing)

LBA แก้ปัญหา CHS limit โดยนับ sector แบบ linear จาก 0:

```
LBA 0 = Sector 1, Head 0, Cylinder 0 (MBR)
LBA 1 = Sector 2, Head 0, Cylinder 0
...

ข้อดีของ LBA:
- ง่ายกว่า CHS มาก
- รองรับ disk ขนาดใหญ่กว่า
- LBA28 = 28-bit = สูงสุด 128GB
- LBA48 = 48-bit = สูงสุด 128PB
```

### 6.2 INT 13h AH=41h: Check Extensions

```nasm
; ตรวจสอบว่า BIOS รองรับ INT 13h Extensions หรือเปล่า
; Input:
;   AH = 41h
;   BX = 55AAh
;   DL = Drive number
; Output:
;   CF = 0: Extensions supported
;        AH = Major version
;        BX = AA55h
;        CX = Interface support bitmask
;   CF = 1: Not supported

check_int13_ext:
    mov ah, 0x41
    mov bx, 0x55AA
    mov dl, 0x80
    int 0x13

    jc .not_supported
    cmp bx, 0xAA55
    jne .not_supported

    ; Extensions supported!
    test cx, 0x01       ; Bit 0 = Extended Disk Access Functions
    jz .not_supported

    clc
    ret

.not_supported:
    stc
    ret
```

### 6.3 Disk Address Packet (DAP) Structure

```nasm
; DAP สำหรับ INT 13h AH=42h (Extended Read)
; ต้องอยู่ใน Data Segment

struc DAP
    .size       resb 1  ; ขนาดของ DAP (16 = 0x10)
    .reserved   resb 1  ; ต้องเป็น 0
    .sectors    resw 1  ; จำนวน sectors ที่ต้องการอ่าน
    .offset     resw 1  ; Offset ของ buffer
    .segment    resw 1  ; Segment ของ buffer
    .lba_low    resd 1  ; LBA address (low 32 bits)
    .lba_high   resd 1  ; LBA address (high 32 bits)
endstruc

; ตัวอย่าง DAP สำหรับอ่าน sector 1 ไปที่ 0x0000:0x7E00
disk_address_packet:
    .size:      db  0x10        ; DAP size = 16 bytes
    .reserved:  db  0x00        ; Reserved = 0
    .sectors:   dw  1           ; อ่าน 1 sector
    .offset:    dw  0x7E00      ; Buffer offset
    .segment:   dw  0x0000      ; Buffer segment
    .lba_low:   dd  1           ; LBA 1 (sector ถัดจาก MBR)
    .lba_high:  dd  0           ; High 32 bits = 0
```

### 6.4 AH=42h: Extended Read Sectors (LBA)

```nasm
; INT 13h AH=42h - Extended Read Sectors
; Input:
;   AH = 42h
;   DL = Drive number
;   DS:SI = Pointer to DAP
; Output:
;   CF = 0: success
;   CF = 1: error, AH = error code

read_sectors_lba:
    ; ตั้งค่า DAP
    mov word [dap_sectors], 4    ; อ่าน 4 sectors
    mov word [dap_offset], 0x8000 ; Buffer ที่ 0x8000
    mov word [dap_segment], 0x0000
    mov dword [dap_lba_low], 1   ; เริ่มจาก LBA 1
    mov dword [dap_lba_high], 0

    ; เรียก INT 13h
    mov ah, 0x42
    mov dl, 0x80            ; Drive 0 (first HDD)
    lea si, [disk_address_packet]
    int 0x13

    jc .error
    ret

.error:
    ; Error handling
    mov si, read_error_msg
    call print_string
    jmp $

; DAP Data Structure
disk_address_packet:
dap_size:     db  0x10
dap_reserved: db  0x00
dap_sectors:  dw  0
dap_offset:   dw  0
dap_segment:  dw  0
dap_lba_low:  dd  0
dap_lba_high: dd  0

read_error_msg: db 'Read Error!', 0x0D, 0x0A, 0
```

---

## 7. A20 Gate

### 7.1 A20 Gate คืออะไร

A20 Gate คือ hardware mechanism ที่ควบคุม address line A20 (bit 20 ของ address bus):

```
ปัญหาเก่า:
- CPU 8086 มี 20 address lines (A0-A19) = 1MB max
- ถ้า address เกิน 0xFFFFF มัน wrap กลับไปที่ 0x00000
- โปรแกรมเก่าบางตัวพึ่งพา wrap-around นี้
- CPU 286 มี 24 address lines แต่ต้องรักษา compatibility
- A20 Gate จึงถูกสร้างขึ้นเพื่อ disable bit 20

ผลลัพธ์:
- A20 disabled: address 0x100000 = 0x000000 (wrap)
- A20 enabled:  address 0x100000 = 0x100000 (ถูกต้อง)
- ต้อง enable A20 เพื่อ access > 1MB
```

### 7.2 วิธี Enable A20

มีหลายวิธี:

```nasm
; ─── วิธี 1: BIOS INT 15h ───
; อาจไม่ทำงานบน hardware เก่า

enable_a20_bios:
    mov ax, 0x2401      ; Enable A20
    int 0x15
    jc .failed          ; CF = 1 ถ้า error
    ret
.failed:
    stc
    ret

; ─── วิธี 2: Keyboard Controller ───
; วิธีดั้งเดิม, ช้าแต่ compatible มาก

enable_a20_keyboard:
    cli

    call .wait_input
    mov al, 0xAD        ; Disable keyboard
    out 0x64, al

    call .wait_input
    mov al, 0xD0        ; Read output port
    out 0x64, al

    call .wait_output
    in al, 0x60         ; อ่านค่า output port
    push ax

    call .wait_input
    mov al, 0xD1        ; Write output port
    out 0x64, al

    call .wait_input
    pop ax
    or al, 0x02         ; Set bit 1 (A20)
    out 0x60, al

    call .wait_input
    mov al, 0xAE        ; Enable keyboard
    out 0x64, al

    call .wait_input
    sti
    ret

.wait_input:
    in al, 0x64
    test al, 0x02       ; Bit 1 = Input buffer full
    jnz .wait_input
    ret

.wait_output:
    in al, 0x64
    test al, 0x01       ; Bit 0 = Output buffer full
    jz .wait_output
    ret

; ─── วิธี 3: Fast A20 (Port 0x92) ───
; เร็วที่สุด แต่ไม่ทำงานทุก hardware

enable_a20_fast:
    in al, 0x92
    test al, 0x02       ; ตรวจสอบว่า A20 enabled แล้วหรือยัง
    jnz .done
    or al, 0x02
    and al, 0xFE        ; ไม่ reset system!
    out 0x92, al
.done:
    ret
```

### 7.3 ตรวจสอบว่า A20 Enabled

```nasm
; ตรวจสอบ A20 โดยเปรียบเทียบ memory addresses
; 0x0000:0x0500 กับ 0xFFFF:0x0510
; ทั้งสองจะ map ไป physical address เดียวกันถ้า A20 disabled

check_a20:
    push ax
    push bx
    push es

    xor ax, ax
    mov es, ax

    mov ax, 0xFFFF
    mov es, ax          ; ES = 0xFFFF

    mov al, byte [es:0x0510]  ; อ่าน 0xFFFF:0x0510 = physical 0x10050F (ถ้า A20 on)
    push ax

    mov byte [0x0500], 0x00   ; เขียน 0x0000:0x0500
    mov byte [es:0x0510], 0xFF ; เขียน 0xFFFF:0x0510

    cmp byte [0x0500], 0xFF   ; ถ้า A20 off จะ wrap และเท่ากัน
    je .disabled

    clc                 ; A20 enabled
    jmp .done

.disabled:
    stc                 ; A20 disabled

.done:
    pop ax
    mov byte [es:0x0510], al  ; คืนค่าเดิม
    pop es
    pop bx
    pop ax
    ret
```

---

## 8. BIOS INT 15h E820: Memory Map

### 8.1 INT 15h AX=E820h

วิธีที่ดีที่สุดในการรับ memory map จาก BIOS:

```
Memory Types:
1 = Usable RAM
2 = Reserved (BIOS, hardware)
3 = ACPI Reclaimable
4 = ACPI NVS (Non-Volatile Storage)
5 = Bad Memory
```

### 8.2 E820 Entry Structure

```nasm
; E820 Memory Map Entry (20 bytes)
struc E820Entry
    .base_low:   resd 1  ; Base address (low 32 bits)
    .base_high:  resd 1  ; Base address (high 32 bits)
    .len_low:    resd 1  ; Length (low 32 bits)
    .len_high:   resd 1  ; Length (high 32 bits)
    .type:       resd 1  ; Memory type (1=usable, 2=reserved)
endstruc
```

### 8.3 Reading Memory Map

```nasm
; อ่าน memory map จาก BIOS
; เก็บไว้ที่ 0x0500 (ใน free memory area)
; Output: [mem_map_count] = จำนวน entries

MEMORY_MAP_ADDR equ 0x0500  ; เก็บ map ที่นี่
MAX_ENTRIES     equ 32

read_memory_map:
    push es
    push di
    push bx

    xor ax, ax
    mov es, ax
    mov di, MEMORY_MAP_ADDR  ; ES:DI ชี้ไปที่ buffer

    xor ebx, ebx             ; EBX = 0 (เริ่มต้น)
    xor bp, bp               ; BP = entry count

.loop:
    mov eax, 0xE820         ; Magic number
    mov ecx, 24             ; Buffer size (อาจต้องการ 24 bytes สำหรับ ACPI 3.0)
    mov edx, 0x534D4150     ; 'SMAP' signature
    int 0x15

    ; ตรวจสอบ error
    jc .done                ; CF = 1 = error หรือจบแล้ว
    cmp eax, 0x534D4150     ; ต้องได้ 'SMAP' กลับมา
    jne .done

    ; ตรวจสอบว่า entry valid
    cmp cx, 20              ; ต้องได้อย่างน้อย 20 bytes
    jl .skip_entry

    ; เก็บ entry
    inc bp                  ; เพิ่ม counter
    add di, 20              ; ชี้ไปยัง next entry

    ; ตรวจสอบว่าครบ max หรือยัง
    cmp bp, MAX_ENTRIES
    jge .done

.skip_entry:
    ; ตรวจสอบว่ายัง continue ได้
    test ebx, ebx
    jz .done                ; EBX = 0 = last entry
    jmp .loop

.done:
    mov [mem_map_count], bp

    pop bx
    pop di
    pop es
    ret

mem_map_count: dw 0

; ─── แสดง Memory Map ───
print_memory_map:
    mov cx, [mem_map_count]
    test cx, cx
    jz .empty

    mov si, mmap_header
    call print_string

    mov di, MEMORY_MAP_ADDR
    xor bx, bx

.loop:
    ; แสดง entry
    ; (ในที่นี้แสดงแค่ type ง่ายๆ)
    push cx
    mov eax, [di + 16]  ; Type
    cmp eax, 1
    je .usable
    mov si, mmap_reserved
    jmp .print_type
.usable:
    mov si, mmap_usable
.print_type:
    call print_string
    pop cx

    add di, 20
    inc bx
    loop .loop
    ret

.empty:
    mov si, mmap_empty
    call print_string
    ret

mmap_header:    db 'Memory Map:', 0x0D, 0x0A, 0
mmap_usable:    db '  Usable RAM', 0x0D, 0x0A, 0
mmap_reserved:  db '  Reserved', 0x0D, 0x0A, 0
mmap_empty:     db '  No entries', 0x0D, 0x0A, 0
```

---

## 9. 2nd Stage Loader

### 9.1 ทำไมต้องมี 2nd Stage

MBR มีพื้นที่แค่ 446 bytes สำหรับ code ซึ่งน้อยมาก 2nd Stage แก้ปัญหานี้:

```
Boot Process:
┌────────────────────────────────────────────────────────┐
│ Stage 1 (MBR, 446 bytes):                              │
│  - เริ่มต้น hardware พื้นฐาน                           │
│  - อ่าน Stage 2 จาก disk                              │
│  - Jump ไป Stage 2                                     │
│                                                        │
│ Stage 2 (~4-128KB, โหลดที่ 0x1000:0):                 │
│  - Enable A20                                          │
│  - อ่าน Memory Map                                     │
│  - Load Kernel จาก disk                               │
│  - ตั้งค่า Protected Mode                              │
│  - Jump ไป Kernel                                      │
└────────────────────────────────────────────────────────┘
```

### 9.2 2nd Stage Load Address

```nasm
; 2nd Stage โหลดที่ 0x1000:0x0000 = physical 0x10000
; หรือโหลดที่ 0x0000:0x8000 = physical 0x8000
; ขึ้นอยู่กับขนาด

STAGE2_SEGMENT equ 0x1000
STAGE2_OFFSET  equ 0x0000
STAGE2_LBA     equ 1       ; เริ่มจาก sector 1 (ถัดจาก MBR)
STAGE2_SECTORS equ 8       ; โหลด 8 sectors = 4KB
```

### 9.3 Load 2nd Stage Code (ใน Stage 1)

```nasm
load_stage2:
    ; ตั้ง buffer
    mov ax, STAGE2_SEGMENT
    mov es, ax
    mov bx, STAGE2_OFFSET

    ; ตั้ง DAP
    mov word [dap_sectors], STAGE2_SECTORS
    mov word [dap_offset], STAGE2_OFFSET
    mov word [dap_segment], STAGE2_SEGMENT
    mov dword [dap_lba_low], STAGE2_LBA
    mov dword [dap_lba_high], 0

    ; อ่านด้วย LBA mode
    mov ah, 0x42
    mov dl, [boot_drive]    ; Drive ที่ boot มา
    lea si, [dap]
    int 0x13

    jc .error
    ret

.error:
    mov si, stage2_err
    call print_string
    cli
    hlt

stage2_err: db 'Cannot load Stage 2!', 0x0D, 0x0A, 0
```

---

## 10. GRUB Multiboot Header

### 10.1 Multiboot Specification

GRUB รองรับ kernel ที่มี Multiboot Header:

```
Multiboot Header Location:
- ต้องอยู่ใน 8192 bytes แรกของ kernel image
- ต้อง 4-byte aligned
- มี magic number 0x1BADB002

Multiboot Header Structure:
┌─────────────────────────────────────────────────┐
│ magic    │ 0x1BADB002                            │
│ flags    │ feature flags                         │
│ checksum │ -(magic + flags)                      │
│ [header_addr  - ถ้า flag bit 16 set]             │
│ [load_addr                                      ]│
│ [load_end_addr                                  ]│
│ [bss_end_addr                                   ]│
│ [entry_addr                                     ]│
│ [mode_type    - ถ้า flag bit 2 set]              │
│ [width                                          ]│
│ [height                                         ]│
│ [depth                                          ]│
└─────────────────────────────────────────────────┘
```

### 10.2 Simple Multiboot Kernel

```nasm
; kernel.asm - Multiboot Kernel
; สำหรับ load ด้วย GRUB

BITS 32

; Multiboot Constants
MULTIBOOT_MAGIC    equ 0x1BADB002
MULTIBOOT_ALIGN    equ 1 << 0    ; Align modules on page boundaries
MULTIBOOT_MEMINFO  equ 1 << 1    ; Memory map
MULTIBOOT_FLAGS    equ MULTIBOOT_ALIGN | MULTIBOOT_MEMINFO
MULTIBOOT_CHECKSUM equ -(MULTIBOOT_MAGIC + MULTIBOOT_FLAGS)

section .multiboot
align 4
    dd MULTIBOOT_MAGIC
    dd MULTIBOOT_FLAGS
    dd MULTIBOOT_CHECKSUM

section .text
global _start

_start:
    ; EAX ควรมีค่า 0x2BADB002 (multiboot magic)
    ; EBX ชี้ไปที่ multiboot info structure

    ; ตั้ง stack
    mov esp, stack_top

    ; เรียก kernel main
    extern kernel_main
    call kernel_main

    ; ถ้า kernel_main return (ไม่ควรเกิด)
    cli
.halt:
    hlt
    jmp .halt

section .bss
align 16
stack_bottom:
    resb 16384          ; 16KB stack
stack_top:
```

---

## 11. Complete NASM Bootloader

### 11.1 Bootloader ที่สมบูรณ์

นี่คือ bootloader ที่: พิมพ์ข้อความ, รับ keyboard input, โหลด Stage 2, และ jump ไป Stage 2:

```nasm
; ============================================================
; boot.asm - Complete MBR Bootloader
; Compile: nasm -f bin boot.asm -o boot.bin
; Test:    qemu-system-x86_64 -fda boot.bin
; ============================================================

BITS 16
ORG 0x7C00

; ─── Constants ───
STAGE2_SEG    equ 0x1000    ; โหลด Stage 2 ที่ 0x1000:0
STAGE2_OFS    equ 0x0000
STAGE2_LBA    equ 1         ; Stage 2 เริ่มที่ sector 1
STAGE2_SECTS  equ 16        ; โหลด 16 sectors = 8KB

VGA_TEXT_MEM  equ 0xB800    ; VGA text buffer
STACK_TOP     equ 0x7C00    ; Stack ก่อน bootloader

; ─── Entry Point ───
start:
    ; === Setup ===
    cli                     ; ปิด interrupts ระหว่าง setup
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, STACK_TOP
    sti                     ; เปิด interrupts

    ; เก็บ drive number ที่ BIOS ส่งมาใน DL
    mov [boot_drive], dl

    ; === Set Video Mode ===
    mov ax, 0x0003          ; AH=0 set mode, AL=3 (80×25 text)
    int 0x10

    ; === Print Welcome Message ===
    mov si, msg_welcome
    call print_string

    ; === Check INT 13h Extensions ===
    call check_lba_support
    jnc .lba_ok

    ; LBA ไม่รองรับ ต้องใช้ CHS
    mov si, msg_chs_mode
    call print_string
    jmp .read_stage2_chs

.lba_ok:
    mov si, msg_lba_mode
    call print_string

    ; === Load Stage 2 via LBA ===
    call load_stage2_lba
    jnc .stage2_loaded
    jmp disk_error_handler

.read_stage2_chs:
    call load_stage2_chs
    jnc .stage2_loaded
    jmp disk_error_handler

.stage2_loaded:
    mov si, msg_stage2_ok
    call print_string

    ; === Wait for keypress (optional) ===
    mov si, msg_press_key
    call print_string
    call read_key
    mov si, msg_newline
    call print_string

    ; === Jump to Stage 2 ===
    jmp STAGE2_SEG:STAGE2_OFS   ; Far jump ไปยัง Stage 2

    ; ไม่ควรมาถึงบรรทัดนี้
    jmp $

; ─────────────────────────────────────────
; check_lba_support
; ตรวจสอบว่า BIOS รองรับ INT 13h Extensions
; Output: CF=0 supported, CF=1 not supported
; ─────────────────────────────────────────
check_lba_support:
    mov ah, 0x41
    mov bx, 0x55AA
    mov dl, [boot_drive]
    int 0x13
    jc .not_supported
    cmp bx, 0xAA55
    jne .not_supported
    test cx, 0x01
    jz .not_supported
    clc
    ret
.not_supported:
    stc
    ret

; ─────────────────────────────────────────
; load_stage2_lba
; โหลด Stage 2 ด้วย LBA Extended Read
; Output: CF=0 success, CF=1 error
; ─────────────────────────────────────────
load_stage2_lba:
    ; ตั้งค่า DAP
    mov byte [dap.size], 0x10
    mov byte [dap.reserved], 0
    mov word [dap.sectors], STAGE2_SECTS
    mov word [dap.offset], STAGE2_OFS
    mov word [dap.segment], STAGE2_SEG
    mov dword [dap.lba_low], STAGE2_LBA
    mov dword [dap.lba_high], 0

    mov ah, 0x42
    mov dl, [boot_drive]
    lea si, [dap]
    int 0x13
    ret                     ; CF จะบอกผลลัพธ์

; ─────────────────────────────────────────
; load_stage2_chs
; โหลด Stage 2 ด้วย CHS (fallback)
; Output: CF=0 success, CF=1 error
; ─────────────────────────────────────────
load_stage2_chs:
    mov ax, STAGE2_SEG
    mov es, ax
    mov bx, STAGE2_OFS

    ; อ่านทีละ sector ด้วย CHS
    ; (สมมติ sectors_per_track = 63, heads = 16 สำหรับ floppy/basic)
    mov ah, 0x02            ; Read sectors
    mov al, STAGE2_SECTS    ; จำนวน sectors
    mov ch, 0               ; Cylinder 0
    mov cl, 2               ; Sector 2 (ถัดจาก MBR ที่ sector 1)
    mov dh, 0               ; Head 0
    mov dl, [boot_drive]
    int 0x13
    ret

; ─────────────────────────────────────────
; disk_error_handler
; จัดการ disk error
; ─────────────────────────────────────────
disk_error_handler:
    mov si, msg_disk_error
    call print_string
    ; แสดง error code ใน AH
    mov al, ah              ; AH = error code
    call print_hex_byte
    mov si, msg_newline
    call print_string
    ; รอ keypress แล้ว reboot
    call read_key
    jmp 0xFFFF:0x0000       ; Reboot (jump to reset vector)

; ─────────────────────────────────────────
; print_string
; พิมพ์ null-terminated string
; Input: SI = pointer to string
; ─────────────────────────────────────────
print_string:
    push ax
    push bx
.loop:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    mov bh, 0
    int 0x10
    jmp .loop
.done:
    pop bx
    pop ax
    ret

; ─────────────────────────────────────────
; print_hex_byte
; พิมพ์ byte ในรูป hex
; Input: AL = byte to print
; ─────────────────────────────────────────
print_hex_byte:
    push ax
    push cx

    ; Print "0x"
    push ax
    mov al, '0'
    mov ah, 0x0E
    mov bh, 0
    int 0x10
    mov al, 'x'
    int 0x10
    pop ax

    ; High nibble
    mov cl, 4
    shr al, cl
    call .nibble

    ; Low nibble (restore original)
    pop ax
    push ax
    and al, 0x0F
    call .nibble

    pop cx
    pop ax
    ret

.nibble:
    cmp al, 9
    jle .digit
    add al, 'A' - 10
    jmp .print
.digit:
    add al, '0'
.print:
    mov ah, 0x0E
    mov bh, 0
    int 0x10
    ret

; ─────────────────────────────────────────
; read_key
; รอและอ่าน keypress
; Output: AL = ASCII code, AH = scan code
; ─────────────────────────────────────────
read_key:
    mov ah, 0x00
    int 0x16
    ret

; ─────────────────────────────────────────
; print_char_at
; พิมพ์ character ที่ตำแหน่งที่กำหนด
; Input: AL = char, BH = row, BL = col, CH = attribute
; ─────────────────────────────────────────
print_char_at:
    push ax
    push bx
    push cx
    push dx

    ; ตั้ง cursor position
    mov ah, 0x02
    mov dh, bh          ; Row
    mov dl, bl          ; Column
    xor bx, bx          ; Page 0
    int 0x10

    ; พิมพ์ character
    mov ah, 0x09
    xor bh, bh          ; Page 0
    mov bl, ch          ; Attribute
    mov cx, 1           ; 1 character
    int 0x10

    pop dx
    pop cx
    pop bx
    pop ax
    ret

; ─────────────────────────────────────────
; clear_screen
; ล้างหน้าจอ
; ─────────────────────────────────────────
clear_screen:
    mov ax, 0x0600      ; AH=06 scroll up, AL=0 clear
    mov bh, 0x07        ; Attribute: white on black
    mov cx, 0x0000      ; Upper-left (row 0, col 0)
    mov dx, 0x184F      ; Lower-right (row 24, col 79)
    int 0x10

    ; Move cursor to 0,0
    mov ah, 0x02
    xor bx, bx
    xor dx, dx
    int 0x10
    ret

; ─────────────────────────────────────────
; Data Section
; ─────────────────────────────────────────

; Boot drive number
boot_drive: db 0x80

; Disk Address Packet
align 2
dap:
  .size:     db 0x10
  .reserved: db 0x00
  .sectors:  dw 0
  .offset:   dw 0
  .segment:  dw 0
  .lba_low:  dd 0
  .lba_high: dd 0

; Messages
msg_welcome:
    db 0x0D, 0x0A
    db '====================================', 0x0D, 0x0A
    db '  Simple MBR Bootloader v1.0        ', 0x0D, 0x0A
    db '  Written in NASM Assembly          ', 0x0D, 0x0A
    db '====================================', 0x0D, 0x0A
    db 0x0D, 0x0A, 0

msg_lba_mode:
    db '[OK] LBA mode supported', 0x0D, 0x0A, 0

msg_chs_mode:
    db '[!!] LBA not supported, using CHS', 0x0D, 0x0A, 0

msg_stage2_ok:
    db '[OK] Stage 2 loaded successfully!', 0x0D, 0x0A, 0

msg_press_key:
    db 'Press any key to continue...', 0

msg_disk_error:
    db 0x0D, 0x0A, '[ERR] Disk error: ', 0

msg_newline:
    db 0x0D, 0x0A, 0

; ─────────────────────────────────────────
; Boot Signature (ต้องอยู่ที่ offset 510)
; ─────────────────────────────────────────
TIMES 510 - ($ - $$) DB 0
DW 0xAA55
```

---

## 12. Stage 2 Loader

### 12.1 stage2.asm - ไฟล์ Stage 2

```nasm
; ============================================================
; stage2.asm - Second Stage Bootloader
; ทำงานที่ physical address 0x10000 (0x1000:0x0000)
; Compile: nasm -f bin stage2.asm -o stage2.bin
; ============================================================

BITS 16
ORG 0x0000          ; เราจะ far jump มาที่ 0x1000:0

; ─── Constants ───
KERNEL_SEG    equ 0x2000    ; โหลด kernel ที่ 0x20000
KERNEL_OFS    equ 0x0000
KERNEL_LBA    equ 17        ; Kernel เริ่มที่ sector 17
KERNEL_SECTS  equ 64        ; โหลด 64 sectors = 32KB

; ─── Entry Point ───
stage2_start:
    ; ตั้ง segment registers
    mov ax, 0x1000
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0xFFFE      ; Stack ท้าย segment 0x1000

    ; ─── Enable A20 ───
    call enable_a20
    call check_a20
    jnc .a20_ok
    mov si, msg_a20_fail
    call print_str16
    jmp hang
.a20_ok:
    mov si, msg_a20_ok
    call print_str16

    ; ─── Read Memory Map ───
    call read_e820
    mov si, msg_mmap_done
    call print_str16

    ; ─── Load Kernel ───
    call load_kernel
    mov si, msg_kernel_loaded
    call print_str16

    ; ─── Switch to Protected Mode ───
    ; (ดู Part 082 สำหรับ Protected Mode)
    mov si, msg_entering_pm
    call print_str16

    cli                 ; ปิด interrupts
    call enable_gdt     ; Load GDT
    call enter_pm       ; Switch to PM

    ; ไม่ควรมาถึงบรรทัดนี้
    jmp hang

; ─── print_str16 (DS:SI) ───
print_str16:
    push ax
    push bx
.loop:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    xor bh, bh
    int 0x10
    jmp .loop
.done:
    pop bx
    pop ax
    ret

; ─── Enable A20 ───
enable_a20:
    ; ลอง BIOS method ก่อน
    mov ax, 0x2401
    int 0x15
    jnc .done

    ; ลอง Fast A20
    in al, 0x92
    or al, 0x02
    and al, 0xFE
    out 0x92, al
.done:
    ret

; ─── Check A20 ───
check_a20:
    push es
    mov ax, 0xFFFF
    mov es, ax

    ; Write ที่ 0x0000:0x0600
    mov byte [ds:0x0600], 0xAA
    ; Write ที่ 0xFFFF:0x0610 (= physical 0x1060F ถ้า A20 on)
    mov byte [es:0x0610], 0x55

    cmp byte [ds:0x0600], 0x55  ; ถ้า A20 off จะ wrap
    je .disabled

    clc
    jmp .done
.disabled:
    stc
.done:
    pop es
    ret

; ─── Read E820 Memory Map ───
E820_BUFFER equ 0x0600  ; relative to DS (0x1000:0x0600 = phys 0x10600)

read_e820:
    push es
    mov ax, ds
    mov es, ax

    mov di, E820_BUFFER
    xor ebx, ebx
    xor bp, bp

.loop:
    mov eax, 0xE820
    mov ecx, 20
    mov edx, 0x534D4150
    int 0x15
    jc .done
    cmp eax, 0x534D4150
    jne .done

    inc bp
    add di, 20

    cmp bp, 20          ; Max 20 entries
    jge .done
    test ebx, ebx
    jz .done
    jmp .loop

.done:
    mov [e820_count], bp
    pop es
    ret

; ─── Load Kernel from Disk ───
load_kernel:
    ; ตั้งค่า DAP
    mov word [k_dap.size], 0x10
    mov word [k_dap.reserved], 0
    mov word [k_dap.sectors], KERNEL_SECTS
    mov word [k_dap.offset], KERNEL_OFS
    mov word [k_dap.segment], KERNEL_SEG
    mov dword [k_dap.lba_low], KERNEL_LBA
    mov dword [k_dap.lba_high], 0

    mov ah, 0x42
    mov dl, [boot_drive_s2]
    lea si, [k_dap]
    int 0x13
    jc .error
    ret

.error:
    mov si, msg_kernel_err
    call print_str16
    jmp hang

; ─── GDT สำหรับ Protected Mode ───
align 8
gdt_start:
    ; Null descriptor
    dq 0

    ; Code segment (base=0, limit=4GB, ring 0, 32-bit)
    dw 0xFFFF           ; Limit low
    dw 0x0000           ; Base low
    db 0x00             ; Base mid
    db 10011010b        ; Access: present, ring0, code, readable
    db 11001111b        ; Flags: 4KB granularity, 32-bit + Limit high
    db 0x00             ; Base high

    ; Data segment (base=0, limit=4GB, ring 0)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 10010010b        ; Access: present, ring0, data, writable
    db 11001111b
    db 0x00

gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1  ; GDT size - 1
    dd gdt_start + (0x1000 << 4) ; Physical address of GDT

CODE_SEG equ 0x08       ; Offset ใน GDT
DATA_SEG equ 0x10

; ─── Load GDT ───
enable_gdt:
    lgdt [gdt_descriptor]
    ret

; ─── Enter Protected Mode ───
enter_pm:
    mov eax, cr0
    or eax, 0x1         ; Set PE bit
    mov cr0, eax

    ; Far jump เพื่อ flush instruction pipeline
    jmp CODE_SEG:pm_entry_32

BITS 32
pm_entry_32:
    ; ตั้ง data segments
    mov ax, DATA_SEG
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    mov esp, 0x00090000 ; Stack ใน Protected Mode

    ; Jump ไปยัง kernel
    jmp KERNEL_SEG * 16 + KERNEL_OFS

BITS 16

; ─── Hang (infinite loop) ───
hang:
    cli
    hlt
    jmp hang

; ─── Data ───
boot_drive_s2: db 0x80

e820_count: dw 0

; Kernel DAP
k_dap:
  .size:     dw 0
  .reserved: dw 0
  .sectors:  dw 0
  .offset:   dw 0
  .segment:  dw 0
  .lba_low:  dd 0
  .lba_high: dd 0

msg_a20_ok:      db '[S2] A20 enabled', 0x0D, 0x0A, 0
msg_a20_fail:    db '[S2] A20 FAILED!', 0x0D, 0x0A, 0
msg_mmap_done:   db '[S2] Memory map read', 0x0D, 0x0A, 0
msg_kernel_loaded: db '[S2] Kernel loaded', 0x0D, 0x0A, 0
msg_kernel_err:  db '[S2] Kernel load error!', 0x0D, 0x0A, 0
msg_entering_pm: db '[S2] Entering Protected Mode...', 0x0D, 0x0A, 0
```

---

## 13. Disk Image Layout

### 13.1 โครงสร้าง disk image

```
Disk Image Layout:
┌──────────────────────────────────────────────┐
│ Sector 0  │ MBR (boot.bin) - 512 bytes       │
│ Sector 1  │ Stage 2 start                    │
│ ...       │ Stage 2 continues (16 sectors)   │
│ Sector 17 │ Kernel start                     │
│ ...       │ Kernel continues (64 sectors)    │
└──────────────────────────────────────────────┘
```

### 13.2 สร้าง Disk Image

```bash
# สร้าง raw disk image ขนาด 1.44MB (floppy)
dd if=/dev/zero of=disk.img bs=512 count=2880

# เขียน MBR
dd if=boot.bin of=disk.img bs=512 count=1 conv=notrunc

# เขียน Stage 2 ที่ sector 1
dd if=stage2.bin of=disk.img bs=512 seek=1 conv=notrunc

# เขียน Kernel ที่ sector 17
dd if=kernel.bin of=disk.img bs=512 seek=17 conv=notrunc
```

---

## 14. Makefile

### 14.1 Makefile สมบูรณ์

```makefile
# Makefile สำหรับ Bootloader Project

NASM    := nasm
QEMU    := qemu-system-x86_64
DD      := dd

# ─── Targets ───
all: disk.img

# Compile MBR bootloader
boot.bin: boot.asm
	$(NASM) -f bin $< -o $@
	@echo "MBR size: $$(wc -c < $@) bytes (max 512)"
	@if [ $$(wc -c < $@) -ne 512 ]; then \
		echo "ERROR: MBR must be exactly 512 bytes!"; exit 1; fi

# Compile Stage 2
stage2.bin: stage2.asm
	$(NASM) -f bin $< -o $@
	@echo "Stage 2 size: $$(wc -c < $@) bytes"

# Compile Kernel (ถ้ามี)
kernel.bin: kernel.asm
	$(NASM) -f bin $< -o $@

# สร้าง disk image
disk.img: boot.bin stage2.bin kernel.bin
	# สร้าง blank image (1.44MB floppy)
	$(DD) if=/dev/zero of=$@ bs=512 count=2880 2>/dev/null
	# เขียน MBR
	$(DD) if=boot.bin of=$@ bs=512 count=1 conv=notrunc 2>/dev/null
	# เขียน Stage 2 (ที่ sector 1)
	$(DD) if=stage2.bin of=$@ bs=512 seek=1 conv=notrunc 2>/dev/null
	# เขียน Kernel (ที่ sector 17)
	$(DD) if=kernel.bin of=$@ bs=512 seek=17 conv=notrunc 2>/dev/null
	@echo "Disk image created: $@"

# ─── Testing ───
# Run ด้วย floppy disk
run-floppy: disk.img
	$(QEMU) -fda $< -boot a

# Run ด้วย hard disk
run-hdd: disk.img
	$(QEMU) -hda $< -boot c

# Run พร้อม debug output
run-debug: disk.img
	$(QEMU) -fda $< -boot a \
		-d int,cpu_reset \
		-no-reboot \
		-no-shutdown

# Run พร้อม GDB server (port 1234)
run-gdb: disk.img
	$(QEMU) -fda $< -boot a \
		-s -S \
		-no-reboot \
		-no-shutdown &
	@echo "QEMU started with GDB server on port 1234"
	@echo "Connect with: gdb -ex 'target remote localhost:1234'"

# Run พร้อม serial output
run-serial: disk.img
	$(QEMU) -fda $< -boot a \
		-serial stdio \
		-no-reboot

# ─── Debug Tools ───
# แสดง hex dump ของ MBR
dump-mbr: boot.bin
	xxd $<

# ตรวจสอบ boot signature
check-sig: boot.bin
	@python3 -c "\
	with open('boot.bin', 'rb') as f:\
	    data = f.read();\
	sig = data[510:512];\
	print(f'Boot signature: {sig.hex().upper()}');\
	print('Valid!' if sig == b'\x55\xaa' else 'INVALID!')"

# แสดง disassembly
disasm: boot.bin
	ndisasm -b 16 -o 0x7C00 boot.bin | head -60

# ─── Clean ───
clean:
	rm -f *.bin *.img *.o *.lst

.PHONY: all run-floppy run-hdd run-debug run-gdb run-serial \
	dump-mbr check-sig disasm clean
```

---

## 15. QEMU Test Commands

### 15.1 QEMU Parameters สำคัญ

```bash
# ─── Basic Testing ───

# Test ด้วย floppy disk image
qemu-system-x86_64 -fda boot.bin

# Test ด้วย hard disk image
qemu-system-x86_64 -hda disk.img -boot c

# Test ด้วย ISO image
qemu-system-x86_64 -cdrom boot.iso -boot d

# ─── Debug Options ───

# แสดง interrupts ที่ถูกเรียก
qemu-system-x86_64 -fda boot.bin -d int

# แสดง CPU resets
qemu-system-x86_64 -fda boot.bin -d cpu_reset

# ไม่ reboot เมื่อ crash
qemu-system-x86_64 -fda boot.bin -no-reboot -no-shutdown

# ─── GDB Debugging ───

# Terminal 1: เริ่ม QEMU พร้อม GDB server
qemu-system-x86_64 -fda boot.bin -s -S -no-reboot

# Terminal 2: เชื่อมต่อ GDB
gdb
(gdb) target remote localhost:1234
(gdb) set architecture i8086    # Real mode
(gdb) break *0x7c00             # Breakpoint ที่ MBR
(gdb) continue
(gdb) layout asm                # แสดง assembly
(gdb) info registers            # ดู registers

# ─── Serial Console ───

# Redirect serial port ไปที่ stdio
qemu-system-x86_64 -fda boot.bin -serial stdio

# Redirect ไปที่ file
qemu-system-x86_64 -fda boot.bin -serial file:serial.log

# ─── Display Options ───

# ไม่แสดง window (headless)
qemu-system-x86_64 -fda boot.bin -nographic

# VGA mode
qemu-system-x86_64 -fda boot.bin -vga std

# ─── Memory ───

# กำหนด RAM (ปกติ 128MB)
qemu-system-x86_64 -fda boot.bin -m 128M

# ─── BIOS ───

# ใช้ SeaBIOS (default)
qemu-system-x86_64 -fda boot.bin -bios /usr/share/seabios/bios.bin
```

### 15.2 ทดสอบ Boot Signature

```bash
# ตรวจสอบว่า boot.bin มี signature ถูกต้อง
python3 << 'EOF'
with open('boot.bin', 'rb') as f:
    data = f.read()

print(f"File size: {len(data)} bytes")
print(f"Last 2 bytes: {data[-2:].hex().upper()}")

if len(data) == 512 and data[510:512] == b'\x55\xaa':
    print("Boot signature: VALID (0x55AA)")
else:
    print("Boot signature: INVALID!")
    
# แสดง bytes ที่ 510-511
print(f"Byte 510: 0x{data[510]:02X}")
print(f"Byte 511: 0x{data[511]:02X}")
EOF
```

### 15.3 QEMU Monitor Commands

```
# ใน QEMU: กด Ctrl+Alt+2 เพื่อเข้า Monitor

QEMU Monitor Commands:
info registers      - แสดง CPU registers
info mem            - แสดง memory map
info tlb            - แสดง TLB
xp /20xb 0x7c00    - แสดง memory ที่ 0x7C00 (20 bytes hex)
x /10i  $pc        - แสดง 10 instructions ที่ PC
stop               - หยุด CPU
cont               - ต่อการทำงาน
step               - ทำ 1 instruction
quit               - ปิด QEMU
```

---

## 16. การ Debug Bootloader

### 16.1 GDB + QEMU Debugging

```bash
# สคริปต์ debug อัตโนมัติ
cat > debug.gdb << 'EOF'
# ตั้ง architecture เป็น 16-bit real mode
set architecture i8086

# เชื่อมต่อ QEMU GDB server
target remote localhost:1234

# ตั้ง breakpoint ที่ entry point MBR
break *0x7c00

# แสดง assembly layout
layout asm

# แสดง registers
layout regs

# ดำเนินการต่อ
continue
EOF

# เริ่ม QEMU ใน background
qemu-system-x86_64 -fda boot.bin -s -S -no-reboot &

# เริ่ม GDB
gdb -x debug.gdb
```

### 16.2 Useful GDB Commands

```
GDB Commands สำหรับ Bootloader:
─────────────────────────────────────────
stepi (si)          - Step 1 instruction
nexti (ni)          - Next (skip calls)
info reg            - แสดง registers
print $eax          - แสดงค่า register
x/10xb 0x7c00       - Examine memory
disas 0x7c00,+100   - Disassemble
break *0x7c50       - Set breakpoint
info breakpoints    - List breakpoints
delete 1            - Delete breakpoint 1
─────────────────────────────────────────
```

### 16.3 วิธีดู Memory ใน Real Mode

```
Real mode memory examination ใน GDB:
Physical address = segment × 16 + offset

ตัวอย่าง:
CS = 0x0000, IP = 0x7C00
Physical = 0x07C00

เพื่อดู memory ที่ 0x0000:0x7C00:
(gdb) x/20xb 0x7c00

เพื่อดู IVT (Interrupt Vector Table):
(gdb) x/20xb 0x0000

เพื่อดู BDA (BIOS Data Area):
(gdb) x/20xb 0x0400
```

---

## 17. Common Errors และวิธีแก้

### 17.1 Boot Loop / Reboot Loop

```
อาการ: QEMU reboot ซ้ำๆ
สาเหตุ: Boot signature ผิด หรือ CPU exception

การแก้:
1. ตรวจสอบ boot signature ใน MBR
   xxd boot.bin | tail -2

2. เพิ่ม -no-reboot flag ใน QEMU
   qemu-system-x86_64 -fda boot.bin -no-reboot

3. ตรวจสอบ code ว่ามี jmp $ เมื่อ done หรือเปล่า
```

### 17.2 Triple Fault

```
อาการ: CPU reset ทันที
สาเหตุ: Triple fault (3 exceptions ติดกัน)
         มักเกิดจาก: stack overflow, invalid segment, หรือ invalid GDT

การแก้:
1. ตรวจสอบ stack pointer
   - sp ต้องชี้ไปที่ valid memory
   - ต้องไม่ชี้ทับ code/data

2. ตรวจสอบ GDT (ถ้าเข้า Protected Mode)

3. Debug ด้วย: qemu-system-x86_64 -d cpu_reset -fda boot.bin
```

### 17.3 Disk Read Error

```
อาการ: Disk error ขณะโหลด Stage 2
สาเหตุ: DAP structure ผิด, หรือ sector ไม่มีข้อมูล

การแก้:
1. ตรวจสอบว่า disk image มี stage2.bin เขียนไว้ถูกต้อง
   dd if=stage2.bin of=disk.img bs=512 seek=1 conv=notrunc

2. ตรวจสอบ DAP structure
   - .size ต้องเป็น 0x10
   - .reserved ต้องเป็น 0

3. ลอง fallback ด้วย CHS
```

### 17.4 "No bootable device"

```
อาการ: BIOS แสดง "No bootable device"
สาเหตุ: Boot signature 0x55AA ไม่อยู่ที่ offset 510

การแก้:
1. ตรวจสอบ NASM directive
   TIMES 510 - ($ - $$) DB 0
   DW 0xAA55

2. ตรวจสอบว่า code ไม่เกิน 446 bytes (ถ้ามี partition table)
   หรือไม่เกิน 510 bytes (ถ้าไม่มี)

3. Verify ด้วย:
   python3 -c "d=open('boot.bin','rb').read(); print(hex(d[510]),hex(d[511]))"
   # ควรได้: 0x55 0xaa
```

---

## 18. ตัวอย่างการทำงานจริง

### 18.1 Minimal "Hello World" Bootloader

```nasm
; hello_boot.asm - Minimal Hello World Bootloader
; nasm -f bin hello_boot.asm -o hello_boot.bin
; qemu-system-x86_64 -fda hello_boot.bin

BITS 16
ORG 0x7C00

start:
    xor ax, ax
    mov ds, ax
    mov es, ax

    mov si, message
    call print

    jmp $           ; หยุดรอ

print:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    int 0x10
    jmp print
.done:
    ret

message: db 'Hello, World! This is my bootloader!', 0x0D, 0x0A, 0

TIMES 510 - ($ - $$) DB 0
DW 0xAA55
```

### 18.2 Bootloader พร้อม Keyboard Input

```nasm
; key_boot.asm - Bootloader ที่รับ keyboard input
; nasm -f bin key_boot.asm -o key_boot.bin

BITS 16
ORG 0x7C00

start:
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00

    mov si, prompt
    call print_str

.read_loop:
    mov ah, 0x00        ; Wait for key
    int 0x16
    ; AL = ASCII code, AH = scan code

    cmp al, 0x0D        ; Enter key?
    je .newline

    ; Echo the character
    mov ah, 0x0E
    int 0x10

    ; เก็บลง buffer (ง่ายๆ แค่ echo)
    jmp .read_loop

.newline:
    mov si, newline_str
    call print_str
    mov si, done_msg
    call print_str
    jmp $

print_str:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    xor bh, bh
    int 0x10
    jmp print_str
.done:
    ret

prompt:      db 'Type something: ', 0
newline_str: db 0x0D, 0x0A, 0
done_msg:    db 'You pressed Enter!', 0x0D, 0x0A, 0

TIMES 510 - ($ - $$) DB 0
DW 0xAA55
```

---

## 19. สรุปและ Build Instructions

### 19.1 Project Structure

```
bootloader/
├── boot.asm          # MBR Stage 1
├── stage2.asm        # Stage 2 Loader
├── kernel.asm        # Simple Kernel (32-bit)
├── Makefile          # Build system
└── README.md         # Documentation
```

### 19.2 Build Commands

```bash
# Compile ทุกไฟล์
make all

# หรือทีละไฟล์:
nasm -f bin boot.asm -o boot.bin
nasm -f bin stage2.asm -o stage2.bin
nasm -f bin kernel.asm -o kernel.bin

# สร้าง disk image:
dd if=/dev/zero of=disk.img bs=512 count=2880
dd if=boot.bin  of=disk.img bs=512 seek=0  conv=notrunc
dd if=stage2.bin of=disk.img bs=512 seek=1  conv=notrunc
dd if=kernel.bin of=disk.img bs=512 seek=17 conv=notrunc

# ตรวจสอบ:
xxd disk.img | head -32        # ดู MBR
xxd disk.img | grep -A2 "55aa" # หา boot signature
```

### 19.3 Test Commands

```bash
# ทดสอบพื้นฐาน
qemu-system-x86_64 -fda disk.img

# ทดสอบแบบ verbose
qemu-system-x86_64 -fda disk.img -d int,cpu_reset -no-reboot 2>&1 | tee qemu.log

# ทดสอบด้วย GDB
qemu-system-x86_64 -fda disk.img -s -S &
gdb -ex "target remote :1234" -ex "break *0x7c00" -ex "continue"

# ทดสอบ MBR เฉพาะ:
qemu-system-x86_64 -fda boot.bin   # ถ้า boot.bin = 512 bytes จะโหลดตรงๆ
```

### 19.4 NASM Build Options

```bash
# คำสั่ง NASM สำหรับ bootloader:
nasm -f bin boot.asm -o boot.bin        # สร้าง binary ตรง
nasm -f bin boot.asm -l boot.lst        # พร้อม listing file
nasm -f bin boot.asm -o boot.bin -E     # Preprocess only
nasm -f bin boot.asm -g                 # Debug info

# ตรวจสอบขนาด:
wc -c boot.bin          # ต้องได้ 512

# Disassemble:
ndisasm -b 16 -o 0x7C00 boot.bin
```

---

## 20. ข้อสรุป (Conclusion)

ในบทนี้เราได้เรียนรู้:

1. **BIOS POST Sequence**: กระบวนการเริ่มต้นของ BIOS ก่อนโหลด bootloader
2. **MBR Structure**: โครงสร้าง 512-byte MBR และ Boot Signature 0x55AA
3. **Real Mode**: การทำงาน 16-bit ใน 1MB address space
4. **BIOS INT 10h**: Video services สำหรับแสดงข้อความ
5. **BIOS INT 13h**: Disk services สำหรับ CHS และ LBA
6. **DAP Structure**: สำหรับ Extended Read ด้วย LBA
7. **A20 Gate**: การ enable access เกิน 1MB
8. **E820 Memory Map**: การอ่าน memory layout จาก BIOS
9. **2nd Stage Loader**: การแบ่ง bootloader เป็น 2 ส่วน
10. **GRUB Multiboot**: Multiboot header สำหรับ kernel

บทต่อไป (Part 082) จะพูดถึงการ Switch เข้า Protected Mode 32-bit และการตั้งค่า GDT/IDT

---

*เนื้อหาส่วนหนึ่งของ Assembly Language Course*
*Part 081: Bootloader Programming (MBR/BIOS)*
