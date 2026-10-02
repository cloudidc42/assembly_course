# Part 085: Paging Implementation

## บทนำ (Introduction)

การ Paging เป็นกลไกสำคัญของ virtual memory ที่ทำให้ระบบปฏิบัติการสามารถให้แต่ละ process มี address space เป็นของตัวเอง และจัดการหน่วยความจำได้อย่างยืดหยุ่น บทนี้จะครอบคลุม implementation ของ paging ใน x86 ทั้ง 32-bit และ 64-bit ตั้งแต่พื้นฐานจนถึงการเขียน page table ใน NASM

---

## 1. Virtual Memory Concept

### 1.1 ทำไมต้องมี Virtual Memory?

ก่อน virtual memory ทุก process ต้องใช้ physical address โดยตรง มีปัญหาหลายอย่าง:

- Process หนึ่งอาจเขียนทับ memory ของอีก process
- ไม่สามารถ relocate code ได้อย่างอิสระ
- หน่วยความจำ fragment แบบแก้ยาก
- ไม่สามารถให้แต่ละ process เห็น address space เดียวกันได้

**Virtual Memory** แก้ปัญหาเหล่านี้โดย:
- แต่ละ process มี virtual address space เป็นของตัวเอง
- Hardware (MMU) แปลง virtual address → physical address โดยอัตโนมัติ
- OS ควบคุม mapping ผ่าน page tables
- Process แยกกัน, ปลอดภัย, สามารถ swap ไปดิสก์ได้

### 1.2 Virtual Address → Physical Address Translation

```
Virtual Address (VA)
        │
        ▼
┌───────────────┐
│     MMU       │  (Memory Management Unit)
│  (Hardware)   │
└───────┬───────┘
        │  ค้นหาใน Page Tables
        ▼
┌───────────────┐
│  Page Tables  │  (เก็บใน memory, ชี้โดย CR3)
└───────┬───────┘
        │
        ▼
Physical Address (PA)
        │
        ▼
┌───────────────┐
│  Physical RAM │
└───────────────┘
```

### 1.3 Page และ Frame

- **Page**: block ของ virtual memory ขนาด 4KB (หรือ 4MB สำหรับ large pages)
- **Frame**: block ของ physical memory ขนาดเดียวกัน
- **Page Table**: ตารางที่ map page → frame

สูตรการแปลง address:
```
Virtual Address  = Page Number   * 4096 + Page Offset
Physical Address = Frame Number  * 4096 + Page Offset

(Page Offset เหมือนกัน เพราะ offset ภายใน page ไม่เปลี่ยน)
```

---

## 2. x86 32-bit Two-Level Paging

### 2.1 โครงสร้าง 32-bit Virtual Address

ใน 32-bit mode virtual address มี 32 bits แบ่งเป็น 3 ส่วน:

```
31        22 21       12 11          0
┌──────────┬───────────┬─────────────┐
│  Dir(10) │  Table(10)│  Offset(12) │
└──────────┴───────────┴─────────────┘
     │            │            │
     │            │            └── Offset ภายใน page (0-4095)
     │            └─────────────── Index ใน Page Table (0-1023)
     └──────────────────────────── Index ใน Page Directory (0-1023)
```

- **Directory (bits 31-22)**: 10 bits = 1024 entries ใน Page Directory
- **Table (bits 21-12)**: 10 bits = 1024 entries ใน Page Table
- **Offset (bits 11-0)**: 12 bits = 4096 bytes ต่อ page

### 2.2 Two-Level Page Table Structure

```
CR3 Register
    │
    ▼ (Physical Address)
┌─────────────────────────────┐
│       Page Directory        │  4KB = 1024 × 4-byte PDE
│  PDE[0]   PDE[1]  ...       │
│  PDE[768] ...    PDE[1023]  │
└──────┬──────────────────────┘
       │  PDE[Dir] → Physical Address of Page Table
       ▼
┌─────────────────────────────┐
│        Page Table           │  4KB = 1024 × 4-byte PTE
│  PTE[0]  PTE[1]  ...        │
│  ...          PTE[1023]     │
└──────┬──────────────────────┘
       │  PTE[Table] → Physical Address of Page Frame
       ▼
┌─────────────────────────────┐
│        Page Frame           │  4KB Physical Memory
│  Bytes[0..4095]             │
└─────────────────────────────┘
```

ขนาดรวม:
- Page Directory: 1 table × 1024 entries × 4 bytes = **4KB**
- Page Tables: สูงสุด 1024 tables × 1024 entries × 4 bytes = **4MB** (ถ้า map ทุก page)
- ปกติไม่ต้อง allocate Page Table ที่ไม่ใช้ (demand paging)

### 2.3 Page Directory Entry (PDE)

```
31                    12 11 10  9  8  7  6  5  4  3  2  1  0
┌────────────────────────┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│  Physical Addr [31:12] │  │  │  │G │PS│  │A │PCD│PWT│U/S│R/W│P│
└────────────────────────┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
```

**PDE fields:**
| Bit | ชื่อ | ความหมาย |
|-----|------|----------|
| 0 | P (Present) | 1 = page table อยู่ใน memory, 0 = not present (page fault) |
| 1 | R/W | 1 = read/write, 0 = read-only |
| 2 | U/S (User/Supervisor) | 1 = user accessible, 0 = kernel only |
| 3 | PWT | Page-level Write-Through |
| 4 | PCD | Page-level Cache Disable |
| 5 | A (Accessed) | Hardware set เมื่อ access |
| 6 | สงวน | (ต้องเป็น 0) |
| 7 | PS (Page Size) | 0 = 4KB page table, 1 = 4MB large page |
| 8 | G (Global) | Global page (ไม่ flush TLB เมื่อ switch context) |
| 9-11 | Available | OS ใช้เองได้ |
| 12-31 | Physical Addr | Physical address ของ page table [31:12] |

### 2.4 Page Table Entry (PTE)

```
31                    12 11 10  9  8  7  6  5  4  3  2  1  0
┌────────────────────────┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│  Physical Addr [31:12] │  │  │  │G │PAT│D │A │PCD│PWT│U/S│R/W│P│
└────────────────────────┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
```

**PTE fields:**
| Bit | ชื่อ | ความหมาย |
|-----|------|----------|
| 0 | P (Present) | 1 = page อยู่ใน memory |
| 1 | R/W | Read/Write permission |
| 2 | U/S | User/Supervisor |
| 3 | PWT | Page Write-Through |
| 4 | PCD | Page Cache Disable |
| 5 | A (Accessed) | Set โดย hardware เมื่อ read/write |
| 6 | D (Dirty) | Set โดย hardware เมื่อ write |
| 7 | PAT | Page Attribute Table index |
| 8 | G (Global) | TLB global page |
| 9-11 | Available | OS ใช้เองได้ |
| 12-31 | Physical Addr | Physical address ของ page frame [31:12] |

---

## 3. Control Registers สำหรับ Paging

### 3.1 CR3: Page Directory Base Register

```
31                    12 11          5  4  3  2  1  0
┌────────────────────────┬────────────┬──┬──┬──┬──┬──┐
│  Page Dir Phys Addr    │  Reserved  │  │  │PCD│PWT│  │
│  [31:12]               │            │  │  │   │   │  │
└────────────────────────┴────────────┴──┴──┴──┴──┴──┘
```

- **Bits 31-12**: Physical address ของ Page Directory (ต้อง align 4KB)
- **Bit 4 (PCD)**: Page Cache Disable สำหรับ page directory
- **Bit 3 (PWT)**: Page Write-Through สำหรับ page directory

การโหลด CR3:
```nasm
; โหลด CR3 ด้วย physical address ของ page directory
mov eax, page_directory     ; physical address
mov cr3, eax                ; โหลดเข้า CR3
```

### 3.2 CR0: System Control Register

```
31  30  29  28  ...  18  17  16   ...   5   4   3   2   1   0
┌───┬───┬───┬───┬─────┬───┬───┬───────┬───┬───┬───┬───┬───┬───┐
│PG │CD │NW │  │ ... │AM │WP │ ...   │NE │ET │TS │EM │MP │PE │
└───┴───┴───┴───┴─────┴───┴───┴───────┴───┴───┴───┴───┴───┴───┘
```

**CR0 flags ที่สำคัญสำหรับ paging:**
| Bit | ชื่อ | ความหมาย |
|-----|------|----------|
| 0 | PE | Protected Mode Enable |
| 31 | PG | Paging Enable — **ต้อง set เพื่อเปิด paging** |

การเปิด Paging:
```nasm
; ต้องอยู่ใน protected mode (PE=1) ก่อน
mov eax, cr0
or  eax, 0x80000000     ; set bit 31 (PG)
mov cr0, eax            ; เปิด paging!
```

### 3.3 CR4: Extended Features Register

