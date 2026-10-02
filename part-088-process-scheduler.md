# Part 088: Process Scheduler Implementation

## บทนำ (Introduction)

Process Scheduler คือหัวใจของระบบปฏิบัติการ มีหน้าที่ตัดสินใจว่า process ใดจะได้ใช้ CPU ในแต่ละช่วงเวลา บทนี้จะอธิบายการ implement scheduler จากระดับ assembly ขึ้นไป รวมถึงโครงสร้างข้อมูล PCB, context switch, timer interrupt และ scheduling algorithms ต่าง ๆ

---

## 1. Process Control Block (PCB / task_struct)

PCB คือโครงสร้างข้อมูลที่เก็บข้อมูลทั้งหมดของ process หนึ่ง ๆ ใน Linux kernel เรียกว่า `task_struct` แต่ในตัวอย่างนี้เราจะเรียกว่า `PCB`

### 1.1 โครงสร้าง PCB ใน Assembly (NASM)

```nasm
; ============================================================
; pcb.inc - Process Control Block definition
; ============================================================

; Process States
PROCESS_RUNNING  equ 0   ; กำลังรัน
PROCESS_READY    equ 1   ; พร้อมรัน รอ CPU
PROCESS_BLOCKED  equ 2   ; รอ I/O หรือ event
PROCESS_ZOMBIE   equ 3   ; จบแล้ว รอ parent wait()
PROCESS_UNUSED   equ 4   ; slot ว่าง

; PCB structure offsets
PCB_PID          equ 0    ; [4 bytes] Process ID
PCB_STATE        equ 4    ; [4 bytes] Process state
PCB_PRIORITY     equ 8    ; [4 bytes] Priority (0=highest)
PCB_QUANTUM      equ 12   ; [4 bytes] Time quantum remaining
PCB_ESP          equ 16   ; [4 bytes] Saved ESP (stack pointer)
PCB_EBP          equ 20   ; [4 bytes] Saved EBP (base pointer)
PCB_EIP          equ 24   ; [4 bytes] Saved EIP (instruction pointer)
PCB_CR3          equ 28   ; [4 bytes] Page directory physical address
PCB_KSTACK       equ 32   ; [4 bytes] Kernel stack top address
PCB_PARENT_PID   equ 36   ; [4 bytes] Parent PID
PCB_EXIT_CODE    equ 40   ; [4 bytes] Exit code (for ZOMBIE)
PCB_FLAGS        equ 44   ; [4 bytes] Process flags
PCB_WAIT_PID     equ 48   ; [4 bytes] PID ที่รอ (BLOCKED state)
PCB_NAME         equ 52   ; [16 bytes] Process name string
PCB_SIZE         equ 68   ; Total PCB size

; Process Flags
PFLAG_KERNEL     equ (1 << 0)  ; Kernel process
PFLAG_USER       equ (1 << 1)  ; User process
PFLAG_FORKABLE   equ (1 << 2)  ; Can be forked
```

### 1.2 PCB Array และ Constants

```nasm
; ============================================================
; scheduler_data.asm - Scheduler data structures
; ============================================================

section .data

MAX_PROCESSES    equ 32          ; จำนวน process สูงสุด
DEFAULT_QUANTUM  equ 10          ; default time quantum (timer ticks)
KSTACK_SIZE      equ 4096        ; kernel stack size = 4KB

; PCB array - เก็บ PCB ทั้งหมด
pcb_table:
    times (MAX_PROCESSES * PCB_SIZE) db 0

; Scheduler state
current_pid:     dd 0            ; PID ของ process ที่รันอยู่
next_pid:        dd 1            ; PID ถัดไปที่จะ assign
tick_count:      dd 0            ; จำนวน timer ticks

; Kernel stacks สำหรับแต่ละ process
; (ใน real OS อาจ allocate dynamically)
kernel_stacks:
    times (MAX_PROCESSES * KSTACK_SIZE) db 0
```

### 1.3 การ Access PCB Fields

```nasm
; ============================================================
; Helper macros สำหรับ access PCB fields
; ============================================================

; GET_PCB_PTR reg, pid
; คำนวณ pointer ไปยัง PCB ของ pid นั้น
; ผล: reg = address ของ PCB
%macro GET_PCB_PTR 2
    mov %1, %2
    imul %1, PCB_SIZE
    add %1, pcb_table
%endmacro

; ตัวอย่างการใช้:
; GET_PCB_PTR eax, 2     ; eax = &pcb_table[2]
; mov ebx, [eax + PCB_STATE]  ; อ่าน state

; ============================================================
; get_current_pcb - คืน pointer ไปยัง PCB ของ current process
; output: eax = pointer to current PCB
; ============================================================
get_current_pcb:
    push ebx
    mov ebx, [current_pid]
    GET_PCB_PTR eax, ebx
    pop ebx
    ret
```

---

## 2. Process States และ State Diagram

```
                    ┌─────────────┐
                    │   UNUSED    │  ← slot ว่างใน PCB table
                    └──────┬──────┘
                           │ create_process()
                           ▼
                    ┌─────────────┐
           ┌───────►│    READY    │◄──────────────────┐
           │        └──────┬──────┘                   │
           │               │ scheduler เลือก          │
           │               │ dispatch()               │
           │               ▼                          │
           │        ┌─────────────┐   quantum หมด     │
           │        │   RUNNING   │──────────────────► │
           │        └──────┬──────┘  preempt          │
           │               │                          │
           │    wait I/O   │ block()                  │
           │    ┌──────────┘                          │
           │    ▼                                     │
           │  ┌─────────────┐   I/O done              │
           │  │   BLOCKED   │──────────────────────── ┘
           │  └─────────────┘  unblock()
           │
           │  exit()
           │  ┌─────────────┐
           └──┤    ZOMBIE   │  รอ parent wait()
              └─────────────┘
                    │ wait() called
                    ▼
              (free PCB slot → UNUSED)
```

### 2.1 State Transition Functions

```nasm
; ============================================================
; set_process_state - เปลี่ยน state ของ process
; input: eax = pid, ebx = new_state
; ============================================================
set_process_state:
    push ecx
    push eax
    
    GET_PCB_PTR ecx, eax        ; ecx = &pcb[pid]
    mov [ecx + PCB_STATE], ebx  ; เปลี่ยน state
    
    pop eax
    pop ecx
    ret

; ============================================================
; block_process - block process ปัจจุบัน
; input: eax = pid ที่รอ (ถ้า 0 = รอ I/O)
; ============================================================
block_process:
    push ebx
    push ecx
    
    mov ebx, [current_pid]
    GET_PCB_PTR ecx, ebx         ; ecx = &pcb[current]
    mov [ecx + PCB_WAIT_PID], eax
    mov dword [ecx + PCB_STATE], PROCESS_BLOCKED
    
    ; trigger reschedule
    call schedule
    
    pop ecx
    pop ebx
    ret

; ============================================================
; unblock_process - unblock process ที่รอ event
; input: eax = pid ที่จะ unblock
; ============================================================
unblock_process:
    push ebx
    push ecx
    
    GET_PCB_PTR ecx, eax         ; ecx = &pcb[pid]
    
    ; ตรวจว่า process นั้น BLOCKED จริง
    cmp dword [ecx + PCB_STATE], PROCESS_BLOCKED
    jne .not_blocked
    
    mov dword [ecx + PCB_STATE], PROCESS_READY
    
.not_blocked:
    pop ecx
    pop ebx
    ret
```

---

## 3. Context Switch

Context switch คือการบันทึก state ของ process ปัจจุบัน และโหลด state ของ process ถัดไป

### 3.1 Context Switch ด้วย PUSHAD/POPAD

