# Part 087: Virtual Memory Manager

## บทนำ (Introduction)

Virtual Memory Manager (VMM) คือส่วนสำคัญที่สุดของ kernel ที่จัดการ address space ของ process แต่ละตัว
ทำให้แต่ละ process คิดว่าตัวเองมี memory ทั้งหมดไว้ใช้เพียงผู้เดียว โดยแปลง virtual address เป็น physical address
ผ่านกลไก page table ที่ hardware (MMU) จัดการให้

**เป้าหมายของบทนี้:**
- เข้าใจ layout ของ process address space
- เขียน VMA (Virtual Memory Area) management
- implement page table operations ใน NASM
- สร้าง page fault handler
- เข้าใจ demand paging และ copy-on-write (COW)
- implement brk(), mmap(), munmap()
- จัดการ context switch และ address space switch

---

## 1. Process Address Space Layout

```
Virtual Address Space (64-bit, x86-64)
┌─────────────────────────────┐  0xFFFFFFFFFFFFFFFF
│   Kernel Space (high half)  │
│   Mapped in ALL processes   │
├─────────────────────────────┤  0xFFFF800000000000
│                             │
│   (Non-canonical gap)       │
│                             │
├─────────────────────────────┤  0x00007FFFFFFFFFFF
│         Stack               │  grows downward ↓
│    (starts at ~top)         │
├─────────────────────────────┤
│           ...               │
├─────────────────────────────┤
│    mmap / shared libs       │  grows downward ↓
├─────────────────────────────┤
│           ...               │
├─────────────────────────────┤
│          Heap               │  grows upward ↑
│      (via brk/mmap)         │
├─────────────────────────────┤
│     BSS segment             │  uninitialized data
├─────────────────────────────┤
│     Data segment            │  initialized data
├─────────────────────────────┤
│     Text segment            │  executable code
└─────────────────────────────┘  0x0000000000400000
```

**ค่า Address ทั่วไปใน Linux x86-64:**
- Text: `0x0000000000400000`
- Heap: เริ่มหลัง BSS (ขยายด้วย brk)
- mmap base: `0x00007FFFF7000000` (ลงมา)
- Stack: `0x00007FFFFFFFE000` (ลงมา)
- Kernel: `0xFFFF800000000000` ขึ้นไป

---

## 2. VMA (Virtual Memory Area) Structure

VMA คือ struct ที่บอกว่า range ของ virtual address นี้ใช้ทำอะไร มี permission อะไร

```nasm
; ============================================================
; vma.asm - Virtual Memory Area definitions
; ============================================================

struc vma_t
    .vm_start:    resq 1   ; start virtual address (inclusive)
    .vm_end:      resq 1   ; end virtual address (exclusive)
    .vm_flags:    resq 1   ; permission and type flags
    .vm_next:     resq 1   ; next VMA in list (linked list)
    .vm_prev:     resq 1   ; previous VMA
    .vm_pgoff:    resq 1   ; page frame offset (for file-backed)
    .vm_file:     resq 1   ; pointer to file (NULL = anonymous)
    .vm_mm:       resq 1   ; back-pointer to mm_struct
endstruc
; sizeof(vma_t) = 64 bytes

; VMA flags (ใช้ OR รวมกันได้)
VM_READ       equ (1 << 0)   ; region is readable
VM_WRITE      equ (1 << 1)   ; region is writable
VM_EXEC       equ (1 << 2)   ; region is executable
VM_SHARED     equ (1 << 3)   ; region is shared
VM_GROWSDOWN  equ (1 << 8)   ; stack grows downward
VM_GROWSUP    equ (1 << 9)   ; heap grows upward
VM_ANON       equ (1 << 10)  ; anonymous mapping (no file)
VM_IO         equ (1 << 11)  ; memory-mapped I/O
VM_DONTCOPY   equ (1 << 12)  ; do not copy on fork
VM_DONTEXPAND equ (1 << 13)  ; cannot expand with mremap
VM_LOCKED     equ (1 << 14)  ; pages are locked in memory
```

### mm_struct - Memory Descriptor ของแต่ละ Process

```nasm
struc mm_struct
    .pgd:         resq 1   ; physical addr ของ PGD (Page Global Dir)
    .mmap:        resq 1   ; pointer to first VMA (linked list head)
    .mmap_cache:  resq 1   ; last used VMA (cache for speed)
    .start_code:  resq 1   ; start of code segment
    .end_code:    resq 1   ; end of code segment
    .start_data:  resq 1   ; start of data segment
    .end_data:    resq 1   ; end of data segment
    .start_brk:   resq 1   ; start of heap
    .brk:         resq 1   ; current top of heap
    .start_stack: resq 1   ; start of stack
    .arg_start:   resq 1   ; start of argv
    .arg_end:     resq 1   ; end of argv
    .env_start:   resq 1   ; start of envp
    .env_end:     resq 1   ; end of envp
    .mm_count:    resd 1   ; reference count
    .map_count:   resd 1   ; number of VMAs
endstruc
```

---

## 3. VMA Management: Linked List Operations

VMA จะถูกเรียงตาม address และเก็บเป็น doubly-linked list (kernel จริงๆ ใช้ red-black tree ด้วย แต่เราจะใช้ linked list ก่อน)

```nasm
; ============================================================
; vma_ops.asm - VMA linked list operations
; ============================================================
section .text
bits 64

; find_vma(mm_struct* mm, uint64_t addr) -> vma_t* or NULL
; หา VMA ที่ครอบคลุม addr
; Args: rdi = mm_struct*, rsi = virtual address
; Return: rax = vma_t* (NULL ถ้าไม่เจอ)
find_vma:
    push rbx
    push r12
    push r13
    
    mov rbx, rdi            ; rbx = mm
    mov r12, rsi            ; r12 = addr
    
    ; ตรวจ cache ก่อน (optimization)
    mov rax, [rbx + mm_struct.mmap_cache]
    test rax, rax
    jz .scan_list
    
    ; ตรวจว่า addr อยู่ใน cached VMA ไหม
    mov r13, [rax + vma_t.vm_start]
    cmp r12, r13
    jb .scan_list           ; addr < vm_start -> ไม่ใช่
    mov r13, [rax + vma_t.vm_end]
    cmp r12, r13
    jb .found               ; vm_start <= addr < vm_end -> เจอแล้ว
    
.scan_list:
    mov rax, [rbx + mm_struct.mmap]  ; เริ่มจาก head
    
.loop:
    test rax, rax
    jz .not_found           ; list หมดแล้ว
    
    mov r13, [rax + vma_t.vm_end]
    cmp r12, r13
    jge .next               ; addr >= vm_end -> ข้ามไป
    
    ; addr < vm_end -> อาจเป็น VMA นี้
    mov r13, [rax + vma_t.vm_start]
    cmp r12, r13
    jb .not_found           ; addr < vm_start -> ไม่มี VMA ครอบคลุม
    
    ; vm_start <= addr < vm_end -> เจอแล้ว
    ; update cache
    mov [rbx + mm_struct.mmap_cache], rax
    jmp .found
    
.next:
    mov rax, [rax + vma_t.vm_next]
    jmp .loop
    
.not_found:
    xor eax, eax
    jmp .done
    
.found:
.done:
    pop r13
    pop r12
    pop rbx
    ret


; insert_vma(mm_struct* mm, vma_t* new_vma)
; แทรก VMA เข้าไปใน list ตาม address order
; Args: rdi = mm_struct*, rsi = vma_t*
insert_vma:
    push rbx
    push r12
    push r13
    push r14
    
    mov rbx, rdi            ; rbx = mm
    mov r12, rsi            ; r12 = new_vma
    
    mov r13, [r12 + vma_t.vm_start]  ; r13 = new vm_start
    
    ; หาตำแหน่งที่จะแทรก (prev)
    xor r14, r14            ; r14 = prev = NULL
    mov rax, [rbx + mm_struct.mmap]  ; rax = current
    
.find_pos:
    test rax, rax
    jz .insert_here
    
    mov rcx, [rax + vma_t.vm_start]
    cmp r13, rcx
    jb .insert_here         ; new_start < current_start -> แทรกก่อน current
    
    mov r14, rax
    mov rax, [rax + vma_t.vm_next]
    jmp .find_pos
    
.insert_here:
    ; แทรก r12 ระหว่าง r14 (prev) และ rax (next)
    mov [r12 + vma_t.vm_next], rax
    mov [r12 + vma_t.vm_prev], r14
    
    test r14, r14
    jz .update_head
    mov [r14 + vma_t.vm_next], r12
    jmp .update_next_prev
    
.update_head:
    mov [rbx + mm_struct.mmap], r12
    
.update_next_prev:
    test rax, rax
    jz .done
    mov [rax + vma_t.vm_prev], r12
    
.done:
    ; increment map_count
    inc dword [rbx + mm_struct.map_count]
    
    pop r14
    pop r13
    pop r12
    pop rbx
    ret


; remove_vma(mm_struct* mm, vma_t* vma)
; ลบ VMA ออกจาก list
; Args: rdi = mm_struct*, rsi = vma_t*
remove_vma:
    push rbx
    push r12
    
    mov rbx, rdi
    mov r12, rsi
    
    mov rax, [r12 + vma_t.vm_prev]
    mov rcx, [r12 + vma_t.vm_next]
    
    test rax, rax
    jz .update_head
    mov [rax + vma_t.vm_next], rcx
    jmp .update_next
    
.update_head:
    mov [rbx + mm_struct.mmap], rcx
    
.update_next:
    test rcx, rcx
    jz .done
    mov [rcx + vma_t.vm_prev], rax
    
.done:
    dec dword [rbx + mm_struct.map_count]
    
    ; invalidate cache ถ้าตรงกัน
    cmp [rbx + mm_struct.mmap_cache], r12
    jne .no_cache_clear
    mov qword [rbx + mm_struct.mmap_cache], 0
    
.no_cache_clear:
    pop r12
    pop rbx
    ret
```

