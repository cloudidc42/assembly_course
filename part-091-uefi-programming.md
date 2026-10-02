# Part 091: UEFI Programming ใน Assembly

## บทนำ (Introduction)

UEFI (Unified Extensible Firmware Interface) คือมาตรฐานของ firmware สมัยใหม่ที่แทนที่ Legacy BIOS
ในฐานะ Assembly programmer การเข้าใจ UEFI เป็นสิ่งจำเป็นสำหรับการเขียน bootloader,
OS kernel, หรือ firmware utilities ที่ทำงานโดยตรงกับ hardware

บทนี้จะครอบคลุม:
- สถาปัตยกรรมของ UEFI และ boot sequence
- การเขียน UEFI application ด้วย NASM
- การใช้งาน EFI protocols
- การเขียน minimal UEFI bootloader

---

## 1. UEFI vs Legacy BIOS

### 1.1 ความแตกต่างหลัก

```
Legacy BIOS                     UEFI
─────────────────────────────────────────────────────
16-bit real mode               64-bit (UEFI 2.x native)
MBR partition table            GPT partition table
INT 13h disk services          EFI File System Protocol
INT 10h video services         Graphics Output Protocol
INT 16h keyboard               Simple Input Protocol
Limited memory map             Full memory map via GetMemoryMap
No security                    Secure Boot support
~1MB addressable               Full address space
C/H/S addressing               LBA addressing
Fixed boot sector at 0x7C00    PE/COFF executable (.efi)
No standard API                GUID-based protocol model
```

### 1.2 UEFI Boot Process (Phase Sequence)

UEFI มี boot sequence ที่ซับซ้อนกว่า BIOS มาก แบ่งเป็น 7 phase:

```
Power On
   │
   ▼
┌─────────────────────────────────────────────┐
│  Phase 1: SEC (Security)                    │
│  - CPU initialization                       │
│  - Cache-as-RAM setup                       │
│  - Verify firmware integrity               │
│  - Find and verify PEI Core               │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  Phase 2: PEI (Pre-EFI Initialization)      │
│  - Memory initialization                   │
│  - CPU/chipset early init                  │
│  - Find and hand off to DXE               │
│  - HOB (Hand-Off Block) list               │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  Phase 3: DXE (Driver Execution Environment)│
│  - Full memory available                   │
│  - Load DXE drivers                       │
│  - Build protocol database                │
│  - Platform initialization                │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  Phase 4: BDS (Boot Device Select)          │
│  - Load boot options                       │
│  - Attempt to boot devices                │
│  - Launch UEFI Shell / Boot Manager       │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  Phase 5: TSL (Transient System Load)       │
│  - OS Loader runs (e.g., GRUB, Windows BL) │
│  - Has access to Boot Services            │
│  - Calls ExitBootServices when ready      │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  Phase 6: RT (Runtime)                      │
│  - OS kernel runs                          │
│  - Only Runtime Services available        │
│  - Boot Services no longer accessible     │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  Phase 7: AL (After Life)                   │
│  - System shutdown/restart                 │
│  - ACPI S3/S4/S5 states                   │
└─────────────────────────────────────────────┘
```

**สิ่งสำคัญ:** UEFI Application ของเรา (bootloader) ทำงานใน **TSL phase**
เรามี Boot Services ทั้งหมด และต้อง call `ExitBootServices` ก่อนส่ง control ให้ OS

### 1.3 PE/COFF Format (ไม่ใช่ ELF!)

UEFI ใช้ **PE32+** (Portable Executable สำหรับ 64-bit) แทน ELF ที่ใช้บน Linux
นี่คือเหตุผลที่เราต้องใช้ `lld-link` หรือ `ld` พิเศษ

```
PE/COFF Structure:
┌──────────────────┐  offset 0x00
│  DOS Header      │  "MZ" signature (compat)
│  e_magic = 'MZ'  │
│  e_lfanew ────────┼──┐ pointer to PE header
├──────────────────┤  │
│  DOS Stub        │  │ (16-bit code ที่พิมพ์ "not for DOS")
├──────────────────┤◄─┘
│  PE Signature    │  "PE\0\0"
├──────────────────┤
│  COFF Header     │  Machine, NumberOfSections, etc.
├──────────────────┤
│  Optional Header │  ImageBase, Entry Point, etc.
│  (required!)     │  Magic = 0x20B (PE32+)
├──────────────────┤
│  Section Table   │  .text, .data, .rdata, ...
├──────────────────┤
│  .text section   │  executable code
├──────────────────┤
│  .data section   │  initialized data
└──────────────────┘
```

สำหรับ UEFI Application:
- `Machine` = `0x8664` (AMD64)
- `Subsystem` = `0x000A` (EFI Application)
- Entry point signature: `EFI_STATUS EFIAPI efi_main(EFI_HANDLE, EFI_SYSTEM_TABLE*)`

### 1.4 GUID-Based Protocol Model

UEFI ใช้ GUID (Globally Unique Identifier) เป็น key สำหรับ protocol ต่างๆ

```
GUID structure (16 bytes):
┌────────────────┬────────┬────────┬─────────────────┐
│  Data1 (4B)    │D2 (2B) │D3 (2B) │  Data4 (8B)     │
└────────────────┴────────┴────────┴─────────────────┘

ตัวอย่าง GUID ของ Graphics Output Protocol:
{0x9042A9DE, 0x23DC, 0x4A38, {0x96, 0xFB, 0x7A, 0xDE, 0xD0, 0x80, 0x51, 0x6A}}

ใน NASM:
GOP_GUID:
    dd 0x9042A9DE
    dw 0x23DC
    dw 0x4A38
    db 0x96, 0xFB, 0x7A, 0xDE, 0xD0, 0x80, 0x51, 0x6A
```

---

## 2. EFI System Table

### 2.1 EFI System Table Structure

เมื่อ UEFI เรียก `efi_main` ของเรา มันส่ง pointer ไปยัง `EFI_SYSTEM_TABLE`
นี่คือ "ประตู" สู่ทุก service ของ UEFI

```
EFI_SYSTEM_TABLE (offset ใน bytes สำหรับ 64-bit):
┌────────┬──────────────────────────────────────────────────┐
│ Offset │ Field                                            │
├────────┼──────────────────────────────────────────────────┤
│  0x00  │ Hdr.Signature     (UINT64) = 0x5453595320494249  │
│  0x08  │ Hdr.Revision      (UINT32) UEFI version          │
│  0x0C  │ Hdr.HeaderSize    (UINT32)                       │
│  0x10  │ Hdr.CRC32         (UINT32)                       │
│  0x14  │ Hdr.Reserved      (UINT32)                       │
├────────┼──────────────────────────────────────────────────┤
│  0x18  │ FirmwareVendor    (CHAR16*) "AMI", "Phoenix", etc│
│  0x20  │ FirmwareRevision  (UINT32)                       │
│  0x28  │ ConsoleInHandle   (EFI_HANDLE)                   │
│  0x30  │ ConIn             (EFI_SIMPLE_TEXT_INPUT*)       │
│  0x38  │ ConsoleOutHandle  (EFI_HANDLE)                   │
│  0x40  │ ConOut            (EFI_SIMPLE_TEXT_OUTPUT*)      │
│  0x48  │ StdErrHandle      (EFI_HANDLE)                   │
│  0x50  │ StdErr            (EFI_SIMPLE_TEXT_OUTPUT*)      │
│  0x58  │ RuntimeServices   (EFI_RUNTIME_SERVICES*)        │
│  0x60  │ BootServices      (EFI_BOOT_SERVICES*)           │
│  0x68  │ NumberOfTableEntries (UINTN)                     │
│  0x70  │ ConfigurationTable  (EFI_CONFIGURATION_TABLE*)   │
└────────┴──────────────────────────────────────────────────┘
```

### 2.2 การเข้าถึง System Table ใน NASM

```nasm
; ใน efi_main: RCX = ImageHandle, RDX = SystemTable*

; บันทึก SystemTable pointer
mov [SystemTable], rdx

; เข้าถึง ConOut (offset 0x40 จาก SystemTable)
mov rax, [rdx + 0x40]    ; rax = ConOut pointer
; rax ชี้ไปยัง EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL

; เข้าถึง BootServices (offset 0x60)
mov rax, [rdx + 0x60]    ; rax = BootServices pointer
```

### 2.3 EFI_TABLE_HEADER

```
EFI_TABLE_HEADER (ขนาด 24 bytes):
┌────────┬───────────────────────────────────────────┐
│ Offset │ ความหมาย                                  │
├────────┼───────────────────────────────────────────┤
│  +0    │ Signature (8 bytes) - magic number        │
│  +8    │ Revision (4 bytes) - 0x00020050 = v2.80  │
│  +12   │ HeaderSize (4 bytes) - size of header     │
│  +16   │ CRC32 (4 bytes) - checksum                │
│  +20   │ Reserved (4 bytes) - must be 0            │
└────────┴───────────────────────────────────────────┘

System Table Signature: 'I','B','I',' ','S','Y','S','T'
= 0x5453595320494249 (little endian)
```

---

## 3. EFI Boot Services

### 3.1 EFI_BOOT_SERVICES Table Layout

Boot Services table มี function pointer ทั้งหมด นี่คือ offset ที่สำคัญ:

```
EFI_BOOT_SERVICES (offset จาก BootServices pointer):
┌────────┬──────────────────────────────────────────────────┐
│ Offset │ Function                                         │
├────────┼──────────────────────────────────────────────────┤
│  0x00  │ Hdr (24 bytes header)                           │
│  0x18  │ RaiseTPL                                        │
│  0x20  │ RestoreTPL                                      │
│  0x28  │ AllocatePages                                   │
│  0x30  │ FreePages                                       │
│  0x38  │ GetMemoryMap                                    │
│  0x40  │ AllocatePool                                    │
│  0x48  │ FreePool                                        │
│  0x50  │ CreateEvent                                     │
│  0x58  │ SetTimer                                        │
│  0x60  │ WaitForEvent                                    │
│  0x68  │ SignalEvent                                     │
│  0x70  │ CloseEvent                                      │
│  0x78  │ CheckEvent                                      │
│  0x80  │ InstallProtocolInterface                        │
│  0x88  │ ReinstallProtocolInterface                      │
│  0x90  │ UninstallProtocolInterface                      │
│  0x98  │ HandleProtocol                                  │
│  0xA0  │ Reserved                                        │
│  0xA8  │ RegisterProtocolNotify                          │
│  0xB0  │ LocateHandle                                    │
│  0xB8  │ LocateDevicePath                               │
│  0xC0  │ InstallConfigurationTable                      │
│  0xC8  │ LoadImage                                       │
│  0xD0  │ StartImage                                      │
│  0xD8  │ Exit                                            │
│  0xE0  │ UnloadImage                                     │
│  0xE8  │ ExitBootServices                               │
│  0xF0  │ GetNextMonotonicCount                          │
│  0xF8  │ Stall                                           │
│  0x100 │ SetWatchdogTimer                               │
│  0x108 │ ConnectController                              │
│  0x110 │ DisconnectController                           │
│  0x118 │ OpenProtocol                                   │
│  0x120 │ CloseProtocol                                  │
│  0x128 │ OpenProtocolInformation                        │
│  0x130 │ ProtocolsPerHandle                             │
│  0x138 │ LocateHandleBuffer                             │
│  0x140 │ LocateProtocol                                 │
│  0x148 │ InstallMultipleProtocolInterfaces              │
│  0x150 │ UninstallMultipleProtocolInterfaces            │
│  0x158 │ CalculateCrc32                                 │
│  0x160 │ CopyMem                                        │
│  0x168 │ SetMem                                         │
│  0x170 │ CreateEventEx                                  │
└────────┴──────────────────────────────────────────────────┘
```

### 3.2 AllocatePool / FreePool

```nasm
; AllocatePool(PoolType, Size, Buffer*)
; RCX = PoolType (EfiLoaderData = 2)
; RDX = Size
; R8  = Buffer** (pointer to receive allocated address)
; Return: RAX = EFI_STATUS (0 = success)

section .data
    pool_ptr dq 0           ; จะเก็บ pointer ที่ได้

section .text
call_allocate_pool:
    push rbp
    mov rbp, rsp
    sub rsp, 32             ; shadow space for Win64 calling convention

    mov rax, [BootServices]
    mov rax, [rax + 0x40]  ; BootServices->AllocatePool

    mov rcx, 2             ; EfiLoaderData
    mov rdx, 4096          ; allocate 4KB
    lea r8, [pool_ptr]     ; buffer to receive pointer
    call rax

    ; rax = status, pool_ptr now contains allocated address
    add rsp, 32
    pop rbp
    ret

; FreePool(Buffer*)
; RCX = pointer to free
call_free_pool:
    sub rsp, 32
    mov rax, [BootServices]
    mov rax, [rax + 0x48]  ; BootServices->FreePool
    mov rcx, [pool_ptr]    ; pointer to free
    call rax
    add rsp, 32
    ret
```

### 3.3 AllocatePages / GetMemoryMap

```nasm
; AllocatePages(AllocateType, MemoryType, Pages, Memory*)
; สำหรับ bootloader: allocate pages สำหรับ kernel

; EFI_ALLOCATE_TYPE:
;   AllocateAnyPages = 0  (UEFI เลือก address ให้)
;   AllocateMaxAddress = 1 (ต่ำกว่า address ที่กำหนด)
;   AllocateAddress = 2   (กำหนด address เอง)

; EFI_MEMORY_TYPE:
;   EfiLoaderCode = 1
;   EfiLoaderData = 2
;   EfiBootServicesCode = 3
;   EfiBootServicesData = 4
;   EfiConventionalMemory = 7

section .data
    kernel_pages dq 256    ; 256 pages = 1MB
    kernel_addr  dq 0x100000  ; load at 1MB

allocate_kernel_memory:
    sub rsp, 32
    mov rax, [BootServices]
    mov rax, [rax + 0x28]   ; BootServices->AllocatePages

    mov rcx, 2              ; AllocateAddress
    mov rdx, 2              ; EfiLoaderData
    mov r8, [kernel_pages]  ; number of 4KB pages
    lea r9, [kernel_addr]   ; address (in/out)
    call rax
    add rsp, 32
    ret
```

### 3.4 GetMemoryMap

```nasm
; GetMemoryMap เป็น function สำคัญมากสำหรับ bootloader
; ต้อง call ก่อน ExitBootServices

; GetMemoryMap(MemoryMapSize*, MemoryMap*, MapKey*, DescriptorSize*, DescriptorVersion*)

section .data
    mmap_buffer     times 65536 db 0  ; 64KB buffer
    mmap_size       dq 65536
    mmap_key        dq 0
    mmap_desc_size  dq 0
    mmap_desc_ver   dq 0

get_memory_map:
    sub rsp, 40
    mov rax, [BootServices]
    mov rax, [rax + 0x38]   ; BootServices->GetMemoryMap

    lea rcx, [mmap_size]
    lea rdx, [mmap_buffer]
    lea r8, [mmap_key]
    lea r9, [mmap_desc_size]
    ; 5th parameter goes on stack (Win64 convention)
    lea rax, [mmap_desc_ver]
    mov [rsp + 32], rax
    call qword [BootServices + 0x38]
    add rsp, 40
    ret
```

### 3.5 LocateProtocol

```nasm
; LocateProtocol(GUID*, Registration*, Interface**)
; ใช้สำหรับหา protocol เช่น Graphics Output Protocol

; Graphics Output Protocol GUID:
GOP_GUID:
    dd 0x9042A9DE
    dw 0x23DC
    dw 0x4A38
    db 0x96, 0xFB, 0x7A, 0xDE, 0xD0, 0x80, 0x51, 0x6A

section .data
    gop_interface dq 0

locate_gop:
    sub rsp, 32
    mov rax, [BootServices]
    mov rax, [rax + 0x140]  ; BootServices->LocateProtocol

    lea rcx, [GOP_GUID]     ; GUID
    xor rdx, rdx            ; Registration = NULL
    lea r8, [gop_interface] ; will receive interface pointer
    call rax

    ; EFI_SUCCESS = 0, gop_interface now has GOP pointer
    add rsp, 32
    ret
```

---

## 4. Simple Text Output Protocol

### 4.1 EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL Structure

```
EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL:
┌────────┬──────────────────────────────────────────┐
│ Offset │ Function Pointer                         │
├────────┼──────────────────────────────────────────┤
│  0x00  │ Reset                                    │
│  0x08  │ OutputString                             │
│  0x10  │ TestString                               │
│  0x18  │ QueryMode                                │
│  0x20  │ SetMode                                  │
│  0x28  │ SetAttribute                             │
│  0x30  │ ClearScreen                              │
│  0x38  │ SetCursorPosition                        │
│  0x40  │ EnableCursor                             │
│  0x48  │ Mode (pointer to SIMPLE_TEXT_OUTPUT_MODE)│
└────────┴──────────────────────────────────────────┘
```

### 4.2 สำคัญมาก: UTF-16 Strings!

UEFI ใช้ **UTF-16LE** สำหรับ string ทั้งหมด ไม่ใช่ ASCII!
แต่ละ character คือ 2 bytes และต้อง null-terminate ด้วย 2 bytes (0x0000)

```nasm
; ใน NASM ให้ใช้ dw สำหรับ UTF-16 strings
section .data
hello_str:
    dw 'H', 'e', 'l', 'l', 'o', ',', ' ', 'W', 'o', 'r', 'l', 'd', '!', 13, 10, 0
    ; 13 = \r (carriage return), 10 = \n (line feed)
    ; UEFI ต้องการ \r\n ไม่ใช่แค่ \n

; หรือแบบสั้นกว่า:
newline_str:
    dw 13, 10, 0
```

### 4.3 OutputString

```nasm
; OutputString(This*, String*)
; RCX = ConOut pointer (This)
; RDX = String pointer (UTF-16)

print_string:
    ; ฟังก์ชันนี้รับ RCX = string pointer
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov rdx, rcx            ; string pointer → RDX
    mov rcx, [ConOut]       ; This pointer
    mov rax, [rcx + 0x08]   ; ConOut->OutputString function
    call rax

    add rsp, 32
    pop rbp
    ret
```

### 4.4 SetAttribute (Colors)

```nasm
; Color attributes ใน UEFI:
; Foreground colors: 0x00-0x0F
; Background colors: 0x00-0x07 (shifted left 4)

; EFI_BLACK        = 0x00
; EFI_BLUE         = 0x01
; EFI_GREEN        = 0x02
; EFI_CYAN         = 0x03
; EFI_RED          = 0x04
; EFI_MAGENTA      = 0x05
; EFI_BROWN        = 0x06
; EFI_LIGHTGRAY    = 0x07
; EFI_DARKGRAY     = 0x08 (foreground only)
; EFI_LIGHTBLUE    = 0x09 (foreground only)
; EFI_LIGHTGREEN   = 0x0A (foreground only)
; EFI_LIGHTCYAN    = 0x0B (foreground only)
; EFI_LIGHTRED     = 0x0C (foreground only)
; EFI_LIGHTMAGENTA = 0x0D (foreground only)
; EFI_YELLOW       = 0x0E (foreground only)
; EFI_WHITE        = 0x0F (foreground only)

; Background + Foreground:
; Attribute = (Background << 4) | Foreground
; เช่น เขียว บน น้ำเงิน = (1 << 4) | 2 = 0x12

set_color:
    ; RCX = foreground (0-15), RDX = background (0-7)
    push rbp
    mov rbp, rsp
    sub rsp, 32

    shl rdx, 4
    or rdx, rcx             ; attribute = (bg << 4) | fg
    mov rcx, [ConOut]       ; This
    mov r8, rdx             ; attribute
    mov rdx, rcx            ; This (shift)
    mov rcx, r8             ; attribute in proper position

    ; จริงๆ แล้ว: SetAttribute(This, Attribute)
    mov rax, [rdx + 0x28]   ; ConOut->SetAttribute
    ; ต้อง fix parameter order:
    ; rcx = This, rdx = Attribute
    add rsp, 32
    pop rbp
    ret
```

### 4.5 Complete Example: Colored Hello World (Code Example 1)