```nasm
; ============================================================
; context_switch - สลับ context ระหว่าง process
; input:  esi = pointer to current PCB (ที่จะ save)
;         edi = pointer to next PCB (ที่จะ restore)
; ============================================================
context_switch:
    ; ===== SAVE current context =====
    
    ; บันทึก general-purpose registers
    ; ณ จุดนี้ stack มี: [return addr]
    ; เราจะ save ESP หลัง pushad
    
    pushad                      ; push EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
                                ; (ESP ที่ push คือค่าก่อน pushad)
    
    ; บันทึก ESP หลัง pushad ลงใน PCB
    mov [esi + PCB_ESP], esp
    
    ; บันทึก EBP (อาจซ้ำ แต่ชัดเจนกว่า)
    mov [esi + PCB_EBP], ebp
    
    ; EIP จะถูก save โดย call instruction แล้ว (return address)
    ; ดึง return address จาก stack (อยู่หลัง pushad = esp + 32)
    mov eax, [esp + 32]
    mov [esi + PCB_EIP], eax
    
    ; บันทึก CR3 (page directory)
    mov eax, cr3
    mov [esi + PCB_CR3], eax
    
    ; เปลี่ยน state เป็น READY
    mov dword [esi + PCB_STATE], PROCESS_READY
    
    ; ===== RESTORE next context =====
    
    ; เปลี่ยน state เป็น RUNNING
    mov dword [edi + PCB_STATE], PROCESS_RUNNING
    
    ; อัพเดต current_pid
    mov eax, [edi + PCB_PID]
    mov [current_pid], eax
    
    ; สลับ CR3 (address space)
    mov eax, [edi + PCB_CR3]
    mov cr3, eax                ; TLB flush เกิดขึ้นอัตโนมัติ
    
    ; โหลด ESP ของ next process
    mov esp, [edi + PCB_ESP]
    
    ; Restore registers
    popad                       ; restore EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI
    
    ; ตอนนี้ esp ชี้ไปยัง stack ของ next process
    ; return address อยู่ที่ [esp] = EIP ที่ save ไว้
    ret                         ; กลับไปที่ EIP ของ next process
```

### 3.2 Context Switch แบบ Minimal (ประหยัดกว่า)

```nasm
; ============================================================
; switch_context_minimal
; บันทึกแค่ callee-saved registers (EBX, ESI, EDI, EBP)
; เพราะ caller ต้องบันทึก EAX, ECX, EDX เองอยู่แล้ว
; input: eax = &old_pcb, ebx = &new_pcb
; ============================================================
switch_context_minimal:
    ; Save callee-saved registers ของ current process
    push ebp
    push ebx
    push esi
    push edi
    
    ; บันทึก ESP ลง old PCB
    mov [eax + PCB_ESP], esp
    
    ; Restore ESP ของ new process
    mov esp, [ebx + PCB_ESP]
    
    ; เปลี่ยน page directory ถ้าต่างกัน
    mov ecx, [eax + PCB_CR3]
    mov edx, [ebx + PCB_CR3]
    cmp ecx, edx
    je .same_address_space
    mov cr3, edx               ; switch page directory → TLB flush

.same_address_space:
    ; อัพเดต current_pid
    mov ecx, [ebx + PCB_PID]
    mov [current_pid], ecx
    
    ; Restore callee-saved registers ของ new process
    pop edi
    pop esi
    pop ebx
    pop ebp
    
    ret
```

### 3.3 Kernel Stack Switch

เมื่อ process ทำ syscall หรือ interrupt เกิด CPU จะ switch ไป kernel stack ของ process นั้น

```nasm
; ============================================================
; setup_tss_esp0 - อัพเดต TSS ให้ชี้ไปยัง kernel stack
; ของ process ที่จะ run
; input: eax = &PCB ของ process ถัดไป
; ============================================================
; TSS structure offset สำหรับ ESP0
TSS_ESP0    equ 4    ; offset ของ ESP0 ใน TSS

extern tss_entry    ; TSS ที่ setup ไว้ใน GDT

setup_tss_esp0:
    push ebx
    mov ebx, [eax + PCB_KSTACK]    ; kernel stack top
    mov [tss_entry + TSS_ESP0], ebx
    pop ebx
    ret
```

---

## 4. Timer Interrupt (IRQ0) - PIT 8253/8254

Timer interrupt เป็นกลไกหลักที่ทำให้ scheduler ทำงานแบบ preemptive

### 4.1 การ Setup PIT (Programmable Interval Timer)

```nasm
; ============================================================
; PIT 8253/8254 registers
; ============================================================

PIT_CHANNEL0    equ 0x40    ; Channel 0 data port
PIT_CHANNEL1    equ 0x41    ; Channel 1 data port
PIT_CHANNEL2    equ 0x42    ; Channel 2 data port
PIT_CMD         equ 0x43    ; Command/Mode register

; PIT Command byte:
; Bit 7-6: Select channel (00=ch0, 01=ch1, 10=ch2, 11=readback)
; Bit 5-4: Access mode (01=LSB, 10=MSB, 11=LSB then MSB)
; Bit 3-1: Operating mode (011=mode 3 square wave)
; Bit 0:   BCD/Binary (0=binary)

PIT_CMD_CH0     equ 0x36    ; Channel 0, LSB+MSB, Mode 3, Binary

; PIT base frequency
PIT_BASE_FREQ   equ 1193182 ; Hz (1.193182 MHz)

; Target frequency
TIMER_FREQ      equ 100     ; 100 Hz = 10ms per tick

; Divisor = PIT_BASE_FREQ / TIMER_FREQ
PIT_DIVISOR     equ (PIT_BASE_FREQ / TIMER_FREQ)  ; = 11931

; ============================================================
; init_pit - Initialize PIT channel 0
; ============================================================
init_pit:
    ; ส่ง command byte
    mov al, PIT_CMD_CH0
    out PIT_CMD, al
    
    ; ส่ง divisor (LSB ก่อน แล้วค่อย MSB)
    mov ax, PIT_DIVISOR
    out PIT_CHANNEL0, al    ; LSB
    mov al, ah
    out PIT_CHANNEL0, al    ; MSB
    
    ret

; ============================================================
; ตัวอย่าง: frequency ต่าง ๆ
; 1000 Hz  → divisor = 1193  → 1ms tick
;  100 Hz  → divisor = 11931 → 10ms tick
;   50 Hz  → divisor = 23863 → 20ms tick
;   18.2 Hz → divisor = 65535 → default BIOS
; ============================================================
```

### 4.2 Timer Interrupt Handler

```nasm
; ============================================================
; timer_handler - IRQ0 interrupt handler
; เรียกโดย CPU เมื่อ PIT ยิง interrupt
; ============================================================

section .text

; IDT entry ต้องตั้งค่าให้ชี้มาที่นี่
; (เราตั้งค่า IDT ใน part ก่อนหน้า)

timer_handler:
    ; CPU push FLAGS, CS, EIP อัตโนมัติแล้ว
    ; Push general registers
    pushad
    push ds
    push es
    push fs
    push gs
    
    ; โหลด kernel data segment
    mov ax, 0x10            ; kernel data selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    ; เพิ่ม tick count
    inc dword [tick_count]
    
    ; ส่ง EOI (End of Interrupt) ไปยัง PIC
    mov al, 0x20
    out 0x20, al            ; Master PIC
    
    ; ลด quantum ของ process ปัจจุบัน
    call get_current_pcb    ; eax = &current_pcb
    dec dword [eax + PCB_QUANTUM]
    
    ; ถ้า quantum ยังไม่หมด ไม่ต้อง switch
    cmp dword [eax + PCB_QUANTUM], 0
    jg .no_switch
    
    ; Quantum หมด → schedule process ถัดไป
    call schedule
    
.no_switch:
    ; Restore registers
    pop gs
    pop fs
    pop es
    pop ds
    popad
    
    iret                    ; กลับจาก interrupt (restore EIP, CS, FLAGS)
```

---

## 5. Scheduling Algorithms

### 5.1 FIFO (First-In First-Out / Run to Completion)