---

## 4. Page Table Management

x86-64 ใช้ 4-level page table: PGD -> PUD -> PMD -> PTE
แต่ละ entry 8 bytes, แต่ละ table 4KB = 512 entries

```
Virtual Address (48-bit canonical):
┌──────┬──────┬──────┬──────┬──────────────┐
│ PGD  │ PUD  │ PMD  │ PTE  │  Page Offset │
│ 9bit │ 9bit │ 9bit │ 9bit │    12bit     │
└──────┴──────┴──────┴──────┴──────────────┘
63    48  47   39   38   30   29   21   20   12  11    0
```

### Page Table Entry Flags

```nasm
; Page Table Entry flags
PTE_PRESENT    equ (1 << 0)   ; page is present in memory
PTE_WRITE      equ (1 << 1)   ; writable
PTE_USER       equ (1 << 2)   ; accessible from userspace
PTE_PWT        equ (1 << 3)   ; write-through cache
PTE_PCD        equ (1 << 4)   ; cache disable
PTE_ACCESSED   equ (1 << 5)   ; set by CPU on access
PTE_DIRTY      equ (1 << 6)   ; set by CPU on write
PTE_HUGE       equ (1 << 7)   ; huge page (2MB or 1GB)
PTE_GLOBAL     equ (1 << 8)   ; global page (don't flush TLB on CR3 switch)
PTE_NX         equ (1 << 63)  ; no-execute (bit 63)

; Address mask: bits 12-51
PTE_ADDR_MASK  equ 0x000FFFFFFFFFF000
```

### Page Table Walk (Virtual to Physical)

```nasm
; ============================================================
; pgtable.asm - Page table walk and management
; ============================================================
section .text
bits 64

; get_physaddr(uint64_t virt) -> uint64_t phys (or 0 if not mapped)
; ใช้ page table ปัจจุบัน (CR3) เพื่อหา physical address
; Args: rdi = virtual address
; Return: rax = physical address (0 ถ้าไม่ได้ map)
get_physaddr:
    push rbx
    push r12
    
    mov r12, rdi            ; r12 = virtual address
    
    ; อ่าน PGD base จาก CR3
    mov rax, cr3
    and rax, ~0xFFF         ; mask off flags (bits 0-11)
    
    ; --- Level 1: PGD ---
    ; PGD index = bits 47:39
    mov rbx, r12
    shr rbx, 39
    and rbx, 0x1FF          ; 9-bit index
    lea rax, [rax + rbx*8]  ; &pgd[index]
    mov rax, [rax]          ; load PGD entry
    test rax, PTE_PRESENT
    jz .not_mapped
    and rax, PTE_ADDR_MASK  ; get PUD base physical addr
    
    ; --- Level 2: PUD ---
    ; PUD index = bits 38:30
    mov rbx, r12
    shr rbx, 30
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]
    mov rax, [rax]
    test rax, PTE_PRESENT
    jz .not_mapped
    ; ตรวจ huge page (1GB)
    test rax, PTE_HUGE
    jnz .huge_1gb
    and rax, PTE_ADDR_MASK
    
    ; --- Level 3: PMD ---
    ; PMD index = bits 29:21
    mov rbx, r12
    shr rbx, 21
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]
    mov rax, [rax]
    test rax, PTE_PRESENT
    jz .not_mapped
    ; ตรวจ huge page (2MB)
    test rax, PTE_HUGE
    jnz .huge_2mb
    and rax, PTE_ADDR_MASK
    
    ; --- Level 4: PTE ---
    ; PTE index = bits 20:12
    mov rbx, r12
    shr rbx, 12
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]
    mov rax, [rax]
    test rax, PTE_PRESENT
    jz .not_mapped
    and rax, PTE_ADDR_MASK
    
    ; add page offset (bits 11:0)
    mov rbx, r12
    and rbx, 0xFFF
    or rax, rbx
    jmp .done
    
.huge_1gb:
    and rax, 0x000FFFFFFC0000000  ; 1GB aligned mask
    mov rbx, r12
    and rbx, 0x3FFFFFFF           ; 30-bit offset
    or rax, rbx
    jmp .done
    
.huge_2mb:
    and rax, 0x000FFFFFFFE00000   ; 2MB aligned mask
    mov rbx, r12
    and rbx, 0x1FFFFF             ; 21-bit offset
    or rax, rbx
    jmp .done
    
.not_mapped:
    xor eax, eax
    
.done:
    pop r12
    pop rbx
    ret


; map_page(uint64_t virt, uint64_t phys, uint64_t flags)
; map virtual address -> physical address ใน page table ปัจจุบัน
; Args: rdi = virtual addr, rsi = physical addr, rdx = PTE flags
; Return: rax = 0 (success), -1 (error)
; Note: ต้อง allocate page table pages ถ้ายังไม่มี
map_page:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi            ; r12 = virt
    mov r13, rsi            ; r13 = phys
    mov r14, rdx            ; r14 = flags
    
    ; อ่าน PGD จาก CR3
    mov rax, cr3
    and rax, ~0xFFF
    mov rbp, rax            ; rbp = PGD base
    
    ; --- Ensure PGD entry exists -> PUD ---
    mov rbx, r12
    shr rbx, 39
    and rbx, 0x1FF
    lea r15, [rbp + rbx*8]  ; r15 = &pgd_entry
    mov rax, [r15]
    test rax, PTE_PRESENT
    jnz .pgd_ok
    ; allocate new PUD page
    call alloc_page_table
    test rax, rax
    jz .error
    or rax, PTE_PRESENT | PTE_WRITE | PTE_USER
    mov [r15], rax
    and rax, PTE_ADDR_MASK
    jmp .pgd_save
.pgd_ok:
    and rax, PTE_ADDR_MASK
.pgd_save:
    mov rbp, rax            ; rbp = PUD base
    
    ; --- Ensure PUD entry exists -> PMD ---
    mov rbx, r12
    shr rbx, 30
    and rbx, 0x1FF
    lea r15, [rbp + rbx*8]
    mov rax, [r15]
    test rax, PTE_PRESENT
    jnz .pud_ok
    call alloc_page_table
    test rax, rax
    jz .error
    or rax, PTE_PRESENT | PTE_WRITE | PTE_USER
    mov [r15], rax
    and rax, PTE_ADDR_MASK
    jmp .pud_save
.pud_ok:
    and rax, PTE_ADDR_MASK
.pud_save:
    mov rbp, rax            ; rbp = PMD base
    
    ; --- Ensure PMD entry exists -> PT ---
    mov rbx, r12
    shr rbx, 21
    and rbx, 0x1FF
    lea r15, [rbp + rbx*8]
    mov rax, [r15]
    test rax, PTE_PRESENT
    jnz .pmd_ok
    call alloc_page_table
    test rax, rax
    jz .error
    or rax, PTE_PRESENT | PTE_WRITE | PTE_USER
    mov [r15], rax
    and rax, PTE_ADDR_MASK
    jmp .pmd_save
.pmd_ok:
    and rax, PTE_ADDR_MASK
.pmd_save:
    mov rbp, rax            ; rbp = PT base
    
    ; --- Write PTE ---
    mov rbx, r12
    shr rbx, 12
    and rbx, 0x1FF
    lea r15, [rbp + rbx*8]
    
    ; สร้าง PTE: phys | flags | PRESENT
    mov rax, r13
    and rax, PTE_ADDR_MASK
    or rax, r14
    or rax, PTE_PRESENT
    mov [r15], rax
    
    ; flush TLB for this page
    invlpg [r12]
    
    xor eax, eax            ; return 0 (success)
    jmp .done
    
.error:
    mov eax, -1
    
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret


; unmap_page(uint64_t virt)
; ลบ mapping ของ virtual address ออกจาก page table
; Args: rdi = virtual address
; Return: rax = physical address ที่เคย map (0 ถ้าไม่มี)
unmap_page:
    push rbx
    push r12
    
    mov r12, rdi
    
    ; walk to PTE (คล้าย get_physaddr แต่จะ clear entry)
    mov rax, cr3
    and rax, ~0xFFF
    
    ; PGD
    mov rbx, r12
    shr rbx, 39
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]
    mov rax, [rax]
    test rax, PTE_PRESENT
    jz .not_mapped
    and rax, PTE_ADDR_MASK
    
    ; PUD
    mov rbx, r12
    shr rbx, 30
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]
    mov rax, [rax]
    test rax, PTE_PRESENT
    jz .not_mapped
    and rax, PTE_ADDR_MASK
    
    ; PMD
    mov rbx, r12
    shr rbx, 21
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]
    mov rax, [rax]
    test rax, PTE_PRESENT
    jz .not_mapped
    and rax, PTE_ADDR_MASK
    
    ; PTE - clear it
    mov rbx, r12
    shr rbx, 12
    and rbx, 0x1FF
    lea rax, [rax + rbx*8]  ; rax = address of PTE
    
    mov rbx, [rax]          ; rbx = old PTE value
    test rbx, PTE_PRESENT
    jz .not_mapped
    
    mov qword [rax], 0      ; clear PTE
    invlpg [r12]            ; flush TLB
    
    ; return physical address
    mov rax, rbx
    and rax, PTE_ADDR_MASK
    jmp .done
    
.not_mapped:
    xor eax, eax
    
.done:
    pop r12
    pop rbx
    ret
```

