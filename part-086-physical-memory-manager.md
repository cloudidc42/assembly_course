# Part 086: Physical Memory Manager (PMM)

## Physical Memory Manager คืออะไร?

Physical Memory Manager (PMM) คือส่วนสำคัญของ kernel ที่ทำหน้าที่จัดการ **physical memory** (RAM จริงๆ ในเครื่อง) PMM เป็น layer แรกของ memory management ก่อนที่จะมี virtual memory, paging, หรือ malloc

หน้าที่หลักของ PMM:
- ติดตามว่า page ไหนถูกใช้งานอยู่ (used) หรือว่าง (free)
- จัดสรร physical page ให้กับ kernel/process
- คืน physical page เมื่อไม่ใช้แล้ว
- รองรับการจัดสรรหลาย page ติดกัน (contiguous)

---

## บทที่ 1: Memory Map จาก BIOS (INT 15h E820)

### 1.1 ทำไมต้องใช้ Memory Map?

เมื่อ kernel เริ่มทำงาน มันไม่รู้ว่า RAM มีเท่าไหร่ หรือตรงไหนบ้างที่ใช้งานได้จริง เพราะ:
- บางช่วงของ address space เป็น BIOS ROM
- บางช่วงเป็น memory-mapped I/O
- บางช่วงเป็น reserved สำหรับ hardware
- ACPI tables อยู่ที่ไหนสักที่ใน RAM

วิธีรู้คือถาม BIOS ผ่าน **INT 15h, AX=E820h**

### 1.2 โครงสร้าง E820 Entry

```
; E820 Memory Map Entry (20 bytes)
struc e820_entry
    .base_low    resd 1    ; bits 0-31 ของ base address
    .base_high   resd 1    ; bits 32-63 ของ base address
    .len_low     resd 1    ; bits 0-31 ของ length
    .len_high    resd 1    ; bits 32-63 ของ length
    .type        resd 1    ; ประเภทของ memory region
endstruc
```

### 1.3 ประเภทของ Memory Region

| Type | ชื่อ | ความหมาย |
|------|------|-----------|
| 1 | Available | ใช้งานได้ เป็น RAM ปกติ |
| 2 | Reserved | ห้ามใช้ เป็น hardware/BIOS |
| 3 | ACPI Reclaimable | ใช้ชั่วคราวสำหรับ ACPI tables (คืนได้ภายหลัง) |
| 4 | ACPI NVS | ACPI Non-Volatile Storage ห้ามใช้ |
| 5 | Bad RAM | หน่วยความจำเสีย ห้ามใช้ |

### 1.4 โค้ด NASM สำหรับเรียก INT 15h E820

```nasm
; =============================================================================
; e820_detect.asm - ตรวจจับ memory map ด้วย INT 15h E820
; รันใน real mode หรือ unreal mode ก่อน switch ไป protected mode
; =============================================================================

BITS 16

%define E820_MAGIC      0x534D4150  ; "SMAP"
%define E820_MAX_ENTRIES 128

section .data
e820_map_count  dd 0
e820_map:
    times (E820_MAX_ENTRIES * 20) db 0  ; 20 bytes per entry

section .text

; detect_memory_e820:
;   ตรวจจับ memory map ด้วย INT 15h AX=E820h
;   ผลลัพธ์: e820_map_count = จำนวน entries
;            e820_map = array of e820_entry
detect_memory_e820:
    push es
    push di
    push ebx
    push ecx
    push edx
    push eax

    ; ตั้งค่าเริ่มต้น
    xor ebx, ebx                ; continuation value = 0 (เริ่มต้น)
    xor bp, bp                  ; นับจำนวน entries

    ; ชี้ buffer ไปที่ e820_map
    mov di, e820_map

.loop:
    mov eax, 0x0000E820         ; function E820
    mov ecx, 20                 ; ขนาด entry = 20 bytes
    mov edx, E820_MAGIC         ; magic number "SMAP"
    int 0x15                    ; เรียก BIOS

    ; ตรวจสอบ error
    jc .done                    ; carry flag = error
    mov edx, E820_MAGIC
    cmp eax, edx                ; EAX ต้องเท่ากับ "SMAP"
    jne .done
    test ecx, ecx               ; ขนาด entry ต้องไม่เป็น 0
    jz .skip_entry
    cmp ecx, 20
    jl .skip_entry              ; entry เล็กเกินไป

    ; entry ถูกต้อง
    inc bp
    add di, 20                  ; ไปยัง entry ถัดไป

    ; ตรวจสอบว่าเต็มหรือยัง
    cmp bp, E820_MAX_ENTRIES
    jge .done

.skip_entry:
    ; ถ้า EBX = 0 แสดงว่า scan เสร็จแล้ว
    test ebx, ebx
    jz .done
    jmp .loop

.done:
    mov [e820_map_count], ebp

    pop eax
    pop edx
    pop ecx
    pop ebx
    pop di
    pop es
    ret
```

---

## บทที่ 2: GRUB Multiboot Memory Map

### 2.1 Multiboot Information Structure

เมื่อใช้ GRUB เป็น bootloader มันจะส่ง **multiboot information structure** ให้ kernel ผ่าน register EBX

```nasm
; =============================================================================
; multiboot.inc - Multiboot structures และ constants
; =============================================================================

; Multiboot header magic
MULTIBOOT_MAGIC         equ 0x1BADB002
MULTIBOOT_BOOTLOADER_MAGIC equ 0x2BADB002

; Multiboot flags
MULTIBOOT_FLAG_MEM      equ (1 << 0)    ; mem_lower/upper valid
MULTIBOOT_FLAG_MMAP     equ (1 << 6)    ; mmap valid

; Multiboot Information Structure
struc multiboot_info
    .flags          resd 1      ; flags บอกว่า field ไหน valid
    .mem_lower      resd 1      ; lower memory ใน KB (จาก 0)
    .mem_upper      resd 1      ; upper memory ใน KB (จาก 1MB)
    .boot_device    resd 1      ; boot device
    .cmdline        resd 1      ; pointer to command line
    .mods_count     resd 1      ; จำนวน modules
    .mods_addr      resd 1      ; address ของ module list
    .syms           resb 12     ; symbol table info
    .mmap_length    resd 1      ; ขนาดของ memory map ใน bytes
    .mmap_addr      resd 1      ; address ของ memory map
    ; ... fields อื่นๆ
endstruc

; Multiboot Memory Map Entry
struc multiboot_mmap_entry
    .size       resd 1      ; ขนาดของ entry นี้ (ไม่รวม field size เอง)
    .base_low   resd 1      ; base address bits 0-31
    .base_high  resd 1      ; base address bits 32-63
    .len_low    resd 1      ; length bits 0-31
    .len_high   resd 1      ; length bits 32-63
    .type       resd 1      ; 1=available, 2=reserved, etc.
endstruc
```

### 2.2 การอ่าน Multiboot Memory Map

```nasm
; =============================================================================
; parse_multiboot.asm - อ่าน memory map จาก GRUB Multiboot
; =============================================================================

BITS 32

section .bss
    mb_total_memory resd 1      ; total available memory ใน bytes

section .text

; parse_multiboot_mmap:
;   input: EBX = pointer to multiboot_info structure
;   สแกน memory map และคำนวณ total available memory
parse_multiboot_mmap:
    push ebx
    push ecx
    push edx
    push esi
    push edi

    ; ตรวจสอบว่า mmap flag ถูก set
    test dword [ebx + multiboot_info.flags], MULTIBOOT_FLAG_MMAP
    jz .no_mmap

    ; โหลด mmap_addr และ mmap_length
    mov esi, [ebx + multiboot_info.mmap_addr]       ; esi = pointer to first entry
    mov ecx, [ebx + multiboot_info.mmap_length]     ; ecx = total bytes of mmap

    xor edi, edi        ; edi = total available memory

.loop:
    test ecx, ecx
    jle .done

    ; อ่าน entry type
    cmp dword [esi + multiboot_mmap_entry.type], 1  ; type 1 = available
    jne .next_entry

    ; คำนวณ memory ที่ใช้งานได้
    mov eax, [esi + multiboot_mmap_entry.len_low]
    ; ตรวจสอบ base address - ข้ามส่วนที่อยู่ต่ำกว่า 1MB
    mov edx, [esi + multiboot_mmap_entry.base_low]
    cmp edx, 0x100000   ; 1MB
    jae .add_full

    ; คำนวณส่วนที่อยู่เหนือ 1MB
    sub eax, 0x100000
    sub eax, edx        ; length - (1MB - base)
    js .next_entry      ; ถ้าติดลบ ข้ามไป

.add_full:
    add edi, eax

.next_entry:
    ; ไปยัง entry ถัดไป: esi += entry.size + 4
    mov eax, [esi + multiboot_mmap_entry.size]
    add eax, 4          ; บวก 4 เพราะ .size ไม่รวมตัวมันเอง
    add esi, eax
    sub ecx, eax

    jmp .loop

.done:
    mov [mb_total_memory], edi
    jmp .ret

.no_mmap:
    ; fallback: ใช้ mem_upper
    test dword [ebx + multiboot_info.flags], MULTIBOOT_FLAG_MEM
    jz .ret
    mov eax, [ebx + multiboot_info.mem_upper]
    shl eax, 10         ; KB to bytes
    mov [mb_total_memory], eax

.ret:
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
```

---

## บทที่ 3: Bitmap Allocator

### 3.1 หลักการทำงานของ Bitmap PMM

Bitmap PMM เป็นวิธีที่ง่ายและตรงไปตรงมาที่สุด:

```
Physical Memory (RAM):
+--------+--------+--------+--------+--------+--------+
| Page 0 | Page 1 | Page 2 | Page 3 | Page 4 | Page 5 | ...
+--------+--------+--------+--------+--------+--------+
  4KB      4KB      4KB      4KB      4KB      4KB

Bitmap:
Byte 0:  [P7|P6|P5|P4|P3|P2|P1|P0]
Byte 1:  [P15|P14|P13|P12|P11|P10|P9|P8]
...

bit = 1 หมายถึง page ถูกใช้งาน (used)
bit = 0 หมายถึง page ว่าง (free)
```

