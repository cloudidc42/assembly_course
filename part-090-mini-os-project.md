# Part 090: Mini OS - Complete Project

## บทนำ (Introduction)

ในบทนี้เราจะสร้าง Mini Operating System ที่สมบูรณ์ตั้งแต่ต้นจนจบ โดยรวมทุกสิ่งที่เรียนมาตลอดหลักสูตรเข้าด้วยกัน ได้แก่ Bootloader, Protected Mode, GDT, IDT, Paging, Memory Management, Process Scheduling, Device Drivers, System Calls และ User Mode

Mini OS นี้จะมี:
- 2-Stage Bootloader
- 32-bit Protected Mode Kernel
- Physical Memory Manager
- Kernel Heap Allocator
- VGA Text Console
- Keyboard Driver
- Timer/PIT Driver
- Process Scheduler (Round-Robin)
- System Call Interface (INT 0x80)
- User Mode (Ring 3)
- Simple ELF Loader
- RAM Filesystem
- Simple Shell

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  User Space (Ring 3)             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  shell   │  │ hello_us │  │  user progs  │   │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │              │               │            │
│       └──────────────┴───────────────┘            │
│                      │ INT 0x80                   │
├──────────────────────┼─────────────────────────── │
│                  Kernel Space (Ring 0)            │
│  ┌────────────────────────────────────────────┐  │
│  │           System Call Interface            │  │
│  └───────┬────────────────────────────────────┘  │
│          │                                        │
│  ┌───────▼────────┐  ┌─────────────────────────┐ │
│  │   Scheduler    │  │     VGA Console          │ │
│  │  (Round-Robin) │  │  (Text Mode 80x25)       │ │
│  └───────┬────────┘  └─────────────────────────┘ │
│          │                                        │
│  ┌───────▼────────┐  ┌──────────────────────────┐│
│  │ Process Manager│  │   Kernel Heap (Slab)     ││
│  │   (PCB/fork)   │  │                          ││
│  └───────┬────────┘  └──────────────────────────┘│
│          │                                        │
│  ┌───────▼────────┐  ┌──────────────────────────┐│
│  │  ELF Loader    │  │  Physical Memory Mgr     ││
│  │                │  │  (Bitmap Allocator)      ││
│  └───────┬────────┘  └──────────────────────────┘│
│          │                                        │
│  ┌───────▼────────────────────────────────────┐  │
│  │              RAM Filesystem                │  │
│  └────────────────────────────────────────────┘  │
│                                                   │
│  ┌──────────────────────────────────────────────┐ │
│  │           Hardware Abstraction               │ │
│  │  GDT │ IDT │ PIC │ PIT │ Paging │ TSS       │ │
│  └──────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
│                                                   │
│  Stage 2 Bootloader (Protected Mode Switch)       │
│  Stage 1 Bootloader (16-bit Real Mode, 512 bytes) │
│                  BIOS / Hardware                  │
└─────────────────────────────────────────────────┘
```

---

## Directory Structure

```
mini_os/
├── Makefile
├── boot/
│   ├── stage1.asm          ; 512-byte MBR bootloader
│   └── stage2.asm          ; Extended bootloader (load kernel ELF)
├── kernel/
│   ├── kernel.asm          ; Kernel entry point
│   ├── gdt.asm             ; Global Descriptor Table
│   ├── idt.asm             ; Interrupt Descriptor Table
│   ├── isr.asm             ; Interrupt Service Routines
│   ├── pic.asm             ; Programmable Interrupt Controller
│   ├── paging.asm          ; Page directory/tables
│   ├── pmm.asm             ; Physical Memory Manager
│   ├── heap.asm            ; Kernel heap allocator
│   ├── scheduler.asm       ; Round-robin scheduler
│   ├── syscall.asm         ; System call handler
│   ├── elf_loader.asm      ; ELF binary loader
│   └── shell.asm           ; Built-in kernel shell
├── drivers/
│   ├── vga.asm             ; VGA text mode driver
│   ├── keyboard.asm        ; Keyboard driver (IRQ1)
│   └── timer.asm           ; PIT timer driver (IRQ0)
├── lib/
│   └── string.asm          ; String utility functions
├── user/
│   ├── hello_user.asm      ; Simple user mode program
│   └── user_lib.asm        ; User mode library (syscall wrappers)
└── linker.ld               ; Kernel linker script
```

---

## Makefile

```makefile
# mini_os/Makefile

NASM    = nasm
LD      = ld
QEMU    = qemu-system-i386

NASMFLAGS_16 = -f bin
NASMFLAGS_32 = -f elf32
LDFLAGS      = -m elf_i386 -T linker.ld

KERNEL_OBJS = kernel/kernel.o   \
              kernel/gdt.o      \
              kernel/idt.o      \
              kernel/isr.o      \
              kernel/pic.o      \
              kernel/paging.o   \
              kernel/pmm.o      \
              kernel/heap.o     \
              kernel/scheduler.o \
              kernel/syscall.o  \
              kernel/elf_loader.o \
              kernel/shell.o    \
              drivers/vga.o     \
              drivers/keyboard.o \
              drivers/timer.o   \
              lib/string.o

.PHONY: all clean run debug

all: mini_os.img

# Stage 1 bootloader (512 bytes, raw binary)
boot/stage1.bin: boot/stage1.asm
	$(NASM) $(NASMFLAGS_16) -o $@ $<

# Stage 2 bootloader (raw binary)
boot/stage2.bin: boot/stage2.asm
	$(NASM) $(NASMFLAGS_16) -o $@ $<

# Kernel objects
%.o: %.asm
	$(NASM) $(NASMFLAGS_32) -o $@ $<

# Link kernel ELF
kernel/kernel.elf: $(KERNEL_OBJS)
	$(LD) $(LDFLAGS) -o $@ $(KERNEL_OBJS)

# User programs
user/hello_user.bin: user/hello_user.asm
	$(NASM) -f elf32 -o user/hello_user.o $<
	$(LD) -m elf_i386 -Ttext 0x400000 --oformat elf32-i386 \
	      -o $@ user/hello_user.o user/user_lib.o

# Create disk image
mini_os.img: boot/stage1.bin boot/stage2.bin kernel/kernel.elf user/hello_user.bin
	# Create 1.44MB floppy image
	dd if=/dev/zero of=$@ bs=512 count=2880
	# Write stage1 to sector 0
	dd if=boot/stage1.bin of=$@ bs=512 seek=0 conv=notrunc
	# Write stage2 starting at sector 1
	dd if=boot/stage2.bin of=$@ bs=512 seek=1 conv=notrunc
	# Write kernel ELF at sector 10
	dd if=kernel/kernel.elf of=$@ bs=512 seek=10 conv=notrunc
	# Write hello_user at sector 200
	dd if=user/hello_user.bin of=$@ bs=512 seek=200 conv=notrunc

# Run in QEMU
run: mini_os.img
	$(QEMU) -fda mini_os.img -boot a

# Debug with GDB
debug: mini_os.img
	$(QEMU) -fda mini_os.img -boot a -s -S &
	gdb -ex "target remote :1234" \
	    -ex "symbol-file kernel/kernel.elf" \
	    -ex "break kernel_main" \
	    -ex "continue"

clean:
	rm -f boot/*.bin kernel/*.o kernel/*.elf \
	      drivers/*.o lib/*.o user/*.o user/*.bin \
	      mini_os.img
```

---

## Linker Script

```ld
/* linker.ld */
ENTRY(kernel_entry)

SECTIONS {
    . = 0x100000;       /* Kernel loads at 1MB */

    .text : {
        *(.text)
    }

    .rodata : {
        *(.rodata*)
    }

    .data : {
        *(.data)
    }

    .bss : {
        *(COMMON)
        *(.bss)
    }

    kernel_end = .;
}
```

---

## Stage 1 Bootloader

Stage 1 อยู่ใน MBR (512 bytes) มีหน้าที่โหลด Stage 2 จาก disk

```nasm
; boot/stage1.asm
; Stage 1 Bootloader - fits in 512 bytes (MBR)
; Loads Stage 2 from sectors 1..9 to 0x7E00

[BITS 16]
[ORG 0x7C00]

STAGE2_SECTORS  equ 9           ; จำนวน sector ของ Stage 2
STAGE2_LOAD_SEG equ 0x07E0      ; โหลด Stage 2 ที่ 0x07E0:0x0000 = 0x7E00
STAGE2_LOAD_OFF equ 0x0000

start:
    ; ตั้งค่า segment registers
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00              ; Stack ต่ำกว่า bootloader

    ; เก็บ boot drive number
    mov [boot_drive], dl

    ; แสดงข้อความ
    mov si, msg_loading
    call print_string

    ; โหลด Stage 2
    mov ax, STAGE2_LOAD_SEG
    mov es, ax
    mov bx, STAGE2_LOAD_OFF     ; ES:BX = ที่อยู่โหลด

    mov ah, 0x02                ; BIOS Read Sectors
    mov al, STAGE2_SECTORS      ; จำนวน sectors
    mov ch, 0                   ; Cylinder 0
    mov cl, 2                   ; Sector 2 (1-based)
    mov dh, 0                   ; Head 0
    mov dl, [boot_drive]        ; Drive number
    int 0x13
    jc disk_error               ; ถ้า carry set = error

    ; ตรวจสอบว่าอ่านครบ
    cmp al, STAGE2_SECTORS
    jne disk_error

    ; กระโดดไป Stage 2
    jmp STAGE2_LOAD_SEG:STAGE2_LOAD_OFF

disk_error:
    mov si, msg_error
    call print_string
    hlt

print_string:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    xor bx, bx
    int 0x10
    jmp print_string
.done:
    ret

msg_loading db 'Loading Stage 2...', 13, 10, 0
msg_error   db 'Disk Error!', 13, 10, 0
boot_drive  db 0

; Padding และ Boot Signature
times 510 - ($ - $$) db 0
dw 0xAA55
```

---

## Stage 2 Bootloader

Stage 2 มีหน้าที่: ตรวจสอบ memory, เข้า Protected Mode, โหลด Kernel ELF

```nasm
; boot/stage2.asm
; Stage 2 Bootloader
; - Detect memory (E820)
; - Enable A20 line
; - Load GDT
; - Enter Protected Mode
; - Load Kernel ELF from disk

[BITS 16]
[ORG 0x7E00]

KERNEL_SECTOR   equ 10          ; Kernel ELF เริ่มที่ sector 10
KERNEL_SECTORS  equ 100         ; อ่านสูงสุด 100 sectors
KERNEL_LOAD_ADDR equ 0x10000    ; โหลด kernel ที่ 0x10000 ชั่วคราว

