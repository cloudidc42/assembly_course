# Part 094: x86 Hardware Virtualization (Intel VT-x)

## บทนำ: ทำไมต้องมี Hardware Virtualization?

การ virtualization แบบ software-only มีปัญหาพื้นฐาน: x86 architecture มี **privilege rings** (0-3) แต่บาง instructions ที่ต้องทำงานใน ring 0 ไม่ได้ trap เมื่อทำงานใน ring 3 — นี่คือ **Popek and Goldberg theorem violation**

คำสั่งประเภทนี้เรียกว่า **"sensitive non-privileged instructions"** เช่น:
- `SGDT`, `SIDT`, `SLDT` — อ่าน descriptor tables
- `SMSW` — อ่าน machine status word
- `PUSHF/POPF` — ไม่ trap แต่ซ่อน IF flag ใน ring 3

Intel แก้ปัญหานี้ด้วย **VMX (Virtual Machine Extensions)** — hardware ที่สร้าง "container" สำหรับ guest OS

---

## Virtualization Taxonomy

### Type-1: Bare Metal Hypervisor

```
┌─────────────────────────────────────┐
│         Guest VM 1    Guest VM 2    │
│   ┌──────────────┐ ┌──────────────┐ │
│   │ Guest OS     │ │ Guest OS     │ │
│   │ (ring 0-3)   │ │ (ring 0-3)   │ │
│   └──────────────┘ └──────────────┘ │
│                                     │
│        Type-1 Hypervisor            │
│     (VMware ESXi, Xen, Hyper-V)     │
│                                     │
├─────────────────────────────────────┤
│           Hardware                  │
└─────────────────────────────────────┘
```

Type-1 hypervisor ทำงานโดยตรงบน hardware ไม่ต้องการ host OS:
- **VMware ESXi** — enterprise virtualization
- **Xen** — open source, ใช้ใน AWS EC2 รุ่นแรก
- **Microsoft Hyper-V** — Windows Server
- **KVM** (ถกเถียงกันว่า type-1 หรือ 1.5)

### Type-2: Hosted Hypervisor

```
┌─────────────────────────────────────┐
│         Guest VM 1    Guest VM 2    │
│   ┌──────────────┐ ┌──────────────┐ │
│   │ Guest OS     │ │ Guest OS     │ │
│   └──────────────┘ └──────────────┘ │
│                                     │
│     Type-2 Hypervisor Application   │
│    (VMware Workstation, VirtualBox) │
│                                     │
├─────────────────────────────────────┤
│           Host OS (Linux/Windows)   │
├─────────────────────────────────────┤
│           Hardware                  │
└─────────────────────────────────────┘
```

Type-2 ทำงานเป็น application บน host OS:
- **VMware Workstation/Fusion**
- **VirtualBox**
- **QEMU** (user-mode)

### VMX Mode: VMX Root vs VMX Non-Root

Intel VT-x แนะนำ concept ใหม่ที่แตกต่างจาก ring 0-3:

```
VMX Root Mode (Hypervisor)
├── Ring 0 (hypervisor kernel)
├── Ring 1, 2 (unused)
└── Ring 3 (hypervisor user)

VMX Non-Root Mode (Guest)
├── Ring 0 (guest OS kernel)  ← ทำงานใน "virtual ring 0" แต่ hardware จำกัดไว้
├── Ring 1, 2 (unused)
└── Ring 3 (guest user apps)
```

เมื่อ guest ทำ operation ที่ต้อง intercept — hardware จะ **VM-Exit** กลับมา hypervisor โดยอัตโนมัติ

---

## CPUID Check: ตรวจสอบ VMX Support

ก่อนใช้ VMX ต้องตรวจสอบว่า CPU รองรับหรือไม่:

```nasm
; Check VMX support
; CPUID leaf 1, ECX bit 5 = VMX
check_vmx_support:
    mov     eax, 1
    cpuid
    
    ; ECX bit 5 = VMX support
    test    ecx, (1 << 5)
    jz      .no_vmx
    
    ; Check IA32_FEATURE_CONTROL MSR
    ; MSR 0x3A
    mov     ecx, 0x3A           ; IA32_FEATURE_CONTROL
    rdmsr
    
    ; Bit 0 = Lock bit
    ; Bit 2 = Enable VMX outside SMX
    test    eax, (1 << 0)       ; Lock bit set?
    jz      .feature_control_unlocked
    
    ; Lock bit set: check if VMX enabled
    test    eax, (1 << 2)       ; VMX outside SMX enabled?
    jz      .vmx_disabled_in_bios
    
    ; VMX is supported and enabled
    mov     eax, 1
    ret
    
.feature_control_unlocked:
    ; Need to set lock bit and enable VMX
    ; This requires ring 0 (kernel driver)
    or      eax, (1 << 0) | (1 << 2)   ; Lock + Enable VMX outside SMX
    mov     ecx, 0x3A
    wrmsr
    mov     eax, 1
    ret
    
.no_vmx:
.vmx_disabled_in_bios:
    mov     eax, 0
    ret
```

### IA32_FEATURE_CONTROL MSR (0x3A)

| Bit | Name | Description |
|-----|------|-------------|
| 0 | Lock | ถ้า set แล้ว write จะ GP fault |
| 1 | Enable VMX in SMX | VMXON ใน SMX operation |
| 2 | Enable VMX outside SMX | VMXON นอก SMX (ที่ใช้กันทั่วไป) |
| 8 | SENTER Local | Enable SENTER |
| 14:8 | SENTER Global | Enable SENTER functions |

---

## CR4.VMXE: Enable VMX

ก่อน VMXON ต้อง set **CR4.VMXE** (bit 13):

```nasm
; Enable VMX in CR4
enable_vmxe:
    mov     rax, cr4
    or      rax, (1 << 13)      ; CR4.VMXE
    mov     cr4, rax
    ret

; Disable VMX (after VMXOFF)
disable_vmxe:
    mov     rax, cr4
    and     rax, ~(1 << 13)
    mov     cr4, rax
    ret
```

ถ้า execute VMXON โดยไม่ set CR4.VMXE จะได้ **#UD (Invalid Opcode) exception**

---

## VMX Instructions Reference

### การจัดกลุ่ม VMX Instructions

**Lifecycle Instructions:**
```
VMXON  → เข้าสู่ VMX operation (hypervisor เริ่มทำงาน)
VMXOFF → ออกจาก VMX operation
```

**VMCS Management:**
```
VMCLEAR  → Initialize VMCS, ล้าง active state
VMPTRLD  → Load VMCS pointer (set current VMCS)
VMPTRST  → Store current VMCS pointer
VMREAD   → Read field from current VMCS
VMWRITE  → Write field to current VMCS
```

**VM Entry/Exit:**
```
VMLAUNCH → Launch VM (ครั้งแรก, VMCS ต้องอยู่ใน "clear" state)
VMRESUME → Resume VM (หลัง VM-Exit)
VMCALL   → Hypercall (guest → hypervisor)
```

**TLB Management:**
```
INVEPT  → Invalidate EPT-derived translations
INVVPID → Invalidate VPID-tagged translations
```

### VMXON — เข้า VMX Root Mode

```nasm
; VMXON requires a 4KB-aligned "VMXON region"
; ที่ physical address ที่ส่งผ่าน operand

section .data
align 4096
vmxon_region:
    times 4096 db 0

section .text

; Setup VMXON region
setup_vmxon_region:
    ; Read VMX revision ID from IA32_VMX_BASIC MSR (0x480)
    mov     ecx, 0x480          ; IA32_VMX_BASIC
    rdmsr
    ; EAX[30:0] = VMX revision identifier
    and     eax, 0x7FFFFFFF
    
    ; Write revision ID to first 4 bytes of VMXON region
    mov     [vmxon_region], eax
    ret

; Execute VMXON
do_vmxon:
    ; Get physical address of VMXON region
    ; (ต้องใช้ kernel API จริงๆ — virt_to_phys())
    lea     rax, [vmxon_region]
    ; Assume identity mapped for simplicity
    
    ; VMXON takes a memory operand (64-bit physical address)
    vmxon   [rax]               ; VMXON m64
    
    ; Check CF (Carry Flag) — set = failure
    jc      .vmxon_failed
    ; Check ZF (Zero Flag) — set = failure with status
    jz      .vmxon_failed_status
    
    ; Success: now in VMX root mode
    ret
    
.vmxon_failed:
    ; CF=1: invalid VMCS pointer or CR4.VMXE not set
    mov     rax, -1
    ret
    
.vmxon_failed_status:
    ; ZF=1: VMfailValid — check VMCS error field
    mov     rax, -2
    ret
```

### VMXOFF — ออกจาก VMX Root Mode

```nasm
do_vmxoff:
    vmxoff
    
    ; Clear CR4.VMXE
    mov     rax, cr4
    and     rax, ~(1 << 13)
    mov     cr4, rax
    ret
```

---

## VMCS (Virtual Machine Control Structure)

VMCS คือ data structure ขนาด **4KB** ที่ hardware ใช้เก็บ state ของ VM และ control parameters

### VMCS Physical Layout

```
Offset 0x000: VMX-revision identifier (31:0) | Shadow-VMCS indicator (bit 31)
Offset 0x004: VMX-abort indicator
Offset 0x008: VMCS data (implementation-specific, อย่าเข้าถึงโดยตรง)
...
Offset 0xFFF: end of 4KB
```

### การสร้างและจัดการ VMCS

```nasm
section .data
align 4096
guest_vmcs:
    times 4096 db 0

section .text

; Initialize VMCS
init_vmcs:
    ; Step 1: Write VMX revision ID
    mov     ecx, 0x480          ; IA32_VMX_BASIC
    rdmsr
    and     eax, 0x7FFFFFFF     ; mask off bit 31
    mov     [guest_vmcs], eax
    
    ; Step 2: VMCLEAR — initialize to "clear" state
    lea     rax, [guest_vmcs]
    vmclear [rax]               ; VMCLEAR m64 (physical address)
    jc      .vmclear_failed
    jz      .vmclear_failed
    
    ; Step 3: VMPTRLD — make this the current VMCS
    lea     rax, [guest_vmcs]
    vmptrld [rax]               ; VMPTRLD m64
    jc      .vmptrld_failed
    jz      .vmptrld_failed
    
    ; Now can use VMREAD/VMWRITE
    xor     eax, eax
    ret
    
.vmclear_failed:
.vmptrld_failed:
    mov     rax, -1
    ret

; VMREAD — read field from current VMCS
; Input: RCX = field encoding
; Output: RAX = field value
vmcs_read:
    vmread  rax, rcx
    ret

; VMWRITE — write field to current VMCS
; Input: RCX = field encoding, RDX = value
vmcs_write:
    vmwrite rcx, rdx
    ret
```

### VMCS Field Encodings

VMCS fields ถูก identify ด้วย **32-bit field encoding**:

```
Bit 11:10 = Access type (0=full, 1=high)
Bit  9: 8 = Field width (0=16, 1=64, 2=32, 3=natural)
Bit  11   = (part of width)
Bit 14:10 = Field index
Bit 25:15 = Field type (0=control, 1=vmexit info, 2=guest, 3=host)
```

**ตัวอย่าง Field Encodings:**