```
Bit 4: PSE (Page Size Extensions) — เปิดใช้ 4MB large pages
Bit 5: PAE (Physical Address Extension) — address 36-bit
Bit 7: PGE (Page Global Enable) — เปิดใช้ global pages
```

การเปิด PSE (4MB pages):
```nasm
mov eax, cr4
or  eax, 0x00000010     ; set bit 4 (PSE)
mov cr4, eax
```

---

## 4. Flag Constants ใน NASM

```nasm
; Page Table/Directory Entry Flags
PTE_PRESENT     equ 0x001   ; P bit: page present
PTE_WRITABLE    equ 0x002   ; R/W bit: writable
PTE_USER        equ 0x004   ; U/S bit: user accessible
PTE_PWT         equ 0x008   ; Page Write-Through
PTE_PCD         equ 0x010   ; Page Cache Disable
PTE_ACCESSED    equ 0x020   ; Accessed (set by hardware)
PTE_DIRTY       equ 0x040   ; Dirty (set by hardware on write)
PTE_LARGE       equ 0x080   ; PS: 4MB page (PDE only, needs CR4.PSE)
PTE_GLOBAL      equ 0x100   ; Global page (needs CR4.PGE)

; Common combinations
PTE_KERNEL_RW   equ (PTE_PRESENT | PTE_WRITABLE)
PTE_KERNEL_RO   equ (PTE_PRESENT)
PTE_USER_RW     equ (PTE_PRESENT | PTE_WRITABLE | PTE_USER)
PTE_USER_RO     equ (PTE_PRESENT | PTE_USER)
PTE_LARGE_RW    equ (PTE_PRESENT | PTE_WRITABLE | PTE_LARGE)
```

---

## 5. Identity Mapping: VA = PA สำหรับ First 4MB

### 5.1 ทำไมต้องทำ Identity Mapping?

เมื่อเราเปิด paging ระหว่าง boot:
1. CPU กำลัง execute code ที่ physical address X
2. ถ้าเราเปิด paging โดยไม่ map VA=PA, instruction ถัดไปจะ fault
3. ต้อง map VA=PA (identity map) สำหรับ region ที่กำลัง execute อยู่

```
Before paging:  CPU fetches instruction at physical 0x100000
After paging:   CPU fetches instruction at virtual 0x100000
                MMU ต้อง map VA 0x100000 → PA 0x100000
                มิฉะนั้น page fault!
```

### 5.2 Identity Map 4MB ด้วย Large Pages

วิธีง่ายที่สุดคือใช้ 4MB large page ใน PDE[0]:

```nasm
section .data
align 4096

; Page Directory: 1024 entries × 4 bytes = 4096 bytes
page_directory:
    ; Entry 0: Identity map first 4MB (VA 0x00000000-0x003FFFFF → PA same)
    dd 0x00000083       ; PA=0, P=1, R/W=1, PS=1 (4MB large page)
    times 1023 dd 0     ; entries 1-1023: not present
```

แต่ละ PDE ที่ 4MB large page:
```
Bits 31-22: Physical address bits 31-22 (= page frame number >> 10)
Bits 21-13: สงวน
Bit 12    : PAT
Bits 11-9 : Available for OS
Bit 8     : G (global)
Bit 7     : PS = 1 (4MB page)
Bit 6     : D (dirty)
Bit 5     : A (accessed)
Bit 4     : PCD
Bit 3     : PWT
Bit 2     : U/S
Bit 1     : R/W
Bit 0     : P

สำหรับ first 4MB (physical 0x00000000):
  Physical address[31:22] = 0
  PS = 1
  R/W = 1
  P = 1
  → 0x00000083
```

---

## 6. Kernel High-Half Mapping

### 6.1 Higher-Half Kernel Concept

Kernel ส่วนใหญ่ map ตัวเองที่ virtual address สูง (high half) เช่น 0xC0000000 (3GB) เพื่อ:
- แยก kernel space จาก user space
- User ได้ virtual address 0x00000000-0xBFFFFFFF (3GB)
- Kernel ได้ virtual address 0xC0000000-0xFFFFFFFF (1GB)

```
Virtual Address Space (32-bit):
0x00000000 ─────────────────────────── User Space
           │    User Code/Data/Stack   │
           │    (0 to 3GB)             │
0xBFFFFFFF ─────────────────────────── 
0xC0000000 ─────────────────────────── Kernel Space
           │    Kernel Code/Data       │
           │    (3GB to 4GB)           │
0xFFFFFFFF ─────────────────────────── 
```

### 6.2 Kernel High-Half Page Directory Setup

```nasm
; VA 0xC0000000 = Page Directory Index 768 (0xC00 >> 22 = 768)
; เราต้องการ map VA 0xC0000000 → PA 0x00000000

section .data
align 4096

page_directory:
    ; Entry 0: Identity map first 4MB (สำหรับ boot)
    dd 0x00000083       ; PA=0x00000000, Large 4MB, P, R/W
    
    times 767 dd 0      ; entries 1-767: not present
    
    ; Entry 768 (0x300): Kernel high-half map
    ; VA 0xC0000000-0xC03FFFFF → PA 0x00000000-0x003FFFFF
    dd 0x00000083       ; PA=0x00000000, Large 4MB, P, R/W
    
    times 255 dd 0      ; entries 769-1023: not present
```

### 6.3 Linker Script สำหรับ High-Half Kernel

```ld
/* kernel.ld */
ENTRY(kernel_main)

SECTIONS {
    /* Kernel ถูก link ที่ virtual 0xC0100000 */
    /* แต่ถูก load ที่ physical 0x00100000 */
    
    . = 0xC0100000;     /* Virtual address */
    
    _kernel_start = .;
    
    .text ALIGN(4096) : AT(ADDR(.text) - 0xC0000000) {
        *(.multiboot)
        *(.text)
    }
    
    .rodata ALIGN(4096) : AT(ADDR(.rodata) - 0xC0000000) {
        *(.rodata)
    }
    
    .data ALIGN(4096) : AT(ADDR(.data) - 0xC0000000) {
        *(.data)
    }
    
    .bss ALIGN(4096) : AT(ADDR(.bss) - 0xC0000000) {
        *(COMMON)
        *(.bss)
    }
    
    _kernel_end = .;
}
```

---

## 7. Complete 32-bit Paging Setup ใน NASM

### 7.1 Boot Code พร้อม Paging

```nasm
; boot.asm - Bootstrap code ที่เปิด paging และกระโดดไป high-half kernel
; สำหรับ Multiboot-compatible bootloader

bits 32

; Multiboot header constants
MULTIBOOT_MAGIC         equ 0x1BADB002
MULTIBOOT_FLAGS         equ 0x00000003   ; align modules + memory map
MULTIBOOT_CHECKSUM      equ -(MULTIBOOT_MAGIC + MULTIBOOT_FLAGS)

; Virtual/Physical offset
KERNEL_VIRTUAL_BASE     equ 0xC0000000
KERNEL_PHYSICAL_BASE    equ 0x00100000   ; 1MB

; Page flags
PAGE_PRESENT            equ 0x001
PAGE_WRITABLE           equ 0x002
PAGE_LARGE              equ 0x080        ; 4MB page (PSE)

section .multiboot
align 4
    dd MULTIBOOT_MAGIC
    dd MULTIBOOT_FLAGS
    dd MULTIBOOT_CHECKSUM

section .bss
align 4096

; Stack สำหรับ early boot (physical)
stack_bottom:
    resb 16384          ; 16KB initial stack
stack_top:

section .data
align 4096

; ─────────────────────────────────────────────────
; Page Directory (physical address ต้องรู้ตอน link)
; ─────────────────────────────────────────────────
boot_page_directory:
    ; [0]: Identity map 4MB (VA 0x00000000 = PA 0x00000000)
    ;      ต้องการเพื่อให้ code ทำงานได้หลัง enable paging
    dd (0x00000000 | PAGE_PRESENT | PAGE_WRITABLE | PAGE_LARGE)
    
    ; [1..767]: Not present
    times 767 dd 0
    
    ; [768] = VA 0xC0000000 → PA 0x00000000 (first 4MB of kernel)
    dd (0x00000000 | PAGE_PRESENT | PAGE_WRITABLE | PAGE_LARGE)
    
    ; [769] = VA 0xC0400000 → PA 0x00400000 (next 4MB)
    dd (0x00400000 | PAGE_PRESENT | PAGE_WRITABLE | PAGE_LARGE)
    
    ; [770..1023]: Not present  
    times 254 dd 0

section .text
global _start

_start:
    ; ตอนนี้อยู่ที่ physical address (Multiboot โหลด kernel ที่ 1MB)
    ; ESP ยังไม่ set, multiboot info อยู่ใน ebx
    
    ; บันทึก multiboot info pointer ไว้ใช้ทีหลัง
    push ebx

    ; ─── Step 1: เปิด PSE (4MB page support) ───
    mov eax, cr4
    or  eax, 0x00000010         ; Set CR4.PSE (bit 4)
    mov cr4, eax

    ; ─── Step 2: โหลด Page Directory ───
    ; boot_page_directory เป็น virtual address จาก linker
    ; ต้องแปลงเป็น physical address โดยลบ KERNEL_VIRTUAL_BASE
    mov eax, (boot_page_directory - KERNEL_VIRTUAL_BASE)
    mov cr3, eax

    ; ─── Step 3: เปิด Paging ───
    mov eax, cr0
    or  eax, 0x80000000         ; Set CR0.PG (bit 31)
    mov cr0, eax
    
    ; ─── ตอนนี้ paging เปิดแล้ว! ───
    ; CPU ยังอยู่ที่ low address (~0x100000) ซึ่ง identity map ไว้
    ; ต้อง jump ไป high-half virtual address
    
    lea eax, [higher_half]
    jmp eax                     ; Long jump ไป high-half

higher_half:
    ; ─── Step 4: Setup stack ที่ high virtual address ───
    mov esp, stack_top

    ; ─── Step 5: ยกเลิก Identity Map (optional แต่ดี) ───
    ; ลบ PDE[0] ออกเพื่อ catch null pointer dereference
    mov dword [boot_page_directory], 0
    
    ; Flush TLB
    mov eax, cr3
    mov cr3, eax

    ; ─── Step 6: เรียก kernel_main ───
    pop ebx                     ; multiboot info pointer
    push ebx
    push 0x2BADB002             ; multiboot magic (สำหรับ kernel ตรวจสอบ)
    
    extern kernel_main
    call kernel_main
    
    ; ถ้า kernel_main return (ไม่ควรเกิด)
.halt:
    cli
    hlt
    jmp .halt
```