```nasm
; =============================================================
; uefi_hello.asm - UEFI Hello World พร้อมสี
; Build: nasm -f win64 uefi_hello.asm -o uefi_hello.obj
;        lld-link /subsystem:efi_application /entry:efi_main
;                 /out:BOOTX64.EFI uefi_hello.obj
; Test:  qemu-system-x86_64 -bios OVMF.fd -drive
;          format=raw,file=fat:rw:esp_dir
; =============================================================

default rel                      ; ใช้ RIP-relative addressing
bits 64

; UEFI Calling Convention (Microsoft x64):
; Parameters: RCX, RDX, R8, R9, then stack
; Return: RAX
; Shadow space: 32 bytes บน stack ก่อน call

; ─── EFI_SYSTEM_TABLE offsets ───
%define ST_ConOut      0x40
%define ST_BootServices 0x60

; ─── EFI_SIMPLE_TEXT_OUTPUT offsets ───
%define STO_Reset           0x00
%define STO_OutputString    0x08
%define STO_SetAttribute    0x28
%define STO_ClearScreen     0x30
%define STO_SetCursorPos    0x38

; ─── EFI Status codes ───
%define EFI_SUCCESS 0

section .data
    ; UTF-16LE strings
    str_clear:
        dw 27, '[', '2', 'J', 0   ; ANSI clear (may not work)
    str_hello:
        dw 'H','e','l','l','o',' ','f','r','o','m',' '
        dw 'U','E','F','I',' ','A','s','s','e','m','b','l','y','!',13,10,0
    str_line2:
        dw 'T','h','a','i',' ','A','s','s','e','m','b','l','y',' '
        dw 'C','o','u','r','s','e',13,10,0
    str_line3:
        dw 'P','a','r','t',' ','0','9','1',':',' ','U','E','F','I',13,10,0
    str_done:
        dw 'P','r','e','s','s',' ','a','n','y',' ','k','e','y','.','.','.',0

    ; Global variables
    gImageHandle dq 0
    gSystemTable dq 0

section .text

global efi_main

; ─────────────────────────────────────────────────
; efi_main(ImageHandle: RCX, SystemTable: RDX)
; Entry point ที่ UEFI firmware เรียก
; ─────────────────────────────────────────────────
efi_main:
    push rbp
    mov rbp, rsp
    sub rsp, 48                  ; shadow space + local vars
    and rsp, -16                 ; align stack to 16 bytes

    ; บันทึก parameters
    mov [gImageHandle], rcx
    mov [gSystemTable], rdx

    ; ── Step 1: Clear screen ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]   ; rax = ConOut*
    mov rcx, rax                 ; This = ConOut*
    mov rax, [rax + STO_ClearScreen]
    call rax                     ; ConOut->ClearScreen(This)

    ; ── Step 2: Set color (สีขาวบนน้ำเงิน) ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax                 ; This
    mov rdx, 0x1F                ; White(0x0F) on Blue(0x01) = 0x1F
    mov rax, [rcx + STO_SetAttribute]
    call rax

    ; ── Step 3: Print hello ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax                 ; This
    lea rdx, [str_hello]         ; String
    mov rax, [rcx + STO_OutputString]
    call rax

    ; ── Step 4: Change color (สีเหลืองบนดำ) ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    mov rdx, 0x0E                ; Yellow(0x0E) on Black(0x00) = 0x0E
    mov rax, [rcx + STO_SetAttribute]
    call rax

    ; ── Step 5: Print line 2 ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_line2]
    mov rax, [rcx + STO_OutputString]
    call rax

    ; ── Step 6: Print line 3 ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_line3]
    mov rax, [rcx + STO_OutputString]
    call rax

    ; ── Step 7: Reset color ──
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    mov rdx, 0x07                ; LightGray on Black
    mov rax, [rcx + STO_SetAttribute]
    call rax

    lea rdx, [str_done]
    mov rax, [gSystemTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    mov rax, [rcx + STO_OutputString]
    call rax

    ; ── Step 8: Wait for key ──
    call wait_for_key

    ; ── Return EFI_SUCCESS ──
    xor eax, eax                 ; EFI_SUCCESS = 0
    add rsp, 48
    pop rbp
    ret

; ─────────────────────────────────────────────────
; wait_for_key - รอการกดแป้น
; ─────────────────────────────────────────────────
wait_for_key:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    ; ConIn->Reset(ConIn, FALSE)
    mov rax, [gSystemTable]
    mov rax, [rax + 0x30]    ; SystemTable->ConIn (offset 0x30)
    mov rcx, rax
    xor rdx, rdx
    mov rax, [rcx]           ; ConIn->Reset
    call rax

    ; WaitForEvent([ConIn->WaitForKey])
    ; ConIn->WaitForKey อยู่ที่ offset 0x18 ของ ConIn
    ; BootServices->WaitForEvent(1, &event, &index)
    mov rax, [gSystemTable]
    mov rbx, [rax + 0x30]    ; ConIn
    lea rcx, [rbx + 0x18]    ; &ConIn->WaitForKey (event)

    mov rax, [gSystemTable]
    mov rax, [rax + ST_BootServices]
    mov rax, [rax + 0x60]    ; WaitForEvent
    mov rdx, 1               ; NumberOfEvents
    ; rcx = &event array (set above)
    ; need index pointer
    sub rsp, 8
    mov r8, rsp              ; &index
    call rax
    add rsp, 8

    add rsp, 48
    pop rbp
    ret
```

---

## 5. Graphics Output Protocol (GOP)

### 5.1 GOP Structure

```
EFI_GRAPHICS_OUTPUT_PROTOCOL:
┌────────┬──────────────────────────────────────────────┐
│ Offset │ Member                                       │
├────────┼──────────────────────────────────────────────┤
│  0x00  │ QueryMode(This*, ModeNumber, SizeOfInfo*,    │
│        │           Info**)                            │
│  0x08  │ SetMode(This*, ModeNumber)                   │
│  0x10  │ Blt(This*, BltBuffer*, BltOperation,         │
│        │    SourceX, SourceY, DestX, DestY,           │
│        │    Width, Height, Delta)                     │
│  0x18  │ Mode (pointer to EFI_GRAPHICS_OUTPUT_MODE)  │
└────────┴──────────────────────────────────────────────┘

EFI_GRAPHICS_OUTPUT_MODE:
┌────────┬──────────────────────────────────────────────┐
│ Offset │ Member                                       │
├────────┼──────────────────────────────────────────────┤
│  0x00  │ MaxMode (UINT32) - จำนวน modes ที่รองรับ   │
│  0x04  │ Mode (UINT32) - current mode number         │
│  0x08  │ Info (pointer to EFI_GRAPHICS_OUTPUT_MODE_  │
│        │       INFORMATION)                          │
│  0x10  │ SizeOfInfo (UINTN)                          │
│  0x18  │ FrameBufferBase (EFI_PHYSICAL_ADDRESS)      │
│  0x20  │ FrameBufferSize (UINTN) - bytes            │
└────────┴──────────────────────────────────────────────┘

EFI_GRAPHICS_OUTPUT_MODE_INFORMATION:
┌────────┬──────────────────────────────────────────────┐
│ Offset │ Member                                       │
├────────┼──────────────────────────────────────────────┤
│  0x00  │ Version (UINT32)                            │
│  0x04  │ HorizontalResolution (UINT32)               │
│  0x08  │ VerticalResolution (UINT32)                 │
│  0x0C  │ PixelFormat (EFI_GRAPHICS_PIXEL_FORMAT)     │
│  0x10  │ PixelInformation (EFI_PIXEL_BITMASK)        │
│  0x20  │ PixelsPerScanLine (UINT32)                  │
└────────┴──────────────────────────────────────────────┘
```

### 5.2 Pixel Format

```
EFI_GRAPHICS_PIXEL_FORMAT:
  PixelRedGreenBlueReserved8BitPerColor = 0  (RGBX)
  PixelBlueGreenRedReserved8BitPerColor = 1  (BGRX) ← ที่พบบ่อย
  PixelBitMask = 2
  PixelBltOnly = 3
  PixelFormatMax = 4

สำหรับ BGRX (format 1):
Byte 0: Blue
Byte 1: Green
Byte 2: Red
Byte 3: Reserved (= 0)

ตัวอย่าง pixel สีแดง (BGRX):
dw 0x00FF0000  ; R=FF, G=00, B=00, X=00? ไม่ถูก!
db 0x00, 0x00, 0xFF, 0x00  ; Blue=0, Green=0, Red=0xFF, Reserved=0 ✓
```

### 5.3 Blt Operations

```
EFI_BLT_OPERATION:
  EfiBltVideoFill = 0        ; เติมสีใน rectangle บน screen
  EfiBltVideoToBltBuffer = 1 ; copy จาก screen ไป buffer
  EfiBltBufferToVideo = 2    ; copy จาก buffer ไป screen
  EfiBltVideoToVideo = 3     ; copy ระหว่าง areas บน screen
```

### 5.4 GOP Drawing Example (Code Example 2)