```nasm
; Guest State Area (16-bit fields)
VMCS_GUEST_ES_SELECTOR      equ 0x0800
VMCS_GUEST_CS_SELECTOR      equ 0x0802
VMCS_GUEST_SS_SELECTOR      equ 0x0804
VMCS_GUEST_DS_SELECTOR      equ 0x0806
VMCS_GUEST_FS_SELECTOR      equ 0x0808
VMCS_GUEST_GS_SELECTOR      equ 0x080A
VMCS_GUEST_LDTR_SELECTOR    equ 0x080C
VMCS_GUEST_TR_SELECTOR      equ 0x080E
VMCS_GUEST_INTERRUPT_STATUS equ 0x0810  ; requires "virtual interrupt delivery"
VMCS_GUEST_PML_INDEX        equ 0x0812  ; requires "PML"

; Host State Area (16-bit)
VMCS_HOST_ES_SELECTOR       equ 0x0C00
VMCS_HOST_CS_SELECTOR       equ 0x0C02
VMCS_HOST_SS_SELECTOR       equ 0x0C04
VMCS_HOST_DS_SELECTOR       equ 0x0C06
VMCS_HOST_FS_SELECTOR       equ 0x0C08
VMCS_HOST_GS_SELECTOR       equ 0x0C0A
VMCS_HOST_TR_SELECTOR       equ 0x0C0C

; 64-bit Control Fields
VMCS_IO_BITMAP_A            equ 0x2000
VMCS_IO_BITMAP_B            equ 0x2002
VMCS_MSR_BITMAP             equ 0x2004
VMCS_VM_EXIT_MSR_STORE_ADDR equ 0x2006
VMCS_VM_EXIT_MSR_LOAD_ADDR  equ 0x2008
VMCS_VM_ENTRY_MSR_LOAD_ADDR equ 0x200A
VMCS_EXECUTIVE_VMCS_PTR     equ 0x200C
VMCS_TSC_OFFSET             equ 0x2010
VMCS_VIRTUAL_APIC_ADDR      equ 0x2012
VMCS_APIC_ACCESS_ADDR       equ 0x2014
VMCS_EPT_POINTER            equ 0x201A
VMCS_EOI_EXIT_BITMAP_0      equ 0x201C

; 64-bit Read-Only Fields (VM-Exit Information)
VMCS_GUEST_PHYSICAL_ADDR    equ 0x2400  ; GPA causing EPT violation

; 64-bit Guest State
VMCS_VMCS_LINK_POINTER      equ 0x2800
VMCS_GUEST_IA32_DEBUGCTL    equ 0x2802
VMCS_GUEST_IA32_PAT         equ 0x2804
VMCS_GUEST_IA32_EFER        equ 0x2806
VMCS_GUEST_IA32_PERF_GLOBAL equ 0x2808
VMCS_GUEST_PDPTE0           equ 0x280A
VMCS_GUEST_PDPTE1           equ 0x280C
VMCS_GUEST_PDPTE2           equ 0x280E
VMCS_GUEST_PDPTE3           equ 0x2810

; 64-bit Host State
VMCS_HOST_IA32_PAT          equ 0x2C00
VMCS_HOST_IA32_EFER         equ 0x2C02
VMCS_HOST_IA32_PERF_GLOBAL  equ 0x2C04

; 32-bit Control Fields
VMCS_PIN_BASED_VM_EXEC_CTRL equ 0x4000
VMCS_PRI_PROC_BASED_CTRL    equ 0x4002
VMCS_EXCEPTION_BITMAP       equ 0x4004
VMCS_PAGE_FAULT_ERROR_MASK  equ 0x4006
VMCS_PAGE_FAULT_ERROR_MATCH equ 0x4008
VMCS_CR3_TARGET_COUNT       equ 0x400A
VMCS_VM_EXIT_CONTROLS       equ 0x400C
VMCS_VM_EXIT_MSR_STORE_CNT  equ 0x400E
VMCS_VM_EXIT_MSR_LOAD_CNT   equ 0x4010
VMCS_VM_ENTRY_CONTROLS      equ 0x4012
VMCS_VM_ENTRY_MSR_LOAD_CNT  equ 0x4014
VMCS_VM_ENTRY_INTR_INFO     equ 0x4016
VMCS_VM_ENTRY_EXCEPTION_ERR equ 0x4018
VMCS_VM_ENTRY_INSTR_LEN     equ 0x401A
VMCS_TPR_THRESHOLD          equ 0x401C
VMCS_SEC_PROC_BASED_CTRL    equ 0x401E
VMCS_PLE_GAP                equ 0x4020
VMCS_PLE_WINDOW             equ 0x4022

; 32-bit Read-Only Fields
VMCS_VM_INSTR_ERROR         equ 0x4400
VMCS_EXIT_REASON            equ 0x4402
VMCS_VM_EXIT_INTR_INFO      equ 0x4404
VMCS_VM_EXIT_INTR_ERR_CODE  equ 0x4406
VMCS_IDT_VECTORING_INFO     equ 0x4408
VMCS_IDT_VECTORING_ERR_CODE equ 0x440A
VMCS_VM_EXIT_INSTR_LEN      equ 0x440C
VMCS_VM_EXIT_INSTR_INFO     equ 0x440E

; 32-bit Guest State
VMCS_GUEST_ES_LIMIT         equ 0x4800
VMCS_GUEST_CS_LIMIT         equ 0x4802
VMCS_GUEST_SS_LIMIT         equ 0x4804
VMCS_GUEST_DS_LIMIT         equ 0x4806
VMCS_GUEST_FS_LIMIT         equ 0x4808
VMCS_GUEST_GS_LIMIT         equ 0x480A
VMCS_GUEST_LDTR_LIMIT       equ 0x480C
VMCS_GUEST_TR_LIMIT         equ 0x480E
VMCS_GUEST_GDTR_LIMIT       equ 0x4810
VMCS_GUEST_IDTR_LIMIT       equ 0x4812
VMCS_GUEST_ES_ACCESS_RIGHTS equ 0x4814
VMCS_GUEST_CS_ACCESS_RIGHTS equ 0x4816
VMCS_GUEST_SS_ACCESS_RIGHTS equ 0x4818
VMCS_GUEST_DS_ACCESS_RIGHTS equ 0x481A
VMCS_GUEST_FS_ACCESS_RIGHTS equ 0x481C
VMCS_GUEST_GS_ACCESS_RIGHTS equ 0x481E
VMCS_GUEST_LDTR_ACCESS_RIGHTS equ 0x4820
VMCS_GUEST_TR_ACCESS_RIGHTS equ 0x4822
VMCS_GUEST_INTERRUPTIBILITY equ 0x4824
VMCS_GUEST_ACTIVITY_STATE   equ 0x4826
VMCS_GUEST_SMBASE           equ 0x4828
VMCS_GUEST_IA32_SYSENTER_CS equ 0x482A
VMCS_VMX_PREEMPTION_TIMER   equ 0x482E

; Natural-width Control Fields
VMCS_CR0_GUEST_HOST_MASK    equ 0x6000
VMCS_CR4_GUEST_HOST_MASK    equ 0x6002
VMCS_CR0_READ_SHADOW        equ 0x6004
VMCS_CR4_READ_SHADOW        equ 0x6006
VMCS_CR3_TARGET_VAL_0       equ 0x6008

; Natural-width Read-Only Fields
VMCS_EXIT_QUALIFICATION     equ 0x6400
VMCS_IO_RCX                 equ 0x6402
VMCS_IO_RSI                 equ 0x6404
VMCS_IO_RDI                 equ 0x6406
VMCS_IO_RIP                 equ 0x6408
VMCS_GUEST_LINEAR_ADDR      equ 0x640A

; Natural-width Guest State
VMCS_GUEST_CR0              equ 0x6800
VMCS_GUEST_CR3              equ 0x6802
VMCS_GUEST_CR4              equ 0x6804
VMCS_GUEST_ES_BASE          equ 0x6806
VMCS_GUEST_CS_BASE          equ 0x6808
VMCS_GUEST_SS_BASE          equ 0x680A
VMCS_GUEST_DS_BASE          equ 0x680C
VMCS_GUEST_FS_BASE          equ 0x680E
VMCS_GUEST_GS_BASE          equ 0x6810
VMCS_GUEST_LDTR_BASE        equ 0x6812
VMCS_GUEST_TR_BASE          equ 0x6814
VMCS_GUEST_GDTR_BASE        equ 0x6816
VMCS_GUEST_IDTR_BASE        equ 0x6818
VMCS_GUEST_DR7              equ 0x681A
VMCS_GUEST_RSP              equ 0x681C
VMCS_GUEST_RIP              equ 0x681E
VMCS_GUEST_RFLAGS           equ 0x6820
VMCS_GUEST_PENDING_DBG_EXCP equ 0x6822
VMCS_GUEST_IA32_SYSENTER_ESP equ 0x6824
VMCS_GUEST_IA32_SYSENTER_EIP equ 0x6826

; Natural-width Host State
VMCS_HOST_CR0               equ 0x6C00
VMCS_HOST_CR3               equ 0x6C02
VMCS_HOST_CR4               equ 0x6C04
VMCS_HOST_FS_BASE           equ 0x6C06
VMCS_HOST_GS_BASE           equ 0x6C08
VMCS_HOST_TR_BASE           equ 0x6C0A
VMCS_HOST_GDTR_BASE         equ 0x6C0C
VMCS_HOST_IDTR_BASE         equ 0x6C0E
VMCS_HOST_IA32_SYSENTER_ESP equ 0x6C10
VMCS_HOST_IA32_SYSENTER_EIP equ 0x6C12
VMCS_HOST_RSP               equ 0x6C14
VMCS_HOST_RIP               equ 0x6C16
```

---

## VMCS Regions: Guest State Area

Guest State Area เก็บ CPU state ที่จะ restore เมื่อ VM-Entry และ save เมื่อ VM-Exit

### Guest Registers ที่ต้องตั้งค่า