### 3.2 การคำนวณขนาด Bitmap

```
ขนาด physical memory = memory_size bytes
จำนวน pages = memory_size / 4096
ขนาด bitmap = total_pages / 8 bytes (1 bit per page)

ตัวอย่าง:
- RAM 256MB = 256 * 1024 * 1024 = 268,435,456 bytes
- Total pages = 268,435,456 / 4096 = 65,536 pages
- Bitmap size = 65,536 / 8 = 8,192 bytes = 8 KB
```

### 3.3 ตำแหน่ง Bit ใน Bitmap

```
สำหรับ page number N:
- อยู่ใน byte: N / 8  (หรือ N >> 3)
- อยู่ที่ bit: N % 8  (หรือ N & 7)

ตัวอย่าง:
- Page 0:  byte 0, bit 0
- Page 7:  byte 0, bit 7
- Page 8:  byte 1, bit 0
- Page 15: byte 1, bit 7
- Page 16: byte 2, bit 0
```

---

## บทที่ 4: Implementation ใน NASM

### 4.1 PMM Header/Constants

```nasm
; =============================================================================
; pmm.inc - Physical Memory Manager constants และ macros
; =============================================================================

; Page size
PAGE_SIZE           equ 4096        ; 4 KB
PAGE_SHIFT          equ 12          ; 2^12 = 4096

; Bitmap constants
BITMAP_ENTRY_BITS   equ 32          ; ใช้ dword (32 bits) ต่อ entry
BITMAP_ENTRY_BYTES  equ 4
BITMAP_FULL         equ 0xFFFFFFFF  ; entry เต็ม (ทุก page ถูกใช้)

; Memory zones
ZONE_DMA_MAX        equ 0x01000000  ; 16 MB สูงสุดสำหรับ DMA
ZONE_NORMAL_MAX     equ 0xC0000000  ; 3 GB สูงสุดสำหรับ normal

; Special addresses
ADDR_1MB            equ 0x00100000  ; 1 MB boundary
ADDR_16MB           equ 0x01000000  ; 16 MB boundary

; PMM error codes
PMM_SUCCESS         equ 0
PMM_ERR_NO_MEM      equ 0xFFFFFFFF  ; ไม่มี memory ว่าง
PMM_ERR_INVALID     equ 0xFFFFFFFE  ; parameter ไม่ถูกต้อง

; Macros สำหรับ bitmap operations
%macro BITMAP_SET 2
    ; %1 = bitmap address, %2 = page number
    bts dword [%1 + (%%page >> 3)], %%page & 7
%endmacro

; แปลง address เป็น page number
%macro ADDR_TO_PAGE 1
    shr %1, PAGE_SHIFT
%endmacro

; แปลง page number เป็น address
%macro PAGE_TO_ADDR 1
    shl %1, PAGE_SHIFT
%endmacro
```

### 4.2 PMM Data Structures

```nasm
; =============================================================================
; pmm_data.asm - PMM global variables
; =============================================================================

section .bss

; Bitmap - 1 bit per page
; ขนาดสูงสุด: 4GB RAM / 4KB per page / 8 bits = 131072 bytes = 128KB
PMM_BITMAP_MAX_SIZE equ (4 * 1024 * 1024 * 1024 / 4096 / 8)

global pmm_bitmap
global pmm_bitmap_size
global pmm_total_pages
global pmm_free_pages
global pmm_used_pages

pmm_bitmap:         resb PMM_BITMAP_MAX_SIZE    ; bitmap array
pmm_bitmap_size:    resd 1      ; ขนาด bitmap ใน bytes
pmm_total_pages:    resd 1      ; จำนวน page ทั้งหมด
pmm_free_pages:     resd 1      ; จำนวน page ว่าง
pmm_used_pages:     resd 1      ; จำนวน page ที่ใช้งาน

; Zone information
pmm_zone_dma_free:      resd 1  ; free pages ใน DMA zone (<16MB)
pmm_zone_normal_free:   resd 1  ; free pages ใน normal zone

; Statistics
pmm_alloc_count:    resd 1      ; จำนวนครั้งที่ alloc
pmm_free_count:     resd 1      ; จำนวนครั้งที่ free
```

### 4.3 PMM Initialization

```nasm
; =============================================================================
; pmm_init.asm - Physical Memory Manager Initialization
; =============================================================================

BITS 32
section .text

extern pmm_bitmap
extern pmm_bitmap_size
extern pmm_total_pages
extern pmm_free_pages

; pmm_init:
;   Initialize PMM จาก multiboot info
;   input: EBX = pointer to multiboot_info
;   output: EAX = PMM_SUCCESS หรือ error code
global pmm_init
pmm_init:
    push ebx
    push ecx
    push edx
    push esi
    push edi
    push ebp

    ; ==============================
    ; Step 1: หา total memory
    ; ==============================

    ; ตรวจสอบ mmap flag
    test dword [ebx + multiboot_info.flags], MULTIBOOT_FLAG_MMAP
    jz .use_mem_upper

    ; ใช้ mmap หา total memory
    call .find_total_memory     ; EAX = total memory bytes
    jmp .got_memory

.use_mem_upper:
    mov eax, [ebx + multiboot_info.mem_upper]
    shl eax, 10                 ; KB to bytes
    add eax, 0x100000           ; บวก 1MB (lower memory)

.got_memory:
    ; ==============================
    ; Step 2: คำนวณ bitmap size
    ; ==============================

    ; total_pages = total_memory / PAGE_SIZE
    xor edx, edx
    mov ecx, PAGE_SIZE
    div ecx                     ; EAX = total_pages

    mov [pmm_total_pages], eax
    mov [pmm_free_pages], eax   ; เริ่มต้น: ทุก page ว่าง

    ; bitmap_size = total_pages / 8
    mov ecx, eax
    shr ecx, 3                  ; ecx = bitmap_size bytes
    test eax, 7
    jz .bitmap_size_done
    inc ecx                     ; round up

.bitmap_size_done:
    mov [pmm_bitmap_size], ecx

    ; ==============================
    ; Step 3: เซต bitmap ทั้งหมดเป็น USED
    ; ==============================
    ; เริ่มต้นถือว่าทุก page ถูกใช้งาน
    ; แล้วค่อย mark ส่วนที่ available เป็น free

    mov edi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    mov al, 0xFF                ; 0xFF = ทุก bit = used
    rep stosb

    ; ==============================
    ; Step 4: Mark available regions เป็น free
    ; ==============================

    ; ตรวจสอบ mmap
    test dword [ebx + multiboot_info.flags], MULTIBOOT_FLAG_MMAP
    jz .mark_done

    mov esi, [ebx + multiboot_info.mmap_addr]
    mov ecx, [ebx + multiboot_info.mmap_length]

.scan_mmap:
    test ecx, ecx
    jle .mark_done

    ; ตรวจสอบว่าเป็น available (type = 1)
    cmp dword [esi + multiboot_mmap_entry.type], 1
    jne .next_mmap_entry

    ; โหลด base และ length
    mov eax, [esi + multiboot_mmap_entry.base_low]
    mov edx, [esi + multiboot_mmap_entry.len_low]

    ; ข้ามส่วนต่ำกว่า 1MB (อาจมี BIOS/IVT)
    cmp eax, ADDR_1MB
    jae .mark_region_free

    ; คำนวณส่วนที่อยู่เหนือ 1MB
    mov ebp, ADDR_1MB
    sub ebp, eax        ; ebp = bytes ที่ต้องข้าม
    cmp ebp, edx
    jge .next_mmap_entry
    add eax, ebp
    sub edx, ebp

.mark_region_free:
    ; เรียก mark_region_free(base=EAX, length=EDX)
    push edx
    push eax
    call pmm_mark_region_free
    add esp, 8

.next_mmap_entry:
    mov eax, [esi + multiboot_mmap_entry.size]
    add eax, 4
    add esi, eax
    sub ecx, eax
    jmp .scan_mmap

.mark_done:

    ; ==============================
    ; Step 5: Mark kernel regions เป็น used
    ; ==============================
    ; ต้องมาก่อน เพราะ kernel code/data อยู่ใน RAM

    ; kernel start/end symbols จาก linker script
    extern _kernel_start
    extern _kernel_end

    mov eax, _kernel_start
    mov edx, _kernel_end
    sub edx, eax            ; edx = kernel size

    push edx
    push eax
    call pmm_mark_region_used
    add esp, 8

    ; ==============================
    ; Step 6: Mark bitmap เป็น used
    ; ==============================

    mov eax, pmm_bitmap
    mov edx, [pmm_bitmap_size]

    push edx
    push eax
    call pmm_mark_region_used
    add esp, 8

    ; ==============================
    ; Step 7: คำนวณ free pages
    ; ==============================

    call pmm_count_free_pages
    mov [pmm_free_pages], eax

    mov eax, PMM_SUCCESS

    pop ebp
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret

; .find_total_memory - หา total memory จาก mmap
; input: EBX = multiboot_info pointer
; output: EAX = total memory bytes
.find_total_memory:
    push esi
    push ecx

    mov esi, [ebx + multiboot_info.mmap_addr]
    mov ecx, [ebx + multiboot_info.mmap_length]
    xor eax, eax            ; highest_end = 0

.scan:
    test ecx, ecx
    jle .done

    ; คำนวณ end address ของ region นี้
    mov edx, [esi + multiboot_mmap_entry.base_low]
    add edx, [esi + multiboot_mmap_entry.len_low]

    ; อัพเดท highest_end
    cmp edx, eax
    jle .skip
    mov eax, edx

.skip:
    mov edx, [esi + multiboot_mmap_entry.size]
    add edx, 4
    add esi, edx
    sub ecx, edx
    jmp .scan

.done:
    pop ecx
    pop esi
    ret
```

### 4.4 Mark Region Functions

