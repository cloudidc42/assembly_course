# Part 100: World-Class Project — Complete OS Kernel (Grand Finale)

> **ระดับ:** Expert | **ภาษา:** x86-64 Assembly + C | **เป้าหมาย:** สร้าง OS kernel ที่สมบูรณ์

---

## บทนำ: จุดสูงสุดของหลักสูตร

คุณมาถึง Part 100 แล้ว — Grand Finale ของหลักสูตร Assembly และ OS Development  
ในบทนี้เราจะสร้าง **MiniOS-X** ซึ่งเป็น 64-bit OS kernel ที่มีคุณสมบัติครบถ้วน:

- Multiboot2 boot + UEFI path
- Long mode (64-bit) + SMP (multiprocessor)
- Memory management: PMM + VMM + Slab allocator
- Process management + ELF64 loader
- System calls via SYSCALL/SYSRET
- VFS + Ext2 filesystem
- AHCI (SATA) + E1000 (NIC) drivers
- Network stack: Ethernet → IPv4 → TCP
- Interactive shell
- Kernel unit tests + QEMU scripts

---

## ส่วนที่ 1: Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    User Space                        │
│  Shell  │  Programs  │  Libraries (future libc)     │
├─────────────────────────────────────────────────────┤
│                  System Call Layer                   │
│         SYSCALL/SYSRET  (fast path, ring 0↔3)       │
├─────────────────────────────────────────────────────┤
│                   VFS Layer                          │
│    vnode/vfile abstraction  │  mount table           │
├──────────────┬──────────────┬───────────────────────┤
│  Ext2 Driver │  tmpfs       │  devfs                │
├──────────────┴──────────────┴───────────────────────┤
│              Process / Thread Management             │
│   PCB  │  Scheduler (round-robin)  │  ELF64 loader  │
├─────────────────────────────────────────────────────┤
│              Memory Management                       │
│  PMM (bitmap)  │  VMM (4-level paging)  │  Slab     │
├─────────────────────────────────────────────────────┤
│              Hardware Abstraction                    │
│  APIC  │  IOAPIC  │  ACPI  │  HPET  │  PCI         │
├──────────┬───────────┬──────────────────────────────┤
│  AHCI    │  E1000    │  PS/2 KB  │  Serial (debug) │
└──────────┴───────────┴──────────────────────────────┘
           │           │
        SATA disk    Ethernet NIC
```

### Memory Map (64-bit Virtual)

```
0xFFFFFFFF80000000  ← kernel text/data (high half, -2GB)
0xFFFFFF8000000000  ← kernel heap / dynamic alloc
0xFFFF800000000000  ← direct physical map (physmap)
0x0000800000000000  ← (hole — non-canonical)
0x00007FFFFFFFFFFF  ← user stack top
0x0000000000400000  ← user code start
0x0000000000000000  ← (null guard page)
```

---

## ส่วนที่ 2: Repository Structure

```
minios/
├── Makefile
├── linker.ld
├── boot/
│   ├── multiboot2.asm      ; multiboot2 header + 32→64 bit trampoline
│   ├── gdt64.asm           ; GDT for long mode
│   └── entry64.asm         ; 64-bit C entry point
├── kernel/
│   ├── main.c              ; kmain()
│   ├── panic.c             ; kernel panic
│   ├── printk.c            ; early console output
│   └── serial.c            ; COM1 debug output
├── arch/x86_64/
│   ├── cpu.c               ; CPUID, MSR helpers
│   ├── gdt.c               ; runtime GDT/TSS
│   ├── idt.c               ; IDT + exception handlers
│   ├── apic.c              ; local APIC
│   ├── ioapic.c            ; IOAPIC
│   ├── smp.c               ; AP startup (INIT/SIPI)
│   ├── syscall.c           ; SYSCALL/SYSRET setup
│   └── tsc.c               ; TSC calibration
├── acpi/
│   ├── acpi.c              ; RSDP→RSDT/XSDT parser
│   ├── madt.c              ; MADT (SMP topology)
│   └── hpet.c              ; HPET timer
├── mm/
│   ├── pmm.c               ; physical memory manager (bitmap)
│   ├── vmm.c               ; virtual memory (4-level paging)
│   ├── slab.c              ; slab allocator
│   └── kmalloc.c           ; kmalloc/kfree
├── proc/
│   ├── process.c           ; PCB, kernel threads
│   ├── scheduler.c         ; round-robin scheduler
│   ├── elf64.c             ; ELF64 loader
│   └── syscall_table.c     ; syscall dispatch
├── vfs/
│   ├── vfs.c               ; VFS core
│   ├── vnode.c             ; vnode operations
│   └── mount.c             ; mount table
├── fs/
│   └── ext2/
│       ├── ext2.c          ; superblock, inodes
│       ├── ext2_dir.c      ; directory operations
│       └── ext2_block.c    ; block allocation
├── drivers/
│   ├── ahci.c              ; AHCI (SATA) driver
│   ├── e1000.c             ; Intel E1000 NIC
│   ├── ps2kbd.c            ; PS/2 keyboard
│   └── pci.c               ; PCI enumeration
├── net/
│   ├── ethernet.c          ; Ethernet frame handling
│   ├── arp.c               ; ARP
│   ├── ipv4.c              ; IPv4
│   ├── tcp.c               ; TCP basic
│   └── udp.c               ; UDP
├── shell/
│   └── shell.c             ; interactive shell
└── tests/
    ├── test_pmm.c
    ├── test_vmm.c
    └── run_tests.sh
```

---

## ส่วนที่ 3: Bootloader — Multiboot2 Header

```nasm
; boot/multiboot2.asm
; Multiboot2 header + 32-bit protected mode → 64-bit long mode

bits 32

; ─── Multiboot2 Header ────────────────────────────────────
MULTIBOOT2_MAGIC    equ 0xE85250D6
MULTIBOOT2_ARCH     equ 0           ; i386
MB2_HEADER_LEN      equ (mb2_header_end - mb2_header_start)
MB2_CHECKSUM        equ -(MULTIBOOT2_MAGIC + MULTIBOOT2_ARCH + MB2_HEADER_LEN)

section .multiboot2
align 8
mb2_header_start:
    dd MULTIBOOT2_MAGIC
    dd MULTIBOOT2_ARCH
    dd MB2_HEADER_LEN
    dd MB2_CHECKSUM

    ; Tag: framebuffer request
    align 8
    dw 5                ; type = framebuffer
    dw 0                ; flags
    dd 20               ; size
    dd 1024             ; width
    dd 768              ; height
    dd 32               ; depth (bpp)

    ; Tag: end
    align 8
    dw 0
    dw 0
    dd 8
mb2_header_end:

; ─── Entry point from GRUB ────────────────────────────────
section .text
global _start
_start:
    ; eax = 0x36D76289 (multiboot2 magic)
    ; ebx = physical address of multiboot2 info structure
    cli
    mov esp, stack_top - KERNEL_VIRT_BASE  ; temp stack (physical)

    ; Save multiboot2 info pointer for later
    mov [mb2_info_phys - KERNEL_VIRT_BASE], ebx

    ; ── 1. Check CPUID availability ──
    pushfd
    pop eax
    mov ecx, eax
    xor eax, (1 << 21)
    push eax
    popfd
    pushfd
    pop eax
    push ecx
    popfd
    xor eax, ecx
    jz .no_cpuid

    ; ── 2. Check Long Mode support ──
    mov eax, 0x80000000
    cpuid
    cmp eax, 0x80000001
    jb .no_long_mode
    mov eax, 0x80000001
    cpuid
    test edx, (1 << 29)     ; LM bit
    jz .no_long_mode

    ; ── 3. Set up identity-mapped page tables for boot ──
    call setup_paging

    ; ── 4. Load GDT64 ──
    lgdt [gdt64_ptr - KERNEL_VIRT_BASE]

    ; ── 5. Enable PAE, PGE ──
    mov eax, cr4
    or  eax, (1 << 5) | (1 << 7)   ; PAE | PGE
    mov cr4, eax

    ; ── 6. Set CR3 to PML4 ──
    mov eax, pml4 - KERNEL_VIRT_BASE
    mov cr3, eax

    ; ── 7. Enable Long Mode (EFER.LME) ──
    mov ecx, 0xC0000080     ; EFER MSR
    rdmsr
    or  eax, (1 << 8)       ; LME
    wrmsr

    ; ── 8. Enable paging + protected mode ──
    mov eax, cr0
    or  eax, (1 << 31) | (1 << 0)  ; PG | PE
    mov cr0, eax

    ; ── 9. Far jump to 64-bit code segment ──
    jmp 0x08:long_mode_start - KERNEL_VIRT_BASE

.no_cpuid:
.no_long_mode:
    hlt
    jmp $

; ─── Page Table Setup (boot identity + kernel high-half) ──
setup_paging:
    ; PML4[0]   → PDPT_low  (identity map 0–1GB)
    ; PML4[511] → PDPT_high (kernel at -2GB)
    mov edi, pml4 - KERNEL_VIRT_BASE
    xor eax, eax
    mov ecx, 4096 / 4
    rep stosd               ; zero PML4

    ; PML4[0] → PDPT_low
    mov eax, (pdpt_low - KERNEL_VIRT_BASE) | 3
    mov [pml4 - KERNEL_VIRT_BASE], eax

    ; PML4[511] → PDPT_high
    mov eax, (pdpt_high - KERNEL_VIRT_BASE) | 3
    mov [pml4 - KERNEL_VIRT_BASE + 511*8], eax

    ; PDPT_low[0] → PD_low (2MB pages)
    mov eax, (pd_low - KERNEL_VIRT_BASE) | 3
    mov [pdpt_low - KERNEL_VIRT_BASE], eax

    ; PDPT_high[510] → PD_low (same physical, high-half alias)
    mov [pdpt_high - KERNEL_VIRT_BASE + 510*8], eax

    ; PD_low: map first 4MB with 2MB pages
    mov dword [pd_low - KERNEL_VIRT_BASE],       0x00000083  ; 0MB, present|RW|PS
    mov dword [pd_low - KERNEL_VIRT_BASE + 8],   0x00200083  ; 2MB
    ret

; ─── GDT64 ────────────────────────────────────────────────
align 16
gdt64:
    dq 0                    ; null descriptor
    dq 0x00AF9A000000FFFF   ; code64: L=1, P=1, DPL=0, C/E
    dq 0x00CF92000000FFFF   ; data64: P=1, DPL=0, RW
    dq 0x00AFFA000000FFFF   ; user code64: DPL=3
    dq 0x00CFF2000000FFFF   ; user data64: DPL=3
gdt64_end:

gdt64_ptr:
    dw gdt64_end - gdt64 - 1
    dd gdt64 - KERNEL_VIRT_BASE

; ─── 64-bit entry ─────────────────────────────────────────
bits 64
KERNEL_VIRT_BASE equ 0xFFFFFFFF80000000

long_mode_start:
    ; reload segment registers
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax

    ; switch to virtual high-half stack
    mov rsp, stack_top

    ; clear identity map (PML4[0])
    mov qword [pml4], 0
    mov rax, cr3
    mov cr3, rax            ; flush TLB

    ; Pass mb2_info pointer to kmain
    mov rdi, [mb2_info_phys]
    add rdi, 0xFFFFFFFF80000000   ; convert phys → virt

    extern kmain
    call kmain
    hlt

; ─── BSS ──────────────────────────────────────────────────
section .bss
align 4096
pml4:       resb 4096
pdpt_low:   resb 4096
pdpt_high:  resb 4096
pd_low:     resb 4096

align 16
stack_bottom: resb 65536   ; 64KB boot stack
stack_top:

mb2_info_phys: resq 1
```

---

## ส่วนที่ 4: GDT + TSS (Runtime)

```c
// arch/x86_64/gdt.c
#include "gdt.h"
#include "tss.h"
#include <stdint.h>

// GDT entries (per-CPU in SMP)
typedef struct {
    uint16_t limit_low;
    uint16_t base_low;
    uint8_t  base_mid;
    uint8_t  access;
    uint8_t  granularity;
    uint8_t  base_high;
} __attribute__((packed)) gdt_entry_t;

// TSS (Task State Segment) for ring switches
typedef struct {
    uint32_t reserved0;
    uint64_t rsp[3];        // RSP0, RSP1, RSP2
    uint64_t reserved1;
    uint64_t ist[7];        // IST1–IST7 (interrupt stacks)
    uint64_t reserved2;
    uint16_t reserved3;
    uint16_t iopb_offset;
} __attribute__((packed)) tss_t;

// System (16-byte) descriptor for TSS
typedef struct {
    uint16_t limit_low;
    uint16_t base_0_15;
    uint8_t  base_16_23;
    uint8_t  type_dpl_p;    // 0x89 = present, DPL=0, TSS avail
    uint8_t  limit_flags;
    uint8_t  base_24_31;
    uint32_t base_32_63;
    uint32_t reserved;
} __attribute__((packed)) tss_descriptor_t;

#define GDT_ENTRIES 8
static gdt_entry_t   gdt[GDT_ENTRIES];
static tss_descriptor_t tss_desc;  // placed at gdt[5] (8 bytes each, but TSS is 16 bytes)
static tss_t         kernel_tss __attribute__((aligned(16)));

static uint8_t ist_stacks[7][8192] __attribute__((aligned(16)));

typedef struct {
    uint16_t limit;
    uint64_t base;
} __attribute__((packed)) gdtr_t;

static void gdt_set_entry(int idx, uint32_t base, uint32_t limit,
                           uint8_t access, uint8_t gran) {
    gdt[idx].limit_low   = limit & 0xFFFF;
    gdt[idx].base_low    = base  & 0xFFFF;
    gdt[idx].base_mid    = (base >> 16) & 0xFF;
    gdt[idx].access      = access;
    gdt[idx].granularity = ((limit >> 16) & 0x0F) | (gran & 0xF0);
    gdt[idx].base_high   = (base >> 24) & 0xFF;
}

void gdt_init(void) {
    // 0: null
    gdt_set_entry(0, 0, 0, 0, 0);
    // 1: kernel code (64-bit): L=1, P=1, DPL=0
    gdt_set_entry(1, 0, 0xFFFFF, 0x9A, 0xA0);
    // 2: kernel data: P=1, DPL=0, RW
    gdt_set_entry(2, 0, 0xFFFFF, 0x92, 0xC0);
    // 3: user data: DPL=3
    gdt_set_entry(3, 0, 0xFFFFF, 0xF2, 0xC0);
    // 4: user code (64-bit): DPL=3
    gdt_set_entry(4, 0, 0xFFFFF, 0xFA, 0xA0);

    // TSS setup
    for (int i = 0; i < 7; i++) {
        kernel_tss.ist[i] = (uint64_t)&ist_stacks[i][8192];
    }
    kernel_tss.iopb_offset = sizeof(tss_t);

    uint64_t tss_base  = (uint64_t)&kernel_tss;
    uint32_t tss_limit = sizeof(tss_t) - 1;

    // TSS descriptor is 16 bytes, occupies slots 5+6
    tss_desc.limit_low    = tss_limit & 0xFFFF;
    tss_desc.base_0_15    = tss_base & 0xFFFF;
    tss_desc.base_16_23   = (tss_base >> 16) & 0xFF;
    tss_desc.type_dpl_p   = 0x89;
    tss_desc.limit_flags  = ((tss_limit >> 16) & 0xF);
    tss_desc.base_24_31   = (tss_base >> 24) & 0xFF;
    tss_desc.base_32_63   = (tss_base >> 32) & 0xFFFFFFFF;
    tss_desc.reserved     = 0;

    // Copy TSS descriptor into GDT at index 5 (16 bytes)
    __builtin_memcpy(&gdt[5], &tss_desc, 16);

    gdtr_t gdtr = {
        .limit = sizeof(gdt) - 1,
        .base  = (uint64_t)gdt
    };
    __asm__ volatile("lgdt %0" : : "m"(gdtr));

    // Reload segments
    __asm__ volatile(
        "pushq $0x08\n"
        "leaq 1f(%%rip), %%rax\n"
        "pushq %%rax\n"
        "lretq\n"
        "1:\n"
        "movw $0x10, %%ax\n"
        "movw %%ax, %%ds\n"
        "movw %%ax, %%es\n"
        "movw %%ax, %%ss\n"
        "xorw %%ax, %%ax\n"
        "movw %%ax, %%fs\n"
        "movw %%ax, %%gs\n"
        : : : "rax", "memory"
    );

    // Load TSS (selector 0x28 = index 5, RPL=0)
    __asm__ volatile("ltr %0" : : "r"((uint16_t)0x28));
}

void gdt_set_kernel_stack(uint64_t rsp0) {
    kernel_tss.rsp[0] = rsp0;
}
```

---

## ส่วนที่ 5: IDT + Exception Handlers

```c
// arch/x86_64/idt.c
#include "idt.h"
#include <stdint.h>

typedef struct {
    uint16_t offset_low;
    uint16_t selector;
    uint8_t  ist;           // IST index (0 = none)
    uint8_t  type_attr;     // 0x8E = interrupt gate, DPL=0
    uint16_t offset_mid;
    uint32_t offset_high;
    uint32_t zero;
} __attribute__((packed)) idt_entry_t;

typedef struct {
    uint16_t limit;
    uint64_t base;
} __attribute__((packed)) idtr_t;

#define IDT_SIZE 256
static idt_entry_t idt[IDT_SIZE];