```nasm
; ============================================================
; FIFO Scheduler - process รันจนจบ ไม่มี preemption
; (เหมาะกับ batch processing)
; ============================================================

; FIFO queue structure
fifo_queue:
    .head:  dd 0            ; index ของ head
    .tail:  dd 0            ; index ของ tail
    .count: dd 0            ; จำนวน element
    .data:  times MAX_PROCESSES dd 0  ; เก็บ PID

; ============================================================
; fifo_enqueue - เพิ่ม PID เข้า queue
; input: eax = pid
; output: eax = 0 (success), -1 (queue full)
; ============================================================
fifo_enqueue:
    push ebx
    push ecx
    
    ; ตรวจ queue full
    mov ecx, [fifo_queue.count]
    cmp ecx, MAX_PROCESSES
    jge .full
    
    ; เพิ่ม pid ที่ tail
    mov ebx, [fifo_queue.tail]
    mov [fifo_queue.data + ebx*4], eax
    
    ; เลื่อน tail (circular)
    inc ebx
    cmp ebx, MAX_PROCESSES
    jl .no_wrap
    xor ebx, ebx            ; wrap around
.no_wrap:
    mov [fifo_queue.tail], ebx
    inc dword [fifo_queue.count]
    
    xor eax, eax            ; return 0 (success)
    jmp .done
    
.full:
    mov eax, -1
    
.done:
    pop ecx
    pop ebx
    ret

; ============================================================
; fifo_dequeue - ดึง PID ออกจาก queue
; output: eax = pid, หรือ -1 ถ้า empty
; ============================================================
fifo_dequeue:
    push ebx
    
    ; ตรวจ empty
    cmp dword [fifo_queue.count], 0
    je .empty
    
    ; ดึง pid จาก head
    mov ebx, [fifo_queue.head]
    mov eax, [fifo_queue.data + ebx*4]
    
    ; เลื่อน head
    inc ebx
    cmp ebx, MAX_PROCESSES
    jl .no_wrap
    xor ebx, ebx
.no_wrap:
    mov [fifo_queue.head], ebx
    dec dword [fifo_queue.count]
    
    jmp .done
    
.empty:
    mov eax, -1
    
.done:
    pop ebx
    ret

; ============================================================
; fifo_schedule - เลือก process ถัดไปแบบ FIFO
; output: eax = pid ของ process ถัดไป, -1 ถ้าไม่มี
; ============================================================
fifo_schedule:
    call fifo_dequeue
    ret
```

### 5.2 Round-Robin Scheduler

```nasm
; ============================================================
; Round-Robin Scheduler
; แต่ละ process ได้รับ time quantum แล้วถูก preempt
; ============================================================

section .data

rr_current_idx:  dd 0       ; index ปัจจุบันใน PCB table

section .text

; ============================================================
; rr_next_process - หา process ถัดไปแบบ Round-Robin
; output: eax = pid, หรือ -1 ถ้าไม่มี READY process
; ============================================================
rr_next_process:
    push ebx
    push ecx
    push edx
    push esi
    
    mov ecx, [rr_current_idx]
    mov edx, ecx            ; บันทึก start index
    
.loop:
    ; เลื่อน index ถัดไป (circular)
    inc ecx
    cmp ecx, MAX_PROCESSES
    jl .check_wrap_done
    xor ecx, ecx            ; wrap to 0
.check_wrap_done:
    
    ; ถ้าวนครบรอบแล้วยังไม่เจอ
    cmp ecx, edx
    je .not_found
    
    ; คำนวณ pointer ไปยัง PCB
    mov esi, ecx
    imul esi, PCB_SIZE
    add esi, pcb_table
    
    ; ตรวจว่า process นี้ READY หรือไม่
    cmp dword [esi + PCB_STATE], PROCESS_READY
    jne .loop
    
    ; พบ process READY
    mov [rr_current_idx], ecx
    mov eax, [esi + PCB_PID]
    jmp .found
    
.not_found:
    mov eax, -1
    
.found:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret

; ============================================================
; schedule - ฟังก์ชัน scheduler หลัก (Round-Robin)
; เรียกจาก timer_handler หรือ block()
; ============================================================
schedule:
    pushad
    
    ; หา next process
    call rr_next_process
    cmp eax, -1
    je .run_idle            ; ไม่มี process ready → idle
    
    ; ตรวจว่า next != current
    cmp eax, [current_pid]
    je .same_process        ; process เดิม → reset quantum
    
    ; ดึง PCB pointers
    mov ebx, [current_pid]
    push eax                ; save next pid
    
    GET_PCB_PTR esi, ebx    ; esi = &current_pcb
    
    pop eax
    GET_PCB_PTR edi, eax    ; edi = &next_pcb
    
    ; Reset quantum ของ next process
    mov dword [edi + PCB_QUANTUM], DEFAULT_QUANTUM
    
    ; อัพเดต TSS ESP0 สำหรับ next process
    push edi
    mov eax, edi
    call setup_tss_esp0
    pop edi
    
    ; ทำ context switch
    ; (ส่งผ่าน esi=old, edi=new)
    call context_switch
    jmp .done
    
.same_process:
    ; Reset quantum
    call get_current_pcb
    mov dword [eax + PCB_QUANTUM], DEFAULT_QUANTUM
    jmp .done
    
.run_idle:
    ; ไม่มี process ready → รัน idle task
    call idle_task
    
.done:
    popad
    ret
```

### 5.3 Priority-Based Scheduler

```nasm
; ============================================================
; Priority Scheduler
; process ที่มี priority สูงกว่า (ค่าน้อยกว่า) ได้รัน priority
; ============================================================

MAX_PRIORITY     equ 0      ; highest priority
MIN_PRIORITY     equ 15     ; lowest priority

; ============================================================
; priority_find_next - หา READY process ที่ priority สูงสุด
; output: eax = pid, หรือ -1 ถ้าไม่มี
; ============================================================
priority_find_next:
    push ebx
    push ecx
    push edx
    push esi
    
    mov edx, -1             ; best pid so far
    mov ecx, MIN_PRIORITY + 1   ; worst priority so far
    
    xor ebx, ebx            ; index = 0
    
.loop:
    cmp ebx, MAX_PROCESSES
    jge .done
    
    ; คำนวณ PCB pointer
    mov esi, ebx
    imul esi, PCB_SIZE
    add esi, pcb_table
    
    ; ตรวจ state
    cmp dword [esi + PCB_STATE], PROCESS_READY
    jne .next
    
    ; เปรียบเทียบ priority
    mov eax, [esi + PCB_PRIORITY]
    cmp eax, ecx
    jge .next               ; priority ไม่ดีกว่า
    
    ; อัพเดต best
    mov ecx, eax
    mov edx, [esi + PCB_PID]
    
.next:
    inc ebx
    jmp .loop
    
.done:
    mov eax, edx            ; return best pid (หรือ -1)
    
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret

; ============================================================
; priority_schedule - scheduler แบบ priority
; ============================================================
priority_schedule:
    pushad
    
    call priority_find_next
    cmp eax, -1
    je .idle
    
    cmp eax, [current_pid]
    je .no_switch
    
    ; ทำ context switch
    mov ebx, [current_pid]
    push eax
    GET_PCB_PTR esi, ebx
    pop eax
    GET_PCB_PTR edi, eax
    
    mov dword [edi + PCB_QUANTUM], DEFAULT_QUANTUM
    call context_switch
    jmp .done
    
.no_switch:
    call get_current_pcb
    mov dword [eax + PCB_QUANTUM], DEFAULT_QUANTUM
    jmp .done
    
.idle:
    call idle_task
    
.done:
    popad
    ret
```

---

## 6. Idle Task

```nasm
; ============================================================
; idle_task - รันเมื่อไม่มี process ใดพร้อมทำงาน
; ใช้ HLT เพื่อประหยัด CPU จนกว่าจะมี interrupt
; ============================================================

section .data

idle_pcb_initialized: dd 0

section .text

idle_loop:
    ; Enable interrupts แล้ว halt
    ; CPU จะตื่นเมื่อ interrupt เกิด (เช่น timer IRQ0)
    sti
    hlt
    
    ; หลังตื่น → ตรวจว่ามี process READY หรือยัง
    call rr_next_process
    cmp eax, -1
    je idle_loop            ; ยังไม่มี → halt ต่อ
    
    ; มี process แล้ว → schedule
    jmp schedule

idle_task:
    ; สร้าง idle process ถ้ายังไม่มี
    cmp dword [idle_pcb_initialized], 0
    jne .already_init
    
    ; Init idle PCB (PID 0)
    mov esi, pcb_table      ; pcb[0]
    mov dword [esi + PCB_PID], 0
    mov dword [esi + PCB_STATE], PROCESS_RUNNING
    mov dword [esi + PCB_PRIORITY], MIN_PRIORITY
    mov dword [esi + PCB_QUANTUM], DEFAULT_QUANTUM
    
    mov dword [idle_pcb_initialized], 1
    
.already_init:
    jmp idle_loop
```