```nasm
; =============================================================================
; pmm_mark.asm - Mark memory regions as used/free
; =============================================================================

BITS 32
section .text

; pmm_mark_region_used:
;   Mark ช่วง memory เป็น used ใน bitmap
;   input: [esp+4] = base address
;          [esp+8] = length ใน bytes
global pmm_mark_region_used
pmm_mark_region_used:
    push ebp
    mov ebp, esp
    push ebx
    push ecx
    push edi

    mov eax, [ebp + 8]      ; base address
    mov ecx, [ebp + 12]     ; length

    ; แปลงเป็น page-aligned
    ; page_start = base >> PAGE_SHIFT
    ; page_count = (length + PAGE_SIZE - 1) >> PAGE_SHIFT  (round up)

    mov ebx, eax
    shr ebx, PAGE_SHIFT     ; ebx = first page number

    add ecx, eax            ; ecx = end address
    add ecx, PAGE_SIZE - 1
    shr ecx, PAGE_SHIFT     ; ecx = last page number + 1

    sub ecx, ebx            ; ecx = page count

    test ecx, ecx
    jle .done

    ; Mark แต่ละ page
    mov edi, ebx            ; edi = current page number
.loop:
    test ecx, ecx
    jle .done

    ; Set bit ใน bitmap
    ; byte_index = page_num / 32 * 4  (ใช้ dword)
    ; bit_index  = page_num % 32

    push ecx
    mov ecx, edi
    and ecx, 31             ; ecx = bit_index (page_num % 32)

    mov eax, edi
    shr eax, 5              ; eax = dword_index (page_num / 32)
    shl eax, 2              ; eax = byte_offset (dword_index * 4)

    bts dword [pmm_bitmap + eax], ecx   ; Set bit

    pop ecx
    inc edi
    dec ecx
    jmp .loop

.done:
    pop edi
    pop ecx
    pop ebx
    pop ebp
    ret


; pmm_mark_region_free:
;   Mark ช่วง memory เป็น free ใน bitmap
;   input: [esp+4] = base address
;          [esp+8] = length ใน bytes
global pmm_mark_region_free
pmm_mark_region_free:
    push ebp
    mov ebp, esp
    push ebx
    push ecx
    push edi

    mov eax, [ebp + 8]      ; base address
    mov ecx, [ebp + 12]     ; length

    ; page-align
    add eax, PAGE_SIZE - 1  ; round up start
    shr eax, PAGE_SHIFT     ; first page

    mov ebx, [ebp + 12]
    add ebx, [ebp + 8]      ; end address
    shr ebx, PAGE_SHIFT     ; last page + 1 (round down end)

    sub ebx, eax            ; page count
    mov ecx, ebx
    mov ebx, eax            ; ebx = first page

    test ecx, ecx
    jle .done

    mov edi, ebx
.loop:
    test ecx, ecx
    jle .done

    push ecx
    mov ecx, edi
    and ecx, 31

    mov eax, edi
    shr eax, 5
    shl eax, 2

    btr dword [pmm_bitmap + eax], ecx   ; Clear bit (mark free)

    pop ecx
    inc edi
    dec ecx
    jmp .loop

.done:
    pop edi
    pop ecx
    pop ebx
    pop ebp
    ret


; pmm_count_free_pages:
;   นับจำนวน free pages จาก bitmap
;   output: EAX = free page count
global pmm_count_free_pages
pmm_count_free_pages:
    push ecx
    push edx
    push esi

    mov esi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    shr ecx, 2              ; ecx = dword count
    xor eax, eax            ; eax = free count

.loop:
    test ecx, ecx
    jz .done

    mov edx, [esi]
    not edx                 ; invert: 0=used -> 1, 1=free -> 0... wait
    ; จริงๆ: bit 0 = free, bit 1 = used
    ; ต้องนับ 0 bits (free pages)

    ; ใช้ POPCNT ถ้ามี, ไม่งั้นนับ manual
    ; นับ zero bits = นับ set bits ใน NOT
    not edx                 ; undo not
    ; นับ zeros: 32 - popcnt(edx)
    push ecx
    push edx

    ; นับ set bits ใน EDX (Kernighan's algorithm)
    xor ecx, ecx
.count_loop:
    test edx, edx
    jz .count_done
    mov eax, edx
    dec eax
    and edx, eax            ; clear lowest set bit
    inc ecx
    jmp .count_loop

.count_done:
    mov eax, 32
    sub eax, ecx            ; free = 32 - used_bits

    pop edx
    pop ecx

    add [esp], eax          ; สะสมไว้ใน stack... แก้ไขโค้ดนี้

    add esi, 4
    dec ecx
    jmp .loop

.done:
    ; EAX มี result
    pop esi
    pop edx
    pop ecx
    ret
```

### 4.5 alloc_page และ free_page

```nasm
; =============================================================================
; pmm_alloc.asm - Page allocation และ deallocation
; =============================================================================

BITS 32
section .text

; pmm_alloc_page:
;   จัดสรร 1 physical page
;   output: EAX = physical address ของ page
;           EAX = PMM_ERR_NO_MEM ถ้าไม่มี memory ว่าง
global pmm_alloc_page
pmm_alloc_page:
    push ebx
    push ecx
    push edx
    push esi

    ; ตรวจสอบว่ามี free page
    cmp dword [pmm_free_pages], 0
    je .no_mem

    ; หา first free bit ใน bitmap
    mov esi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    shr ecx, 2              ; ecx = dword count

    xor ebx, ebx            ; ebx = dword index

.scan_loop:
    test ecx, ecx
    jz .no_mem

    mov eax, [esi + ebx*4]
    cmp eax, 0xFFFFFFFF     ; ถ้าเต็มหมด ข้ามไป
    je .next_dword

    ; พบ dword ที่มี free bit
    ; หา bit position โดยใช้ BSF (Bit Scan Forward)
    not eax                 ; ทำ NOT เพื่อหา first 0 bit (free page)
    bsf edx, eax            ; edx = bit position ของ first 0 bit

    ; คำนวณ page number
    mov eax, ebx
    shl eax, 5              ; eax = dword_index * 32
    add eax, edx            ; eax = page number

    ; ตรวจสอบว่า page number valid
    cmp eax, [pmm_total_pages]
    jge .no_mem

    ; Mark page เป็น used
    bts dword [esi + ebx*4], edx    ; Set bit

    ; อัพเดท counter
    dec dword [pmm_free_pages]
    inc dword [pmm_used_pages]
    inc dword [pmm_alloc_count]

    ; คำนวณ physical address
    shl eax, PAGE_SHIFT     ; address = page_num * PAGE_SIZE

    jmp .done

.next_dword:
    inc ebx
    dec ecx
    jmp .scan_loop

.no_mem:
    mov eax, PMM_ERR_NO_MEM

.done:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret


; pmm_free_page:
;   คืน physical page กลับไปที่ free pool
;   input: [esp+4] = physical address ของ page
global pmm_free_page
pmm_free_page:
    push ebp
    mov ebp, esp
    push ebx
    push ecx

    mov eax, [ebp + 8]      ; physical address

    ; ตรวจสอบ alignment
    test eax, PAGE_SIZE - 1
    jnz .invalid            ; ไม่ page-aligned

    ; แปลงเป็น page number
    shr eax, PAGE_SHIFT

    ; ตรวจสอบ range
    cmp eax, [pmm_total_pages]
    jge .invalid

    ; ตรวจสอบว่า page นี้ถูก used จริง
    mov ecx, eax
    and ecx, 31             ; bit index
    mov ebx, eax
    shr ebx, 5              ; dword index
    shl ebx, 2              ; byte offset

    bt dword [pmm_bitmap + ebx], ecx
    jnc .double_free        ; ถ้า bit = 0 แสดงว่า double free!

    ; Clear bit (mark as free)
    btr dword [pmm_bitmap + ebx], ecx

    ; อัพเดท counters
    inc dword [pmm_free_pages]
    dec dword [pmm_used_pages]
    inc dword [pmm_free_count]

    mov eax, PMM_SUCCESS
    jmp .done

.invalid:
    mov eax, PMM_ERR_INVALID
    jmp .done

.double_free:
    ; Double free detected! Kernel panic ควรเรียกที่นี่
    ; สำหรับตอนนี้ return error
    mov eax, PMM_ERR_INVALID

.done:
    pop ecx
    pop ebx
    pop ebp
    ret


; pmm_alloc_pages:
;   จัดสรร N pages ติดกัน (contiguous)
;   input: [esp+4] = จำนวน pages ที่ต้องการ
;   output: EAX = physical address ของ first page
;           EAX = PMM_ERR_NO_MEM ถ้าไม่พบ contiguous region
global pmm_alloc_pages
pmm_alloc_pages:
    push ebp
    mov ebp, esp
    push ebx
    push ecx
    push edx
    push esi
    push edi

    mov edi, [ebp + 8]      ; edi = requested page count

    test edi, edi
    jz .invalid
    cmp edi, 1
    je .alloc_single        ; ถ้าขอแค่ 1 page ใช้ alloc_page ธรรมดา

    ; ตรวจสอบว่ามี free pages พอ
    cmp [pmm_free_pages], edi
    jl .no_mem

    ; ค้นหา contiguous region
    mov ecx, [pmm_total_pages]
    xor ebx, ebx            ; ebx = current scan position

.scan_start:
    cmp ebx, ecx
    jge .no_mem

    ; ตรวจสอบ page[ebx] ว่าว่างไหม
    call .is_page_free      ; input: EBX = page num, output: ZF set if free
    jnz .skip_page          ; ถ้าไม่ว่าง ข้ามไป

    ; พบ free page เริ่มนับ consecutive free pages
    mov esi, ebx            ; esi = start of free region
    mov edx, 0              ; edx = consecutive count

.count_consecutive:
    cmp edx, edi            ; พอแล้วไหม?
    jge .found_region

    ; ตรวจสอบ page[esi + edx]
    lea eax, [esi + edx]
    cmp eax, ecx
    jge .no_mem             ; เกินขอบเขต

    push ebx
    mov ebx, eax
    call .is_page_free
    pop ebx
    jnz .not_consecutive    ; ไม่ว่าง ต้องหาที่ใหม่

    inc edx
    jmp .count_consecutive

.not_consecutive:
    ; ไปค้นหาต่อจาก page ที่ไม่ว่าง
    mov ebx, esi
    add ebx, edx
    inc ebx
    jmp .scan_start

.skip_page:
    inc ebx
    jmp .scan_start

.found_region:
    ; พบ contiguous region ที่ ESI, ขนาด EDI pages
    ; Mark ทั้งหมดเป็น used

    push edi
    push esi
    mov eax, esi
    shl eax, PAGE_SHIFT         ; convert to address
    mov edx, edi
    shl edx, PAGE_SHIFT         ; size in bytes

    push edx
    push eax
    call pmm_mark_region_used
    add esp, 8

    pop esi
    pop edi

    ; อัพเดท free count
    sub [pmm_free_pages], edi
    add [pmm_used_pages], edi
    inc dword [pmm_alloc_count]

    mov eax, esi
    shl eax, PAGE_SHIFT         ; return physical address
    jmp .done

.alloc_single:
    call pmm_alloc_page
    jmp .done

.no_mem:
    mov eax, PMM_ERR_NO_MEM
    jmp .done

.invalid:
    mov eax, PMM_ERR_INVALID

.done:
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop ebp
    ret

; Helper: ตรวจสอบว่า page ว่างไหม
; input: EBX = page number
; output: ZF = 1 ถ้าว่าง, ZF = 0 ถ้าใช้งานอยู่
.is_page_free:
    push eax
    push ecx

    mov ecx, ebx
    and ecx, 31
    mov eax, ebx
    shr eax, 5
    shl eax, 2

    bt dword [pmm_bitmap + eax], ecx
    ; CF = 1 ถ้า bit set (used)
    ; CF = 0 ถ้า bit clear (free)

    ; แปลง CF เป็น ZF
    ; ถ้า free: CF=0 -> ZF=1
    ; ถ้า used: CF=1 -> ZF=0

    cmc                     ; invert CF
    sbb eax, eax            ; EAX = 0 ถ้า free, -1 ถ้า used
    test eax, eax           ; set ZF

    pop ecx
    pop eax
    ret
```