---

## 5. Page Fault Handler

Page fault เกิดเมื่อ CPU พยายาม access virtual address ที่:
1. ไม่ได้ map (not present)
2. ไม่มี permission (เช่น write to read-only)
3. เกิดใน kernel mode แต่ address เป็น user space (หรือกลับกัน)

```nasm
; ============================================================
; pagefault.asm - Page Fault Handler (#PF, interrupt 14)
; ============================================================

; Error code bits สำหรับ Page Fault:
PF_PRESENT   equ (1 << 0)  ; 0=not-present, 1=protection violation
PF_WRITE     equ (1 << 1)  ; 0=read, 1=write
PF_USER      equ (1 << 2)  ; 0=supervisor, 1=user
PF_RSVD      equ (1 << 3)  ; reserved bit violation
PF_INSTR     equ (1 << 4)  ; instruction fetch

section .text
bits 64

; page_fault_handler - IDT entry สำหรับ interrupt 14
; CPU push: SS, RSP, RFLAGS, CS, RIP, error_code
; CR2 = faulting virtual address
global page_fault_handler
page_fault_handler:
    ; save registers
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
    
    ; อ่าน faulting address จาก CR2
    mov rdi, cr2            ; rdi = fault address
    mov rsi, [rsp + 15*8]   ; rsi = error code (pushed by CPU)
    
    call do_page_fault
    
    ; ถ้า do_page_fault return 0 = handled, ไม่ต้องทำอะไร
    ; ถ้า return -1 = segfault -> kill process
    test rax, rax
    jnz .segfault
    
    ; restore registers
    pop r15
    pop r14
    pop r13
    pop r12
    pop r11
    pop r10
    pop r9
    pop r8
    pop rbp
    pop rsi     ; (รวมกันที่นี่)
    pop rdi
    pop rdx
    pop rcx
    pop rbx
    pop rax
    add rsp, 8  ; skip error code
    iretq
    
.segfault:
    ; TODO: send SIGSEGV to current process
    ; สำหรับ demo นี้แค่ halt
    hlt


; do_page_fault(uint64_t fault_addr, uint64_t error_code)
; ตรรกะหลักของ page fault handling
; Return: 0 = handled, -1 = segfault
do_page_fault:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi            ; r12 = fault_addr
    mov r13, rsi            ; r13 = error_code
    
    ; --- 1. หา current process mm_struct ---
    call get_current_mm     ; return rax = mm_struct*
    test rax, rax
    jz .kernel_fault        ; kernel fault
    mov r14, rax            ; r14 = mm
    
    ; --- 2. หา VMA ที่ครอบคลุม fault_addr ---
    mov rdi, r14
    mov rsi, r12
    call find_vma
    test rax, rax
    jz .check_stack_growth  ; ไม่มี VMA -> อาจเป็น stack growth
    mov rbx, rax            ; rbx = vma
    
    ; --- 3. ตรวจ permission ---
    ; ถ้า write fault ต้องตรวจ VM_WRITE
    test r13, PF_WRITE
    jz .check_read
    test qword [rbx + vma_t.vm_flags], VM_WRITE
    jz .check_cow           ; ไม่มี WRITE flag -> อาจเป็น COW
    jmp .handle_anon_fault
    
.check_read:
    test qword [rbx + vma_t.vm_flags], VM_READ
    jz .segfault_exit
    
.handle_anon_fault:
    ; --- 4. Demand paging: allocate physical page ---
    call alloc_physical_page    ; return rax = physical addr
    test rax, rax
    jz .oom_kill                ; out of memory
    mov rbx, rax                ; rbx = phys page
    
    ; zero-fill new page (anonymous pages ต้องเป็น 0)
    push rdi
    mov rdi, rbx
    mov rcx, 4096 / 8
    xor eax, eax
    rep stosq
    pop rdi
    
    ; --- 5. Map page ใน page table ---
    ; คำนวณ flags จาก VMA flags
    mov rdi, r12
    and rdi, ~0xFFF         ; align to page
    mov rsi, rbx
    mov rdx, PTE_USER | PTE_ACCESSED
    
    ; เพิ่ม WRITE flag ถ้า VMA อนุญาต
    mov rax, [r12 + vma_t.vm_flags]  ; BUG: ควรใช้ rbx ที่เก็บ vma
    ; แก้ไข: load จาก vma ที่หาไว้ก่อนหน้า
    ; (ตัวอย่างนี้แค่ใช้ WRITE เสมอ เพื่อความง่าย)
    or rdx, PTE_WRITE
    
    call map_page
    
    xor eax, eax            ; return 0 (handled)
    jmp .done
    
.check_stack_growth:
    ; ตรวจว่า fault_addr อยู่ใต้ stack หรือไม่ (stack growth)
    mov rdi, r14
    mov rsi, r12
    add rsi, PAGE_SIZE      ; ดูว่า addr+PAGE มี VMA ไหม
    call find_vma
    test rax, rax
    jz .segfault_exit
    
    ; ตรวจว่า VMA นั้นเป็น stack (VM_GROWSDOWN)
    test qword [rax + vma_t.vm_flags], VM_GROWSDOWN
    jz .segfault_exit
    
    ; ขยาย VMA ลงมา (grow stack)
    mov rbx, r12
    and rbx, ~0xFFF         ; page-align
    mov [rax + vma_t.vm_start], rbx
    
    ; allocate page
    call alloc_physical_page
    test rax, rax
    jz .oom_kill
    
    mov rdi, rbx
    mov rsi, rax
    mov rdx, PTE_USER | PTE_WRITE | PTE_ACCESSED
    call map_page
    
    xor eax, eax
    jmp .done
    
.check_cow:
    ; Protection violation on writable VMA -> might be COW
    test r13, PF_PRESENT    ; page present?
    jz .segfault_exit
    test r13, PF_WRITE      ; write fault?
    jz .segfault_exit
    
    ; COW fault
    mov rdi, r12
    call handle_cow_fault
    jmp .done
    
.kernel_fault:
    ; kernel mode fault -> panic
    jmp .segfault_exit
    
.oom_kill:
    ; out of memory -> kill process
    ; TODO: ส่ง SIGKILL
    
.segfault_exit:
    mov eax, -1
    jmp .done
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 6. Demand Paging

Demand paging หมายความว่าเราไม่ allocate physical page ตอน map แต่รอจนกว่า process จะ access page นั้นจริงๆ

```nasm
; ============================================================
; demand_paging.asm - Lazy allocation of physical pages
; ============================================================

; เมื่อ process ขอ memory ด้วย mmap หรือ brk:
; - สร้าง VMA (virtual range) แต่ยังไม่ allocate physical page
; - ตั้ง PTE = 0 (not present)
;
; เมื่อ process access -> page fault -> do_page_fault:
; - เจอ VMA -> allocate physical page -> map -> return
;
; ข้อดี: ประหยัด physical memory มาก

; alloc_vma_lazy(mm_struct* mm, uint64_t start, uint64_t size, uint64_t flags)
; สร้าง VMA โดยไม่ allocate physical pages
; Args: rdi = mm, rsi = start_addr, rdx = size, rcx = vm_flags
; Return: rax = vma_t* (NULL ถ้า error)
alloc_vma_lazy:
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi
    mov r13, rsi
    mov r14, rdx
    
    ; allocate vma struct จาก kmalloc
    mov rdi, vma_t_size     ; sizeof(vma_t)
    call kmalloc
    test rax, rax
    jz .error
    mov rbx, rax            ; rbx = new vma
    
    ; กำหนดค่า
    mov [rbx + vma_t.vm_start], r13
    
    ; คำนวณ end (align to page)
    lea rax, [r13 + r14]
    add rax, 0xFFF
    and rax, ~0xFFF
    mov [rbx + vma_t.vm_end], rax
    
    mov [rbx + vma_t.vm_flags], rcx
    mov qword [rbx + vma_t.vm_next], 0
    mov qword [rbx + vma_t.vm_prev], 0
    mov qword [rbx + vma_t.vm_file], 0   ; anonymous
    mov [rbx + vma_t.vm_mm], r12
    
    ; insert into mm's VMA list
    mov rdi, r12
    mov rsi, rbx
    call insert_vma
    
    mov rax, rbx
    jmp .done
    