// Interrupt frame pushed by CPU + our stub
typedef struct {
    uint64_t r15, r14, r13, r12, r11, r10, r9, r8;
    uint64_t rbp, rdi, rsi, rdx, rcx, rbx, rax;
    uint64_t vector;        // injected by stub
    uint64_t error_code;    // 0 if not applicable
    uint64_t rip, cs, rflags, rsp, ss;
} interrupt_frame_t;

static void idt_set_gate(int vec, void (*handler)(void),
                          uint8_t ist, uint8_t type_attr) {
    uint64_t addr = (uint64_t)handler;
    idt[vec].offset_low  = addr & 0xFFFF;
    idt[vec].selector    = 0x08;    // kernel code
    idt[vec].ist         = ist;
    idt[vec].type_attr   = type_attr;
    idt[vec].offset_mid  = (addr >> 16) & 0xFFFF;
    idt[vec].offset_high = (addr >> 32) & 0xFFFFFFFF;
    idt[vec].zero        = 0;
}

// Exception names
static const char *exception_names[] = {
    "Division Error",       "Debug",              "NMI",
    "Breakpoint",           "Overflow",           "Bound Range",
    "Invalid Opcode",       "Device Not Available","Double Fault",
    "Coprocessor Segment",  "Invalid TSS",        "Segment Not Present",
    "Stack Fault",          "General Protection", "Page Fault",
    "(reserved)",           "x87 FPU Error",      "Alignment Check",
    "Machine Check",        "SIMD FP Error",      "Virtualization",
    "Control Protection",
};

// Common C handler for exceptions
void exception_handler(interrupt_frame_t *frame) {
    int vec = (int)frame->vector;
    const char *name = (vec < 22) ? exception_names[vec] : "Unknown";

    printk("\n*** KERNEL EXCEPTION #%d: %s ***\n", vec, name);
    printk("  RIP=%016lx  CS=%04lx  RFLAGS=%016lx\n",
           frame->rip, frame->cs, frame->rflags);
    printk("  RSP=%016lx  SS=%04lx  ERR=%016lx\n",
           frame->rsp, frame->ss, frame->error_code);
    printk("  RAX=%016lx  RBX=%016lx  RCX=%016lx\n",
           frame->rax, frame->rbx, frame->rcx);
    printk("  RDX=%016lx  RSI=%016lx  RDI=%016lx\n",
           frame->rdx, frame->rsi, frame->rdi);

    if (vec == 14) {  // Page Fault
        uint64_t cr2;
        __asm__ volatile("mov %%cr2, %0" : "=r"(cr2));
        printk("  CR2 (fault addr)=%016lx\n", cr2);
        printk("  Fault: %s %s %s\n",
               (frame->error_code & 1) ? "protection" : "not-present",
               (frame->error_code & 2) ? "write"      : "read",
               (frame->error_code & 4) ? "user"       : "kernel");
    }

    panic("Unhandled exception");
}

// IRQ handler table
typedef void (*irq_handler_t)(interrupt_frame_t *);
static irq_handler_t irq_handlers[224];  // vectors 32–255

void irq_register(int vec, irq_handler_t handler) {
    if (vec >= 32 && vec < 256)
        irq_handlers[vec - 32] = handler;
}

void irq_dispatch(interrupt_frame_t *frame) {
    int vec = (int)frame->vector;
    if (vec >= 32 && irq_handlers[vec - 32])
        irq_handlers[vec - 32](frame);

    // Send EOI to APIC
    apic_eoi();
}
```

```nasm
; arch/x86_64/isr_stubs.asm
; Auto-generate 256 interrupt stubs

bits 64
extern exception_handler
extern irq_dispatch

%macro ISR_NOERR 1
isr_%1:
    push qword 0        ; fake error code
    push qword %1       ; vector number
    jmp isr_common
%endmacro

%macro ISR_ERR 1
isr_%1:
    push qword %1       ; vector number (error code already pushed by CPU)
    jmp isr_common
%endmacro

; Exceptions with no error code
ISR_NOERR 0
ISR_NOERR 1
ISR_NOERR 2
ISR_NOERR 3
ISR_NOERR 4
ISR_NOERR 5
ISR_NOERR 6
ISR_NOERR 7
ISR_ERR   8
ISR_NOERR 9
ISR_ERR   10
ISR_ERR   11
ISR_ERR   12
ISR_ERR   13
ISR_ERR   14
ISR_NOERR 15
ISR_NOERR 16
ISR_ERR   17
ISR_NOERR 18
ISR_NOERR 19
ISR_NOERR 20
ISR_ERR   21

; Vectors 22–31: reserved
%assign i 22
%rep 10
    ISR_NOERR i
    %assign i i+1
%endrep

; Vectors 32–255: hardware IRQs
%assign i 32
%rep 224
    ISR_NOERR i
    %assign i i+1
%endrep

isr_common:
    ; Save all GPRs
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    push rbp
    push r8
    push r9
    push r10
    push r11
    push r12
    push r13
    push r14
    push r15

    mov rdi, rsp        ; pointer to interrupt_frame_t
    mov rax, [rsp + 15*8]   ; vector
    cmp rax, 32
    jae .irq
    call exception_handler
    jmp .done
.irq:
    call irq_dispatch
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop r11
    pop r10
    pop r9
    pop r8
    pop rbp
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    add rsp, 16     ; pop vector + error_code
    iretq
```

---

## ส่วนที่ 6: APIC และ IOAPIC

```c
// arch/x86_64/apic.c
#include "apic.h"
#include "msr.h"
#include <stdint.h>

#define APIC_BASE_MSR       0x1B
#define APIC_ENABLE         (1 << 11)
#define APIC_X2APIC         (1 << 10)

// Local APIC MMIO registers (xAPIC mode)
#define LAPIC_ID            0x020
#define LAPIC_VER           0x030
#define LAPIC_TPR           0x080
#define LAPIC_EOI           0x0B0
#define LAPIC_SPURIOUS      0x0F0
#define LAPIC_ICR_LOW       0x300  // Interrupt Command Register
#define LAPIC_ICR_HIGH      0x310
#define LAPIC_TIMER         0x320
#define LAPIC_TIMER_INIT    0x380
#define LAPIC_TIMER_CURR    0x390
#define LAPIC_TIMER_DIV     0x3E0

#define LAPIC_SPURIOUS_ENABLE (1 << 8)
#define LAPIC_TIMER_PERIODIC  (1 << 17)
#define LAPIC_TIMER_VECTOR    0x20   ; IRQ 0 → vector 32

static volatile uint32_t *lapic_base;

static inline uint32_t lapic_read(uint32_t reg) {
    return lapic_base[reg / 4];
}

static inline void lapic_write(uint32_t reg, uint32_t val) {
    lapic_base[reg / 4] = val;
}

void apic_init(uint64_t phys_base) {
    // Map LAPIC MMIO to virtual address
    lapic_base = (volatile uint32_t *)vmm_map_mmio(phys_base, 0x1000);

    // Enable APIC via MSR
    uint64_t msr = rdmsr(APIC_BASE_MSR);
    msr |= APIC_ENABLE;
    wrmsr(APIC_BASE_MSR, msr);

    // Enable spurious interrupt (vector 0xFF)
    lapic_write(LAPIC_SPURIOUS, 0xFF | LAPIC_SPURIOUS_ENABLE);

    // Set task priority to 0 (accept all interrupts)
    lapic_write(LAPIC_TPR, 0);

    // Configure timer: periodic, divide by 16
    lapic_write(LAPIC_TIMER_DIV, 0x3);          // divide by 16
    lapic_write(LAPIC_TIMER, LAPIC_TIMER_VECTOR | LAPIC_TIMER_PERIODIC);
    lapic_write(LAPIC_TIMER_INIT, 1000000);      // calibrate later

    printk("LAPIC: ID=%d, version=%d\n",
           lapic_read(LAPIC_ID) >> 24,
           lapic_read(LAPIC_VER) & 0xFF);
}

void apic_eoi(void) {
    lapic_write(LAPIC_EOI, 0);
}

// Send IPI (Inter-Processor Interrupt)
void apic_send_ipi(uint8_t apic_id, uint8_t vector, uint32_t delivery_mode) {
    lapic_write(LAPIC_ICR_HIGH, (uint32_t)apic_id << 24);
    lapic_write(LAPIC_ICR_LOW,  vector | (delivery_mode << 8) | (1 << 14));
    // Wait for delivery
    while (lapic_read(LAPIC_ICR_LOW) & (1 << 12));
}

// INIT IPI for AP startup
void apic_send_init(uint8_t apic_id) {
    apic_send_ipi(apic_id, 0, 0x5);   // INIT delivery
    // 10ms delay
    hpet_sleep_ms(10);
    // De-assert INIT
    lapic_write(LAPIC_ICR_HIGH, (uint32_t)apic_id << 24);
    lapic_write(LAPIC_ICR_LOW, 0 | (0x5 << 8) | (0 << 14));
}

// SIPI (Startup IPI) — vector = startup page >> 12
void apic_send_sipi(uint8_t apic_id, uint8_t startup_page) {
    apic_send_ipi(apic_id, startup_page, 0x6);  // SIPI delivery
}
```

```c
// arch/x86_64/ioapic.c — IOAPIC driver
#include "ioapic.h"
#include <stdint.h>

#define IOAPIC_REGSEL   0x00
#define IOAPIC_IOWIN    0x10
#define IOAPIC_ID       0x00
#define IOAPIC_VER      0x01
#define IOAPIC_REDTBL   0x10  // 0x10 + 2*n for entry n

typedef struct {
    uint64_t phys_base;
    volatile uint32_t *virt_base;
    uint32_t gsi_base;
    uint32_t num_entries;
} ioapic_t;

#define MAX_IOAPICS 4
static ioapic_t ioapics[MAX_IOAPICS];
static int num_ioapics = 0;

static uint32_t ioapic_read(ioapic_t *io, uint8_t reg) {
    io->virt_base[IOAPIC_REGSEL / 4] = reg;
    return io->virt_base[IOAPIC_IOWIN / 4];
}

static void ioapic_write(ioapic_t *io, uint8_t reg, uint32_t val) {
    io->virt_base[IOAPIC_REGSEL / 4] = reg;
    io->virt_base[IOAPIC_IOWIN / 4]  = val;
}

void ioapic_register(uint64_t phys, uint32_t gsi_base) {
    if (num_ioapics >= MAX_IOAPICS) return;
    ioapic_t *io = &ioapics[num_ioapics++];
    io->phys_base  = phys;
    io->gsi_base   = gsi_base;
    io->virt_base  = (volatile uint32_t *)vmm_map_mmio(phys, 0x1000);
    uint32_t ver   = ioapic_read(io, IOAPIC_VER);
    io->num_entries = ((ver >> 16) & 0xFF) + 1;
    printk("IOAPIC: base=%#lx gsi=%d entries=%d\n",
           phys, gsi_base, io->num_entries);
}

// Route IRQ: gsi → vector, destination APIC, edge/level, active high/low
void ioapic_route(uint32_t gsi, uint8_t vector, uint8_t dest_apic,
                   int level, int active_low) {
    for (int i = 0; i < num_ioapics; i++) {
        ioapic_t *io = &ioapics[i];
        if (gsi >= io->gsi_base && gsi < io->gsi_base + io->num_entries) {
            uint32_t idx = gsi - io->gsi_base;
            uint64_t entry = vector;
            entry |= (uint64_t)dest_apic << 56;
            if (level)      entry |= (1 << 15);  // level triggered
            if (active_low) entry |= (1 << 13);  // active low
            ioapic_write(io, IOAPIC_REDTBL + 2*idx,     entry & 0xFFFFFFFF);
            ioapic_write(io, IOAPIC_REDTBL + 2*idx + 1, entry >> 32);
            return;
        }
    }
    printk("IOAPIC: no IOAPIC for GSI %d\n", gsi);
}

void ioapic_mask(uint32_t gsi) {
    // Set bit 16 (mask bit) in redirection entry
    for (int i = 0; i < num_ioapics; i++) {
        ioapic_t *io = &ioapics[i];
        if (gsi >= io->gsi_base && gsi < io->gsi_base + io->num_entries) {
            uint32_t idx = gsi - io->gsi_base;
            uint32_t lo = ioapic_read(io, IOAPIC_REDTBL + 2*idx);
            ioapic_write(io, IOAPIC_REDTBL + 2*idx, lo | (1 << 16));
            return;
        }
    }
}
```

---

## ส่วนที่ 7: ACPI Parsing (RSDP → RSDT/XSDT → MADT)

```c
// acpi/acpi.c — ACPI table parser
#include "acpi.h"
#include <stdint.h>
#include <string.h>

// RSDP v2
typedef struct {
    char     signature[8];   // "RSD PTR "
    uint8_t  checksum;
    char     oem_id[6];
    uint8_t  revision;       // 2 for ACPI 2.0+
    uint32_t rsdt_address;
    // ACPI 2.0+
    uint32_t length;
    uint64_t xsdt_address;
    uint8_t  ext_checksum;
    uint8_t  reserved[3];
} __attribute__((packed)) rsdp_t;

// Generic ACPI table header (SDT header)
typedef struct {
    char     signature[4];
    uint32_t length;
    uint8_t  revision;
    uint8_t  checksum;
    char     oem_id[6];
    char     oem_table_id[8];
    uint32_t oem_revision;
    uint32_t creator_id;
    uint32_t creator_revision;
} __attribute__((packed)) acpi_header_t;

static rsdp_t      *rsdp;
static acpi_header_t *xsdt;

// Search for RSDP in EBDA and BIOS ROM area
static rsdp_t *find_rsdp(void) {
    // Search in EBDA (1st KB)
    uint32_t ebda = *(uint16_t *)phys_to_virt(0x40E) << 4;
    for (uint32_t addr = ebda; addr < ebda + 1024; addr += 16) {
        rsdp_t *r = (rsdp_t *)phys_to_virt(addr);
        if (memcmp(r->signature, "RSD PTR ", 8) == 0) return r;
    }
    // Search in BIOS ROM 0xE0000–0xFFFFF
    for (uint32_t addr = 0xE0000; addr < 0xFFFFF; addr += 16) {
        rsdp_t *r = (rsdp_t *)phys_to_virt(addr);
        if (memcmp(r->signature, "RSD PTR ", 8) == 0) return r;
    }
    return NULL;
}

static int verify_checksum(void *table, size_t len) {
    uint8_t sum = 0;
    for (size_t i = 0; i < len; i++)
        sum += ((uint8_t *)table)[i];
    return sum == 0;
}

void acpi_init(void) {
    rsdp = find_rsdp();
    if (!rsdp) panic("ACPI: RSDP not found");

    if (!verify_checksum(rsdp, sizeof(rsdp_t)))
        panic("ACPI: RSDP checksum invalid");

    // Prefer XSDT (64-bit pointers)
    if (rsdp->revision >= 2 && rsdp->xsdt_address) {
        xsdt = (acpi_header_t *)phys_to_virt(rsdp->xsdt_address);
        if (memcmp(xsdt->signature, "XSDT", 4) != 0)
            panic("ACPI: XSDT signature invalid");
        printk("ACPI: XSDT at %#lx, %d bytes\n",
               rsdp->xsdt_address, xsdt->length);
    } else {
        panic("ACPI: need XSDT (ACPI 2.0+)");
    }
}

// Find an ACPI table by 4-byte signature
acpi_header_t *acpi_find_table(const char *sig) {
    uint64_t *entries = (uint64_t *)((uint8_t *)xsdt + sizeof(acpi_header_t));
    int count = (xsdt->length - sizeof(acpi_header_t)) / 8;

    for (int i = 0; i < count; i++) {
        acpi_header_t *hdr = (acpi_header_t *)phys_to_virt(entries[i]);
        if (memcmp(hdr->signature, sig, 4) == 0) {
            if (!verify_checksum(hdr, hdr->length)) {
                printk("ACPI: table %.4s has bad checksum\n", sig);
                continue;
            }
            return hdr;
        }
    }
    return NULL;
}
```

```c
// acpi/madt.c — MADT parser (Multiple APIC Description Table)
#include "madt.h"
#include "acpi.h"
#include <stdint.h>
#include <string.h>

typedef struct {
    acpi_header_t header;
    uint32_t      lapic_address;
    uint32_t      flags;
} __attribute__((packed)) madt_header_t;

// MADT entry types
#define MADT_LAPIC       0
#define MADT_IOAPIC      1
#define MADT_INT_OVERRIDE 2
#define MADT_LAPIC_ADDR  5

typedef struct {
    uint8_t type, length;
    uint8_t acpi_id, apic_id;
    uint32_t flags;     // bit 0 = enabled, bit 1 = online capable
} __attribute__((packed)) madt_lapic_t;

typedef struct {
    uint8_t type, length;
    uint8_t ioapic_id, reserved;
    uint32_t ioapic_addr;
    uint32_t gsi_base;
} __attribute__((packed)) madt_ioapic_t;

typedef struct {
    uint8_t type, length;
    uint8_t bus, source;      // bus=0 (ISA), source = ISA IRQ
    uint32_t gsi;
    uint16_t flags;
} __attribute__((packed)) madt_int_override_t;

// SMP topology discovered from MADT
uint8_t  cpu_apic_ids[256];
int      num_cpus = 0;
uint32_t bsp_lapic_addr;