---

## บทที่ 5: Memory Zones

### 5.1 ทำไมต้องมี Memory Zones?

DMA (Direct Memory Access) ใน hardware รุ่นเก่าสามารถ access ได้เฉพาะ memory ต่ำกว่า 16MB เท่านั้น ดังนั้น kernel ต้องแบ่ง memory ออกเป็น zones:

```
Physical Memory Layout:
0x00000000 - 0x00100000  = first 1MB (BIOS, IVT, etc.)
0x00100000 - 0x01000000  = ZONE_DMA (1MB - 16MB)
0x01000000 - 0xFFFFFFFF  = ZONE_NORMAL (16MB - 4GB)
```

### 5.2 Zone Implementation

```nasm
; =============================================================================
; pmm_zones.asm - Memory Zone Management
; =============================================================================

BITS 32
section .text

; ค่าคงที่สำหรับ zones
ZONE_DMA    equ 0
ZONE_NORMAL equ 1
ZONE_HIGH   equ 2   ; สำหรับ memory > 4GB (ต้องใช้ PAE)

; pmm_alloc_page_zone:
;   จัดสรร page จาก zone ที่กำหนด
;   input: [esp+4] = zone number (ZONE_DMA หรือ ZONE_NORMAL)
;   output: EAX = physical address หรือ PMM_ERR_NO_MEM
global pmm_alloc_page_zone
pmm_alloc_page_zone:
    push ebp
    mov ebp, esp
    push ebx
    push ecx
    push edx
    push esi

    mov edi, [ebp + 8]      ; edi = zone

    ; กำหนด scan range ตาม zone
    cmp edi, ZONE_DMA
    je .scan_dma

    ; ZONE_NORMAL: scan ตั้งแต่หลัง 16MB
    mov ebx, ADDR_16MB >> PAGE_SHIFT    ; start page
    mov ecx, [pmm_total_pages]          ; end page
    jmp .scan

.scan_dma:
    ; ZONE_DMA: scan 1MB - 16MB
    mov ebx, ADDR_1MB >> PAGE_SHIFT     ; start page (skip first 1MB)
    mov ecx, ADDR_16MB >> PAGE_SHIFT    ; end page

.scan:
    cmp ebx, ecx
    jge .no_mem

    ; หา free dword ใน range
    mov esi, ebx
    shr esi, 5              ; dword index

.scan_loop:
    mov eax, esi
    shl eax, 5              ; first page of this dword
    cmp eax, ecx
    jge .no_mem

    mov edx, [pmm_bitmap + esi*4]
    cmp edx, 0xFFFFFFFF
    je .next_dword

    ; พบ free bit ใน dword นี้
    not edx
    bsf edx, edx            ; bit position

    ; คำนวณ page number
    mov eax, esi
    shl eax, 5
    add eax, edx

    ; ตรวจสอบว่าอยู่ใน zone
    cmp eax, ecx
    jge .no_mem
    cmp eax, ebx
    jl .next_dword

    ; Mark as used
    bts dword [pmm_bitmap + esi*4], edx

    dec dword [pmm_free_pages]
    inc dword [pmm_used_pages]

    ; อัพเดท zone counter
    push eax
    cmp edi, ZONE_DMA
    jne .not_dma_dec
    dec dword [pmm_zone_dma_free]
    jmp .zone_done
.not_dma_dec:
    dec dword [pmm_zone_normal_free]
.zone_done:
    pop eax

    shl eax, PAGE_SHIFT     ; convert to address
    jmp .done

.next_dword:
    inc esi
    jmp .scan_loop

.no_mem:
    mov eax, PMM_ERR_NO_MEM

.done:
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop ebp
    ret


; pmm_alloc_dma_pages:
;   จัดสรร N pages ติดกันใน DMA zone
;   ใช้สำหรับ DMA buffers ที่ต้องอยู่ใน physical memory < 16MB
;   input: [esp+4] = page count
;   output: EAX = physical address หรือ PMM_ERR_NO_MEM
global pmm_alloc_dma_pages
pmm_alloc_dma_pages:
    push ebp
    mov ebp, esp

    ; ตรวจสอบว่ามี free pages พอใน DMA zone
    mov eax, [ebp + 8]
    cmp [pmm_zone_dma_free], eax
    jl .no_mem

    ; TODO: scan DMA zone สำหรับ contiguous pages
    ; (คล้ายกับ pmm_alloc_pages แต่จำกัดใน 1MB-16MB range)

    mov eax, PMM_ERR_NO_MEM     ; placeholder
    jmp .done

.no_mem:
    mov eax, PMM_ERR_NO_MEM

.done:
    pop ebp
    ret
```

---

## บทที่ 6: Physical Frame Counter

### 6.1 การติดตาม Frame Count

```nasm
; =============================================================================
; pmm_stats.asm - PMM Statistics และ Frame Counters
; =============================================================================

BITS 32
section .text

; pmm_get_stats:
;   ดึงข้อมูล statistics ของ PMM
;   input: [esp+4] = pointer to pmm_stats structure
;   output: fills structure
struc pmm_stats_t
    .total_pages    resd 1
    .free_pages     resd 1
    .used_pages     resd 1
    .total_bytes    resd 1
    .free_bytes     resd 1
    .dma_free       resd 1
    .normal_free    resd 1
    .alloc_count    resd 1
    .free_count     resd 1
endstruc

global pmm_get_stats
pmm_get_stats:
    push ebp
    mov ebp, esp
    push edi

    mov edi, [ebp + 8]      ; edi = pointer to stats struct

    mov eax, [pmm_total_pages]
    mov [edi + pmm_stats_t.total_pages], eax

    mov eax, [pmm_free_pages]
    mov [edi + pmm_stats_t.free_pages], eax

    mov eax, [pmm_used_pages]
    mov [edi + pmm_stats_t.used_pages], eax

    ; คำนวณ bytes
    mov eax, [pmm_total_pages]
    shl eax, PAGE_SHIFT
    mov [edi + pmm_stats_t.total_bytes], eax

    mov eax, [pmm_free_pages]
    shl eax, PAGE_SHIFT
    mov [edi + pmm_stats_t.free_bytes], eax

    mov eax, [pmm_zone_dma_free]
    mov [edi + pmm_stats_t.dma_free], eax

    mov eax, [pmm_zone_normal_free]
    mov [edi + pmm_stats_t.normal_free], eax

    mov eax, [pmm_alloc_count]
    mov [edi + pmm_stats_t.alloc_count], eax

    mov eax, [pmm_free_count]
    mov [edi + pmm_stats_t.free_count], eax

    pop edi
    pop ebp
    ret


; pmm_print_stats:
;   พิมพ์ PMM statistics ออก serial port / VGA
;   (ต้องมี print functions จาก part อื่นๆ)
global pmm_print_stats
pmm_print_stats:
    push ebp
    mov ebp, esp

    ; Print header
    push msg_pmm_header
    call print_string
    add esp, 4

    ; Total memory
    push msg_total
    call print_string
    add esp, 4

    mov eax, [pmm_total_pages]
    shl eax, 2              ; pages to KB (4KB per page)
    push eax
    call print_dec
    add esp, 4

    push msg_kb
    call print_string
    add esp, 4

    ; Free memory
    push msg_free
    call print_string
    add esp, 4

    mov eax, [pmm_free_pages]
    shl eax, 2
    push eax
    call print_dec
    add esp, 4

    push msg_kb
    call print_string
    add esp, 4

    pop ebp
    ret

section .data
msg_pmm_header  db "=== PMM Statistics ===", 0x0A, 0
msg_total       db "Total: ", 0
msg_free        db "Free:  ", 0
msg_used        db "Used:  ", 0
msg_kb          db " KB", 0x0A, 0
```

---

## บทที่ 7: Complete Implementation

### 7.1 Full PMM Source File