.error:
    xor eax, eax
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 7. Copy-on-Write (COW)

COW เป็น optimization สำหรับ `fork()`: แทนที่จะ copy physical pages ทั้งหมดทันที
เราแค่ share pages เดิมแต่ mark เป็น read-only ใน page table ของทั้ง parent และ child
เมื่อใครเขียน -> page fault -> copy เฉพาะ page นั้น

```nasm
; ============================================================
; cow.asm - Copy-on-Write implementation
; ============================================================

; Page reference count structure
; เก็บใน array indexed by (physical_page_number)
; ref_count[pfn] = number of processes sharing this page

section .bss
; สมมติ 4GB physical memory / 4KB pages = 1M pages
; ref_count: array of uint32_t, 1M entries = 4MB
ref_count_table:  resd 1048576   ; [pfn] = ref count

section .text

; get_pfn(uint64_t phys_addr) -> uint64_t pfn
get_pfn:
    shr rdi, 12
    mov rax, rdi
    ret

; inc_ref(uint64_t phys_addr)
; เพิ่ม reference count ของ page
inc_ref:
    shr rdi, 12                     ; pfn
    inc dword [ref_count_table + rdi*4]
    ret

; dec_ref(uint64_t phys_addr) -> uint32_t new_count
; ลด reference count, คืน count ใหม่
dec_ref:
    shr rdi, 12
    dec dword [ref_count_table + rdi*4]
    mov eax, [ref_count_table + rdi*4]
    ret


; fork_copy_page_tables(mm_struct* parent_mm, mm_struct* child_mm)
; copy page tables จาก parent ไป child พร้อม set COW
; Args: rdi = parent mm, rsi = child mm
fork_copy_page_tables:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi            ; r12 = parent mm
    mov r13, rsi            ; r13 = child mm
    
    ; วน loop ผ่านทุก VMA ของ parent
    mov r14, [r12 + mm_struct.mmap]  ; current vma
    
.vma_loop:
    test r14, r14
    jz .done
    
    ; skip VMAs ที่ไม่ต้อง copy (VM_DONTCOPY)
    test qword [r14 + vma_t.vm_flags], VM_DONTCOPY
    jnz .next_vma
    
    ; copy pages ใน range นี้
    mov r15, [r14 + vma_t.vm_start]  ; current addr
    
.page_loop:
    cmp r15, [r14 + vma_t.vm_end]
    jge .next_vma
    
    ; หา physical page ของ parent
    ; (ใช้ parent's page table)
    mov rax, [r12 + mm_struct.pgd]
    ; ... walk page table ...
    ; (simplified: call get_physaddr_in_mm)
    mov rdi, r12
    mov rsi, r15
    call get_physaddr_in_mm
    test rax, rax
    jz .next_page           ; page not present (demand paged) -> skip
    
    mov rbp, rax            ; rbp = physical page
    
    ; increment ref count
    mov rdi, rbp
    call inc_ref
    
    ; map same physical page in child, but READ-ONLY
    mov rdi, r13
    mov rsi, r15
    mov rdx, rbp
    mov rcx, PTE_PRESENT | PTE_USER | PTE_ACCESSED
    ; ไม่ใส่ PTE_WRITE -> read-only (COW)
    call map_page_in_mm
    
    ; ทำ parent's mapping เป็น read-only ด้วย (COW on both)
    mov rdi, r12
    mov rsi, r15
    call make_pte_readonly
    
.next_page:
    add r15, 4096
    jmp .page_loop
    
.next_vma:
    mov r14, [r14 + vma_t.vm_next]
    jmp .vma_loop
    
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret


; handle_cow_fault(uint64_t fault_addr)
; จัดการ COW fault: copy page, update PTE
; Args: rdi = fault address
; Return: rax = 0 (success), -1 (error)
handle_cow_fault:
    push rbx
    push r12
    push r13
    
    mov r12, rdi
    and r12, ~0xFFF         ; page-align
    
    ; หา current physical page
    mov rdi, r12
    call get_physaddr
    test rax, rax
    jz .error
    mov r13, rax            ; r13 = old physical page
    
    ; ตรวจ ref count
    mov rdi, r13
    shr rdi, 12
    mov eax, [ref_count_table + rdi*4]
    
    cmp eax, 1
    je .sole_owner          ; เราเป็นคนเดียวที่ใช้ page นี้
    
    ; ref count > 1: ต้อง copy page
    ; allocate new page
    call alloc_physical_page
    test rax, rax
    jz .error
    mov rbx, rax            ; rbx = new physical page
    
    ; copy content: dst=rbx, src=r13, 4096 bytes
    push rsi
    push rdi
    mov rdi, rbx
    mov rsi, r13
    mov rcx, 4096 / 8
    rep movsq
    pop rdi
    pop rsi
    
    ; ลด ref count ของ old page
    mov rdi, r13
    call dec_ref
    
    ; set ref count ของ new page = 1
    mov rdi, rbx
    shr rdi, 12
    mov dword [ref_count_table + rdi*4], 1
    
    jmp .update_pte
    
.sole_owner:
    ; เราคนเดียวใช้ page -> แค่ make writable
    mov rbx, r13
    jmp .update_pte_write_only
    
.update_pte:
    ; map new page (writable) แทน old page
    mov rdi, r12
    mov rsi, rbx
    mov rdx, PTE_PRESENT | PTE_WRITE | PTE_USER | PTE_ACCESSED | PTE_DIRTY
    call map_page
    
    xor eax, eax
    jmp .done
    
.update_pte_write_only:
    ; แค่เพิ่ม WRITE bit ใน existing PTE
    mov rdi, r12
    call set_pte_write
    xor eax, eax
    jmp .done
    
.error:
    mov eax, -1
    
.done:
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 8. Anonymous Pages และ Stack Growth

```nasm
; ============================================================
; anon_stack.asm - Anonymous pages and stack growth
; ============================================================

; Anonymous page = page ที่ไม่ได้ backed โดย file
; ถ้า evict (swap out) -> เขียนลง swap space
; swap entry ใน PTE: present=0, swap_type|swap_offset ใน bits อื่น

; Stack Growth - expand VMA downward เมื่อ fault ใต้ stack

; stack_grow_down(mm_struct* mm, uint64_t fault_addr)
; ขยาย stack VMA ลงมาครอบ fault_addr
; Return: 0 = success, -1 = error (stack limit exceeded)
stack_grow_down:
    push rbx
    push r12
    push r13
    
    mov r12, rdi            ; r12 = mm
    mov r13, rsi            ; r13 = fault_addr
    and r13, ~0xFFF         ; page-align downward
    
    ; หา VMA ที่อยู่เหนือ fault_addr (stack VMA)
    ; (VMA ที่มี vm_start > fault_addr และ VM_GROWSDOWN)
    mov rax, [r12 + mm_struct.mmap]
    
.find_stack_vma:
    test rax, rax
    jz .error
    
    mov rbx, [rax + vma_t.vm_start]
    cmp r13, rbx
    jge .next              ; fault >= vm_start -> ข้ามไป
    
    ; fault < vm_start -> ตรวจ flag
    test qword [rax + vma_t.vm_flags], VM_GROWSDOWN
    jnz .found_stack_vma
    
.next:
    mov rax, [rax + vma_t.vm_next]
    jmp .find_stack_vma
    
.found_stack_vma:
    ; ตรวจ stack size limit (ไม่ขยายเกิน RLIMIT_STACK)
    ; สมมติ limit = 8MB
    STACK_LIMIT equ (8 * 1024 * 1024)
    mov rbx, [rax + vma_t.vm_end]
    sub rbx, r13
    cmp rbx, STACK_LIMIT
    ja .error               ; stack too large
    
    ; ขยาย VMA
    mov [rax + vma_t.vm_start], r13
    
    ; allocate physical page
    call alloc_physical_page
    test rax, rax
    jz .error
    
    ; map page
    push rax
    mov rdi, r13
    pop rsi
    mov rdx, PTE_PRESENT | PTE_WRITE | PTE_USER
    call map_page
    
    xor eax, eax
    jmp .done
    
.error:
    mov eax, -1
    
.done:
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 9. brk() - Heap Expansion

`brk(new_end)` ขยายหรือย่อ heap ของ process โดยเปลี่ยน `mm->brk`