start16:
    mov ax, cs
    mov ds, ax
    mov es, ax

    mov si, msg_stage2
    call print16

    ; --- 1. Detect Memory via E820 ---
    call detect_memory

    ; --- 2. Enable A20 Line ---
    call enable_a20

    ; --- 3. Load Kernel ELF from disk ---
    call load_kernel

    ; --- 4. เข้า Protected Mode ---
    cli
    lgdt [gdt_descriptor]

    mov eax, cr0
    or eax, 1
    mov cr0, eax

    jmp 0x08:protected_mode_entry   ; far jump to 32-bit code

;-------------------------------------
; Detect Memory (INT 15h, E820)
;-------------------------------------
detect_memory:
    mov di, 0x500           ; เก็บ memory map ที่ 0x500
    xor ebx, ebx
    xor bp, bp              ; นับจำนวน entries

.loop:
    mov eax, 0xE820
    mov edx, 0x534D4150     ; 'SMAP'
    mov ecx, 24
    int 0x15
    jc .done
    cmp eax, 0x534D4150
    jne .done

    inc bp
    add di, 24

    test ebx, ebx
    jz .done
    jmp .loop

.done:
    mov [mem_map_count], bp
    ret

;-------------------------------------
; Enable A20 via Fast A20
;-------------------------------------
enable_a20:
    in al, 0x92
    or al, 0x02
    and al, 0xFE
    out 0x92, al
    ret

;-------------------------------------
; Load Kernel ELF from floppy
;-------------------------------------
load_kernel:
    mov ax, KERNEL_LOAD_ADDR >> 4
    mov es, ax
    xor bx, bx

    mov ah, 0x02
    mov al, KERNEL_SECTORS
    mov ch, 0
    mov cl, KERNEL_SECTOR + 1  ; 1-based sector
    mov dh, 0
    mov dl, 0                   ; drive 0 = floppy
    int 0x13
    jc .error
    ret

.error:
    mov si, msg_load_err
    call print16
    hlt

;-------------------------------------
; Print String (16-bit)
;-------------------------------------
print16:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    xor bx, bx
    int 0x10
    jmp print16
.done:
    ret

msg_stage2   db 'Stage 2 OK', 13, 10, 0
msg_load_err db 'Kernel load error!', 13, 10, 0
mem_map_count dw 0

;-------------------------------------
; GDT (Global Descriptor Table)
;-------------------------------------
align 8
gdt_start:
    ; Null descriptor
    dq 0

    ; Kernel Code Segment: base=0, limit=4GB, Ring 0, Execute/Read
    dw 0xFFFF       ; limit[15:0]
    dw 0x0000       ; base[15:0]
    db 0x00         ; base[23:16]
    db 10011010b    ; P=1, DPL=00, S=1, Type=1010 (Execute/Read)
    db 11001111b    ; G=1, D/B=1, L=0, AVL=0, limit[19:16]=1111
    db 0x00         ; base[31:24]

    ; Kernel Data Segment: base=0, limit=4GB, Ring 0, Read/Write
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 10010010b    ; P=1, DPL=00, S=1, Type=0010 (Read/Write)
    db 11001111b
    db 0x00

    ; User Code Segment: Ring 3
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11111010b    ; P=1, DPL=11, S=1, Type=1010
    db 11001111b
    db 0x00

    ; User Data Segment: Ring 3
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11110010b    ; P=1, DPL=11, S=1, Type=0010
    db 11001111b
    db 0x00

    ; TSS Descriptor (placeholder, filled at runtime)
    dq 0
    dq 0            ; TSS ต้องการ 2 entries ใน 64-bit mode แต่เราใช้ 32-bit

gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1
    dd gdt_start

;-------------------------------------
; Protected Mode Entry (32-bit)
;-------------------------------------
[BITS 32]
protected_mode_entry:
    ; ตั้งค่า segment registers
    mov ax, 0x10            ; Kernel Data Segment selector (index 2, RPL=0)
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    mov esp, 0x90000        ; Stack

    ; ตอนนี้ kernel อยู่ที่ 0x10000
    ; ต้องทำ ELF loading ต่อ แต่เราจะให้ kernel entry จัดการ
    ; กระโดดไป ELF entry point (เราโหลด kernel.elf ไว้ที่ 0x10000)
    ; ELF header อยู่ที่ 0x10000, entry point ที่ offset 0x18

    mov eax, [0x10018]      ; e_entry จาก ELF header (offset 24)
    jmp eax                 ; กระโดดไป kernel entry point

times 9*512 - ($ - $$) db 0  ; Pad to 9 sectors
```

---

## Kernel Entry Point

```nasm
; kernel/kernel.asm
; Kernel entry point - ถูกเรียกจาก Stage 2

[BITS 32]

global kernel_entry
extern gdt_init
extern idt_init
extern pic_init
extern paging_init
extern pmm_init
extern heap_init
extern vga_init
extern keyboard_init
extern timer_init
extern scheduler_init
extern syscall_init
extern shell_start
extern vga_printf

section .text

kernel_entry:
    ; Stack ได้ถูกตั้งไว้แล้วจาก Stage 2
    ; แต่เราตั้งใหม่ให้ชัดเจน
    mov esp, kernel_stack_top

    ; เคลียร์ BSS
    mov edi, bss_start
    mov ecx, bss_end
    sub ecx, bss_start
    xor eax, eax
    rep stosb

    ; Initialize subsystems
    call gdt_init           ; ตั้งค่า GDT พร้อม TSS
    call idt_init           ; ตั้งค่า IDT
    call pic_init           ; Remap PIC
    call paging_init        ; เปิด Paging
    call pmm_init           ; Physical Memory Manager
    call heap_init          ; Kernel Heap
    call vga_init           ; VGA Console
    call keyboard_init      ; Keyboard Driver
    call timer_init         ; Timer/PIT
    call scheduler_init     ; Process Scheduler
    call syscall_init       ; System Call Interface

    ; แสดงข้อความต้อนรับ
    push msg_welcome
    call vga_printf
    add esp, 4

    ; เริ่ม shell
    call shell_start

    ; ไม่ควรมาถึงที่นี่
    cli
    hlt

section .data
msg_welcome db 'Mini OS v1.0 - Ready', 10, 0

section .bss
    alignb 16
kernel_stack:
    resb 0x4000             ; 16KB kernel stack
kernel_stack_top:

global bss_start, bss_end
bss_start:
    resb 0
bss_end:
```

---

## GDT พร้อม TSS

```nasm
; kernel/gdt.asm
; GDT initialization with TSS

[BITS 32]

global gdt_init
global tss_entry

%define GDT_NULL    0x00    ; Null descriptor
%define GDT_KCODE   0x08    ; Kernel code
%define GDT_KDATA   0x10    ; Kernel data
%define GDT_UCODE   0x18    ; User code (Ring 3)
%define GDT_UDATA   0x20    ; User data (Ring 3)
%define GDT_TSS     0x28    ; TSS descriptor

section .data

align 8
gdt:
.null:
    dq 0

.kernel_code:               ; selector 0x08
    dw 0xFFFF               ; limit[15:0]
    dw 0x0000               ; base[15:0]
    db 0x00                 ; base[23:16]
    db 10011010b            ; flags: Present, Ring0, Code, Execute/Read
    db 11001111b            ; flags: 4K gran, 32-bit, limit[19:16]
    db 0x00                 ; base[31:24]

.kernel_data:               ; selector 0x10
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 10010010b            ; Present, Ring0, Data, Read/Write
    db 11001111b
    db 0x00

.user_code:                 ; selector 0x18 (+ RPL=3 = 0x1B)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11111010b            ; Present, Ring3, Code, Execute/Read
    db 11001111b
    db 0x00

.user_data:                 ; selector 0x20 (+ RPL=3 = 0x23)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 11110010b            ; Present, Ring3, Data, Read/Write
    db 11001111b
    db 0x00

.tss_desc:                  ; selector 0x28
    dw tss_size - 1         ; limit[15:0]
    dw 0x0000               ; base[15:0] (filled at runtime)
    db 0x00                 ; base[23:16]
    db 10001001b            ; Present, Ring0, 32-bit TSS Available
    db 00000000b            ; G=0 (byte granularity)
    db 0x00                 ; base[31:24]

gdt_end:

gdt_ptr:
    dw gdt_end - gdt - 1
    dd gdt

; TSS (Task State Segment)
align 4
tss_entry:
    dd 0                    ; prev_tss
    dd 0                    ; esp0 (kernel stack pointer) - filled at runtime
    dd GDT_KDATA            ; ss0 = kernel data segment
    times 23 dd 0           ; esp1,ss1,esp2,ss2,cr3,eip,eflags,eax..edi,es..gs
    dw 0                    ; reserved
    dw tss_size             ; iomap base (past end = no iomap)
tss_size equ $ - tss_entry

section .text

gdt_init:
    ; คำนวณ base address ของ TSS แล้วใส่ใน GDT descriptor
    mov eax, tss_entry
    mov [gdt.tss_desc + 2], ax      ; base[15:0]
    shr eax, 16
    mov [gdt.tss_desc + 4], al      ; base[23:16]
    mov [gdt.tss_desc + 7], ah      ; base[31:24]

    ; ตั้งค่า ESP0 ใน TSS (kernel stack top)
    mov dword [tss_entry + 4], 0x90000  ; esp0 = kernel stack

    ; โหลด GDT
    lgdt [gdt_ptr]

    ; Reload segment registers
    jmp GDT_KCODE:.reload
.reload:
    mov ax, GDT_KDATA
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax

    ; โหลด TSS
    mov ax, GDT_TSS
    ltr ax

    ret
```

---

## IDT และ ISR

```nasm
; kernel/idt.asm
; Interrupt Descriptor Table setup

[BITS 32]

global idt_init
global idt_set_gate

%define IDT_ENTRIES     256
%define IDT_INTERRUPT   0x8E    ; Present, Ring0, 32-bit interrupt gate
%define IDT_TRAP        0x8F    ; Present, Ring0, 32-bit trap gate
%define IDT_USER_INT    0xEE    ; Present, Ring3, 32-bit interrupt gate (syscall)

section .data

align 8
idt:
    times IDT_ENTRIES * 8 db 0   ; 256 entries x 8 bytes each

idt_ptr:
    dw IDT_ENTRIES * 8 - 1
    dd idt

section .text

; idt_set_gate(num, base, sel, flags)
; num: interrupt number (byte)
; base: handler address (dword)
; sel: code segment selector (word)
; flags: gate type/flags (byte)
idt_set_gate:
    push ebp
    mov ebp, esp
    push eax ebx ecx

    movzx eax, byte [ebp+8]     ; num
    mov ebx, [ebp+12]           ; base
    movzx ecx, word [ebp+16]    ; sel
    mov dl, [ebp+20]            ; flags

    ; คำนวณ offset ใน IDT
    imul eax, eax, 8
    add eax, idt

    ; เขียน IDT entry:
    ; [0..1]   base[15:0]
    ; [2..3]   selector
    ; [4]      reserved (0)
    ; [5]      flags
    ; [6..7]   base[31:16]
    mov [eax],   bx             ; base[15:0]
    mov [eax+2], cx             ; selector
    mov byte [eax+4], 0         ; reserved
    mov [eax+5], dl             ; flags
    shr ebx, 16
    mov [eax+6], bx             ; base[31:16]

    pop ecx ebx eax
    pop ebp
    ret