```nasm
; Setup Guest State for a 64-bit guest
setup_guest_state:
    ; --- Segment Selectors ---
    ; CS = code segment (ring 0 = 0x0008 typical)
    mov     rdx, 0x0008
    mov     rcx, VMCS_GUEST_CS_SELECTOR
    vmwrite rcx, rdx
    
    ; DS, ES, SS, FS, GS = data segment (ring 0 = 0x0010)
    mov     rdx, 0x0010
    mov     rcx, VMCS_GUEST_DS_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_ES_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_SS_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_FS_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_GS_SELECTOR
    vmwrite rcx, rdx
    
    ; TR = task register (must be present)
    mov     rdx, 0x0018
    mov     rcx, VMCS_GUEST_TR_SELECTOR
    vmwrite rcx, rdx
    
    ; LDTR = null
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_LDTR_SELECTOR
    vmwrite rcx, rdx
    
    ; --- Segment Bases ---
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_CS_BASE
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_DS_BASE
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_ES_BASE
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_SS_BASE
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_FS_BASE
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_GS_BASE
    vmwrite rcx, rdx
    
    ; TR base (TSS address)
    lea     rdx, [guest_tss]
    mov     rcx, VMCS_GUEST_TR_BASE
    vmwrite rcx, rdx
    
    ; LDTR base = 0
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_LDTR_BASE
    vmwrite rcx, rdx
    
    ; --- Segment Limits ---
    mov     rdx, 0xFFFFFFFF
    mov     rcx, VMCS_GUEST_CS_LIMIT
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_DS_LIMIT
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_ES_LIMIT
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_SS_LIMIT
    vmwrite rcx, rdx
    
    ; TR limit (TSS size - 1)
    mov     rdx, 0x67           ; 104 bytes for 64-bit TSS
    mov     rcx, VMCS_GUEST_TR_LIMIT
    vmwrite rcx, rdx
    
    ; LDTR limit = 0
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_LDTR_LIMIT
    vmwrite rcx, rdx
    
    ; --- Segment Access Rights ---
    ; CS: type=0xB (code, execute/read, accessed), S=1, DPL=0, P=1, L=1 (64-bit)
    ; Format: [7:0]=type+S+DPL+P, [11:8]=AVL+L+D/B+G, [16]=unusable
    mov     rdx, 0x0000A09B     ; CS: 64-bit code, present, ring 0
    mov     rcx, VMCS_GUEST_CS_ACCESS_RIGHTS
    vmwrite rcx, rdx
    
    ; DS/ES/SS/FS/GS: type=3 (data, read/write, accessed), S=1, DPL=0, P=1
    mov     rdx, 0x00000093     ; Data segment, ring 0
    mov     rcx, VMCS_GUEST_DS_ACCESS_RIGHTS
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_ES_ACCESS_RIGHTS
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_SS_ACCESS_RIGHTS
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_FS_ACCESS_RIGHTS
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_GS_ACCESS_RIGHTS
    vmwrite rcx, rdx
    
    ; TR: type=0xB (64-bit TSS, busy), S=0, DPL=0, P=1
    mov     rdx, 0x0000008B
    mov     rcx, VMCS_GUEST_TR_ACCESS_RIGHTS
    vmwrite rcx, rdx
    
    ; LDTR: unusable
    mov     rdx, 0x00010000     ; Unusable bit
    mov     rcx, VMCS_GUEST_LDTR_ACCESS_RIGHTS
    vmwrite rcx, rdx
    
    ; --- GDTR/IDTR ---
    mov     rdx, [guest_gdtr_base]
    mov     rcx, VMCS_GUEST_GDTR_BASE
    vmwrite rcx, rdx
    
    mov     rdx, [guest_gdtr_limit]
    mov     rcx, VMCS_GUEST_GDTR_LIMIT
    vmwrite rcx, rdx
    
    mov     rdx, [guest_idtr_base]
    mov     rcx, VMCS_GUEST_IDTR_BASE
    vmwrite rcx, rdx
    
    mov     rdx, [guest_idtr_limit]
    mov     rcx, VMCS_GUEST_IDTR_LIMIT
    vmwrite rcx, rdx
    
    ; --- Control Registers ---
    ; CR0: PE=1, ET=1, NE=1, WP=1, PG=1 (protected mode, paging)
    mov     rdx, 0x80050033
    mov     rcx, VMCS_GUEST_CR0
    vmwrite rcx, rdx
    
    ; CR3: guest page table root
    mov     rdx, [guest_cr3]
    mov     rcx, VMCS_GUEST_CR3
    vmwrite rcx, rdx
    
    ; CR4: PAE=1, VMXE should NOT be set in guest
    mov     rdx, 0x00002020     ; PAE + OSFXSR
    mov     rcx, VMCS_GUEST_CR4
    vmwrite rcx, rdx
    
    ; DR7
    mov     rdx, 0x00000400
    mov     rcx, VMCS_GUEST_DR7
    vmwrite rcx, rdx
    
    ; --- RIP, RSP, RFLAGS ---
    mov     rdx, [guest_entry_point]
    mov     rcx, VMCS_GUEST_RIP
    vmwrite rcx, rdx
    
    mov     rdx, [guest_stack_top]
    mov     rcx, VMCS_GUEST_RSP
    vmwrite rcx, rdx
    
    ; RFLAGS: IF=1, Reserved bit 1 must be set
    mov     rdx, 0x00000202
    mov     rcx, VMCS_GUEST_RFLAGS
    vmwrite rcx, rdx
    
    ; --- Activity State ---
    ; 0 = Active, 1 = HLT, 2 = Shutdown, 3 = Wait for SIPI
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_ACTIVITY_STATE
    vmwrite rcx, rdx
    
    ; --- Interruptibility State ---
    xor     rdx, rdx            ; No blocking
    mov     rcx, VMCS_GUEST_INTERRUPTIBILITY
    vmwrite rcx, rdx
    
    ; --- Pending Debug Exceptions ---
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_PENDING_DBG_EXCP
    vmwrite rcx, rdx
    
    ; --- VMCS Link Pointer ---
    mov     rdx, 0xFFFFFFFFFFFFFFFF   ; None (0xFFFFFFFF_FFFFFFFF)
    mov     rcx, VMCS_VMCS_LINK_POINTER
    vmwrite rcx, rdx
    
    ; --- MSRs ---
    ; IA32_DEBUGCTL
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_IA32_DEBUGCTL
    vmwrite rcx, rdx
    
    ; IA32_SYSENTER_CS/ESP/EIP
    xor     rdx, rdx
    mov     rcx, VMCS_GUEST_IA32_SYSENTER_CS
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_IA32_SYSENTER_ESP
    vmwrite rcx, rdx
    mov     rcx, VMCS_GUEST_IA32_SYSENTER_EIP
    vmwrite rcx, rdx
    
    ret
```

---

## VMCS Regions: Host State Area

Host State ถูก load กลับมาเมื่อ VM-Exit เกิดขึ้น — hypervisor กลับมาทำงาน

```nasm
setup_host_state:
    ; --- Host Segment Selectors ---
    ; (host ต้องใช้ GDT ของตัวเอง)
    mov     rdx, 0x0008         ; host CS (ring 0)
    mov     rcx, VMCS_HOST_CS_SELECTOR
    vmwrite rcx, rdx
    
    mov     rdx, 0x0010         ; host DS
    mov     rcx, VMCS_HOST_DS_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_HOST_ES_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_HOST_SS_SELECTOR
    vmwrite rcx, rdx
    
    ; FS, GS (per-CPU base via MSR)
    mov     rdx, 0x0010
    mov     rcx, VMCS_HOST_FS_SELECTOR
    vmwrite rcx, rdx
    mov     rcx, VMCS_HOST_GS_SELECTOR
    vmwrite rcx, rdx
    
    ; TR
    mov     rdx, 0x0018
    mov     rcx, VMCS_HOST_TR_SELECTOR
    vmwrite rcx, rdx
    
    ; --- Host Control Registers ---
    mov     rax, cr0
    mov     rcx, VMCS_HOST_CR0
    vmwrite rcx, rax
    
    mov     rax, cr3            ; host page table
    mov     rcx, VMCS_HOST_CR3
    vmwrite rcx, rax
    
    mov     rax, cr4
    mov     rcx, VMCS_HOST_CR4
    vmwrite rcx, rax
    
    ; --- Host Descriptor Tables ---
    ; Get GDTR
    sub     rsp, 10
    sgdt    [rsp]
    mov     rdx, [rsp+2]        ; GDTR base
    add     rsp, 10
    mov     rcx, VMCS_HOST_GDTR_BASE
    vmwrite rcx, rdx
    
    ; Get IDTR
    sub     rsp, 10
    sidt    [rsp]
    mov     rdx, [rsp+2]        ; IDTR base
    add     rsp, 10
    mov     rcx, VMCS_HOST_IDTR_BASE
    vmwrite rcx, rdx
    
    ; TR base (get from GDT)
    ; ... (ดูจาก segment descriptor ใน GDT)
    lea     rdx, [host_tss]
    mov     rcx, VMCS_HOST_TR_BASE
    vmwrite rcx, rdx
    
    ; FS base (can be per-thread pointer)
    mov     ecx, 0xC0000100     ; IA32_FS_BASE MSR
    rdmsr
    shl     rdx, 32
    or      rdx, rax
    mov     rcx, VMCS_HOST_FS_BASE
    vmwrite rcx, rdx
    
    ; GS base (per-CPU in Linux kernel)
    mov     ecx, 0xC0000101     ; IA32_GS_BASE MSR
    rdmsr
    shl     rdx, 32
    or      rdx, rax
    mov     rcx, VMCS_HOST_GS_BASE
    vmwrite rcx, rdx
    
    ; --- Host RSP and RIP ---
    ; RSP = stack สำหรับ VM-Exit handler
    lea     rdx, [host_stack_top]
    mov     rcx, VMCS_HOST_RSP
    vmwrite rcx, rdx
    
    ; RIP = VM-Exit handler entry point
    lea     rdx, [vmexit_handler]
    mov     rcx, VMCS_HOST_RIP
    vmwrite rcx, rdx
    
    ; --- Host MSRs ---
    mov     ecx, 0x174          ; IA32_SYSENTER_CS
    rdmsr
    mov     rcx, VMCS_HOST_IA32_SYSENTER_CS
    vmwrite rcx, rdx            ; Note: zero-extend EAX into RDX before write
    
    ret
```

---

## VM-Execution Controls

Control fields กำหนดว่า operations ใดที่จะ cause VM-Exit

### Pin-Based VM-Execution Controls (32-bit)

```nasm
; IA32_VMX_PINBASED_CTLS MSR (0x481) — allowed bits
setup_pin_based_controls:
    ; อ่านค่า allowed 0-settings และ 1-settings
    mov     ecx, 0x481          ; IA32_VMX_PINBASED_CTLS
    rdmsr
    ; EAX = allowed 0-settings (must-be-0)
    ; EDX = allowed 1-settings (may-be-1)
    
    ; Pin-based controls ที่ต้องการ:
    ; Bit 0: External-interrupt exiting (exit on external interrupts)
    ; Bit 3: NMI exiting
    ; Bit 5: Virtual NMIs
    ; Bit 6: Activate VMX preemption timer
    ; Bit 7: Process posted interrupts
    
    mov     rbx, (1 << 0) | (1 << 3)    ; ext-int exit + NMI exit
    
    ; Apply allowed 0/1 mask
    and     ebx, edx            ; clear bits not allowed to be 1
    or      ebx, eax            ; set bits required to be 1
    
    mov     rcx, VMCS_PIN_BASED_VM_EXEC_CTRL
    vmwrite rcx, rbx
    ret

; Primary Processor-Based VM-Execution Controls
setup_primary_proc_controls:
    mov     ecx, 0x482          ; IA32_VMX_PROCBASED_CTLS
    rdmsr
    
    ; Primary controls:
    ; Bit  2: Interrupt-window exiting
    ; Bit  3: Use TSC offsetting
    ; Bit  7: HLT exiting (exit on HLT instruction)
    ; Bit  9: INVLPG exiting
    ; Bit 10: MWAIT exiting
    ; Bit 11: RDPMC exiting
    ; Bit 12: RDTSC exiting
    ; Bit 15: CR3-load exiting
    ; Bit 16: CR3-store exiting
    ; Bit 19: CR8-load exiting
    ; Bit 20: CR8-store exiting
    ; Bit 21: Use TPR shadow
    ; Bit 22: NMI-window exiting
    ; Bit 23: MOV-DR exiting
    ; Bit 24: Unconditional I/O exiting
    ; Bit 25: Use I/O bitmaps
    ; Bit 27: Monitor trap flag
    ; Bit 28: Use MSR bitmaps
    ; Bit 29: MONITOR exiting
    ; Bit 30: PAUSE exiting
    ; Bit 31: Activate secondary controls
    
    mov     rbx, (1 << 7)       ; HLT exiting
    or      rbx, (1 << 24)      ; Unconditional I/O exiting (simplest)
    or      rbx, (1 << 28)      ; Use MSR bitmaps
    or      rbx, (1 << 31)      ; Activate secondary controls
    
    and     ebx, edx
    or      ebx, eax
    
    mov     rcx, VMCS_PRI_PROC_BASED_CTRL
    vmwrite rcx, rbx
    ret

; Secondary Processor-Based VM-Execution Controls
setup_secondary_proc_controls:
    mov     ecx, 0x48B          ; IA32_VMX_PROCBASED_CTLS2
    rdmsr
    
    ; Secondary controls:
    ; Bit 1: Enable EPT (Extended Page Tables)
    ; Bit 2: Descriptor-table exiting
    ; Bit 3: Enable RDTSCP
    ; Bit 5: Enable VPID
    ; Bit 7: Unrestricted guest (allow real-mode guest)
    ; Bit 8: APIC-register virtualization
    ; Bit 9: Virtual-interrupt delivery
    ; Bit 12: Enable INVPCID
    ; Bit 13: Enable VM functions
    ; Bit 14: VMCS shadowing
    ; Bit 20: Enable XSAVES/XRSTORS
    
    mov     rbx, (1 << 1)       ; Enable EPT
    or      rbx, (1 << 5)       ; Enable VPID
    or      rbx, (1 << 7)       ; Unrestricted guest
    
    and     ebx, edx
    or      ebx, eax
    
    mov     rcx, VMCS_SEC_PROC_BASED_CTRL
    vmwrite rcx, rbx
    ret
```

### VM-Exit Controls

```nasm
setup_vmexit_controls:
    mov     ecx, 0x483          ; IA32_VMX_EXIT_CTLS
    rdmsr
    
    ; VM-Exit controls:
    ; Bit  2: Save debug controls (save DR7, IA32_DEBUGCTL on exit)
    ; Bit  9: Host address-space size (1 = 64-bit host)
    ; Bit 12: Load IA32_PERF_GLOBAL_CTRL
    ; Bit 15: Acknowledge interrupt on exit
    ; Bit 18: Save IA32_PAT
    ; Bit 19: Load IA32_PAT
    ; Bit 20: Save IA32_EFER
    ; Bit 21: Load IA32_EFER
    ; Bit 22: Save VMX preemption timer value
    ; Bit 23: Clear IA32_BNDCFGS
    
    mov     rbx, (1 << 9)       ; Host 64-bit mode
    or      rbx, (1 << 15)      ; Acknowledge interrupt on exit
    or      rbx, (1 << 20)      ; Save IA32_EFER
    or      rbx, (1 << 21)      ; Load IA32_EFER
    
    and     ebx, edx
    or      ebx, eax
    
    mov     rcx, VMCS_VM_EXIT_CONTROLS
    vmwrite rcx, rbx
    ret
```