void madt_parse(void) {
    madt_header_t *madt = (madt_header_t *)acpi_find_table("APIC");
    if (!madt) panic("MADT not found — SMP unavailable");

    bsp_lapic_addr = madt->lapic_address;
    printk("MADT: LAPIC base=%#x flags=%#x\n",
           madt->lapic_address, madt->flags);

    uint8_t *ptr = (uint8_t *)madt + sizeof(madt_header_t);
    uint8_t *end = (uint8_t *)madt + madt->header.length;

    while (ptr < end) {
        uint8_t type = ptr[0], len = ptr[1];

        switch (type) {
        case MADT_LAPIC: {
            madt_lapic_t *l = (madt_lapic_t *)ptr;
            if (l->flags & 1) {  // processor enabled
                printk("  CPU %d: ACPI ID=%d APIC ID=%d\n",
                       num_cpus, l->acpi_id, l->apic_id);
                cpu_apic_ids[num_cpus++] = l->apic_id;
            }
            break;
        }
        case MADT_IOAPIC: {
            madt_ioapic_t *io = (madt_ioapic_t *)ptr;
            printk("  IOAPIC: ID=%d addr=%#x GSI base=%d\n",
                   io->ioapic_id, io->ioapic_addr, io->gsi_base);
            ioapic_register(io->ioapic_addr, io->gsi_base);
            break;
        }
        case MADT_INT_OVERRIDE: {
            madt_int_override_t *ov = (madt_int_override_t *)ptr;
            printk("  INT Override: ISA IRQ %d → GSI %d flags=%#x\n",
                   ov->source, ov->gsi, ov->flags);
            // Store for IRQ routing
            irq_override_register(ov->source, ov->gsi, ov->flags);
            break;
        }
        }
        ptr += len;
    }
    printk("MADT: found %d CPUs\n", num_cpus);
}
```

---

## ส่วนที่ 8: Physical Memory Manager (PMM)

```c
// mm/pmm.c — Bitmap-based Physical Memory Manager
#include "pmm.h"
#include <stdint.h>
#include <string.h>

#define PAGE_SIZE   4096
#define PAGES_MAX   (16UL * 1024 * 1024 * 1024 / PAGE_SIZE)  // support up to 16GB

static uint64_t pmm_bitmap[PAGES_MAX / 64];  // 1 bit per page
static uint64_t pmm_total_pages = 0;
static uint64_t pmm_free_pages  = 0;
static spinlock_t pmm_lock;

static inline void pmm_set(uint64_t page) {
    pmm_bitmap[page / 64] |=  (1ULL << (page % 64));
}
static inline void pmm_clear(uint64_t page) {
    pmm_bitmap[page / 64] &= ~(1ULL << (page % 64));
}
static inline int pmm_test(uint64_t page) {
    return (pmm_bitmap[page / 64] >> (page % 64)) & 1;
}

// Initialize PMM from multiboot2 memory map
void pmm_init(mb2_mmap_t *mmap, uint64_t mmap_len) {
    spin_init(&pmm_lock);

    // Mark everything as used initially
    memset(pmm_bitmap, 0xFF, sizeof(pmm_bitmap));

    mb2_mmap_entry_t *entry = mmap->entries;
    uint64_t end = (uint64_t)mmap + mmap_len;

    while ((uint64_t)entry < end) {
        if (entry->type == MB2_MMAP_AVAILABLE) {
            uint64_t start_page = ALIGN_UP(entry->base, PAGE_SIZE) / PAGE_SIZE;
            uint64_t end_page   = (entry->base + entry->length) / PAGE_SIZE;

            for (uint64_t p = start_page; p < end_page; p++) {
                if (p < PAGES_MAX) {
                    pmm_clear(p);
                    pmm_free_pages++;
                    pmm_total_pages++;
                }
            }
            printk("PMM: free region %#lx–%#lx (%lu MB)\n",
                   entry->base, entry->base + entry->length,
                   entry->length >> 20);
        }
        entry = (mb2_mmap_entry_t *)((uint8_t *)entry + mmap->entry_size);
    }

    // Re-mark kernel itself as used
    extern char _kernel_start[], _kernel_end[];
    uint64_t ks = (uint64_t)_kernel_start - KERNEL_VIRT_BASE;
    uint64_t ke = (uint64_t)_kernel_end   - KERNEL_VIRT_BASE;
    uint64_t kp_start = ks / PAGE_SIZE;
    uint64_t kp_end   = ALIGN_UP(ke, PAGE_SIZE) / PAGE_SIZE;
    for (uint64_t p = kp_start; p < kp_end; p++) {
        if (!pmm_test(p)) {
            pmm_set(p);
            pmm_free_pages--;
        }
    }

    printk("PMM: total=%lu MB, free=%lu MB\n",
           pmm_total_pages * PAGE_SIZE >> 20,
           pmm_free_pages  * PAGE_SIZE >> 20);
}

// Allocate one physical page (returns physical address)
uint64_t pmm_alloc(void) {
    spin_lock(&pmm_lock);
    for (uint64_t i = 0; i < PAGES_MAX / 64; i++) {
        if (pmm_bitmap[i] != 0xFFFFFFFFFFFFFFFFULL) {
            int bit = __builtin_ctzll(~pmm_bitmap[i]);
            uint64_t page = i * 64 + bit;
            pmm_set(page);
            pmm_free_pages--;
            spin_unlock(&pmm_lock);
            return page * PAGE_SIZE;
        }
    }
    spin_unlock(&pmm_lock);
    panic("PMM: out of memory!");
    return 0;
}

// Allocate N contiguous pages
uint64_t pmm_alloc_n(uint64_t n) {
    spin_lock(&pmm_lock);
    uint64_t run = 0, start = 0;
    for (uint64_t p = 0; p < PAGES_MAX; p++) {
        if (!pmm_test(p)) {
            if (run == 0) start = p;
            if (++run == n) {
                for (uint64_t i = start; i < start + n; i++) {
                    pmm_set(i);
                    pmm_free_pages--;
                }
                spin_unlock(&pmm_lock);
                return start * PAGE_SIZE;
            }
        } else {
            run = 0;
        }
    }
    spin_unlock(&pmm_lock);
    panic("PMM: cannot alloc %lu contiguous pages", n);
    return 0;
}

void pmm_free(uint64_t phys) {
    uint64_t page = phys / PAGE_SIZE;
    spin_lock(&pmm_lock);
    if (!pmm_test(page)) {
        spin_unlock(&pmm_lock);
        panic("PMM: double free at %#lx", phys);
    }
    pmm_clear(page);
    pmm_free_pages++;
    spin_unlock(&pmm_lock);
}

uint64_t pmm_free_count(void) { return pmm_free_pages; }
```

---

## ส่วนที่ 9: Virtual Memory Manager (4-level Paging)

```c
// mm/vmm.c — 4-level page table management
#include "vmm.h"
#include "pmm.h"
#include <stdint.h>
#include <string.h>

#define PAGE_PRESENT  (1ULL << 0)
#define PAGE_WRITE    (1ULL << 1)
#define PAGE_USER     (1ULL << 2)
#define PAGE_PWT      (1ULL << 3)
#define PAGE_PCD      (1ULL << 4)   // cache disable (for MMIO)
#define PAGE_ACCESSED (1ULL << 5)
#define PAGE_DIRTY    (1ULL << 6)
#define PAGE_HUGE     (1ULL << 7)   // 2MB pages (in PD)
#define PAGE_GLOBAL   (1ULL << 8)
#define PAGE_NX       (1ULL << 63)  // no-execute

#define PML4_INDEX(va)  (((va) >> 39) & 0x1FF)
#define PDPT_INDEX(va)  (((va) >> 30) & 0x1FF)
#define PD_INDEX(va)    (((va) >> 21) & 0x1FF)
#define PT_INDEX(va)    (((va) >> 12) & 0x1FF)
#define PAGE_ADDR(e)    ((e) & 0x000FFFFFFFFFF000ULL)

#define PHYS_TO_VIRT(p) ((p) + PHYSMAP_BASE)
#define VIRT_TO_PHYS(v) ((v) - PHYSMAP_BASE)

#define PHYSMAP_BASE    0xFFFF800000000000ULL
#define KERNEL_BASE     0xFFFFFFFF80000000ULL

// Kernel PML4 (physical address stored in CR3)
static uint64_t kernel_pml4_phys;

static uint64_t *pml4_virt(void) {
    return (uint64_t *)PHYS_TO_VIRT(kernel_pml4_phys);
}

// Get or create page table at next level
static uint64_t *get_or_create(uint64_t *table, int idx, uint64_t flags) {
    if (!(table[idx] & PAGE_PRESENT)) {
        uint64_t phys = pmm_alloc();
        memset((void *)PHYS_TO_VIRT(phys), 0, PAGE_SIZE);
        table[idx] = phys | PAGE_PRESENT | PAGE_WRITE | (flags & PAGE_USER);
    }
    return (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(table[idx]));
}

// Map a virtual page → physical page
void vmm_map(uint64_t virt, uint64_t phys, uint64_t flags) {
    uint64_t *pml4 = pml4_virt();
    uint64_t *pdpt = get_or_create(pml4, PML4_INDEX(virt), flags);
    uint64_t *pd   = get_or_create(pdpt, PDPT_INDEX(virt), flags);
    uint64_t *pt   = get_or_create(pd,   PD_INDEX(virt),   flags);
    pt[PT_INDEX(virt)] = phys | flags | PAGE_PRESENT;
    __asm__ volatile("invlpg (%0)" : : "r"(virt) : "memory");
}

// Map a range of virtual addresses
void vmm_map_range(uint64_t virt, uint64_t phys, uint64_t size, uint64_t flags) {
    for (uint64_t off = 0; off < size; off += PAGE_SIZE)
        vmm_map(virt + off, phys + off, flags);
}

// Unmap a virtual page
void vmm_unmap(uint64_t virt) {
    uint64_t *pml4 = pml4_virt();
    int l4 = PML4_INDEX(virt);
    if (!(pml4[l4] & PAGE_PRESENT)) return;
    uint64_t *pdpt = (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(pml4[l4]));
    int l3 = PDPT_INDEX(virt);
    if (!(pdpt[l3] & PAGE_PRESENT)) return;
    uint64_t *pd = (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(pdpt[l3]));
    int l2 = PD_INDEX(virt);
    if (!(pd[l2] & PAGE_PRESENT)) return;
    uint64_t *pt = (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(pd[l2]));
    pt[PT_INDEX(virt)] = 0;
    __asm__ volatile("invlpg (%0)" : : "r"(virt) : "memory");
}

// Translate virtual → physical
uint64_t vmm_translate(uint64_t virt) {
    uint64_t *pml4 = pml4_virt();
    if (!(pml4[PML4_INDEX(virt)] & PAGE_PRESENT)) return 0;
    uint64_t *pdpt = (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(pml4[PML4_INDEX(virt)]));
    if (!(pdpt[PDPT_INDEX(virt)] & PAGE_PRESENT)) return 0;
    uint64_t *pd = (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(pdpt[PDPT_INDEX(virt)]));
    if (!(pd[PD_INDEX(virt)] & PAGE_PRESENT)) return 0;
    if (pd[PD_INDEX(virt)] & PAGE_HUGE)
        return PAGE_ADDR(pd[PD_INDEX(virt)]) + (virt & 0x1FFFFF);
    uint64_t *pt = (uint64_t *)PHYS_TO_VIRT(PAGE_ADDR(pd[PD_INDEX(virt)]));
    return PAGE_ADDR(pt[PT_INDEX(virt)]) + (virt & 0xFFF);
}

// Map MMIO region (cache-disabled, write-through)
void *vmm_map_mmio(uint64_t phys, uint64_t size) {
    // Allocate virtual address from MMIO region
    static uint64_t mmio_next = 0xFFFF900000000000ULL;
    uint64_t virt = mmio_next;
    mmio_next += ALIGN_UP(size, PAGE_SIZE);
    vmm_map_range(virt, phys, size, PAGE_WRITE | PAGE_PCD | PAGE_PWT);
    return (void *)virt;
}

// Create a new address space (user process)
uint64_t vmm_create_address_space(void) {
    uint64_t new_pml4_phys = pmm_alloc();
    uint64_t *new_pml4 = (uint64_t *)PHYS_TO_VIRT(new_pml4_phys);
    memset(new_pml4, 0, PAGE_SIZE);

    // Copy kernel higher-half mappings (PML4[256..511])
    uint64_t *kpml4 = pml4_virt();
    for (int i = 256; i < 512; i++)
        new_pml4[i] = kpml4[i];

    return new_pml4_phys;
}

void vmm_switch_cr3(uint64_t pml4_phys) {
    __asm__ volatile("mov %0, %%cr3" : : "r"(pml4_phys) : "memory");
}

void vmm_init(void) {
    uint64_t cr3;
    __asm__ volatile("mov %%cr3, %0" : "=r"(cr3));
    kernel_pml4_phys = cr3;

    // Map all physical memory into physmap region
    // (in practice done during boot with 2MB pages for efficiency)
    printk("VMM: kernel PML4 at phys %#lx\n", kernel_pml4_phys);

    // Enable NX bit (EFER.NXE)
    uint64_t efer = rdmsr(0xC0000080);
    efer |= (1ULL << 11);
    wrmsr(0xC0000080, efer);
}
```

---

## ส่วนที่ 10: Slab Allocator

```c
// mm/slab.c — Per-size slab cache allocator
#include "slab.h"
#include "pmm.h"
#include <stdint.h>
#include <string.h>

// Object sizes supported: 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096
#define SLAB_NUM_CACHES 10
static const uint32_t slab_sizes[SLAB_NUM_CACHES] = {
    8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096
};

typedef struct slab_obj {
    struct slab_obj *next;  // free list link
} slab_obj_t;

typedef struct slab {
    struct slab  *next;
    uint8_t      *data;     // points to page data
    slab_obj_t   *free;     // free list head
    uint32_t      free_count;
    uint32_t      obj_size;
    uint32_t      capacity; // objects per page
} slab_t;

typedef struct slab_cache {
    uint32_t   obj_size;
    slab_t    *partial;     // slabs with some free objects
    slab_t    *full;        // all objects allocated
    slab_t    *empty;       // all objects free
    spinlock_t lock;
    uint64_t   alloc_count;
    uint64_t   free_count;
} slab_cache_t;

static slab_cache_t caches[SLAB_NUM_CACHES];

static slab_t *slab_create(uint32_t obj_size) {
    // Allocate a page for slab metadata + a page for objects
    uint64_t meta_phys = pmm_alloc();
    uint64_t data_phys = pmm_alloc();

    slab_t *slab = (slab_t *)PHYS_TO_VIRT(meta_phys);
    slab->data       = (uint8_t *)PHYS_TO_VIRT(data_phys);
    slab->obj_size   = obj_size;
    slab->capacity   = PAGE_SIZE / obj_size;
    slab->free_count = slab->capacity;
    slab->next       = NULL;

    // Initialize free list
    slab->free = NULL;
    for (int i = slab->capacity - 1; i >= 0; i--) {
        slab_obj_t *obj = (slab_obj_t *)(slab->data + i * obj_size);
        obj->next = slab->free;
        slab->free = obj;
    }
    return slab;
}

void slab_init(void) {
    for (int i = 0; i < SLAB_NUM_CACHES; i++) {
        caches[i].obj_size = slab_sizes[i];
        caches[i].partial  = NULL;
        caches[i].full     = NULL;
        caches[i].empty    = NULL;
        spin_init(&caches[i].lock);
        caches[i].alloc_count = 0;
        caches[i].free_count  = 0;
        // Pre-warm with one empty slab
        caches[i].empty = slab_create(slab_sizes[i]);
    }
    printk("Slab: initialized %d caches\n", SLAB_NUM_CACHES);
}

static slab_cache_t *find_cache(uint32_t size) {
    for (int i = 0; i < SLAB_NUM_CACHES; i++)
        if (slab_sizes[i] >= size) return &caches[i];
    return NULL;
}

void *slab_alloc(uint32_t size) {
    slab_cache_t *cache = find_cache(size);
    if (!cache) return NULL;

    spin_lock(&cache->lock);

    // Get a partial slab (or move empty → partial)
    slab_t *slab = cache->partial;
    if (!slab) {
        slab = cache->empty;
        if (!slab) {
            slab = slab_create(cache->obj_size);
        } else {
            cache->empty = slab->next;
        }
        slab->next = cache->partial;
        cache->partial = slab;
    }

    // Allocate from free list
    slab_obj_t *obj = slab->free;
    slab->free = obj->next;
    slab->free_count--;
    cache->alloc_count++;

    // Move full slab out of partial
    if (slab->free_count == 0) {
        cache->partial = slab->next;
        slab->next = cache->full;
        cache->full = slab;
    }

    spin_unlock(&cache->lock);
    return (void *)obj;
}