---

## 7. Process Creation

### 7.1 allocate_pcb - หา PCB slot ว่าง

```nasm
; ============================================================
; allocate_pcb - หา PCB slot ว่าง
; output: eax = pointer to free PCB, หรือ 0 ถ้าเต็ม
; ============================================================
allocate_pcb:
    push ebx
    push ecx
    
    xor ebx, ebx            ; index = 0
    
.loop:
    cmp ebx, MAX_PROCESSES
    jge .full
    
    mov ecx, ebx
    imul ecx, PCB_SIZE
    add ecx, pcb_table
    
    cmp dword [ecx + PCB_STATE], PROCESS_UNUSED
    je .found
    
    inc ebx
    jmp .loop
    
.found:
    ; ล้าง PCB ทั้งหมด
    push edi
    push ecx
    mov edi, ecx
    xor eax, eax
    mov ecx, PCB_SIZE / 4   ; ล้าง dword ละครั้ง
    rep stosd
    pop ecx
    pop edi
    
    mov eax, ecx            ; return pointer
    jmp .done
    
.full:
    xor eax, eax            ; return 0
    
.done:
    pop ecx
    pop ebx
    ret
```

### 7.2 create_process - สร้าง Process ใหม่

```nasm
; ============================================================
; create_process - สร้าง process ใหม่
; input:  eax = entry point (function pointer)
;         ebx = priority
; output: eax = pid (หรือ -1 ถ้า error)
; ============================================================
create_process:
    push ebp
    mov ebp, esp
    push esi
    push edi
    push ecx
    push edx
    
    mov edx, eax            ; save entry point
    mov ecx, ebx            ; save priority
    
    ; หา PCB slot
    call allocate_pcb
    test eax, eax
    jz .error
    
    mov esi, eax            ; esi = &new_pcb
    
    ; Assign PID
    mov eax, [next_pid]
    mov [esi + PCB_PID], eax
    inc dword [next_pid]
    
    ; Set state และ priority
    mov dword [esi + PCB_STATE], PROCESS_READY
    mov [esi + PCB_PRIORITY], ecx
    mov dword [esi + PCB_QUANTUM], DEFAULT_QUANTUM
    
    ; Set parent PID
    mov eax, [current_pid]
    mov [esi + PCB_PARENT_PID], eax
    
    ; Allocate kernel stack
    ; (ใน real OS ต้อง allocate pages จาก memory manager)
    ; สำหรับตัวอย่างนี้ ใช้ static array
    mov eax, [esi + PCB_PID]
    imul eax, KSTACK_SIZE
    add eax, kernel_stacks
    add eax, KSTACK_SIZE    ; top of stack
    mov [esi + PCB_KSTACK], eax
    
    ; Setup initial kernel stack สำหรับ context switch
    ; stack ต้องมี: edi, esi, ebx, ebp, return_address
    ; เหมือนกับว่า switch_context_minimal เพิ่งถูกเรียก
    mov edi, [esi + PCB_KSTACK]
    sub edi, 4
    mov dword [edi], edx    ; EIP = entry point (return address)
    sub edi, 4
    mov dword [edi], 0      ; EBP = 0
    sub edi, 4
    mov dword [edi], 0      ; EBX = 0
    sub edi, 4
    mov dword [edi], 0      ; ESI = 0
    sub edi, 4
    mov dword [edi], 0      ; EDI = 0
    
    ; บันทึก ESP เริ่มต้น
    mov [esi + PCB_ESP], edi
    
    ; Setup page directory (ใช้ kernel page directory เป็น default)
    extern kernel_page_dir
    mov eax, [kernel_page_dir]
    mov [esi + PCB_CR3], eax
    
    ; Return PID
    mov eax, [esi + PCB_PID]
    jmp .done
    
.error:
    mov eax, -1
    
.done:
    pop edx
    pop ecx
    pop edi
    pop esi
    pop ebp
    ret
```

### 7.3 fork() - Copy Process

```nasm
; ============================================================
; sys_fork - สร้าง process ใหม่โดย copy parent
; output: eax = child PID (ใน parent), 0 (ใน child), -1 (error)
; ============================================================
sys_fork:
    pushad
    
    ; หา PCB slot สำหรับ child
    call allocate_pcb
    test eax, eax
    jz .error
    
    mov edi, eax            ; edi = &child_pcb
    
    ; Copy parent PCB ไปยัง child
    call get_current_pcb
    mov esi, eax            ; esi = &parent_pcb
    
    push ecx
    push edi
    push esi
    mov ecx, PCB_SIZE / 4
    rep movsd               ; copy PCB
    pop esi
    pop edi
    pop ecx
    
    ; Assign ใหม่: PID, parent PID, state
    mov eax, [next_pid]
    mov [edi + PCB_PID], eax
    inc dword [next_pid]
    
    mov dword [edi + PCB_STATE], PROCESS_READY
    
    mov eax, [esi + PCB_PID]
    mov [edi + PCB_PARENT_PID], eax  ; child's parent = current
    
    ; Allocate kernel stack สำหรับ child
    mov eax, [edi + PCB_PID]
    imul eax, KSTACK_SIZE
    add eax, kernel_stacks
    add eax, KSTACK_SIZE
    mov [edi + PCB_KSTACK], eax
    
    ; Copy kernel stack ของ parent ไปยัง child
    ; (เพื่อให้ child เริ่มที่จุดเดียวกัน)
    mov esi, [esi + PCB_KSTACK]
    sub esi, KSTACK_SIZE
    mov ecx, KSTACK_SIZE / 4
    push edi
    mov edi, eax
    sub edi, KSTACK_SIZE
    rep movsd
    pop edi
    
    ; ปรับ ESP ของ child ให้ชี้ไปยัง kernel stack ใหม่
    ; offset เดิม: parent_esp - parent_kstack_base
    ; ใน child:    child_kstack_base + offset
    ; TODO: ใน full implementation ต้องคำนวณ offset จาก stack base
    
    ; ตั้งค่า return value = 0 สำหรับ child
    ; (EAX ใน saved context บน stack)
    mov dword [edi + PCB_ESP], 0    ; placeholder
    
    ; Clone page tables (copy-on-write)
    ; (ใน real implementation ต้อง clone page directory)
    
    ; Return child PID ให้ parent
    mov eax, [edi + PCB_PID]
    
    popad
    mov eax, [edi + PCB_PID]    ; parent ได้รับ child PID
    ret
    
.error:
    popad
    mov eax, -1
    ret
```

---

## 8. Process Destruction

### 8.1 sys_exit - จบ Process

```nasm
; ============================================================
; sys_exit - process จบการทำงาน
; input: eax = exit code
; ============================================================
sys_exit:
    cli                     ; ปิด interrupt ชั่วคราว
    push eax                ; save exit code
    
    ; เปลี่ยน state เป็น ZOMBIE
    call get_current_pcb
    mov esi, eax
    
    pop eax
    mov [esi + PCB_EXIT_CODE], eax
    mov dword [esi + PCB_STATE], PROCESS_ZOMBIE
    
    ; Wake parent ถ้ากำลัง wait()
    mov eax, [esi + PCB_PARENT_PID]
    push esi
    call wake_waiting_parent
    pop esi
    
    ; Free resources (memory, file descriptors, etc.)
    ; (ใน full implementation)
    
    sti
    
    ; Schedule process ถัดไป
    call schedule
    
    ; ไม่ควรมาถึงนี่
    jmp $

; ============================================================
; wake_waiting_parent - ปลุก parent ถ้ากำลัง wait()
; input: eax = parent_pid
; ============================================================
wake_waiting_parent:
    push esi
    push ebx
    
    GET_PCB_PTR esi, eax    ; esi = &parent_pcb
    
    ; ตรวจว่า parent BLOCKED
    cmp dword [esi + PCB_STATE], PROCESS_BLOCKED
    jne .done
    
    ; ตรวจว่า parent รอ child นี้
    mov ebx, [current_pid]
    cmp [esi + PCB_WAIT_PID], ebx
    jne .done
    
    ; Unblock parent
    mov dword [esi + PCB_STATE], PROCESS_READY
    
.done:
    pop ebx
    pop esi
    ret
```