```nasm
; =============================================================
; uefi_gop.asm - UEFI Graphics Drawing Demo
; วาด pixel และ rectangle บน screen โดยตรง
; =============================================================

default rel
bits 64

; GOP GUID
GOP_GUID:
    dd 0x9042A9DE
    dw 0x23DC
    dw 0x4A38
    db 0x96, 0xFB, 0x7A, 0xDE, 0xD0, 0x80, 0x51, 0x6A

; ─── Offsets ───
%define ST_BootServices  0x60
%define BS_LocateProtocol 0x140

%define GOP_QueryMode    0x00
%define GOP_SetMode      0x08
%define GOP_Blt          0x10
%define GOP_Mode         0x18

%define MODE_MaxMode     0x00
%define MODE_Mode        0x04
%define MODE_Info        0x08
%define MODE_FrameBuffer 0x18
%define MODE_FBSize      0x20

%define INFO_HorzRes     0x04
%define INFO_VertRes     0x08
%define INFO_PixelFormat 0x0C
%define INFO_PixPerScan  0x20

; BLT operations
%define EfiBltVideoFill   0
%define EfiBltBufferToVideo 2

section .data
    gSystemTable dq 0
    gGOP         dq 0
    gFBBase      dq 0
    gWidth       dd 0
    gHeight      dd 0
    gPixPerScan  dd 0
    gPixFormat   dd 0

    ; BLT pixel buffer (4 bytes per pixel, BGRX)
    blt_pixel:
        db 0, 0, 0, 0    ; BGRX placeholder

section .text
global efi_main

efi_main:
    push rbp
    mov rbp, rsp
    sub rsp, 64
    and rsp, -16

    mov [gSystemTable], rdx   ; Save SystemTable

    ; ── Locate GOP ──
    call locate_gop
    test rax, rax
    jnz .fail

    ; ── Get framebuffer info ──
    call get_fb_info

    ; ── Set graphics mode (mode 0 = default) ──
    ; (skip SetMode, use current mode)

    ; ── Fill screen with dark blue ──
    ; FillRect(0, 0, width, height, color=navy blue)
    mov dword [blt_pixel], 0x00200000  ; B=0x20, G=0, R=0, X=0
    movzx ecx, word [gWidth]
    movzx edx, word [gHeight]
    xor r8, r8          ; destX = 0
    xor r9, r9          ; destY = 0
    call fill_rect

    ; ── Draw red rectangle in center ──
    ; color = red (BGRX: B=0, G=0, R=0xFF, X=0)
    mov dword [blt_pixel], 0x00FF0000
    ; กำหนดตำแหน่งและขนาด
    xor r8, r8
    xor r9, r9
    ; ต้องคำนวณ center...
    ; สำหรับ demo: วาดที่ (100, 100) ขนาด 200x100
    call draw_red_box

    ; ── Write pixels directly to framebuffer ──
    call draw_gradient

    xor eax, eax
    add rsp, 64
    pop rbp
    ret

.fail:
    xor eax, eax
    add rsp, 64
    pop rbp
    ret

; ─────────────────────────────────────────────────
; locate_gop - หา Graphics Output Protocol
; Return: RAX = 0 on success
; ─────────────────────────────────────────────────
locate_gop:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    mov rax, [gSystemTable]
    mov rax, [rax + ST_BootServices]
    mov rax, [rax + BS_LocateProtocol]

    lea rcx, [GOP_GUID]
    xor rdx, rdx
    lea r8, [gGOP]
    call rax           ; BootServices->LocateProtocol(GUID, NULL, &gGOP)

    add rsp, 48
    pop rbp
    ret

; ─────────────────────────────────────────────────
; get_fb_info - อ่านข้อมูล framebuffer
; ─────────────────────────────────────────────────
get_fb_info:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov rax, [gGOP]
    mov rbx, [rax + GOP_Mode]   ; Mode pointer

    ; FrameBufferBase
    mov rcx, [rbx + MODE_FrameBuffer]
    mov [gFBBase], rcx

    ; Info pointer
    mov rsi, [rbx + MODE_Info]

    ; Width, Height
    mov eax, [rsi + INFO_HorzRes]
    mov [gWidth], eax
    mov eax, [rsi + INFO_VertRes]
    mov [gHeight], eax

    ; PixelsPerScanLine (may != Width due to padding)
    mov eax, [rsi + INFO_PixPerScan]
    mov [gPixPerScan], eax

    ; PixelFormat
    mov eax, [rsi + INFO_PixelFormat]
    mov [gPixFormat], eax

    add rsp, 32
    pop rbp
    ret

; ─────────────────────────────────────────────────
; fill_rect - เติมสีใน rectangle โดยใช้ Blt
; ─────────────────────────────────────────────────
fill_rect:
    ; ใช้ EfiBltVideoFill ผ่าน GOP->Blt
    push rbp
    mov rbp, rsp
    sub rsp, 80

    mov rax, [gGOP]
    mov r10, [rax + GOP_Blt]    ; Blt function pointer

    ; Blt(This, Buffer, Op, SrcX, SrcY, DstX, DstY, W, H, Delta)
    ; Win64: RCX=This, RDX=Buffer, R8=Op, R9=SrcX
    ; Stack: SrcY, DstX, DstY, W, H, Delta
    mov rcx, rax                ; This = GOP
    lea rdx, [blt_pixel]        ; Buffer
    mov r8, EfiBltVideoFill     ; Op
    xor r9, r9                  ; SrcX = 0

    xor rax, rax
    mov [rsp + 32], rax         ; SrcY = 0
    mov [rsp + 40], rax         ; DstX = 0
    mov [rsp + 48], rax         ; DstY = 0

    movzx rax, dword [gWidth]
    mov [rsp + 56], rax         ; Width
    movzx rax, dword [gHeight]
    mov [rsp + 64], rax         ; Height
    xor rax, rax
    mov [rsp + 72], rax         ; Delta = 0

    call r10

    add rsp, 80
    pop rbp
    ret

; ─────────────────────────────────────────────────
; draw_red_box - วาด rectangle สีแดงที่ (100,100)
; ─────────────────────────────────────────────────
draw_red_box:
    push rbp
    mov rbp, rsp
    sub rsp, 80

    ; Set color to red (BGRX: B=0, G=0, R=0xFF, X=0)
    mov dword [blt_pixel], 0x00FF0000

    mov rax, [gGOP]
    mov r10, [rax + GOP_Blt]

    mov rcx, rax                ; This
    lea rdx, [blt_pixel]        ; BltBuffer
    mov r8, EfiBltVideoFill     ; Operation
    xor r9, r9                  ; SrcX = 0

    xor rax, rax
    mov [rsp + 32], rax         ; SrcY = 0
    mov rax, 100
    mov [rsp + 40], rax         ; DstX = 100
    mov [rsp + 48], rax         ; DstY = 100
    mov rax, 200
    mov [rsp + 56], rax         ; Width = 200
    mov rax, 100
    mov [rsp + 64], rax         ; Height = 100
    xor rax, rax
    mov [rsp + 72], rax         ; Delta = 0

    call r10

    add rsp, 80
    pop rbp
    ret

; ─────────────────────────────────────────────────
; draw_gradient - เขียน pixel โดยตรงไปยัง framebuffer
; วาด gradient แนวนอน 256 pixels กว้าง
; ─────────────────────────────────────────────────
draw_gradient:
    push rbp
    mov rbp, rsp
    push rbx
    push r12
    push r13

    mov r12, [gFBBase]          ; framebuffer base address
    movzx r13, dword [gPixPerScan] ; pixels per scan line

    ; คำนวณ address ของ row 50
    ; address = FBBase + (y * PixPerScanLine + x) * 4
    mov rax, 50                 ; y = row 50
    imul rax, r13               ; y * stride
    add rax, 50                 ; + x = 50
    shl rax, 2                  ; * 4 bytes per pixel
    add r12, rax                ; r12 = address of pixel(50, 50)

    xor rbx, rbx                ; x counter = 0
.pixel_loop:
    cmp rbx, 256
    jge .done

    ; BGRX: write gradient (increasing blue)
    ; pixel = B=rbx, G=0, R=0, X=0
    mov [r12], bl               ; Blue = counter
    mov byte [r12 + 1], 0       ; Green = 0
    mov byte [r12 + 2], 0       ; Red = 0
    mov byte [r12 + 3], 0       ; Reserved = 0

    add r12, 4                  ; next pixel (4 bytes)
    inc rbx
    jmp .pixel_loop

.done:
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 6. Simple File System Protocol

### 6.1 Protocol Chain

```
EFI File System Access Chain:
┌─────────────────────────────────────────────────────┐
│ BootServices->LocateHandleBuffer(ByProtocol,         │
│     &SFSP_GUID, NULL, &count, &handles)             │
└──────────────────────┬──────────────────────────────┘
                       │ ได้ array ของ handles
                       ▼
┌─────────────────────────────────────────────────────┐
│ BootServices->OpenProtocol(handles[i], &SFSP_GUID,  │
│     &sfsp, ImageHandle, NULL,                       │
│     EFI_OPEN_PROTOCOL_GET_PROTOCOL)                 │
└──────────────────────┬──────────────────────────────┘
                       │ ได้ EFI_SIMPLE_FILE_SYSTEM_PROTOCOL*
                       ▼
┌─────────────────────────────────────────────────────┐
│ sfsp->OpenVolume(sfsp, &root)                       │
└──────────────────────┬──────────────────────────────┘
                       │ ได้ EFI_FILE_PROTOCOL* (root directory)
                       ▼
┌─────────────────────────────────────────────────────┐
│ root->Open(root, &file, L"\\path\\file.bin",        │
│            EFI_FILE_MODE_READ, 0)                   │
└──────────────────────┬──────────────────────────────┘
                       │ ได้ EFI_FILE_PROTOCOL* (file handle)
                       ▼
┌─────────────────────────────────────────────────────┐
│ file->Read(file, &size, buffer)                     │
│ file->Close(file)                                   │
└─────────────────────────────────────────────────────┘
```

### 6.2 EFI_FILE_PROTOCOL Structure

```
EFI_FILE_PROTOCOL (offsets):
┌────────┬──────────────────────────────────────────┐
│ Offset │ Function                                 │
├────────┼──────────────────────────────────────────┤
│  0x00  │ Revision (UINT64)                        │
│  0x08  │ Open(This, NewHandle, FileName, OpenMode,│
│        │      Attributes)                         │
│  0x10  │ Close(This)                              │
│  0x18  │ Delete(This)                             │
│  0x20  │ Read(This, BufferSize*, Buffer*)         │
│  0x28  │ Write(This, BufferSize*, Buffer*)        │
│  0x30  │ GetPosition(This, Position*)             │
│  0x38  │ SetPosition(This, Position)              │
│  0x40  │ GetInfo(This, InformationType*, BuffSize*│
│        │         Buffer*)                         │
│  0x48  │ SetInfo(This, InformationType*, BufSize, │
│        │         Buffer*)                         │
│  0x50  │ Flush(This)                              │
└────────┴──────────────────────────────────────────┘

Open Mode flags:
  EFI_FILE_MODE_READ   = 0x0000000000000001
  EFI_FILE_MODE_WRITE  = 0x0000000000000002
  EFI_FILE_MODE_CREATE = 0x8000000000000000
```

### 6.3 EFI_FILE_INFO Structure

```
EFI_FILE_INFO:
┌────────┬──────────────────────────────────────────┐
│ Offset │ Field                                    │
├────────┼──────────────────────────────────────────┤
│  0x00  │ Size (UINT64) - total size of this struct│
│  0x08  │ FileSize (UINT64) - bytes in file        │
│  0x10  │ PhysicalSize (UINT64)                    │
│  0x18  │ CreateTime (EFI_TIME, 16 bytes)          │
│  0x28  │ LastAccessTime (EFI_TIME, 16 bytes)      │
│  0x38  │ ModificationTime (EFI_TIME, 16 bytes)    │
│  0x48  │ Attribute (UINT64)                       │
│  0x50  │ FileName (CHAR16[]) - variable length!  │
└────────┴──────────────────────────────────────────┘