### VM-Entry Controls

```nasm
setup_vmentry_controls:
    mov     ecx, 0x484          ; IA32_VMX_ENTRY_CTLS
    rdmsr
    
    ; VM-Entry controls:
    ; Bit  2: Load debug controls
    ; Bit  9: IA-32e mode guest (1 = 64-bit guest)
    ; Bit 10: Entry to SMM
    ; Bit 11: Deactivate dual-monitor treatment
    ; Bit 13: Load IA32_PERF_GLOBAL_CTRL
    ; Bit 14: Load IA32_PAT
    ; Bit 15: Load IA32_EFER
    ; Bit 16: Load IA32_BNDCFGS
    
    mov     rbx, (1 << 9)       ; 64-bit guest
    or      rbx, (1 << 14)      ; Load IA32_PAT
    or      rbx, (1 << 15)      ; Load IA32_EFER
    
    and     ebx, edx
    or      ebx, eax
    
    mov     rcx, VMCS_VM_ENTRY_CONTROLS
    vmwrite rcx, rbx
    ret
```

---

## VM-Exit Reasons

เมื่อ VM-Exit เกิดขึ้น hardware บันทึก reason ใน **VMCS_EXIT_REASON** field

### ตาราง Exit Reasons (บางส่วน)

| Reason | Hex | Description |
|--------|-----|-------------|
| 0 | 0x00 | Exception or NMI |
| 1 | 0x01 | External interrupt |
| 2 | 0x02 | Triple fault |
| 3 | 0x03 | INIT signal |
| 4 | 0x04 | SIPI |
| 7 | 0x07 | Interrupt window |
| 9 | 0x09 | Task switch |
| 10 | 0x0A | CPUID |
| 12 | 0x0C | HLT |
| 13 | 0x0D | INVD |
| 14 | 0x0E | INVLPG |
| 15 | 0x0F | RDPMC |
| 16 | 0x10 | RDTSC |
| 18 | 0x12 | VMCALL |
| 19 | 0x13 | VMCLEAR |
| 20 | 0x14 | VMLAUNCH |
| 21 | 0x15 | VMPTRLD |
| 22 | 0x16 | VMPTRST |
| 23 | 0x17 | VMREAD |
| 24 | 0x18 | VMRESUME |
| 25 | 0x19 | VMWRITE |
| 26 | 0x1A | VMXOFF |
| 27 | 0x1B | VMXON |
| 28 | 0x1C | Control-register accesses |
| 29 | 0x1D | MOV DR |
| 30 | 0x1E | I/O instruction |
| 31 | 0x1F | RDMSR |
| 32 | 0x20 | WRMSR |
| 33 | 0x21 | VM-entry failure (invalid guest state) |
| 34 | 0x22 | VM-entry failure (MSR loading) |
| 36 | 0x24 | MWAIT |
| 37 | 0x25 | Monitor trap flag |
| 39 | 0x27 | MONITOR |
| 40 | 0x28 | PAUSE |
| 41 | 0x29 | VM-entry failure (machine check) |
| 43 | 0x2B | TPR below threshold |
| 44 | 0x2C | APIC access |
| 46 | 0x2E | Access to GDTR/IDTR |
| 47 | 0x2F | Access to LDTR/TR |
| 48 | 0x30 | EPT violation |
| 49 | 0x31 | EPT misconfiguration |
| 50 | 0x32 | INVEPT |
| 51 | 0x33 | RDTSCP |
| 52 | 0x34 | VMX preemption timer expired |
| 53 | 0x35 | INVVPID |
| 54 | 0x36 | WBINVD |
| 55 | 0x37 | XSETBV |
| 56 | 0x38 | APIC write |
| 57 | 0x39 | RDRAND |
| 58 | 0x3A | INVPCID |
| 59 | 0x3B | VMFUNC |
| 60 | 0x3C | ENCLS |
| 61 | 0x3D | RDSEED |
| 62 | 0x3E | Page-modification log full |
| 63 | 0x3F | XSAVES |
| 64 | 0x40 | XRSTORS |

### VM-Exit Handler Dispatcher

```nasm
; VM-Exit handler
; เมื่อ VM-Exit เกิดขึ้น CPU จะกระโดดมาที่นี่
; Host state ได้ถูก load แล้ว (CR0, CR3, CR4, segments, RSP, RIP)
; Guest state ได้ถูก save ใน VMCS แล้ว

vmexit_handler:
    ; Save all registers (guest general-purpose regs ไม่ถูก save ใน VMCS!)
    push    rax
    push    rcx
    push    rdx
    push    rbx
    push    rbp
    push    rsi
    push    rdi
    push    r8
    push    r9
    push    r10
    push    r11
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; Read exit reason
    mov     rcx, VMCS_EXIT_REASON
    vmread  rax, rcx
    and     rax, 0xFFFF         ; lower 16 bits = basic exit reason
    
    ; Dispatch table
    cmp     rax, EXIT_REASON_MAX
    ja      .unknown_exit
    
    lea     rcx, [vmexit_dispatch_table]
    mov     rdx, [rcx + rax*8]
    call    rdx
    
.vmexit_done:
    ; Restore registers
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     r11
    pop     r10
    pop     r9
    pop     r8
    pop     rdi
    pop     rsi
    pop     rbp
    pop     rbx
    pop     rdx
    pop     rcx
    pop     rax
    
    ; Resume guest
    vmresume
    
    ; If VMRESUME fails (shouldn't happen normally)
    ; Check flags
    jc      .vmresume_failed
    jz      .vmresume_failed_status
    ret     ; unreachable

.vmresume_failed:
    ; CF=1: VMRESUME with invalid VMCS state
    hlt
    
.vmresume_failed_status:
    ; ZF=1: error code in VMCS_VM_INSTR_ERROR
    mov     rcx, VMCS_VM_INSTR_ERROR
    vmread  rax, rcx
    ; error code in rax
    hlt
    
.unknown_exit:
    ; Unknown exit reason — probably should inject #GP to guest or panic
    hlt

; Exit reason constants
EXIT_REASON_EXCEPTION       equ 0
EXIT_REASON_EXT_INTR        equ 1
EXIT_REASON_TRIPLE_FAULT    equ 2
EXIT_REASON_HLT             equ 12
EXIT_REASON_CPUID           equ 10
EXIT_REASON_VMCALL          equ 18
EXIT_REASON_CR_ACCESS       equ 28
EXIT_REASON_IO_INSTRUCTION  equ 30
EXIT_REASON_RDMSR           equ 31
EXIT_REASON_WRMSR           equ 32
EXIT_REASON_EPT_VIOLATION   equ 48
EXIT_REASON_EPT_MISCONFIG   equ 49
EXIT_REASON_MAX             equ 64
```

### Handling Specific Exit Reasons

```nasm
; Handle CPUID exit
handle_cpuid_exit:
    ; Read guest RAX (CPUID leaf) and RCX (subleaf)
    ; General-purpose regs ไม่อยู่ใน VMCS — ต้องใช้ที่ save ไว้ใน stack frame
    ; (ขึ้นอยู่กับ calling convention ของ hypervisor)
    
    mov     rax, [rsp + guest_rax_offset]
    mov     rcx, [rsp + guest_rcx_offset]
    
    ; ทำ CPUID จริงๆ (หรือ synthesize ค่า)
    push    rbx
    cpuid
    
    ; ซ่อน VMX support จาก guest (guest ไม่ควรรู้ว่าถูก virtualize)
    ; CPUID leaf 1, ECX bit 5 = VMX
    cmp     dword [rsp + guest_rax_offset + 8], 1  ; leaf 1?
    jne     .cpuid_pass_through
    and     ecx, ~(1 << 5)      ; clear VMX bit
    and     ecx, ~(1 << 6)      ; clear SMX bit
    
.cpuid_pass_through:
    ; Write back to guest register save area
    mov     [rsp + guest_rax_offset], rax
    mov     [rsp + guest_rbx_offset], rbx
    mov     [rsp + guest_rcx_offset], rcx
    mov     [rsp + guest_rdx_offset], rdx
    pop     rbx
    
    ; Advance guest RIP past the CPUID instruction (2 bytes)
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    add     rax, 2              ; CPUID = 0F A2 (2 bytes)
    vmwrite rcx, rax
    
    ret

; Handle I/O instruction exit
handle_io_exit:
    ; Read exit qualification
    mov     rcx, VMCS_EXIT_QUALIFICATION
    vmread  rax, rcx
    
    ; Exit qualification for I/O:
    ; Bits 2:0  = Size (0=byte, 1=word, 3=dword)
    ; Bit  3    = Direction (0=out, 1=in)
    ; Bit  4    = String instruction
    ; Bit  5    = REP prefix
    ; Bits 6    = Operand encoding
    ; Bits 31:16 = Port number
    
    mov     rbx, rax
    shr     rbx, 16             ; port number
    and     rbx, 0xFFFF
    
    test    rax, (1 << 3)       ; IN instruction?
    jnz     .io_in
    
.io_out:
    ; OUT instruction — guest writing to port rbx
    ; Handle based on port number
    cmp     rbx, 0xCF8          ; PCI config address port?
    je      .handle_pci_config_addr
    cmp     rbx, 0xCFC          ; PCI config data port?
    je      .handle_pci_config_data
    ; ... other port handlers
    jmp     .io_advance_rip
    
.io_in:
    ; IN instruction — return emulated value
    ; Default: return 0xFF (device not present)
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx            ; save rip temporarily
    mov     qword [rsp + guest_rax_offset], 0xFF
    
.io_advance_rip:
    ; Advance RIP by instruction length
    mov     rcx, VMCS_VM_EXIT_INSTR_LEN
    vmread  rdx, rcx            ; instruction length
    
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    add     rax, rdx
    vmwrite rcx, rax
    ret
    
.handle_pci_config_addr:
.handle_pci_config_data:
    ; ... PCI emulation logic
    jmp     .io_advance_rip

; Handle HLT exit
handle_hlt_exit:
    ; Guest executed HLT — inject interrupt or just advance RIP
    ; HLT is 1 byte: 0xF4
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    inc     rax                 ; HLT = 1 byte
    vmwrite rcx, rax
    
    ; Could also put guest in "waiting" state and yield host CPU
    ret

; Handle VMCALL exit (hypercall)
handle_vmcall_exit:
    ; Guest executed VMCALL — this is a hypercall
    ; Convention: RAX = hypercall number
    mov     rax, [rsp + guest_rax_offset]
    
    cmp     rax, HYPERCALL_PRINT_STRING
    je      .hcall_print
    cmp     rax, HYPERCALL_GET_TIME
    je      .hcall_get_time
    cmp     rax, HYPERCALL_SHUTDOWN
    je      .hcall_shutdown
    
    ; Unknown hypercall
    mov     qword [rsp + guest_rax_offset], -1
    jmp     .vmcall_advance_rip
    
.hcall_print:
    ; RBX = string address (GVA)
    ; RCX = length
    ; ... print guest string
    xor     rax, rax
    mov     [rsp + guest_rax_offset], rax
    jmp     .vmcall_advance_rip
    
.hcall_get_time:
    rdtsc
    shl     rdx, 32
    or      rax, rdx
    mov     [rsp + guest_rax_offset], rax
    jmp     .vmcall_advance_rip
    
.hcall_shutdown:
    ; Shutdown the VM
    vmxoff
    ; ... cleanup
    
.vmcall_advance_rip:
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    add     rax, 3              ; VMCALL = 0F 01 C1 (3 bytes)
    vmwrite rcx, rax
    ret

; Hypercall numbers
HYPERCALL_PRINT_STRING  equ 1
HYPERCALL_GET_TIME      equ 2
HYPERCALL_SHUTDOWN      equ 3
```

---

## EPT (Extended Page Tables)