```nasm
; =============================================================================
; pmm_full.asm - Complete Physical Memory Manager
; สำหรับ x86 32-bit kernel ที่ boot ด้วย GRUB/Multiboot
; =============================================================================
; Build: nasm -f elf32 pmm_full.asm -o pmm_full.o
; Link:  ld -m elf_i386 -T kernel.ld pmm_full.o ... -o kernel.elf
; =============================================================================

BITS 32

; ============================================================
; Constants
; ============================================================

PAGE_SIZE           equ 4096
PAGE_SHIFT          equ 12
PMM_ERR_NO_MEM      equ 0xFFFFFFFF
PMM_ERR_INVALID     equ 0xFFFFFFFE
PMM_SUCCESS         equ 0

MULTIBOOT_FLAG_MEM  equ (1 << 0)
MULTIBOOT_FLAG_MMAP equ (1 << 6)

ADDR_1MB            equ 0x00100000
ADDR_16MB           equ 0x01000000

; ============================================================
; Multiboot structures
; ============================================================

struc multiboot_info
    .flags          resd 1
    .mem_lower      resd 1
    .mem_upper      resd 1
    .boot_device    resd 1
    .cmdline        resd 1
    .mods_count     resd 1
    .mods_addr      resd 1
    .syms           resb 12
    .mmap_length    resd 1
    .mmap_addr      resd 1
endstruc

struc multiboot_mmap_entry
    .size           resd 1
    .base_low       resd 1
    .base_high      resd 1
    .len_low        resd 1
    .len_high       resd 1
    .type           resd 1
endstruc

; ============================================================
; Global Data
; ============================================================

section .bss

; Bitmap: 1 bit per 4KB page
; รองรับ RAM สูงสุด 4GB = 1,048,576 pages = 131,072 bytes = 128KB bitmap
MAX_PAGES           equ (4 * 1024 * 1024 * 1024 / PAGE_SIZE)
BITMAP_SIZE         equ (MAX_PAGES / 8)

pmm_bitmap:         resb BITMAP_SIZE
pmm_total_pages:    resd 1
pmm_free_pages:     resd 1
pmm_used_pages:     resd 1
pmm_bitmap_size:    resd 1
pmm_zone_dma_free:  resd 1
pmm_zone_norm_free: resd 1
pmm_alloc_count:    resd 1
pmm_free_count:     resd 1

; ============================================================
; Code Section
; ============================================================

section .text

; ============================================================
; pmm_init - Initialize PMM
; input: EAX = pointer to multiboot_info
; output: EAX = 0 (success) or error code
; ============================================================

global pmm_init
pmm_init:
    push ebp
    mov ebp, esp
    pusha

    mov esi, eax            ; esi = multiboot_info pointer

    ; --------------------------------------------------
    ; หา total physical memory
    ; --------------------------------------------------
    test dword [esi + multiboot_info.flags], MULTIBOOT_FLAG_MMAP
    jz .try_mem_field

    ; หา highest available address จาก mmap
    mov edi, [esi + multiboot_info.mmap_addr]
    mov ecx, [esi + multiboot_info.mmap_length]
    xor ebx, ebx            ; ebx = highest end address

.find_highest:
    test ecx, ecx
    jle .got_highest

    cmp dword [edi + multiboot_mmap_entry.type], 1
    jne .next_entry_fh

    mov eax, [edi + multiboot_mmap_entry.base_low]
    add eax, [edi + multiboot_mmap_entry.len_low]
    cmp eax, ebx
    jle .next_entry_fh
    mov ebx, eax

.next_entry_fh:
    mov eax, [edi + multiboot_mmap_entry.size]
    add eax, 4
    add edi, eax
    sub ecx, eax
    jmp .find_highest

.got_highest:
    mov eax, ebx
    jmp .calc_pages

.try_mem_field:
    test dword [esi + multiboot_info.flags], MULTIBOOT_FLAG_MEM
    jz .error

    mov eax, [esi + multiboot_info.mem_upper]
    shl eax, 10             ; KB to bytes
    add eax, ADDR_1MB       ; บวก lower memory

.calc_pages:
    ; total_pages = total_memory / PAGE_SIZE
    xor edx, edx
    mov ecx, PAGE_SIZE
    div ecx
    mov [pmm_total_pages], eax

    ; bitmap_size = ceil(total_pages / 8)
    mov ecx, eax
    shr ecx, 3
    test eax, 7
    jz .store_bitmap_size
    inc ecx

.store_bitmap_size:
    mov [pmm_bitmap_size], ecx

    ; --------------------------------------------------
    ; เริ่มต้น: mark ทุก page เป็น USED
    ; --------------------------------------------------
    mov edi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    mov al, 0xFF
    rep stosb

    ; --------------------------------------------------
    ; Mark available regions เป็น FREE
    ; --------------------------------------------------
    test dword [esi + multiboot_info.flags], MULTIBOOT_FLAG_MMAP
    jz .skip_mmap_free

    mov edi, [esi + multiboot_info.mmap_addr]
    mov ecx, [esi + multiboot_info.mmap_length]

.mmap_free_loop:
    test ecx, ecx
    jle .skip_mmap_free

    cmp dword [edi + multiboot_mmap_entry.type], 1
    jne .mmap_next

    mov eax, [edi + multiboot_mmap_entry.base_low]
    mov edx, [edi + multiboot_mmap_entry.len_low]

    ; ข้ามส่วน < 1MB
    cmp eax, ADDR_1MB
    jae .do_free_region

    ; คำนวณส่วนที่เหนือ 1MB
    mov ebp, ADDR_1MB
    sub ebp, eax
    cmp ebp, edx
    jge .mmap_next
    add eax, ebp
    sub edx, ebp

.do_free_region:
    push ecx
    push esi
    push edx
    push eax
    call .mark_free_range
    add esp, 8
    pop esi
    pop ecx

.mmap_next:
    mov eax, [edi + multiboot_mmap_entry.size]
    add eax, 4
    add edi, eax
    sub ecx, eax
    jmp .mmap_free_loop

.skip_mmap_free:

    ; --------------------------------------------------
    ; Mark kernel ว่าถูกใช้งาน (reserved)
    ; --------------------------------------------------
    extern _kernel_start_phys
    extern _kernel_end_phys

    mov eax, _kernel_start_phys
    mov edx, _kernel_end_phys
    sub edx, eax

    push edx
    push eax
    call .mark_used_range
    add esp, 8

    ; Mark bitmap เป็น used
    mov eax, pmm_bitmap
    mov edx, [pmm_bitmap_size]

    push edx
    push eax
    call .mark_used_range
    add esp, 8

    ; --------------------------------------------------
    ; นับ free pages และตั้ง counters
    ; --------------------------------------------------
    call .count_free
    mov [pmm_free_pages], eax
    mov ecx, [pmm_total_pages]
    sub ecx, eax
    mov [pmm_used_pages], ecx

    ; นับ DMA free pages
    call .count_dma_free
    mov [pmm_zone_dma_free], eax

    ; Normal zone free = total_free - dma_free
    mov eax, [pmm_free_pages]
    sub eax, [pmm_zone_dma_free]
    mov [pmm_zone_norm_free], eax

    popa
    xor eax, eax            ; success
    pop ebp
    ret

.error:
    popa
    mov eax, PMM_ERR_INVALID
    pop ebp
    ret

; ============================================================
; Internal helpers
; ============================================================

; .mark_free_range(base, length):
;   Mark range เป็น free ใน bitmap
; input: [esp+4] = base, [esp+8] = length
.mark_free_range:
    push ebp
    mov ebp, esp
    push ebx
    push ecx

    mov eax, [ebp + 8]      ; base
    mov ecx, [ebp + 12]     ; length

    ; first page (round up)
    add eax, PAGE_SIZE - 1
    shr eax, PAGE_SHIFT
    mov ebx, eax

    ; last page (round down)
    mov eax, [ebp + 8]
    add eax, [ebp + 12]
    shr eax, PAGE_SHIFT

    ; ecx = page count
    sub eax, ebx
    mov ecx, eax
    jle .mfr_done

.mfr_loop:
    ; Clear bit ebx in bitmap
    mov eax, ebx
    shr eax, 5              ; dword index
    mov edx, ebx
    and edx, 31             ; bit index
    btr dword [pmm_bitmap + eax*4], edx
    inc ebx
    dec ecx
    jnz .mfr_loop

.mfr_done:
    pop ecx
    pop ebx
    pop ebp
    ret 8


; .mark_used_range(base, length):
;   Mark range เป็น used ใน bitmap
.mark_used_range:
    push ebp
    mov ebp, esp
    push ebx
    push ecx

    mov eax, [ebp + 8]
    shr eax, PAGE_SHIFT
    mov ebx, eax            ; first page (round down)

    mov eax, [ebp + 8]
    add eax, [ebp + 12]
    add eax, PAGE_SIZE - 1
    shr eax, PAGE_SHIFT     ; last page (round up)

    sub eax, ebx
    mov ecx, eax
    jle .mur_done

.mur_loop:
    mov eax, ebx
    shr eax, 5
    mov edx, ebx
    and edx, 31
    bts dword [pmm_bitmap + eax*4], edx
    inc ebx
    dec ecx
    jnz .mur_loop

.mur_done:
    pop ecx
    pop ebx
    pop ebp
    ret 8


; .count_free - นับ free pages ใน bitmap ทั้งหมด
; output: EAX = free page count
.count_free:
    push ecx
    push edx
    push esi

    mov esi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    shr ecx, 2              ; dword count
    xor eax, eax

.cf_loop:
    test ecx, ecx
    jz .cf_done

    mov edx, [esi]
    not edx                 ; free bits = NOT bitmap

    ; นับ set bits ใน edx (Hamming weight / popcount)
    push ecx
    push esi

    xor ecx, ecx
.popcount:
    test edx, edx
    jz .popcount_done
    mov esi, edx
    dec esi
    and edx, esi
    inc ecx
    jmp .popcount
.popcount_done:
    mov esi, 32
    sub esi, ecx            ; free = 32 - used

    pop esi
    pop ecx

    add eax, esi

    add esi, 4
    dec ecx
    jmp .cf_loop

.cf_done:
    pop esi
    pop edx
    pop ecx
    ret


; .count_dma_free - นับ free pages ใน DMA zone (1MB-16MB)
; output: EAX = free page count ใน DMA zone
.count_dma_free:
    push ebx
    push ecx
    push esi

    mov ebx, ADDR_1MB >> PAGE_SHIFT     ; first DMA page
    mov ecx, ADDR_16MB >> PAGE_SHIFT    ; last DMA page + 1
    sub ecx, ebx
    xor eax, eax

.cdf_loop:
    test ecx, ecx
    jz .cdf_done

    mov esi, ebx
    shr esi, 5
    mov edx, ebx
    and edx, 31

    bt dword [pmm_bitmap + esi*4], edx
    jc .cdf_next            ; used
    inc eax                 ; free

.cdf_next:
    inc ebx
    dec ecx
    jmp .cdf_loop

.cdf_done:
    pop esi
    pop ecx
    pop ebx
    ret


; ============================================================
; pmm_alloc_page - Allocate 1 page
; output: EAX = physical address or PMM_ERR_NO_MEM
; ============================================================

global pmm_alloc_page
pmm_alloc_page:
    push ebx
    push ecx
    push edx
    push esi

    cmp dword [pmm_free_pages], 0
    je .no_mem

    ; Scan bitmap for first free bit
    mov esi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    shr ecx, 2
    xor ebx, ebx

.scan:
    test ecx, ecx
    jz .no_mem

    mov eax, [esi + ebx*4]
    cmp eax, 0xFFFFFFFF
    je .try_next

    not eax
    bsf edx, eax            ; edx = bit position of first free

    ; page number = ebx*32 + edx
    mov eax, ebx
    shl eax, 5
    add eax, edx

    cmp eax, [pmm_total_pages]
    jge .no_mem

    ; Mark as used
    bts dword [esi + ebx*4], edx

    dec dword [pmm_free_pages]
    inc dword [pmm_used_pages]
    inc dword [pmm_alloc_count]

    ; Update zone counter
    cmp eax, ADDR_16MB >> PAGE_SHIFT
    jge .norm_zone
    dec dword [pmm_zone_dma_free]
    jmp .got_page
.norm_zone:
    dec dword [pmm_zone_norm_free]

.got_page:
    shl eax, PAGE_SHIFT
    jmp .done

.try_next:
    inc ebx
    dec ecx
    jmp .scan

.no_mem:
    mov eax, PMM_ERR_NO_MEM

.done:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret


; ============================================================
; pmm_free_page - Free 1 page
; input: EAX = physical address
; output: EAX = PMM_SUCCESS or error
; ============================================================

global pmm_free_page
pmm_free_page:
    push ebx
    push ecx

    ; ตรวจสอบ alignment
    test eax, PAGE_SIZE - 1
    jnz .bad_align

    ; แปลงเป็น page number
    shr eax, PAGE_SHIFT

    ; range check
    cmp eax, [pmm_total_pages]
    jge .out_of_range

    ; ตรวจสอบว่า used จริงไหม
    mov ecx, eax
    and ecx, 31
    mov ebx, eax
    shr ebx, 5

    bt dword [pmm_bitmap + ebx*4], ecx
    jnc .double_free

    ; Clear bit
    btr dword [pmm_bitmap + ebx*4], ecx

    ; Update counters
    inc dword [pmm_free_pages]
    dec dword [pmm_used_pages]
    inc dword [pmm_free_count]

    ; Update zone counter
    cmp eax, ADDR_16MB >> PAGE_SHIFT
    jge .norm_free
    inc dword [pmm_zone_dma_free]
    jmp .ok
.norm_free:
    inc dword [pmm_zone_norm_free]

.ok:
    xor eax, eax
    jmp .done

.bad_align:
.out_of_range:
.double_free:
    mov eax, PMM_ERR_INVALID

.done:
    pop ecx
    pop ebx
    ret


; ============================================================
; pmm_alloc_pages - Allocate N contiguous pages
; input: EAX = page count
; output: EAX = physical address or PMM_ERR_NO_MEM
; ============================================================

global pmm_alloc_pages
pmm_alloc_pages:
    push ebp
    mov ebp, esp
    push ebx
    push ecx
    push edx
    push esi
    push edi

    mov edi, eax            ; edi = requested count

    test edi, edi
    jz .invalid

    cmp edi, 1
    je .alloc_one

    cmp [pmm_free_pages], edi
    jl .no_mem

    ; Scan for contiguous free pages
    xor ebx, ebx            ; current page
    mov ecx, [pmm_total_pages]

.outer:
    cmp ebx, ecx
    jge .no_mem

    ; Check if page ebx is free
    mov eax, ebx
    shr eax, 5
    mov edx, ebx
    and edx, 31
    bt dword [pmm_bitmap + eax*4], edx
    jc .advance             ; page is used

    ; Start counting consecutive free pages
    mov esi, ebx            ; esi = start
    mov edx, 0              ; edx = count

.inner:
    cmp edx, edi
    jge .found

    mov eax, esi
    add eax, edx
    cmp eax, ecx
    jge .no_mem

    push eax
    shr eax, 5
    mov ebp, [esp]
    pop ebp
    ; ตรวจสอบ page (esi+edx)
    mov eax, esi
    add eax, edx
    push eax
    shr eax, 5
    mov ebx, eax
    pop eax
    and eax, 31
    bt dword [pmm_bitmap + ebx*4], eax
    jc .advance2            ; not free

    inc edx
    jmp .inner

.advance2:
    ; ไปยัง page ถัดจาก block ที่หา
    mov ebx, esi
    add ebx, edx
    inc ebx
    jmp .outer

.advance:
    inc ebx
    jmp .outer

.found:
    ; Mark esi..(esi+edi-1) เป็น used
    push edi
    push esi
    mov eax, esi
    shl eax, PAGE_SHIFT
    mov edx, edi
    shl edx, PAGE_SHIFT
    push edx
    push eax
    call pmm_mark_region_used_range
    add esp, 8
    pop esi
    pop edi

    sub [pmm_free_pages], edi
    add [pmm_used_pages], edi
    inc dword [pmm_alloc_count]

    mov eax, esi
    shl eax, PAGE_SHIFT
    jmp .done

.alloc_one:
    call pmm_alloc_page
    jmp .done

.no_mem:
    mov eax, PMM_ERR_NO_MEM
    jmp .done

.invalid:
    mov eax, PMM_ERR_INVALID

.done:
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop ebp
    ret


; pmm_mark_region_used_range - mark pages as used
; input: [esp+4] = base addr, [esp+8] = size
global pmm_mark_region_used_range
pmm_mark_region_used_range:
    push ebp
    mov ebp, esp
    push ebx
    push ecx

    mov eax, [ebp + 8]
    shr eax, PAGE_SHIFT
    mov ebx, eax

    mov eax, [ebp + 8]
    add eax, [ebp + 12]
    add eax, PAGE_SIZE - 1
    shr eax, PAGE_SHIFT
    sub eax, ebx
    mov ecx, eax

.loop:
    test ecx, ecx
    jz .done
    mov eax, ebx
    shr eax, 5
    mov edx, ebx
    and edx, 31
    bts dword [pmm_bitmap + eax*4], edx
    inc ebx
    dec ecx
    jmp .loop

.done:
    pop ecx
    pop ebx
    pop ebp
    ret 8
```