idt_init:
    ; ลงทะเบียน exception handlers (0-31)
    push dword IDT_INTERRUPT
    push dword GDT_KCODE
    push dword isr0
    push dword 0
    call idt_set_gate
    add esp, 16

    ; (ทำซ้ำสำหรับ ISR 1-31 แต่ย่อไว้)
    ; IRQ handlers (32-47) จะถูก set โดย pic_init

    ; System call (INT 0x80)
    push dword IDT_USER_INT     ; Ring 3 สามารถเรียกได้
    push dword GDT_KCODE
    push dword syscall_handler
    push dword 0x80
    call idt_set_gate
    add esp, 16

    ; โหลด IDT
    lidt [idt_ptr]

    ret

%define GDT_KCODE 0x08
```

---

## ISR (Interrupt Service Routines)

```nasm
; kernel/isr.asm
; Exception and IRQ handlers

[BITS 32]

global isr0, isr_common
global irq0, irq1, irq_common
extern handle_exception
extern handle_irq
extern vga_printf

; Macro สำหรับ exception ที่ไม่มี error code
%macro ISR_NOERR 1
isr%1:
    cli
    push dword 0            ; dummy error code
    push dword %1           ; interrupt number
    jmp isr_common
%endmacro

; Macro สำหรับ exception ที่มี error code
%macro ISR_ERR 1
isr%1:
    cli
    push dword %1           ; interrupt number (error code อยู่ใน stack แล้ว)
    jmp isr_common
%endmacro

; Macro สำหรับ IRQ
%macro IRQ 2
irq%1:
    cli
    push dword 0
    push dword %2           ; interrupt number = IRQ + 32
    jmp irq_common
%endmacro

; Exception handlers
ISR_NOERR 0     ; Divide by Zero
ISR_NOERR 1     ; Debug
ISR_NOERR 2     ; NMI
ISR_NOERR 3     ; Breakpoint
ISR_NOERR 4     ; Overflow
ISR_NOERR 5     ; Bound Range Exceeded
ISR_NOERR 6     ; Invalid Opcode
ISR_NOERR 7     ; Device Not Available
ISR_ERR   8     ; Double Fault
ISR_NOERR 9     ; Coprocessor Segment Overrun
ISR_ERR   10    ; Invalid TSS
ISR_ERR   11    ; Segment Not Present
ISR_ERR   12    ; Stack Fault
ISR_ERR   13    ; General Protection Fault
ISR_ERR   14    ; Page Fault
ISR_NOERR 15    ; Reserved
ISR_NOERR 16    ; FPU Exception

; IRQ handlers
IRQ 0, 32       ; Timer
IRQ 1, 33       ; Keyboard
IRQ 2, 34
IRQ 3, 35
IRQ 4, 36
IRQ 5, 37
IRQ 6, 38
IRQ 7, 39

section .text

; โครงสร้าง registers_t บน stack:
; gs, fs, es, ds
; edi, esi, ebp, esp, ebx, edx, ecx, eax  (จาก pusha)
; int_no, err_code
; eip, cs, eflags, useresp, ss  (จาก CPU)

isr_common:
    pusha                   ; บันทึก registers ทั้งหมด
    push ds
    push es
    push fs
    push gs

    mov ax, 0x10            ; Kernel data segment
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    push esp                ; ส่ง pointer ไปยัง register frame
    call handle_exception
    add esp, 4

    pop gs
    pop fs
    pop es
    pop ds
    popa
    add esp, 8              ; ลบ err_code และ int_no
    iret

irq_common:
    pusha
    push ds
    push es
    push fs
    push gs

    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    push esp
    call handle_irq
    add esp, 4

    pop gs
    pop fs
    pop es
    pop ds
    popa
    add esp, 8
    iret

; Exception handler (C-like, called from isr_common)
handle_exception:
    push ebp
    mov ebp, esp
    push eax ebx esi edi

    mov ebx, [ebp+8]        ; pointer to register frame
    mov eax, [ebx+36]       ; int_no (offset เพราะ pusha + segs)

    ; แสดง exception
    push eax
    push msg_exception
    call vga_printf
    add esp, 8

    ; หยุด execution
    cli
    hlt

    pop edi esi ebx eax
    pop ebp
    ret

section .data
msg_exception db 'KERNEL PANIC: Exception #%d', 10, 0
```

---

## PIC (Programmable Interrupt Controller)

```nasm
; kernel/pic.asm
; 8259 PIC setup - remap IRQs to 0x20-0x2F

[BITS 32]

global pic_init
global pic_send_eoi

%define PIC1_CMD    0x20    ; Master PIC command port
%define PIC1_DATA   0x21    ; Master PIC data port
%define PIC2_CMD    0xA0    ; Slave PIC command port
%define PIC2_DATA   0xA1    ; Slave PIC data port

%define ICW1_INIT   0x11    ; Initialize + ICW4 needed
%define ICW4_8086   0x01    ; 8086 mode

%define PIC1_OFFSET 0x20    ; Remap Master IRQ0-7 to 0x20-0x27
%define PIC2_OFFSET 0x28    ; Remap Slave  IRQ8-15 to 0x28-0x2F

section .text

pic_init:
    ; บันทึก interrupt masks เดิม
    in al, PIC1_DATA
    mov [pic1_mask], al
    in al, PIC2_DATA
    mov [pic2_mask], al

    ; เริ่ม Initialization sequence (ICW1)
    mov al, ICW1_INIT
    out PIC1_CMD, al
    out PIC2_CMD, al

    ; ICW2: ตั้ง offset
    mov al, PIC1_OFFSET     ; Master: IRQ0-7 → INT 0x20-0x27
    out PIC1_DATA, al
    mov al, PIC2_OFFSET     ; Slave: IRQ8-15 → INT 0x28-0x2F
    out PIC2_DATA, al

    ; ICW3: cascade
    mov al, 0x04            ; Master: slave อยู่ที่ IRQ2 (bit 2)
    out PIC1_DATA, al
    mov al, 0x02            ; Slave: cascade identity = 2
    out PIC2_DATA, al

    ; ICW4: 8086 mode
    mov al, ICW4_8086
    out PIC1_DATA, al
    out PIC2_DATA, al

    ; เปิด mask สำหรับ IRQ0 (timer) และ IRQ1 (keyboard)
    ; mask = 0 หมายถึงเปิด, 1 หมายถึงปิด
    mov al, 11111100b       ; เปิด IRQ0, IRQ1 เท่านั้น
    out PIC1_DATA, al
    mov al, 11111111b       ; ปิด Slave ทั้งหมด
    out PIC2_DATA, al

    ret

; ส่ง End of Interrupt
; pic_send_eoi(irq_num)
pic_send_eoi:
    mov al, [esp+4]         ; irq number
    cmp al, 8
    jl .master_only

    ; ส่ง EOI ให้ Slave ก่อน
    mov al, 0x20
    out PIC2_CMD, al

.master_only:
    mov al, 0x20
    out PIC1_CMD, al
    ret

section .data
pic1_mask db 0
pic2_mask db 0
```

---

## Paging

```nasm
; kernel/paging.asm
; Setup paging: identity map + kernel at high half

[BITS 32]

global paging_init
global page_directory

%define PAGE_PRESENT    0x01
%define PAGE_WRITE      0x02
%define PAGE_USER       0x04
%define PAGE_SIZE_4MB   0x80    ; สำหรับ 4MB pages (PSE)

section .data

; Page Directory (4096 bytes aligned)
align 4096
page_directory:
    times 1024 dd 0

; Page Table สำหรับ identity map แรก 4MB
align 4096
page_table_low:
    times 1024 dd 0

section .text

paging_init:
    ; Identity map แรก 4MB (0x00000000 - 0x003FFFFF)
    ; เติม page_table_low
    mov ecx, 0              ; page index
    mov eax, 0              ; physical address

.fill_table:
    or eax, PAGE_PRESENT | PAGE_WRITE
    mov [page_table_low + ecx*4], eax
    add eax, 0x1000         ; next page (4KB)
    inc ecx
    cmp ecx, 1024
    jl .fill_table

    ; ตั้งค่า Page Directory entry 0
    mov eax, page_table_low
    or eax, PAGE_PRESENT | PAGE_WRITE
    mov [page_directory], eax

    ; โหลด CR3 ด้วย page directory
    mov eax, page_directory
    mov cr3, eax

    ; เปิด PSE (Page Size Extension) ใน CR4
    mov eax, cr4
    or eax, 0x10
    mov cr4, eax

    ; เปิด Paging ใน CR0
    mov eax, cr0
    or eax, 0x80000000
    mov cr0, eax

    ret
```

---

## Physical Memory Manager (Bitmap)

```nasm
; kernel/pmm.asm
; Physical Memory Manager using bitmap
; แต่ละ bit แทน 1 page (4KB)
; 0 = free, 1 = used

[BITS 32]

global pmm_init
global pmm_alloc
global pmm_free

%define PAGE_SIZE   4096
%define BITMAP_ADDR 0x30000     ; เก็บ bitmap ที่ 0x30000
%define MEM_SIZE    (64 * 1024 * 1024)  ; สมมติ RAM 64MB
%define TOTAL_PAGES (MEM_SIZE / PAGE_SIZE)  ; = 16384 pages
%define BITMAP_SIZE (TOTAL_PAGES / 8)       ; = 2048 bytes

section .text

pmm_init:
    ; เคลียร์ bitmap (ทุกหน้าว่าง)
    mov edi, BITMAP_ADDR
    mov ecx, BITMAP_SIZE
    xor al, al
    rep stosb

    ; Mark pages ที่ใช้แล้วว่า "used"
    ; 0x00000 - 0x00FFF: Real mode IVT, BIOS data
    ; 0x07C00 - 0x07FFF: Bootloader
    ; 0x10000 - 0x2FFFF: Kernel + Stage2
    ; 0x30000 - 0x307FF: PMM bitmap เอง

    ; Mark page 0..47 (0x00000-0x2FFFF) = 48 pages
    mov ecx, 48
    xor eax, eax
.mark_used:
    call pmm_mark_used_internal
    inc eax
    loop .mark_used

    ; Mark bitmap pages (page 0x30 = 48..51)
    mov eax, 48
    call pmm_mark_used_internal

    ret