```nasm
; ============================================================
; brk.asm - brk() system call implementation
; ============================================================

; sys_brk(uint64_t new_brk) -> uint64_t actual_new_brk
; ขยาย/ย่อ heap VMA
; Args: rdi = new_brk (0 = query current brk)
; Return: rax = new brk value
sys_brk:
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi            ; r12 = requested new_brk
    
    ; หา current mm
    call get_current_mm
    mov r14, rax            ; r14 = mm
    
    ; ถ้า new_brk == 0 -> return current brk
    test r12, r12
    jz .return_current
    
    ; page-align new_brk upward
    add r12, 0xFFF
    and r12, ~0xFFF
    
    mov r13, [r14 + mm_struct.brk]  ; r13 = current brk
    
    cmp r12, r13
    je .return_current      ; ไม่เปลี่ยน
    ja .grow_heap           ; ขยาย heap
    jb .shrink_heap         ; ย่อ heap
    
.grow_heap:
    ; ตรวจว่าไม่ชนกับ mmap area
    ; (simplified: ตรวจแค่ว่าไม่เกิน limit)
    HEAP_LIMIT equ (3 * 1024 * 1024 * 1024)  ; 3GB
    mov rax, [r14 + mm_struct.start_brk]
    add rax, HEAP_LIMIT
    cmp r12, rax
    ja .return_current      ; เกิน limit -> ไม่ขยาย (return old brk)
    
    ; ตรวจว่ามี VMA ไหนขัดกัน
    mov rdi, r14
    mov rsi, r13            ; current brk = start of new region
    call find_vma_intersection   ; หา VMA ที่ overlap กับ [r13, r12)
    test rax, rax
    jnz .return_current     ; มี VMA ชน -> ไม่ขยาย
    
    ; หา heap VMA
    mov rdi, r14
    mov rsi, [r14 + mm_struct.start_brk]
    call find_vma
    test rax, rax
    jz .create_heap_vma     ; ยังไม่มี heap VMA -> สร้างใหม่
    mov rbx, rax            ; rbx = heap VMA
    
    ; ขยาย heap VMA
    mov [rbx + vma_t.vm_end], r12
    
    ; ไม่ต้อง allocate pages ตอนนี้ (demand paging)
    ; pages จะถูก allocate เมื่อ process access
    
    mov [r14 + mm_struct.brk], r12
    mov rax, r12
    jmp .done
    
.create_heap_vma:
    ; สร้าง heap VMA ใหม่
    mov rdi, r14
    mov rsi, [r14 + mm_struct.start_brk]
    mov rdx, r12
    sub rdx, rsi            ; size
    mov rcx, VM_READ | VM_WRITE | VM_ANON | VM_GROWSUP
    call alloc_vma_lazy
    test rax, rax
    jz .return_current
    
    mov [r14 + mm_struct.brk], r12
    mov rax, r12
    jmp .done
    
.shrink_heap:
    ; ย่อ heap: unmap pages ที่เกิน
    mov rax, r12            ; rax = new (smaller) brk
    
.unmap_loop:
    cmp rax, r13
    jge .done_unmap
    
    mov rdi, rax
    call unmap_page
    
    add rax, 4096
    jmp .unmap_loop
    
.done_unmap:
    ; ย่อ heap VMA
    mov rdi, r14
    mov rsi, [r14 + mm_struct.start_brk]
    call find_vma
    test rax, rax
    jz .return_current
    
    mov [rax + vma_t.vm_end], r12
    mov [r14 + mm_struct.brk], r12
    mov rax, r12
    jmp .done
    
.return_current:
    mov rax, [r14 + mm_struct.brk]
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 10. mmap() Implementation

```nasm
; ============================================================
; mmap.asm - mmap() system call
; ============================================================

; mmap flags
MAP_SHARED    equ 0x01     ; share changes
MAP_PRIVATE   equ 0x02     ; changes private (COW)
MAP_FIXED     equ 0x10     ; interpret addr exactly
MAP_ANON      equ 0x20     ; anonymous (no file)
MAP_GROWSDOWN equ 0x100    ; stack-like mapping
MAP_POPULATE  equ 0x8000   ; populate (prefault) pages

; mmap prot flags
PROT_NONE     equ 0
PROT_READ     equ 1
PROT_WRITE    equ 2
PROT_EXEC     equ 4

; sys_mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset)
; Return: rax = mapped address, or -ERRNO on error
; Args via registers (Linux calling convention):
; rdi=addr, rsi=length, rdx=prot, rcx=flags, r8=fd, r9=offset
sys_mmap:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi            ; addr
    mov r13, rsi            ; length
    mov r14, rdx            ; prot
    mov r15, rcx            ; flags
    ; r8=fd, r9=offset
    
    ; ตรวจ length
    test r13, r13
    jz .einval
    
    ; page-align length
    add r13, 0xFFF
    and r13, ~0xFFF
    
    ; แปลง prot -> VMA flags
    xor rbx, rbx
    test r14, PROT_READ
    jz .no_read
    or rbx, VM_READ
.no_read:
    test r14, PROT_WRITE
    jz .no_write
    or rbx, VM_WRITE
.no_write:
    test r14, PROT_EXEC
    jz .no_exec
    ; no VM_EXEC flag defined above, skip
.no_exec:
    test r15, MAP_SHARED
    jz .not_shared
    or rbx, VM_SHARED
.not_shared:
    test r15, MAP_ANON
    jz .not_anon
    or rbx, VM_ANON
.not_anon:
    
    ; หา address ที่จะ map
    test r15, MAP_FIXED
    jnz .use_fixed_addr
    
    ; หา free area ใน address space
    call get_current_mm
    mov rdi, rax
    mov rsi, r13            ; size
    call find_free_area     ; return rax = start addr
    test rax, rax
    jz .enomem
    mov r12, rax
    jmp .do_map
    
.use_fixed_addr:
    ; ตรวจว่า addr valid และ page-aligned
    test r12, 0xFFF
    jnz .einval
    ; unmap existing VMAs ใน range นี้
    call get_current_mm
    mov rdi, rax
    mov rsi, r12
    mov rdx, r13
    call unmap_range
    
.do_map:
    ; สร้าง VMA
    call get_current_mm
    mov rdi, rax
    mov rsi, r12
    mov rdx, r13
    mov rcx, rbx            ; vm_flags
    call alloc_vma_lazy
    test rax, rax
    jz .enomem
    
    ; ถ้า MAP_POPULATE -> prefault ทุก page
    test r15, MAP_POPULATE
    jz .no_populate
    
    mov rbp, r12
.populate_loop:
    cmp rbp, r12
    jge .no_populate        ; BUG: ควรเป็น r12+r13
    ; ... allocate and map each page ...
    add rbp, 4096
    jmp .populate_loop
    
.no_populate:
    mov rax, r12            ; return mapped address
    jmp .done
    
.einval:
    mov rax, -22            ; -EINVAL
    jmp .done
    
.enomem:
    mov rax, -12            ; -ENOMEM
    
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret


; find_free_area(mm_struct* mm, uint64_t size)
; หา virtual address range ที่ว่าง ขนาด size bytes
; Return: rax = start address (0 = ไม่เจอ)
find_free_area:
    push rbx
    push r12
    push r13
    
    mov r12, rdi            ; mm
    mov r13, rsi            ; size
    
    ; เริ่มจาก mmap base (ลงมา: high to low)
    MMAP_BASE equ 0x00007FFFF7000000
    mov rbx, MMAP_BASE
    
.search_loop:
    ; ตรวจว่า [rbx-size, rbx) ว่างไหม
    mov rax, rbx
    sub rax, r13
    test rax, rax
    jz .not_found
    
    ; หา VMA ที่อาจ overlap
    mov rdi, r12
    mov rsi, rax
    call find_vma
    test rax, rax
    jz .found               ; ไม่มี VMA -> free!
    
    ; มี VMA -> ลองต่ำกว่า vm_start ของ VMA นั้น
    mov rbx, [rax + vma_t.vm_start]
    jmp .search_loop
    
.found:
    mov rax, rbx
    sub rax, r13
    jmp .done
    
.not_found:
    xor eax, eax
    
.done:
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 11. munmap() - Remove Mapping