### 8.2 sys_wait - รอ Child Process

```nasm
; ============================================================
; sys_wait - รอ child process จบ
; input:  eax = child_pid (หรือ -1 รอ child ใด ๆ)
; output: eax = exit code ของ child, ebx = child pid
; ============================================================
sys_wait:
    push esi
    push ecx
    push edx
    
    mov ecx, eax            ; save target pid
    
.retry:
    ; ค้นหา ZOMBIE child
    xor edx, edx            ; index
    
.search_loop:
    cmp edx, MAX_PROCESSES
    jge .not_found
    
    mov esi, edx
    imul esi, PCB_SIZE
    add esi, pcb_table
    
    ; ตรวจ state = ZOMBIE
    cmp dword [esi + PCB_STATE], PROCESS_ZOMBIE
    jne .next_proc
    
    ; ตรวจ parent_pid = current
    mov eax, [current_pid]
    cmp [esi + PCB_PARENT_PID], eax
    jne .next_proc
    
    ; ถ้าระบุ pid เฉพาะ ตรวจ pid ด้วย
    cmp ecx, -1
    je .found               ; รอ any child → ใช้ child นี้
    cmp [esi + PCB_PID], ecx
    je .found
    
.next_proc:
    inc edx
    jmp .search_loop
    
.not_found:
    ; ยังไม่มี zombie child → block จนกว่าจะมี
    call get_current_pcb
    mov dword [eax + PCB_STATE], PROCESS_BLOCKED
    mov [eax + PCB_WAIT_PID], ecx
    
    call schedule           ; รอจน unblock
    jmp .retry              ; ตรวจใหม่หลัง unblock
    
.found:
    ; ได้ zombie child แล้ว
    mov eax, [esi + PCB_EXIT_CODE]  ; exit code
    mov ebx, [esi + PCB_PID]        ; child pid
    
    ; Free PCB slot
    mov dword [esi + PCB_STATE], PROCESS_UNUSED
    
    pop edx
    pop ecx
    pop esi
    ret
```

---

## 9. Inter-Process Communication (IPC) - Wake Blocked Processes

### 9.1 Simple Event Wake

```nasm
; ============================================================
; ipc_wake_by_event - ปลุก process ที่รอ event
; input: eax = event ID
; ============================================================
ipc_wake_by_event:
    push esi
    push ecx
    
    mov ecx, eax            ; event id
    xor eax, eax            ; index
    
.loop:
    cmp eax, MAX_PROCESSES
    jge .done
    
    mov esi, eax
    imul esi, PCB_SIZE
    add esi, pcb_table
    
    ; ตรวจ BLOCKED และ รอ event นี้
    cmp dword [esi + PCB_STATE], PROCESS_BLOCKED
    jne .next
    
    cmp [esi + PCB_WAIT_PID], ecx   ; WAIT_PID ใช้เป็น event ID ด้วย
    jne .next
    
    ; Unblock
    mov dword [esi + PCB_STATE], PROCESS_READY
    
.next:
    inc eax
    jmp .loop
    
.done:
    pop ecx
    pop esi
    ret
```

### 9.2 Semaphore (Simple)

```nasm
; ============================================================
; Semaphore structure
; ============================================================

SEM_VALUE   equ 0       ; current count
SEM_SIZE    equ 4

; sem_wait - decrement semaphore, block ถ้า 0
; input: eax = pointer to semaphore
sem_wait:
    push ebx
    mov ebx, eax            ; save semaphore pointer
    
.retry:
    ; ลด value
    dec dword [ebx + SEM_VALUE]
    jns .acquired           ; ถ้าไม่ negative → ได้ semaphore
    
    ; Value < 0 → block
    inc dword [ebx + SEM_VALUE]  ; คืนค่ากลับ
    
    ; Block จนกว่าจะมีคน signal
    ; (ใน real implementation ต้องมี wait queue)
    xor eax, eax
    call block_process
    jmp .retry
    
.acquired:
    pop ebx
    ret

; sem_signal - increment semaphore, wake blocked process
; input: eax = pointer to semaphore
sem_signal:
    inc dword [eax + SEM_VALUE]
    ; ถ้ามี waiter → wake them (simplified)
    ret
```

---

## 10. Complete Scheduler: Timer Handler → Save → Choose → Restore

นี่คือ complete flow ของ scheduler:

```nasm
; ============================================================
; complete_timer_handler - Full scheduler flow
; IRQ0 → save → choose → restore
; ============================================================

section .text

complete_timer_handler:
    ; ===== PHASE 1: Save context =====
    pushad
    push ds
    push es
    push fs
    push gs
    
    ; Setup kernel segments
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    ; ===== PHASE 2: PIC EOI =====
    mov al, 0x20
    out 0x20, al
    
    ; ===== PHASE 3: Update tick count =====
    inc dword [tick_count]
    
    ; ===== PHASE 4: Update quantum =====
    call get_current_pcb    ; eax = &current_pcb
    mov esi, eax
    
    dec dword [esi + PCB_QUANTUM]
    cmp dword [esi + PCB_QUANTUM], 0
    jg .restore_and_return  ; quantum ยังเหลือ → ไม่ switch
    
    ; ===== PHASE 5: Choose next process =====
    call rr_next_process    ; eax = next_pid หรือ -1
    cmp eax, -1
    je .run_idle
    
    ; ตรวจว่าเหมือน current หรือไม่
    cmp eax, [current_pid]
    je .reset_quantum
    
    ; ===== PHASE 6: Switch context =====
    GET_PCB_PTR edi, eax    ; edi = &next_pcb
    
    ; บันทึก ESP ปัจจุบัน (หลัง push all ด้านบน)
    mov [esi + PCB_ESP], esp
    mov dword [esi + PCB_STATE], PROCESS_READY
    
    ; อัพเดต current_pid
    mov eax, [edi + PCB_PID]
    mov [current_pid], eax
    
    ; Reset quantum ของ next
    mov dword [edi + PCB_QUANTUM], DEFAULT_QUANTUM
    mov dword [edi + PCB_STATE], PROCESS_RUNNING
    
    ; อัพเดต TSS
    push edi
    mov eax, edi
    call setup_tss_esp0
    pop edi
    
    ; Switch address space (CR3)
    mov eax, [esi + PCB_CR3]
    mov ebx, [edi + PCB_CR3]
    cmp eax, ebx
    je .same_cr3
    mov cr3, ebx            ; TLB flush

.same_cr3:
    ; Switch stack → restore next process's context
    mov esp, [edi + PCB_ESP]
    
    pop gs
    pop fs
    pop es
    pop ds
    popad
    
    iret                    ; return ไปยัง next process
    
.reset_quantum:
    mov dword [esi + PCB_QUANTUM], DEFAULT_QUANTUM
    jmp .restore_and_return
    
.run_idle:
    ; ไม่มี process → idle (HLT)
    pop gs
    pop fs
    pop es
    pop ds
    popad
    sti
    hlt
    ; หลัง interrupt → timer จะยิงอีกรอบ
    iret
    
.restore_and_return:
    ; Restore และกลับ process เดิม
    pop gs
    pop fs
    pop es
    pop ds
    popad
    iret
```

---

## 11. Simple Round-Robin Implementation (ครบสมบูรณ์)

ด้านล่างนี้คือ implementation ที่สมบูรณ์กว่า โดยใช้ PCB array + current index:

```nasm
; ============================================================
; scheduler_rr.asm - Simple Round-Robin Scheduler (complete)
; ============================================================

%include "pcb.inc"

section .data

; PCB table
pcbs:   times (MAX_PROCESSES * PCB_SIZE) db 0

; scheduler variables
rr_idx:         dd 0
cur_pid:        dd 0
next_pid_alloc: dd 1
tick:           dd 0

; idle process state
idle_running:   dd 0

section .text

global scheduler_init
global scheduler_add_process
global scheduler_tick
global scheduler_get_current

; ============================================================
; scheduler_init - Initialize scheduler
; ============================================================
scheduler_init:
    push eax
    push ecx
    push edi
    
    ; ล้าง PCB table
    mov edi, pcbs
    xor eax, eax
    mov ecx, (MAX_PROCESSES * PCB_SIZE) / 4
    rep stosd
    
    ; Mark all as UNUSED
    xor eax, eax
.init_loop:
    cmp eax, MAX_PROCESSES
    jge .init_done
    
    push eax
    imul eax, PCB_SIZE
    add eax, pcbs
    mov dword [eax + PCB_STATE], PROCESS_UNUSED
    pop eax
    inc eax
    jmp .init_loop
    
.init_done:
    pop edi
    pop ecx
    pop eax
    ret

; ============================================================
; scheduler_add_process - เพิ่ม process เข้า scheduler
; input:  eax = entry_point
;         ebx = priority (0-15)
; output: eax = pid หรือ -1
; ============================================================
scheduler_add_process:
    push ebp
    mov ebp, esp
    push esi
    push edi
    push ecx
    push edx
    
    mov ecx, eax    ; save entry point
    mov edx, ebx    ; save priority
    
    ; ค้นหา UNUSED PCB
    xor eax, eax
.find_loop:
    cmp eax, MAX_PROCESSES
    jge .no_slot
    
    mov esi, eax
    imul esi, PCB_SIZE
    add esi, pcbs
    
    cmp dword [esi + PCB_STATE], PROCESS_UNUSED
    je .found_slot
    
    inc eax
    jmp .find_loop
    
.no_slot:
    mov eax, -1
    jmp .exit
    
.found_slot:
    ; ตั้งค่า PCB
    mov eax, [next_pid_alloc]
    mov [esi + PCB_PID], eax
    inc dword [next_pid_alloc]
    
    mov dword [esi + PCB_STATE], PROCESS_READY
    mov [esi + PCB_PRIORITY], edx
    mov dword [esi + PCB_QUANTUM], DEFAULT_QUANTUM
    
    ; Setup stack: entry point + initial frame
    ; (ใช้ static kernel stack area)
    mov eax, [esi + PCB_PID]
    dec eax
    imul eax, KSTACK_SIZE
    add eax, kernel_stacks
    add eax, KSTACK_SIZE        ; top of stack
    
    ; วาง return address (entry_point) ไว้ที่ top
    sub eax, 4
    mov [eax], ecx              ; entry point
    sub eax, 4
    mov dword [eax], 0          ; EBP
    sub eax, 4
    mov dword [eax], 0          ; EBX
    sub eax, 4
    mov dword [eax], 0          ; ESI
    sub eax, 4
    mov dword [eax], 0          ; EDI
    
    mov [esi + PCB_ESP], eax
    mov [esi + PCB_KSTACK], eax
    
    ; Return PID
    mov eax, [esi + PCB_PID]
    
.exit:
    pop edx
    pop ecx
    pop edi
    pop esi
    pop ebp
    ret

; ============================================================
; scheduler_tick - เรียกจาก timer interrupt
; ============================================================
scheduler_tick:
    push eax
    push esi
    
    inc dword [tick]
    
    ; ลด quantum
    mov eax, [cur_pid]
    imul eax, PCB_SIZE
    add eax, pcbs
    mov esi, eax
    
    dec dword [esi + PCB_QUANTUM]
    cmp dword [esi + PCB_QUANTUM], 0
    jg .no_preempt
    
    ; หา next
    call .find_next_ready
    cmp eax, -1
    je .no_preempt
    
    ; Switch
    push eax
    mov eax, esi            ; old PCB
    pop ebx                 ; new PCB index
    
    push ebx
    imul ebx, PCB_SIZE
    add ebx, pcbs           ; new PCB pointer
    
    call do_switch          ; eax=old, ebx=new
    pop ebx
    
.no_preempt:
    pop esi
    pop eax
    ret

; ============================================================
; .find_next_ready - local helper
; ============================================================
.find_next_ready:
    push ecx
    push edx
    
    mov ecx, [rr_idx]
    mov edx, ecx            ; save start
    
.fnr_loop:
    inc ecx
    cmp ecx, MAX_PROCESSES
    jl .fnr_check
    xor ecx, ecx
    
.fnr_check:
    cmp ecx, edx
    je .fnr_none
    
    push eax
    mov eax, ecx
    imul eax, PCB_SIZE
    add eax, pcbs
    cmp dword [eax + PCB_STATE], PROCESS_READY
    pop eax
    jne .fnr_loop
    
    mov [rr_idx], ecx
    mov eax, ecx
    jmp .fnr_done
    
.fnr_none:
    mov eax, -1
    
.fnr_done:
    pop edx
    pop ecx
    ret

; ============================================================
; do_switch - ทำ context switch จริง ๆ
; input: eax = &old_pcb, ebx = &new_pcb
; ============================================================
do_switch:
    ; บันทึก ESP ลง old PCB
    mov [eax + PCB_ESP], esp
    mov dword [eax + PCB_STATE], PROCESS_READY
    
    ; อัพเดต current_pid
    mov ecx, [ebx + PCB_PID]
    mov [cur_pid], ecx
    
    mov dword [ebx + PCB_STATE], PROCESS_RUNNING
    mov dword [ebx + PCB_QUANTUM], DEFAULT_QUANTUM
    
    ; Switch address space ถ้าต่าง
    mov ecx, [eax + PCB_CR3]
    mov edx, [ebx + PCB_CR3]
    cmp ecx, edx
    je .same_space
    mov cr3, edx

.same_space:
    ; Switch stack
    mov esp, [ebx + PCB_ESP]
    ret

; ============================================================
; scheduler_get_current - return current pid
; output: eax = current pid
; ============================================================
scheduler_get_current:
    mov eax, [cur_pid]
    ret
```

---

## 12. Kernel Stack และ Ring Transition

เมื่อ user process ทำ interrupt/syscall CPU จะ switch ไป kernel mode และเปลี่ยน stack ไปใช้ kernel stack ของ process นั้น

```nasm
; ============================================================
; TSS (Task State Segment) - ใช้สำหรับ ring 0/3 transition
; ============================================================

struc TSS
    .prev_task  resd 1    ; link to previous task
    .esp0       resd 1    ; ESP for ring 0
    .ss0        resd 1    ; SS for ring 0
    .esp1       resd 1    ; ESP for ring 1
    .ss1        resd 1    ; SS for ring 1
    .esp2       resd 1    ; ESP for ring 2
    .ss2        resd 1    ; SS for ring 2
    .cr3        resd 1    ; page directory
    .eip        resd 1
    .eflags     resd 1
    .eax        resd 1
    .ecx        resd 1
    .edx        resd 1
    .ebx        resd 1
    .esp        resd 1
    .ebp        resd 1
    .esi        resd 1
    .edi        resd 1
    .es         resw 1, resw 1
    .cs         resw 1, resw 1
    .ss         resw 1, resw 1
    .ds         resw 1, resw 1
    .fs         resw 1, resw 1
    .gs         resw 1, resw 1
    .ldt        resw 1, resw 1
    .trap       resw 1
    .io_bitmap  resw 1
endstruc

section .data
align 8
tss_entry:
    times TSS_size db 0

section .text

; ============================================================
; setup_tss - Initialize TSS
; ============================================================
setup_tss:
    ; ตั้งค่า SS0 = kernel data segment
    mov word [tss_entry + TSS.ss0], 0x10
    
    ; ESP0 จะถูก update ทุกครั้งที่ switch process
    ; ดู setup_tss_esp0
    
    ; Load TSS descriptor ใน GDT แล้ว ltr
    mov ax, 0x28            ; TSS selector ใน GDT
    ltr ax
    
    ret
```

---

## 13. Makefile สำหรับ Build