Attribute flags:
  EFI_FILE_READ_ONLY  = 0x0000000000000001
  EFI_FILE_HIDDEN     = 0x0000000000000002
  EFI_FILE_SYSTEM     = 0x0000000000000004
  EFI_FILE_DIRECTORY  = 0x0000000000000010
  EFI_FILE_ARCHIVE    = 0x0000000000000020
```

### 6.4 File Reading Example (Code Example 3)

```nasm
; =============================================================
; uefi_file.asm - อ่านไฟล์จาก ESP (EFI System Partition)
; =============================================================

default rel
bits 64

; Simple File System Protocol GUID
SFSP_GUID:
    dd 0x0964E5B22
    dw 0x6459
    dw 0x11D2
    db 0x8E, 0x39, 0x00, 0xA0, 0xC9, 0x69, 0x72, 0x3B

; File Info GUID (สำหรับ GetInfo)
EFI_FILE_INFO_GUID:
    dd 0x09576E92
    dw 0x6D3F
    dw 0x11D2
    db 0x8E, 0x39, 0x00, 0xA0, 0xC9, 0x69, 0x72, 0x3B

%define BS_OpenProtocol         0x118
%define BS_LocateHandleBuffer   0x138

%define SFSP_OpenVolume 0x08
%define FILE_Open       0x08
%define FILE_Close      0x10
%define FILE_Read       0x20
%define FILE_GetInfo    0x40

%define EFI_FILE_MODE_READ  1
%define EFI_OPEN_PROTOCOL_GET_PROTOCOL 2

section .data
    gImageHandle dq 0
    gSystemTable dq 0
    gBootSvc     dq 0
    gSFSP        dq 0
    gRootDir     dq 0
    gFileHandle  dq 0
    gHandleCount dq 0
    gHandleBuffer dq 0
    gFileBuffer  times 65536 db 0  ; 64KB buffer

    ; filename ใน UTF-16
    filename:
        dw '\','k','e','r','n','e','l','.','b','i','n',0

    ; file info buffer
    file_info_buf times 256 db 0
    file_info_size dq 256

section .text
global efi_main

efi_main:
    push rbp
    mov rbp, rsp
    sub rsp, 64
    and rsp, -16

    mov [gImageHandle], rcx
    mov [gSystemTable], rdx
    mov rax, [rdx + 0x60]   ; BootServices
    mov [gBootSvc], rax

    ; ── Step 1: หา handles ที่มี SimpleFileSystem protocol ──
    call find_fs_handle
    test rax, rax
    jnz .done

    ; ── Step 2: OpenVolume ──
    call open_volume
    test rax, rax
    jnz .done

    ; ── Step 3: Open file ──
    call open_file
    test rax, rax
    jnz .done

    ; ── Step 4: Read file ──
    call read_file

    ; ── Step 5: Close file ──
    mov rax, [gFileHandle]
    mov rcx, rax
    mov rax, [rcx + FILE_Close]
    call rax

.done:
    xor eax, eax
    add rsp, 64
    pop rbp
    ret

; ─────────────────────────────────────────────────
; find_fs_handle
; ─────────────────────────────────────────────────
find_fs_handle:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    mov rax, [gBootSvc]
    mov rax, [rax + BS_LocateHandleBuffer]

    ; LocateHandleBuffer(ByProtocol=2, GUID, NULL, Count*, Buffer**)
    mov rcx, 2              ; ByProtocol
    lea rdx, [SFSP_GUID]
    xor r8, r8              ; NULL
    lea r9, [gHandleCount]
    mov rax, [gBootSvc]
    mov rax, [rax + BS_LocateHandleBuffer]
    ; 5th parameter
    lea r10, [gHandleBuffer]
    mov [rsp + 32], r10
    call rax

    add rsp, 48
    pop rbp
    ret

; ─────────────────────────────────────────────────
; open_volume - เปิด volume แรก
; ─────────────────────────────────────────────────
open_volume:
    push rbp
    mov rbp, rsp
    sub rsp, 80

    ; OpenProtocol(handle, GUID, &interface, agent, NULL, attrs)
    mov rax, [gHandleBuffer]  ; handle buffer pointer
    mov rcx, [rax]            ; first handle

    mov rax, [gBootSvc]
    mov rax, [rax + BS_OpenProtocol]

    ; RCX = Handle (already set)
    lea rdx, [SFSP_GUID]
    lea r8, [gSFSP]           ; &interface
    mov r9, [gImageHandle]    ; AgentHandle
    mov rax, 0
    mov [rsp + 32], rax       ; ControllerHandle = NULL
    mov rax, EFI_OPEN_PROTOCOL_GET_PROTOCOL
    mov [rsp + 40], rax       ; Attributes
    mov rax, [gBootSvc]
    mov rax, [rax + BS_OpenProtocol]
    call rax

    test rax, rax
    jnz .fail

    ; SFSP->OpenVolume(SFSP, &RootDir)
    mov rax, [gSFSP]
    mov rcx, rax
    lea rdx, [gRootDir]
    mov rax, [rcx + SFSP_OpenVolume]
    call rax

.fail:
    add rsp, 80
    pop rbp
    ret

; ─────────────────────────────────────────────────
; open_file - เปิดไฟล์ kernel.bin
; ─────────────────────────────────────────────────
open_file:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    mov rax, [gRootDir]
    mov rcx, rax            ; This = RootDir
    lea rdx, [gFileHandle]  ; &NewHandle
    lea r8, [filename]      ; FileName (UTF-16)
    mov r9, EFI_FILE_MODE_READ  ; OpenMode = READ
    ; 5th param: Attributes = 0
    xor rax, rax
    mov [rsp + 32], rax
    mov rax, [gRootDir]
    mov rax, [rax + FILE_Open]
    call rax

    add rsp, 64
    pop rbp
    ret

; ─────────────────────────────────────────────────
; read_file - อ่านข้อมูลจากไฟล์
; ─────────────────────────────────────────────────
read_file:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    ; กำหนดขนาด buffer ก่อน
    mov qword [gHandleCount], 65536   ; ใช้ gHandleCount เป็น temp

    mov rax, [gFileHandle]
    mov rcx, rax            ; This
    lea rdx, [gHandleCount] ; &BufferSize (in/out)
    lea r8, [gFileBuffer]   ; Buffer
    mov rax, [rcx + FILE_Read]
    call rax

    ; rax = status, gHandleCount = bytes actually read

    add rsp, 48
    pop rbp
    ret
```

---

## 7. Writing a Complete UEFI Application in NASM

### 7.1 UEFI Calling Convention

UEFI ใช้ **Microsoft x64 calling convention**:

```
Parameters:
  1st: RCX
  2nd: RDX
  3rd: R8
  4th: R9
  5th+: บน stack (RSP+32, RSP+40, ...)

Stack:
  - ต้อง 16-byte aligned ก่อน call
  - ต้องจอง "shadow space" 32 bytes ก่อน call เสมอ
  - Caller saves: RAX, RCX, RDX, R8, R9, R10, R11
  - Callee saves: RBX, RBP, RDI, RSI, R12-R15, XMM6-XMM15

Return value: RAX
```

### 7.2 Stack Frame Setup

```nasm
; Pattern ที่ถูกต้องสำหรับ UEFI function call:
my_function:
    push rbp
    mov rbp, rsp
    sub rsp, 32          ; shadow space สำหรับ call ที่ไม่มี extra params
    and rsp, -16         ; align to 16 bytes (ทำที่ entry point)

    ; ... code ...

    ; Call UEFI function ที่มี <= 4 parameters:
    mov rcx, param1
    mov rdx, param2
    mov r8, param3
    mov r9, param4
    call [function_ptr]

    ; Call ที่มี 5+ parameters:
    sub rsp, 16          ; เพิ่ม stack space สำหรับ param5, param6
    mov rcx, param1
    mov rdx, param2
    mov r8, param3
    mov r9, param4
    mov [rsp + 32], param5   ; 5th param at RSP+32
    mov [rsp + 40], param6   ; 6th param at RSP+40
    call [function_ptr]
    add rsp, 16

    add rsp, 32
    pop rbp
    ret
```

### 7.3 Compilation and Testing

**วิธีที่ 1: ใช้ lld-link (LLVM)**

```bash
# ติดตั้ง tools
sudo apt install nasm lld

# Compile
nasm -f win64 hello_uefi.asm -o hello_uefi.obj

# Link เป็น UEFI Application
lld-link \
    /subsystem:efi_application \
    /entry:efi_main \
    /out:BOOTX64.EFI \
    hello_uefi.obj
```

**วิธีที่ 2: ใช้ GNU ld**

```bash
# ใช้ cross-linker
x86_64-w64-mingw32-ld \
    --subsystem=10 \
    --entry=efi_main \
    -o BOOTX64.EFI \
    hello_uefi.obj
```

**วิธีที่ 3: ใช้ GNU-EFI toolchain**

```bash
# ติดตั้ง
sudo apt install gnu-efi

# Compile object file
nasm -f elf64 hello_uefi.asm -o hello_uefi.o

# Link
ld -nostdlib -znocombreloc \
    -T /usr/lib/elf_x86_64_efi.lds \
    -shared \
    -Bsymbolic \
    /usr/lib/crt0-efi-x86_64.o \
    hello_uefi.o \
    /usr/lib/libgnuefi.a \
    /usr/lib/libefi.a \
    -o hello_uefi.so

# Convert to EFI
objcopy \
    -j .text -j .sdata -j .data -j .dynamic \
    -j .dynsym -j .rel -j .rela -j .reloc \
    --target=efi-app-x86_64 \
    hello_uefi.so BOOTX64.EFI
```

**ทดสอบด้วย QEMU + OVMF:**

```bash
# ดาวน์โหลด OVMF (UEFI firmware for QEMU)
sudo apt install ovmf

# สร้าง FAT32 disk image
mkdir -p esp/EFI/BOOT
cp BOOTX64.EFI esp/EFI/BOOT/

# รัน QEMU
qemu-system-x86_64 \
    -bios /usr/share/OVMF/OVMF_CODE.fd \
    -drive format=raw,file=fat:rw:esp \
    -net none \
    -nographic

# หรือใช้ OVMF แบบ split vars:
qemu-system-x86_64 \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=OVMF_VARS.fd \
    -drive format=raw,file=fat:rw:esp \
    -net none \
    -serial stdio