```nasm
; ============================================================
; munmap.asm - munmap() system call
; ============================================================

; sys_munmap(void* addr, size_t length)
; Args: rdi = addr, rsi = length
; Return: rax = 0 (success), -1 (error)
sys_munmap:
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi            ; addr
    mov r13, rsi            ; length
    
    ; ตรวจ alignment
    test r12, 0xFFF
    jnz .einval
    test r13, r13
    jz .einval
    
    ; page-align length
    add r13, 0xFFF
    and r13, ~0xFFF
    
    call get_current_mm
    mov r14, rax
    
    ; คำนวณ end address
    lea rbx, [r12 + r13]    ; rbx = end
    
    ; unmap ทุก page ใน range
    mov rax, r12
.unmap_pages:
    cmp rax, rbx
    jge .done_unmap_pages
    
    ; unmap page จาก page table
    mov rdi, rax
    call unmap_page
    
    add rax, 4096
    jmp .unmap_pages
    
.done_unmap_pages:
    ; ลบ/แบ่ง VMAs ที่ overlap กับ [r12, rbx)
    mov rdi, r14
    mov rsi, r12
    mov rdx, rbx
    call remove_vma_range
    
    xor eax, eax
    jmp .done
    
.einval:
    mov eax, -22
    
.done:
    pop r14
    pop r13
    pop r12
    pop rbx
    ret


; remove_vma_range(mm_struct* mm, uint64_t start, uint64_t end)
; ลบ/ตัด VMAs ที่ overlap กับ [start, end)
remove_vma_range:
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi            ; mm
    mov r13, rsi            ; start
    mov r14, rdx            ; end
    
    ; scan VMAs
    mov r15, [r12 + mm_struct.mmap]
    
.scan:
    test r15, r15
    jz .done
    
    mov rax, [r15 + vma_t.vm_start]
    mov rbx, [r15 + vma_t.vm_end]
    mov rcx, [r15 + vma_t.vm_next]   ; save next
    
    ; ตรวจ overlap: overlap ถ้า start < vma_end และ end > vma_start
    cmp r13, rbx
    jge .next_vma           ; munmap_start >= vma_end -> ไม่ overlap
    cmp r14, rax
    jle .next_vma           ; munmap_end <= vma_start -> ไม่ overlap
    
    ; Overlap! ตรวจว่าเป็น partial หรือ full removal
    cmp r13, rax
    jg .partial_left        ; munmap ไม่ครอบคลุม ซ้ายของ vma
    cmp r14, rbx
    jl .partial_right       ; munmap ไม่ครอบคลุม ขวาของ vma
    
    ; Full removal: munmap ครอบ vma ทั้งหมด
    mov rdi, r12
    mov rsi, r15
    call remove_vma
    ; free vma struct
    mov rdi, r15
    call kfree
    jmp .next_vma_saved
    
.partial_left:
    ; munmap เริ่มกลาง vma -> ตัดซ้ายออก
    ; [vma_start ... r13 | r13 ... vma_end]
    ; เก็บซ้าย: [vma_start, r13)
    ; ลบ: [r13, min(r14, vma_end))
    ; ถ้า r14 < vma_end -> ยังมีขวาเหลือ -> ต้องแบ่ง vma
    cmp r14, rbx
    jge .cut_right_part
    
    ; แบ่ง VMA: สร้าง new VMA สำหรับส่วนขวา [r14, vma_end)
    ; (ต้อง allocate new vma struct)
    push rcx
    mov rdi, vma_t_size
    call kmalloc
    pop rcx
    test rax, rax
    jz .next_vma_saved
    
    ; copy vma content
    push rsi
    mov rsi, r15
    mov rbx, rax
    ; memcpy(rbx, r15, sizeof(vma_t))
    mov rdx, vma_t_size
    ; ... copy ...
    
    ; update new vma: start = r14
    mov [rbx + vma_t.vm_start], r14
    ; update old vma: end = r13
    mov [r15 + vma_t.vm_end], r13
    
    ; insert new vma
    mov rdi, r12
    mov rsi, rbx
    call insert_vma
    pop rsi
    jmp .next_vma_saved
    
.cut_right_part:
    ; ตัดขวา: vma end = r13
    ; (ส่วน [r13, vma_end) ถูก munmap)
    mov [r15 + vma_t.vm_end], r13
    jmp .next_vma_saved
    
.partial_right:
    ; munmap สิ้นสุดกลาง vma -> ตัดส่วน [vma_start, r14) ออก
    ; เก็บขวา: [r14, vma_end)
    mov [r15 + vma_t.vm_start], r14
    jmp .next_vma_saved
    
.next_vma:
    mov rcx, [r15 + vma_t.vm_next]
.next_vma_saved:
    mov r15, rcx
    jmp .scan
    
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

---

## 12. Context Switch: Address Space Switch

เมื่อ switch process ต้อง load CR3 ของ process ใหม่ เพื่อเปลี่ยน page table

```nasm
; ============================================================
; context_switch.asm - Address space switch via CR3
; ============================================================

; switch_mm(mm_struct* old_mm, mm_struct* new_mm, task_struct* new_task)
; Switch address space
; Args: rdi = old_mm, rsi = new_mm, rdx = new_task
switch_mm:
    push rbx
    
    ; ถ้า old == new -> ไม่ต้อง switch
    cmp rdi, rsi
    je .done
    
    ; โหลด PGD ของ process ใหม่เข้า CR3
    mov rax, [rsi + mm_struct.pgd]
    
    ; CR3 format: [PCID(11:0) | PGD_phys(63:12)]
    ; ถ้าใช้ PCID (Process Context ID):
    ;   set bit 63 ใน CR3 เพื่อ skip TLB flush (INVPCID)
    ;   แต่เราจะ flush ทั้งหมดแบบ simple ก่อน
    
    ; Flush TLB: load CR3 จะ flush ทั้งหมด (ยกเว้น GLOBAL pages)
    mov cr3, rax
    
    ; Kernel pages (ที่มี PTE_GLOBAL) จะไม่ถูก flush
    ; ซึ่งเป็นสิ่งที่ต้องการ เพราะ kernel mapped ใน ทุก process
    
.done:
    pop rbx
    ret


; full context switch: save/restore registers + switch mm
; (simplified version, real kernel จะซับซ้อนกว่านี้มาก)

; switch_context(task_struct* prev, task_struct* next)
; save prev's registers, load next's registers
switch_context:
    push rbp
    mov rbp, rsp
    
    ; save callee-saved registers ของ prev
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    ; สมมติ task_struct มี:
    ;   .mm offset 0: mm_struct*
    ;   .rsp offset 8: saved stack pointer
    ;   .rip offset 16: saved instruction pointer (via ret)
    
    ; save RSP ของ prev
    mov [rdi + 8], rsp
    
    ; switch address space
    mov rax, [rdi]          ; old_mm = prev->mm
    mov rcx, [rsi]          ; new_mm = next->mm
    
    push rsi                ; save next
    mov rdi, rax
    mov rsi, rcx
    call switch_mm
    pop rsi                 ; restore next
    
    ; load RSP ของ next
    mov rsp, [rsi + 8]
    
    ; restore next's callee-saved registers
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    
    pop rbp
    ret                     ; return ไปยัง next->rip
```

---

## 13. Kernel High-Half Mapping

Kernel จะถูก map ใน virtual address สูง (high half) ของทุก process
ทำให้ system calls สามารถทำงานได้โดยไม่ต้อง switch page table

```nasm
; ============================================================
; kernel_map.asm - Kernel space mapping
; ============================================================

; x86-64 canonical addresses:
; User space:   0x0000000000000000 - 0x00007FFFFFFFFFFF
; Non-canonical: 0x0000800000000000 - 0xFFFF7FFFFFFFFFFF
; Kernel space: 0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF

KERNEL_BASE     equ 0xFFFFFFFF80000000   ; kernel image base (typical)
KERNEL_PHYS     equ 0x0000000001000000   ; kernel physical load address
KERNEL_SIZE     equ 0x0000000002000000   ; 32MB max kernel size
DIRECT_MAP_BASE equ 0xFFFF800000000000   ; direct mapping of all physical mem
PAGE_OFFSET     equ DIRECT_MAP_BASE       ; phys -> virt: virt = phys + PAGE_OFFSET

; phys_to_virt(uint64_t phys) -> uint64_t virt
phys_to_virt:
    mov rax, rdi
    add rax, PAGE_OFFSET
    ret

; virt_to_phys(uint64_t virt) -> uint64_t phys
; (สำหรับ kernel virtual addresses ใน direct map)
virt_to_phys:
    mov rax, rdi
    sub rax, PAGE_OFFSET
    ret


; setup_kernel_pgtable(uint64_t* pgd_phys)
; ตั้งค่า kernel mappings ใน PGD ที่กำหนด
; ใช้ตอน boot และตอน create process ใหม่
; Args: rdi = PGD physical address
setup_kernel_pgtable:
    push rbx
    push r12
    push r13
    push r14
    
    mov r12, rdi            ; r12 = new PGD phys addr
    
    ; copy kernel portion ของ PGD จาก initial_page_table
    ; kernel ใช้ entries 256-511 ของ PGD (high half)
    
    ; source: kernel's own PGD (เข้าถึงผ่าน direct map)
    mov rax, cr3
    and rax, ~0xFFF         ; rax = current PGD phys
    add rax, PAGE_OFFSET    ; rax = current PGD virt (direct map)
    
    ; destination: new PGD + high half offset
    mov rbx, r12
    add rbx, PAGE_OFFSET    ; rbx = new PGD virt
    
    ; copy entries 256-511 (4 bytes * 256 = 2KB, แต่ 8 bytes * 256 = 2KB)
    ; index 256 = offset 256*8 = 2048 = 0x800
    lea rsi, [rax + 0x800]
    lea rdi, [rbx + 0x800]
    mov rcx, 256            ; 256 entries
    rep movsq               ; copy 256 quadwords
    
    pop r14
    pop r13
    pop r12
    pop rbx
    ret


; create_process_pgd() -> uint64_t pgd_phys
; สร้าง PGD ใหม่สำหรับ process ใหม่
; Return: rax = physical address ของ PGD page (NULL = error)
create_process_pgd:
    push rbx
    
    ; allocate 1 page สำหรับ PGD
    call alloc_physical_page
    test rax, rax
    jz .done
    mov rbx, rax            ; rbx = pgd phys
    
    ; zero-fill PGD (user space entries = 0)
    mov rdi, rbx
    add rdi, PAGE_OFFSET    ; virt addr
    mov rcx, 4096 / 8
    xor eax, eax
    rep stosq
    
    ; copy kernel entries (high half)
    mov rdi, rbx
    call setup_kernel_pgtable
    
    mov rax, rbx
    