### 7.2 4KB Page Tables (Non-Large Pages)

```nasm
; page_tables.asm - Setup 4KB page tables อย่างละเอียด
bits 32

PAGE_PRESENT    equ 0x001
PAGE_WRITABLE   equ 0x002
PAGE_USER       equ 0x004

section .bss
align 4096

; Page Directory: 1024 × 4 bytes = 4096 bytes
page_directory:
    resb 4096

; Page Table สำหรับ first 4MB: 1024 × 4 bytes = 4096 bytes
page_table_0:
    resb 4096

; Page Table สำหรับ kernel area (0xC0000000+): 
page_table_kernel:
    resb 4096

section .text

; ─────────────────────────────────────────────
; init_paging - ตั้งค่า page tables
; Input: ไม่มี
; Output: ไม่มี (เปิด paging)
; ─────────────────────────────────────────────
global init_paging
init_paging:
    push edi
    push ecx
    push eax
    
    ; ─── Clear page directory ───
    mov  edi, page_directory
    xor  eax, eax
    mov  ecx, 1024
    rep  stosd

    ; ─── Fill page_table_0: identity map 0x000000-0x3FFFFF ───
    ; แต่ละ entry: physical addr | flags
    mov  edi, page_table_0
    xor  eax, eax                   ; เริ่มที่ physical 0
.fill_identity:
    or   eax, (PAGE_PRESENT | PAGE_WRITABLE)    ; ใส่ flags
    stosd                           ; เขียน entry, edi += 4
    add  eax, 0x1000                ; physical address ของ page ถัดไป
    and  eax, ~0xFFF                ; clear flags (เหลือแต่ address)
    cmp  edi, page_table_0 + 4096
    jb   .fill_identity

    ; ─── Fill page_table_kernel: map 0xC0000000 → 0x000000 ───
    mov  edi, page_table_kernel
    xor  eax, eax
.fill_kernel:
    or   eax, (PAGE_PRESENT | PAGE_WRITABLE)
    stosd
    add  eax, 0x1000
    and  eax, ~0xFFF
    cmp  edi, page_table_kernel + 4096
    jb   .fill_kernel

    ; ─── ตั้งค่า Page Directory entries ───
    
    ; PDE[0] → page_table_0 (identity map)
    mov  eax, page_table_0
    or   eax, (PAGE_PRESENT | PAGE_WRITABLE)
    mov  [page_directory], eax

    ; PDE[768] (= 0xC0000000 >> 22) → page_table_kernel
    mov  eax, page_table_kernel
    or   eax, (PAGE_PRESENT | PAGE_WRITABLE)
    mov  [page_directory + 768 * 4], eax

    ; ─── โหลด CR3 ───
    mov  eax, page_directory
    mov  cr3, eax
    
    ; ─── เปิด Paging ───
    mov  eax, cr0
    or   eax, 0x80000000
    mov  cr0, eax

    pop  eax
    pop  ecx
    pop  edi
    ret
```

---

## 8. 4MB Large Pages (PSE)

### 8.1 Page Size Extension

4MB large pages ข้าม level ของ Page Table ไปเลย:

```
VA ─── Dir[10] → PDE → ถ้า PS=1, address ก็มาจาก PDE โดยตรง
                        ไม่ต้องผ่าน Page Table
```

**PDE สำหรับ 4MB Page:**
```
31          22  21      13  12  11  10  9   8   7   6   5   4   3   2   1   0
┌────────────┬───────────┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ Phys[31:22]│ Phys[39:32]│   │Rsv│PAT│Avl│ G │PS=1│ D │ A │PCD│PWT│U/S│R/W│ P │
└────────────┴───────────┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

- Bits 31-22: Physical address ของ 4MB frame (aligned to 4MB)
- Bits 20-13: Physical bits 39-32 (สำหรับ PAE mode)
- Bit 7: PS = 1 บอกว่าเป็น 4MB page

### 8.2 ตัวอย่าง: Map หลาย 4MB Regions

```nasm
; map_4mb_regions.asm
bits 32

PAGE_PRESENT    equ 0x001
PAGE_WRITABLE   equ 0x002
PAGE_LARGE      equ 0x080

section .data
align 4096

; Page Directory สำหรับ identity map 32MB แรก
pd_32mb:
    ; 8 entries × 4MB = 32MB identity map
    %assign i 0
    %rep 8
        dd ((i * 0x400000) | PAGE_PRESENT | PAGE_WRITABLE | PAGE_LARGE)
        %assign i i+1
    %endrep
    
    ; Entries 8-767: not present
    times 760 dd 0
    
    ; Entry 768: Kernel at 0xC0000000 → PA 0x00000000
    dd (0x00000000 | PAGE_PRESENT | PAGE_WRITABLE | PAGE_LARGE)
    
    ; Entry 769: 0xC0400000 → PA 0x00400000
    dd (0x00400000 | PAGE_PRESENT | PAGE_WRITABLE | PAGE_LARGE)
    
    ; Remaining entries: not present
    times 254 dd 0

section .text
global enable_paging_pse

enable_paging_pse:
    ; Enable PSE
    mov eax, cr4
    or  eax, 0x10
    mov cr4, eax
    
    ; Load CR3
    mov eax, pd_32mb
    mov cr3, eax
    
    ; Enable paging
    mov eax, cr0
    or  eax, 0x80000000
    mov cr0, eax
    
    ret
```

---

## 9. Page Fault Handler

### 9.1 CR2: Page Fault Linear Address Register

เมื่อเกิด page fault, CPU จะ:
1. Push error code onto stack
2. Push EIP (faulting instruction address)
3. Jump ไป IDT entry 14 (interrupt 0x0E)
4. **CR2 = virtual address ที่ทำให้เกิด fault**

### 9.2 Page Fault Error Code

```
Bits:  31...... 4   3   2   1   0
       ┌──────┬───┬───┬───┬───┬───┐
       │Reserv│ I │ R │ U │ W │ P │
       └──────┴───┴───┴───┴───┴───┘
```

| Bit | ชื่อ | 0 = | 1 = |
|-----|------|-----|-----|
| 0 | P | Page not present | Protection violation |
| 1 | W | Read access | Write access |
| 2 | U | Supervisor mode | User mode |
| 3 | R | - | Reserved bit set in page entry |
| 4 | I | - | Instruction fetch |

### 9.3 Page Fault Handler ใน NASM

```nasm
; page_fault.asm - Page Fault Handler (Interrupt 0x0E)
bits 32

section .text

extern handle_page_fault_c   ; C handler