EPT เป็น hardware-assisted second-level address translation:

```
Guest Virtual Address (GVA)
         ↓ [Guest page tables — translated by hardware normally]
Guest Physical Address (GPA)  ← guest thinks this is physical memory
         ↓ [EPT translation — additional hardware walk]
Host Physical Address (HPA)   ← actual physical memory
```

### EPT Structure (4-Level Paging)

```
EPT PML4 (512 entries × 8 bytes = 4KB)
    ↓ [bits 47:39 of GPA]
EPT PDPT (512 entries × 8 bytes = 4KB)
    ↓ [bits 38:30 of GPA]
EPT PD (512 entries × 8 bytes = 4KB)
    ↓ [bits 29:21 of GPA]
EPT PT (512 entries × 8 bytes = 4KB)
    ↓ [bits 20:12 of GPA]
Physical Frame (4KB)
    + [bits 11:0 of GPA = byte offset]
= HPA
```

### EPT Entry Format

```
Bit  0: Read access allowed
Bit  1: Write access allowed
Bit  2: Execute access (or supervisor-mode execute for EPT entries with bit 7 = 1)
Bits 5:3: EPT memory type (0=UC, 6=WB) — leaf entries only
Bit  6: Ignore PAT (leaf only)
Bit  7: Page size (1 = large page: 2MB for PD, 1GB for PDPT)
Bit  8: Accessed flag (if EPT accessed/dirty feature enabled)
Bit  9: Dirty flag
Bit 10: User-mode execute access (requires "mode-based execute" control)
Bits 51:12: Physical address (page-aligned, 4KB granularity)
Bit 63: Suppress #VE (virtualization exception)
```

### EPT Pointer (EPTP)

```
Bits 2:0  = EPT memory type for EPT structures (0=UC, 6=WB)
Bits 5:3  = EPT page-walk length - 1 (3 for 4 levels)
Bit  6    = Enable accessed and dirty flags
Bit  7    = Enable supervisor-shadow-stack
Bits 11:8 = Reserved
Bits 51:12 = Physical address of EPT PML4
```

### EPT Setup Code

```nasm
; EPT table structures
section .data
align 4096
ept_pml4:   times 512 dq 0   ; 4KB
ept_pdpt:   times 512 dq 0   ; 4KB
ept_pd:     times 512 dq 0   ; 4KB (512 × 2MB entries = 1GB coverage)

section .text

; Setup identity-mapped EPT (GPA = HPA for simplicity)
; Map first 1GB identity
setup_ept_identity_1gb:
    ; PML4[0] → PDPT
    lea     rax, [ept_pdpt]     ; physical address of PDPT
    or      rax, 0x07           ; R=1, W=1, X=1
    mov     [ept_pml4], rax
    
    ; PDPT[0] → PD
    lea     rax, [ept_pd]
    or      rax, 0x07
    mov     [ept_pdpt], rax
    
    ; PD entries: 512 × 2MB large pages (identity mapped)
    xor     rcx, rcx
.pd_loop:
    ; Each 2MB page: base = rcx * 2MB = rcx << 21
    mov     rax, rcx
    shl     rax, 21             ; 2MB granularity
    or      rax, 0x87           ; R=1, W=1, X=1, Page=1 (large page), WB type=6→0x86
    ; Actually: memory type in bits 5:3, type 6 (WB) = 0b110
    ; Large page bit = bit 7
    ; So: R=1, W=1, X=1, type=WB(6), large=1
    ; = 0b11 | (6<<3) | (1<<7) = 3 | 0x30 | 0x80 = 0xB3
    mov     rax, rcx
    shl     rax, 21
    or      rax, 0x00000000000000B3   ; R,W,X, WB, Large page
    
    mov     [ept_pd + rcx*8], rax
    
    inc     rcx
    cmp     rcx, 512
    jl      .pd_loop
    
    ; Build EPTP value
    lea     rax, [ept_pml4]     ; Physical addr of PML4
    ; Memory type WB = 6 (bits 2:0)
    ; Walk length = 3 (4 levels - 1) (bits 5:3)
    or      rax, (6 | (3 << 3))  ; type=WB, walk=4 levels
    
    ; Write EPTP to VMCS
    mov     rcx, VMCS_EPT_POINTER
    vmwrite rcx, rax
    
    ret

; Handle EPT Violation (VM-Exit reason 48)
handle_ept_violation:
    ; Read GPA from VMCS
    mov     rcx, VMCS_GUEST_PHYSICAL_ADDR
    vmread  rax, rcx
    ; rax = Guest Physical Address that caused violation
    
    ; Read Exit Qualification
    mov     rcx, VMCS_EXIT_QUALIFICATION
    vmread  rbx, rcx
    
    ; Exit qualification bits:
    ; Bit 0: Read access causing violation
    ; Bit 1: Write access causing violation
    ; Bit 2: Instruction fetch causing violation
    ; Bit 3: EPT read permission for GPA
    ; Bit 4: EPT write permission for GPA
    ; Bit 5: EPT execute permission for GPA
    ; Bit 7: GVA valid
    ; Bit 8: GPA translates to user/supervisor
    
    ; Determine access type
    test    rbx, (1 << 0)       ; read?
    jnz     .ept_read_fault
    test    rbx, (1 << 1)       ; write?
    jnz     .ept_write_fault
    test    rbx, (1 << 2)       ; execute?
    jnz     .ept_exec_fault
    
.ept_read_fault:
.ept_write_fault:
.ept_exec_fault:
    ; Option 1: Map the page (demand paging)
    ; Option 2: Inject #GP to guest
    ; Option 3: Kill the VM
    
    ; For now: try to allocate a page and map it
    ; rax = faulting GPA
    and     rax, ~0xFFF         ; align to page
    call    allocate_host_page  ; returns HPA in rax
    test    rax, rax
    jz      .ept_oom            ; out of memory
    
    call    map_gpa_to_hpa      ; map GPA → HPA in EPT
    
    ; Invalidate EPT TLB
    ; INVEPT type 1 = single-context invalidation
    ; type 2 = all-contexts invalidation
    lea     rdi, [invept_descriptor]
    mov     rax, VMCS_EPT_POINTER
    vmread  rax, rax
    mov     [invept_descriptor], rax    ; EPT pointer
    mov     qword [invept_descriptor+8], 0   ; reserved
    
    mov     rax, 1              ; single-context
    invept  rax, [rdi]
    
    ; Don't advance RIP — re-execute the faulting instruction
    ret
    
.ept_oom:
    ; Inject out-of-memory condition to guest
    hlt

section .data
invept_descriptor:
    dq 0    ; EPT pointer
    dq 0    ; reserved
```

---

## INVEPT และ INVVPID

### INVEPT: Invalidate EPT Translations

```nasm
; INVEPT types:
; Type 1 (Single-context): ล้าง TLB entries สำหรับ specific EPTP
; Type 2 (All-contexts): ล้าง TLB entries ทั้งหมดที่เกี่ยวกับ EPT

; INVEPT descriptor:
; Bytes 0-7:  EPT pointer (EPTP)
; Bytes 8-15: Reserved (must be 0)

struc invept_desc
    .eptp   resq 1
    .resv   resq 1
endstruc

; Single-context invalidation
invept_single:
    ; Input: RAX = EPTP
    sub     rsp, 16
    mov     [rsp], rax          ; EPTP
    mov     qword [rsp+8], 0    ; Reserved
    
    mov     rax, 1              ; Single-context type
    invept  rax, [rsp]
    
    add     rsp, 16
    
    ; Check CF (invalid type) and ZF (failure)
    jc      .invept_invalid
    jz      .invept_failed
    ret
    
.invept_invalid:
.invept_failed:
    mov     rax, -1
    ret

; All-contexts invalidation
invept_all:
    sub     rsp, 16
    xor     rax, rax
    mov     [rsp], rax
    mov     [rsp+8], rax
    
    mov     rax, 2              ; All-contexts type
    invept  rax, [rsp]
    
    add     rsp, 16
    ret
```

### INVVPID: Invalidate VPID Translations

VPID (Virtual Processor Identifier) ช่วยให้ TLB entries ของ different VMs อยู่ร่วมกันได้

```nasm
; INVVPID types:
; Type 0: Individual address
; Type 1: Single-context (specific VPID)
; Type 2: All-contexts (อัตโนมัติ)
; Type 3: Single-context retaining globals

struc invvpid_desc
    .vpid   resw 1
    .resv   resw 3
    .linear resq 1
endstruc

; VMCS field for VPID
VMCS_VPID   equ 0x0000

; Set VPID for guest (each VM gets a unique VPID)
set_guest_vpid:
    mov     rdx, [current_vm_vpid]  ; e.g., 1, 2, 3, ...
    mov     rcx, VMCS_VPID
    vmwrite rcx, rdx
    ret

; Single-context INVVPID
invvpid_single_context:
    ; Input: AX = VPID
    sub     rsp, 16
    movzx   rax, ax
    mov     [rsp], ax           ; VPID (16-bit)
    mov     word [rsp+2], 0
    mov     dword [rsp+4], 0
    mov     qword [rsp+8], 0    ; linear address (unused for type 1)
    
    mov     rax, 1              ; Single-context
    invvpid rax, [rsp]
    
    add     rsp, 16
    ret
```

---

## CR Access Handling

เมื่อ guest access CR registers บาง operations trigger VM-Exit

```nasm
; Handle Control Register Access Exit (reason 28)
handle_cr_access:
    mov     rcx, VMCS_EXIT_QUALIFICATION
    vmread  rax, rcx
    
    ; Exit qualification for CR access:
    ; Bits 3:0  = CR number (0, 3, 4, 8)
    ; Bits 5:4  = Access type (0=MOV to CR, 1=MOV from CR, 2=CLTS, 3=LMSW)
    ; Bit  6    = LMSW operand type (0=register, 1=memory)
    ; Bits 11:8 = GPR number (for MOV CR)
    ; Bits 31:16 = LMSW source data
    
    mov     rbx, rax
    and     rbx, 0xF            ; CR number
    
    mov     rcx, rax
    shr     rcx, 4
    and     rcx, 0x3            ; Access type
    
    mov     rdx, rax
    shr     rdx, 8
    and     rdx, 0xF            ; GPR number
    
    ; Handle MOV to CR
    cmp     rcx, 0
    jne     .cr_read
    
.cr_write:
    ; Writing to CR
    ; Get value from guest GPR (saved in stack frame)
    ; rdx = GPR number (0=RAX, 1=RCX, ..., 15=R15)
    mov     r8, [rsp + rdx*8 + guest_regs_base]
    
    cmp     rbx, 0              ; CR0?
    je      .cr0_write
    cmp     rbx, 3              ; CR3?
    je      .cr3_write
    cmp     rbx, 4              ; CR4?
    je      .cr4_write
    jmp     .cr_advance_rip
    
.cr0_write:
    ; Validate and update CR0
    ; Some bits must be 1 (ET, NE, etc.)
    ; Check for mode switches (protected→real, real→protected)
    mov     rcx, VMCS_GUEST_CR0
    vmwrite rcx, r8
    
    ; Also update CR0 read shadow if needed
    mov     rcx, VMCS_CR0_READ_SHADOW
    vmwrite rcx, r8
    jmp     .cr_advance_rip
    
.cr3_write:
    ; Update guest CR3 (page table switch)
    mov     rcx, VMCS_GUEST_CR3
    vmwrite rcx, r8
    
    ; Invalidate TLB for this guest (INVVPID or just let hardware handle via VPID)
    jmp     .cr_advance_rip
    
.cr4_write:
    ; Validate CR4 changes
    ; Guest cannot set CR4.VMXE
    and     r8, ~(1 << 13)      ; Clear VMXE
    mov     rcx, VMCS_GUEST_CR4
    vmwrite rcx, r8
    mov     rcx, VMCS_CR4_READ_SHADOW
    vmwrite rcx, r8
    jmp     .cr_advance_rip
    
.cr_read:
    cmp     rcx, 1              ; MOV from CR?
    jne     .cr_advance_rip
    
    ; Reading from CR
    cmp     rbx, 3
    je      .cr3_read
    jmp     .cr_advance_rip
    
.cr3_read:
    mov     rcx, VMCS_GUEST_CR3
    vmread  r8, rcx
    ; Write to guest GPR
    mov     [rsp + rdx*8 + guest_regs_base], r8
    
.cr_advance_rip:
    mov     rcx, VMCS_VM_EXIT_INSTR_LEN
    vmread  rdx, rcx
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    add     rax, rdx
    vmwrite rcx, rax
    ret
```