```makefile
# Makefile for Process Scheduler
# NASM 2.x + LD + QEMU

AS      = nasm
LD      = ld
QEMU    = qemu-system-i386

ASFLAGS = -f elf32
LDFLAGS = -T linker.ld -m elf_i386

# Source files
SRCS = boot.asm \
       gdt.asm \
       idt.asm \
       pic.asm \
       pit.asm \
       scheduler.asm \
       process.asm \
       kmain.asm

OBJS = $(SRCS:.asm=.o)

KERNEL = kernel.bin

all: $(KERNEL)

$(KERNEL): $(OBJS)
	$(LD) $(LDFLAGS) -o $@ $^

%.o: %.asm
	$(AS) $(ASFLAGS) -o $@ $<

# สร้าง floppy disk image
floppy.img: $(KERNEL)
	dd if=/dev/zero of=$@ bs=512 count=2880
	dd if=$(KERNEL) of=$@ conv=notrunc

# รัน QEMU
run: floppy.img
	$(QEMU) -fda $< -monitor stdio

# รัน QEMU พร้อม debug
debug: floppy.img
	$(QEMU) -fda $< -s -S &
	gdb -ex "target remote :1234" \
	    -ex "symbol-file $(KERNEL)" \
	    -ex "break kmain"

# Build สำหรับ kernel เฉย ๆ ไม่มี floppy
kernel: $(KERNEL)

clean:
	rm -f $(OBJS) $(KERNEL) floppy.img

.PHONY: all run debug clean kernel
```

### linker.ld

```ld
/* linker.ld - Linker script สำหรับ x86 kernel */
OUTPUT_FORMAT("elf32-i386")
ENTRY(start)

SECTIONS
{
    . = 0x100000;           /* โหลดที่ 1MB */

    .text ALIGN(4096) : {
        *(.text)
    }

    .data ALIGN(4096) : {
        *(.data)
    }

    .bss ALIGN(4096) : {
        *(COMMON)
        *(.bss)
    }
    
    kernel_end = .;
}
```

---

## 14. QEMU Test Commands

```bash
# Build ทุกอย่าง
make all

# รัน kernel ใน QEMU (พร้อม serial output)
qemu-system-i386 -kernel kernel.bin \
                 -serial stdio \
                 -d int,cpu_reset \
                 -no-reboot

# รัน พร้อม monitor (สำหรับ debug)
qemu-system-i386 -kernel kernel.bin \
                 -monitor telnet:127.0.0.1:1234,server,nowait \
                 -serial stdio

# รัน พร้อม GDB stub
qemu-system-i386 -kernel kernel.bin \
                 -s -S \
                 -serial stdio

# เชื่อมต่อ GDB
gdb kernel.bin
(gdb) target remote :1234
(gdb) break schedule
(gdb) continue

# ดู timer interrupt count ผ่าน QEMU monitor
# (ต่อ monitor แล้วพิมพ์)
info registers
info pic

# Dump memory บริเวณ PCB table
(gdb) x/32xw &pcb_table

# ดู current process
(gdb) x/4xw &current_pid

# ติดตาม context switch
(gdb) break context_switch
(gdb) commands
> printf "Switching: curr=%d next=%d\n", *(int*)(&current_pid), $eax
> continue
> end
```

---

## 15. Debugging Scheduler

### 15.1 Debug Print Functions

```nasm
; ============================================================
; debug_print_pcb - แสดงข้อมูล PCB ผ่าน serial port
; input: eax = pointer to PCB
; (สำหรับ debug เท่านั้น)
; ============================================================

; Serial port registers (COM1)
SERIAL_PORT     equ 0x3F8
SERIAL_THR      equ 0       ; Transmitter Holding Register
SERIAL_LSR      equ 5       ; Line Status Register
SERIAL_LSR_THRE equ (1<<5)  ; Transmit Holding Register Empty

; ============================================================
; serial_putc - ส่ง 1 ตัวอักษร ไป serial
; input: al = character
; ============================================================
serial_putc:
    push dx
    push ax
    
.wait:
    mov dx, SERIAL_PORT + SERIAL_LSR
    in al, dx
    test al, SERIAL_LSR_THRE
    jz .wait
    
    pop ax
    mov dx, SERIAL_PORT + SERIAL_THR
    out dx, al
    
    pop dx
    ret

; ============================================================
; debug_dump_scheduler - dump scheduler state
; ============================================================
debug_dump_scheduler:
    push eax
    push ebx
    push ecx
    push esi
    
    ; พิมพ์ header
    mov esi, .msg_header
    call serial_puts
    
    ; วนลูปแสดง PCB ทุกตัวที่ไม่ UNUSED
    xor ecx, ecx
    
.loop:
    cmp ecx, MAX_PROCESSES
    jge .done
    
    mov esi, ecx
    imul esi, PCB_SIZE
    add esi, pcb_table
    
    cmp dword [esi + PCB_STATE], PROCESS_UNUSED
    je .next
    
    ; แสดง PID และ state
    mov eax, [esi + PCB_PID]
    call debug_print_hex
    
    mov al, ':'
    call serial_putc
    
    mov eax, [esi + PCB_STATE]
    call debug_print_state
    
    mov al, 0x0A        ; newline
    call serial_putc
    
.next:
    inc ecx
    jmp .loop
    
.done:
    pop esi
    pop ecx
    pop ebx
    pop eax
    ret

section .data
.msg_header: db "=== Scheduler State ===", 0x0A, 0

state_names:
    .running: db "RUNNING", 0
    .ready:   db "READY  ", 0
    .blocked: db "BLOCKED", 0
    .zombie:  db "ZOMBIE ", 0
    .unused:  db "UNUSED ", 0
```

### 15.2 Scheduler Statistics

```nasm
; ============================================================
; Scheduler Statistics (สำหรับ performance monitoring)
; ============================================================

section .data

sched_stats:
    .total_switches:    dd 0    ; จำนวน context switch ทั้งหมด
    .idle_ticks:        dd 0    ; ticks ที่วิ่ง idle
    .last_switch_tick:  dd 0    ; tick ที่ switch ครั้งล่าสุด

section .text

; ============================================================
; update_stats - อัพเดต statistics หลัง context switch
; ============================================================
update_stats:
    inc dword [sched_stats.total_switches]
    
    mov eax, [tick_count]
    mov [sched_stats.last_switch_tick], eax
    
    ret

; ============================================================
; print_sched_stats - แสดง statistics
; ============================================================
print_sched_stats:
    push eax
    push esi
    
    mov esi, .msg_switches
    call serial_puts
    mov eax, [sched_stats.total_switches]
    call debug_print_dec
    
    mov esi, .msg_ticks
    call serial_puts
    mov eax, [tick_count]
    call debug_print_dec
    
    mov al, 0x0A
    call serial_putc
    
    pop esi
    pop eax
    ret

section .data
.msg_switches: db "Total switches: ", 0
.msg_ticks:    db " | Total ticks: ", 0
```

---

## 16. ตัวอย่าง Process ที่รันบน Scheduler

```nasm
; ============================================================
; test_processes.asm - ตัวอย่าง process ที่ใช้ทดสอบ scheduler
; ============================================================

section .text

; ============================================================
; process_a - Process A: นับและพิมพ์
; ============================================================
process_a:
    push ebx
    xor ebx, ebx            ; counter = 0
    
.loop:
    ; พิมพ์ "A: <count>"
    mov esi, .msg
    call serial_puts
    mov eax, ebx
    call debug_print_dec
    mov al, 0x0A
    call serial_putc
    
    inc ebx
    
    ; หน่วงเวลา (busy wait)
    mov ecx, 1000000
.delay:
    loop .delay
    
    cmp ebx, 10
    jl .loop
    
    ; จบ process
    xor eax, eax
    int 0x80                ; sys_exit(0)

section .data
.msg: db "Process A: ", 0

section .text

; ============================================================
; process_b - Process B: ทำงานลักษณะอื่น
; ============================================================
process_b:
    push ebx
    mov ebx, 100            ; countdown
    
.loop:
    mov esi, .msg
    call serial_puts
    mov eax, ebx
    call debug_print_dec
    mov al, 0x0A
    call serial_putc
    
    dec ebx
    
    mov ecx, 500000
.delay:
    loop .delay
    
    test ebx, ebx
    jnz .loop
    
    mov eax, 42             ; exit code = 42
    int 0x80

section .data
.msg: db "Process B: ", 0

section .text

; ============================================================
; kmain - kernel main, setup และ launch processes
; ============================================================
kmain:
    ; Init subsystems
    call gdt_init
    call idt_init
    call pic_init
    call init_pit
    call setup_tss
    call scheduler_init
    
    ; สร้าง process A
    mov eax, process_a
    mov ebx, 5              ; priority 5
    call scheduler_add_process
    
    ; สร้าง process B
    mov eax, process_b
    mov ebx, 5              ; priority 5
    call scheduler_add_process
    
    ; Enable interrupts → timer จะขับ scheduler
    sti
    
    ; Idle loop
.idle:
    hlt
    jmp .idle
```