```

---

## 8. UEFI Bootloader for Custom OS

### 8.1 Bootloader Tasks

```
UEFI Bootloader ต้องทำ:
┌─────────────────────────────────────────────┐
│ 1. Locate kernel file on ESP               │
│    └── SimpleFileSystem->OpenVolume->Open  │
├─────────────────────────────────────────────┤
│ 2. Get kernel size (GetInfo)               │
│    └── EFI_FILE_INFO.FileSize              │
├─────────────────────────────────────────────┤
│ 3. Allocate memory for kernel              │
│    └── AllocatePages(AllocateAddress, ...)  │
├─────────────────────────────────────────────┤
│ 4. Read kernel into memory                 │
│    └── File->Read(handle, &size, buffer)   │
├─────────────────────────────────────────────┤
│ 5. Get memory map                          │
│    └── GetMemoryMap(...)                   │
├─────────────────────────────────────────────┤
│ 6. Exit Boot Services                      │
│    └── ExitBootServices(ImageHandle, key)  │
├─────────────────────────────────────────────┤
│ 7. Disable interrupts                      │
│    └── cli                                 │
├─────────────────────────────────────────────┤
│ 8. Setup minimal page tables (if needed)   │
│    └── CR3 = pml4_table                   │
├─────────────────────────────────────────────┤
│ 9. Jump to kernel                          │
│    └── jmp kernel_entry                    │
└─────────────────────────────────────────────┘
```

### 8.2 Complete Minimal Bootloader (Code Example 4)

```nasm
; =============================================================
; uefi_boot.asm - Minimal UEFI Bootloader
; โหลด kernel.bin จาก ESP และ jump ไปรัน
; =============================================================

default rel
bits 64

; ─── GUIDs ───
SFSP_GUID:
    dd 0x0964E5B22
    dw 0x6459
    dw 0x11D2
    db 0x8E, 0x39, 0x00, 0xA0, 0xC9, 0x69, 0x72, 0x3B

FILE_INFO_GUID:
    dd 0x09576E92
    dw 0x6D3F
    dw 0x11D2
    db 0x8E, 0x39, 0x00, 0xA0, 0xC9, 0x69, 0x72, 0x3B

; ─── Constants ───
%define KERNEL_LOAD_ADDR  0x100000   ; 1MB
%define KERNEL_MAX_PAGES  512        ; 2MB max

%define BS_AllocatePages  0x28
%define BS_GetMemoryMap   0x38
%define BS_LocateProtocol 0x140
%define BS_ExitBootServices 0xE8
%define BS_OpenProtocol   0x118
%define BS_LocateHandleBuffer 0x138

%define EFI_FILE_MODE_READ  1
%define EFI_OPEN_PROTOCOL_GET_PROTOCOL 2
%define AllocateAddress 2
%define EfiLoaderData   2

section .data align=16
    gImageHandle    dq 0
    gSystemTable    dq 0
    gBootSvc        dq 0
    gSFSP           dq 0
    gRoot           dq 0
    gKernelFile     dq 0
    gKernelAddr     dq KERNEL_LOAD_ADDR
    gHandleCount    dq 0
    gHandleBuffer   dq 0

    ; Memory map data
    mmap_buffer     times 65536 db 0
    mmap_size       dq 65536
    mmap_key        dq 0
    mmap_desc_size  dq 0
    mmap_desc_ver   dq 0

    ; File info buffer
    finfo_buf       times 512 db 0
    finfo_size      dq 512

    ; Kernel filename (UTF-16)
    kernel_filename:
        dw '\','k','e','r','n','e','l','.','b','i','n',0

    ; Boot info structure to pass to kernel
    boot_info:
        .mmap_addr    dq 0
        .mmap_size    dq 0
        .mmap_dsize   dq 0
        .fb_addr      dq 0
        .fb_width     dd 0
        .fb_height    dd 0
        .fb_stride    dd 0

section .text
global efi_main

; ─────────────────────────────────────────────────
; ENTRY POINT
; ─────────────────────────────────────────────────
efi_main:
    push rbp
    mov rbp, rsp
    sub rsp, 64
    and rsp, -16

    ; บันทึก parameters สำคัญ
    mov [gImageHandle], rcx
    mov [gSystemTable], rdx

    ; ดึง BootServices pointer
    mov rax, [rdx + 0x60]
    mov [gBootSvc], rax

    ; ─── 1. หา SimpleFileSystem ───
    call .find_sfsp
    test rax, rax
    jnz .halt

    ; ─── 2. OpenVolume ───
    call .open_root
    test rax, rax
    jnz .halt

    ; ─── 3. Open kernel file ───
    call .open_kernel
    test rax, rax
    jnz .halt

    ; ─── 4. Get kernel size ───
    call .get_kernel_size
    test rax, rax
    jnz .halt

    ; ─── 5. Allocate pages for kernel ───
    mov rax, [gBootSvc]
    mov rax, [rax + BS_AllocatePages]
    mov rcx, AllocateAddress    ; AllocateAddress
    mov rdx, EfiLoaderData      ; EfiLoaderData
    mov r8, KERNEL_MAX_PAGES    ; pages
    lea r9, [gKernelAddr]       ; &address
    call rax
    test rax, rax
    jnz .halt

    ; ─── 6. Read kernel into memory ───
    call .read_kernel
    test rax, rax
    jnz .halt

    ; ─── 7. Close file ───
    mov rax, [gKernelFile]
    mov rcx, rax
    mov rax, [rcx + 0x10]   ; FILE_Close
    call rax

    ; ─── 8. Get memory map ───
    call .get_mmap
    test rax, rax
    jnz .halt

    ; ─── 9. Populate boot_info ───
    lea rax, [mmap_buffer]
    mov [boot_info.mmap_addr], rax
    mov rax, [mmap_size]
    mov [boot_info.mmap_size], rax
    mov rax, [mmap_desc_size]
    mov [boot_info.mmap_dsize], rax

    ; ─── 10. ExitBootServices ───
    ; ต้อง retry หาก memory map เปลี่ยนระหว่างนี้
.exit_retry:
    mov rax, [gBootSvc]
    mov rax, [rax + BS_ExitBootServices]
    mov rcx, [gImageHandle]
    mov rdx, [mmap_key]
    call rax

    test rax, rax
    jnz .exit_retry_mmap    ; EFI_INVALID_PARAMETER = mmap stale, retry

    ; ─── 11. Jump to kernel ───
    ; ตอนนี้ Boot Services ไม่ใช้ได้แล้ว!
    cli                     ; disable interrupts

    ; ส่ง boot_info pointer ใน RDI (System V ABI สำหรับ kernel)
    lea rdi, [boot_info]
    mov rax, KERNEL_LOAD_ADDR
    jmp rax                 ; jump to kernel!

.exit_retry_mmap:
    ; อ่าน memory map ใหม่ แล้วลองอีกครั้ง
    mov qword [mmap_size], 65536
    call .get_mmap
    jmp .exit_retry

.halt:
    cli
    hlt
    jmp .halt

    add rsp, 64
    pop rbp
    ret

; ─── Helper: find SFSP ───
.find_sfsp:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    mov rax, [gBootSvc]
    mov rax, [rax + BS_LocateHandleBuffer]
    mov rcx, 2              ; ByProtocol
    lea rdx, [SFSP_GUID]
    xor r8, r8
    lea r9, [gHandleCount]
    lea r10, [gHandleBuffer]
    mov [rsp + 32], r10
    call rax

    add rsp, 48
    pop rbp
    ret

; ─── Helper: open root directory ───
.open_root:
    push rbp
    mov rbp, rsp
    sub rsp, 80

    ; ใช้ handle แรก
    mov rax, [gHandleBuffer]
    mov r11, [rax]          ; first handle

    mov rax, [gBootSvc]
    mov rax, [rax + BS_OpenProtocol]
    mov rcx, r11
    lea rdx, [SFSP_GUID]
    lea r8, [gSFSP]
    mov r9, [gImageHandle]
    xor rax, rax
    mov [rsp + 32], rax
    mov rax, EFI_OPEN_PROTOCOL_GET_PROTOCOL
    mov [rsp + 40], rax
    mov rax, [gBootSvc]
    mov rax, [rax + BS_OpenProtocol]
    call rax
    test rax, rax
    jnz .open_root_done

    ; OpenVolume
    mov rax, [gSFSP]
    mov rcx, rax
    lea rdx, [gRoot]
    mov rax, [rcx + 0x08]   ; SFSP->OpenVolume
    call rax

.open_root_done:
    add rsp, 80
    pop rbp
    ret

; ─── Helper: open kernel file ───
.open_kernel:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    mov rax, [gRoot]
    mov rcx, rax
    lea rdx, [gKernelFile]
    lea r8, [kernel_filename]
    mov r9, EFI_FILE_MODE_READ
    xor rax, rax
    mov [rsp + 32], rax
    mov rax, [gRoot]
    mov rax, [rax + 0x08]   ; FILE_Open
    call rax

    add rsp, 64
    pop rbp
    ret

; ─── Helper: get kernel file size ───
.get_kernel_size:
    push rbp
    mov rbp, rsp
    sub rsp, 64

    mov rax, [gKernelFile]
    mov rcx, rax
    lea rdx, [FILE_INFO_GUID]
    lea r8, [finfo_size]
    lea r9, [finfo_buf]
    mov rax, [rcx + 0x40]   ; FILE_GetInfo
    call rax

    ; finfo_buf + 0x08 = FileSize
    add rsp, 64
    pop rbp
    ret

; ─── Helper: read kernel ───
.read_kernel:
    push rbp
    mov rbp, rsp
    sub rsp, 48

    ; ใช้ finfo_buf + 8 เป็น read size
    ; (FileSize จาก GetInfo)
    mov rax, [gKernelFile]
    mov rcx, rax
    lea rdx, [finfo_buf + 8] ; &FileSize (บอกขนาดที่อยากอ่าน)
    mov r8, [gKernelAddr]    ; buffer = kernel load address
    mov rax, [rcx + 0x20]   ; FILE_Read
    call rax

    add rsp, 48
    pop rbp
    ret

; ─── Helper: GetMemoryMap ───
.get_mmap:
    push rbp
    mov rbp, rsp
    sub rsp, 56

    mov rax, [gBootSvc]
    mov rax, [rax + BS_GetMemoryMap]
    lea rcx, [mmap_size]
    lea rdx, [mmap_buffer]
    lea r8, [mmap_key]
    lea r9, [mmap_desc_size]
    lea r10, [mmap_desc_ver]
    mov [rsp + 32], r10
    call rax

    add rsp, 56
    pop rbp
    ret