; ─────────────────────────────────────────────
; page_fault_handler - ISR สำหรับ Page Fault
; Stack เมื่อเรียก:
;   [ESP+0]  = Error Code (push โดย CPU)
;   [ESP+4]  = EIP ที่ทำให้ fault
;   [ESP+8]  = CS
;   [ESP+12] = EFLAGS
;   [ESP+16] = ESP (ถ้า privilege change)
;   [ESP+20] = SS  (ถ้า privilege change)
; ─────────────────────────────────────────────
global page_fault_handler
page_fault_handler:
    ; เก็บ register ทุกตัว
    pushad              ; Push EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
    push ds
    push es
    push fs
    push gs
    
    ; อ่าน CR2 (faulting virtual address)
    mov  eax, cr2
    push eax            ; argument 1: faulting address
    
    ; Stack ตอนนี้:
    ; [ESP+0]  = faulting_address (CR2)
    ; [ESP+4]  = GS
    ; [ESP+8]  = FS
    ; [ESP+12] = ES
    ; [ESP+16] = DS
    ; [ESP+20] = PUSHAD (8 registers × 4 = 32 bytes)
    ; [ESP+52] = Error Code
    ; [ESP+56] = EIP (return address)
    
    ; อ่าน error code
    mov  eax, [esp + 56]    ; Error code
    push eax                ; argument 2: error code
    
    ; เรียก C handler
    call handle_page_fault_c
    add  esp, 8             ; ล้าง arguments
    
    ; Restore registers
    pop  gs
    pop  fs
    pop  es
    pop  ds
    popad
    
    add  esp, 4             ; ล้าง error code ออกจาก stack
    iret                    ; Return from interrupt

; ─────────────────────────────────────────────
; C handler (สำหรับอ้างอิง):
; void handle_page_fault_c(uint32_t fault_addr, uint32_t error_code)
; {
;     if (!(error_code & 0x1)) {
;         // Page not present — allocate and map
;         alloc_and_map_page(fault_addr);
;     } else if (error_code & 0x2) {
;         // Write to read-only page — access violation
;         kernel_panic("Write to read-only page at 0x%x", fault_addr);
;     } else {
;         // Other protection violation
;         kernel_panic("Page fault at 0x%x, error=0x%x", fault_addr, error_code);
;     }
; }
; ─────────────────────────────────────────────
```

### 9.4 IDT Setup สำหรับ Page Fault

```nasm
; idt.asm - Interrupt Descriptor Table
bits 32

; IDT Entry structure (8 bytes):
; Bits 63-48: offset[31:16]
; Bits 47-32: type and attributes
; Bits 31-16: segment selector
; Bits 15-0:  offset[15:0]

%macro IDT_ENTRY 2      ; %1=handler, %2=type_attr
    dw (%1 & 0xFFFF)            ; offset[15:0]
    dw 0x08                     ; kernel code segment
    db 0                        ; reserved
    db %2                       ; type/attr
    dw ((%1 >> 16) & 0xFFFF)   ; offset[31:16]
%endmacro

; Type attributes:
; 0x8E = 10001110 = Present, Ring 0, 32-bit interrupt gate
; 0x8F = 10001111 = Present, Ring 0, 32-bit trap gate

section .data
align 8

idt_table:
    times 14 dq 0               ; Interrupts 0-13: ยังไม่ตั้งค่า
    IDT_ENTRY page_fault_handler, 0x8E   ; Interrupt 14: page fault
    times (256-15) dq 0         ; Interrupts 15-255

idt_pointer:
    dw (256 * 8 - 1)            ; limit
    dd idt_table                ; base address

section .text
global load_idt

load_idt:
    lidt [idt_pointer]
    ret
```

---

## 10. TLB Invalidation

### 10.1 Translation Lookaside Buffer (TLB)

TLB คือ cache ของ page table lookups ใน CPU:
- ทุกครั้งที่ MMU แปลง VA→PA, มันเก็บผลลัพธ์ใน TLB
- ครั้งต่อไปที่ access VA เดิม, ใช้ TLB แทนการอ่าน page table อีกครั้ง
- เร็วกว่ามาก (TLB hit ~1 cycle vs memory access ~100+ cycles)

**ปัญหา**: ถ้าเราแก้ page table แต่ TLB ยังเก็บ entry เก่า → access ผิด

### 10.2 INVLPG: Invalidate Single Page

```nasm
; Invalidate TLB entry สำหรับ virtual address เดียว
; Syntax: INVLPG [mem]

; ตัวอย่าง: invalidate page ที่ VA 0xC0001000
invlpg [0xC0001000]

; กรณีทั่วไปที่ VA อยู่ใน register
; eax = virtual address ที่ต้องการ invalidate
invlpg [eax]

; ตัวอย่างใน C-callable function:
; void invlpg(uint32_t vaddr)
global tlb_invlpg
tlb_invlpg:
    mov  eax, [esp + 4]     ; vaddr argument
    invlpg [eax]
    ret
```

### 10.3 MOV CR3: Flush Entire TLB

```nasm
; Flush ทุก TLB entry (ยกเว้น global pages)
; วิธีที่ 1: เขียน CR3 ด้วยค่าเดิม
global tlb_flush_all
tlb_flush_all:
    mov  eax, cr3
    mov  cr3, eax       ; Re-load CR3 → flush all non-global TLB entries
    ret

; วิธีที่ 2: เปลี่ยน page directory
; (ใช้ตอน context switch)
global switch_page_directory
switch_page_directory:
    mov  eax, [esp + 4]     ; physical address ของ page directory ใหม่
    mov  cr3, eax           ; Load CR3 ใหม่ → flush TLB อัตโนมัติ
    ret
```

### 10.4 Global Pages และ CR4.PGE

```nasm
; Global pages ไม่ถูก flush เมื่อ MOV CR3
; ใช้สำหรับ kernel pages ที่ทุก process share กัน

; เปิด PGE feature
enable_pge:
    mov  eax, cr4
    or   eax, 0x80          ; Set CR4.PGE (bit 7)
    mov  cr4, eax
    ret

; Flush global pages ด้วย: toggle CR4.PGE
flush_all_including_global:
    mov  eax, cr4
    and  eax, ~0x80         ; Clear PGE
    mov  cr4, eax
    or   eax, 0x80          ; Set PGE again
    mov  cr4, eax
    ret
```

---

## 11. x86-64 Four-Level Paging

### 11.1 64-bit Virtual Address Structure

ใน x86-64 virtual address มี 48 bits ที่ใช้งานจริง (bits 0-47) แบ่งเป็น 5 ส่วน:

```
63  48 47    39 38    30 29    21 20    12 11          0
┌──────┬────────┬────────┬────────┬────────┬────────────┐
│Sign  │ PML4(9)│ PDPT(9)│  PD(9) │  PT(9) │ Offset(12) │
│Ext.  │        │        │        │        │            │
└──────┴────────┴────────┴────────┴────────┴────────────┘
```

- **Bits 63-48**: Sign extension (ต้องเหมือน bit 47)
- **Bits 47-39**: PML4 index (9 bits = 512 entries)
- **Bits 38-30**: PDPT (Page Directory Pointer Table) index
- **Bits 29-21**: PD (Page Directory) index
- **Bits 20-12**: PT (Page Table) index
- **Bits 11-0**: Offset ภายใน page (4KB)

48-bit virtual address space = 2^48 = 256TB ต่อ process

### 11.2 Four-Level Structure

```
CR3
 │
 ▼ (Physical Address)
┌─────────────────────────────┐
│    PML4 (512 entries)       │  4KB
└──────────────┬──────────────┘
               │ PML4E → Physical addr of PDPT
               ▼
┌─────────────────────────────┐
│    PDPT (512 entries)       │  4KB
└──────────────┬──────────────┘
               │ PDPTE → Physical addr of PD
               ▼
┌─────────────────────────────┐
│     PD (512 entries)        │  4KB
└──────────────┬──────────────┘
               │ PDE → Physical addr of PT
               ▼
┌─────────────────────────────┐
│     PT (512 entries)        │  4KB
└──────────────┬──────────────┘
               │ PTE → Physical addr of Page
               ▼
┌─────────────────────────────┐
│      Page Frame             │  4KB
└─────────────────────────────┘
```

### 11.3 x86-64 Page Table Entry (64-bit)

```
63  52 51     12 11  9  8  7  6  5  4  3  2  1  0
┌─────┬────────┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ NX/R│Phys[51:12]│Avl│ G │PAT│ D │ A │PCD│PWT│U/S│R/W│ P │
└─────┴────────┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

- **Bit 63 (NX/XD)**: No-Execute bit — ถ้า set ห้าม execute code ใน page นี้
- **Bits 51-12**: Physical address (52-bit physical address space)
- **Bits 11-9**: Available for OS
- **Bit 8 (G)**: Global
- **Bit 7 (PS/PAT)**: Page Size (ใน PD/PDPT) หรือ PAT
- **Bits 6-0**: เหมือน 32-bit

### 11.4 x86-64 Page Table ใน NASM