.done:
    pop rbx
    ret
```

---

## 14. Page Table Walk Function (Full)

```nasm
; ============================================================
; pgtable_walk.asm - Complete page table walk utility
; ============================================================

; walk_page_range(mm_struct* mm, uint64_t start, uint64_t end,
;                 pte_callback_fn* fn, void* data)
; วน loop ผ่าน PTEs ใน range [start, end)
; เรียก fn(pte_ptr, addr, data) สำหรับแต่ละ page
; Args: rdi=mm, rsi=start, rdx=end, rcx=callback, r8=data
walk_page_range:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15
    
    mov r12, rdi            ; mm
    mov r13, rsi            ; current addr
    mov r14, rdx            ; end addr
    mov r15, rcx            ; callback
    ; r8 = data (pass through)
    
    mov rax, [r12 + mm_struct.pgd]
    add rax, PAGE_OFFSET    ; virt addr of PGD
    mov rbp, rax            ; rbp = PGD virt base
    
.addr_loop:
    cmp r13, r14
    jge .done
    
    ; --- PGD level ---
    mov rbx, r13
    shr rbx, 39
    and rbx, 0x1FF
    mov rax, [rbp + rbx*8]  ; pgd_entry
    test rax, PTE_PRESENT
    jz .skip_pgd
    and rax, PTE_ADDR_MASK
    add rax, PAGE_OFFSET    ; PUD virt base
    push rax                ; save PUD base
    
    ; --- PUD level ---
    mov rbx, r13
    shr rbx, 30
    and rbx, 0x1FF
    pop rax                 ; restore PUD base
    push rax
    mov rax, [rax + rbx*8]  ; pud_entry
    test rax, PTE_PRESENT
    jz .skip_pud
    test rax, PTE_HUGE      ; 1GB page
    jnz .handle_1gb
    and rax, PTE_ADDR_MASK
    add rax, PAGE_OFFSET    ; PMD virt base
    push rax                ; save PMD base
    
    ; --- PMD level ---
    mov rbx, r13
    shr rbx, 21
    and rbx, 0x1FF
    pop rax                 ; restore PMD base
    push rax
    mov rax, [rax + rbx*8]  ; pmd_entry
    test rax, PTE_PRESENT
    jz .skip_pmd
    test rax, PTE_HUGE      ; 2MB page
    jnz .handle_2mb
    and rax, PTE_ADDR_MASK
    add rax, PAGE_OFFSET    ; PT virt base
    push rax                ; save PT base
    
    ; --- PTE level ---
    mov rbx, r13
    shr rbx, 12
    and rbx, 0x1FF
    pop rax                 ; restore PT base
    lea rax, [rax + rbx*8]  ; pointer to PTE
    
    ; call callback: fn(pte_ptr, addr, data)
    push r8
    push r13
    push r14
    push r15
    push rax
    
    mov rdi, rax            ; pte_ptr
    mov rsi, r13            ; addr
    mov rdx, r8             ; data
    call r15                ; callback
    
    pop rax
    pop r15
    pop r14
    pop r13
    pop r8
    
    add r13, 4096
    pop rax                 ; clean PUD save
    pop rax                 ; clean PGD save (these need careful balancing)
    jmp .addr_loop
    
.handle_2mb:
    pop rax                 ; clean PMD save
    add r13, 0x200000       ; skip 2MB
    pop rax
    pop rax
    jmp .addr_loop
    
.handle_1gb:
    add r13, 0x40000000     ; skip 1GB
    pop rax
    pop rax
    jmp .addr_loop
    
.skip_pmd:
    pop rax                 ; clean PMD push
    ; skip to next PMD boundary
    add r13, 0x200000
    and r13, ~0x1FFFFF
    pop rax
    pop rax
    jmp .addr_loop
    
.skip_pud:
    ; skip to next PUD boundary
    pop rax                 ; clean PUD push
    add r13, 0x40000000
    and r13, ~0x3FFFFFFF
    pop rax
    jmp .addr_loop
    
.skip_pgd:
    pop rax                 ; clean PGD dummy
    ; skip to next PGD boundary (512GB)
    mov rbx, 1
    shl rbx, 39
    add r13, rbx
    mov rbx, ~((1 << 39) - 1)
    and r13, rbx
    jmp .addr_loop
    
.done:
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp
    ret
```

---

## 15. Physical Page Allocator (Simple)

```nasm
; ============================================================
; page_alloc.asm - Simple physical page allocator
; ============================================================
; ใช้ free-list แบบง่าย (kernel จริงๆ ใช้ buddy allocator)

section .bss
; Free page list: สร้าง linked list ของ physical pages
; แต่ละ free page เก็บ pointer ไปยัง next free page
; ที่ virtual address = phys + PAGE_OFFSET
free_page_list:  resq 1     ; head pointer (virt addr ของ free page แรก)
free_page_count: resq 1     ; จำนวน free pages

section .text

; init_page_allocator(uint64_t start_phys, uint64_t end_phys)
; initialize free list จาก physical memory range
; Args: rdi = start physical address, rsi = end physical address
init_page_allocator:
    push rbx
    push r12
    push r13
    
    ; page-align start upward
    add rdi, 0xFFF
    and rdi, ~0xFFF
    ; page-align end downward
    and rsi, ~0xFFF
    
    mov r12, rdi            ; current
    mov r13, rsi            ; end
    mov qword [free_page_list], 0
    mov qword [free_page_count], 0
    
.build_list:
    cmp r12, r13
    jge .done
    
    ; virt addr ของ page นี้
    mov rbx, r12
    add rbx, PAGE_OFFSET
    
    ; เก็บ pointer ไปยัง current head ใน page นี้
    mov rax, [free_page_list]
    mov [rbx], rax
    
    ; update head
    mov [free_page_list], rbx
    inc qword [free_page_count]
    
    add r12, 4096
    jmp .build_list
    
.done:
    pop r13
    pop r12
    pop rbx
    ret


; alloc_physical_page() -> uint64_t phys_addr (0 = OOM)
alloc_physical_page:
    ; ตรวจ free list
    mov rax, [free_page_list]
    test rax, rax
    jz .oom
    
    ; pop from list
    mov rcx, [rax]          ; next
    mov [free_page_list], rcx
    dec qword [free_page_count]
    
    ; convert virt -> phys
    sub rax, PAGE_OFFSET
    ret
    
.oom:
    xor eax, eax
    ret


; free_physical_page(uint64_t phys_addr)
; คืน physical page กลับสู่ free list
free_physical_page:
    ; convert phys -> virt
    add rdi, PAGE_OFFSET    ; rdi = virt addr
    
    ; push to free list
    mov rax, [free_page_list]
    mov [rdi], rax
    mov [free_page_list], rdi
    inc qword [free_page_count]
    ret


; alloc_page_table() -> uint64_t phys_addr (0 = OOM)
; allocate และ zero-fill page table page
alloc_page_table:
    push rbx
    
    call alloc_physical_page
    test rax, rax
    jz .done
    mov rbx, rax
    
    ; zero-fill (page tables ต้องเป็น 0 ก่อนใช้)
    mov rdi, rbx
    add rdi, PAGE_OFFSET
    mov rcx, 4096 / 8
    xor eax, eax
    rep stosq
    
    mov rax, rbx
    
.done:
    pop rbx
    ret
```

---

## 16. Makefile สำหรับ Compile และทดสอบ

```makefile
# Makefile for Virtual Memory Manager demo
# ใช้กับ NASM 2.x และ LD

NASM    = nasm
LD      = ld
QEMU    = qemu-system-x86_64

NASM_FLAGS = -f elf64 -g -F dwarf
LD_FLAGS   = -T kernel.ld -nostdlib

# source files
SRCS = boot.asm \
       vma.asm \
       pgtable.asm \
       pagefault.asm \
       cow.asm \
       brk.asm \
       mmap.asm \
       munmap.asm \
       context_switch.asm \
       page_alloc.asm \
       kernel_map.asm

OBJS = $(SRCS:.asm=.o)

.PHONY: all clean run debug

all: kernel.bin

%.o: %.asm
	$(NASM) $(NASM_FLAGS) -o $@ $<

kernel.elf: $(OBJS)
	$(LD) $(LD_FLAGS) -o $@ $^

kernel.bin: kernel.elf
	objcopy -O binary $< $@

# รัน kernel ใน QEMU (no display, serial output)
run: kernel.bin
	$(QEMU) \
	    -kernel kernel.bin \
	    -m 128M \
	    -serial stdio \
	    -display none \
	    -no-reboot \
	    -no-shutdown

# รันด้วย GDB debug
debug: kernel.bin
	$(QEMU) \
	    -kernel kernel.bin \
	    -m 128M \
	    -serial stdio \
	    -display none \
	    -no-reboot \
	    -no-shutdown \
	    -s -S &
	gdb kernel.elf \
	    -ex "target remote localhost:1234" \
	    -ex "break page_fault_handler" \
	    -ex "continue"