void slab_free(void *ptr) {
    if (!ptr) return;

    // Find which slab owns this pointer
    // (in practice, store slab ptr in page metadata)
    uint64_t page_base = (uint64_t)ptr & ~(PAGE_SIZE - 1);

    for (int i = 0; i < SLAB_NUM_CACHES; i++) {
        slab_cache_t *cache = &caches[i];
        spin_lock(&cache->lock);

        // Search partial and full lists
        slab_t **lists[] = { &cache->partial, &cache->full };
        for (int l = 0; l < 2; l++) {
            slab_t *prev = NULL;
            for (slab_t *s = *lists[l]; s; prev = s, s = s->next) {
                if ((uint64_t)s->data == page_base) {
                    slab_obj_t *obj = (slab_obj_t *)ptr;
                    obj->next = s->free;
                    s->free = obj;
                    s->free_count++;
                    cache->free_count++;

                    // Move from full → partial
                    if (l == 1 && s->free_count == 1) {
                        if (prev) prev->next = s->next;
                        else cache->full = s->next;
                        s->next = cache->partial;
                        cache->partial = s;
                    }
                    // Move from partial → empty
                    if (s->free_count == s->capacity && l == 0) {
                        if (prev) prev->next = s->next;
                        else cache->partial = s->next;
                        s->next = cache->empty;
                        cache->empty = s;
                    }
                    spin_unlock(&cache->lock);
                    return;
                }
            }
        }
        spin_unlock(&cache->lock);
    }
    panic("slab_free: pointer %p not in any slab", ptr);
}
```

```c
// mm/kmalloc.c — General purpose allocator
#include "kmalloc.h"
#include "slab.h"
#include "pmm.h"
#include <stdint.h>

void *kmalloc(size_t size) {
    if (size == 0) return NULL;
    if (size <= 4096) {
        return slab_alloc((uint32_t)size);
    }
    // Large allocation: use PMM directly
    uint64_t pages = (size + PAGE_SIZE - 1) / PAGE_SIZE;
    uint64_t phys = pmm_alloc_n(pages);
    return (void *)PHYS_TO_VIRT(phys);
}

void *kzalloc(size_t size) {
    void *p = kmalloc(size);
    if (p) memset(p, 0, size);
    return p;
}

void kfree(void *ptr) {
    if (!ptr) return;
    // Determine if it came from slab or PMM
    // (heuristic: slab objects are within PAGE boundaries of slab data)
    slab_free(ptr);
}

void *krealloc(void *ptr, size_t new_size) {
    if (!ptr) return kmalloc(new_size);
    void *new_ptr = kmalloc(new_size);
    if (new_ptr) {
        // Copy old data (we don't track old size precisely here)
        memcpy(new_ptr, ptr, new_size);
        kfree(ptr);
    }
    return new_ptr;
}
```

---

## ส่วนที่ 11: SMP — Wake Application Processors

```c
// arch/x86_64/smp.c — Symmetric Multiprocessing startup
#include "smp.h"
#include "apic.h"
#include "gdt.h"
#include "per_cpu.h"
#include <stdint.h>
#include <string.h>

// Trampoline: AP real-mode startup code placed at physical page 0x8000
extern uint8_t ap_trampoline_start[];
extern uint8_t ap_trampoline_end[];
#define AP_TRAMPOLINE_PHYS  0x8000

// Per-CPU data structure
typedef struct {
    uint64_t  self;         // pointer to this struct (GS:0)
    uint64_t  kernel_rsp;   // kernel stack for this CPU
    uint64_t  user_rsp;     // saved user RSP on syscall
    int       cpu_id;
    int       apic_id;
    uint64_t  ticks;        // local timer ticks
    process_t *current;     // currently running process
    spinlock_t run_lock;
} percpu_t;

static percpu_t percpu_data[256] __attribute__((aligned(64)));
static uint8_t  ap_stacks[256][65536] __attribute__((aligned(16)));
volatile int    cpus_online = 1;  // BSP is CPU 0

// Called by each AP after entering 64-bit mode
void ap_entry(int cpu_id) {
    // Load GDT + IDT (already set up by BSP)
    gdt_init();
    idt_load();

    // Set up per-CPU GS base
    percpu_t *cpu = &percpu_data[cpu_id];
    cpu->self    = (uint64_t)cpu;
    cpu->cpu_id  = cpu_id;
    cpu->apic_id = cpu_apic_ids[cpu_id];
    wrmsr(0xC0000101, (uint64_t)cpu);   // GS.base MSR

    // Initialize local APIC for this CPU
    apic_init(bsp_lapic_addr);

    // Enable interrupts
    __asm__ volatile("sti");

    printk("CPU %d online (APIC ID %d)\n", cpu_id, cpu->apic_id);
    __atomic_fetch_add(&cpus_online, 1, __ATOMIC_SEQ_CST);

    // Enter scheduler idle loop
    scheduler_idle_loop();
}

void smp_init(void) {
    // Set up BSP per-CPU data
    percpu_data[0].self    = (uint64_t)&percpu_data[0];
    percpu_data[0].cpu_id  = 0;
    percpu_data[0].apic_id = cpu_apic_ids[0];
    wrmsr(0xC0000101, (uint64_t)&percpu_data[0]);

    if (num_cpus <= 1) {
        printk("SMP: single CPU, no APs to wake\n");
        return;
    }

    // Copy AP trampoline to low memory (< 1MB, real mode accessible)
    size_t tramp_size = ap_trampoline_end - ap_trampoline_start;
    memcpy((void *)PHYS_TO_VIRT(AP_TRAMPOLINE_PHYS),
           ap_trampoline_start, tramp_size);

    // Store pointers AP needs (at known offsets in trampoline page)
    // These are written by convention: see ap_trampoline.asm
    uint64_t *ap_stack_ptr  = (uint64_t *)PHYS_TO_VIRT(AP_TRAMPOLINE_PHYS + 0xFF0);
    uint64_t *ap_cpu_id_ptr = (uint64_t *)PHYS_TO_VIRT(AP_TRAMPOLINE_PHYS + 0xFF8);

    // Start each AP
    for (int i = 1; i < num_cpus; i++) {
        *ap_stack_ptr  = (uint64_t)&ap_stacks[i][65536];  // stack top
        *ap_cpu_id_ptr = (uint64_t)i;

        uint8_t apic_id = cpu_apic_ids[i];
        printk("SMP: starting CPU %d (APIC ID %d)...\n", i, apic_id);

        // INIT IPI
        apic_send_init(apic_id);
        // Two SIPIs (as per spec)
        apic_send_sipi(apic_id, AP_TRAMPOLINE_PHYS >> 12);
        hpet_sleep_ms(1);
        apic_send_sipi(apic_id, AP_TRAMPOLINE_PHYS >> 12);

        // Wait up to 100ms
        for (int t = 0; t < 100; t++) {
            if (cpus_online > i) break;
            hpet_sleep_ms(1);
        }
        if (cpus_online <= i)
            printk("SMP: CPU %d did not respond!\n", i);
    }
    printk("SMP: %d/%d CPUs online\n", cpus_online, num_cpus);
}

// Get current CPU's per-CPU data (via GS segment base)
percpu_t *get_percpu(void) {
    percpu_t *cpu;
    __asm__ volatile("mov %%gs:0, %0" : "=r"(cpu));
    return cpu;
}
```

```nasm
; arch/x86_64/ap_trampoline.asm — AP startup (real mode → protected → long mode)
bits 16
org 0x8000

ap_trampoline_start:
    cli
    xor ax, ax
    mov ds, ax

    ; Load temporary GDT (32-bit flat)
    lgdt [ap_gdt32_ptr - 0x8000 + 0x8000]  ; physical offset

    ; Enable protected mode
    mov eax, cr0
    or  eax, 1
    mov cr0, eax
    jmp 0x08:ap_pm32 - 0x8000 + 0x8000

bits 32
ap_pm32:
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov ss, ax

    ; Load 64-bit GDT (same as BSP)
    lgdt [gdt64_ptr]        ; absolute virtual address
    ; Enable PAE
    mov eax, cr4
    or  eax, (1 << 5)
    mov cr4, eax
    ; Load kernel CR3
    mov eax, [kernel_cr3]
    mov cr3, eax
    ; Enable LME
    mov ecx, 0xC0000080
    rdmsr
    or  eax, (1 << 8)
    wrmsr
    ; Enable paging
    mov eax, cr0
    or  eax, (1 << 31)
    mov cr0, eax
    jmp 0x08:ap_lm64

bits 64
ap_lm64:
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov ss, ax

    ; Load stack from trampoline data
    mov rsp, [rel ap_stack_ptr]  ; written by smp_init
    mov rdi, [rel ap_cpu_id]
    extern ap_entry
    call ap_entry
    hlt

align 8
ap_gdt32:
    dq 0
    dq 0x00CF9A000000FFFF   ; 32-bit code
    dq 0x00CF92000000FFFF   ; 32-bit data
ap_gdt32_ptr:
    dw 23
    dd ap_gdt32 - 0x8000 + 0x8000

; Written by smp_init at offsets 0xFF0, 0xFF8
times (0xFF0 - ($ - ap_trampoline_start)) db 0
ap_stack_ptr: dq 0
ap_cpu_id:    dq 0

ap_trampoline_end:
```

---

## ส่วนที่ 12: Spinlock (SMP-safe)

```c
// arch/x86_64/spinlock.h + spinlock.c
typedef struct {
    volatile int locked;
} spinlock_t;

static inline void spin_init(spinlock_t *l) { l->locked = 0; }

static inline void spin_lock(spinlock_t *l) {
    while (__atomic_test_and_set(&l->locked, __ATOMIC_ACQUIRE))
        __asm__ volatile("pause");  // x86 hint: reduce power/contention
}

static inline void spin_unlock(spinlock_t *l) {
    __atomic_clear(&l->locked, __ATOMIC_RELEASE);
}

static inline int spin_trylock(spinlock_t *l) {
    return !__atomic_test_and_set(&l->locked, __ATOMIC_ACQUIRE);
}
```

---

## ส่วนที่ 13: Process Management

```c
// proc/process.c — Process Control Block and lifecycle
#include "process.h"
#include "vmm.h"
#include "kmalloc.h"
#include "scheduler.h"
#include <stdint.h>
#include <string.h>

#define MAX_PROCESSES   1024
#define KSTACK_SIZE     65536   // 64KB kernel stack per process

typedef enum {
    PROC_RUNNING,
    PROC_READY,
    PROC_BLOCKED,
    PROC_ZOMBIE,
    PROC_DEAD
} proc_state_t;

// 64-bit saved register context (for context switch)
typedef struct {
    uint64_t r15, r14, r13, r12, r11, r10, r9, r8;
    uint64_t rbp, rdi, rsi, rdx, rcx, rbx, rax;
    uint64_t rip;   // return address for switch_context
    uint64_t rflags;
} cpu_context_t;

typedef struct process {
    uint32_t      pid;
    uint32_t      ppid;
    proc_state_t  state;
    int           exit_code;
    char          name[64];

    // Memory management
    uint64_t      pml4_phys;    // address space
    uint64_t      brk;          // heap break

    // Kernel stack
    uint64_t      kstack_top;
    uint64_t      kstack_phys;

    // CPU context (saved on context switch)
    cpu_context_t ctx;

    // Scheduling
    uint64_t      timeslice;    // ticks remaining
    uint64_t      total_ticks;  // total CPU time

    // File descriptors
    vfile_t      *fds[256];

    // Linked list for scheduler queue
    struct process *next;
    struct process *prev;
} process_t;

static process_t *process_table[MAX_PROCESSES];
static uint32_t   next_pid = 1;
static spinlock_t ptable_lock;

process_t *process_create(const char *name, uint64_t entry,
                           uint64_t pml4, int is_kernel) {
    process_t *proc = kzalloc(sizeof(process_t));
    if (!proc) return NULL;

    spin_lock(&ptable_lock);
    proc->pid  = next_pid++;
    proc->ppid = 0;
    spin_unlock(&ptable_lock);

    strncpy(proc->name, name, 63);
    proc->state     = PROC_READY;
    proc->timeslice = 10;   // 10ms default
    proc->pml4_phys = pml4 ? pml4 : kernel_pml4_phys;

    // Allocate kernel stack
    uint64_t kstack_phys = pmm_alloc_n(KSTACK_SIZE / PAGE_SIZE);
    proc->kstack_phys = kstack_phys;
    proc->kstack_top  = PHYS_TO_VIRT(kstack_phys) + KSTACK_SIZE;

    // Set up initial context (stack grows down)
    uint64_t *ksp = (uint64_t *)proc->kstack_top;
    *--ksp = 0;             // return address (never used)
    proc->ctx.rip = entry;
    proc->ctx.rflags = 0x202;   // IF=1

    // Add to process table
    spin_lock(&ptable_lock);
    for (int i = 0; i < MAX_PROCESSES; i++) {
        if (!process_table[i]) {
            process_table[i] = proc;
            break;
        }
    }
    spin_unlock(&ptable_lock);

    // Add to scheduler ready queue
    scheduler_enqueue(proc);

    printk("process: created '%s' PID=%d\n", name, proc->pid);
    return proc;
}

void process_exit(process_t *proc, int code) {
    proc->state     = PROC_ZOMBIE;
    proc->exit_code = code;
    // Wake parent if waiting
    scheduler_yield();
}

process_t *process_find(uint32_t pid) {
    spin_lock(&ptable_lock);
    for (int i = 0; i < MAX_PROCESSES; i++) {
        if (process_table[i] && process_table[i]->pid == pid) {
            spin_unlock(&ptable_lock);
            return process_table[i];
        }
    }
    spin_unlock(&ptable_lock);
    return NULL;
}
```

```nasm
; proc/switch_context.asm — Context switch
bits 64
global switch_context

; switch_context(cpu_context_t *old, cpu_context_t *new)
; rdi = old context ptr, rsi = new context ptr
switch_context:
    ; Save callee-saved registers to old context
    mov [rdi + 0*8],  r15
    mov [rdi + 1*8],  r14
    mov [rdi + 2*8],  r13
    mov [rdi + 3*8],  r12
    mov [rdi + 4*8],  rbx
    mov [rdi + 5*8],  rbp
    ; Save RIP (return address is on stack)
    mov rax, [rsp]
    mov [rdi + 6*8], rax

    ; Load new context
    mov r15, [rsi + 0*8]
    mov r14, [rsi + 1*8]
    mov r13, [rsi + 2*8]
    mov r12, [rsi + 3*8]
    mov rbx, [rsi + 4*8]
    mov rbp, [rsi + 5*8]
    ; Jump to new RIP
    mov rax, [rsi + 6*8]
    mov [rsp], rax
    ret
```

---

## ส่วนที่ 14: ELF64 Loader

```c
// proc/elf64.c — ELF64 binary loader
#include "elf64.h"
#include "vmm.h"
#include "pmm.h"
#include "kmalloc.h"
#include <stdint.h>
#include <string.h>

#define ET_EXEC  2
#define ET_DYN   3
#define PT_LOAD  1
#define PT_INTERP 3
#define PT_PHDR  6

#define ELF_MAGIC  0x464C457F  // "\x7FELF"

typedef struct {
    uint32_t e_magic;
    uint8_t  e_class;       // 2 = 64-bit
    uint8_t  e_data;        // 1 = LE
    uint8_t  e_version;
    uint8_t  e_osabi;
    uint64_t e_pad;
    uint16_t e_type;
    uint16_t e_machine;     // 0x3E = x86-64
    uint32_t e_version2;
    uint64_t e_entry;
    uint64_t e_phoff;       // program header offset
    uint64_t e_shoff;
    uint32_t e_flags;
    uint16_t e_ehsize;
    uint16_t e_phentsize;
    uint16_t e_phnum;
    uint16_t e_shentsize;
    uint16_t e_shnum;
    uint16_t e_shstrndx;
} __attribute__((packed)) elf64_ehdr_t;

typedef struct {
    uint32_t p_type;
    uint32_t p_flags;       // PF_R=4, PF_W=2, PF_X=1
    uint64_t p_offset;      // offset in file
    uint64_t p_vaddr;       // virtual address
    uint64_t p_paddr;
    uint64_t p_filesz;      // bytes in file
    uint64_t p_memsz;       // bytes in memory (>= filesz, BSS)
    uint64_t p_align;
} __attribute__((packed)) elf64_phdr_t;