---

## 17. Advanced: Multilevel Queue Scheduler

```nasm
; ============================================================
; Multilevel Queue Scheduler
; หลาย queue แต่ละ queue มี priority ต่างกัน
; ============================================================

NUM_QUEUES      equ 4       ; จำนวน priority levels

; Queue structure (circular buffer สำหรับแต่ละ level)
struc MLQ_QUEUE
    .head   resd 1
    .tail   resd 1
    .count  resd 1
    .data   resd MAX_PROCESSES
endstruc

section .data

mlq_queues:
    times (NUM_QUEUES * MLQ_QUEUE_size) db 0

; Quantum per level (level 0 = highest priority, ได้ quantum น้อย)
mlq_quantums:
    dd 2    ; level 0: 2 ticks (interactive)
    dd 4    ; level 1: 4 ticks
    dd 8    ; level 2: 8 ticks
    dd 16   ; level 3: 16 ticks (batch)

section .text

; ============================================================
; mlq_enqueue - เพิ่ม process เข้า queue ตาม level
; input: eax = pid, ebx = queue level (0-3)
; ============================================================
mlq_enqueue:
    push esi
    push ecx
    push edx
    
    ; คำนวณ pointer ไปยัง queue
    mov esi, ebx
    imul esi, MLQ_QUEUE_size
    add esi, mlq_queues
    
    ; ตรวจ full
    cmp dword [esi + MLQ_QUEUE.count], MAX_PROCESSES
    jge .full
    
    ; เพิ่มที่ tail
    mov ecx, [esi + MLQ_QUEUE.tail]
    mov [esi + MLQ_QUEUE.data + ecx*4], eax
    
    inc ecx
    cmp ecx, MAX_PROCESSES
    jl .no_wrap
    xor ecx, ecx
.no_wrap:
    mov [esi + MLQ_QUEUE.tail], ecx
    inc dword [esi + MLQ_QUEUE.count]
    
    xor eax, eax
    jmp .done
    
.full:
    mov eax, -1
    
.done:
    pop edx
    pop ecx
    pop esi
    ret

; ============================================================
; mlq_schedule - เลือก process จาก multilevel queue
; output: eax = pid, ebx = level, หรือ eax=-1
; ============================================================
mlq_schedule:
    push esi
    push ecx
    
    xor ecx, ecx            ; level = 0 (highest)
    
.check_level:
    cmp ecx, NUM_QUEUES
    jge .no_process
    
    mov esi, ecx
    imul esi, MLQ_QUEUE_size
    add esi, mlq_queues
    
    cmp dword [esi + MLQ_QUEUE.count], 0
    je .next_level
    
    ; Dequeue จาก level นี้
    mov ebx, [esi + MLQ_QUEUE.head]
    mov eax, [esi + MLQ_QUEUE.data + ebx*4]
    
    inc ebx
    cmp ebx, MAX_PROCESSES
    jl .no_head_wrap
    xor ebx, ebx
.no_head_wrap:
    mov [esi + MLQ_QUEUE.head], ebx
    dec dword [esi + MLQ_QUEUE.count]
    
    mov ebx, ecx            ; return level
    jmp .found
    
.next_level:
    inc ecx
    jmp .check_level
    
.no_process:
    mov eax, -1
    
.found:
    pop ecx
    pop esi
    ret

; ============================================================
; mlq_demote - ย้าย process ลง level ต่ำกว่า ถ้าใช้ quantum หมด
; input: eax = pid, ebx = current level
; ============================================================
mlq_demote:
    ; ถ้าอยู่ level ต่ำสุดแล้ว ไม่ย้าย
    cmp ebx, NUM_QUEUES - 1
    jge .lowest
    
    inc ebx                 ; level + 1 (ต่ำกว่า)
    call mlq_enqueue
    ret
    
.lowest:
    call mlq_enqueue        ; อยู่ level เดิม
    ret
```

---

## 18. สรุป (Summary)

### 18.1 Flow การทำงาน

```
Boot → Init GDT/IDT/PIC/PIT → Init Scheduler → Create Processes → STI
         │
         └─► PIT IRQ0 fires every 10ms
                 │
                 ▼
         timer_handler:
             1. PUSHAD / push segments
             2. EOI to PIC
             3. tick++
             4. quantum-- for current process
             5. if quantum <= 0:
                    a. Find next READY process (round-robin)
                    b. Save ESP → current PCB
                    c. Load ESP from next PCB
                    d. Switch CR3 if different
                    e. Update current_pid, TSS.ESP0
             6. POPAD / pop segments
             7. IRET → resume process (old or new)
```

### 18.2 Key Registers

| Register | บทบาทใน Scheduler |
|----------|-------------------|
| ESP      | kernel stack pointer ของแต่ละ process |
| CR3      | page directory address (address space) |
| EIP      | instruction pointer (บันทึกโดย CALL/IRET) |
| EFLAGS   | processor flags (บันทึกโดย PUSHAD/IRET) |

### 18.3 Context Switch Timing

- **Timer-based**: IRQ0 ทุก 10ms (100Hz)
- **Voluntary**: process เรียก `block()`, `yield()`, หรือ `exit()`
- **I/O wait**: process รอ I/O แล้วถูก block อัตโนมัติ

### 18.4 Files สำคัญ

| File | เนื้อหา |
|------|---------|
| `pcb.inc` | PCB structure definitions |
| `scheduler.asm` | Round-robin scheduler |
| `process.asm` | Process create/exit/fork/wait |
| `timer.asm` | PIT init + timer handler |
| `context.asm` | Context switch implementation |

---

## 19. แบบฝึกหัด (Exercises)

1. **Basic**: แก้ไข `DEFAULT_QUANTUM` และสังเกตว่า processes สลับกันบ่อยขึ้นหรือน้อยลง
2. **Intermediate**: เพิ่ม `sys_yield()` ที่ให้ process สละ CPU โดยสมัครใจ
3. **Intermediate**: Implement priority aging ที่เพิ่ม priority ให้ process ที่รอนาน
4. **Advanced**: เพิ่ม Completely Fair Scheduler (CFS) แบบง่าย โดยใช้ virtual runtime
5. **Advanced**: Implement sleep() ที่ block process ตามจำนวน milliseconds

---

## 20. ข้อควรระวัง (Pitfalls)

```
1. RACE CONDITION: ถ้า scheduler ถูกเรียกขณะ context switch กำลังทำอยู่
   → ต้อง disable interrupt ระหว่าง critical section

2. STACK OVERFLOW: kernel stack ขนาด 4KB อาจน้อยเกินไปถ้า interrupt nesting
   → ตั้ง KSTACK_SIZE ให้ใหญ่พอ หรือใช้ double fault handler

3. TSS ESP0: ต้อง update ทุกครั้งที่ switch process
   → ถ้าลืม user process จะ fault เมื่อทำ syscall

4. CR3 FLUSH: การเขียน CR3 จะ flush TLB ทั้งหมด
   → ถ้า process อยู่ address space เดียวกัน ไม่ต้อง switch CR3

5. ZOMBIE LEAK: ถ้า parent ไม่เรียก wait() → zombie accumulate
   → init process (PID 1) ต้อง adopt orphan processes
```

---

*จบ Part 088: Process Scheduler Implementation*

*ส่วนถัดไป Part 089: Memory Management - kmalloc/kfree Implementation*