```nasm
; paging64.asm - 64-bit paging setup
bits 32          ; เริ่มใน 32-bit mode (จาก bootloader)

; Page table flags
PTE64_PRESENT   equ (1 << 0)
PTE64_WRITABLE  equ (1 << 1)
PTE64_USER      equ (1 << 2)
PTE64_HUGE      equ (1 << 7)    ; 2MB page ใน PT, 1GB ใน PDPT
PTE64_NX        equ (1 << 63)   ; No-execute

section .bss
align 4096

; Four-level page tables (ขนาดละ 4KB)
pml4_table:     resb 4096
pdpt_table:     resb 4096
pd_table:       resb 4096
pt_table:       resb 4096

section .text
global setup_paging_64

; ─────────────────────────────────────────────
; setup_paging_64 - ตั้งค่า page tables สำหรับ long mode
; Identity map first 2MB ด้วย 2MB huge pages
; ─────────────────────────────────────────────
setup_paging_64:
    ; ─── Clear all page tables ───
    mov edi, pml4_table
    xor eax, eax
    mov ecx, 4096               ; 4 tables × 1024 DWORDs
    rep stosd

    ; ─── PML4[0] → PDPT ───
    mov eax, pdpt_table
    or  eax, (PTE64_PRESENT | PTE64_WRITABLE)
    mov [pml4_table], eax
    mov dword [pml4_table + 4], 0   ; Upper 32 bits = 0

    ; ─── PDPT[0] → PD ───
    mov eax, pd_table
    or  eax, (PTE64_PRESENT | PTE64_WRITABLE)
    mov [pdpt_table], eax
    mov dword [pdpt_table + 4], 0

    ; ─── PD[0] → 2MB Huge Page (PA 0x000000) ───
    ; ใช้ 2MB huge page แทน PT เพื่อความง่าย
    mov dword [pd_table], (PTE64_PRESENT | PTE64_WRITABLE | PTE64_HUGE)
    mov dword [pd_table + 4], 0

    ; ─── โหลด CR3 ───
    mov eax, pml4_table
    mov cr3, eax

    ret
```

---

## 12. Long Mode Transition: Setup Paging → Enter Long Mode

### 12.1 ขั้นตอน Enter Long Mode

1. Disable paging (ถ้าเปิดอยู่)
2. Enable PAE (CR4.PAE bit 5)
3. Setup PML4 page tables
4. Load CR3 ด้วย PML4 base
5. Enable long mode ใน EFER MSR (bit 8 = LME)
6. Enable paging (CR0.PG)
7. Far jump ไปยัง 64-bit code segment
8. CPU enters 64-bit mode

### 12.2 Complete Bootloader: Protected Mode → Long Mode พร้อม Paging

```nasm
; boot32_to_64.asm
; Bootloader ที่ switch จาก 32-bit protected mode ไปยัง 64-bit long mode
; พร้อม paging

bits 32

; MSR addresses
EFER_MSR        equ 0xC0000080  ; Extended Feature Enable Register
EFER_LME        equ (1 << 8)    ; Long Mode Enable
EFER_NXE        equ (1 << 11)   ; No-Execute Enable

section .bss
align 4096

pml4:   resb 4096
pdpt:   resb 4096
pd:     resb 4096

section .text
global start_long_mode
extern long_mode_entry          ; 64-bit entry point

; ─────────────────────────────────────────────
; start_long_mode - เข้า 64-bit mode
; ─────────────────────────────────────────────
start_long_mode:
    ; ─── 1. Check CPUID support for long mode ───
    mov  eax, 0x80000001
    cpuid
    test edx, (1 << 29)     ; LM bit
    jz   .no_long_mode

    ; ─── 2. Setup page tables ───
    call setup_page_tables_64

    ; ─── 3. Enable PAE ───
    mov  eax, cr4
    or   eax, 0x20          ; CR4.PAE = bit 5
    mov  cr4, eax

    ; ─── 4. Load PML4 into CR3 ───
    mov  eax, pml4
    mov  cr3, eax

    ; ─── 5. Enable Long Mode in EFER ───
    mov  ecx, EFER_MSR
    rdmsr                   ; อ่าน EFER (result ใน EDX:EAX)
    or   eax, EFER_LME      ; Set LME bit
    or   eax, EFER_NXE      ; Set NXE bit (optional แต่ดี)
    wrmsr                   ; เขียนกลับ

    ; ─── 6. Enable Paging (ซึ่งจะ activate long mode) ───
    mov  eax, cr0
    or   eax, 0x80000000    ; CR0.PG
    mov  cr0, eax
    ; ตอนนี้อยู่ใน "compatibility mode" (32-bit code ใน long mode)

    ; ─── 7. Far Jump ไปยัง 64-bit code ───
    ; ต้องใช้ far jump เพื่อ load 64-bit CS
    jmp  0x08:long_mode_entry   ; 0x08 = 64-bit code segment ใน GDT

.no_long_mode:
    ; Error: CPU ไม่รองรับ long mode
    mov  dword [0xB8000], 0x4F524F45   ; 'ER' ใน VGA
    mov  dword [0xB8004], 0x4F524F4F   ; 'OO' ใน VGA
    cli
    hlt

; ─────────────────────────────────────────────
; setup_page_tables_64 - ตั้งค่า PML4/PDPT/PD
; Identity map first 1GB ด้วย 2MB pages
; ─────────────────────────────────────────────
setup_page_tables_64:
    ; Clear tables
    mov  edi, pml4
    xor  eax, eax
    mov  ecx, 3 * 1024      ; 3 tables × 1024 DWORDs
    rep  stosd

    ; PML4[0] → PDPT
    mov  eax, pdpt
    or   eax, 0x03              ; Present + Writable
    mov  [pml4], eax
    mov  dword [pml4 + 4], 0

    ; PDPT[0] → PD
    mov  eax, pd
    or   eax, 0x03
    mov  [pdpt], eax
    mov  dword [pdpt + 4], 0

    ; PD: 512 entries, แต่ละ entry = 2MB huge page
    ; Map 512 × 2MB = 1GB ตั้งแต่ physical 0
    mov  edi, pd
    mov  eax, 0x000083         ; PA=0, Present, Writable, PS=1 (2MB)
    mov  edx, 0                ; Upper 32 bits
    mov  ecx, 512
.fill_pd:
    mov  [edi], eax
    mov  [edi + 4], edx
    add  eax, 0x200000          ; +2MB
    adc  edx, 0                 ; carry
    add  edi, 8                 ; next 8-byte entry
    loop .fill_pd

    ret
```

### 12.3 64-bit Entry Code

```nasm
; entry64.asm - 64-bit entry point หลัง long mode switch
bits 64

extern kernel_main_64

section .text
global long_mode_entry

long_mode_entry:
    ; อยู่ใน 64-bit mode แล้ว!
    ; ตั้งค่า segment registers (long mode ใช้ flat model)
    mov  ax, 0x10           ; data segment
    mov  ds, ax
    mov  es, ax
    mov  fs, ax
    mov  gs, ax
    mov  ss, ax

    ; ตั้งค่า stack
    mov  rsp, 0x90000       ; stack ที่ high address

    ; เรียก 64-bit kernel
    call kernel_main_64

.halt:
    cli
    hlt
    jmp  .halt
```

---

## 13. GDT สำหรับ Long Mode

Long mode ต้องการ GDT ที่มี 64-bit code segment:

```nasm
; gdt64.asm - GDT สำหรับ Long Mode
bits 32

; GDT Entry structure:
; Bits 63-56: base[31:24]
; Bit  55:    G (granularity)
; Bit  54:    D/B (32-bit = 1, 64-bit code = 0)
; Bit  53:    L (64-bit code segment = 1)
; Bit  52:    AVL
; Bits 51-48: limit[19:16]
; Bits 47-40: access byte
; Bits 39-32: base[23:16]
; Bits 31-16: base[15:0]
; Bits 15-0:  limit[15:0]

section .data
align 8

gdt64:
.null:
    ; Null descriptor
    dq 0

.code:
    ; 64-bit code segment: base=0, limit=0 (ignored), L=1, P=1, DPL=0, Type=A
    ; 0x00AF9A000000FFFF
    dw 0xFFFF               ; limit[15:0]
    dw 0x0000               ; base[15:0]
    db 0x00                 ; base[23:16]
    db 0x9A                 ; access: P=1, DPL=0, S=1, Type=A (code, execute/read)
    db 0xAF                 ; flags: G=1, D=0, L=1, AVL=0, limit[19:16]=0xF
    db 0x00                 ; base[31:24]

.data:
    ; 64-bit data segment
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0x92                 ; access: P=1, DPL=0, S=1, Type=2 (data, read/write)
    db 0xCF                 ; flags: G=1, D=1, L=0, AVL=0
    db 0x00

gdt64_end:

gdt64_pointer:
    dw (gdt64_end - gdt64 - 1)     ; limit
    dd gdt64                        ; base address

section .text
global load_gdt64

load_gdt64:
    lgdt [gdt64_pointer]
    ret
```

---

## 14. Complete Example: Kernel ที่มี Paging เต็มรูปแบบ

### 14.1 Project Structure