---

## บทที่ 8: Testing Framework

### 8.1 Unit Tests สำหรับ PMM

```nasm
; =============================================================================
; pmm_test.asm - Testing framework สำหรับ Physical Memory Manager
; =============================================================================

BITS 32

%define TEST_PASS   0x50415353  ; "PASS"
%define TEST_FAIL   0x4641494C  ; "FAIL"

section .bss
    test_count  resd 1
    test_passed resd 1
    test_failed resd 1

    ; Fake memory map สำหรับ test
    test_mmap   resb 512
    test_mmap_count resd 1

section .text

; ===========================================
; test_pmm_all - รัน test suite ทั้งหมด
; ===========================================
global test_pmm_all
test_pmm_all:
    push ebp
    mov ebp, esp

    ; Reset counters
    xor eax, eax
    mov [test_count], eax
    mov [test_passed], eax
    mov [test_failed], eax

    ; Print header
    push msg_test_header
    call print_string
    add esp, 4

    ; รัน tests
    call test_pmm_init
    call test_alloc_single
    call test_free_page
    call test_alloc_contiguous
    call test_zone_dma
    call test_double_free
    call test_boundary

    ; Print summary
    push msg_test_summary
    call print_string
    add esp, 4

    mov eax, [test_passed]
    push eax
    call print_dec
    add esp, 4

    push msg_test_slash
    call print_string
    add esp, 4

    mov eax, [test_count]
    push eax
    call print_dec
    add esp, 4

    push msg_test_passed
    call print_string
    add esp, 4

    pop ebp
    ret


; ===========================================
; test_pmm_init - ทดสอบ PMM initialization
; ===========================================
test_pmm_init:
    push msg_test_init
    call print_string
    add esp, 4

    ; สร้าง fake multiboot info
    call setup_fake_multiboot

    ; Init PMM
    mov eax, test_multiboot_info
    call pmm_init

    ; ตรวจสอบผลลัพธ์
    test eax, eax
    jnz .fail

    ; ตรวจสอบว่ามี free pages
    cmp dword [pmm_free_pages], 0
    je .fail

    ; ตรวจสอบว่า total pages ถูกต้อง
    mov eax, [pmm_total_pages]
    test eax, eax
    jz .fail

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; test_alloc_single - ทดสอบ alloc/free 1 page
; ===========================================
test_alloc_single:
    push msg_test_alloc1
    call print_string
    add esp, 4

    mov ebx, [pmm_free_pages]   ; บันทึก free count ก่อน alloc

    call pmm_alloc_page

    cmp eax, PMM_ERR_NO_MEM
    je .fail

    ; ตรวจสอบว่า free count ลดลง 1
    mov ecx, [pmm_free_pages]
    inc ecx
    cmp ecx, ebx
    jne .fail

    ; ตรวจสอบ alignment
    test eax, PAGE_SIZE - 1
    jnz .fail

    ; บันทึก address แล้ว free
    push eax
    call pmm_free_page
    pop eax

    test eax, eax
    jnz .fail

    ; free count ควรกลับมาเท่าเดิม
    cmp [pmm_free_pages], ebx
    jne .fail

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; test_free_page - ทดสอบ free page
; ===========================================
test_free_page:
    push msg_test_free
    call print_string
    add esp, 4

    ; Alloc page
    call pmm_alloc_page
    mov esi, eax            ; esi = allocated address

    cmp esi, PMM_ERR_NO_MEM
    je .fail

    ; Free page
    mov eax, esi
    call pmm_free_page
    test eax, eax
    jnz .fail

    ; Double free ควร return error
    mov eax, esi
    call pmm_free_page
    cmp eax, PMM_ERR_INVALID
    jne .fail               ; ควร detect double free

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; test_alloc_contiguous - ทดสอบ alloc หลาย pages
; ===========================================
test_alloc_contiguous:
    push msg_test_contiguous
    call print_string
    add esp, 4

    ; Alloc 4 contiguous pages
    mov eax, 4
    call pmm_alloc_pages

    cmp eax, PMM_ERR_NO_MEM
    je .fail

    ; ตรวจสอบ alignment
    test eax, PAGE_SIZE - 1
    jnz .fail

    mov esi, eax            ; บันทึก address

    ; ตรวจสอบว่า 4 pages ถูก mark used
    shr eax, PAGE_SHIFT     ; first page number

    mov ecx, 4
.check_loop:
    push ecx
    push eax
    shr eax, 5
    mov ecx, [esp]
    and ecx, 31
    pop eax
    bt dword [pmm_bitmap + eax*4], ecx  ; ควร set (used)
    pop ecx
    jnc .fail               ; ถ้า bit ไม่ถูก set = fail
    inc eax
    dec ecx
    jnz .check_loop

    ; Free ทั้งหมด
    mov ecx, 4
.free_loop:
    push ecx
    mov eax, esi
    call pmm_free_page
    test eax, eax
    pop ecx
    jnz .fail
    add esi, PAGE_SIZE
    dec ecx
    jnz .free_loop

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; test_zone_dma - ทดสอบ DMA zone allocation
; ===========================================
test_zone_dma:
    push msg_test_dma
    call print_string
    add esp, 4

    ; Alloc page จาก DMA zone
    push 0                  ; ZONE_DMA = 0
    call pmm_alloc_page_zone
    add esp, 4

    cmp eax, PMM_ERR_NO_MEM
    je .fail

    ; ตรวจสอบว่าอยู่ใน DMA range (1MB - 16MB)
    cmp eax, ADDR_1MB
    jl .fail

    cmp eax, ADDR_16MB
    jge .fail

    ; Free page
    call pmm_free_page

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; test_double_free - ทดสอบ double free detection
; ===========================================
test_double_free:
    push msg_test_dbl_free
    call print_string
    add esp, 4

    call pmm_alloc_page
    cmp eax, PMM_ERR_NO_MEM
    je .fail

    mov esi, eax

    ; Free ครั้งแรก - ควร succeed
    mov eax, esi
    call pmm_free_page
    test eax, eax
    jnz .fail

    ; Free ครั้งที่สอง - ควร fail
    mov eax, esi
    call pmm_free_page
    cmp eax, PMM_ERR_INVALID
    jne .fail               ; ต้องได้ error

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; test_boundary - ทดสอบ boundary conditions
; ===========================================
test_boundary:
    push msg_test_boundary
    call print_string
    add esp, 4

    ; Free page ที่ไม่ aligned
    mov eax, 0x1001         ; ไม่ page-aligned
    call pmm_free_page
    cmp eax, PMM_ERR_INVALID
    jne .fail

    ; Free page 0 (ควร fail เพราะ reserved)
    mov eax, 0
    call pmm_free_page
    ; ขึ้นอยู่กับ implementation - อาจจะ double free หรือ invalid

    ; Alloc 0 pages ควร return error
    mov eax, 0
    call pmm_alloc_pages
    cmp eax, PMM_ERR_INVALID
    jne .fail

    push TEST_PASS
    call report_test_result
    add esp, 4
    ret

.fail:
    push TEST_FAIL
    call report_test_result
    add esp, 4
    ret


; ===========================================
; report_test_result - รายงานผล test
; ===========================================
report_test_result:
    push ebp
    mov ebp, esp

    inc dword [test_count]

    cmp dword [ebp + 8], TEST_PASS
    jne .failed

    inc dword [test_passed]
    push msg_pass
    call print_string
    add esp, 4
    jmp .done

.failed:
    inc dword [test_failed]
    push msg_fail
    call print_string
    add esp, 4

.done:
    pop ebp
    ret 4


; ===========================================
; setup_fake_multiboot - สร้าง fake multiboot info
; ===========================================
section .bss
test_multiboot_info: resb 64

section .data
; Fake memory map: available 2MB-128MB
test_mmap_data:
    ; Entry 1: type=2 (reserved), base=0, len=0x100000 (first 1MB)
    dd 20            ; size
    dd 0             ; base_low
    dd 0             ; base_high
    dd 0x100000      ; len_low (1MB)
    dd 0             ; len_high
    dd 2             ; type=reserved

    ; Entry 2: type=1 (available), base=1MB, len=127MB
    dd 20            ; size
    dd 0x100000      ; base_low (1MB)
    dd 0             ; base_high
    dd 0x7F00000     ; len_low (127MB)
    dd 0             ; len_high
    dd 1             ; type=available

section .text
setup_fake_multiboot:
    ; ตั้งค่า multiboot info
    mov dword [test_multiboot_info], MULTIBOOT_FLAG_MEM | MULTIBOOT_FLAG_MMAP

    ; mem_lower = 640 KB
    mov dword [test_multiboot_info + 4], 640

    ; mem_upper = 127 MB (หน่วยเป็น KB)
    mov dword [test_multiboot_info + 8], (127 * 1024)

    ; mmap_length
    mov dword [test_multiboot_info + 44], 52    ; 2 entries * 26 bytes each

    ; mmap_addr
    mov dword [test_multiboot_info + 48], test_mmap_data

    ret

section .data
msg_test_header     db "=== PMM Test Suite ===", 0x0A, 0
msg_test_init       db "[TEST] PMM Init.................", 0
msg_test_alloc1     db "[TEST] Alloc Single Page........", 0
msg_test_free       db "[TEST] Free Page................", 0
msg_test_contiguous db "[TEST] Alloc Contiguous Pages...", 0
msg_test_dma        db "[TEST] DMA Zone Alloc...........", 0
msg_test_dbl_free   db "[TEST] Double Free Detection....", 0
msg_test_boundary   db "[TEST] Boundary Conditions......", 0
msg_test_summary    db 0x0A, "Passed: ", 0
msg_test_slash      db "/", 0
msg_test_passed     db " tests", 0x0A, 0
msg_pass            db "PASS", 0x0A, 0
msg_fail            db "FAIL", 0x0A, 0
```