; pmm_mark_used_internal(page_num in eax)
pmm_mark_used_internal:
    push eax ecx edx
    mov ecx, eax
    shr ecx, 3              ; byte index = page_num / 8
    and eax, 7              ; bit index = page_num % 8
    mov edx, 1
    shl edx, al             ; mask
    or [BITMAP_ADDR + ecx], dl
    pop edx ecx eax
    ret

; pmm_alloc() → eax = physical address, 0 on failure
pmm_alloc:
    push ebx ecx edx edi

    mov ecx, BITMAP_SIZE
    mov edi, BITMAP_ADDR

.search_byte:
    mov al, [edi]
    cmp al, 0xFF            ; byte เต็มหมด?
    je .next_byte

    ; หา bit ที่ว่าง
    mov ebx, 0
.find_bit:
    mov edx, 1
    shl edx, bl
    test al, dl
    jz .found_bit
    inc ebx
    cmp ebx, 8
    jl .find_bit

.next_byte:
    inc edi
    loop .search_byte

    ; ไม่มีหน้าว่าง
    xor eax, eax
    jmp .done

.found_bit:
    ; คำนวณ page number
    sub edi, BITMAP_ADDR
    imul eax, edi, 8
    add eax, ebx            ; page_num

    ; Mark as used
    mov edx, 1
    shl edx, bl
    or [BITMAP_ADDR + edi], dl

    ; คืน physical address
    imul eax, eax, PAGE_SIZE

.done:
    pop edi edx ecx ebx
    ret

; pmm_free(phys_addr)
pmm_free:
    push eax ebx ecx

    mov eax, [esp+16]       ; phys_addr
    shr eax, 12             ; page_num = addr / 4096

    mov ecx, eax
    shr ecx, 3              ; byte index
    and eax, 7              ; bit index
    mov ebx, 1
    shl ebx, al
    not bl                  ; invert mask
    and [BITMAP_ADDR + ecx], bl  ; clear bit

    pop ecx ebx eax
    ret
```

---

## Kernel Heap (Slab Allocator)

```nasm
; kernel/heap.asm
; Simple slab/pool allocator for kernel heap

[BITS 32]

global heap_init
global kmalloc
global kfree

extern pmm_alloc

%define HEAP_START  0x400000    ; Kernel heap starts at 4MB
%define HEAP_MAX    0x800000    ; Heap can grow to 8MB

; Block header structure:
; [size: 4 bytes][free: 4 bytes][next: 4 bytes]
%define HDR_SIZE    12

section .data
heap_start  dd HEAP_START
heap_end    dd HEAP_START
heap_inited db 0

section .text

heap_init:
    ; จอง page แรกสำหรับ heap
    call pmm_alloc
    test eax, eax
    jz .error

    ; Map physical address to virtual (สมมติ identity map)
    ; ในระบบจริงต้องทำ page mapping ที่นี่

    ; สร้าง free block แรก
    mov dword [HEAP_START],   0         ; size = 0 (จะถูกกำหนดเมื่อ alloc)
    mov dword [HEAP_START+4], 1         ; free = 1
    mov dword [HEAP_START+8], 0         ; next = NULL

    mov dword [heap_end], HEAP_START + HDR_SIZE
    mov byte [heap_inited], 1

.error:
    ret

; kmalloc(size) → eax = pointer, 0 on failure
kmalloc:
    push ebp
    mov ebp, esp
    push ebx ecx edx

    mov ecx, [ebp+8]        ; requested size
    ; Align to 8 bytes
    add ecx, 7
    and ecx, ~7

    ; ค้นหา free block ที่ใหญ่พอ
    mov ebx, HEAP_START

.search:
    cmp ebx, [heap_end]
    jge .extend_heap

    mov eax, [ebx+4]        ; free flag
    test eax, eax
    jz .next_block

    mov eax, [ebx]          ; block size
    cmp eax, ecx
    jge .found_block

.next_block:
    mov eax, [ebx+8]        ; next pointer
    test eax, eax
    jz .extend_heap
    mov ebx, eax
    jmp .search

.found_block:
    ; Mark as used
    mov dword [ebx+4], 0
    ; คืน pointer หลัง header
    lea eax, [ebx + HDR_SIZE]
    jmp .done

.extend_heap:
    ; ต้องการ page ใหม่
    call pmm_alloc
    test eax, eax
    jz .fail

    ; เพิ่ม block ใหม่
    mov edx, [heap_end]
    mov [edx],     ecx          ; size
    mov dword [edx+4], 0        ; not free (จะใช้เลย)
    mov dword [edx+8], 0        ; next = NULL

    ; อัปเดต heap_end
    add ecx, HDR_SIZE
    add [heap_end], ecx

    lea eax, [edx + HDR_SIZE]
    jmp .done

.fail:
    xor eax, eax

.done:
    pop edx ecx ebx
    pop ebp
    ret

; kfree(ptr)
kfree:
    push eax
    mov eax, [esp+8]        ; ptr
    sub eax, HDR_SIZE       ; ถอย pointer ไป header
    mov dword [eax+4], 1    ; mark free
    pop eax
    ret
```

---

## VGA Console Driver

```nasm
; drivers/vga.asm
; VGA Text Mode Driver - 80x25 characters

[BITS 32]

global vga_init
global vga_putchar
global vga_puts
global vga_printf
global vga_clear
global vga_set_color

%define VGA_BASE    0xB8000     ; VGA text buffer
%define VGA_WIDTH   80
%define VGA_HEIGHT  25
%define VGA_SIZE    (VGA_WIDTH * VGA_HEIGHT * 2)

; สี VGA
%define VGA_BLACK       0
%define VGA_BLUE        1
%define VGA_GREEN       2
%define VGA_CYAN        3
%define VGA_RED         4
%define VGA_MAGENTA     5
%define VGA_BROWN       6
%define VGA_LIGHT_GRAY  7
%define VGA_WHITE       15

section .data
vga_x       dw 0
vga_y       dw 0
vga_color   db (VGA_WHITE | (VGA_BLACK << 4))  ; ขาวบนดำ

section .text

vga_init:
    call vga_clear
    ret

vga_clear:
    push edi eax ecx
    mov edi, VGA_BASE
    movzx eax, byte [vga_color]
    shl eax, 8
    or eax, ' '             ; space character
    mov ah, [vga_color]     ; color byte
    ; เติม 80*25 words
    mov ecx, VGA_WIDTH * VGA_HEIGHT
.loop:
    mov word [edi], ax
    add edi, 2
    loop .loop

    mov word [vga_x], 0
    mov word [vga_y], 0
    pop ecx eax edi
    ret

; vga_set_color(fg, bg)
vga_set_color:
    push eax ebx
    movzx eax, byte [esp+12]    ; fg
    movzx ebx, byte [esp+16]    ; bg
    shl ebx, 4
    or eax, ebx
    mov [vga_color], al
    pop ebx eax
    ret

; vga_putchar(char)
vga_putchar:
    push eax ebx ecx edx

    mov al, [esp+20]        ; character

    cmp al, 10              ; newline?
    je .newline
    cmp al, 13              ; carriage return?
    je .cr
    cmp al, 8               ; backspace?
    je .backspace

    ; คำนวณตำแหน่งใน VGA buffer
    movzx ecx, word [vga_y]
    imul ecx, VGA_WIDTH
    movzx edx, word [vga_x]
    add ecx, edx
    imul ecx, 2             ; * 2 bytes per char

    mov ah, [vga_color]
    mov word [VGA_BASE + ecx], ax

    ; ขยับ cursor
    inc word [vga_x]
    cmp word [vga_x], VGA_WIDTH
    jl .done

.newline:
    mov word [vga_x], 0
    inc word [vga_y]
    cmp word [vga_y], VGA_HEIGHT
    jl .done
    call vga_scroll
    dec word [vga_y]
    jmp .done

.cr:
    mov word [vga_x], 0
    jmp .done

.backspace:
    cmp word [vga_x], 0
    je .done
    dec word [vga_x]
    ; ลบตัวอักษร
    movzx ecx, word [vga_y]
    imul ecx, VGA_WIDTH
    movzx edx, word [vga_x]
    add ecx, edx
    imul ecx, 2
    mov ah, [vga_color]
    mov al, ' '
    mov word [VGA_BASE + ecx], ax

.done:
    call vga_update_cursor
    pop edx ecx ebx eax
    ret

; เลื่อน VGA buffer ขึ้น 1 บรรทัด
vga_scroll:
    push esi edi ecx
    mov esi, VGA_BASE + VGA_WIDTH * 2
    mov edi, VGA_BASE
    mov ecx, VGA_WIDTH * (VGA_HEIGHT - 1)
    rep movsw               ; copy words

    ; เคลียร์บรรทัดล่างสุด
    mov ecx, VGA_WIDTH
    mov ah, [vga_color]
    mov al, ' '
.clear_last:
    mov word [edi], ax
    add edi, 2
    loop .clear_last

    pop ecx edi esi
    ret

; อัปเดต hardware cursor
vga_update_cursor:
    push eax ecx edx

    movzx ecx, word [vga_y]
    imul ecx, VGA_WIDTH
    movzx edx, word [vga_x]
    add ecx, edx            ; cursor position (linear)

    ; ส่ง position ไป VGA cursor registers
    mov dx, 0x3D4
    mov al, 0x0F            ; Cursor Location Low
    out dx, al
    inc dx
    mov al, cl
    out dx, al

    dec dx
    mov al, 0x0E            ; Cursor Location High
    out dx, al
    inc dx
    mov al, ch
    out dx, al

    pop edx ecx eax
    ret

; vga_puts(str)
vga_puts:
    push eax esi
    mov esi, [esp+12]       ; str pointer
.loop:
    lodsb
    or al, al
    jz .done
    push eax
    call vga_putchar
    add esp, 4
    jmp .loop
.done:
    pop esi eax
    ret

; vga_printf(fmt, ...) - very simple, supports %d, %s, %x, %c
vga_printf:
    push ebp
    mov ebp, esp
    push eax ebx ecx edx esi edi

    mov esi, [ebp+8]        ; format string
    lea edi, [ebp+12]       ; first argument

.loop:
    lodsb
    or al, al
    jz .done

    cmp al, '%'
    je .format

    push eax
    call vga_putchar
    add esp, 4
    jmp .loop

.format:
    lodsb                   ; get format specifier
    cmp al, 'd'
    je .fmt_int
    cmp al, 's'
    je .fmt_str
    cmp al, 'x'
    je .fmt_hex
    cmp al, 'c'
    je .fmt_char
    cmp al, '%'
    je .fmt_percent

    jmp .loop

.fmt_char:
    mov eax, [edi]
    add edi, 4
    push eax
    call vga_putchar
    add esp, 4
    jmp .loop

.fmt_str:
    mov eax, [edi]
    add edi, 4
    push eax
    call vga_puts
    add esp, 4
    jmp .loop