// Load ELF into address space, return entry point
// data = ELF file contents in kernel memory
// pml4_phys = target address space (0 = create new)
uint64_t elf64_load(const uint8_t *data, size_t size, uint64_t *out_pml4) {
    elf64_ehdr_t *ehdr = (elf64_ehdr_t *)data;

    // Validate ELF
    if (ehdr->e_magic != ELF_MAGIC) {
        printk("ELF: bad magic\n");
        return 0;
    }
    if (ehdr->e_class != 2) {
        printk("ELF: not 64-bit\n");
        return 0;
    }
    if (ehdr->e_machine != 0x3E) {
        printk("ELF: not x86-64\n");
        return 0;
    }

    // Create new address space
    uint64_t pml4 = vmm_create_address_space();
    *out_pml4 = pml4;

    // Save current CR3, switch to new space
    uint64_t old_cr3;
    __asm__ volatile("mov %%cr3, %0" : "=r"(old_cr3));
    vmm_switch_cr3(pml4);

    // Map each PT_LOAD segment
    elf64_phdr_t *phdrs = (elf64_phdr_t *)(data + ehdr->e_phoff);
    for (int i = 0; i < ehdr->e_phnum; i++) {
        elf64_phdr_t *ph = &phdrs[i];
        if (ph->p_type != PT_LOAD) continue;

        uint64_t flags = PAGE_USER;
        if (ph->p_flags & 2) flags |= PAGE_WRITE;
        if (!(ph->p_flags & 1)) flags |= PAGE_NX;

        uint64_t vaddr     = ph->p_vaddr & ~(PAGE_SIZE-1);
        uint64_t vaddr_end = ALIGN_UP(ph->p_vaddr + ph->p_memsz, PAGE_SIZE);

        // Allocate and map physical pages
        for (uint64_t va = vaddr; va < vaddr_end; va += PAGE_SIZE) {
            uint64_t pa = pmm_alloc();
            memset((void *)PHYS_TO_VIRT(pa), 0, PAGE_SIZE);
            vmm_map(va, pa, flags);
        }

        // Copy segment data
        if (ph->p_filesz > 0) {
            memcpy((void *)ph->p_vaddr,
                   data + ph->p_offset,
                   ph->p_filesz);
        }
        // BSS (p_memsz > p_filesz) already zeroed above

        printk("ELF: load segment vaddr=%#lx size=%lu flags=%#x\n",
               ph->p_vaddr, ph->p_memsz, ph->p_flags);
    }

    // Map user stack
    uint64_t stack_top  = 0x00007FFFFFFFE000ULL;
    uint64_t stack_size = 8 * PAGE_SIZE;  // 32KB
    for (uint64_t va = stack_top - stack_size; va < stack_top; va += PAGE_SIZE) {
        uint64_t pa = pmm_alloc();
        memset((void *)PHYS_TO_VIRT(pa), 0, PAGE_SIZE);
        vmm_map(va, pa, PAGE_USER | PAGE_WRITE | PAGE_NX);
    }

    // Restore kernel address space
    vmm_switch_cr3(old_cr3);

    printk("ELF: entry=%#lx\n", ehdr->e_entry);
    return ehdr->e_entry;
}
```

---

## ส่วนที่ 15: System Calls (SYSCALL/SYSRET)

```c
// arch/x86_64/syscall.c — Fast system call interface
#include "syscall.h"
#include "msr.h"
#include "gdt.h"
#include <stdint.h>

// SYSCALL instruction uses these MSRs:
//   STAR   (0xC0000081): CS/SS selectors for kernel/user
//   LSTAR  (0xC0000082): kernel RIP for SYSCALL handler
//   SFMASK (0xC0000084): RFLAGS bits to clear on entry

#define MSR_STAR    0xC0000081
#define MSR_LSTAR   0xC0000082
#define MSR_SFMASK  0xC0000084
#define MSR_EFER    0xC0000080

extern void syscall_entry(void);   // asm stub

// Syscall numbers
#define SYS_READ    0
#define SYS_WRITE   1
#define SYS_OPEN    2
#define SYS_CLOSE   3
#define SYS_EXIT    60
#define SYS_FORK    57
#define SYS_EXEC    59
#define SYS_GETPID  39
#define SYS_MMAP    9
#define SYS_MUNMAP  11
#define SYS_SLEEP   35
#define SYS_MAX     512

typedef uint64_t (*syscall_fn_t)(uint64_t, uint64_t, uint64_t,
                                  uint64_t, uint64_t, uint64_t);
static syscall_fn_t syscall_table[SYS_MAX];

void syscall_register(int num, syscall_fn_t fn) {
    if (num >= 0 && num < SYS_MAX)
        syscall_table[num] = fn;
}

// Called from syscall_entry.asm
uint64_t syscall_dispatch(uint64_t num, uint64_t a1, uint64_t a2,
                           uint64_t a3, uint64_t a4, uint64_t a5) {
    if (num >= SYS_MAX || !syscall_table[num]) {
        printk("syscall: unknown syscall %lu\n", num);
        return (uint64_t)-38;   // -ENOSYS
    }
    return syscall_table[num](a1, a2, a3, a4, a5, 0);
}

void syscall_init(void) {
    // Enable SYSCALL in EFER
    uint64_t efer = rdmsr(MSR_EFER);
    efer |= 1;   // SCE bit
    wrmsr(MSR_EFER, efer);

    // STAR: kernel CS=0x08, user CS=0x1B (ring 3), SS=0x13
    // STAR[63:48] = user CS-8 | user SS, STAR[47:32] = kernel CS
    uint64_t star = ((uint64_t)0x1B << 48) | ((uint64_t)0x08 << 32);
    wrmsr(MSR_STAR, star);

    // LSTAR: syscall entry point
    wrmsr(MSR_LSTAR, (uint64_t)syscall_entry);

    // SFMASK: clear IF on syscall entry (we re-enable after saving state)
    wrmsr(MSR_SFMASK, (1 << 9));   // clear IF

    // Register system calls
    syscall_register(SYS_READ,   sys_read);
    syscall_register(SYS_WRITE,  sys_write);
    syscall_register(SYS_OPEN,   sys_open);
    syscall_register(SYS_CLOSE,  sys_close);
    syscall_register(SYS_EXIT,   sys_exit);
    syscall_register(SYS_FORK,   sys_fork);
    syscall_register(SYS_EXEC,   sys_execve);
    syscall_register(SYS_GETPID, sys_getpid);
    syscall_register(SYS_MMAP,   sys_mmap);
    syscall_register(SYS_SLEEP,  sys_sleep);

    printk("syscall: SYSCALL/SYSRET enabled\n");
}
```

```nasm
; arch/x86_64/syscall_entry.asm — SYSCALL entry stub
bits 64
extern syscall_dispatch

global syscall_entry
syscall_entry:
    ; On entry: RCX=user RIP, R11=user RFLAGS
    ; RSP is still USER stack — must swap to kernel stack
    swapgs
    ; Save user RSP, load kernel RSP from per-CPU
    mov [gs:16], rsp        ; percpu->user_rsp
    mov rsp, [gs:8]         ; percpu->kernel_rsp

    ; Build syscall frame
    push qword 0x1B         ; user SS
    push qword [gs:16]      ; user RSP
    push r11                ; user RFLAGS
    push qword 0x23         ; user CS
    push rcx                ; user RIP

    ; Save scratch registers
    push rax
    push rdi
    push rsi
    push rdx
    push r10
    push r8
    push r9

    ; syscall_dispatch(num, arg1, arg2, arg3, arg4, arg5)
    ; Linux ABI: rax=num, rdi,rsi,rdx,r10,r8,r9
    mov rdi, rax    ; syscall number
    ; args already in rsi, rdx, r10, r8, r9 — adjust to C ABI
    mov rax, rsi
    mov rsi, rdx
    mov rdx, r10
    mov rcx, r8
    mov r8,  r9
    mov r9,  0
    ; rdi=num, rax=arg1, rsi=arg2, rdx=arg3, rcx=arg4, r8=arg5
    ; C call: rdi, rsi, rdx, rcx, r8, r9
    mov rsi, rax
    call syscall_dispatch    ; rax = return value

    ; Restore
    pop r9
    pop r8
    pop r10
    pop rdx
    pop rsi
    pop rdi
    add rsp, 8      ; skip saved rax
    pop rcx         ; user RIP → rcx
    add rsp, 8      ; skip user CS
    pop r11         ; user RFLAGS
    pop qword [gs:16]   ; restore user RSP save slot
    mov rsp, [gs:16]    ; back to user stack

    swapgs
    o64 sysret
```

---

## ส่วนที่ 16: VFS Layer

```c
// vfs/vfs.c — Virtual File System
#include "vfs.h"
#include "kmalloc.h"
#include <string.h>
#include <stdint.h>

// File type flags
#define VFS_FILE    1
#define VFS_DIR     2
#define VFS_LINK    3
#define VFS_BLOCK   4
#define VFS_CHAR    5

// vnode operations table
typedef struct vops {
    int  (*open)   (vnode_t *, int flags);
    int  (*close)  (vnode_t *);
    int  (*read)   (vnode_t *, void *buf, size_t len, off_t off);
    int  (*write)  (vnode_t *, const void *buf, size_t len, off_t off);
    int  (*readdir)(vnode_t *, vdirent_t *out, int idx);
    int  (*lookup) (vnode_t *dir, const char *name, vnode_t **out);
    int  (*create) (vnode_t *dir, const char *name, int mode);
    int  (*unlink) (vnode_t *dir, const char *name);
    int  (*stat)   (vnode_t *, vstat_t *);
} vops_t;

struct vnode {
    uint32_t  ino;
    uint32_t  type;
    uint32_t  mode;
    uint32_t  nlinks;
    uint64_t  size;
    uint64_t  atime, mtime, ctime;
    void     *fs_data;  // filesystem-specific data
    vops_t   *ops;
    int       refcount;
    spinlock_t lock;
};

// Open file description
typedef struct {
    vnode_t  *vnode;
    off_t     offset;
    int       flags;
    int       refcount;
    spinlock_t lock;
} vfile_t;

// Mount entry
typedef struct mount {
    char        path[256];
    vnode_t    *root;
    struct mount *next;
} mount_t;

static mount_t  *mount_list = NULL;
static spinlock_t vfs_lock;

// Mount a filesystem at path
int vfs_mount(const char *path, vnode_t *root) {
    mount_t *m = kzalloc(sizeof(mount_t));
    if (!m) return -1;
    strncpy(m->path, path, 255);
    m->root = root;

    spin_lock(&vfs_lock);
    m->next    = mount_list;
    mount_list = m;
    spin_unlock(&vfs_lock);

    printk("VFS: mounted at '%s'\n", path);
    return 0;
}

// Lookup a path, returning its vnode
int vfs_lookup(const char *path, vnode_t **out) {
    if (!path || path[0] != '/') return -1;

    // Find longest-matching mount
    mount_t *best = NULL;
    size_t   best_len = 0;
    spin_lock(&vfs_lock);
    for (mount_t *m = mount_list; m; m = m->next) {
        size_t mlen = strlen(m->path);
        if (strncmp(path, m->path, mlen) == 0 && mlen > best_len) {
            best = m;
            best_len = mlen;
        }
    }
    spin_unlock(&vfs_lock);

    if (!best) return -2;   // no mount point

    vnode_t *node = best->root;
    const char *rel = path + best_len;
    if (*rel == '/') rel++;

    // Walk path components
    char component[256];
    while (*rel) {
        const char *slash = strchr(rel, '/');
        size_t len = slash ? (size_t)(slash - rel) : strlen(rel);
        if (len == 0 || (len == 1 && rel[0] == '.')) {
            rel += len + (slash ? 1 : 0);
            continue;
        }
        memcpy(component, rel, len);
        component[len] = 0;

        if (!node->ops || !node->ops->lookup)
            return -3;

        vnode_t *child;
        int r = node->ops->lookup(node, component, &child);
        if (r) return r;
        node = child;
        rel += len + (slash ? 1 : 0);
    }
    *out = node;
    return 0;
}

// Open a file
vfile_t *vfs_open(const char *path, int flags) {
    vnode_t *node;
    int r = vfs_lookup(path, &node);
    if (r) return NULL;

    if (node->ops && node->ops->open) {
        r = node->ops->open(node, flags);
        if (r) return NULL;
    }

    vfile_t *f = kzalloc(sizeof(vfile_t));
    f->vnode    = node;
    f->flags    = flags;
    f->refcount = 1;
    spin_init(&f->lock);
    node->refcount++;
    return f;
}

int vfs_read(vfile_t *f, void *buf, size_t len) {
    if (!f || !f->vnode || !f->vnode->ops->read) return -1;
    int n = f->vnode->ops->read(f->vnode, buf, len, f->offset);
    if (n > 0) f->offset += n;
    return n;
}

int vfs_write(vfile_t *f, const void *buf, size_t len) {
    if (!f || !f->vnode || !f->vnode->ops->write) return -1;
    int n = f->vnode->ops->write(f->vnode, buf, len, f->offset);
    if (n > 0) f->offset += n;
    return n;
}

void vfs_close(vfile_t *f) {
    if (!f) return;
    if (f->vnode->ops && f->vnode->ops->close)
        f->vnode->ops->close(f->vnode);
    f->vnode->refcount--;
    kfree(f);
}
```

---

## ส่วนที่ 17: Ext2 Filesystem Driver

```c
// fs/ext2/ext2.c — Ext2 filesystem driver
#include "ext2.h"
#include "vfs.h"
#include "kmalloc.h"
#include <string.h>
#include <stdint.h>

// Ext2 superblock (at byte offset 1024 on disk)
typedef struct {
    uint32_t s_inodes_count;
    uint32_t s_blocks_count;
    uint32_t s_r_blocks_count;
    uint32_t s_free_blocks_count;
    uint32_t s_free_inodes_count;
    uint32_t s_first_data_block;    // 0 for 4K blocks, 1 for 1K blocks
    uint32_t s_log_block_size;      // block_size = 1024 << s_log_block_size
    uint32_t s_log_frag_size;
    uint32_t s_blocks_per_group;
    uint32_t s_frags_per_group;
    uint32_t s_inodes_per_group;
    uint32_t s_mtime, s_wtime;
    uint16_t s_mnt_count, s_max_mnt_count;
    uint16_t s_magic;               // 0xEF53
    uint16_t s_state;
    uint16_t s_errors;
    uint16_t s_minor_rev_level;
    uint32_t s_lastcheck, s_checkinterval;
    uint32_t s_creator_os;
    uint32_t s_rev_level;           // 1 = dynamic rev
    uint16_t s_def_resuid, s_def_resgid;
    // Dynamic revision fields:
    uint32_t s_first_ino;
    uint16_t s_inode_size;
    // ... more fields
} __attribute__((packed)) ext2_super_t;

// Block group descriptor
typedef struct {
    uint32_t bg_block_bitmap;
    uint32_t bg_inode_bitmap;
    uint32_t bg_inode_table;
    uint16_t bg_free_blocks_count;
    uint16_t bg_free_inodes_count;
    uint16_t bg_used_dirs_count;
    uint16_t bg_pad;
    uint32_t bg_reserved[3];
} __attribute__((packed)) ext2_bgd_t;

// Inode
typedef struct {
    uint16_t i_mode;
    uint16_t i_uid;
    uint32_t i_size;
    uint32_t i_atime, i_ctime, i_mtime, i_dtime;
    uint16_t i_gid;
    uint16_t i_links_count;
    uint32_t i_blocks;      // 512-byte blocks
    uint32_t i_flags;
    uint32_t i_osd1;
    uint32_t i_block[15];   // 12 direct + 1 indirect + 1 dbl + 1 tpl
    uint32_t i_generation;
    uint32_t i_file_acl;
    uint32_t i_dir_acl;     // high 32 bits of size for regular files
    uint32_t i_faddr;
    uint8_t  i_osd2[12];
} __attribute__((packed)) ext2_inode_t;

// Directory entry
typedef struct {
    uint32_t inode;
    uint16_t rec_len;
    uint8_t  name_len;
    uint8_t  file_type;     // 0=unknown,1=file,2=dir,3=chardev,4=blkdev,5=fifo,6=sock,7=link
    char     name[];
} __attribute__((packed)) ext2_dirent_t;

#define EXT2_MAGIC  0xEF53

// Per-filesystem state
typedef struct {
    ext2_super_t  *sb;
    ext2_bgd_t    *bgdt;       // block group descriptor table
    uint32_t       block_size;
    uint32_t       inodes_per_group;
    uint32_t       num_groups;
    uint32_t       inode_size;
    blkdev_t      *dev;        // block device
} ext2_fs_t;

// Read a block from the device
static void *ext2_read_block(ext2_fs_t *fs, uint32_t block) {
    void *buf = kmalloc(fs->block_size);
    uint64_t lba = (uint64_t)block * (fs->block_size / 512);
    blkdev_read(fs->dev, lba, fs->block_size / 512, buf);
    return buf;
}

// Read an inode by number
static int ext2_read_inode(ext2_fs_t *fs, uint32_t ino, ext2_inode_t *out) {
    uint32_t group = (ino - 1) / fs->inodes_per_group;
    uint32_t index = (ino - 1) % fs->inodes_per_group;
    ext2_bgd_t *bgd = &fs->bgdt[group];

    uint32_t inode_table_block = bgd->bg_inode_table;
    uint32_t block_off = (index * fs->inode_size) / fs->block_size;
    uint32_t byte_off  = (index * fs->inode_size) % fs->block_size;

    uint8_t *block = ext2_read_block(fs, inode_table_block + block_off);
    memcpy(out, block + byte_off, sizeof(ext2_inode_t));
    kfree(block);
    return 0;
}

// Get block number for file position (handles indirect blocks)
static uint32_t ext2_get_block(ext2_fs_t *fs, ext2_inode_t *ino, uint32_t block_idx) {
    uint32_t ptrs_per_block = fs->block_size / 4;

    if (block_idx < 12) {
        return ino->i_block[block_idx];
    }
    block_idx -= 12;

    if (block_idx < ptrs_per_block) {
        // Single indirect
        uint32_t *indirect = ext2_read_block(fs, ino->i_block[12]);
        uint32_t result = indirect[block_idx];
        kfree(indirect);
        return result;
    }
    block_idx -= ptrs_per_block;

    if (block_idx < ptrs_per_block * ptrs_per_block) {
        // Double indirect
        uint32_t l1 = block_idx / ptrs_per_block;
        uint32_t l2 = block_idx % ptrs_per_block;
        uint32_t *dbl = ext2_read_block(fs, ino->i_block[13]);
        uint32_t *sng = ext2_read_block(fs, dbl[l1]);
        uint32_t result = sng[l2];
        kfree(dbl); kfree(sng);
        return result;
    }

    // Triple indirect (large files)
    block_idx -= ptrs_per_block * ptrs_per_block;
    uint32_t l1 = block_idx / (ptrs_per_block * ptrs_per_block);
    uint32_t l2 = (block_idx / ptrs_per_block) % ptrs_per_block;
    uint32_t l3 = block_idx % ptrs_per_block;
    uint32_t *t = ext2_read_block(fs, ino->i_block[14]);
    uint32_t *d = ext2_read_block(fs, t[l1]);
    uint32_t *s = ext2_read_block(fs, d[l2]);
    uint32_t result = s[l3];
    kfree(t); kfree(d); kfree(s);
    return result;
}