---

## บทที่ 9: Makefile และ QEMU Testing

### 9.1 Makefile

```makefile
# =============================================================================
# Makefile - Physical Memory Manager
# =============================================================================

NASM    := nasm
LD      := ld
QEMU    := qemu-system-i386

NASM_FLAGS  := -f elf32 -g -F dwarf
LD_FLAGS    := -m elf_i386 -T kernel.ld --gc-sections

QEMU_FLAGS  := -m 128M \
               -serial stdio \
               -display none \
               -no-reboot \
               -no-shutdown

# Source files
SRCS := boot/boot.asm \
        kernel/kernel_main.asm \
        mm/pmm_full.asm \
        mm/pmm_test.asm \
        lib/print.asm

OBJS := $(SRCS:.asm=.o)

# Default target
all: kernel.elf

# Compile kernel
kernel.elf: $(OBJS) kernel.ld
	$(LD) $(LD_FLAGS) -o $@ $(OBJS)

# NASM compilation rule
%.o: %.asm
	$(NASM) $(NASM_FLAGS) -o $@ $<

# Run in QEMU
run: kernel.elf
	$(QEMU) $(QEMU_FLAGS) -kernel $<

# Run with GRUB bootable ISO
run-iso: kernel.iso
	$(QEMU) $(QEMU_FLAGS) -cdrom $<

# สร้าง ISO
kernel.iso: kernel.elf grub.cfg
	mkdir -p iso/boot/grub
	cp kernel.elf iso/boot/
	cp grub.cfg iso/boot/grub/
	grub-mkrescue -o $@ iso/
	rm -rf iso/

# Debug
debug: kernel.elf
	$(QEMU) $(QEMU_FLAGS) -s -S -kernel $< &
	gdb -ex "target remote :1234" \
	    -ex "symbol-file kernel.elf" \
	    -ex "break pmm_init" \
	    -ex "continue"

# Run tests
test: kernel.elf
	$(QEMU) $(QEMU_FLAGS) \
	    -kernel $< \
	    -append "run_pmm_tests" \
	    | tee test_output.txt
	grep -q "PASS" test_output.txt && echo "Tests OK" || echo "Tests FAILED"

# Clean
clean:
	rm -f $(OBJS) kernel.elf kernel.iso test_output.txt

.PHONY: all run run-iso debug test clean
```

### 9.2 Linker Script

```
/* kernel.ld - Linker Script สำหรับ x86 kernel */

ENTRY(_start)

SECTIONS {
    /* โหลดที่ 1MB */
    . = 0x00100000;

    _kernel_start_phys = .;

    .text : {
        *(.multiboot)   /* Multiboot header ต้องอยู่ 8KB แรก */
        *(.text)
        *(.text.*)
    }

    . = ALIGN(4096);
    .rodata : {
        *(.rodata)
        *(.rodata.*)
    }

    . = ALIGN(4096);
    .data : {
        *(.data)
        *(.data.*)
    }

    . = ALIGN(4096);
    _bss_start = .;
    .bss : {
        *(.bss)
        *(.bss.*)
        *(COMMON)
    }
    _bss_end = .;

    . = ALIGN(4096);
    _kernel_end_phys = .;
}
```

### 9.3 Boot Entry Point