# ทดสอบด้วย unit test (user-space simulation)
test_vmm: test_vmm.c vmm_sim.c
	gcc -g -O0 -o test_vmm test_vmm.c vmm_sim.c
	./test_vmm

clean:
	rm -f $(OBJS) kernel.elf kernel.bin

# แสดง memory map ของ kernel elf
map: kernel.elf
	nm -n $@ | head -50
```

---

## 17. Linker Script

```
/* kernel.ld - Linker script สำหรับ kernel */
OUTPUT_FORMAT("elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
ENTRY(_start)

SECTIONS {
    . = 0xFFFFFFFF80000000;  /* Kernel virtual base */

    .text : AT(0x1000000) {  /* Load at phys 0x1000000 */
        _text_start = .;
        *(.text.boot)        /* Boot code ต้องอยู่ก่อน */
        *(.text*)
        _text_end = .;
    }

    . = ALIGN(4096);
    .rodata : {
        _rodata_start = .;
        *(.rodata*)
        _rodata_end = .;
    }

    . = ALIGN(4096);
    .data : {
        _data_start = .;
        *(.data*)
        _data_end = .;
    }

    . = ALIGN(4096);
    .bss : {
        _bss_start = .;
        *(.bss*)
        *(COMMON)
        _bss_end = .;
    }

    /* Page tables ใน .init.data */
    . = ALIGN(4096);
    .init_pgtable : {
        _init_pgtable_start = .;
        *(.init.pgtable)
        _init_pgtable_end = .;
    }

    _kernel_end = .;
}
```

---

## 18. QEMU Test Commands

```bash
# 1. Build kernel
make all

# 2. รัน kernel ปกติ (serial output ออก terminal)
make run

# 3. Debug ด้วย GDB
make debug
# ใน GDB:
# (gdb) break do_page_fault
# (gdb) continue
# (gdb) info registers cr2 rip rsp
# (gdb) x/4xg $cr3   <- ดู page table

# 4. ดู memory ใน QEMU monitor
# ใส่ option: -monitor stdio แทน -serial
qemu-system-x86_64 -kernel kernel.bin -m 128M -monitor stdio -display none

# QEMU Monitor commands:
# (qemu) info mem          <- แสดง virtual memory layout
# (qemu) info tlb          <- แสดง TLB entries
# (qemu) x /10xg 0xffff800000000000   <- ดู kernel memory
# (qemu) xp /4xg 0x1000000             <- ดู physical memory

# 5. ทดสอบ page fault:
qemu-system-x86_64 \
    -kernel kernel.bin \
    -m 64M \
    -serial file:serial.log \
    -d int,cpu_reset \
    -D qemu_debug.log \
    -no-reboot

# 6. ดู page fault log
cat qemu_debug.log | grep "exception 0xe"   # exception 14 = page fault

# 7. ทดสอบด้วย valgrind (สำหรับ user-space simulation)
gcc -g -O0 -o vmm_test vmm_test.c
valgrind --leak-check=full --track-origins=yes ./vmm_test

# 8. Profile page faults
perf stat -e page-faults ./my_program
```

---

## 19. ตัวอย่าง Boot + VMM Initialization

```nasm
; ============================================================
; boot_vmm.asm - Boot code ที่ initialize VMM
; ============================================================
section .text.boot
bits 64
global _start

_start:
    ; Setup stack
    lea rsp, [rel boot_stack_top]
    
    ; Clear BSS
    lea rdi, [rel _bss_start]
    lea rcx, [rel _bss_end]
    sub rcx, rdi
    shr rcx, 3
    xor eax, eax
    rep stosq
    
    ; Initialize physical page allocator
    ; สมมติ 128MB RAM, เริ่มหลัง kernel end
    lea rdi, [rel _kernel_end]
    sub rdi, 0xFFFFFFFF80000000  ; virt -> phys offset
    add rdi, 0x1000000           ; + kernel phys base
    mov rsi, (128 * 1024 * 1024) ; 128MB
    call init_page_allocator
    
    ; สร้าง initial kernel page table
    ; (boot page table ที่ BIOS/bootloader set ไว้แล้ว)
    ; เราแค่ setup ส่วนที่ kernel ต้องการเพิ่ม
    
    ; Create initial mm_struct สำหรับ kernel thread
    mov rdi, mm_struct_size
    call kmalloc
    test rax, rax
    jz .panic
    mov [kernel_mm], rax
    
    ; ตั้งค่า kernel mm
    mov rbx, rax
    mov rax, cr3
    and rax, ~0xFFF
    mov [rbx + mm_struct.pgd], rax
    ; kernel มองทั้งหมดผ่าน direct map
    
    ; Setup IDT สำหรับ page fault (interrupt 14)
    lea rax, [rel page_fault_handler]
    mov rdi, 14
    mov rsi, rax
    call set_idt_entry
    
    ; Enable interrupts
    sti
    
    ; ทดสอบ: ลอง access uninitialized virtual address
    ; (ควร trigger page fault แล้ว handle ได้)
    mov rdi, 0x1000000  ; จงใจ access uninitialized page
    ; ... (ใน production จะมี VMA สำหรับ kernel heap)
    
    ; Main kernel loop
    call kernel_main
    
.halt:
    hlt
    jmp .halt
    
.panic:
    ; kernel panic
    hlt

section .bss
kernel_mm:       resq 1
boot_stack:      resb 0x4000    ; 16KB boot stack
boot_stack_top:
```

---

## 20. สรุปแนวคิด Virtual Memory Manager

```
┌─────────────────────────────────────────────────────────────┐
│              Virtual Memory Manager Flow                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Process access virtual address X                          │
│           │                                                 │
│           ▼                                                 │
│     MMU translates X -> physical                            │
│           │                                                 │
│    ┌──────┴──────┐                                          │
│    │  Mapped?    │                                          │
│    └──────┬──────┘                                          │
│         Yes │ No                                            │
│             │   └──> Page Fault Handler (#PF)               │
│             │              │                                │
│             │         find_vma(X) ─── found ──> Alloc page  │
│             │              │                    map_page()  │
│             │          not found ──────────────> SEGFAULT   │
│             │                                               │
│             ▼                                               │
│       Access succeeds                                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  fork() COW Flow:                                           │
│                                                             │
│  parent fork() ──> copy page tables ──> mark all READ-ONLY  │
│                                          inc ref_count      │
│                                                             │
│  child/parent writes ──> COW fault ──> allocate new page   │
│                                        copy content         │
│                                        update PTE writable  │
│                                        dec old ref_count    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  VMA Operations:                                            │
│                                                             │
│  mmap()  -> alloc_vma_lazy() -> insert_vma()               │
│  munmap()-> remove_vma_range() -> unmap_pages()             │
│  brk()   -> extend/shrink heap VMA                         │
│  stack   -> grow_down on fault if VM_GROWSDOWN              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Data Structures:

| Struct | หน้าที่ |
|--------|---------|
| `mm_struct` | memory descriptor ของแต่ละ process |
| `vma_t` | ช่วง virtual address region หนึ่งๆ |
| PGD/PUD/PMD/PTE | 4-level page table (512 entries each) |
| `ref_count_table` | นับว่า physical page ถูกใช้ร่วมกี่ process |
| `free_page_list` | physical page allocator free list |

### Key Functions:

| Function | หน้าที่ |
|----------|---------|
| `find_vma()` | หา VMA ที่ครอบคลุม address |
| `map_page()` | เพิ่ม mapping ใน page table |
| `unmap_page()` | ลบ mapping และ flush TLB |
| `get_physaddr()` | แปลง virtual -> physical |
| `do_page_fault()` | handle page fault |
| `handle_cow_fault()` | copy-on-write fault handler |
| `sys_brk()` | ขยาย/ย่อ heap |
| `sys_mmap()` | สร้าง mapping ใหม่ |
| `sys_munmap()` | ลบ mapping |
| `switch_mm()` | เปลี่ยน address space (load CR3) |

---

## แบบฝึกหัด (Exercises)

1. **แก้ไข bug ใน `do_page_fault`:** ในส่วน `handle_anon_fault` มีการ load VMA flags ผิด variable ให้หาและแก้ไข

2. **Implement `msync()`:** flush dirty pages ของ mmap'd file กลับลง disk (hint: ต้องวน PTE ใน range และตรวจ dirty bit)

3. **เพิ่ม PCID support:** แก้ `switch_mm()` ให้ใช้ Process Context ID เพื่อลด TLB flush overhead

4. **Implement `mprotect()`:** เปลี่ยน permission ของ VMA range (hint: update vm_flags และ PTEs ใน range)

5. **Red-Black Tree:** แทนที่ linked list ของ VMA ด้วย red-black tree เพื่อให้ `find_vma()` เป็น O(log n) แทน O(n)

6. **Swap Support:** implement swap-out (write page to disk, mark PTE as swap entry) และ swap-in (read back on fault)

---

*Part 087 - Virtual Memory Manager | Assembly Course*
*เนื้อหา: VMA management, Page tables, Demand paging, COW, brk/mmap/munmap*