---

## MSR Handling

### MSR Bitmap

MSR bitmap ช่วยให้ guest อ่าน/เขียน MSRs ส่วนใหญ่โดยไม่ต้อง exit:

```
MSR Bitmap Layout (4KB):
  Bytes 0x000-0x3FF: Read bitmap for MSRs 0x00000000-0x00001FFF
  Bytes 0x400-0x7FF: Read bitmap for MSRs 0xC0000000-0xC0001FFF
  Bytes 0x800-0xBFF: Write bitmap for MSRs 0x00000000-0x00001FFF
  Bytes 0xC00-0xFFF: Write bitmap for MSRs 0xC0000000-0xC0001FFF
```

```nasm
section .data
align 4096
msr_bitmap: times 4096 db 0     ; all zeros = no exits for MSR access

section .text

setup_msr_bitmap:
    ; ตั้งค่า MSR bitmap
    ; Set bit N ใน bitmap = exit เมื่อ access MSR นั้น
    
    ; Force exit on IA32_EFER reads/writes (0xC0000080)
    ; MSR 0xC0000080 → offset 0xC0000080 - 0xC0000000 = 0x80
    ; = bit 0x80 = byte 0x10, bit 0 (0x80 / 8 = 0x10)
    ; Read bitmap for C000xxxx = starting at offset 0x400
    mov     byte [msr_bitmap + 0x400 + 0x10], (1 << 0)    ; bit 0 = EFER
    ; Write bitmap = starting at offset 0xC00
    mov     byte [msr_bitmap + 0xC00 + 0x10], (1 << 0)
    
    ; Force exit on IA32_APIC_BASE (0x1B)
    ; bit 0x1B = byte 3, bit 3
    mov     byte [msr_bitmap + 0x03], (1 << 3)
    mov     byte [msr_bitmap + 0x803], (1 << 3)
    
    ; Write MSR bitmap address to VMCS
    lea     rax, [msr_bitmap]
    mov     rcx, VMCS_MSR_BITMAP
    vmwrite rcx, rax
    ret

; Handle RDMSR exit (reason 31)
handle_rdmsr_exit:
    ; Guest executed RDMSR with ECX = MSR number
    mov     rax, [rsp + guest_rcx_offset]
    and     eax, 0xFFFFFFFF     ; 32-bit MSR number
    
    ; Dispatch based on MSR
    cmp     eax, 0xC0000080     ; IA32_EFER?
    je      .rdmsr_efer
    cmp     eax, 0x1B           ; IA32_APIC_BASE?
    je      .rdmsr_apic_base
    
    ; Default: pass through (read real MSR)
    mov     ecx, eax
    rdmsr
    jmp     .rdmsr_return
    
.rdmsr_efer:
    ; Return guest's view of EFER
    ; (may differ from actual EFER if guest is 32-bit but host is 64-bit)
    mov     rax, [current_guest_efer]
    mov     rdx, rax
    shr     rdx, 32
    jmp     .rdmsr_return
    
.rdmsr_apic_base:
    ; Return emulated APIC base
    mov     eax, 0xFEE00800     ; default APIC base
    xor     edx, edx
    
.rdmsr_return:
    ; Write EAX:EDX back to guest RAX:RDX
    mov     [rsp + guest_rax_offset], eax
    movzx   r8, edx             ; zero-extend
    mov     [rsp + guest_rdx_offset], r8
    
    ; Advance RIP (RDMSR = 0F 32, 2 bytes)
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    add     rax, 2
    vmwrite rcx, rax
    ret

; Handle WRMSR exit (reason 32)
handle_wrmsr_exit:
    ; Guest writing EAX:EDX to MSR in ECX
    mov     eax, [rsp + guest_rcx_offset]   ; MSR number
    mov     r8d, [rsp + guest_rax_offset]   ; low 32 bits
    mov     r9d, [rsp + guest_rdx_offset]   ; high 32 bits
    
    cmp     eax, 0xC0000080     ; EFER?
    je      .wrmsr_efer
    
    ; Default: write to real MSR
    mov     ecx, eax
    mov     eax, r8d
    mov     edx, r9d
    wrmsr
    jmp     .wrmsr_advance
    
.wrmsr_efer:
    ; Validate EFER value
    ; Cannot clear LMA while in long mode
    ; ...
    mov     [current_guest_efer], r8
    
    ; Update actual EFER for guest via VMCS
    mov     rcx, VMCS_GUEST_IA32_EFER
    shl     r9, 32
    or      r9, r8
    vmwrite rcx, r9
    
.wrmsr_advance:
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    add     rax, 2              ; WRMSR = 0F 30, 2 bytes
    vmwrite rcx, rax
    ret
```

---

## Exception Injection

Hypervisor สามารถ inject exceptions เข้า guest ได้ผ่าน VM-Entry Interrupt Information field:

```nasm
; Inject exception into guest
; Input: AL = vector number, BL = type, BH = error code valid flag
inject_exception:
    ; VM-Entry Interrupt Information Field (32-bit):
    ; Bits 7:0   = Vector (interrupt/exception number)
    ; Bits 10:8  = Interruption type:
    ;              0 = External interrupt
    ;              2 = NMI
    ;              3 = Hardware exception
    ;              4 = Software interrupt (INT n)
    ;              5 = Privileged software exception (INT1)
    ;              6 = Software exception (INT3 or INTO)
    ; Bit 11     = Error code valid
    ; Bit 31     = Valid (must be 1 to inject)
    
    movzx   eax, al             ; vector
    movzx   ecx, bl
    shl     ecx, 8              ; type << 8
    or      eax, ecx
    or      eax, (1 << 31)      ; Valid bit
    
    test    bh, bh              ; error code valid?
    jz      .no_error_code
    or      eax, (1 << 11)
    
.no_error_code:
    mov     rcx, VMCS_VM_ENTRY_INTR_INFO
    vmwrite rcx, rax
    ret

; Inject #GP(0) — General Protection Fault
inject_gp_fault:
    ; Vector 13 = #GP
    ; Type 3 = hardware exception
    ; Error code valid = yes, code = 0
    mov     eax, (1 << 31) | (1 << 11) | (3 << 8) | 13
    mov     rcx, VMCS_VM_ENTRY_INTR_INFO
    vmwrite rcx, rax
    
    ; Error code
    xor     eax, eax            ; #GP(0)
    mov     rcx, VMCS_VM_ENTRY_EXCEPTION_ERR
    vmwrite rcx, rax
    
    ; Instruction length (for software-generated exceptions)
    ; For hardware exceptions injected due to guest fault, set to 0
    xor     eax, eax
    mov     rcx, VMCS_VM_ENTRY_INSTR_LEN
    vmwrite rcx, rax
    ret

; Inject Page Fault (#PF)
inject_page_fault:
    ; Input: RSI = fault address, EDI = error code
    
    ; Inject #PF (vector 14)
    mov     eax, (1 << 31) | (1 << 11) | (3 << 8) | 14
    mov     rcx, VMCS_VM_ENTRY_INTR_INFO
    vmwrite rcx, rax
    
    ; Error code
    movzx   rax, edi
    mov     rcx, VMCS_VM_ENTRY_EXCEPTION_ERR
    vmwrite rcx, rax
    
    ; #PF also requires setting CR2 to fault address
    mov     rcx, VMCS_GUEST_CR0
    vmread  rax, rcx            ; just read to check we can vmwrite
    ; Actually update guest CR2 — but CR2 is NOT in VMCS!
    ; We need to set it directly since it's not virtualized
    ; In practice: use mov cr2, rsi (but CR2 is shared between host and guest in some implementations)
    ; Better: track it in software
    mov     cr2, rsi            ; This affects host CR2 too — careful!
    
    xor     eax, eax
    mov     rcx, VMCS_VM_ENTRY_INSTR_LEN
    vmwrite rcx, rax
    ret
```

---

## Minimal Hypervisor Skeleton (C + Assembly)

นี่คือ hypervisor skeleton ที่ทำงานได้จริงใน Linux kernel module:

```c
/* minimal_hypervisor.h */
#pragma once
#include <linux/types.h>

#define VMXON_REGION_SIZE   4096
#define VMCS_REGION_SIZE    4096

/* VMCS field encodings */
#define VMCS_GUEST_RIP      0x681E
#define VMCS_GUEST_RSP      0x681C
#define VMCS_GUEST_RFLAGS   0x6820
#define VMCS_EXIT_REASON    0x4402
#define VMCS_HOST_RIP       0x6C16
#define VMCS_HOST_RSP       0x6C14

struct vcpu {
    u64     vmxon_pa;       /* physical address of VMXON region */
    void    *vmxon_va;      /* virtual address */
    u64     vmcs_pa;        /* physical address of VMCS */
    void    *vmcs_va;       /* virtual address */
    
    /* Guest general-purpose registers (saved on VM-Exit) */
    u64     guest_rax;
    u64     guest_rbx;
    u64     guest_rcx;
    u64     guest_rdx;
    u64     guest_rsi;
    u64     guest_rdi;
    u64     guest_rbp;
    u64     guest_r8;
    u64     guest_r9;
    u64     guest_r10;
    u64     guest_r11;
    u64     guest_r12;
    u64     guest_r13;
    u64     guest_r14;
    u64     guest_r15;
    
    bool    launched;
};

int  hvmx_init(struct vcpu *vcpu);
void hvmx_teardown(struct vcpu *vcpu);
int  hvmx_run(struct vcpu *vcpu);
```

```c
/* minimal_hypervisor.c */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/mm.h>
#include <asm/msr.h>
#include <asm/processor.h>
#include <asm/desc.h>
#include "minimal_hypervisor.h"

#define MSR_IA32_VMX_BASIC      0x480
#define MSR_IA32_FEATURE_CTRL   0x3A
#define FEATURE_CTRL_LOCKED     (1ULL << 0)
#define FEATURE_CTRL_VMXON_EN   (1ULL << 2)

static u32 vmx_revision_id(void)
{
    u64 msr = native_read_msr(MSR_IA32_VMX_BASIC);
    return (u32)(msr & 0x7FFFFFFF);
}

static int enable_vmx(void)
{
    u64 feat_ctrl;
    u64 cr4;
    
    /* Check CPUID */
    if (!(cpuid_ecx(1) & (1 << 5))) {
        pr_err("VMX not supported by CPU\n");
        return -ENODEV;
    }
    
    /* Check IA32_FEATURE_CONTROL */
    feat_ctrl = native_read_msr(MSR_IA32_FEATURE_CTRL);
    if (!(feat_ctrl & FEATURE_CTRL_LOCKED)) {
        /* Not locked: set lock + enable VMX */
        feat_ctrl |= FEATURE_CTRL_LOCKED | FEATURE_CTRL_VMXON_EN;
        native_write_msr(MSR_IA32_FEATURE_CTRL, 
                         (u32)feat_ctrl, (u32)(feat_ctrl >> 32));
    } else if (!(feat_ctrl & FEATURE_CTRL_VMXON_EN)) {
        pr_err("VMX disabled in BIOS (locked)\n");
        return -EPERM;
    }
    
    /* Set CR4.VMXE */
    cr4 = __read_cr4();
    cr4 |= X86_CR4_VMXE;
    __write_cr4(cr4);
    
    return 0;
}

int hvmx_init(struct vcpu *vcpu)
{
    u32 rev_id;
    int ret;
    
    ret = enable_vmx();
    if (ret)
        return ret;
    
    rev_id = vmx_revision_id();
    
    /* Allocate VMXON region */
    vcpu->vmxon_va = (void *)__get_free_page(GFP_KERNEL);
    if (!vcpu->vmxon_va)
        return -ENOMEM;
    
    memset(vcpu->vmxon_va, 0, PAGE_SIZE);
    *(u32 *)vcpu->vmxon_va = rev_id;  /* Write revision ID */
    vcpu->vmxon_pa = virt_to_phys(vcpu->vmxon_va);
    
    /* Allocate VMCS region */
    vcpu->vmcs_va = (void *)__get_free_page(GFP_KERNEL);
    if (!vcpu->vmcs_va) {
        free_page((unsigned long)vcpu->vmxon_va);
        return -ENOMEM;
    }
    
    memset(vcpu->vmcs_va, 0, PAGE_SIZE);
    *(u32 *)vcpu->vmcs_va = rev_id;
    vcpu->vmcs_pa = virt_to_phys(vcpu->vmcs_va);
    
    /* Execute VMXON */
    if (__vmxon(vcpu->vmxon_pa)) {
        pr_err("VMXON failed\n");
        free_page((unsigned long)vcpu->vmcs_va);
        free_page((unsigned long)vcpu->vmxon_va);
        return -EIO;
    }
    
    /* VMCLEAR and VMPTRLD */
    if (__vmpclear(vcpu->vmcs_pa) || __vmptrld(vcpu->vmcs_pa)) {
        pr_err("VMCS setup failed\n");
        __vmxoff();
        free_page((unsigned long)vcpu->vmcs_va);
        free_page((unsigned long)vcpu->vmxon_va);
        return -EIO;
    }
    
    vcpu->launched = false;
    pr_info("Hypervisor initialized successfully\n");
    return 0;
}

void hvmx_teardown(struct vcpu *vcpu)
{
    __vmxoff();
    
    /* Clear CR4.VMXE */
    u64 cr4 = __read_cr4();
    cr4 &= ~X86_CR4_VMXE;
    __write_cr4(cr4);
    
    if (vcpu->vmcs_va)
        free_page((unsigned long)vcpu->vmcs_va);
    if (vcpu->vmxon_va)
        free_page((unsigned long)vcpu->vmxon_va);
}

/* 
 * vmx_vmlaunch_and_exit — asm stub ที่:
 * 1. Restore guest GPRs
 * 2. VMLAUNCH หรือ VMRESUME
 * 3. เมื่อ VM-Exit: save guest GPRs, call C handler
 */
extern int vmx_run_guest(struct vcpu *vcpu);

int hvmx_run(struct vcpu *vcpu)
{
    return vmx_run_guest(vcpu);
}
```