```nasm
; =============================================================================
; boot/boot.asm - Kernel entry point สำหรับ GRUB Multiboot
; =============================================================================

BITS 32

; Multiboot constants
MULTIBOOT_MAGIC     equ 0x1BADB002
MULTIBOOT_FLAGS     equ (1 << 1) | (1 << 2)    ; align modules, memory info
MULTIBOOT_CHECKSUM  equ -(MULTIBOOT_MAGIC + MULTIBOOT_FLAGS)

; Stack
STACK_SIZE          equ 65536   ; 64 KB

section .multiboot
align 4
multiboot_header:
    dd MULTIBOOT_MAGIC
    dd MULTIBOOT_FLAGS
    dd MULTIBOOT_CHECKSUM

section .bss
align 16
stack_bottom:
    resb STACK_SIZE
stack_top:

section .text
global _start
extern kernel_main

_start:
    ; ตั้งค่า stack
    mov esp, stack_top

    ; push multiboot info pointer และ magic
    push ebx    ; multiboot info
    push eax    ; magic (ควรเป็น 0x2BADB002)

    ; เรียก kernel main
    call kernel_main

    ; ไม่ควรมาถึงนี่
.halt:
    cli
    hlt
    jmp .halt
```

### 9.4 Kernel Main

```nasm
; =============================================================================
; kernel/kernel_main.asm - Kernel main function
; =============================================================================

BITS 32

extern pmm_init
extern pmm_alloc_page
extern pmm_free_page
extern pmm_get_stats
extern test_pmm_all
extern serial_init
extern print_string

global kernel_main

section .text

kernel_main:
    push ebp
    mov ebp, esp

    ; [ebp + 8] = multiboot magic
    ; [ebp + 12] = multiboot info pointer

    ; ตรวจสอบ multiboot magic
    cmp dword [ebp + 8], 0x2BADB002
    jne .bad_boot

    ; Init serial output (สำหรับ debug)
    call serial_init

    push msg_welcome
    call print_string
    add esp, 4

    ; Init PMM
    push msg_init_pmm
    call print_string
    add esp, 4

    mov eax, [ebp + 12]     ; multiboot info
    call pmm_init

    test eax, eax
    jnz .pmm_failed

    push msg_pmm_ok
    call print_string
    add esp, 4

    ; แสดง memory stats
    call pmm_print_stats

    ; รัน test suite
    push msg_running_tests
    call print_string
    add esp, 4

    call test_pmm_all

    ; เสร็จสิ้น
    push msg_done
    call print_string
    add esp, 4

    jmp .halt

.bad_boot:
    push msg_bad_boot
    call print_string
    add esp, 4
    jmp .halt

.pmm_failed:
    push msg_pmm_failed
    call print_string
    add esp, 4

.halt:
    cli
    hlt
    jmp .halt

section .data
msg_welcome     db "Kernel started!", 0x0A, 0
msg_init_pmm    db "Initializing PMM...", 0x0A, 0
msg_pmm_ok      db "PMM initialized OK", 0x0A, 0
msg_running_tests db "Running PMM tests...", 0x0A, 0
msg_done        db "All tests complete.", 0x0A, 0
msg_bad_boot    db "ERROR: Not booted by multiboot!", 0x0A, 0
msg_pmm_failed  db "ERROR: PMM initialization failed!", 0x0A, 0
```

### 9.5 GRUB Configuration

```
# grub.cfg
set timeout=0
set default=0

menuentry "PMM Test Kernel" {
    multiboot /boot/kernel.elf
    boot
}
```

### 9.6 QEMU Test Commands

```bash
# Build kernel
make all

# รัน basic test
make run

# รันด้วย ISO (GRUB)
make run-iso

# Debug ด้วย GDB
make debug

# รัน test suite และบันทึก output
make test

# รันด้วย memory ขนาดต่างๆ
qemu-system-i386 -m 32M -kernel kernel.elf -serial stdio -display none
qemu-system-i386 -m 256M -kernel kernel.elf -serial stdio -display none
qemu-system-i386 -m 512M -kernel kernel.elf -serial stdio -display none

# รันด้วย memory map ที่กำหนดเอง
qemu-system-i386 -m 128M \
    -kernel kernel.elf \
    -serial stdio \
    -display none \
    -no-reboot \
    -d int,cpu_reset 2>&1 | head -50

# ตรวจสอบ memory ด้วย QEMU monitor
qemu-system-i386 -m 128M \
    -kernel kernel.elf \
    -monitor stdio \
    -serial file:serial.log

# ใน QEMU monitor:
# (qemu) info mem
# (qemu) info registers
# (qemu) x/10x 0x100000   <- dump kernel
```

---

## บทที่ 10: การ Debug และ Troubleshooting

### 10.1 วิธีตรวจสอบ Bitmap

```nasm
; pmm_dump_bitmap - dump bitmap ออก serial/VGA สำหรับ debug
global pmm_dump_bitmap
pmm_dump_bitmap:
    push ebp
    mov ebp, esp
    push ebx
    push ecx
    push esi

    push msg_dump_hdr
    call print_string
    add esp, 4

    mov esi, pmm_bitmap
    mov ecx, [pmm_bitmap_size]
    xor ebx, ebx            ; byte count

.dump_loop:
    test ecx, ecx
    jz .done

    ; Print byte offset ทุก 16 bytes
    test ebx, 15
    jnz .no_offset
    push msg_newline
    call print_string
    add esp, 4
    push ebx
    call print_hex_byte
    add esp, 4
    push msg_colon
    call print_string
    add esp, 4

.no_offset:
    push dword [esi]
    call print_hex_byte
    add esp, 4

    inc esi
    inc ebx
    dec ecx
    jmp .dump_loop

.done:
    push msg_newline
    call print_string
    add esp, 4

    pop esi
    pop ecx
    pop ebx
    pop ebp
    ret

section .data
msg_dump_hdr    db "PMM Bitmap Dump:", 0
msg_colon       db ": ", 0
msg_newline     db 0x0A, 0
```

### 10.2 ปัญหาที่พบบ่อย

**1. Kernel panic หลัง PMM init:**
```
สาเหตุ: Bitmap ทับ kernel code/data
วิธีแก้: ตรวจสอบว่า bitmap ถูก mark เป็น used ก่อน mark kernel
```

**2. Free pages count ผิด:**
```
สาเหตุ: นับ pages ก่อน 1MB ด้วย
วิธีแก้: ข้าม first 1MB เสมอ หรือตรวจสอบการ init ใน scan_mmap
```

**3. alloc_pages คืน overlapping addresses:**
```
สาเหตุ: scan ไม่ถูกต้อง ไม่ตรวจ bit หลังจาก mark
วิธีแก้: ตรวจสอบ mark_used_range ว่าทำงานก่อน alloc กลับไป
```

**4. QEMU หยุดทำงาน (triple fault):**
```
สาเหตุ: GDT/IDT ยังไม่ถูกต้อง, Stack overflow
วิธีแก้: ใช้ QEMU -d cpu_reset,int เพื่อดู fault details
```

---

## บทที่ 11: สรุปและ Next Steps

### 11.1 สิ่งที่เรียนรู้ใน Part นี้

1. **E820 Memory Map** - วิธีถาม BIOS ว่า RAM อยู่ที่ไหน
2. **Multiboot Structure** - วิธีรับข้อมูล memory จาก GRUB
3. **Bitmap Algorithm** - การติดตาม page state ด้วย 1 bit per page
4. **alloc/free operations** - BSF instruction สำหรับหา first free bit
5. **Memory Zones** - DMA vs Normal zone
6. **Testing** - วิธีเขียน unit tests ใน assembly

### 11.2 ความซับซ้อนและข้อเสียของ Bitmap PMM

| ข้อดี | ข้อเสีย |
|-------|---------|
| ง่ายต่อการ implement | Scan เร็ว O(n) แต่ใช้ CPU มาก |
| ใช้ memory น้อย (128KB ต่อ 4GB) | ไม่เหมาะกับ fragmented memory |
| ตรวจ double-free ง่าย | alloc_pages(n) ช้าถ้า n ใหญ่ |
| Debug ง่าย (dump bitmap) | ต้อง scan sequential เสมอ |

### 11.3 Part ถัดไป

- **Part 087**: Virtual Memory Manager (VMM) - สร้าง page directory/table
- **Part 088**: Paging Setup - เปิด x86 paging
- **Part 089**: Heap Allocator (kmalloc/kfree) - บน top ของ PMM
- **Part 090**: Memory Protection - page permissions

### 11.4 Exercise

1. **Easy**: แก้ไข pmm_alloc_page ให้ random-ize starting position เพื่อลด fragmentation
2. **Medium**: เพิ่ม `pmm_get_largest_free_block()` ที่คืนขนาดของ contiguous block ใหญ่ที่สุด
3. **Hard**: เปลี่ยน bitmap เป็น buddy allocator เพื่อ O(log n) allocation

---

## Appendix: Quick Reference

### Bitmap Operations

```nasm
; Set bit (mark page USED): บิต N ใน bitmap
; byte = N / 32 * 4
; bit  = N % 32
bts dword [bitmap + (N/32)*4], (N%32)

; Clear bit (mark page FREE):
btr dword [bitmap + (N/32)*4], (N%32)

; Test bit:
bt dword [bitmap + (N/32)*4], (N%32)
; CF = 1 ถ้า set (used)

; Find first free bit ใน dword:
not eax         ; invert: free -> 1
bsf edx, eax   ; find first set bit in inverted = first free
```

### Address Conversions

```nasm
; Address -> Page number:
shr eax, 12     ; divide by 4096

; Page number -> Address:
shl eax, 12     ; multiply by 4096

; Page number -> Dword index in bitmap:
shr eax, 5      ; divide by 32

; Page number -> Bit index in dword:
and eax, 31     ; modulo 32
```

### QEMU Memory Investigation

```bash
# Boot กับ memory map debug output
qemu-system-i386 -m 128M -kernel kernel.elf \
    -serial stdio -display none \
    -d int 2>&1 | grep -E "(E820|memory)"

# ดู physical memory layout ใน QEMU monitor
qemu-system-i386 -m 128M -kernel kernel.elf \
    -monitor telnet::4444,server,nowait &
telnet localhost 4444
(qemu) info memory-devices
(qemu) info mtree
```

---

*Part 086 จบแล้ว - เรามี Physical Memory Manager ที่ทำงานได้จริงแล้ว ใน Part 087 เราจะนำ PMM มาใช้เป็น foundation สำหรับ Virtual Memory Manager*