.fmt_int:
    mov eax, [edi]
    add edi, 4
    call print_int
    jmp .loop

.fmt_hex:
    mov eax, [edi]
    add edi, 4
    call print_hex
    jmp .loop

.fmt_percent:
    push dword '%'
    call vga_putchar
    add esp, 4
    jmp .loop

.done:
    pop edi esi edx ecx ebx eax
    pop ebp
    ret

; print integer (eax)
print_int:
    push eax ebx ecx edx

    test eax, eax
    jns .positive
    push dword '-'
    call vga_putchar
    add esp, 4
    neg eax

.positive:
    mov ecx, 0              ; digit count
    mov ebx, 10
.divloop:
    xor edx, edx
    div ebx
    push edx                ; remainder (digit)
    inc ecx
    test eax, eax
    jnz .divloop

.printloop:
    pop eax
    add eax, '0'
    push ecx
    push eax
    call vga_putchar
    add esp, 4
    pop ecx
    loop .printloop

    pop edx ecx ebx eax
    ret

; print hex (eax)
print_hex:
    push eax ebx ecx

    push dword 'x'
    call vga_putchar
    add esp, 4
    push dword '0'
    call vga_putchar
    add esp, 4

    mov ecx, 8              ; 8 hex digits
.hexloop:
    rol eax, 4
    mov ebx, eax
    and ebx, 0xF
    cmp ebx, 10
    jl .digit
    add ebx, 'A' - 10
    jmp .print_hex_digit
.digit:
    add ebx, '0'
.print_hex_digit:
    push ecx eax
    push ebx
    call vga_putchar
    add esp, 4
    pop eax ecx
    loop .hexloop

    pop ecx ebx eax
    ret
```

---

## Keyboard Driver

```nasm
; drivers/keyboard.asm
; PS/2 Keyboard Driver - IRQ1 handler

[BITS 32]

global keyboard_init
global keyboard_handler
global keyboard_getchar

%define KB_DATA_PORT    0x60
%define KB_STATUS_PORT  0x64
%define KB_BUF_SIZE     256

section .data

; Scancode to ASCII table (US layout, set 1)
scancode_table:
    db 0,   27,  '1', '2', '3', '4', '5', '6'  ; 0x00-0x07
    db '7', '8', '9', '0', '-', '=',   8,   9  ; 0x08-0x0F
    db 'q', 'w', 'e', 'r', 't', 'y', 'u', 'i'  ; 0x10-0x17
    db 'o', 'p', '[', ']',  13,   0, 'a', 's'  ; 0x18-0x1F
    db 'd', 'f', 'g', 'h', 'j', 'k', 'l', ';'  ; 0x20-0x27
    db 39,  '`',   0, 92,  'z', 'x', 'c', 'v'  ; 0x28-0x2F
    db 'b', 'n', 'm', ',', '.', '/',   0,  '*'  ; 0x30-0x37
    db 0,  ' ',   0,   0,   0,   0,   0,   0   ; 0x38-0x3F

kb_buffer:
    times KB_BUF_SIZE db 0
kb_buf_head dd 0
kb_buf_tail dd 0
kb_shift    db 0            ; shift key state

section .text

keyboard_init:
    ; ลงทะเบียน IRQ1 handler
    ; (ทำผ่าน idt.asm ซึ่งได้ทำไปแล้ว)
    ret

; keyboard_handler ถูกเรียกจาก IRQ1
keyboard_handler:
    push eax ebx

    ; อ่าน scancode จาก keyboard port
    in al, KB_DATA_PORT

    ; ตรวจสอบ key release (bit 7 set)
    test al, 0x80
    jnz .key_release

    ; ตรวจสอบ shift
    cmp al, 0x2A            ; Left shift
    je .shift_on
    cmp al, 0x36            ; Right shift
    je .shift_on

    ; แปลง scancode → ASCII
    movzx ebx, al
    cmp ebx, 64             ; ตาราง scancode มีแค่ 64 entries
    jge .done

    movzx eax, byte [scancode_table + ebx]
    test eax, eax
    jz .done

    ; Apply shift
    cmp byte [kb_shift], 1
    jne .no_shift
    cmp al, 'a'
    jl .no_shift
    cmp al, 'z'
    jg .no_shift
    sub al, 32              ; lowercase → uppercase

.no_shift:
    ; ใส่ character ลงใน buffer
    mov ebx, [kb_buf_tail]
    mov [kb_buffer + ebx], al
    inc ebx
    and ebx, KB_BUF_SIZE - 1
    mov [kb_buf_tail], ebx
    jmp .done

.shift_on:
    mov byte [kb_shift], 1
    jmp .done

.key_release:
    and al, 0x7F            ; ลบ bit 7
    cmp al, 0x2A
    je .shift_off
    cmp al, 0x36
    je .shift_off
    jmp .done

.shift_off:
    mov byte [kb_shift], 0

.done:
    pop ebx eax
    ret

; keyboard_getchar() → al = character (blocking)
keyboard_getchar:
.wait:
    mov eax, [kb_buf_head]
    cmp eax, [kb_buf_tail]
    je .wait                ; buffer ว่าง, รอ

    ; ดึง character จาก buffer
    movzx eax, byte [kb_buffer + eax]
    push eax
    mov eax, [kb_buf_head]
    inc eax
    and eax, KB_BUF_SIZE - 1
    mov [kb_buf_head], eax
    pop eax
    ret
```

---

## Timer Driver (PIT)

```nasm
; drivers/timer.asm
; Programmable Interval Timer (PIT 8253/8254) driver

[BITS 32]

global timer_init
global timer_handler
global timer_get_ticks
global timer_sleep

extern scheduler_tick

%define PIT_CH0     0x40    ; Channel 0 data port
%define PIT_CMD     0x43    ; Command port
%define PIT_FREQ    1193182 ; PIT input frequency (Hz)
%define TICK_RATE   100     ; ต้องการ 100 Hz (10ms per tick)
%define PIT_DIVISOR (PIT_FREQ / TICK_RATE)

section .data
timer_ticks dd 0

section .text

timer_init:
    ; ตั้งค่า PIT mode 3 (Square Wave) สำหรับ channel 0
    mov al, 0x36            ; channel 0, lobyte/hibyte, mode 3, binary
    out PIT_CMD, al

    ; ส่ง divisor
    mov ax, PIT_DIVISOR     ; = 11932 ≈ 100Hz
    out PIT_CH0, al         ; low byte
    mov al, ah
    out PIT_CH0, al         ; high byte

    ret

; timer_handler ถูกเรียกจาก IRQ0 (INT 0x20)
timer_handler:
    inc dword [timer_ticks]

    ; บอก scheduler ให้ทำ context switch
    call scheduler_tick

    ret

timer_get_ticks:
    mov eax, [timer_ticks]
    ret

; timer_sleep(ticks) - busy wait
timer_sleep:
    push ebx
    mov ebx, [esp+8]        ; ticks to wait
    mov eax, [timer_ticks]
    add ebx, eax            ; target = current + wait

.wait:
    mov eax, [timer_ticks]
    cmp eax, ebx
    jl .wait

    pop ebx
    ret
```

---

## Process Scheduler (Round-Robin)

```nasm
; kernel/scheduler.asm
; Round-Robin Process Scheduler

[BITS 32]

global scheduler_init
global scheduler_tick
global scheduler_add_process
global scheduler_exit
global current_process

extern pmm_alloc
extern vga_printf

%define MAX_PROCESSES   16
%define STACK_SIZE      4096
%define PROCESS_RUNNING 1
%define PROCESS_READY   2
%define PROCESS_BLOCKED 3
%define PROCESS_DEAD    4

; PCB (Process Control Block) structure
; Offsets:
%define PCB_PID         0    ; dword: process ID
%define PCB_STATE       4    ; dword: state
%define PCB_ESP         8    ; dword: saved stack pointer
%define PCB_EIP         12   ; dword: saved instruction pointer (ไม่ใช้ตรงนี้)
%define PCB_CR3         16   ; dword: page directory
%define PCB_STACK       20   ; dword: stack base
%define PCB_NEXT        24   ; dword: next PCB pointer
%define PCB_NAME        28   ; 16 bytes: process name
%define PCB_SIZE        44   ; total size

section .data
process_list    dd 0         ; linked list head
current_process dd 0         ; currently running PCB
next_pid        dd 1         ; auto-increment PID

; PCB pool (static allocation)
align 16
pcb_pool:
    times MAX_PROCESSES * PCB_SIZE db 0

pcb_pool_used:
    times MAX_PROCESSES db 0

section .text

scheduler_init:
    mov dword [process_list], 0
    mov dword [current_process], 0
    ret

; scheduler_add_process(entry_point, name) → eax = PID
scheduler_add_process:
    push ebp
    mov ebp, esp
    push ebx ecx edx esi edi

    ; หา PCB ว่างจาก pool
    mov ecx, MAX_PROCESSES
    mov edi, pcb_pool
    mov esi, 0
.find_pcb:
    cmp byte [pcb_pool_used + esi], 0
    je .found_pcb
    inc esi
    add edi, PCB_SIZE
    loop .find_pcb

    ; ไม่มี PCB ว่าง
    xor eax, eax
    jmp .done

.found_pcb:
    mov byte [pcb_pool_used + esi], 1

    ; จอง stack
    call pmm_alloc
    test eax, eax
    jz .no_mem
    mov [edi + PCB_STACK], eax

    ; ตั้งค่า PCB
    mov eax, [next_pid]
    mov [edi + PCB_PID], eax
    inc dword [next_pid]

    mov dword [edi + PCB_STATE], PROCESS_READY

    ; ตั้งค่า initial stack
    ; เราจะ push arguments สำหรับ entry function
    mov eax, [edi + PCB_STACK]
    add eax, STACK_SIZE     ; top of stack
    sub eax, 4
    mov dword [eax], 0      ; return address = 0 (process จะไม่ return)
    mov [edi + PCB_ESP], eax

    ; ตั้งค่า EIP (entry point)
    mov eax, [ebp+8]        ; entry_point
    mov [edi + PCB_EIP], eax

    ; Copy process name
    mov esi, [ebp+12]       ; name
    push edi
    lea edi, [edi + PCB_NAME]
    mov ecx, 15
    rep movsb
    mov byte [edi], 0
    pop edi

    ; เพิ่มเข้า linked list
    mov eax, [process_list]
    mov [edi + PCB_NEXT], eax
    mov [process_list], edi

    ; ตั้ง current process ถ้าเป็นตัวแรก
    cmp dword [current_process], 0
    jne .already_has_current
    mov [current_process], edi

.already_has_current:
    mov eax, [edi + PCB_PID]
    jmp .done

.no_mem:
    xor eax, eax

.done:
    pop edi esi edx ecx ebx
    pop ebp
    ret