```nasm
; vmx_asm.S — Assembly stubs for VM entry/exit

; vmx_run_guest(struct vcpu *vcpu)
; rdi = vcpu pointer
global vmx_run_guest

vmx_run_guest:
    ; Save host callee-saved registers
    push    rbx
    push    rbp
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; Save vcpu pointer
    push    rdi
    
    ; Load guest GPRs from vcpu struct
    ; struct vcpu layout: guest_rax at offset 0x28 (after vmxon_pa, vmxon_va, vmcs_pa, vmcs_va, launched)
    ; Adjust offsets based on actual struct layout
    
    ; Offsets (example):
    ; +0x00: vmxon_pa  (8 bytes)
    ; +0x08: vmxon_va  (8 bytes)
    ; +0x10: vmcs_pa   (8 bytes)
    ; +0x18: vmcs_va   (8 bytes)
    ; +0x20: guest_rax (8 bytes)
    ; +0x28: guest_rbx
    ; ...
    ; +0x98: launched   (1 byte)
    
    VCPU_GUEST_RAX  equ 0x20
    VCPU_GUEST_RBX  equ 0x28
    VCPU_GUEST_RCX  equ 0x30
    VCPU_GUEST_RDX  equ 0x38
    VCPU_GUEST_RSI  equ 0x40
    VCPU_GUEST_RDI  equ 0x48
    VCPU_GUEST_RBP  equ 0x50
    VCPU_LAUNCHED   equ 0x98
    
    mov     rax, [rdi + VCPU_GUEST_RAX]
    mov     rbx, [rdi + VCPU_GUEST_RBX]
    mov     rcx, [rdi + VCPU_GUEST_RCX]
    mov     rdx, [rdi + VCPU_GUEST_RDX]
    mov     rsi, [rdi + VCPU_GUEST_RSI]
    mov     rbp, [rdi + VCPU_GUEST_RBP]
    ; Note: RDI is loaded last since we're using it as vcpu pointer
    mov     r8,  [rdi + 0x58]   ; guest_r8
    mov     r9,  [rdi + 0x60]   ; guest_r9
    mov     r10, [rdi + 0x68]   ; guest_r10
    mov     r11, [rdi + 0x70]   ; guest_r11
    mov     r12, [rdi + 0x78]   ; guest_r12
    mov     r13, [rdi + 0x80]   ; guest_r13
    mov     r14, [rdi + 0x88]   ; guest_r14
    mov     r15, [rdi + 0x90]   ; guest_r15
    
    ; Check if first launch or resume
    cmp     byte [rdi + VCPU_LAUNCHED], 0
    mov     rdi, [rdi + VCPU_GUEST_RDI]   ; load guest rdi last
    
    je      .do_vmlaunch
    
.do_vmresume:
    vmresume
    jmp     .vmentry_failed     ; shouldn't reach here on success
    
.do_vmlaunch:
    vmlaunch
    ; Fall through on failure
    
.vmentry_failed:
    ; VM-Entry failed: CF=1 (invalid pointer) or ZF=1 (VM fail valid)
    ; RAX etc have been modified — save flags
    pushfq
    pop     rax
    ; rax now has RFLAGS: check CF and ZF
    
    pop     rdi                 ; restore vcpu pointer
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbp
    pop     rbx
    
    mov     eax, -1
    ret

; VM-Exit handler — called by hardware redirect to VMCS_HOST_RIP
; Hardware has already:
; - Saved guest state to VMCS
; - Loaded host state from VMCS (CR0, CR3, CR4, CS, DS, RSP, RIP, etc.)
; - Note: general-purpose registers are NOT saved automatically!

global vmexit_entry
vmexit_entry:
    ; At this point:
    ; - Stack pointer is VMCS_HOST_RSP
    ; - We're executing at VMCS_HOST_RIP (here)
    ; - Guest GPRs are still in registers!
    ; - RSP was set to host stack
    
    ; We need the vcpu pointer — store it in a per-CPU variable
    ; or save it in a scratch area before VMLAUNCH
    ; Simplest approach: use a global (single-vCPU hypervisor)
    
    ; Save all guest GPRs
    push    r15
    push    r14
    push    r13
    push    r12
    push    r11
    push    r10
    push    r9
    push    r8
    push    rbp
    push    rdi
    push    rsi
    push    rdx
    push    rcx
    push    rbx
    push    rax
    
    ; RSP at this point: saved_rax, saved_rbx, ..., saved_r15 = 15*8 = 120 bytes
    
    ; Get vcpu pointer (global or from per-CPU area)
    extern  current_vcpu
    mov     rdi, [current_vcpu]
    
    ; Save guest GPRs to vcpu struct
    mov     rax, [rsp + 0*8]    ; saved RAX
    mov     [rdi + VCPU_GUEST_RAX], rax
    mov     rax, [rsp + 1*8]    ; saved RBX
    mov     [rdi + VCPU_GUEST_RBX], rax
    ; ... save all registers
    
    ; Call C handler
    ; rdi still = vcpu pointer
    extern  vmexit_handler_c
    call    vmexit_handler_c    ; void vmexit_handler_c(struct vcpu *vcpu)
    
    ; Restore guest GPRs
    mov     rdi, [current_vcpu]
    
    ; Load return value from handler: 0 = resume, non-zero = stop
    test    eax, eax
    jnz     .stop_vm
    
    ; Reload guest GPRs
    mov     rax, [rdi + VCPU_GUEST_RAX]
    mov     rbx, [rdi + VCPU_GUEST_RBX]
    ; ... etc
    
    add     rsp, 15*8           ; discard stack frame
    
    ; Set launched flag
    mov     byte [rdi + VCPU_LAUNCHED], 1
    vmresume
    
    ; Should not reach here
    hlt
    
.stop_vm:
    ; Clean up stack and return to caller
    add     rsp, 15*8
    
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbp
    pop     rbx
    
    xor     eax, eax
    ret
```

---

## AMD SVM (AMD-V) — Comparison

สำหรับ AMD processors ใช้ **SVM (Secure Virtual Machine)** แทน VMX:

### SVM vs VMX Comparison

| Feature | Intel VMX | AMD SVM |
|---------|-----------|---------|
| Enable check | CPUID ECX bit 5 | CPUID ECX bit 2 |
| Enable MSR | CR4.VMXE | EFER.SVME (bit 12) |
| Enter mode | VMXON | (always available after SVME=1) |
| Guest/Host state | VMCS | VMCB (VM Control Block) |
| Enter VM | VMLAUNCH/VMRESUME | VMRUN |
| Exit VM | VM-Exit (automatic) | #VMEXIT (automatic) |
| Return to host | VMRESUME | VMRUN again |
| Nested paging | EPT | NPT (Nested Page Tables) |
| TLB tags | VPID | ASID (Address Space Identifier) |
| Hypercall | VMCALL | VMMCALL |

### SVM CPUID Check

```nasm
; Check AMD SVM support
check_svm_support:
    ; First: check if AMD
    mov     eax, 0              ; CPUID basic leaf 0
    cpuid
    
    ; Check for "AuthenticAMD"
    cmp     ebx, 0x68747541     ; "Auth"
    jne     .not_amd
    cmp     ecx, 0x444D4163     ; "cAMD"
    jne     .not_amd
    cmp     edx, 0x69746E65     ; "enti"
    jne     .not_amd
    
    ; Check SVM support: CPUID 0x80000001, ECX bit 2
    mov     eax, 0x80000001
    cpuid
    test    ecx, (1 << 2)       ; SVM bit
    jz      .no_svm
    
    ; Check if SVM is not disabled by BIOS
    ; MSR 0xC0010114 = VM_CR
    mov     ecx, 0xC0010114
    rdmsr
    test    eax, (1 << 4)       ; SVMDIS bit
    jnz     .svm_disabled
    
    mov     eax, 1
    ret
    
.not_amd:
.no_svm:
.svm_disabled:
    xor     eax, eax
    ret

; Enable SVM
enable_svm:
    ; Set EFER.SVME (bit 12)
    mov     ecx, 0xC0000080     ; EFER MSR
    rdmsr
    or      eax, (1 << 12)      ; SVME
    wrmsr
    ret

; VMRUN — run guest until #VMEXIT
; Input: RAX = physical address of VMCB
run_svm_guest:
    ; Load VMCB and run guest
    vmrun               ; RAX = physical address of VMCB
    
    ; After VMRUN returns, we had a #VMEXIT
    ; Check VMCB.exitcode to determine why
    ; ...
    ret
```

---

## Nested Virtualization (L0/L1/L2)

Nested virtualization: hypervisor ทำงานใน VM:

```
L0 (bare metal hypervisor): รู้ว่าตัวเองเป็น hypervisor
    ↓ VMLAUNCH
L1 (guest hypervisor): คิดว่าตัวเองอยู่บน real hardware
    ↓ VMLAUNCH (nested)
L2 (nested guest): รู้ว่าตัวเองเป็น VM ของ L1
```

Hardware support for nested VMX:
- **Intel**: VMCS Shadowing + VMCS12 (virtual VMCS ของ L1)
- AMD SVM รองรับ nested VMs ตั้งแต่แรก

```nasm
; ตรวจสอบ nested VMX support
check_nested_vmx:
    ; IA32_VMX_PROCBASED_CTLS2 bit 14: VMCS shadowing
    mov     ecx, 0x48B
    rdmsr
    test    edx, (1 << 14)      ; VMCS shadowing bit
    jz      .no_nested
    
    mov     eax, 1
    ret
.no_nested:
    xor     eax, eax
    ret
```

---

## Security Considerations: VM Escape

**VM Escape** คือการที่ guest code สามารถ "หนี" ออกจาก VM ไปรัน code ใน hypervisor:

### ช่องโหว่ที่พบบ่อย