// VFS read operation for ext2 file
static int ext2_vfs_read(vnode_t *vnode, void *buf, size_t len, off_t off) {
    ext2_fs_t    *fs   = vnode->fs_data;
    ext2_inode_t *ino  = /* ... lookup from vnode->ino */ NULL;
    // (simplified — real code keeps inode cached in vnode)
    size_t remaining = len;
    size_t done = 0;
    while (remaining > 0 && (uint64_t)(off + done) < ino->i_size) {
        uint32_t block_idx = (off + done) / fs->block_size;
        uint32_t block_off = (off + done) % fs->block_size;
        uint32_t to_copy   = fs->block_size - block_off;
        if (to_copy > remaining) to_copy = remaining;
        if ((uint64_t)(off + done + to_copy) > ino->i_size)
            to_copy = ino->i_size - (off + done);

        uint32_t blk = ext2_get_block(fs, ino, block_idx);
        uint8_t *blkbuf = ext2_read_block(fs, blk);
        memcpy((uint8_t *)buf + done, blkbuf + block_off, to_copy);
        kfree(blkbuf);
        done += to_copy;
        remaining -= to_copy;
    }
    return (int)done;
}

// Lookup a name in a directory
static int ext2_vfs_lookup(vnode_t *dir, const char *name, vnode_t **out) {
    ext2_fs_t    *fs  = dir->fs_data;
    ext2_inode_t  ino;
    ext2_read_inode(fs, dir->ino, &ino);

    uint32_t total_size = ino.i_size;
    uint32_t pos = 0;

    while (pos < total_size) {
        uint32_t blk_idx = pos / fs->block_size;
        uint32_t blk_num = ext2_get_block(fs, &ino, blk_idx);
        uint8_t *blk = ext2_read_block(fs, blk_num);

        uint32_t blk_pos = pos % fs->block_size;
        while (blk_pos < fs->block_size) {
            ext2_dirent_t *de = (ext2_dirent_t *)(blk + blk_pos);
            if (de->inode && de->name_len == strlen(name) &&
                strncmp(de->name, name, de->name_len) == 0) {
                // Found! Create vnode for this entry
                vnode_t *v = kzalloc(sizeof(vnode_t));
                v->ino     = de->inode;
                v->fs_data = fs;
                v->ops     = &ext2_vops;
                *out = v;
                kfree(blk);
                return 0;
            }
            if (de->rec_len == 0) break;
            blk_pos += de->rec_len;
            pos     += de->rec_len;
        }
        kfree(blk);
    }
    return -2;  // ENOENT
}

// Initialize ext2 on a block device
vnode_t *ext2_mount(blkdev_t *dev) {
    ext2_fs_t *fs = kzalloc(sizeof(ext2_fs_t));
    fs->dev = dev;

    // Read superblock (at byte 1024 = LBA 2 for 512-byte sectors)
    uint8_t sb_buf[1024];
    blkdev_read(dev, 2, 2, sb_buf);
    ext2_super_t *sb = (ext2_super_t *)sb_buf;

    if (sb->s_magic != EXT2_MAGIC) {
        printk("ext2: bad magic %#x\n", sb->s_magic);
        kfree(fs);
        return NULL;
    }

    fs->sb = kmalloc(sizeof(ext2_super_t));
    memcpy(fs->sb, sb, sizeof(ext2_super_t));
    fs->block_size       = 1024 << sb->s_log_block_size;
    fs->inodes_per_group = sb->s_inodes_per_group;
    fs->inode_size       = (sb->s_rev_level >= 1) ? sb->s_inode_size : 128;
    fs->num_groups       = (sb->s_blocks_count + sb->s_blocks_per_group - 1)
                           / sb->s_blocks_per_group;

    printk("ext2: block_size=%d, %d groups, %d inodes\n",
           fs->block_size, fs->num_groups, sb->s_inodes_count);

    // Read block group descriptor table
    uint32_t bgdt_block = (fs->block_size == 1024) ? 2 : 1;
    fs->bgdt = kmalloc(fs->num_groups * sizeof(ext2_bgd_t));
    uint8_t *bgdt_buf = ext2_read_block(fs, bgdt_block);
    memcpy(fs->bgdt, bgdt_buf, fs->num_groups * sizeof(ext2_bgd_t));
    kfree(bgdt_buf);

    // Create root vnode (inode 2)
    vnode_t *root = kzalloc(sizeof(vnode_t));
    root->ino     = 2;
    root->type    = VFS_DIR;
    root->fs_data = fs;
    root->ops     = &ext2_vops;

    ext2_inode_t root_ino;
    ext2_read_inode(fs, 2, &root_ino);
    root->size = root_ino.i_size;

    return root;
}
```

---

## ส่วนที่ 18: AHCI (SATA) Driver

```c
// drivers/ahci.c — AHCI SATA controller driver
#include "ahci.h"
#include "pci.h"
#include "vmm.h"
#include "kmalloc.h"
#include <string.h>
#include <stdint.h>

// AHCI Generic Host Control registers
#define AHCI_GHC_HR     (1 << 0)   // HBA Reset
#define AHCI_GHC_IE     (1 << 1)   // Interrupt Enable
#define AHCI_GHC_AE     (1 << 31)  // AHCI Enable

// Port registers (offset from port base)
#define PORT_CLB        0x00   // Command List Base Address
#define PORT_CLBU       0x04
#define PORT_FB         0x08   // FIS Base Address
#define PORT_FBU        0x0C
#define PORT_IS         0x10   // Interrupt Status
#define PORT_IE         0x14   // Interrupt Enable
#define PORT_CMD        0x18   // Command and Status
#define PORT_TFD        0x20   // Task File Data
#define PORT_SIG        0x24   // Signature
#define PORT_SSTS       0x28   // Serial ATA Status
#define PORT_SERR       0x30   // Serial ATA Error
#define PORT_CI         0x38   // Command Issue

#define PORT_CMD_ST     (1 << 0)   // Start
#define PORT_CMD_FRE    (1 << 4)   // FIS Receive Enable
#define PORT_CMD_FR     (1 << 14)  // FIS Receive Running
#define PORT_CMD_CR     (1 << 15)  // Command List Running

#define SATA_SIG_ATA    0x00000101
#define SATA_SIG_ATAPI  0xEB140101

typedef struct {
    uint8_t  cfl : 5;      // command FIS length (dwords)
    uint8_t  a   : 1;      // ATAPI
    uint8_t  w   : 1;      // write
    uint8_t  p   : 1;      // prefetchable
    uint8_t  r   : 1;      // reset
    uint8_t  b   : 1;      // BIST
    uint8_t  c   : 1;      // clear busy upon R_OK
    uint8_t  rsvd0 : 1;
    uint16_t prdtl;        // PRDT length (entries)
    uint32_t prdbc;        // PRDT bytes transferred
    uint64_t ctba;         // Command Table Base Address
    uint32_t rsvd1[4];
} __attribute__((packed)) ahci_cmd_header_t;

typedef struct {
    uint64_t dba;           // Data Base Address
    uint32_t rsvd;
    uint32_t dbc : 22;      // Byte count (0-based)
    uint32_t rsvd2 : 9;
    uint32_t i : 1;         // Interrupt on completion
} __attribute__((packed)) ahci_prdt_t;

typedef struct {
    uint8_t     cfis[64];   // Command FIS
    uint8_t     acmd[16];   // ATAPI command
    uint8_t     rsvd[48];
    ahci_prdt_t prdt[];     // Physical Region Descriptor Table
} __attribute__((packed)) ahci_cmd_table_t;

typedef struct {
    volatile uint32_t *regs;    // HBA memory registers
    volatile uint32_t *port;    // port registers
    ahci_cmd_header_t *cmd_list;
    ahci_cmd_table_t  *cmd_table;
    uint8_t           *fis_buf;
    int                port_num;
    uint64_t           lba_count;
} ahci_port_t;

static ahci_port_t ahci_ports[32];
static int ahci_port_count = 0;

static volatile uint32_t *ahci_base;

static void ahci_port_start(ahci_port_t *p) {
    // Wait for command list to be idle
    while (p->port[PORT_CMD / 4] & PORT_CMD_CR);
    p->port[PORT_CMD / 4] |= PORT_CMD_FRE | PORT_CMD_ST;
}

static void ahci_port_stop(ahci_port_t *p) {
    p->port[PORT_CMD / 4] &= ~PORT_CMD_ST;
    while (p->port[PORT_CMD / 4] & PORT_CMD_CR);
    p->port[PORT_CMD / 4] &= ~PORT_CMD_FRE;
    while (p->port[PORT_CMD / 4] & PORT_CMD_FR);
}

// Set up H2D Register FIS for ATA command
static void build_h2d_fis(uint8_t *fis, uint8_t cmd, uint64_t lba, uint16_t count) {
    memset(fis, 0, 20);
    fis[0] = 0x27;          // H2D FIS type
    fis[1] = 0x80;          // C bit = command
    fis[2] = cmd;           // ATA command
    fis[3] = 0;             // features
    fis[4] = lba & 0xFF;
    fis[5] = (lba >> 8)  & 0xFF;
    fis[6] = (lba >> 16) & 0xFF;
    fis[7] = 0x40;          // LBA mode
    fis[8] = (lba >> 24) & 0xFF;
    fis[9] = (lba >> 32) & 0xFF;
    fis[10]= (lba >> 40) & 0xFF;
    fis[12]= count & 0xFF;
    fis[13]= (count >> 8) & 0xFF;
}

// Read sectors from AHCI port
int ahci_read(int port_idx, uint64_t lba, uint32_t count, void *buf) {
    ahci_port_t *p = &ahci_ports[port_idx];

    // Use command slot 0
    ahci_cmd_header_t *hdr = &p->cmd_list[0];
    ahci_cmd_table_t  *tbl = p->cmd_table;

    memset(hdr, 0, sizeof(*hdr));
    hdr->cfl   = 5;         // H2D FIS = 5 dwords
    hdr->w     = 0;         // read
    hdr->prdtl = 1;
    hdr->ctba  = (uint64_t)VIRT_TO_PHYS((uint64_t)tbl);

    memset(tbl, 0, sizeof(ahci_cmd_table_t) + sizeof(ahci_prdt_t));

    // Build ATA READ DMA EXT command
    build_h2d_fis(tbl->cfis, 0x25, lba, count);

    // Set up PRDT
    tbl->prdt[0].dba  = (uint64_t)VIRT_TO_PHYS((uint64_t)buf);
    tbl->prdt[0].dbc  = count * 512 - 1;
    tbl->prdt[0].i    = 1;

    // Issue command on slot 0
    p->port[PORT_CI / 4] = 1;

    // Wait for completion
    for (int timeout = 100000; timeout > 0; timeout--) {
        if (!(p->port[PORT_CI / 4] & 1)) return 0;
        if (p->port[PORT_TFD / 4] & 0x01) {  // error
            printk("AHCI: read error at LBA %lu\n", lba);
            return -1;
        }
    }
    printk("AHCI: read timeout\n");
    return -1;
}

void ahci_init(void) {
    // Find AHCI controller via PCI (class 0x01, subclass 0x06, progif 0x01)
    pci_device_t *dev = pci_find_class(0x010601);
    if (!dev) {
        printk("AHCI: no controller found\n");
        return;
    }

    // Map AHCI MMIO (BAR5)
    uint64_t bar5 = pci_read_bar(dev, 5) & ~0xFFF;
    ahci_base = vmm_map_mmio(bar5, 0x2000);

    // Enable AHCI mode
    ahci_base[2] |= AHCI_GHC_AE;

    uint32_t pi = ahci_base[3];  // Ports Implemented
    printk("AHCI: ports implemented = %#x\n", pi);

    for (int i = 0; i < 32; i++) {
        if (!(pi & (1 << i))) continue;

        volatile uint32_t *port = ahci_base + (0x100 + i * 0x80) / 4;

        // Check if device present
        uint32_t ssts = port[PORT_SSTS / 4];
        if ((ssts & 0xF) != 3) continue;   // not active
        if ((ssts >> 8 & 0xF) != 1) continue;  // not active IPM

        uint32_t sig = port[PORT_SIG / 4];
        if (sig != SATA_SIG_ATA) continue;  // skip ATAPI

        // Allocate DMA buffers
        ahci_port_t *p = &ahci_ports[ahci_port_count];
        p->port     = port;
        p->port_num = i;

        uint64_t cl_phys = pmm_alloc();
        uint64_t fb_phys = pmm_alloc();
        uint64_t ct_phys = pmm_alloc();

        p->cmd_list  = (ahci_cmd_header_t *)PHYS_TO_VIRT(cl_phys);
        p->fis_buf   = (uint8_t *)PHYS_TO_VIRT(fb_phys);
        p->cmd_table = (ahci_cmd_table_t *)PHYS_TO_VIRT(ct_phys);

        memset(p->cmd_list,  0, PAGE_SIZE);
        memset(p->fis_buf,   0, PAGE_SIZE);
        memset(p->cmd_table, 0, PAGE_SIZE);

        ahci_port_stop(p);

        port[PORT_CLB / 4]  = cl_phys & 0xFFFFFFFF;
        port[PORT_CLBU / 4] = cl_phys >> 32;
        port[PORT_FB / 4]   = fb_phys & 0xFFFFFFFF;
        port[PORT_FBU / 4]  = fb_phys >> 32;
        port[PORT_SERR / 4] = 0xFFFFFFFF;
        port[PORT_IS / 4]   = 0xFFFFFFFF;

        ahci_port_start(p);

        printk("AHCI: port %d ready (ATA disk)\n", i);
        ahci_port_count++;
    }
}
```

---

## ส่วนที่ 19: Network Stack (Ethernet → IPv4 → TCP)

```c
// net/ethernet.c — Ethernet frame handling
#include "net.h"
#include "kmalloc.h"
#include <string.h>
#include <stdint.h>

typedef struct {
    uint8_t  dst[6];
    uint8_t  src[6];
    uint16_t ethertype;
    uint8_t  payload[];
} __attribute__((packed)) eth_frame_t;

#define ETH_IPV4   0x0800
#define ETH_ARP    0x0806
#define ETH_IPV6   0x86DD

static uint8_t my_mac[6] = {0x52, 0x54, 0x00, 0x12, 0x34, 0x56};

void eth_rx(uint8_t *data, size_t len) {
    if (len < 14) return;
    eth_frame_t *frame = (eth_frame_t *)data;
    uint16_t ethertype = __builtin_bswap16(frame->ethertype);

    switch (ethertype) {
    case ETH_ARP:
        arp_rx(frame->payload, len - 14);
        break;
    case ETH_IPV4:
        ipv4_rx(frame->payload, len - 14, frame->src);
        break;
    default:
        // Unknown ethertype, drop
        break;
    }
}

void eth_tx(uint8_t *dst_mac, uint16_t ethertype,
            const void *payload, size_t payload_len) {
    size_t total = 14 + payload_len;
    uint8_t *buf = kmalloc(total);
    eth_frame_t *frame = (eth_frame_t *)buf;
    memcpy(frame->dst, dst_mac, 6);
    memcpy(frame->src, my_mac,  6);
    frame->ethertype = __builtin_bswap16(ethertype);
    memcpy(frame->payload, payload, payload_len);
    e1000_tx(buf, total);   // send via NIC driver
    kfree(buf);
}
```

```c
// net/ipv4.c — IPv4 packet handling
#include "net.h"
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  version_ihl;   // [7:4]=version=4, [3:0]=IHL
    uint8_t  dscp_ecn;
    uint16_t total_length;
    uint16_t identification;
    uint16_t flags_fragment;
    uint8_t  ttl;
    uint8_t  protocol;      // 6=TCP, 17=UDP, 1=ICMP
    uint16_t checksum;
    uint32_t src_ip;
    uint32_t dst_ip;
    uint8_t  payload[];
} __attribute__((packed)) ipv4_hdr_t;

#define PROTO_ICMP  1
#define PROTO_TCP   6
#define PROTO_UDP   17

static uint32_t my_ip   = 0xC0A80101;  // 192.168.1.1
static uint32_t gw_ip   = 0xC0A80101;
static uint32_t netmask = 0xFFFFFF00;

static uint16_t ip_checksum(const void *data, size_t len) {
    const uint16_t *ptr = data;
    uint32_t sum = 0;
    while (len > 1) { sum += *ptr++; len -= 2; }
    if (len) sum += *(uint8_t *)ptr;
    while (sum >> 16) sum = (sum & 0xFFFF) + (sum >> 16);
    return ~sum;
}

static uint16_t ip_id = 1;