; scheduler_tick - เรียกจาก timer interrupt
; ทำ context switch แบบ round-robin
scheduler_tick:
    push eax ebx

    mov eax, [current_process]
    test eax, eax
    jz .done                ; ไม่มี process

    ; หา next process ที่พร้อม
    mov ebx, [eax + PCB_NEXT]
    test ebx, ebx
    jnz .check_next
    mov ebx, [process_list] ; wrap around

.check_next:
    cmp dword [ebx + PCB_STATE], PROCESS_READY
    je .switch

    ; ลองต่อไป
    mov ebx, [ebx + PCB_NEXT]
    test ebx, ebx
    jnz .check_next
    mov ebx, [process_list]

    cmp ebx, eax            ; กลับมาที่ current?
    je .done                ; มี process เดียว ไม่ต้อง switch

.switch:
    ; บันทึก ESP ของ current process
    ; (จริงๆ ต้องทำใน interrupt frame แต่ simplified ที่นี่)
    mov [eax + PCB_STATE], PROCESS_READY

    ; สลับไป process ใหม่
    mov [ebx + PCB_STATE], PROCESS_RUNNING
    mov [current_process], ebx

    ; (Context switch จริงๆ จะทำโดยการ restore ESP/EBP)
    ; ในระบบจริงต้องบันทึก/restore registers ทั้งหมด

.done:
    pop ebx eax
    ret

; scheduler_exit - process ยุติตัวเอง
scheduler_exit:
    mov eax, [current_process]
    mov dword [eax + PCB_STATE], PROCESS_DEAD
    ; ค้นหา PCB index แล้ว free
    push eax
    sub eax, pcb_pool
    push eax
    mov ecx, PCB_SIZE
    xor edx, edx
    ; eax / PCB_SIZE = index
    div ecx
    mov [pcb_pool_used + eax], byte 0
    pop eax
    pop eax

    ; เรียก scheduler ให้สลับไป process อื่น
    call scheduler_tick
    ; ถ้ากลับมาที่นี่แสดงว่าไม่มี process อื่น
    hlt
```

---

## System Call Interface

```nasm
; kernel/syscall.asm
; System Call handler via INT 0x80
; Conventions: eax=syscall_num, ebx,ecx,edx=args, eax=return

[BITS 32]

global syscall_init
global syscall_handler

extern vga_putchar
extern vga_puts
extern vga_printf
extern keyboard_getchar
extern scheduler_exit
extern scheduler_add_process

; Syscall numbers
%define SYS_EXIT    1
%define SYS_WRITE   2
%define SYS_READ    3
%define SYS_FORK    4
%define SYS_EXEC    5
%define SYS_GETPID  6
%define SYS_SLEEP   7
%define SYS_PUTS    8

section .text

syscall_init:
    ; INT 0x80 handler ได้ถูก set ใน idt_init แล้ว
    ret

; syscall_handler - เรียกจาก INT 0x80
syscall_handler:
    ; บันทึก registers
    push ebp
    mov ebp, esp
    push ebx ecx edx esi edi

    ; eax = syscall number
    cmp eax, SYS_EXIT
    je .sys_exit
    cmp eax, SYS_WRITE
    je .sys_write
    cmp eax, SYS_READ
    je .sys_read
    cmp eax, SYS_GETPID
    je .sys_getpid
    cmp eax, SYS_SLEEP
    je .sys_sleep
    cmp eax, SYS_PUTS
    je .sys_puts

    ; Unknown syscall
    mov eax, -1
    jmp .done

.sys_exit:
    ; ebx = exit code
    call scheduler_exit
    ; ไม่ return
    jmp .done

.sys_write:
    ; ebx = fd (1=stdout), ecx = buf, edx = count
    push edx ecx
.write_loop:
    test edx, edx
    jz .write_done
    push dword [ecx]
    call vga_putchar
    add esp, 4
    inc ecx
    dec edx
    jmp .write_loop
.write_done:
    pop ecx edx
    mov eax, edx            ; return bytes written
    jmp .done

.sys_read:
    ; ebx = fd, ecx = buf, edx = count
    ; อ่าน 1 character จาก keyboard
    call keyboard_getchar
    mov [ecx], al
    mov eax, 1
    jmp .done

.sys_getpid:
    ; คืน PID ของ current process
    ; (simplified: return 1)
    mov eax, 1
    jmp .done

.sys_sleep:
    ; ebx = ticks
    push ebx
    call timer_sleep
    add esp, 4
    xor eax, eax
    jmp .done

.sys_puts:
    ; ebx = string pointer
    push ebx
    call vga_puts
    add esp, 4
    xor eax, eax

.done:
    pop edi esi edx ecx ebx
    pop ebp
    iret                    ; คืนสู่ user mode
```

---

## User Mode: Ring 3 Transition

```nasm
; kernel/usermode.asm
; Transition to Ring 3 via IRET trick

[BITS 32]

global enter_user_mode

%define USER_CODE_SEL   (0x18 | 3)  ; index 3, RPL=3
%define USER_DATA_SEL   (0x20 | 3)  ; index 4, RPL=3

; enter_user_mode(entry_point, user_stack)
; เข้า Ring 3 โดยใช้ IRET
enter_user_mode:
    push ebp
    mov ebp, esp

    mov eax, [ebp+8]        ; entry point
    mov ecx, [ebp+12]       ; user stack pointer

    ; ตั้งค่า segment registers เป็น user segments
    mov ax, USER_DATA_SEL
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    ; สร้าง IRET frame บน stack:
    ; [ss, esp, eflags, cs, eip]
    push USER_DATA_SEL      ; ss (user data)
    push ecx                ; esp (user stack)
    pushf                   ; eflags
    or dword [esp], 0x200   ; set IF (enable interrupts in user mode)
    push USER_CODE_SEL      ; cs (user code)
    push dword [ebp+8]      ; eip (entry point)

    iret                    ; กระโดดไป Ring 3!
```

---

## ELF Loader

```nasm
; kernel/elf_loader.asm
; Simple ELF32 Loader - loads executable segments

[BITS 32]

global elf_load
global elf_get_entry

extern pmm_alloc
extern vga_printf

; ELF Header offsets
%define ELF_MAGIC       0       ; 4 bytes: 0x7F 'E' 'L' 'F'
%define ELF_CLASS       4       ; 1 byte: 1=32bit
%define ELF_ENTRY       24      ; 4 bytes: entry point
%define ELF_PHOFF       28      ; 4 bytes: program header offset
%define ELF_PHENTSIZE   42      ; 2 bytes: ph entry size
%define ELF_PHNUM       44      ; 2 bytes: number of ph entries

; Program Header offsets
%define PH_TYPE         0       ; 4 bytes: segment type
%define PH_OFFSET       4       ; 4 bytes: file offset
%define PH_VADDR        8       ; 4 bytes: virtual address
%define PH_FILESZ       16      ; 4 bytes: size in file
%define PH_MEMSZ        20      ; 4 bytes: size in memory
%define PT_LOAD         1       ; loadable segment

section .text

; elf_load(elf_data) → eax = entry point, 0 on error
elf_load:
    push ebp
    mov ebp, esp
    push ebx ecx edx esi edi

    mov esi, [ebp+8]        ; elf_data pointer

    ; ตรวจสอบ ELF magic
    cmp dword [esi + ELF_MAGIC], 0x464C457F  ; \x7FELF
    jne .invalid

    ; ตรวจสอบว่าเป็น 32-bit
    cmp byte [esi + ELF_CLASS], 1
    jne .invalid

    ; อ่าน program headers
    movzx ecx, word [esi + ELF_PHNUM]
    movzx edx, word [esi + ELF_PHENTSIZE]
    mov ebx, [esi + ELF_PHOFF]
    add ebx, esi            ; ebx = pointer to first PH

.load_segments:
    test ecx, ecx
    jz .done

    ; ตรวจสอบ PT_LOAD
    cmp dword [ebx + PH_TYPE], PT_LOAD
    jne .next_ph

    ; โหลด segment
    mov edi, [ebx + PH_VADDR]   ; destination
    mov esi, [ebp+8]
    add esi, [ebx + PH_OFFSET]  ; source

    mov eax, [ebx + PH_FILESZ]
    push ecx
    mov ecx, eax
    rep movsb               ; copy file data

    ; Zero-fill remaining (memsz - filesz)
    mov eax, [ebx + PH_MEMSZ]
    sub eax, [ebx + PH_FILESZ]
    mov ecx, eax
    xor eax, eax
    rep stosb
    pop ecx

    mov esi, [ebp+8]        ; restore esi

.next_ph:
    add ebx, edx            ; next program header
    dec ecx
    jmp .load_segments

.done:
    ; คืน entry point
    mov esi, [ebp+8]
    mov eax, [esi + ELF_ENTRY]
    jmp .ret

.invalid:
    xor eax, eax

.ret:
    pop edi esi edx ecx ebx
    pop ebp
    ret
```

---

## RAM Filesystem

```nasm
; kernel/ramfs.asm
; Simple in-memory filesystem

[BITS 32]

global ramfs_init
global ramfs_create
global ramfs_open
global ramfs_read
global ramfs_write
global ramfs_list

extern kmalloc
extern kfree
extern vga_printf

%define RAMFS_MAX_FILES     32
%define RAMFS_MAX_NAME      32
%define RAMFS_MAX_DATA      4096

; File entry structure:
; [name: 32 bytes][data: 4096 bytes][size: 4 bytes][used: 1 byte]
%define FE_NAME     0
%define FE_DATA     32
%define FE_SIZE     (32 + 4096)
%define FE_USED     (32 + 4096 + 4)
%define FE_TOTAL    (32 + 4096 + 4 + 1 + 3)  ; +3 for alignment

section .data

align 4
ramfs_table:
    times RAMFS_MAX_FILES * FE_TOTAL db 0

section .text

ramfs_init:
    ; เคลียร์ filesystem table
    mov edi, ramfs_table
    mov ecx, RAMFS_MAX_FILES * FE_TOTAL
    xor eax, eax
    rep stosb
    ret

; ramfs_create(name) → eax = fd (index), -1 on error
ramfs_create:
    push ebp
    mov ebp, esp
    push ebx ecx esi edi

    ; หา slot ว่าง
    mov ecx, RAMFS_MAX_FILES
    mov ebx, 0
    mov edi, ramfs_table

.find_slot:
    cmp byte [edi + FE_USED], 0
    je .found

    add edi, FE_TOTAL
    inc ebx
    loop .find_slot

    mov eax, -1
    jmp .done

.found:
    ; Copy name
    mov esi, [ebp+8]
    push edi
    add edi, FE_NAME
    mov ecx, RAMFS_MAX_NAME - 1
    rep movsb
    mov byte [edi], 0
    pop edi

    ; Mark as used
    mov byte [edi + FE_USED], 1
    mov dword [edi + FE_SIZE], 0

    mov eax, ebx            ; return fd