```
kernel_paging/
├── boot/
│   ├── boot.asm          ; Multiboot entry, setup paging
│   └── gdt.asm           ; GDT setup
├── kernel/
│   ├── kernel.asm        ; Kernel entry point
│   └── page_fault.asm    ; Page fault handler
├── include/
│   └── paging.inc        ; Page table constants
├── kernel.ld             ; Linker script
├── Makefile
└── grub.cfg
```

### 14.2 paging.inc

```nasm
; include/paging.inc - Constants สำหรับ paging

; ─── Page Flags ───
%define PF_PRESENT      (1 << 0)    ; Page present
%define PF_WRITABLE     (1 << 1)    ; Writable
%define PF_USER         (1 << 2)    ; User accessible
%define PF_PWT          (1 << 3)    ; Write-through
%define PF_PCD          (1 << 4)    ; Cache disable
%define PF_ACCESSED     (1 << 5)    ; Accessed
%define PF_DIRTY        (1 << 6)    ; Dirty
%define PF_LARGE        (1 << 7)    ; 4MB page (PDE) / 2MB page (PDE64)
%define PF_GLOBAL       (1 << 8)    ; Global page

; Common combinations
%define PF_KERNEL       (PF_PRESENT | PF_WRITABLE)
%define PF_KERNEL_LARGE (PF_PRESENT | PF_WRITABLE | PF_LARGE)
%define PF_USER_RW      (PF_PRESENT | PF_WRITABLE | PF_USER)

; ─── Address Constants ───
%define KERNEL_VIRT_BASE    0xC0000000
%define KERNEL_PHYS_BASE    0x00100000
%define PAGE_SIZE           4096
%define PAGE_SIZE_LARGE     (4 * 1024 * 1024)   ; 4MB

; ─── CR bit masks ───
%define CR0_PE      (1 << 0)        ; Protected Mode Enable
%define CR0_PG      (1 << 31)       ; Paging
%define CR4_PSE     (1 << 4)        ; Page Size Extension (4MB)
%define CR4_PAE     (1 << 5)        ; Physical Address Extension
%define CR4_PGE     (1 << 7)        ; Page Global Enable

; ─── MSR ───
%define EFER_MSR    0xC0000080
%define EFER_LME    (1 << 8)        ; Long Mode Enable
%define EFER_LMA    (1 << 10)       ; Long Mode Active (read-only)
%define EFER_NXE    (1 << 11)       ; No-Execute Enable
```

### 14.3 boot.asm (Full Version)

```nasm
; boot/boot.asm
bits 32

%include "include/paging.inc"

; Multiboot
MULTIBOOT_MAGIC     equ 0x1BADB002
MULTIBOOT_FLAGS     equ 0x00000003
MULTIBOOT_CHECKSUM  equ -(MULTIBOOT_MAGIC + MULTIBOOT_FLAGS)

section .multiboot
align 4
    dd MULTIBOOT_MAGIC
    dd MULTIBOOT_FLAGS
    dd MULTIBOOT_CHECKSUM

; ─── Static Page Tables (Physical) ───
section .data
align 4096

global boot_page_directory
boot_page_directory:
    ; [0]: Identity map (VA 0x00000000-0x003FFFFF = PA same)
    ;      ต้องการตอน CPU ยังอยู่ที่ low address หลัง enable paging
    dd PF_KERNEL_LARGE
    
    times 767 dd 0      ; [1..767] not present
    
    ; [768]: Kernel high half (VA 0xC0000000-0xC03FFFFF → PA 0x00000000-0x003FFFFF)
    dd PF_KERNEL_LARGE
    
    ; [769]: Second 4MB (VA 0xC0400000-0xC07FFFFF → PA 0x00400000-0x007FFFFF)
    dd (0x00400000 | PF_KERNEL_LARGE)
    
    times 254 dd 0      ; [770..1023] not present

; ─── Stack ───
section .bss
align 16

resb 16384              ; 16KB initial stack
stack_top:

section .text
global _start

_start:
    ; Disable interrupts
    cli
    
    ; บันทึก multiboot magic และ info
    mov esi, ebx        ; multiboot info
    mov edi, eax        ; multiboot magic

    ; ─── Enable PSE (4MB large pages) ───
    mov eax, cr4
    or  eax, CR4_PSE
    mov cr4, eax

    ; ─── Load Page Directory ───
    ; boot_page_directory เป็น virtual address จาก linker
    ; แต่ตอนนี้ paging ยังปิด ดังนั้น virtual = physical (ไม่มีการแปล)
    ; KERNEL_VIRT_BASE = 0xC0000000, KERNEL_PHYS_BASE = 0x00100000
    ; แต่ NASM/LD จะ resolve boot_page_directory เป็น virtual address
    ; ถ้า link ที่ 0xC0100000, ต้องลบ offset
    mov eax, (boot_page_directory - KERNEL_VIRT_BASE)
    mov cr3, eax

    ; ─── Enable Paging ───
    mov eax, cr0
    or  eax, CR0_PG
    mov cr0, eax

    ; Jump ไปยัง high virtual address
    lea eax, [.higher_half]
    jmp eax

.higher_half:
    ; ตั้งค่า stack ที่ high address
    mov esp, stack_top

    ; ลบ identity map (PDE[0])
    mov dword [boot_page_directory], 0

    ; Flush TLB
    mov eax, cr3
    mov cr3, eax

    ; ส่ง multiboot info ไปยัง kernel
    push esi            ; multiboot_info*
    push edi            ; multiboot_magic

    extern kernel_main
    call kernel_main

    ; Should never reach here
    cli
.halt:
    hlt
    jmp .halt
```

### 14.4 kernel.asm

```nasm
; kernel/kernel.asm
bits 32

%include "include/paging.inc"

VGA_BUFFER      equ 0xB8000
VGA_WHITE       equ 0x0F

section .text
global kernel_main

; ─────────────────────────────────────────────
; kernel_main - Entry point หลัง paging setup
; Arguments (cdecl):
;   [esp+4]  = multiboot_magic
;   [esp+8]  = multiboot_info*
; ─────────────────────────────────────────────
kernel_main:
    push ebp
    mov  ebp, esp

    ; ── ทดสอบว่า paging ทำงาน ──
    ; ถ้า paging ถูกต้อง เราจะอยู่ที่ virtual address 0xC0xxxxxx
    
    ; พิมพ์ "PAGING OK" ใน VGA
    mov  edi, VGA_BUFFER
    lea  esi, [msg_paging_ok]
    call print_string

    ; ── ทดสอบ identity map ถูกลบออกแล้ว ──
    ; Virtual address 0x00001000 ควร fault แล้ว
    ; (แสดงว่า kernel ทำงานใน isolated address space)

    ; ── Loop ──
.done:
    cli
    hlt
    jmp .done

; ─────────────────────────────────────────────
; print_string - พิมพ์ null-terminated string ที่ VGA
; Input: EDI = VGA buffer position, ESI = string
; ─────────────────────────────────────────────
print_string:
    push eax
.loop:
    lodsb               ; AL = *ESI++
    test al, al
    jz   .done
    mov  ah, VGA_WHITE
    stosw               ; *EDI++ = AX
    jmp  .loop
.done:
    pop  eax
    ret

section .rodata
msg_paging_ok:  db "Paging OK! Kernel at 0xC0000000", 0
```

---

## 15. Makefile

```makefile
# Makefile สำหรับ kernel พร้อม paging

AS      := nasm
LD      := ld
ASFLAGS := -f elf32 -I./
LDFLAGS := -m elf_i386 -T kernel.ld

# Source files
BOOT_ASM    := boot/boot.asm
KERNEL_ASM  := kernel/kernel.asm
PF_ASM      := kernel/page_fault.asm

# Object files
OBJS := boot/boot.o kernel/kernel.o kernel/page_fault.o

# Output
KERNEL := kernel.bin
ISO    := kernel.iso

.PHONY: all clean iso run debug

all: $(KERNEL)

# Assemble
boot/boot.o: boot/boot.asm include/paging.inc
	$(AS) $(ASFLAGS) -o $@ $<

kernel/kernel.o: kernel/kernel.asm include/paging.inc
	$(AS) $(ASFLAGS) -o $@ $<

kernel/page_fault.o: kernel/page_fault.asm
	$(AS) $(ASFLAGS) -o $@ $<

# Link
$(KERNEL): $(OBJS) kernel.ld
	$(LD) $(LDFLAGS) -o $@ $(OBJS)

# Create ISO
iso: $(KERNEL)
	mkdir -p iso/boot/grub
	cp $(KERNEL) iso/boot/
	cp grub.cfg  iso/boot/grub/
	grub-mkrescue -o $(ISO) iso/

# Run in QEMU
run: $(KERNEL)
	qemu-system-i386 \
		-kernel $(KERNEL) \
		-m 128M \
		-serial stdio \
		-no-reboot

# Run ISO
run-iso: iso
	qemu-system-i386 \
		-cdrom $(ISO) \
		-m 128M \
		-serial stdio \
		-no-reboot

# Debug
debug: $(KERNEL)
	qemu-system-i386 \
		-kernel $(KERNEL) \
		-m 128M \
		-serial stdio \
		-no-reboot \
		-s -S &
	gdb -ex "target remote :1234" \
	    -ex "symbol-file $(KERNEL)"

clean:
	rm -f $(OBJS) $(KERNEL) $(ISO)
	rm -rf iso/
```