```

---

## 9. Secure Boot Concepts

### 9.1 Secure Boot Chain

```
Secure Boot Key Hierarchy:
┌─────────────────────────────────────────────────────┐
│  Platform Key (PK)                                  │
│  - 1 key only (X.509 certificate)                  │
│  - Owner: OEM/manufacturer                         │
│  - Signs: KEK                                      │
└──────────────────────┬──────────────────────────────┘
                       │ signs
                       ▼
┌─────────────────────────────────────────────────────┐
│  Key Exchange Key (KEK)                             │
│  - Multiple keys allowed                           │
│  - Owners: OEM + OS vendors (Microsoft, etc.)      │
│  - Signs: db and dbx updates                       │
└──────────────────────┬──────────────────────────────┘
                       │ signs
                       ▼
┌─────────────────────────────────────────────────────┐
│  Signature Database (db)                            │
│  - List of trusted certificates/hashes             │
│  - UEFI apps must be signed by key in db           │
├─────────────────────────────────────────────────────┤
│  Forbidden Signature Database (dbx)                 │
│  - Revoked keys/hashes (blacklist)                 │
└─────────────────────────────────────────────────────┘

Boot Process with Secure Boot:
Firmware → verify bootloader signature against db
         → bootloader verifies kernel signature
         → kernel verifies drivers/modules
```

### 9.2 Signing UEFI Applications (Legitimate)

```bash
# สร้าง key pair ของตัวเอง
openssl req -newkey rsa:2048 -nodes -keyout mykey.key \
    -new -x509 -sha256 -days 3650 \
    -subj "/CN=My UEFI Key" -out mykey.crt

# Convert certificate
openssl x509 -outform DER -in mykey.crt -out mykey.der

# Sign UEFI application ด้วย sbsign
sbsign --key mykey.key --cert mykey.crt \
       --output BOOTX64.signed.EFI BOOTX64.EFI

# ตรวจสอบ signature
sbverify --cert mykey.crt BOOTX64.signed.EFI
```

### 9.3 Enrolling Custom Key (UEFI Setup)

```bash
# ใช้ efi-updatevar เพื่อลง key ของเราใน firmware

# สร้าง GUID สำหรับ key ของเรา
GUID=$(python3 -c "import uuid; print(uuid.uuid4())")

# แปลง certificate เป็น EFI signature list
cert-to-efi-sig-list -g "$GUID" mykey.crt mykey.esl

# Sign ด้วย PK เพื่อ update KEK
sign-efi-sig-list -g "$GUID" -k PK.key -c PK.crt KEK mykey.esl myKEK.auth

# Enroll (ต้องรันใน UEFI shell หรือ setup mode)
efi-updatevar -f myKEK.auth KEK
efi-updatevar -f mykey.auth db
```

---

## 10. Practical Examples

### 10.1 Memory Map Display (Code Example 5)

```nasm
; =============================================================
; uefi_mmap.asm - แสดง Memory Map ของระบบ
; เหมาะสำหรับ debug ก่อนเขียน memory manager
; =============================================================

default rel
bits 64

%define ST_ConOut    0x40
%define ST_BootSvc   0x60
%define STO_OutputString 0x08
%define BS_GetMemoryMap 0x38
%define BS_AllocatePool 0x40

; Memory type names (ASCII แล้วแปลงเป็น UTF-16 ตอน print)
; EFI Memory Types:
; 0  = EfiReservedMemoryType
; 1  = EfiLoaderCode
; 2  = EfiLoaderData
; 3  = EfiBootServicesCode
; 4  = EfiBootServicesData
; 5  = EfiRuntimeServicesCode
; 6  = EfiRuntimeServicesData
; 7  = EfiConventionalMemory   ← usable RAM
; 8  = EfiUnusableMemory
; 9  = EfiACPIReclaimMemory
; 10 = EfiACPIMemoryNVS
; 11 = EfiMemoryMappedIO
; 12 = EfiMemoryMappedIOPortSpace
; 13 = EfiPalCode
; 14 = EfiPersistentMemory

section .data
    gSysTable   dq 0
    gImgHandle  dq 0

    mmap_buf    times 65536 db 0
    mmap_size   dq 65536
    mmap_key    dq 0
    mmap_desc_size dq 0
    mmap_desc_ver  dq 0

    ; Output strings (UTF-16)
    str_header:
        dw 'M','e','m','o','r','y',' ','M','a','p',':',13,10,0
    str_fmt1:
        dw 'B','a','s','e',':',' ',0
    str_fmt2:
        dw ' ','S','i','z','e',':',' ',0
    str_fmt3:
        dw ' ','T','y','p','e',':',' ',0
    str_nl:
        dw 13,10,0
    str_sep:
        dw '-','-','-','-','-','-','-','-',13,10,0

    ; hex digits buffer (UTF-16, 17 chars + null)
    hex_buf:    times 36 db 0   ; 17 * 2 bytes

    hex_digits: db '0','1','2','3','4','5','6','7'
                db '8','9','A','B','C','D','E','F'

section .text
global efi_main

efi_main:
    push rbp
    mov rbp, rsp
    sub rsp, 64
    and rsp, -16

    mov [gImgHandle], rcx
    mov [gSysTable], rdx

    ; Print header
    call .print_header

    ; Get memory map
    call .get_mmap

    ; Iterate and print
    call .print_mmap

    xor eax, eax
    add rsp, 64
    pop rbp
    ret

.print_header:
    sub rsp, 32
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_header]
    mov rax, [rcx + STO_OutputString]
    call rax
    add rsp, 32
    ret

.get_mmap:
    sub rsp, 48
    mov rax, [gSysTable]
    mov rax, [rax + ST_BootSvc]
    lea rcx, [mmap_size]
    lea rdx, [mmap_buf]
    lea r8, [mmap_key]
    lea r9, [mmap_desc_size]
    lea r10, [mmap_desc_ver]
    mov [rsp + 32], r10
    mov rax, [gSysTable]
    mov rax, [rax + ST_BootSvc]
    mov rax, [rax + BS_GetMemoryMap]
    call rax
    add rsp, 48
    ret

.print_mmap:
    push rbx
    push r12
    push r13
    push r14
    push r15
    sub rsp, 32

    lea rbx, [mmap_buf]     ; current descriptor pointer
    mov r12, [mmap_size]    ; total map size
    mov r13, [mmap_desc_size] ; descriptor size
    xor r14, r14            ; offset counter

.mmap_loop:
    cmp r14, r12
    jge .mmap_done

    ; EFI_MEMORY_DESCRIPTOR layout (48 bytes):
    ; +0:  Type (UINT32)
    ; +4:  Padding (UINT32)
    ; +8:  PhysicalStart (UINT64)
    ; +16: VirtualStart (UINT64)
    ; +24: NumberOfPages (UINT64)
    ; +32: Attribute (UINT64)
    ; +40: Padding

    mov eax, [rbx + 0]      ; Type
    cmp eax, 7              ; EfiConventionalMemory
    jne .skip_entry

    ; Print base address
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_fmt1]
    mov rax, [rcx + STO_OutputString]
    call rax

    mov rax, [rbx + 8]      ; PhysicalStart
    call .print_hex64

    ; Print size
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_fmt2]
    mov rax, [rcx + STO_OutputString]
    call rax

    mov rax, [rbx + 24]     ; NumberOfPages
    shl rax, 12             ; * 4096 = bytes
    call .print_hex64

    ; Newline
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_nl]
    mov rax, [rcx + STO_OutputString]
    call rax

.skip_entry:
    add rbx, r13            ; next descriptor
    add r14, r13
    jmp .mmap_loop

.mmap_done:
    add rsp, 32
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    ret

; ─────────────────────────────────────────────────
; print_hex64 - print RAX เป็น hex (UTF-16)
; ─────────────────────────────────────────────────
.print_hex64:
    push rbx
    push rcx
    push rdx
    push r8
    sub rsp, 32

    ; สร้าง hex string ใน hex_buf
    ; "0x" + 16 hex digits + null
    lea rbx, [hex_buf]

    ; เขียน "0" ใน UTF-16
    mov word [rbx], '0'
    add rbx, 2
    mov word [rbx], 'x'
    add rbx, 2

    ; เขียน 16 hex digits (MSB first)
    mov rcx, 60             ; shift counter (start from 60)
    mov r8, rax             ; save value

.hex_loop:
    mov rax, r8
    shr rax, cl             ; shift right
    and rax, 0xF            ; get nibble
    lea rdx, [hex_digits]
    mov al, [rdx + rax]     ; get hex char
    mov [rbx], al           ; store ASCII
    mov byte [rbx + 1], 0   ; UTF-16 high byte
    add rbx, 2
    sub rcx, 4
    js .hex_done
    jmp .hex_loop

.hex_done:
    mov word [rbx], 0       ; null terminate

    ; Print hex_buf
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [hex_buf]
    mov rax, [rcx + STO_OutputString]
    call rax

    add rsp, 32
    pop r8
    pop rdx
    pop rcx
    pop rbx
    ret