.done:
    pop edi esi ecx ebx
    pop ebp
    ret

; ramfs_write(fd, data, size) → eax = bytes written
ramfs_write:
    push ebp
    mov ebp, esp
    push ebx ecx esi edi

    mov ebx, [ebp+8]        ; fd
    imul edi, ebx, FE_TOTAL
    add edi, ramfs_table    ; file entry

    cmp byte [edi + FE_USED], 0
    je .error

    mov esi, [ebp+12]       ; data
    mov ecx, [ebp+16]       ; size

    ; จำกัดขนาด
    cmp ecx, RAMFS_MAX_DATA
    jle .ok_size
    mov ecx, RAMFS_MAX_DATA
.ok_size:
    mov [edi + FE_SIZE], ecx

    ; copy data
    push edi
    add edi, FE_DATA
    rep movsb
    pop edi

    mov eax, [edi + FE_SIZE]
    jmp .done

.error:
    mov eax, -1

.done:
    pop edi esi ecx ebx
    pop ebp
    ret

; ramfs_read(fd, buf, size) → eax = bytes read
ramfs_read:
    push ebp
    mov ebp, esp
    push ebx ecx esi edi

    mov ebx, [ebp+8]        ; fd
    imul esi, ebx, FE_TOTAL
    add esi, ramfs_table
    add esi, FE_DATA        ; source = data portion

    mov edi, [ebp+12]       ; destination buffer
    mov ecx, [ebp+16]       ; max size

    ; ปรับตามขนาดไฟล์จริง
    imul ebx, ebx, FE_TOTAL
    add ebx, ramfs_table
    mov eax, [ebx + FE_SIZE]
    cmp ecx, eax
    jle .do_copy
    mov ecx, eax

.do_copy:
    push ecx
    rep movsb
    pop eax                 ; return actual size

    pop edi esi ecx ebx
    pop ebp
    ret

; ramfs_list - แสดงไฟล์ทั้งหมด
ramfs_list:
    push ebx ecx

    mov ecx, RAMFS_MAX_FILES
    mov ebx, ramfs_table

.loop:
    cmp byte [ebx + FE_USED], 0
    je .next

    ; แสดงชื่อและขนาด
    push dword [ebx + FE_SIZE]
    lea eax, [ebx + FE_NAME]
    push eax
    push msg_ls_fmt
    call vga_printf
    add esp, 12

.next:
    add ebx, FE_TOTAL
    loop .loop

    pop ecx ebx
    ret

section .data
msg_ls_fmt db '  %s  (%d bytes)', 10, 0
```

---

## Simple Shell

```nasm
; kernel/shell.asm
; Built-in kernel shell

[BITS 32]

global shell_start

extern vga_printf
extern vga_puts
extern vga_putchar
extern keyboard_getchar
extern ramfs_list
extern ramfs_create
extern elf_load
extern enter_user_mode
extern scheduler_add_process

%define MAX_CMD     128
%define MAX_ARGS    8

section .data

prompt          db 'mini_os> ', 0
msg_unknown     db 'Unknown command: ', 0
msg_newline     db 10, 0
msg_help        db 'Commands: help, ls, echo, clear, run, reboot', 10, 0
msg_hello       db 'Hello from Mini OS!', 10, 0

cmd_help        db 'help', 0
cmd_ls          db 'ls', 0
cmd_echo        db 'echo', 0
cmd_clear       db 'clear', 0
cmd_run         db 'run', 0
cmd_reboot      db 'reboot', 0

cmd_buf:        times MAX_CMD db 0

section .text

shell_start:
    push msg_hello
    call vga_puts
    add esp, 4

.shell_loop:
    ; แสดง prompt
    push prompt
    call vga_puts
    add esp, 4

    ; อ่านคำสั่ง
    call read_command

    ; แยกและ execute คำสั่ง
    call execute_command

    jmp .shell_loop

; อ่าน command line จาก keyboard
read_command:
    push esi ecx
    mov esi, cmd_buf
    xor ecx, ecx

.read_loop:
    call keyboard_getchar

    cmp al, 13              ; Enter
    je .done_read

    cmp al, 8               ; Backspace
    je .backspace

    cmp ecx, MAX_CMD - 1
    jge .read_loop          ; ไม่ให้เกิน buffer

    mov [esi + ecx], al
    inc ecx

    push eax
    call vga_putchar
    add esp, 4

    jmp .read_loop

.backspace:
    test ecx, ecx
    jz .read_loop
    dec ecx
    push dword 8
    call vga_putchar
    add esp, 4
    jmp .read_loop

.done_read:
    mov byte [esi + ecx], 0  ; null terminate
    push dword 10
    call vga_putchar         ; newline
    add esp, 4

    pop ecx esi
    ret

; execute command
execute_command:
    push ebp
    mov ebp, esp
    push ebx

    ; ตรวจสอบ command ว่างเปล่า
    cmp byte [cmd_buf], 0
    je .done

    ; เปรียบเทียบกับ commands ที่รู้จัก
    mov ebx, cmd_buf

    push cmd_help
    push ebx
    call strcmp
    add esp, 8
    test eax, eax
    jz .do_help

    push cmd_ls
    push ebx
    call strcmp
    add esp, 8
    test eax, eax
    jz .do_ls

    push cmd_echo
    push ebx
    call strcmp
    add esp, 8
    test eax, eax
    jz .do_echo

    push cmd_clear
    push ebx
    call strcmp
    add esp, 8
    test eax, eax
    jz .do_clear

    push cmd_reboot
    push ebx
    call strcmp
    add esp, 8
    test eax, eax
    jz .do_reboot

    ; Unknown command
    push cmd_buf
    push msg_unknown
    call vga_printf
    add esp, 8
    push msg_newline
    call vga_puts
    add esp, 4
    jmp .done

.do_help:
    push msg_help
    call vga_puts
    add esp, 4
    jmp .done

.do_ls:
    call ramfs_list
    jmp .done

.do_echo:
    ; พิมพ์ส่วนที่เหลือหลัง "echo "
    lea eax, [cmd_buf + 5]
    push eax
    call vga_puts
    add esp, 4
    push msg_newline
    call vga_puts
    add esp, 4
    jmp .done

.do_clear:
    call vga_clear
    jmp .done

.do_reboot:
    ; Reboot via keyboard controller
    mov al, 0xFE
    out 0x64, al
    hlt

.done:
    pop ebx
    pop ebp
    ret

; strcmp(s1, s2) → 0 if equal
strcmp:
    push esi edi
    mov esi, [esp+12]       ; s1
    mov edi, [esp+16]       ; s2
.loop:
    lodsb
    mov cl, [edi]
    inc edi
    cmp al, cl
    jne .not_equal
    test al, al
    jnz .loop
    xor eax, eax            ; equal
    jmp .done
.not_equal:
    mov eax, 1
.done:
    pop edi esi
    ret
```

---

## User Mode Library

```nasm
; user/user_lib.asm
; System call wrappers for user programs

[BITS 32]

global sys_write
global sys_read
global sys_exit
global sys_puts
global sys_getpid

%define SYS_EXIT    1
%define SYS_WRITE   2
%define SYS_READ    3
%define SYS_GETPID  6
%define SYS_PUTS    8

section .text

; sys_exit(code)
sys_exit:
    mov eax, SYS_EXIT
    mov ebx, [esp+4]        ; exit code
    int 0x80
    ; ไม่ return

; sys_write(fd, buf, count) → bytes written
sys_write:
    push ebx ecx edx
    mov eax, SYS_WRITE
    mov ebx, [esp+16]       ; fd
    mov ecx, [esp+20]       ; buf
    mov edx, [esp+24]       ; count
    int 0x80
    pop edx ecx ebx
    ret

; sys_puts(str)
sys_puts:
    push ebx
    mov eax, SYS_PUTS
    mov ebx, [esp+8]        ; string
    int 0x80
    pop ebx
    ret

; sys_read(fd, buf, count) → bytes read
sys_read:
    push ebx ecx edx
    mov eax, SYS_READ
    mov ebx, [esp+16]
    mov ecx, [esp+20]
    mov edx, [esp+24]
    int 0x80
    pop edx ecx ebx
    ret

; sys_getpid() → pid
sys_getpid:
    mov eax, SYS_GETPID
    int 0x80
    ret
```

---

## Hello User Program

```nasm
; user/hello_user.asm
; Simple user mode program (Ring 3)

[BITS 32]

extern sys_puts
extern sys_exit
extern sys_getpid

section .data
msg1 db 'Hello from user mode (Ring 3)!', 10, 0
msg2 db 'My PID is: ', 0
msg3 db 10, 0

section .text
global _start

_start:
    ; แสดง hello
    push msg1
    call sys_puts
    add esp, 4

    ; แสดง PID
    push msg2
    call sys_puts
    add esp, 4

    call sys_getpid
    ; (จริงๆ ต้องแปลง integer เป็น string ก่อน)

    push msg3
    call sys_puts
    add esp, 4

    ; ออกจากโปรแกรม
    push 0
    call sys_exit
```

---

## String Library

```nasm
; lib/string.asm
; String utility functions

[BITS 32]

global strlen
global strcpy
global strncpy
global memset
global memcpy
global itoa

section .text

; strlen(str) → eax = length
strlen:
    push edi ecx
    mov edi, [esp+12]       ; str
    xor ecx, ecx
    xor al, al
    mov ecx, 0xFFFFFFFF
    repne scasb             ; scan for null
    not ecx
    dec ecx
    mov eax, ecx
    pop ecx edi
    ret

; strcpy(dst, src) → dst
strcpy:
    push esi edi
    mov edi, [esp+12]       ; dst
    mov esi, [esp+16]       ; src
.loop:
    lodsb
    stosb
    test al, al
    jnz .loop
    mov eax, [esp+12]       ; return dst
    pop edi esi
    ret

; memset(ptr, val, count)
memset:
    push edi ecx
    mov edi, [esp+12]       ; ptr
    mov eax, [esp+16]       ; val
    mov ecx, [esp+20]       ; count
    rep stosb
    pop ecx edi
    ret

; memcpy(dst, src, count)
memcpy:
    push esi edi ecx
    mov edi, [esp+16]       ; dst
    mov esi, [esp+20]       ; src
    mov ecx, [esp+24]       ; count
    rep movsb
    pop ecx edi esi
    ret

; itoa(num, buf, base)
; แปลง integer เป็น string
itoa:
    push ebp
    mov ebp, esp
    push ebx ecx edx esi edi

    mov eax, [ebp+8]        ; num
    mov edi, [ebp+12]       ; buf
    mov ebx, [ebp+16]       ; base

    ; Handle negative
    test eax, eax
    jns .positive
    mov byte [edi], '-'
    inc edi
    neg eax