### 15.1 Linker Script (kernel.ld)

```ld
/* kernel.ld - Linker script สำหรับ higher-half kernel */

ENTRY(_start)

KERNEL_VIRT = 0xC0000000;
KERNEL_PHYS = 0x00100000;

SECTIONS {
    /* Virtual address เริ่มที่ 0xC0100000 */
    /* แต่ loaded ที่ physical 0x00100000 */
    
    . = KERNEL_VIRT + KERNEL_PHYS;
    
    _kernel_start = .;
    
    /* Multiboot header ต้องอยู่ใน 8KB แรก */
    .multiboot ALIGN(4) : AT(ADDR(.multiboot) - KERNEL_VIRT) {
        *(.multiboot)
    }
    
    .text ALIGN(4096) : AT(ADDR(.text) - KERNEL_VIRT) {
        *(.text*)
    }
    
    .rodata ALIGN(4096) : AT(ADDR(.rodata) - KERNEL_VIRT) {
        *(.rodata*)
    }
    
    .data ALIGN(4096) : AT(ADDR(.data) - KERNEL_VIRT) {
        *(.data*)
    }
    
    .bss ALIGN(4096) : AT(ADDR(.bss) - KERNEL_VIRT) {
        *(COMMON)
        *(.bss*)
    }
    
    _kernel_end = .;
    
    /DISCARD/ : {
        *(.comment)
        *(.note.*)
    }
}
```

---

## 16. QEMU Test Commands

### 16.1 Basic Testing

```bash
# Compile และ run
nasm -f elf32 boot/boot.asm -o boot/boot.o -I./
ld -m elf_i386 -T kernel.ld -o kernel.bin boot/boot.o

# Run ด้วย QEMU
qemu-system-i386 -kernel kernel.bin -m 128M -serial stdio -no-reboot

# Run พร้อม display VGA
qemu-system-i386 \
    -kernel kernel.bin \
    -m 128M \
    -display sdl \
    -serial stdio \
    -no-reboot

# Run พร้อม monitor (QEMU interactive console)
qemu-system-i386 \
    -kernel kernel.bin \
    -m 128M \
    -monitor stdio \
    -no-reboot
```

### 16.2 Debug Commands

```bash
# เริ่ม QEMU รอ debugger
qemu-system-i386 \
    -kernel kernel.bin \
    -m 128M \
    -serial stdio \
    -no-reboot \
    -s -S          # -s = GDB server port 1234, -S = wait before start

# ใน terminal อื่น เริ่ม GDB
gdb

# ใน GDB:
(gdb) target remote :1234
(gdb) symbol-file kernel.bin
(gdb) break kernel_main
(gdb) continue
(gdb) info registers                    # ดู CR0, CR3, ฯลฯ
(gdb) x/10i $eip                       # disassemble รอบ EIP
(gdb) x/4xw 0xC0000000                 # อ่าน memory ที่ virtual address
```

### 16.3 QEMU Monitor Commands (ตรวจสอบ Paging)

```
# เปิด QEMU monitor ด้วย Ctrl+Alt+2
# หรือใช้ -monitor stdio

# ดู registers รวมถึง CR0, CR3
(qemu) info registers

# ดู page tables ที่ QEMU ตีความ
(qemu) info tlb

# ดู physical memory ที่ virtual address
(qemu) xp /4xw 0xC0000000    # x = hex, p = physical

# ดู virtual memory
(qemu) x /4xw 0xC0000000     # virtual

# ตรวจสอบ page table tree
(qemu) info mem

# Dump page tables อย่างละเอียด
(qemu) x /1024xw [cr3_value]   # Page Directory entries
```

### 16.4 ทดสอบ Page Fault

```bash
# เพิ่มใน kernel เพื่อทดสอบ page fault handler:
# mov dword [0x00001000], 0   ; Access unmapped address
# ถ้า handler ทำงาน จะเห็น "Page Fault!" ใน output

# Run พร้อม log
qemu-system-i386 \
    -kernel kernel.bin \
    -m 128M \
    -d int \               # log interrupts
    -D /tmp/qemu.log \
    -no-reboot

# ดู log
cat /tmp/qemu.log | grep "page fault"
```

---

## 17. Debugging Page Table Issues

### 17.1 ปัญหาที่พบบ่อย

**1. Triple Fault หลัง Enable Paging**
- สาเหตุ: ไม่ได้ทำ identity map สำหรับ code ที่กำลัง execute อยู่
- แก้ไข: ตรวจสอบว่า PDE[0] หรือ PTE สำหรับ physical address ปัจจุบัน = present

**2. Page Fault ที่ address แปลกๆ**
- สาเหตุ: Stack อยู่ใน unmapped region
- แก้ไข: ตรวจสอบว่า stack address ถูก map ไว้

**3. Kernel Code ไม่ Run หลัง Jump to High Half**
- สาเหตุ: PDE[768] (0xC0000000) ไม่ได้ map ถูกต้อง
- แก้ไข: ตรวจสอบ physical address ใน PDE[768]

**4. Wrong CR3 Value**
- สาเหตุ: ใส่ virtual address แทน physical address ใน CR3
- แก้ไข: ใช้ `(boot_page_directory - KERNEL_VIRT_BASE)` ถ้า link ที่ high half

### 17.2 Checklist

```nasm
; ─── Debug: พิมพ์ CR3 value ───
debug_print_cr3:
    mov eax, cr3
    ; พิมพ์ eax เป็น hex...
    ret

; ─── Debug: ตรวจสอบ PDE ───
; eax = virtual address ที่ต้องการตรวจสอบ
check_pde:
    push ebx
    mov  ebx, cr3               ; Page Directory base (physical)
    shr  eax, 22                ; PD index = VA[31:22]
    and  eax, 0x3FF             ; mask 10 bits
    shl  eax, 2                 ; × 4 bytes per entry
    add  eax, ebx               ; physical address ของ PDE
    mov  eax, [eax]             ; อ่าน PDE
    ; ตรวจสอบ bit 0 (Present)
    test eax, 1
    jz   .not_present
    ; PDE present, ดู bit 7 (PS) สำหรับ large page
    test eax, 0x80
    jnz  .large_page
    ; 4KB paging: ดู PT
    ; ...
.not_present:
    ; แสดง error
.large_page:
    ; Large page: physical = PDE[31:22] << 22 | VA[21:0]
    pop  ebx
    ret
```

---

## 18. Physical Memory Manager Integration

เมื่อ implement paging ต้องมี physical memory allocator ด้วย:

```nasm
; pmm.asm - Simple Physical Memory Manager (Bitmap-based)
bits 32

PAGE_SIZE       equ 4096

section .bss
; Bitmap: 1 bit per 4KB page
; สำหรับ 128MB RAM = 32768 pages = 4096 bytes bitmap
pmm_bitmap:
    resb 4096

section .data
pmm_total_pages:    dd 0
pmm_free_pages:     dd 0

section .text

; ─────────────────────────────────────────────
; pmm_init - Initialize physical memory manager
; Input: EAX = total RAM in bytes
; ─────────────────────────────────────────────
global pmm_init
pmm_init:
    push edi
    
    ; คำนวณจำนวน pages
    shr  eax, 12                    ; / 4096
    mov  [pmm_total_pages], eax
    mov  [pmm_free_pages], eax
    
    ; ล้าง bitmap (0 = free, 1 = used)
    mov  edi, pmm_bitmap
    xor  eax, eax
    mov  ecx, 1024                  ; 4096/4 DWORDs
    rep  stosd
    
    ; Mark pages 0-1MB ว่าใช้งาน (kernel, VGA, etc.)
    ; เขียน 1 ใน bitmap สำหรับ 256 pages แรก (0-1MB)
    mov  edi, pmm_bitmap
    mov  ecx, 8                     ; 256 pages / 32 bits per byte
    mov  eax, 0xFFFFFFFF
    rep  stosd
    
    sub  dword [pmm_free_pages], 256
    
    pop  edi
    ret

; ─────────────────────────────────────────────
; pmm_alloc_page - Allocate one physical page
; Output: EAX = physical address (or 0 if OOM)
; ─────────────────────────────────────────────
global pmm_alloc_page
pmm_alloc_page:
    push ebx
    push ecx
    push edx
    
    cmp  dword [pmm_free_pages], 0
    jz   .oom
    
    ; หา free bit ใน bitmap (bsf = bit scan forward)
    mov  edi, pmm_bitmap
    mov  ecx, [pmm_total_pages]
    shr  ecx, 5                     ; / 32 (words)
.search:
    mov  eax, [edi]
    cmp  eax, 0xFFFFFFFF
    jne  .found_word
    add  edi, 4
    loop .search
    jmp  .oom

.found_word:
    not  eax                        ; invert: หา 0-bit
    bsf  ebx, eax                   ; ebx = bit index (0-31)
    
    ; Set bit ใน bitmap
    bts  [edi], ebx                 ; atomic bit set
    
    ; คำนวณ physical address
    sub  edi, pmm_bitmap
    shl  edi, 3                     ; edi = byte_offset * 8 = bit index ของ word
    add  edi, ebx                   ; + bit ใน word = page number
    shl  edi, 12                    ; * 4096 = physical address
    mov  eax, edi
    
    dec  dword [pmm_free_pages]
    
    pop  edx
    pop  ecx
    pop  ebx
    ret

.oom:
    xor  eax, eax                   ; return 0
    pop  edx
    pop  ecx
    pop  ebx
    ret

; ─────────────────────────────────────────────
; pmm_free_page - Free a physical page
; Input: EAX = physical address
; ─────────────────────────────────────────────
global pmm_free_page
pmm_free_page:
    push ebx
    
    shr  eax, 12                    ; page number
    mov  ebx, eax
    shr  ebx, 5                     ; word index
    and  eax, 31                    ; bit index
    
    ; Clear bit ใน bitmap
    btr  [pmm_bitmap + ebx * 4], eax
    
    inc  dword [pmm_free_pages]
    
    pop  ebx
    ret
```