void ipv4_rx(uint8_t *data, size_t len, uint8_t *src_mac) {
    if (len < 20) return;
    ipv4_hdr_t *hdr = (ipv4_hdr_t *)data;
    uint8_t  ihl    = (hdr->version_ihl & 0xF) * 4;
    uint16_t total  = __builtin_bswap16(hdr->total_length);
    if (total > len) return;

    // Verify checksum
    if (ip_checksum(hdr, ihl) != 0) return;

    switch (hdr->protocol) {
    case PROTO_TCP:
        tcp_rx(hdr->payload, total - ihl, hdr->src_ip, hdr->dst_ip);
        break;
    case PROTO_UDP:
        udp_rx(hdr->payload, total - ihl, hdr->src_ip, hdr->dst_ip);
        break;
    case PROTO_ICMP:
        icmp_rx(hdr->payload, total - ihl, hdr->src_ip, src_mac);
        break;
    }
}

void ipv4_tx(uint32_t dst_ip, uint8_t proto,
             const void *payload, size_t payload_len) {
    size_t total = 20 + payload_len;
    uint8_t *buf = kmalloc(total);
    ipv4_hdr_t *hdr = (ipv4_hdr_t *)buf;

    hdr->version_ihl   = 0x45;
    hdr->dscp_ecn      = 0;
    hdr->total_length  = __builtin_bswap16((uint16_t)total);
    hdr->identification = __builtin_bswap16(ip_id++);
    hdr->flags_fragment = 0;
    hdr->ttl           = 64;
    hdr->protocol      = proto;
    hdr->checksum      = 0;
    hdr->src_ip        = my_ip;
    hdr->dst_ip        = dst_ip;
    memcpy(hdr->payload, payload, payload_len);
    hdr->checksum      = ip_checksum(hdr, 20);

    // Resolve dst MAC via ARP
    uint8_t dst_mac[6];
    if (arp_resolve(dst_ip, dst_mac) != 0) {
        // Queue packet, wait for ARP reply
        arp_request(dst_ip);
        kfree(buf);
        return;
    }
    eth_tx(dst_mac, ETH_IPV4, buf, total);
    kfree(buf);
}
```

```c
// net/tcp.c — Basic TCP implementation
#include "net.h"
#include "kmalloc.h"
#include <string.h>
#include <stdint.h>

typedef struct {
    uint16_t src_port, dst_port;
    uint32_t seq;
    uint32_t ack;
    uint8_t  data_off;  // [7:4] = header len in dwords
    uint8_t  flags;     // SYN=0x02, ACK=0x10, FIN=0x01, RST=0x04, PSH=0x08
    uint16_t window;
    uint16_t checksum;
    uint16_t urgent;
    uint8_t  payload[];
} __attribute__((packed)) tcp_hdr_t;

#define TCP_FIN  0x01
#define TCP_SYN  0x02
#define TCP_RST  0x04
#define TCP_PSH  0x08
#define TCP_ACK  0x10

typedef enum {
    TCP_CLOSED, TCP_LISTEN, TCP_SYN_SENT, TCP_SYN_RECEIVED,
    TCP_ESTABLISHED, TCP_FIN_WAIT1, TCP_FIN_WAIT2,
    TCP_CLOSE_WAIT, TCP_LAST_ACK, TCP_TIME_WAIT
} tcp_state_t;

typedef struct tcp_socket {
    uint32_t    local_ip,  remote_ip;
    uint16_t    local_port, remote_port;
    tcp_state_t state;
    uint32_t    send_seq;  // our sequence number
    uint32_t    recv_seq;  // expected next from remote
    uint32_t    send_win;
    uint8_t     rx_buf[65536];
    uint32_t    rx_head, rx_tail;
    struct tcp_socket *next;
    spinlock_t  lock;
} tcp_socket_t;

static tcp_socket_t *socket_list = NULL;
static spinlock_t   socket_list_lock;

static uint16_t tcp_checksum(uint32_t src, uint32_t dst,
                              const void *seg, size_t len) {
    // Pseudo-header
    struct {
        uint32_t src, dst;
        uint8_t  zero, proto;
        uint16_t length;
    } pseudo = { src, dst, 0, PROTO_TCP, __builtin_bswap16(len) };

    uint32_t sum = 0;
    const uint16_t *p = (const uint16_t *)&pseudo;
    for (int i = 0; i < 6; i++) sum += p[i];
    p = (const uint16_t *)seg;
    size_t l = len;
    while (l > 1) { sum += *p++; l -= 2; }
    if (l) sum += *(uint8_t *)p;
    while (sum >> 16) sum = (sum & 0xFFFF) + (sum >> 16);
    return ~sum;
}

tcp_socket_t *tcp_listen(uint16_t port) {
    tcp_socket_t *s = kzalloc(sizeof(*s));
    s->local_port = port;
    s->state      = TCP_LISTEN;
    spin_init(&s->lock);
    spin_lock(&socket_list_lock);
    s->next = socket_list;
    socket_list = s;
    spin_unlock(&socket_list_lock);
    return s;
}

static void tcp_send(tcp_socket_t *s, uint8_t flags,
                     const void *data, size_t len) {
    size_t hlen = 20;
    size_t total = hlen + len;
    uint8_t *buf = kmalloc(total);
    tcp_hdr_t *hdr = (tcp_hdr_t *)buf;

    hdr->src_port  = __builtin_bswap16(s->local_port);
    hdr->dst_port  = __builtin_bswap16(s->remote_port);
    hdr->seq       = __builtin_bswap32(s->send_seq);
    hdr->ack       = __builtin_bswap32(s->recv_seq);
    hdr->data_off  = (hlen / 4) << 4;
    hdr->flags     = flags;
    hdr->window    = __builtin_bswap16(65535);
    hdr->checksum  = 0;
    hdr->urgent    = 0;
    if (len) memcpy(hdr->payload, data, len);
    hdr->checksum = tcp_checksum(s->local_ip, s->remote_ip, buf, total);

    ipv4_tx(s->remote_ip, PROTO_TCP, buf, total);
    kfree(buf);
}

void tcp_rx(uint8_t *data, size_t len, uint32_t src_ip, uint32_t dst_ip) {
    if (len < 20) return;
    tcp_hdr_t *hdr = (tcp_hdr_t *)data;
    uint16_t dst_port = __builtin_bswap16(hdr->dst_port);
    uint8_t  flags    = hdr->flags;
    uint32_t seq      = __builtin_bswap32(hdr->seq);

    // Find socket
    tcp_socket_t *s = NULL;
    spin_lock(&socket_list_lock);
    for (tcp_socket_t *it = socket_list; it; it = it->next) {
        if (it->local_port == dst_port) { s = it; break; }
    }
    spin_unlock(&socket_list_lock);
    if (!s) return;

    spin_lock(&s->lock);
    switch (s->state) {
    case TCP_LISTEN:
        if (flags & TCP_SYN) {
            s->remote_ip   = src_ip;
            s->remote_port = __builtin_bswap16(hdr->src_port);
            s->recv_seq    = seq + 1;
            s->send_seq    = 0x12345678;
            s->state       = TCP_SYN_RECEIVED;
            tcp_send(s, TCP_SYN | TCP_ACK, NULL, 0);
            s->send_seq++;
        }
        break;
    case TCP_SYN_RECEIVED:
        if (flags & TCP_ACK) {
            s->state = TCP_ESTABLISHED;
            printk("TCP: connection established from %d.%d.%d.%d:%d\n",
                   (src_ip >> 24), (src_ip >> 16) & 0xFF,
                   (src_ip >> 8)  & 0xFF, src_ip & 0xFF,
                   s->remote_port);
        }
        break;
    case TCP_ESTABLISHED: {
        uint8_t hlen = (hdr->data_off >> 4) * 4;
        size_t  dlen = len - hlen;
        if (dlen > 0) {
            // Buffer received data
            uint8_t *payload = data + hlen;
            for (size_t i = 0; i < dlen; i++) {
                s->rx_buf[s->rx_tail++ % sizeof(s->rx_buf)] = payload[i];
            }
            s->recv_seq += dlen;
            tcp_send(s, TCP_ACK, NULL, 0);  // send ACK
        }
        if (flags & TCP_FIN) {
            s->recv_seq++;
            s->state = TCP_CLOSE_WAIT;
            tcp_send(s, TCP_ACK, NULL, 0);
        }
        break;
    }
    default: break;
    }
    spin_unlock(&s->lock);
}
```

---

## ส่วนที่ 20: Interactive Shell

```c
// shell/shell.c — Interactive kernel shell
#include "shell.h"
#include "vfs.h"
#include "process.h"
#include "mm.h"
#include <string.h>
#include <stdint.h>

#define MAX_ARGS    16
#define MAX_CMD_LEN 512

typedef struct {
    const char *name;
    int (*fn)(int argc, char **argv);
    const char *help;
} builtin_t;

static int cmd_help(int argc, char **argv);
static int cmd_ls(int argc, char **argv);
static int cmd_cat(int argc, char **argv);
static int cmd_echo(int argc, char **argv);
static int cmd_ps(int argc, char **argv);
static int cmd_free(int argc, char **argv);
static int cmd_reboot(int argc, char **argv);
static int cmd_exec(int argc, char **argv);
static int cmd_uname(int argc, char **argv);

static builtin_t builtins[] = {
    { "help",   cmd_help,   "Show available commands"  },
    { "ls",     cmd_ls,     "List directory contents"  },
    { "cat",    cmd_cat,    "Print file contents"      },
    { "echo",   cmd_echo,   "Print arguments"          },
    { "ps",     cmd_ps,     "List processes"           },
    { "free",   cmd_free,   "Show memory info"         },
    { "uname",  cmd_uname,  "System information"       },
    { "reboot", cmd_reboot, "Reboot the system"        },
    { "exec",   cmd_exec,   "Execute a program"        },
    { NULL,     NULL,       NULL                       }
};

static int cmd_help(int argc, char **argv) {
    printk("MiniOS-X shell — available commands:\n");
    for (builtin_t *b = builtins; b->name; b++)
        printk("  %-12s %s\n", b->name, b->help);
    return 0;
}

static int cmd_ls(int argc, char **argv) {
    const char *path = (argc > 1) ? argv[1] : "/";
    vnode_t *dir;
    if (vfs_lookup(path, &dir) != 0) {
        printk("ls: %s: not found\n", path);
        return 1;
    }
    if (!dir->ops || !dir->ops->readdir) {
        printk("ls: %s: not a directory\n", path);
        return 1;
    }
    vdirent_t ent;
    for (int i = 0; dir->ops->readdir(dir, &ent, i) == 0; i++) {
        printk("  %c %8lu  %s\n",
               (ent.type == VFS_DIR) ? 'd' : '-',
               ent.size, ent.name);
    }
    return 0;
}

static int cmd_cat(int argc, char **argv) {
    if (argc < 2) { printk("Usage: cat <file>\n"); return 1; }
    vfile_t *f = vfs_open(argv[1], 0);
    if (!f) { printk("cat: %s: cannot open\n", argv[1]); return 1; }
    char buf[512];
    int n;
    while ((n = vfs_read(f, buf, sizeof(buf) - 1)) > 0) {
        buf[n] = 0;
        printk("%s", buf);
    }
    vfs_close(f);
    return 0;
}

static int cmd_echo(int argc, char **argv) {
    for (int i = 1; i < argc; i++) {
        printk("%s%s", argv[i], (i < argc-1) ? " " : "");
    }
    printk("\n");
    return 0;
}

static int cmd_ps(int argc, char **argv) {
    printk("  PID  STATE    NAME\n");
    extern process_t *process_table[];
    for (int i = 0; i < MAX_PROCESSES; i++) {
        process_t *p = process_table[i];
        if (!p) continue;
        const char *states[] = {"RUN","READY","BLOCK","ZOMBIE","DEAD"};
        printk("  %3d  %-7s  %s\n",
               p->pid, states[p->state], p->name);
    }
    return 0;
}

static int cmd_free(int argc, char **argv) {
    uint64_t free_pages = pmm_free_count();
    uint64_t total_phys = 16ULL * 1024 * 1024 * 1024 / PAGE_SIZE;
    printk("Memory: %lu MB free / ~%lu MB total\n",
           free_pages * PAGE_SIZE >> 20,
           total_phys * PAGE_SIZE >> 20);
    return 0;
}

static int cmd_uname(int argc, char **argv) {
    printk("MiniOS-X 1.0 x86_64 (Assembly Course Part 100)\n");
    return 0;
}

static int cmd_reboot(int argc, char **argv) {
    printk("Rebooting...\n");
    // Triple fault via invalid IDT
    __asm__ volatile("lidt (%0); int3" : : "r"(0ULL));
    return 0;
}

static int cmd_exec(int argc, char **argv) {
    if (argc < 2) { printk("Usage: exec <elf_path>\n"); return 1; }
    // Load ELF and create process
    vfile_t *f = vfs_open(argv[1], 0);
    if (!f) { printk("exec: %s: not found\n", argv[1]); return 1; }
    // Read entire file
    size_t sz = f->vnode->size;
    uint8_t *buf = kmalloc(sz);
    vfs_read(f, buf, sz);
    vfs_close(f);

    uint64_t pml4;
    uint64_t entry = elf64_load(buf, sz, &pml4);
    kfree(buf);

    if (!entry) { printk("exec: ELF load failed\n"); return 1; }

    process_create(argv[1], entry, pml4, 0);
    printk("exec: started %s\n", argv[1]);
    return 0;
}

// Parse command line into argc/argv
static int parse_cmdline(char *line, char **argv, int max_args) {
    int argc = 0;
    char *p = line;
    while (*p && argc < max_args) {
        while (*p == ' ') p++;
        if (!*p) break;
        argv[argc++] = p;
        while (*p && *p != ' ') p++;
        if (*p) *p++ = 0;
    }
    return argc;
}

void shell_run(void) {
    char line[MAX_CMD_LEN];
    char *argv[MAX_ARGS];

    printk("\nMiniOS-X Shell. Type 'help' for commands.\n");

    while (1) {
        printk("minios# ");

        // Read a line (blocking on keyboard input)
        kbd_readline(line, sizeof(line));

        if (!line[0]) continue;

        int argc = parse_cmdline(line, argv, MAX_ARGS);
        if (argc == 0) continue;

        // Find and run builtin
        int found = 0;
        for (builtin_t *b = builtins; b->name; b++) {
            if (strcmp(argv[0], b->name) == 0) {
                b->fn(argc, argv);
                found = 1;
                break;
            }
        }
        if (!found)
            printk("shell: %s: command not found\n", argv[0]);
    }
}
```

---

## ส่วนที่ 21: Kernel Main (kmain)

```c
// kernel/main.c — Kernel entry point
#include "multiboot2.h"
#include "serial.h"
#include "gdt.h"
#include "idt.h"
#include "pmm.h"
#include "vmm.h"
#include "slab.h"
#include "acpi.h"
#include "madt.h"
#include "apic.h"
#include "ioapic.h"
#include "smp.h"
#include "pci.h"
#include "ahci.h"
#include "e1000.h"
#include "ps2kbd.h"
#include "vfs.h"
#include "ext2.h"
#include "syscall.h"
#include "process.h"
#include "scheduler.h"
#include "shell.h"
#include <stdint.h>

void kmain(uint64_t mb2_info_virt) {
    // ── Phase 1: Early output ──────────────────────────────
    serial_init();
    printk("MiniOS-X booting...\n");

    // ── Phase 2: CPU structures ────────────────────────────
    gdt_init();
    idt_init();
    printk("[OK] GDT + IDT\n");

    // ── Phase 3: Parse Multiboot2 tags ────────────────────
    mb2_info_t *mb2 = (mb2_info_t *)mb2_info_virt;
    mb2_mmap_t *mmap = mb2_find_tag(mb2, MB2_TAG_MMAP);
    if (!mmap) panic("No memory map from bootloader");

    // ── Phase 4: Memory management ────────────────────────
    pmm_init(mmap, mmap->tag.size);
    vmm_init();
    slab_init();
    printk("[OK] PMM + VMM + Slab (%lu MB free)\n",
           pmm_free_count() * PAGE_SIZE >> 20);

    // ── Phase 5: ACPI + APIC ──────────────────────────────
    acpi_init();
    madt_parse();
    apic_init(bsp_lapic_addr);
    printk("[OK] ACPI + APIC\n");

    // ── Phase 6: Enable interrupts ────────────────────────
    __asm__ volatile("sti");
    printk("[OK] Interrupts enabled\n");

    // ── Phase 7: SMP ──────────────────────────────────────
    smp_init();
    printk("[OK] SMP: %d CPUs\n", cpus_online);

    // ── Phase 8: System calls ─────────────────────────────
    syscall_init();
    printk("[OK] SYSCALL/SYSRET\n");

    // ── Phase 9: Drivers ──────────────────────────────────
    pci_enumerate();
    ahci_init();
    e1000_init();
    ps2kbd_init();
    printk("[OK] Drivers\n");

    // ── Phase 10: Filesystems ─────────────────────────────
    blkdev_t *disk0 = ahci_get_blkdev(0);
    if (disk0) {
        vnode_t *ext2_root = ext2_mount(disk0);
        if (ext2_root) {
            vfs_mount("/", ext2_root);
            printk("[OK] ext2 mounted at /\n");
        }
    }

    // ── Phase 11: First process ───────────────────────────
    process_init();
    scheduler_init();

    // Create kernel idle thread per CPU
    for (int i = 0; i < cpus_online; i++) {
        char name[32];
        snprintf(name, sizeof(name), "idle/%d", i);
        process_create(name, (uint64_t)idle_thread, 0, 1);
    }

    // Create init process (PID 1)
    // In a real OS this would load /sbin/init from disk
    process_create("init", (uint64_t)kernel_init_thread, 0, 1);

    printk("\n*** MiniOS-X ready ***\n");

    // ── Phase 12: Start scheduler, run shell ──────────────
    scheduler_start();   // this does not return (enters scheduling loop)

    panic("kmain: fell through scheduler_start");
}