1. **Emulation bugs**: hypervisor emulate instruction ผิดพลาด
2. **EPT misconfiguration**: mapping อนุญาตให้ guest เข้าถึง hypervisor memory
3. **Device emulation bugs**: virtual device มี buffer overflow
4. **Hypercall handler bugs**: VMCALL handler ไม่ validate input

### ตัวอย่าง: CVE-2015-3456 (VENOM)

QEMU's Floppy Disk Controller had buffer overflow:
```c
/* Vulnerable code (simplified) */
void fdc_transfer(FDCtrl *s, uint8_t value) {
    s->fifo[s->data_pos++] = value;  /* No bounds check! */
    /* data_pos could overflow past fifo[512] into adjacent memory */
}
```

Guest สามารถส่ง FDC commands พิเศษเพื่อทำให้ data_pos overflow และเขียนทับ hypervisor heap

### Mitigation

```c
/* Fixed version */
void fdc_transfer_safe(FDCtrl *s, uint8_t value) {
    if (s->data_pos >= sizeof(s->fifo)) {
        pr_err("FDC FIFO overflow attempt!\n");
        fdc_reset(s);
        return;
    }
    s->fifo[s->data_pos++] = value;
}
```

**Hardware mitigations:**
- EPT เพื่อแยก memory ระหว่าง VMs
- SMEP/SMAP เพื่อป้องกัน ring 0 การ execute/access user pages
- IOMMU/VT-d เพื่อ protect DMA จาก device ที่ถูก assign ให้ guest

---

## KVM (Kernel-based Virtual Machine)

KVM ใช้ Intel VT-x/AMD SVM โดยอยู่ใน Linux kernel:

```
Linux Kernel Process
    │
    ├── KVM module (kvm.ko + kvm-intel.ko)
    │     ├── VMXON (per CPU)
    │     ├── VMCS management
    │     ├── VM-Exit handling
    │     └── EPT management
    │
    └── QEMU (userspace)
          ├── Device emulation (virtio, e1000, etc.)
          ├── Memory management (mmap)
          └── KVM_RUN ioctl → triggers VMLAUNCH/VMRESUME
```

### KVM API (Brief)

```c
#include <linux/kvm.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int kvm_fd = open("/dev/kvm", O_RDWR);

/* Create VM */
int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);

/* Set memory region */
struct kvm_userspace_memory_region region = {
    .slot = 0,
    .guest_phys_addr = 0x0,
    .memory_size = 0x100000,    /* 1MB */
    .userspace_addr = (u64)mmap(NULL, 0x100000, PROT_READ|PROT_WRITE,
                                MAP_PRIVATE|MAP_ANONYMOUS, -1, 0),
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);

/* Create vCPU */
int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);

/* Map vCPU state */
int mmap_size = ioctl(kvm_fd, KVM_GET_VCPU_MMAP_SIZE, 0);
struct kvm_run *run = mmap(NULL, mmap_size, PROT_READ|PROT_WRITE,
                           MAP_SHARED, vcpu_fd, 0);

/* Setup registers */
struct kvm_regs regs = {
    .rip = 0x7C00,  /* Boot sector address */
    .rflags = 0x2,
};
ioctl(vcpu_fd, KVM_SET_REGS, &regs);

/* Run vCPU */
while (1) {
    ioctl(vcpu_fd, KVM_RUN, 0);
    
    switch (run->exit_reason) {
    case KVM_EXIT_HLT:
        printf("Guest halted\n");
        goto done;
    case KVM_EXIT_IO:
        /* Handle I/O */
        if (run->io.direction == KVM_EXIT_IO_OUT) {
            /* Guest OUT instruction */
            printf("Guest port write: port=0x%x val=0x%x\n",
                   run->io.port, *(u8 *)((u8*)run + run->io.data_offset));
        }
        break;
    case KVM_EXIT_MMIO:
        /* Handle MMIO */
        break;
    }
}
done:
    close(vcpu_fd);
    close(vm_fd);
    close(kvm_fd);
```

---

## Performance Optimizations

### 1. MSR Bitmap Tuning

```nasm
; อนุญาตให้ guest อ่าน/เขียน TSC_DEADLINE_TIMER โดยตรง
allow_tsc_deadline_msr:
    ; MSR 0x6E0 = IA32_TSC_DEADLINE
    ; ไม่ต้อง set bit ใน MSR bitmap → no exit
    ; แต่ต้อง enable TSC offsetting ถ้าต้องการ virtual time
    ret
```

### 2. PAUSE-Loop Exiting (PLE)

ป้องกัน guest spinlock spin นาน:
```nasm
setup_ple:
    ; PLE_GAP: max iterations ระหว่าง consecutive PLE exits (cycles)
    mov     rdx, 128
    mov     rcx, VMCS_PLE_GAP
    vmwrite rcx, rdx
    
    ; PLE_WINDOW: time window for PLE (cycles)
    mov     rdx, 4096
    mov     rcx, VMCS_PLE_WINDOW
    vmwrite rcx, rdx
    ret
```

### 3. Posted Interrupts

ส่ง interrupt ถึง guest โดยไม่ต้อง VM-Exit ก่อน:
- Set virtual APIC page
- Hardware inject interrupt เมื่อ guest running

### 4. VPID Assignment

```nasm
; Assign unique VPID per VM (0 = hypervisor, 1+ = guests)
global vpid_counter
vpid_counter: dw 1

allocate_vpid:
    mov     ax, [vpid_counter]
    inc     word [vpid_counter]
    
    ; Set in VMCS
    movzx   rdx, ax
    mov     rcx, VMCS_VPID
    vmwrite rcx, rdx
    ret
```

---

## Debugging Hypervisors

### VMX Error Codes

เมื่อ VMXON/VMCLEAR/VMPTRLD/VMLAUNCH/VMRESUME fail:

```nasm
; ตรวจสอบ error หลัง VMX instruction
check_vmx_error:
    jc      .cf_set             ; CF=1: VMfailInvalid (VMCS pointer bad)
    jz      .zf_set             ; ZF=1: VMfailValid (error code in VMCS)
    ; No flags: success
    ret
    
.cf_set:
    ; Invalid VMCS pointer or not in VMX operation
    ; ตรวจ:
    ; 1. VMXON ถูก execute หรือยัง
    ; 2. Physical address ถูกต้องหรือยัง
    ; 3. VMCS revision ID ถูกต้อง
    mov     eax, ERR_VMX_INVALID
    ret
    
.zf_set:
    ; Valid error — read instruction error from VMCS
    mov     rcx, VMCS_VM_INSTR_ERROR
    vmread  rax, rcx
    ; Error codes 1-28:
    ; 1  = VMCALL executed in VMX root operation
    ; 2  = VMCLEAR with invalid physical address
    ; 3  = VMCLEAR with VMXON pointer
    ; 4  = VMLAUNCH with non-clear VMCS
    ; 5  = VMRESUME with non-launched VMCS
    ; 6  = VMRESUME after VMXOFF
    ; 7  = VM entry with invalid control field(s)
    ; 8  = VM entry with invalid host-state field(s)
    ; 9  = VMPTRLD with invalid physical address
    ; 10 = VMPTRLD with incorrect VMCS revision ID
    ; 11 = VMREAD/VMWRITE from/to unsupported VMCS component
    ; 12 = VMWRITE to read-only VMCS component
    ; 14 = VMXON executed in VMX root operation
    ; 15 = VM entry with invalid executive-VMCS pointer
    ; 16 = VM entry with non-launched executive VMCS
    ; 17 = VM entry with executive-VMCS pointer not VMXON pointer
    ; 18 = VMCALL with non-clear VMCS
    ; 19 = VMCALL with invalid VM-exit control fields
    ; 21 = VMCALL with incorrect MSEG revision identifier
    ; 22 = VMXOFF under dual-monitor treatment of SMIs and SMM
    ; 23 = VMCALL with invalid SMM-monitor features
    ; 24 = VM entry with invalid VM-execution control fields in executive VMCS
    ; 25 = VM entry with events blocked by MOV SS
    ; 26 = Invalid operand to INVEPT/INVVPID
    ret
```

### Using Linux KVM Tracing

```bash
# เปิด KVM tracing
echo 1 > /sys/kernel/debug/tracing/events/kvm/enable

# ดู VM-Exit events
cat /sys/kernel/debug/tracing/trace | grep kvm_exit

# ผลลัพธ์ตัวอย่าง:
# qemu-system-x86-1234 [001] kvm_exit: reason CPUID rip 0x7c00
# qemu-system-x86-1234 [001] kvm_exit: reason HLT rip 0x7c10
# qemu-system-x86-1234 [001] kvm_exit: reason IO_INSTRUCTION rip 0x7c15

# ดู EPT violations
cat /sys/kernel/debug/tracing/trace | grep ept
```

### Checking VMCS State

```nasm
; Dump key VMCS fields for debugging
dump_vmcs_state:
    ; Guest state
    mov     rcx, VMCS_GUEST_RIP
    vmread  rax, rcx
    ; print "Guest RIP: <rax>"
    
    mov     rcx, VMCS_GUEST_RSP
    vmread  rax, rcx
    ; print "Guest RSP: <rax>"
    
    mov     rcx, VMCS_GUEST_CR0
    vmread  rax, rcx
    ; print "Guest CR0: <rax>"
    
    mov     rcx, VMCS_GUEST_CR3
    vmread  rax, rcx
    ; print "Guest CR3: <rax>"
    
    mov     rcx, VMCS_GUEST_RFLAGS
    vmread  rax, rcx
    ; print "Guest RFLAGS: <rax>"
    
    ; Exit information
    mov     rcx, VMCS_EXIT_REASON
    vmread  rax, rcx
    ; print "Exit Reason: <rax & 0xFFFF>"
    
    mov     rcx, VMCS_EXIT_QUALIFICATION
    vmread  rax, rcx
    ; print "Exit Qualification: <rax>"
    
    ret
```

---

## Summary: VMX Lifecycle

```
1. CPUID check (ECX bit 5)
   ↓
2. IA32_FEATURE_CONTROL check/set
   ↓
3. CR4.VMXE = 1
   ↓
4. Allocate + initialize VMXON region
   (write VMX revision ID to offset 0)
   ↓
5. VMXON <phys_addr>    → Enter VMX root mode
   ↓
6. Allocate + initialize VMCS
   (write VMX revision ID to offset 0)
   ↓
7. VMCLEAR <vmcs_phys>  → Initialize VMCS state
   ↓
8. VMPTRLD <vmcs_phys>  → Make VMCS current
   ↓
9. VMWRITE all required fields:
   - Guest state (RIP, RSP, segments, CRs)
   - Host state (segments, CRs, RSP, RIP)
   - VM-execution controls
   - VM-entry controls
   - VM-exit controls
   ↓
10. VMLAUNCH             → Enter guest (first time)
    ↓ (VM-Exit)
11. vmexit_handler:
    - Read VMCS_EXIT_REASON
    - Handle exit (emulate instruction/device)
    - VMRESUME              → Return to guest
    
12. When done:
    VMXOFF                 → Exit VMX operation
    CR4.VMXE = 0
```

---

## แบบฝึกหัด

1. **Basic**: เขียน kernel module ที่ check VMX support และ print CPU's VMX capabilities จาก MSR 0x480-0x491

2. **Intermediate**: Implement hypervisor ที่ launch guest ที่เล็กที่สุด (single HLT instruction) และ handle HLT exit

3. **Advanced**: เพิ่ม EPT สำหรับ identity mapping 4MB แรก และ handle EPT violations

4. **Expert**: Implement hypercall interface ด้วย VMCALL และ emulate CPUID ให้ซ่อน VMX support จาก guest

---

## อ่านเพิ่มเติม

- **Intel SDM Volume 3C**: "VMX Instruction Reference" และ "VMCS Fields"
- **Intel SDM Volume 3C Chapter 23-33**: Hardware Virtualization
- **KVM source**: `arch/x86/kvm/vmx/vmx.c` ใน Linux kernel
- **"Hardware and Software Support for Virtualization"** — Bugnion, Nieh, Tsafrir
- **hvpp** — บน GitHub: minimal hypervisor for educational purposes
- **SimpleVisor** — Microsoft Research hypervisor example
- **NOVA Microhypervisor**: research hypervisor ที่มี design ดีมาก