---

## 19. Virtual Memory Manager: Map Page

```nasm
; vmm.asm - Virtual Memory Manager
bits 32

%include "include/paging.inc"

section .text

extern pmm_alloc_page

; ─────────────────────────────────────────────
; vmm_map_page - Map virtual page to physical page
; Input:
;   EAX = virtual address
;   EBX = physical address
;   ECX = flags (PF_PRESENT | PF_WRITABLE etc.)
; Output: EAX = 0 success, -1 error
; ─────────────────────────────────────────────
global vmm_map_page
vmm_map_page:
    push esi
    push edi
    push edx
    push ebp
    mov  ebp, esp
    
    ; ─── หา PDE ───
    mov  esi, eax                   ; save VA
    shr  eax, 22                    ; PD index
    mov  edi, cr3                   ; Page Directory physical address
    lea  edi, [edi + eax * 4]       ; &PDE[index]
    
    ; ─── ถ้า PDE ไม่ present ต้อง allocate Page Table ───
    mov  edx, [edi]
    test edx, PF_PRESENT
    jnz  .pt_exists
    
    ; Allocate new page table
    call pmm_alloc_page
    test eax, eax
    jz   .error
    
    push ecx
    
    ; Clear new page table
    push eax
    mov  edi, eax
    xor  eax, eax
    mov  ecx, 1024
    rep  stosd
    pop  eax
    
    pop  ecx
    
    ; Install PDE
    or   eax, (PF_PRESENT | PF_WRITABLE | PF_USER)
    mov  [edi], eax
    mov  edx, eax

.pt_exists:
    ; EDX = PDE value, address of PT = EDX & ~0xFFF
    and  edx, ~0xFFF                ; extract PT physical address
    
    ; ─── หา PTE ───
    mov  eax, esi                   ; VA
    shr  eax, 12
    and  eax, 0x3FF                 ; PT index
    lea  edi, [edx + eax * 4]       ; &PTE[index]
    
    ; ─── Install PTE ───
    mov  eax, ebx                   ; physical address
    and  eax, ~0xFFF                ; align
    or   eax, ecx                   ; flags
    mov  [edi], eax                 ; write PTE
    
    ; ─── Invalidate TLB ───
    invlpg [esi]
    
    xor  eax, eax                   ; success
    jmp  .done

.error:
    mov  eax, -1

.done:
    pop  ebp
    pop  edx
    pop  edi
    pop  esi
    ret

; ─────────────────────────────────────────────
; vmm_unmap_page - Unmap a virtual page
; Input: EAX = virtual address
; ─────────────────────────────────────────────
global vmm_unmap_page
vmm_unmap_page:
    push edi
    push edx
    
    mov  edx, eax                   ; save VA
    
    ; หา PDE
    shr  eax, 22
    mov  edi, cr3
    mov  edi, [edi + eax * 4]       ; PDE
    test edi, PF_PRESENT
    jz   .done                      ; ถ้า PDE ไม่ present, ไม่ต้องทำ
    
    ; หา PTE
    and  edi, ~0xFFF                ; PT physical address
    mov  eax, edx
    shr  eax, 12
    and  eax, 0x3FF                 ; PT index
    lea  edi, [edi + eax * 4]       ; &PTE
    
    ; Clear PTE
    mov  dword [edi], 0
    
    ; Invalidate TLB
    invlpg [edx]

.done:
    pop  edx
    pop  edi
    ret
```

---

## 20. สรุปและตัวอย่าง Complete Flow

### 20.1 Flow การ Boot พร้อม Paging

```
BIOS/Bootloader Load Kernel
         │
         ▼
_start (32-bit, protected mode)
         │
         ├─ Enable CR4.PSE
         │
         ├─ Load CR3 (physical address ของ page directory)
         │
         ├─ Enable CR0.PG
         │
         ├─ Far jump ไปยัง high-half virtual address
         │
         ▼
higher_half (virtual 0xC0xxxxxx)
         │
         ├─ ลบ identity map (PDE[0] = 0)
         │
         ├─ Flush TLB
         │
         ├─ Setup IDT (page fault handler)
         │
         ├─ Initialize PMM
         │
         └─ Call kernel_main()
```

### 20.2 Flow การ Map Page ใหม่

```
kernel ต้องการ allocate virtual memory
         │
         ▼
vmm_map_page(virt_addr, phys_addr, flags)
         │
         ├─ หา PD index = virt_addr >> 22
         │
         ├─ ตรวจ PDE[index]
         │   ├─ Not present: allocate new Page Table (pmm_alloc_page)
         │   └─ Present: ใช้ PT ที่มีอยู่
         │
         ├─ หา PT index = (virt_addr >> 12) & 0x3FF
         │
         ├─ เขียน PTE = phys_addr | flags
         │
         └─ INVLPG [virt_addr] (flush TLB entry)
```

### 20.3 Quick Reference: สิ่งที่ต้องจำ

| เรื่อง | รายละเอียด |
|--------|-----------|
| CR3 | Physical address ของ Page Directory |
| CR0.PG | Bit 31: เปิด paging |
| CR4.PSE | Bit 4: เปิด 4MB pages |
| CR4.PAE | Bit 5: เปิด Physical Address Extension (สำหรับ PAE/64-bit) |
| PDE PS=1 | Large 4MB page (ต้องมี CR4.PSE=1) |
| CR2 | Virtual address ที่เกิด page fault |
| INVLPG [addr] | Invalidate TLB entry สำหรับ addr เดียว |
| MOV CR3, x | Flush ทุก TLB entries (ยกเว้น global) |
| PML4 | 64-bit first level (CR3 → PML4 → PDPT → PD → PT) |

---

## แบบฝึกหัด (Exercises)

1. **Identity Mapping**: เขียน NASM code ที่ทำ identity map ขนาด 16MB (4 PDE entries × 4MB) ด้วย large pages

2. **Dynamic Page Mapping**: เขียนฟังก์ชัน `map_page(virt, phys, flags)` ที่ allocate page table ใหม่เมื่อจำเป็น

3. **Page Fault Handler**: เขียน page fault handler ที่แสดง CR2, error code และ EIP เมื่อเกิด fault

4. **High-Half Kernel**: ปรับ linker script และ boot code เพื่อให้ kernel อยู่ที่ 0xC0000000 และทดสอบด้วย QEMU

5. **64-bit Paging**: เขียน setup code สำหรับ PML4/PDPT/PD/PT เพื่อ identity map 2GB แรกด้วย 2MB pages และทดสอบใน QEMU

6. **TLB Test**: เขียน code ที่แก้ PTE ของ page หนึ่ง แล้วทดสอบว่าต้อง INVLPG ก่อนที่ CPU จะเห็น mapping ใหม่

---

## อ้างอิง (References)

- Intel Manual Vol. 3A: System Programming Guide, Chapter 4 (Paging)
- AMD64 Architecture Programmer's Manual Vol. 2: System Programming
- OSDev Wiki: Paging — https://wiki.osdev.org/Paging
- OSDev Wiki: Higher Half Kernel — https://wiki.osdev.org/Higher_Half_Kernel
- OSDev Wiki: Setting Up Long Mode — https://wiki.osdev.org/Setting_Up_Long_Mode

---

*Part 085 จบ — ถัดไป Part 086: Memory Allocation Algorithms*