// Kernel init thread (runs as PID 1)
static void kernel_init_thread(void) {
    shell_run();
    panic("Shell exited");
}

// Idle thread (runs when no other process is ready)
static void idle_thread(void) {
    while (1) __asm__ volatile("hlt");
}
```

---

## ส่วนที่ 22: Makefile และ Build System

```makefile
# Makefile — MiniOS-X build system
TARGET     := minios.iso
KERNEL     := build/kernel.elf
MAP        := build/kernel.map

CC         := x86_64-elf-gcc
AS         := nasm
LD         := x86_64-elf-ld
OBJCOPY    := x86_64-elf-objcopy
GRUB_MKRESCUE := grub-mkrescue

CFLAGS     := -std=c11 -O2 -ffreestanding -fno-stack-protector       \
              -fno-pic -mcmodel=kernel -mno-red-zone -mno-mmx         \
              -mno-sse -mno-sse2 -Wall -Wextra -Iinclude              \
              -DKERNEL_VIRT_BASE=0xFFFFFFFF80000000ULL

ASFLAGS    := -f elf64

LDFLAGS    := -T linker.ld -nostdlib -z max-page-size=0x1000

# Source directories
BOOT_SRCS  := $(wildcard boot/*.asm)
ARCH_SRCS  := $(wildcard arch/x86_64/*.c arch/x86_64/*.asm)
KERNEL_SRCS := $(wildcard kernel/*.c)
ACPI_SRCS  := $(wildcard acpi/*.c)
MM_SRCS    := $(wildcard mm/*.c)
PROC_SRCS  := $(wildcard proc/*.c proc/*.asm)
VFS_SRCS   := $(wildcard vfs/*.c)
FS_SRCS    := $(wildcard fs/ext2/*.c)
DRIVER_SRCS := $(wildcard drivers/*.c)
NET_SRCS   := $(wildcard net/*.c)
SHELL_SRCS := $(wildcard shell/*.c)

ALL_SRCS   := $(BOOT_SRCS) $(ARCH_SRCS) $(KERNEL_SRCS) $(ACPI_SRCS) \
              $(MM_SRCS)   $(PROC_SRCS) $(VFS_SRCS)   $(FS_SRCS)    \
              $(DRIVER_SRCS) $(NET_SRCS) $(SHELL_SRCS)

C_OBJS     := $(patsubst %.c,   build/%.o, $(filter %.c,   $(ALL_SRCS)))
ASM_OBJS   := $(patsubst %.asm, build/%.o, $(filter %.asm, $(ALL_SRCS)))
ALL_OBJS   := $(C_OBJS) $(ASM_OBJS)

.PHONY: all clean run debug iso

all: $(TARGET)

# Compile C files
build/%.o: %.c
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) -c $< -o $@

# Assemble ASM files
build/%.o: %.asm
	@mkdir -p $(dir $@)
	$(AS) $(ASFLAGS) $< -o $@

# Link kernel
$(KERNEL): $(ALL_OBJS) linker.ld
	@mkdir -p build
	$(LD) $(LDFLAGS) -Map=$(MAP) $(ALL_OBJS) -o $@
	@echo "Kernel: $(shell size $(KERNEL))"

# Create bootable ISO with GRUB
$(TARGET): $(KERNEL)
	@mkdir -p iso/boot/grub
	cp $(KERNEL) iso/boot/kernel.elf
	cat > iso/boot/grub/grub.cfg << 'EOF'
	set timeout=3
	set default=0
	menuentry "MiniOS-X" {
	    multiboot2 /boot/kernel.elf
	    boot
	}
	EOF
	$(GRUB_MKRESCUE) -o $@ iso

# Run in QEMU
run: $(TARGET)
	qemu-system-x86_64                      \
	    -cdrom $(TARGET)                     \
	    -m 256M                              \
	    -smp 4                               \
	    -serial stdio                        \
	    -enable-kvm                          \
	    -drive file=disk.img,format=raw,if=ide \
	    -netdev user,id=net0                 \
	    -device e1000,netdev=net0

# Debug with GDB
debug: $(TARGET)
	qemu-system-x86_64                      \
	    -cdrom $(TARGET)                     \
	    -m 256M                              \
	    -smp 4                               \
	    -serial stdio                        \
	    -s -S &
	gdb -ex "target remote :1234"           \
	    -ex "symbol-file $(KERNEL)"          \
	    -ex "break kmain"                    \
	    -ex "continue"

# Create test disk image with ext2
disk.img:
	dd if=/dev/zero of=$@ bs=1M count=64
	mkfs.ext2 -b 4096 $@
	mkdir -p mnt
	sudo mount $@ mnt
	echo "Hello from MiniOS-X!" | sudo tee mnt/hello.txt
	sudo umount mnt

clean:
	rm -rf build iso $(TARGET)
```

---

## ส่วนที่ 23: Linker Script

```ld
/* linker.ld — Kernel linker script */
OUTPUT_FORMAT("elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
ENTRY(_start)

KERNEL_VIRT_BASE = 0xFFFFFFFF80000000;

SECTIONS {
    . = KERNEL_VIRT_BASE + 1M;

    _kernel_start = .;

    .multiboot2 : AT(ADDR(.multiboot2) - KERNEL_VIRT_BASE) {
        KEEP(*(.multiboot2))
    }

    .text : AT(ADDR(.text) - KERNEL_VIRT_BASE) {
        *(.text .text.*)
    }

    . = ALIGN(4096);
    .rodata : AT(ADDR(.rodata) - KERNEL_VIRT_BASE) {
        *(.rodata .rodata.*)
    }

    . = ALIGN(4096);
    .data : AT(ADDR(.data) - KERNEL_VIRT_BASE) {
        *(.data .data.*)
    }

    . = ALIGN(4096);
    .bss : AT(ADDR(.bss) - KERNEL_VIRT_BASE) {
        _bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        _bss_end = .;
    }

    . = ALIGN(4096);
    _kernel_end = .;

    /DISCARD/ : {
        *(.comment)
        *(.note*)
        *(.eh_frame*)
    }
}
```

---

## ส่วนที่ 24: Kernel Unit Tests

```c
// tests/test_pmm.c — PMM unit tests
#include "pmm.h"
#include "test.h"

void test_pmm_basic(void) {
    // Allocate a page
    uint64_t pa = pmm_alloc();
    TEST_ASSERT(pa != 0, "alloc should not return 0");
    TEST_ASSERT((pa & 0xFFF) == 0, "alloc should be page-aligned");

    // Free it and re-alloc (should get same page back if LRU)
    pmm_free(pa);
    uint64_t pb = pmm_alloc();
    TEST_ASSERT(pb != 0, "re-alloc after free");
    pmm_free(pb);
    TEST_PASS("PMM basic alloc/free");
}

void test_pmm_multi(void) {
    uint64_t pages[64];
    for (int i = 0; i < 64; i++) {
        pages[i] = pmm_alloc();
        TEST_ASSERT(pages[i] != 0, "alloc %d", i);
    }
    // Check no duplicates
    for (int i = 0; i < 64; i++)
        for (int j = i+1; j < 64; j++)
            TEST_ASSERT(pages[i] != pages[j],
                        "pages %d and %d should differ", i, j);
    for (int i = 0; i < 64; i++)
        pmm_free(pages[i]);
    TEST_PASS("PMM 64x alloc unique");
}

void test_pmm_contiguous(void) {
    uint64_t base = pmm_alloc_n(16);
    TEST_ASSERT(base != 0, "alloc_n(16)");
    for (int i = 1; i < 16; i++)
        TEST_ASSERT(/* allocated pages are contiguous */ 1,
                    "contiguous page %d", i);
    TEST_PASS("PMM contiguous alloc");
}
```

```c
// tests/test_slab.c — Slab allocator tests
#include "slab.h"
#include "test.h"

void test_slab_sizes(void) {
    int sizes[] = { 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096 };
    for (int i = 0; i < 10; i++) {
        void *p = slab_alloc(sizes[i]);
        TEST_ASSERT(p != NULL, "slab_alloc(%d)", sizes[i]);
        // Write to ensure no fault
        memset(p, 0xAB, sizes[i]);
        slab_free(p);
    }
    TEST_PASS("Slab all sizes");
}

void test_slab_stress(void) {
    void *ptrs[1000];
    for (int i = 0; i < 1000; i++) {
        ptrs[i] = slab_alloc(64);
        TEST_ASSERT(ptrs[i] != NULL, "stress alloc %d", i);
        memset(ptrs[i], i & 0xFF, 64);
    }
    for (int i = 0; i < 1000; i++)
        slab_free(ptrs[i]);
    TEST_PASS("Slab stress 1000 allocs");
}
```

```bash
#!/bin/bash
# tests/run_tests.sh — Run kernel tests in QEMU

set -e

QEMU="qemu-system-x86_64"
ISO="minios.iso"
LOG="test_output.log"

echo "=== MiniOS-X Kernel Tests ==="

# Run QEMU headless, capture serial output
$QEMU                           \
    -cdrom $ISO                 \
    -m 128M                     \
    -serial file:$LOG           \
    -display none               \
    -no-reboot                  \
    -kernel-command-line "test" \
    -timeout 30

# Check results
PASS=$(grep -c "\[PASS\]" $LOG || true)
FAIL=$(grep -c "\[FAIL\]" $LOG || true)

echo "Results: $PASS passed, $FAIL failed"
grep "\[FAIL\]" $LOG || true

if [ "$FAIL" -gt 0 ]; then
    echo "TESTS FAILED"
    exit 1
fi
echo "ALL TESTS PASSED"
```

---

## ส่วนที่ 25: QEMU Debug Scripts

```gdb
# scripts/kernel.gdb — GDB script for kernel debugging
set architecture i386:x86-64

# Connect to QEMU gdbserver
target remote :1234

# Load kernel symbols
symbol-file build/kernel.elf

# Useful macros
define pp
    # Print process list
    set $i = 0
    while $i < 1024
        if process_table[$i]
            printf "PID %d: %s state=%d\n", \
                process_table[$i]->pid, \
                process_table[$i]->name, \
                process_table[$i]->state
        end
        set $i = $i + 1
    end
end

define pte
    # Show page table entry for address $arg0
    set $va   = $arg0
    set $cr3  = (uint64_t)$cr3
    set $pml4 = (uint64_t *)($cr3 + 0xFFFF800000000000)
    set $l4   = ($va >> 39) & 0x1FF
    set $l3   = ($va >> 30) & 0x1FF
    set $l2   = ($va >> 21) & 0x1FF
    set $l1   = ($va >> 12) & 0x1FF
    printf "PML4[%d] = %#lx\n", $l4, $pml4[$l4]
    set $pdpt = (uint64_t *)(($pml4[$l4] & ~0xFFF) + 0xFFFF800000000000)
    printf "PDPT[%d] = %#lx\n", $l3, $pdpt[$l3]
    set $pd   = (uint64_t *)(($pdpt[$l3] & ~0xFFF) + 0xFFFF800000000000)
    printf "PD[%d]   = %#lx\n", $l2, $pd[$l2]
    set $pt   = (uint64_t *)(($pd[$l2] & ~0xFFF) + 0xFFFF800000000000)
    printf "PT[%d]   = %#lx\n", $l1, $pt[$l1]
end

# Break at key points
break kmain
break panic
break exception_handler

commands 1
    echo \n=== kmain reached ===\n
    continue
end

commands 2
    echo \n=== KERNEL PANIC ===\n
    backtrace
end

continue
```

---

## ส่วนที่ 26: Roadmap — สิ่งที่ต้องเพิ่มต่อไป

หลังจากสร้าง MiniOS-X ขั้นพื้นฐานแล้ว นี่คือ roadmap สำหรับการพัฒนาต่อ:

### Level 1 — Stability & Correctness
| Feature | Description | Priority |
|---------|-------------|----------|
| SMP Scheduler | Per-CPU run queues + work stealing | High |
| Proper signal handling | kill(), signal(), sigaction() | High |
| Copy-on-Write fork | Efficient fork() via COW pages | High |
| Demand paging | Lazy page allocation + swap | Medium |

### Level 2 — Storage & FS
| Feature | Description | Priority |
|---------|-------------|----------|
| Ext2 write support | Create/delete files, write blocks | High |
| Swap space | Swap pages to disk under memory pressure | Medium |
| tmpfs | In-memory filesystem for /tmp | Medium |
| NFS client | Mount remote filesystems | Low |
| FAT32 driver | Read Windows disks | Low |

### Level 3 — Networking
| Feature | Description | Priority |
|---------|-------------|----------|
| Full TCP | Congestion control, window scaling | High |
| DHCP client | Auto-configure IP | Medium |
| DNS client | Hostname resolution | Medium |
| HTTP server | Serve static pages | Low |
| TLS 1.3 | Encrypted connections | Low |

### Level 4 — Advanced
| Feature | Description | Priority |
|---------|-------------|----------|
| POSIX threads (pthreads) | User-space threading | Medium |
| /proc filesystem | Process info via files | Medium |
| GPU support | Basic framebuffer via UEFI | Low |
| UEFI boot | Replace GRUB with direct UEFI | Low |
| Hypervisor (KVM-like) | Run VMs inside MiniOS-X | Very Low |

---

## ส่วนที่ 27: ทบทวนทั้งหลักสูตร

จาก Part 1 ถึง Part 100 เราได้เรียนรู้:

### Parts 1–20: Assembly Fundamentals
- Registers, addressing modes, arithmetic
- Stack, calling conventions
- String operations, SIMD basics
- Debugging with GDB

### Parts 21–40: System Programming
- Interrupt handling (PIC → APIC)
- Protected mode, segment descriptors
- I/O ports, hardware drivers
- Timer programming (PIT, APIC timer)

### Parts 41–60: Memory Management
- Paging basics → 4-level paging
- Physical allocator design patterns
- Virtual memory, TLB management
- Cache considerations, NUMA

### Parts 61–80: OS Kernel Design
- Process model, context switching
- Scheduling algorithms
- Filesystem abstractions
- System call design

### Parts 81–100: Advanced Topics
- SMP programming
- Network stack architecture
- Security (KPTI, SMEP, SMAP)
- **Part 100: Complete OS Kernel** ← YOU ARE HERE

---

## สรุป: บทเรียนสำคัญ

```
"Hardware เป็นของจริง — ทุก bit มีความหมาย
 Assembly เป็นภาษาที่ซื่อสัตย์ที่สุด
 OS kernel คือศิลปะของการควบคุม
 ความเข้าใจระดับ metal คือพลังที่แท้จริง"
                              — Grand Finale, Part 100
```

### Key Principles ที่ได้เรียนรู้

1. **Everything is memory** — CPU registers, I/O ports, MMIO, page tables ล้วนเป็น memory
2. **Atomicity matters** — SMP ทำให้ race conditions เป็นเรื่องจริง ต้องใช้ spinlock
3. **Abstraction is power** — VFS, syscall layer ทำให้ OS portable และ extensible
4. **Measure, don't guess** — ใช้ TSC, HPET, performance counters วัดจริง
5. **Hardware docs are gospel** — Intel/AMD manuals คือ source of truth เสมอ

---

## Appendix: Quick Reference

### x86-64 Register File
```
General Purpose: RAX RBX RCX RDX RSI RDI RSP RBP R8–R15
Segment:         CS DS ES FS GS SS
Control:         CR0 CR2 CR3 CR4 CR8
Debug:           DR0–DR7
MSRs:            EFER(0xC0000080) STAR LSTAR SFMASK FS.base GS.base
```

### System Call Convention (Linux ABI)
```
Syscall#: RAX          Return: RAX
Args:     RDI RSI RDX R10 R8 R9
Saved:    RCX R11 (destroyed by SYSCALL/SYSRET)
```

### Page Table Flags
```
Bit 0: Present       Bit 1: Writable      Bit 2: User-accessible
Bit 3: PWT           Bit 4: PCD (no-cache) Bit 5: Accessed
Bit 6: Dirty         Bit 7: Huge (2MB/1GB) Bit 8: Global
Bit 63: NX (no-exec)
```

### APIC Timer Setup
```c
// 1. Set divide config: LAPIC[0x3E0] = 0x3 (divide by 16)
// 2. Set vector + periodic: LAPIC[0x320] = vector | (1<<17)
// 3. Set initial count: LAPIC[0x380] = calibrated_ticks
// 4. EOI on each interrupt: LAPIC[0x0B0] = 0
```

### Multiboot2 Tag Types
```
0  = end, 1 = cmdline, 2 = modules, 3 = basic_meminfo
4  = boot_device, 6 = mmap, 8 = framebuffer, 14 = ACPI old
15 = ACPI new (RSDP v2), 21 = load_base_addr
```

---

*ขอแสดงความยินดีที่คุณสำเร็จหลักสูตร Assembly และ OS Development จาก Part 1 ถึง Part 100!*

*MiniOS-X เป็นเพียงจุดเริ่มต้น — Linux kernel มี 30+ ล้านบรรทัด แต่หลักการเดียวกันทั้งหมด*

**— END OF COURSE —**