```

### 10.2 Common Pitfalls และวิธีแก้

```
╔══════════════════════════════════════════════════════════════╗
║                    UEFI PITFALLS                            ║
╠══════════════════════════════════════════════════════════════╣
║                                                            ║
║  1. UTF-16 Strings                                         ║
║     ✗ WRONG: db "Hello", 0                                 ║
║     ✓ RIGHT: dw 'H','e','l','l','o',0                     ║
║              หรือ db 'H',0,'e',0,'l',0,'l',0,'o',0,0,0   ║
║                                                            ║
║  2. \r\n ไม่ใช่แค่ \n                                    ║
║     ✗ WRONG: dw 10, 0                                      ║
║     ✓ RIGHT: dw 13, 10, 0                                  ║
║                                                            ║
║  3. Shadow Space                                           ║
║     ✗ WRONG: call [fn_ptr]  ; ไม่ allocate shadow space   ║
║     ✓ RIGHT: sub rsp, 32                                   ║
║              call [fn_ptr]                                 ║
║              add rsp, 32                                   ║
║                                                            ║
║  4. Stack Alignment                                        ║
║     - ก่อน call ต้อง 16-byte aligned                      ║
║     - push rbp ทำให้ misaligned 8 bytes                   ║
║     - sub rsp, N ต้องให้ (RSP % 16) == 0 ก่อน call       ║
║                                                            ║
║  5. This pointer ใน Protocol calls                        ║
║     ✗ WRONG: call [STO->OutputString]  ; ไม่ส่ง This     ║
║     ✓ RIGHT: mov rcx, STO_ptr          ; This = protocol* ║
║              lea rdx, [string]         ; String           ║
║              call [rcx + STO_OutputString_offset]          ║
║                                                            ║
║  6. ExitBootServices ต้องใช้ MapKey ล่าสุด               ║
║     - MapKey เปลี่ยนทุกครั้งที่ memory map เปลี่ยน        ║
║     - ต้อง retry GetMemoryMap + ExitBootServices          ║
║     - หลัง ExitBootServices ห้ามใช้ Boot Services!        ║
║                                                            ║
║  7. PE/COFF vs ELF                                        ║
║     - UEFI ต้องการ PE32+ ไม่ใช่ ELF                      ║
║     - ใช้ -f win64 กับ nasm                               ║
║     - Link ด้วย lld-link ไม่ใช่ GNU ld โดยตรง            ║
║                                                            ║
║  8. Memory Regions หลัง ExitBootServices                  ║
║     - EfiBootServicesCode/Data: available (reclaim)       ║
║     - EfiRuntimeServicesCode/Data: ห้ามแตะ!              ║
║     - EfiConventionalMemory: ใช้ได้เต็มที่               ║
║                                                            ║
║  9. PixelFormat ใน GOP                                    ║
║     - อย่า assume BGRA เสมอ                               ║
║     - ตรวจสอบ Mode->Info->PixelFormat ก่อน               ║
║     - บางระบบเป็น RGBA บางระบบเป็น BGRA                   ║
║                                                            ║
║  10. GUID Layout                                          ║
║     - Data1 (4B): stored little-endian                    ║
║     - Data2, Data3 (2B each): stored little-endian        ║
║     - Data4 (8B): stored as-is (big-endian)               ║
║     ตัวอย่าง: {0x9042A9DE, 0x23DC, 0x4A38,              ║
║              {0x96,0xFB,0x7A,0xDE,0xD0,0x80,0x51,0x6A}} ║
╚══════════════════════════════════════════════════════════════╝
```

### 10.3 UEFI Shell Command (Code Example แบบง่าย)

```nasm
; =============================================================
; uefi_shell_cmd.asm - สร้าง UEFI Shell Application
; ทำงานเหมือน command line tool ภายใน UEFI Shell
; =============================================================

default rel
bits 64

%define ST_ConOut    0x40
%define ST_BootSvc   0x60
%define STO_OutputString 0x08
%define STO_ClearScreen  0x30

; System Info GUIDs
ACPI_GUID:
    dd 0xEB9D2D30
    dw 0x2D88
    dw 0x11D3
    db 0x9A, 0x16, 0x00, 0x90, 0x27, 0x3F, 0xC1, 0x4D

section .data
    gSysTable dq 0
    gImgHandle dq 0

    ; Application strings
    str_title:
        dw '=','=','=','=','=','=','=','=','=','=',13,10,0
    str_app_name:
        dw 'U','E','F','I',' ','S','y','s','t','e','m'
        dw ' ','I','n','f','o',13,10,0
    str_uefi_ver:
        dw 'U','E','F','I',' ','V','e','r','s','i','o','n',':',' ',0
    str_vendor:
        dw 'F','i','r','m','w','a','r','e',':',' ',0
    str_nl:
        dw 13,10,0
    ver_buf: times 32 db 0   ; version string buffer

section .text
global efi_main

efi_main:
    push rbp
    mov rbp, rsp
    sub rsp, 64
    and rsp, -16

    mov [gImgHandle], rcx
    mov [gSysTable], rdx

    ; Clear screen
    mov rax, [rdx + ST_ConOut]
    mov rcx, rax
    mov rax, [rcx + STO_ClearScreen]
    call rax

    ; Print title
    call .print_title

    ; Print UEFI version
    call .print_version

    ; Print firmware vendor
    call .print_vendor

    xor eax, eax
    add rsp, 64
    pop rbp
    ret

.print_title:
    sub rsp, 32
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_title]
    mov rax, [rcx + STO_OutputString]
    call rax

    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_app_name]
    mov rax, [rcx + STO_OutputString]
    call rax

    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_title]
    mov rax, [rcx + STO_OutputString]
    call rax

    add rsp, 32
    ret

.print_version:
    sub rsp, 32
    ; Print label
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_uefi_ver]
    mov rax, [rcx + STO_OutputString]
    call rax

    ; Print version number (from SystemTable->Hdr.Revision)
    ; Revision = 0x00020080 for UEFI 2.8.0
    ; = (Major << 16) | (Minor * 10 + patch)
    mov rax, [gSysTable]
    mov eax, [rax + 8]      ; Hdr.Revision (offset 8)

    ; แสดง major version
    mov ecx, eax
    shr ecx, 16             ; major
    ; ... (convert to UTF-16 number)

    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_nl]
    mov rax, [rcx + STO_OutputString]
    call rax

    add rsp, 32
    ret

.print_vendor:
    sub rsp, 32
    ; Print label
    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_vendor]
    mov rax, [rcx + STO_OutputString]
    call rax

    ; FirmwareVendor อยู่ที่ offset 0x18 ของ SystemTable
    ; เป็น CHAR16* อยู่แล้ว!
    mov rax, [gSysTable]
    mov rdx, [rax + 0x18]   ; FirmwareVendor (CHAR16*)

    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    ; rdx already has the string pointer
    mov rax, [rcx + STO_OutputString]
    call rax

    mov rax, [gSysTable]
    mov rax, [rax + ST_ConOut]
    mov rcx, rax
    lea rdx, [str_nl]
    mov rax, [rcx + STO_OutputString]
    call rax

    add rsp, 32
    ret
```

---

## สรุป UEFI Programming Checklist

```
┌─────────────────────────────────────────────────────────────┐
│           UEFI ASSEMBLY PROGRAMMING CHECKLIST               │
├─────────────────────────────────────────────────────────────┤
│ □ ใช้ `default rel` และ `bits 64` เสมอ                    │
│ □ Entry point: efi_main(RCX=ImageHandle, RDX=SystemTable)  │
│ □ บันทึก SystemTable pointer ทันที                        │
│ □ ทุก string เป็น UTF-16 (dw) และจบด้วย 0                │
│ □ Newline = dw 13, 10, 0 (\r\n)                           │
│ □ Shadow space 32 bytes ก่อนทุก call                      │
│ □ Stack 16-byte aligned ก่อน call                         │
│ □ Protocol calls ต้องส่ง This pointer เป็น RCX            │
│ □ ตรวจ EFI_STATUS return value (0 = success)              │
│ □ GetMemoryMap ก่อน ExitBootServices                      │
│ □ Retry ExitBootServices หาก map stale                    │
│ □ หลัง ExitBootServices: ห้าม print, ห้าม alloc          │
│ □ Build: nasm -f win64 + lld-link /subsystem:efi_app      │
│ □ Test: QEMU + OVMF.fd                                    │
│ □ Debug: เพิ่ม OutputString calls ดู progress             │
└─────────────────────────────────────────────────────────────┘
```

## Quick Reference: Useful UEFI GUIDs

```nasm
; Graphics Output Protocol
GOP_GUID:           dd 0x9042A9DE
                    dw 0x23DC, 0x4A38
                    db 0x96,0xFB,0x7A,0xDE,0xD0,0x80,0x51,0x6A

; Simple File System Protocol
SFSP_GUID:          dd 0x0964E5B22
                    dw 0x6459, 0x11D2
                    db 0x8E,0x39,0x00,0xA0,0xC9,0x69,0x72,0x3B

; EFI File Info
FILE_INFO_GUID:     dd 0x09576E92
                    dw 0x6D3F, 0x11D2
                    db 0x8E,0x39,0x00,0xA0,0xC9,0x69,0x72,0x3B

; ACPI Table GUID (ใน ConfigurationTable)
ACPI_20_GUID:       dd 0x8868E871
                    dw 0xE4F1, 0x11D3
                    db 0xBC,0x22,0x00,0x80,0xC7,0x3C,0x88,0x81

; SMBIOS Table GUID
SMBIOS_GUID:        dd 0xEB9D2D31
                    dw 0x2D88, 0x11D3
                    db 0x9A,0x16,0x00,0x90,0x27,0x3F,0xC1,0x4D

; Simple Network Protocol
SNP_GUID:           dd 0xA19832B9
                    dw 0xAC25, 0x11D3
                    db 0x9A,0x2D,0x00,0x90,0x27,0x3F,0xC1,0x4D
```

## Build Scripts

```makefile
# Makefile สำหรับ UEFI projects
NASM = nasm
NASM_FLAGS = -f win64
LLDLINK = lld-link
LLDLINK_FLAGS = /subsystem:efi_application /entry:efi_main

# QEMU
QEMU = qemu-system-x86_64
OVMF = /usr/share/OVMF/OVMF_CODE.fd
ESP_DIR = esp/EFI/BOOT

all: BOOTX64.EFI

%.obj: %.asm
	$(NASM) $(NASM_FLAGS) $< -o $@

BOOTX64.EFI: main.obj
	$(LLDLINK) $(LLDLINK_FLAGS) /out:$@ $<
	mkdir -p $(ESP_DIR)
	cp $@ $(ESP_DIR)/

run: BOOTX64.EFI
	$(QEMU) \
		-bios $(OVMF) \
		-drive format=raw,file=fat:rw:esp \
		-net none \
		-serial stdio \
		-m 256M

clean:
	rm -f *.obj BOOTX64.EFI
	rm -rf esp/EFI/BOOT/BOOTX64.EFI
```

## อ่านเพิ่มเติม

- **UEFI Specification**: https://uefi.org/specifications (ฟรี PDF)
- **TianoCore EDK2**: source code ของ OVMF และ reference implementation
- **OSDev Wiki UEFI**: https://wiki.osdev.org/UEFI
- **Bare Metal UEFI**: Andrei Warkentin's tutorials

---

*Part 091 - UEFI Programming ใน Assembly*
*Thai/English Mixed - Assembly Course*