.positive:
    ; แปลงตัวเลข (reverse order)
    mov esi, edi            ; บันทึก start
    mov ecx, 0              ; digit count

.convert_loop:
    xor edx, edx
    div ebx                 ; eax/base, remainder in edx
    add edx, '0'
    cmp edx, '9'
    jle .store
    add edx, 'A' - '9' - 1 ; A-F สำหรับ hex

.store:
    mov [edi], dl
    inc edi
    inc ecx
    test eax, eax
    jnz .convert_loop

    mov byte [edi], 0       ; null terminate

    ; Reverse string
    dec edi
.reverse:
    cmp esi, edi
    jge .done_reverse
    mov al, [esi]
    mov ah, [edi]
    mov [esi], ah
    mov [edi], al
    inc esi
    dec edi
    jmp .reverse

.done_reverse:
    pop edi esi edx ecx ebx
    pop ebp
    ret
```

---

## การทดสอบด้วย QEMU

### Build และ Run
```bash
# Build ทุกอย่าง
cd mini_os
make all

# Run ใน QEMU (floppy boot)
make run

# หรือรันด้วย options เพิ่มเติม
qemu-system-i386 \
    -fda mini_os.img \
    -boot a \
    -m 64M \
    -no-reboot \
    -no-shutdown

# รันพร้อม serial console (debug output)
qemu-system-i386 \
    -fda mini_os.img \
    -boot a \
    -m 64M \
    -serial stdio \
    -no-reboot

# รันแบบ headless (ไม่มี GUI)
qemu-system-i386 \
    -fda mini_os.img \
    -boot a \
    -nographic \
    -no-reboot
```

### GDB Debugging

```bash
# Terminal 1: เริ่ม QEMU พร้อม GDB server
qemu-system-i386 \
    -fda mini_os.img \
    -boot a \
    -m 64M \
    -s -S          # -s = GDB port 1234, -S = หยุดรอ

# Terminal 2: เชื่อมต่อ GDB
gdb

# ใน GDB:
(gdb) target remote :1234
(gdb) symbol-file kernel/kernel.elf
(gdb) break kernel_entry
(gdb) break vga_printf
(gdb) break syscall_handler
(gdb) continue

# ดู registers
(gdb) info registers
(gdb) info registers eax ebx esp

# ดู memory
(gdb) x/10x 0x100000     # hex dump ที่ kernel load address
(gdb) x/10i $eip         # disassemble 10 instructions จาก PC

# Watch points
(gdb) watch timer_ticks
(gdb) watch [current_process]

# Step
(gdb) stepi              # step 1 instruction
(gdb) nexti              # next instruction (skip calls)

# Backtrace
(gdb) backtrace

# ตั้ง breakpoint ที่ interrupt handler
(gdb) break isr_common
(gdb) break irq_common
(gdb) break keyboard_handler
```

---

## การ Debug ทั่วไป

### ตรวจสอบ GRUB multiboot (ถ้าใช้)
```bash
# ตรวจสอบ ELF ของ kernel
readelf -h kernel/kernel.elf
readelf -l kernel/kernel.elf    # program headers
readelf -S kernel/kernel.elf    # section headers

# ตรวจสอบ symbols
nm kernel/kernel.elf | grep kernel_entry
nm kernel/kernel.elf | sort
```

### ตรวจสอบ Boot sector
```bash
# ดู binary ของ stage1
xxd boot/stage1.bin | head -40

# ตรวจสอบ boot signature
xxd boot/stage1.bin | tail -5

# ตรวจสอบ disk image
xxd mini_os.img | head -50
```

### ตรวจสอบ GDT ใน QEMU Monitor
```
# ใน QEMU กด Ctrl+Alt+2 เข้า monitor
(qemu) info registers
(qemu) info mem
(qemu) info tlb
(qemu) xp /10xw 0x100000    # examine physical memory
```

---

## QEMU Test Scenarios

```bash
# Test 1: Boot สำเร็จ
make run
# คาดหวัง: เห็น "Mini OS v1.0 - Ready" และ prompt "mini_os> "

# Test 2: Keyboard input
# พิมพ์ "help" แล้วกด Enter
# คาดหวัง: เห็น list of commands

# Test 3: ls command
# พิมพ์ "ls"
# คาดหวัง: แสดงไฟล์ใน ramfs (ถ้ามี)

# Test 4: echo command
# พิมพ์ "echo hello world"
# คาดหวัง: เห็น "hello world"

# Test 5: Memory test
# ใน GDB หลัง boot:
(gdb) print pmm_alloc()
# คาดหวัง: ได้ address ที่ถูกต้อง

# Test 6: Exception handling
# ทำให้เกิด divide by zero ใน user mode
# คาดหวัง: KERNEL PANIC message
```

---

## สรุป Architecture Decisions

### 1. Two-Stage Bootloader
- Stage 1 (MBR) ขนาด 512 bytes จำกัดมาก ใช้แค่โหลด Stage 2
- Stage 2 ทำงานหนักกว่า: memory detection, A20, GDT, เข้า Protected Mode

### 2. GDT Design
```
Index 0: Null descriptor
Index 1 (0x08): Kernel Code - Ring 0, Execute/Read
Index 2 (0x10): Kernel Data - Ring 0, Read/Write
Index 3 (0x18): User Code  - Ring 3, Execute/Read
Index 4 (0x20): User Data  - Ring 3, Read/Write
Index 5 (0x28): TSS        - Task State Segment
```

### 3. Memory Layout
```
0x00000 - 0x00FFF: Real Mode IVT
0x00500 - 0x07BFF: Free (ใช้เก็บ E820 memory map)
0x07C00 - 0x07DFF: Stage 1 MBR
0x07E00 - 0x0FFFF: Stage 2
0x10000 - 0x2FFFF: Temporary kernel load area
0x30000 - 0x307FF: PMM Bitmap
0x90000 - 0x9FFFF: Kernel Stack
0x100000+         : Kernel (loaded by ELF)
0x400000+         : Kernel Heap
0x40000000+       : User Space
```

### 4. Interrupt Architecture
```
INT 0-31:  CPU Exceptions (handled by isr_common)
INT 32:    IRQ0 - Timer (PIT)
INT 33:    IRQ1 - Keyboard (PS/2)
INT 34-47: IRQ2-15 (reserved)
INT 0x80:  System Call (Ring 3 → Ring 0)
```

### 5. System Call Table
```
1  - exit(code)
2  - write(fd, buf, count)
3  - read(fd, buf, count)
4  - fork() [simplified]
5  - exec(path)
6  - getpid()
7  - sleep(ticks)
8  - puts(str)
```

---

## แบบฝึกหัดและการปรับปรุง

### ระดับ Beginner
1. เพิ่ม command `date` ที่อ่าน RTC (CMOS) และแสดงวันเวลา
2. เพิ่มสีให้ shell prompt (VGA color codes)
3. เพิ่ม command `uptime` ที่แสดง timer_ticks

### ระดับ Intermediate
1. เพิ่ม Paging ที่สมบูรณ์พร้อม virtual address spaces แยกกันต่อ process
2. สร้าง proper fork() ที่ copy page tables
3. เพิ่ม pipe() และ redirect สำหรับ shell
4. สร้าง simple FAT12 filesystem บน floppy image แทน ramfs

### ระดับ Advanced
1. เพิ่ม SMP support (multi-processor)
2. สร้าง network stack (NE2000 emulated NIC ใน QEMU)
3. เพิ่ม VESA framebuffer mode (1024x768 graphics)
4. Port ไปยัง x86_64 (Long Mode)

---

## Troubleshooting

### ปัญหา: Bootloader ไม่ boot

```bash
# ตรวจสอบ boot signature
xxd mini_os.img | grep "55 aa"
# ควรเห็น 0x1FE: 55 aa

# ตรวจสอบขนาด stage1
ls -la boot/stage1.bin
# ต้องเป็น 512 bytes พอดี
```

### ปัญหา: Triple Fault (QEMU restart ซ้ำๆ)

```bash
# เปิด logging ใน QEMU
qemu-system-i386 -fda mini_os.img -boot a \
    -d cpu_reset,int,exec \
    -D /tmp/qemu.log

# ดู log
tail -100 /tmp/qemu.log
```

### ปัญหา: General Protection Fault (#13)

สาเหตุที่พบบ่อย:
1. Segment selector ผิด (เช่น ใช้ Ring 3 selector ใน Ring 0)
2. Stack alignment ผิด
3. Access ไปยัง address ที่ไม่ได้ map
4. ใช้ instruction ที่ไม่อนุญาตใน Ring 3

```nasm
; ตรวจสอบใน GDB เมื่อเกิด GPF
(gdb) break isr13          ; General Protection Fault
(gdb) commands
> print $eax
> print $cs
> print $eip
> backtrace
> end
(gdb) continue
```

### ปัญหา: Page Fault (#14)

```nasm
; CR2 register เก็บ faulting address
(gdb) break isr14
(gdb) commands
> print/x $cr2
> print/x $eip
> end
```

---

## สรุป

ในบทนี้เราได้สร้าง Mini OS ที่มีองค์ประกอบครบถ้วน:

| Component | File | หน้าที่ |
|-----------|------|---------|
| Stage 1 Bootloader | boot/stage1.asm | โหลด Stage 2 จาก MBR |
| Stage 2 Bootloader | boot/stage2.asm | เข้า Protected Mode, โหลด Kernel |
| Kernel Entry | kernel/kernel.asm | Initialize ทุก subsystem |
| GDT | kernel/gdt.asm | Segment descriptors + TSS |
| IDT | kernel/idt.asm | Interrupt gate descriptors |
| ISR | kernel/isr.asm | Exception/IRQ handlers |
| PIC | kernel/pic.asm | Remap และ enable interrupts |
| Paging | kernel/paging.asm | Identity mapping |
| PMM | kernel/pmm.asm | Bitmap physical allocator |
| Heap | kernel/heap.asm | Kernel dynamic memory |
| VGA | drivers/vga.asm | Text mode console |
| Keyboard | drivers/keyboard.asm | IRQ1 keyboard input |
| Timer | drivers/timer.asm | PIT 100Hz timer |
| Scheduler | kernel/scheduler.asm | Round-robin PCB |
| Syscall | kernel/syscall.asm | INT 0x80 interface |
| ELF Loader | kernel/elf_loader.asm | Load ELF32 binaries |
| Shell | kernel/shell.asm | Basic command shell |
| RAMFS | kernel/ramfs.asm | In-memory filesystem |
| User Lib | user/user_lib.asm | Syscall wrappers |
| String Lib | lib/string.asm | String utilities |

Mini OS นี้เป็นพื้นฐานที่ดีในการเรียนรู้ Systems Programming และ OS Development ขั้นสูงต่อไป!

---

*จบ Part 090: Mini OS - Complete Project*
