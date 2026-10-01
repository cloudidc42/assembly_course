# Part 006: x86 Architecture เชิงลึก (Deep Dive)

**ระดับ:** พื้นฐาน-กลาง  
**เวลาที่ใช้:** 5-7 ชั่วโมง  
**ความต่อเนื่อง:** ต่อจาก Part 005

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:

- อธิบายประวัติและวิวัฒนาการของ x86 architecture ตั้งแต่ 8086 จนถึง modern CPU
- เข้าใจ register ทุกประเภทใน x86: General Purpose, Segment, Control, Debug
- อธิบาย AX/EAX/RAX hierarchy และรู้ว่าจะใช้ขนาดไหนเมื่อไร
- เข้าใจความแตกต่างระหว่าง Real Mode, Protected Mode และ Long Mode
- อ่านและตีความ FLAGS/EFLAGS/RFLAGS register ทุก bit
- ใช้คำสั่ง CPUID เพื่อตรวจสอบความสามารถของ CPU
- เข้าใจ x86-64 extensions รวมถึง REX prefix และ registers R8-R15
- ใช้ Control Registers และ Debug Registers ได้ (ระดับพื้นฐาน)

---

## 1. ประวัติและวิวัฒนาการ x86 Architecture

### 1.1 ต้นกำเนิด: Intel 8086 (1978)

```
ไทม์ไลน์วิวัฒนาการ x86:

1978  │ Intel 8086        │ 16-bit, 1MHz-10MHz, Real Mode only
      │                   │ Registers: AX, BX, CX, DX, SI, DI, BP, SP
      │                   │ Address space: 1MB (20-bit address bus)
      │
1982  │ Intel 80286       │ 16-bit, Protected Mode เพิ่มเข้ามา
      │                   │ Address space: 16MB (24-bit address bus)
      │                   │ Virtual memory support
      │
1985  │ Intel 80386 (386) │ 32-bit registers! (EAX, EBX, ...)
      │                   │ Address space: 4GB (32-bit address bus)
      │                   │ Paging support เพิ่มเข้ามา
      │                   │ Real Mode + Protected Mode + Virtual 8086 Mode
      │
1989  │ Intel 80486 (486) │ 32-bit, FPU built-in (บางรุ่น)
      │                   │ On-chip cache (L1 cache)
      │                   │ Pipeline architecture
      │
1993  │ Intel Pentium     │ Superscalar (2 pipelines ทำงานพร้อมกัน)
      │                   │ 64-bit data bus (แต่ยัง 32-bit registers)
      │                   │ MMX (ต่อมา) สำหรับ multimedia
      │
1995  │ Intel Pentium Pro │ Out-of-order execution
      │                   │ L2 cache on-die
      │
1999  │ Intel Pentium III │ SSE (Streaming SIMD Extensions)
      │                   │ 128-bit XMM registers เพิ่มเข้ามา
      │
2000  │ AMD Athlon 64     │ x86-64 / AMD64 extension!
      │ (2003)            │ 64-bit registers (RAX, RBX, ...)
      │                   │ R8-R15 registers ใหม่ 8 ตัว
      │
2004  │ Intel Core/       │ Multi-core CPUs
      │ Intel 64          │ Intel นำ x86-64 มาใช้ (Intel 64)
      │
2006+ │ Intel Core 2/     │ SSE4, AVX extensions
      │ Nehalem/Sandy     │ 256-bit YMM registers
      │ Bridge            │
      │
2017+ │ AMD Zen           │ Ryzen, Threadripper
      │ Architecture      │ AVX-512 (บางรุ่น), improved IPC
      │
2020+ │ Intel Tiger Lake  │ AVX-512 support
      │ AMD Zen 3/4       │ PCIe 4.0/5.0, DDR5
```

### 1.2 ทำไม x86 ถึงยังอยู่มาจนถึงทุกวันนี้?

x86 architecture มีอายุมากกว่า 45 ปีแต่ยังคงครองตลาด PC/Server ด้วยเหตุผล:

1. **Backward Compatibility** - โปรแกรม DOS เก่าๆ ยังรันได้บน CPU ใหม่
2. **Mature Ecosystem** - toolchain, OS, libraries สมบูรณ์มาก
3. **Performance** - Intel/AMD ลงทุนมหาศาลในการ optimize
4. **Network Effect** - ทุกคนใช้ จึงมีคนพัฒนา software ให้มาก

---

## 2. x86 Registers ทั้งหมด

### 2.1 General Purpose Registers (GPRs)

#### 16-bit (8086 era)
```
┌────────────────────────────────────────────────────────────┐
│                    AX (16-bit)                             │
├────────────────────┬───────────────────────────────────────┤
│    AH (8-bit)      │           AL (8-bit)                  │
└────────────────────┴───────────────────────────────────────┘
```

#### 32-bit (386 era) - ขยายจาก 16-bit
```
┌────────────────────────────────────────────────────────────┐
│                    EAX (32-bit)                            │
├────────────────────────────┬───────────────────────────────┤
│        (upper 16 bits)     │          AX (16-bit)          │
│        (ไม่มีชื่อเฉพาะ)    ├───────────────┬───────────────┤
│                            │    AH (8-bit) │   AL (8-bit)  │
└────────────────────────────┴───────────────┴───────────────┘
```

#### 64-bit (x86-64 era) - ขยายจาก 32-bit
```
┌────────────────────────────────────────────────────────────┐
│                    RAX (64-bit)                            │
├────────────────────────────────┬───────────────────────────┤
│        (upper 32 bits)         │         EAX (32-bit)      │
│                                ├───────────────┬───────────┤
│                                │  (upper 16b)  │  AX(16b)  │
│                                │               ├─────┬─────┤
│                                │               │ AH  │ AL  │
└────────────────────────────────┴───────────────┴─────┴─────┘
```

### 2.2 ตารางสรุป General Purpose Registers

| 64-bit | 32-bit | 16-bit | 8-bit High | 8-bit Low | ใช้งานหลัก |
|--------|--------|--------|------------|-----------|-----------|
| RAX    | EAX    | AX     | AH         | AL        | Accumulator, Return value |
| RBX    | EBX    | BX     | BH         | BL        | Base register, Callee-saved |
| RCX    | ECX    | CX     | CH         | CL        | Counter (loops), 4th arg |
| RDX    | EDX    | DX     | DH         | DL        | Data, 3rd arg, I/O |
| RSI    | ESI    | SI     | -          | SIL       | Source Index, 2nd arg |
| RDI    | EDI    | DI     | -          | DIL       | Dest Index, 1st arg |
| RBP    | EBP    | BP     | -          | BPL       | Base Pointer (stack frame) |
| RSP    | ESP    | SP     | -          | SPL       | Stack Pointer |
| R8     | R8D    | R8W    | -          | R8B       | 5th argument |
| R9     | R9D    | R9W    | -          | R9B       | 6th argument |
| R10    | R10D   | R10W   | -          | R10B      | General purpose |
| R11    | R11D   | R11W   | -          | R11B      | General purpose |
| R12    | R12D   | R12W   - | -         | R12B      | Callee-saved |
| R13    | R13D   | R13W   | -          | R13B      | Callee-saved |
| R14    | R14D   | R14W   | -          | R14B      | Callee-saved |
| R15    | R15D   | R15W   | -          | R15B      | Callee-saved |

### 2.3 สิ่งที่ต้องรู้เกี่ยวกับ Register Widths

```nasm
; ไฟล์: register_demo.asm
; คอมไพล์: nasm -f elf64 register_demo.asm -o register_demo.o
;          ld register_demo.o -o register_demo
; รัน:     ./register_demo

section .data
    fmt db "RAX = 0x%016lx", 10, 0    ; format string

section .text
global _start

_start:
    ; ===== ทดลองขนาด register =====
    
    ; ใส่ค่าใน RAX (64-bit)
    mov rax, 0x1122334455667788    ; RAX = 0x1122334455667788
    
    ; อ่านค่าจาก EAX (32-bit lower half ของ RAX)
    ; EAX จะเห็น: 0x55667788
    ; สำคัญ! การ write ไปที่ EAX จะ zero-extend ขึ้นไปที่ RAX
    
    mov eax, 0xDEADBEEF            ; RAX กลายเป็น 0x00000000DEADBEEF
    ;                              ; upper 32 bits ถูก clear เป็น 0 อัตโนมัติ!
    
    ; แต่ AX (16-bit) ไม่ clear upper bits
    mov rax, 0x1122334455667788    ; reset
    mov ax, 0xFFFF                 ; RAX กลายเป็น 0x112233445566FFFF
    ;                              ; เปลี่ยนแค่ 16 bits ล่าง, ที่เหลือคงเดิม
    
    ; AH และ AL (8-bit)
    mov rax, 0x1122334455667788    ; reset
    mov ah, 0x00                   ; RAX กลายเป็น 0x1122334455660088
    ;                              ; AH = bits 8-15, AL = bits 0-7
    mov al, 0x99                   ; RAX กลายเป็น 0x1122334455660099
    
    ; Exit
    mov rax, 60                    ; syscall: exit
    xor rdi, rdi                   ; exit code 0
    syscall
```

### 2.4 กฎสำคัญ: 32-bit Write Zero-Extends!

นี่คือกฎที่คนมักลืมและทำให้เกิด bug:

```nasm
; ตัวอย่างที่แสดงให้เห็น zero-extension
section .text
global _start

_start:
    ; ตั้งค่า RAX เป็น 64-bit value
    mov rax, 0xFFFFFFFFFFFFFFFF    ; RAX = 0xFFFFFFFFFFFFFFFF (ทุก bit = 1)
    
    ; Write ไปที่ EAX (32-bit) -> zero-extends!
    mov eax, 1                     ; RAX = 0x0000000000000001
    ;                              ; ไม่ใช่ 0xFFFFFFFF00000001 !
    
    ; เปรียบเทียบกับ 16-bit และ 8-bit write
    mov rax, 0xFFFFFFFFFFFFFFFF    ; reset
    mov ax, 1                      ; RAX = 0xFFFFFFFFFFFF0001
    ;                              ; upper bits คงเดิม!
    
    mov rax, 0xFFFFFFFFFFFFFFFF    ; reset
    mov al, 1                      ; RAX = 0xFFFFFFFFFFFFFF01
    ;                              ; upper bits คงเดิม!
    
    ; สรุป:
    ; 32-bit write: zero-extends ทั้ง 64 bits
    ; 16-bit write: เปลี่ยนแค่ 16 bits ล่าง
    ;  8-bit write: เปลี่ยนแค่ 8 bits ที่ระบุ
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 3. Segment Registers

### 3.1 ทำไมถึงมี Segment Registers?

ใน 8086 มี 16-bit address bus แต่ต้องการเข้าถึง memory มากกว่า 64KB จึงออกแบบ segmented memory model:

```
Physical Address = Segment Register * 16 + Offset
                 = (Segment << 4) + Offset

ตัวอย่าง:
CS = 0x1000, IP = 0x0100
Physical Address = 0x1000 * 16 + 0x0100
                 = 0x10000 + 0x0100
                 = 0x10100
```

### 3.2 Segment Registers ทั้ง 6 ตัว

| Register | ชื่อเต็ม | การใช้งาน |
|----------|----------|-----------|
| CS | Code Segment | ชี้ไปที่ code segment ที่กำลังทำงาน |
| DS | Data Segment | default segment สำหรับ data access |
| ES | Extra Segment | extra segment สำหรับ string operations |
| FS | F Segment | general purpose (OS ใช้เก็บ thread-local storage) |
| GS | G Segment | general purpose (OS ใช้เก็บ kernel data) |
| SS | Stack Segment | ชี้ไปที่ stack segment |

### 3.3 การใช้งาน Segment Registers

```nasm
; ตัวอย่าง Segment Override Prefix
; (ใน Real Mode หรือ 16-bit code)

section .data
    mydata db 0x42

section .text
global _start

_start:
    ; การเข้าถึง memory แบบปกติ (ใช้ DS เป็น default)
    mov al, [mydata]               ; DS:[mydata]
    
    ; การใช้ Segment Override (ES override)
    ; mov al, [es:mydata]          ; ใช้ ES แทน DS
    
    ; ใน 64-bit mode:
    ; CS, DS, ES, SS ถูก treat เป็น base = 0 (flat model)
    ; FS และ GS ยังคงสามารถ set base ได้ผ่าน MSR
    
    ; Linux ใช้ FS สำหรับ Thread Local Storage (TLS)
    ; Windows ใช้ GS สำหรับ Thread Information Block (TIB)
    
    ; อ่านค่าจาก FS (Thread Local Storage)
    ; mov rax, [fs:0]              ; อ่านค่าแรกใน TLS
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 3.4 Segment Registers ใน 64-bit Mode

```
ใน Long Mode (64-bit):
- CS, DS, ES, SS: base = 0 เสมอ, limit = ไม่จำกัด
  → เหมือน flat memory model
  → ค่าใน register ยังมีความหมาย (privilege level, etc.)
  
- FS, GS: ยังคง configurable base ได้
  → ผ่าน WRMSR instruction หรือ syscall
  → Linux: FS.base = TLS base address
  → Windows: GS.base = TEB (Thread Environment Block)
```

---

## 4. FLAGS/EFLAGS/RFLAGS Register

### 4.1 โครงสร้าง FLAGS Register

```
RFLAGS (64-bit):
Bit  63...22: Reserved (ไม่ใช้)
Bit  21: ID     - CPUID instruction support
Bit  20: VIP    - Virtual Interrupt Pending
Bit  19: VIF    - Virtual Interrupt Flag
Bit  18: AC     - Alignment Check
Bit  17: VM     - Virtual-8086 Mode
Bit  16: RF     - Resume Flag (debug)
Bit  15: Reserved
Bit  14: NT     - Nested Task
Bit  13-12: IOPL - I/O Privilege Level (0-3)
Bit  11: OF     - Overflow Flag ⭐
Bit  10: DF     - Direction Flag ⭐
Bit   9: IF     - Interrupt Enable Flag ⭐
Bit   8: TF     - Trap Flag (single-step debug)
Bit   7: SF     - Sign Flag ⭐
Bit   6: ZF     - Zero Flag ⭐
Bit   5: Reserved
Bit   4: AF     - Auxiliary Carry Flag (BCD arithmetic)
Bit   3: Reserved
Bit   2: PF     - Parity Flag
Bit   1: Reserved (always 1)
Bit   0: CF     - Carry Flag ⭐
```

### 4.2 Flags ที่สำคัญที่สุด

```nasm
; ไฟล์: flags_demo.asm
; คอมไพล์: nasm -f elf64 flags_demo.asm -o flags_demo.o && ld flags_demo.o -o flags_demo

section .data
    ; Messages
    msg_zf      db "ZF set: result was zero", 10, 0
    msg_nozf    db "ZF clear: result was non-zero", 10, 0
    msg_sf      db "SF set: result was negative", 10, 0
    msg_nosf    db "SF clear: result was non-negative", 10, 0
    msg_cf      db "CF set: carry/borrow occurred", 10, 0
    msg_nocf    db "CF clear: no carry/borrow", 10, 0
    msg_of      db "OF set: overflow occurred", 10, 0
    msg_noof    db "OF clear: no overflow", 10, 0

section .text
global _start

; Helper function: print null-terminated string
; Input: RSI = string address, RDX = length
print_str:
    push rax
    push rdi
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; fd: stdout
    syscall
    pop rdi
    pop rax
    ret

_start:
    ; ============================================
    ; ทดสอบ Zero Flag (ZF)
    ; ZF = 1 เมื่อผลลัพธ์เป็น 0
    ; ============================================
    
    ; Test 1: 5 - 5 = 0 → ZF = 1
    mov rax, 5
    sub rax, 5          ; rax = 0, ZF = 1
    
    jz .zf_set          ; กระโดดถ้า ZF = 1
    ; ZF ไม่ set
    mov rsi, msg_nozf
    mov rdx, 31
    call print_str
    jmp .test_sf
.zf_set:
    ; ZF set
    mov rsi, msg_zf
    mov rdx, 24
    call print_str
    
.test_sf:
    ; ============================================
    ; ทดสอบ Sign Flag (SF)
    ; SF = 1 เมื่อผลลัพธ์เป็น negative (MSB = 1)
    ; ============================================
    
    ; Test: 3 - 5 = -2 → SF = 1
    mov rax, 3
    sub rax, 5          ; rax = -2 (0xFFFFFFFFFFFFFFFE), SF = 1
    
    js .sf_set          ; กระโดดถ้า SF = 1 (negative)
    mov rsi, msg_nosf
    mov rdx, 34
    call print_str
    jmp .test_cf
.sf_set:
    mov rsi, msg_sf
    mov rdx, 28
    call print_str
    
.test_cf:
    ; ============================================
    ; ทดสอบ Carry Flag (CF)
    ; CF = 1 เมื่อเกิด carry (unsigned overflow)
    ; ============================================
    
    ; Test: 0xFFFFFFFFFFFFFFFF + 1 → CF = 1
    mov rax, 0xFFFFFFFFFFFFFFFF    ; max unsigned 64-bit value
    add rax, 1                      ; overflow! CF = 1, RAX = 0
    
    jc .cf_set          ; กระโดดถ้า CF = 1
    mov rsi, msg_nocf
    mov rdx, 28
    call print_str
    jmp .test_of
.cf_set:
    mov rsi, msg_cf
    mov rdx, 30
    call print_str
    
.test_of:
    ; ============================================
    ; ทดสอบ Overflow Flag (OF)
    ; OF = 1 เมื่อเกิด signed overflow
    ; ============================================
    
    ; Test: 0x7FFFFFFFFFFFFFFF + 1 → OF = 1
    ; (max positive signed 64-bit + 1 = negative!)
    mov rax, 0x7FFFFFFFFFFFFFFF    ; max positive signed 64-bit
    add rax, 1                      ; signed overflow! OF = 1
    
    jo .of_set          ; กระโดดถ้า OF = 1
    mov rsi, msg_noof
    mov rdx, 27
    call print_str
    jmp .exit
.of_set:
    mov rsi, msg_of
    mov rdx, 26
    call print_str
    
.exit:
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 4.3 Direction Flag (DF) - สำคัญสำหรับ String Operations

```nasm
; Direction Flag ควบคุมทิศทาง string operations
; DF = 0: เพิ่ม index (forward, ซ้ายไปขวา)
; DF = 1: ลด index (backward, ขวาไปซ้าย)

section .data
    src  db "Hello, World!", 0
    dst  resb 20

section .text
global _start

_start:
    ; Copy string forward (DF = 0)
    cld                     ; Clear Direction Flag (DF = 0)
    mov rsi, src            ; source
    mov rdi, dst            ; destination
    mov rcx, 13             ; length
    rep movsb               ; copy bytes (เพิ่ม RSI, RDI ทีละ 1)
    
    ; ถ้าใช้ std (Set Direction Flag) แทน cld:
    ; std                   ; DF = 1
    ; lea rsi, [src+12]     ; ชี้ไปที่ท้าย source
    ; lea rdi, [dst+12]     ; ชี้ไปที่ท้าย destination
    ; mov rcx, 13
    ; rep movsb             ; copy backward
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 4.4 การอ่านและเขียน FLAGS

```nasm
; อ่าน FLAGS register
pushf                   ; push EFLAGS/RFLAGS onto stack
pop rax                 ; อ่านค่า

; เขียน FLAGS register
push rax                ; push new flags value
popf                    ; pop into FLAGS register

; LAHF/SAHF: อ่าน/เขียน FLAGS ผ่าน AH
lahf                    ; Load AH = FLAGS[7:0] (SF,ZF,-,AF,-,PF,-,CF)
sahf                    ; Store FLAGS[7:0] = AH
```

---

## 5. Instruction Pointer: IP/EIP/RIP

### 5.1 Instruction Pointer คืออะไร?

```
IP  (16-bit, 8086)  : ชี้ไปที่ instruction ถัดไปที่จะทำงาน
EIP (32-bit, 386)   : ขยายเป็น 32-bit
RIP (64-bit, x86-64): ขยายเป็น 64-bit

สำคัญ: ไม่สามารถ read/write RIP ได้โดยตรงในส่วนใหญ่!
ยกเว้นใช้ LEA instruction ใน 64-bit mode:
    lea rax, [rip]    ; RIP-relative addressing
    lea rax, [rip + symbol]  ; แบบนี้คือ get address ของ symbol
```

### 5.2 RIP-Relative Addressing (64-bit feature!)

```nasm
; ไฟล์: rip_relative.asm
; คอมไพล์: nasm -f elf64 rip_relative.asm -o rip_relative.o && ld rip_relative.o -o rip_relative

section .data
    message db "Hello from RIP-relative!", 10
    msglen  equ $ - message

section .text
global _start

_start:
    ; ใน 64-bit mode, NASM ใช้ RIP-relative addressing อัตโนมัติ
    ; เมื่อเราเขียน [message] ใน 64-bit code
    
    ; แบบ explicit (RIP-relative)
    lea rsi, [rel message]  ; RIP-relative: address ของ message
    ; หรือ
    lea rsi, [message]      ; NASM ทำ RIP-relative ให้อัตโนมัติ
    
    ; ทำไม RIP-relative ถึงสำคัญ?
    ; - Position Independent Code (PIC) - สำหรับ shared libraries
    ; - Code สามารถโหลดที่ address ใดก็ได้ใน memory
    ; - ไม่ต้อง relocation สำหรับ data references
    
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; stdout
    ; rsi ตั้งค่าแล้วข้างบน
    mov rdx, msglen
    syscall
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 6. Operating Modes ของ x86

### 6.1 ภาพรวม Operating Modes

```
x86 CPU Modes:

┌─────────────────────────────────────────────────────────────┐
│                        Long Mode                            │
│                    (64-bit mode)                            │
│  ┌─────────────────┐  ┌─────────────────────────────────┐  │
│  │ 64-bit Sub-mode │  │    Compatibility Sub-mode       │  │
│  │  (64-bit code)  │  │  (32-bit code ใน 64-bit OS)     │  │
│  └─────────────────┘  └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Protected Mode (32-bit)                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
│  │  Normal PM    │  │  Virtual 8086 │  │    System     │   │
│  │               │  │    Mode       │  │  Management   │   │
│  └───────────────┘  └───────────────┘  └───────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Real Mode (16-bit)                       │
│  CPU เริ่มต้นใน mode นี้เมื่อ boot                         │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Real Mode (16-bit)

```
Real Mode Features:
- 20-bit address space (1MB)
- 16-bit registers (AX, BX, CX, DX, ...)
- Memory access: Physical Address = Segment * 16 + Offset
- ไม่มี memory protection
- ไม่มี privilege levels
- ทุก program เข้าถึง hardware ได้โดยตรง
- CPU เริ่ม boot ใน mode นี้เสมอ
- BIOS ทำงานใน Real Mode

Memory Layout ใน Real Mode:
┌──────────────────────────┐ 0xFFFFF (1MB)
│    BIOS ROM              │
├──────────────────────────┤ 0xC0000
│    Video ROM             │
├──────────────────────────┤ 0xA0000
│    Video RAM             │
├──────────────────────────┤ 0x9FFFF
│    Extended BIOS Data    │
├──────────────────────────┤ 0x9FC00
│    Free Memory           │
│    (สำหรับ DOS/Program)  │
├──────────────────────────┤ 0x07C00
│    Boot Sector           │ (512 bytes)
├──────────────────────────┤ 0x00400
│    BIOS Data Area        │
├──────────────────────────┤ 0x00000
│    Interrupt Vector Table│ (1KB)
└──────────────────────────┘
```

### 6.3 Real Mode Code Example (Bootloader)

```nasm
; ไฟล์: bootloader.asm
; คอมไพล์: nasm -f bin bootloader.asm -o bootloader.bin
; ทดสอบ: qemu-system-x86_64 -drive format=raw,file=bootloader.bin
;        (ต้องติดตั้ง qemu)

BITS 16                     ; บอก NASM ว่าเป็น 16-bit code
ORG 0x7C00                  ; Boot sector โหลดที่ address 0x7C00

; Entry point - เริ่มต้น bootloader
boot_start:
    ; ตั้งค่า segment registers
    cli                     ; Disable interrupts ขณะ setup
    xor ax, ax              ; AX = 0
    mov ds, ax              ; DS = 0
    mov es, ax              ; ES = 0
    mov ss, ax              ; SS = 0
    mov sp, 0x7C00          ; Stack pointer ก่อน boot sector
    sti                     ; Enable interrupts กลับ
    
    ; Clear screen ด้วย BIOS interrupt
    mov ah, 0x00            ; BIOS function: Set Video Mode
    mov al, 0x03            ; Video mode: 80x25 text, 16 colors
    int 0x10                ; BIOS Video Interrupt
    
    ; Print message ด้วย BIOS
    mov si, msg_hello       ; SI = address ของ message
    call print_string       ; เรียกฟังก์ชัน print
    
    ; วน loop ค้างไว้ (halt)
.halt_loop:
    hlt                     ; หยุด CPU รอ interrupt
    jmp .halt_loop          ; ถ้ามี interrupt → กลับมา halt อีก

; ฟังก์ชัน print string
; Input: SI = address ของ null-terminated string
print_string:
    push ax
    push bx
.loop:
    lodsb                   ; AL = [DS:SI], SI++
    test al, al             ; ตรวจว่า null terminator หรือเปล่า
    jz .done                ; ถ้า AL = 0 → จบ
    
    mov ah, 0x0E            ; BIOS function: Teletype output
    mov bh, 0               ; Page number
    mov bl, 0x0F            ; Color: white on black
    int 0x10                ; BIOS Video Interrupt
    jmp .loop
.done:
    pop bx
    pop ax
    ret

; Data
msg_hello:
    db "Hello from 16-bit Real Mode Bootloader!", 13, 10
    db "Assembly x86 is AWESOME!", 13, 10, 0

; Boot sector padding
; Boot sector ต้องมีขนาด 512 bytes
; และ 2 bytes สุดท้ายต้องเป็น 0x55AA (boot signature)
times 510 - ($ - $$) db 0  ; pad ด้วย zeros จนถึง byte 510
dw 0xAA55                   ; Boot signature (little-endian: 55 AA)
```

### 6.4 Protected Mode (32-bit)

```
Protected Mode Features:
- 32-bit registers (EAX, EBX, ...)
- 32-bit address space (4GB)
- Memory Protection:
  * Privilege levels (Ring 0-3)
  * Segment-based protection
  * Page-based protection (ถ้าเปิด paging)
- Multitasking support (Task State Segment)
- Virtual 8086 Mode (รัน Real Mode code ใน PM)

Privilege Levels (Rings):
┌──────────────────────────────────────┐
│  Ring 0 (Kernel Mode)                │ ← OS Kernel
│  Ring 1 (Device Drivers - ไม่ค่อยใช้)│ ← Device drivers (บาง OS)
│  Ring 2 (Device Drivers - ไม่ค่อยใช้)│
│  Ring 3 (User Mode)                  │ ← Applications
└──────────────────────────────────────┘
```

### 6.5 Long Mode (64-bit)

```
Long Mode Features:
- 64-bit registers (RAX, RBX, ...)
- 48-bit virtual address space (256TB per process)
  (ปัจจุบัน: 48-bit = 256TB, future: 57-bit = 128PB)
- 8 registers เพิ่ม: R8-R15
- No segments (flat model สำหรับ CS, DS, ES, SS)
- FS, GS ยัง configurable
- REX prefix สำหรับ encode registers ใหม่
- RIP-relative addressing

เข้าสู่ Long Mode ต้องทำตามลำดับ:
1. ต้องอยู่ใน Protected Mode ก่อน
2. ตั้งค่า paging (ต้องใช้)
3. เปิด Long Mode Enable bit ใน EFER MSR
4. โหลด 64-bit GDT
5. Far jump เพื่อ reload CS
```

---

## 7. Control Registers

### 7.1 Overview of Control Registers

```
Control Register  │ หน้าที่
──────────────────┼────────────────────────────────────────────
CR0               │ System flags: PE, PG, WP, etc.
CR1               │ Reserved (ไม่ใช้)
CR2               │ Page Fault Linear Address (PFLA)
CR3               │ Page Directory Base Register (PDBR)
CR4               │ Extended features: PAE, PSE, VME, etc.
CR8               │ Task Priority Register (x86-64 only)
```

### 7.2 CR0 Register

```
CR0 bits ที่สำคัญ:

Bit 31: PG  - Paging Enable
         1 = เปิด paging
         0 = paging ปิด (linear address = physical address)
         
Bit 16: WP  - Write Protect
         1 = Ring 0 ไม่สามารถเขียน read-only pages
         0 = Ring 0 เขียน read-only pages ได้
         
Bit  5: NE  - Numeric Error (FPU errors)
Bit  4: ET  - Extension Type (reserved, always 1)
Bit  3: TS  - Task Switched (FPU context switching)
Bit  2: EM  - Emulation (FPU emulation)
Bit  1: MP  - Monitor co-Processor (FWAIT instruction)
Bit  0: PE  - Protection Enable
         1 = Protected Mode
         0 = Real Mode

การเข้า Protected Mode:
    mov eax, cr0        ; อ่าน CR0
    or eax, 1           ; set bit 0 (PE)
    mov cr0, eax        ; เขียนกลับ → เข้า Protected Mode!
    
การเปิด Paging:
    mov eax, cr0
    or eax, 0x80000000  ; set bit 31 (PG)
    mov cr0, eax        ; เปิด paging
```

### 7.3 CR2 และ CR3

```nasm
; CR2: Page Fault Linear Address
; เมื่อเกิด Page Fault (#PF exception), CPU จะ save address
; ที่ก่อให้เกิด fault ไว้ใน CR2

; Handler อ่าน CR2 เพื่อรู้ว่า address ไหนทำให้ fault:
page_fault_handler:
    push rax
    mov rax, cr2        ; อ่าน faulting address
    ; ... handle the fault using rax as the address ...
    pop rax
    iretq               ; return from interrupt

; CR3: Page Directory Base Register
; ชี้ไปที่ top-level page table (PML4 ใน 64-bit)
; ใช้เปลี่ยน address space (เช่น context switch ระหว่าง processes)

; อ่าน CR3:
mov rax, cr3            ; RAX = physical address ของ PML4 table

; เปลี่ยน CR3 (flush TLB):
mov cr3, rax            ; set new page table, flush TLB อัตโนมัติ
```

### 7.4 CR4 Register

```
CR4 bits ที่สำคัญ:

Bit 18: OSXSAVE - XSAVE/XRESTORE support
Bit 17: PCIDE   - Process-Context Identifiers
Bit 10: OSXMMEXCPT - OS Unmasked SIMD FP Exceptions
Bit  9: OSFXSR  - OS FXSAVE/FXRSTOR Support
Bit  7: PGE     - Page Global Enable (TLB global pages)
Bit  6: MCE     - Machine Check Enable
Bit  5: PAE     - Physical Address Extension
         1 = 36-bit physical addresses (64GB) หรือมากกว่า
         ต้องเปิดก่อนเข้า Long Mode!
Bit  4: PSE     - Page Size Extension (4MB pages)
Bit  3: DE      - Debugging Extensions
Bit  2: TSD     - Time Stamp Disable
Bit  1: PVI     - Protected-mode Virtual Interrupts
Bit  0: VME     - Virtual-8086 Mode Extensions
```

---

## 8. Debug Registers

### 8.1 Debug Registers DR0-DR7

```
Debug Registers ใช้สำหรับ Hardware Breakpoints:
(Hardware breakpoints เร็วกว่า software breakpoints (int 3)
 และสามารถ break on data access ได้)

DR0: Breakpoint 0 Address - address ของ breakpoint ที่ 1
DR1: Breakpoint 1 Address - address ของ breakpoint ที่ 2
DR2: Breakpoint 2 Address - address ของ breakpoint ที่ 3
DR3: Breakpoint 3 Address - address ของ breakpoint ที่ 4

DR4: Reserved/Alias ของ DR6
DR5: Reserved/Alias ของ DR7

DR6: Debug Status Register - บอกว่า breakpoint ไหน triggered
DR7: Debug Control Register - config แต่ละ breakpoint
```

### 8.2 DR7 Control Register

```
DR7 bit layout:

Bits 31-30: Condition for BP3 (00=execute, 01=write, 10=I/O, 11=read/write)
Bits 29-28: Size for BP3      (00=1byte, 01=2byte, 10=8byte, 11=4byte)
Bits 27-26: Condition for BP2
Bits 25-24: Size for BP2
Bits 23-22: Condition for BP1
Bits 21-20: Size for BP1
Bits 19-18: Condition for BP0
Bits 17-16: Size for BP0
Bit  13:    GD  - General Detect Enable (debug register access)
Bit   9:    GE  - Global Exact Breakpoint Enable
Bit   8:    LE  - Local Exact Breakpoint Enable
Bit   7:    G3  - Global Enable BP3
Bit   6:    L3  - Local Enable BP3
Bit   5:    G2  - Global Enable BP2
Bit   4:    L2  - Local Enable BP2
Bit   3:    G1  - Global Enable BP1
Bit   2:    L1  - Local Enable BP1
Bit   1:    G0  - Global Enable BP0
Bit   0:    L0  - Local Enable BP0
```

### 8.3 การใช้ Hardware Breakpoint

```nasm
; ตัวอย่างการตั้ง hardware breakpoint (Ring 0 code)
; หมายเหตุ: ต้องรันใน kernel mode (Ring 0)

set_hw_breakpoint:
    ; ตั้ง breakpoint ที่ address ใน RAX
    ; Breakpoint 0: หยุดเมื่อ execute instruction ที่ address นั้น
    
    mov dr0, rax            ; ตั้ง breakpoint address
    
    mov rbx, dr7            ; อ่าน DR7
    or rbx, 0x3             ; Enable BP0: set L0 (bit 0) และ G0 (bit 1)
    ; Condition = 00 (execute), Size = 00 (1 byte) - ค่า default
    mov dr7, rbx            ; เปิด breakpoint
    ret

; ตั้ง watchpoint (break on data write)
set_write_watchpoint:
    ; RAX = address ที่ต้องการ watch
    mov dr1, rax            ; ตั้งที่ DR1 (breakpoint 1)
    
    mov rbx, dr7
    or rbx, (1 << 2)        ; L1 enable (bit 2)
    or rbx, (1 << 24)       ; Condition for BP1 = 01 (write)
    or rbx, (1 << 28)       ; Size for BP1 = 11 (4 bytes)
    mov dr7, rbx
    ret
```

---

## 9. x86-64 Extensions: REX Prefix

### 9.1 ทำไมต้องมี REX Prefix?

x86 architecture เดิมมี registers จำนวนจำกัดใน encoding instruction ต้องการ 3 bits เพื่อระบุ 8 registers (2³ = 8) แต่ x86-64 เพิ่มมาอีก 8 ตัว (R8-R15) จึงต้องขยาย encoding

```
REX Prefix Format (1 byte):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 0 │ 0 │ W │ R │ X │ B │
└───┴───┴───┴───┴───┴───┴───┴───┘
  7   6   5   4   3   2   1   0

Bits 7-4: 0100 = fixed pattern (identifies it as REX)
Bit  3: W = Width
         1 = 64-bit operand size
         0 = default operand size (32-bit ใน most cases)
Bit  2: R = ModRM.reg extension (bit 3 ของ reg field)
Bit  1: X = SIB.index extension
Bit  0: B = ModRM.rm / SIB.base / Opcode.reg extension

REX values:
0x40 = REX      (prefix แต่ไม่ขยายอะไร)
0x41 = REX.B    (extend B field: registers R8-R15 ใน rm/base)
0x42 = REX.X    (extend X field: R8-R15 ใน SIB index)
0x43 = REX.XB
0x44 = REX.R    (extend R field: R8-R15 ใน reg)
0x45 = REX.RB
0x46 = REX.RX
0x47 = REX.RXB
0x48 = REX.W    (64-bit operand size)
0x49 = REX.WB
0x4A = REX.WX
0x4B = REX.WXBS
0x4C = REX.WR
0x4D = REX.WRB
0x4E = REX.WRX
0x4F = REX.WRXB
```

### 9.2 ตัวอย่าง REX Encoding

```nasm
; Instruction encoding ตัวอย่าง

; mov rax, rbx
; Machine code: 48 89 D8
;   48 = REX.W (W=1, ขนาด 64-bit)
;   89 = MOV opcode
;   D8 = ModRM: mod=11 (register), reg=RBX(011), rm=RAX(000)
;        = 11 011 000 = 0xD8

; mov r8, r9
; Machine code: 4D 89 C8
;   4D = REX.WRB (W=1, R=1 extend reg=R9, B=1 extend rm=R8)
;   89 = MOV opcode
;   C8 = ModRM: mod=11, reg=001 (→ R9 with R=1 → reg#9), rm=000 (→ R8 with B=1 → reg#8)

; ตรวจสอบ encoding ด้วย:
; nasm -f bin -l /dev/stdout encoding_test.asm

section .text
global _start

_start:
    ; ทดลอง R8-R15 registers
    mov r8,  0x1111111111111111   ; REX.W + REX.B ต้องใช้
    mov r9,  0x2222222222222222
    mov r10, 0x3333333333333333
    mov r11, 0x4444444444444444
    mov r12, 0x5555555555555555
    mov r13, 0x6666666666666666
    mov r14, 0x7777777777777777
    mov r15, 0x8888888888888888
    
    ; ใช้ R8-R15 ใน arithmetic
    add r8, r9                    ; r8 += r9
    sub r10, r11                  ; r10 -= r11
    imul r12, r13                 ; r12 *= r13 (signed)
    
    ; ใช้เป็น addressing base
    lea rax, [r8 + r9*2 + 100]   ; complex addressing ด้วย new regs
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 10. CPUID Instruction

### 10.1 CPUID คืออะไร?

CPUID เป็น instruction พิเศษที่ใช้ query ข้อมูลเกี่ยวกับ CPU:
- CPU vendor (Intel, AMD, etc.)
- CPU family, model, stepping
- Features supported (SSE, AVX, AES, etc.)
- Cache sizes
- Core count
- Maximum CPUID leaf

```
การใช้ CPUID:
1. ใส่ leaf number ใน EAX (บางครั้งมี sub-leaf ใน ECX)
2. Execute CPUID
3. ผลลัพธ์อยู่ใน EAX, EBX, ECX, EDX

Leaf 0x00000000: Maximum Standard Leaf + Vendor String
Leaf 0x00000001: Processor Info + Feature Flags
Leaf 0x00000002: Cache/TLB Info
Leaf 0x00000004: Deterministic Cache Parameters
Leaf 0x00000007: Extended Features
Leaf 0x80000000: Maximum Extended Leaf
Leaf 0x80000001: Extended Processor Info + Features
Leaf 0x80000002-4: Processor Brand String
```

### 10.2 ตัวอย่าง CPUID - Vendor String

```nasm
; ไฟล์: cpuid_vendor.asm
; คอมไพล์: nasm -f elf64 cpuid_vendor.asm -o cpuid_vendor.o && ld cpuid_vendor.o -o cpuid_vendor
; รัน: ./cpuid_vendor

section .data
    msg_vendor  db "CPU Vendor: ", 0
    msg_newline db 10, 0
    
section .bss
    vendor_string resb 13   ; 12 chars + null terminator

section .text
global _start

_start:
    ; ===== CPUID Leaf 0: Get Vendor String =====
    xor eax, eax            ; EAX = 0 (leaf 0)
    cpuid
    ; หลังจาก CPUID:
    ; EAX = maximum supported standard CPUID leaf
    ; EBX, EDX, ECX = vendor string (ลำดับ: EBX, EDX, ECX!)
    
    ; บันทึก vendor string
    ; Intel: "GenuineIntel" → EBX="Genu", EDX="ineI", ECX="ntel"
    ; AMD:   "AuthenticAMD" → EBX="Auth", EDX="enti", ECX="cAMD"
    
    mov [vendor_string],    ebx     ; 4 bytes แรก
    mov [vendor_string+4],  edx     ; 4 bytes กลาง
    mov [vendor_string+8],  ecx     ; 4 bytes สุดท้าย
    mov byte [vendor_string+12], 0  ; null terminator
    
    ; Print "CPU Vendor: "
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_vendor
    mov rdx, 12
    syscall
    
    ; Print vendor string
    mov rax, 1
    mov rdi, 1
    mov rsi, vendor_string
    mov rdx, 12
    syscall
    
    ; Print newline
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_newline
    mov rdx, 1
    syscall

    ; ===== CPUID Leaf 1: Processor Info =====
    mov eax, 1              ; Leaf 1
    cpuid
    ; EAX = Processor Version Info:
    ;   Bits  3-0 : Stepping ID
    ;   Bits  7-4 : Model
    ;   Bits 11-8 : Family
    ;   Bits 13-12: Processor Type
    ;   Bits 19-16: Extended Model
    ;   Bits 27-20: Extended Family
    
    ; ECX, EDX = Feature Flags
    ; EDX Feature Flags (commonly used):
    ;   Bit 0:  FPU - x87 FPU on Chip
    ;   Bit 4:  TSC - Time Stamp Counter
    ;   Bit 15: CMOV - Conditional Move
    ;   Bit 23: MMX - MMX Technology
    ;   Bit 25: SSE - SSE Instructions
    ;   Bit 26: SSE2 - SSE2 Instructions
    ; ECX Feature Flags (commonly used):
    ;   Bit 0:  SSE3 - SSE3 Instructions
    ;   Bit 19: SSE4.1
    ;   Bit 20: SSE4.2
    ;   Bit 25: AES - AES Instructions
    ;   Bit 28: AVX - AVX Instructions
    
    ; บันทึกค่า Feature flags
    push ecx                ; บันทึก ECX (ECX features)
    push edx                ; บันทึก EDX (EDX features)
    
    ; ตรวจสอบ SSE2 support (EDX bit 26)
    pop rdx                 ; คืนค่า EDX features
    test edx, (1 << 26)     ; ทดสอบ bit 26
    jz .no_sse2
    
    ; SSE2 supported
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_has_sse2
    mov rdx, msg_has_sse2_len
    syscall
    jmp .check_avx
    
.no_sse2:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_no_sse2
    mov rdx, msg_no_sse2_len
    syscall

.check_avx:
    ; ตรวจสอบ AVX support (ECX bit 28)
    pop rcx                 ; คืนค่า ECX features
    test ecx, (1 << 28)     ; ทดสอบ bit 28
    jz .no_avx
    
    ; AVX supported
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_has_avx
    mov rdx, msg_has_avx_len
    syscall
    jmp .exit
    
.no_avx:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_no_avx
    mov rdx, msg_no_avx_len
    syscall

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

section .data
    msg_has_sse2     db "SSE2: Supported", 10, 0
    msg_has_sse2_len equ 16
    msg_no_sse2      db "SSE2: Not Supported", 10, 0
    msg_no_sse2_len  equ 20
    msg_has_avx      db "AVX:  Supported", 10, 0
    msg_has_avx_len  equ 16
    msg_no_avx       db "AVX:  Not Supported", 10, 0
    msg_no_avx_len   equ 20
```

### 10.3 ตัวอย่าง CPUID - Brand String (ชื่อ CPU)

```nasm
; ไฟล์: cpuid_brand.asm
; คอมไพล์: nasm -f elf64 cpuid_brand.asm -o cpuid_brand.o && ld cpuid_brand.o -o cpuid_brand

section .bss
    brand_string resb 49    ; 48 chars + null

section .data
    msg_brand db "CPU Brand: ", 0

section .text
global _start

_start:
    ; Brand String ใช้ leaves 0x80000002, 0x80000003, 0x80000004
    ; แต่ละ leaf ให้ 16 bytes (EAX+EBX+ECX+EDX = 4*4 bytes)
    ; รวม 48 bytes = ชื่อ CPU เต็มๆ
    
    ; ตรวจสอบก่อนว่า extended leaves รองรับหรือเปล่า
    mov eax, 0x80000000
    cpuid
    cmp eax, 0x80000004     ; ต้องรองรับถึง leaf 0x80000004
    jb .no_brand_string
    
    ; Leaf 0x80000002: brand string characters 0-15
    mov eax, 0x80000002
    cpuid
    mov [brand_string+0],  eax
    mov [brand_string+4],  ebx
    mov [brand_string+8],  ecx
    mov [brand_string+12], edx
    
    ; Leaf 0x80000003: brand string characters 16-31
    mov eax, 0x80000003
    cpuid
    mov [brand_string+16], eax
    mov [brand_string+20], ebx
    mov [brand_string+24], ecx
    mov [brand_string+28], edx
    
    ; Leaf 0x80000004: brand string characters 32-47
    mov eax, 0x80000004
    cpuid
    mov [brand_string+32], eax
    mov [brand_string+36], ebx
    mov [brand_string+40], ecx
    mov [brand_string+44], edx
    
    mov byte [brand_string+48], 0  ; null terminator
    
    ; Print "CPU Brand: "
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_brand
    mov rdx, 11
    syscall
    
    ; หา length ของ brand string
    mov rdi, brand_string
    xor rcx, rcx
.find_len:
    cmp byte [rdi+rcx], 0
    je .found_len
    inc rcx
    jmp .find_len
.found_len:
    
    ; Print brand string
    mov rax, 1
    mov rdi, 1
    mov rsi, brand_string
    mov rdx, rcx
    syscall
    
    ; Newline
    mov rax, 1
    mov rdi, 1
    mov rsi, newline
    mov rdx, 1
    syscall
    
    jmp .exit

.no_brand_string:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg_no_brand
    mov rdx, msg_no_brand_len
    syscall

.exit:
    mov rax, 60
    xor rdi, rdi
    syscall

section .data
    newline          db 10
    msg_no_brand     db "Brand string not supported", 10
    msg_no_brand_len equ $ - msg_no_brand
```

### 10.4 CPUID สมบูรณ์แบบ - ตรวจสอบ Features ทั้งหมด

```nasm
; ไฟล์: cpuid_full.asm
; โปรแกรมตรวจสอบ CPU features อย่างละเอียด
; คอมไพล์: nasm -f elf64 cpuid_full.asm -o cpuid_full.o && ld cpuid_full.o -o cpuid_full

section .data
    ; Feature flag names สำหรับ CPUID.1:EDX
    feat_fpu    db "FPU  ", 0       ; x87 Floating Point Unit
    feat_vme    db "VME  ", 0       ; Virtual Mode Extension
    feat_pse    db "PSE  ", 0       ; Page Size Extension
    feat_tsc    db "TSC  ", 0       ; Time Stamp Counter (RDTSC)
    feat_msr    db "MSR  ", 0       ; Model Specific Registers
    feat_cx8    db "CX8  ", 0       ; CMPXCHG8 instruction
    feat_sep    db "SEP  ", 0       ; SYSENTER/SYSEXIT
    feat_cmov   db "CMOV ", 0       ; Conditional Move
    feat_mmx    db "MMX  ", 0       ; MMX Technology
    feat_sse    db "SSE  ", 0       ; Streaming SIMD Extensions
    feat_sse2   db "SSE2 ", 0       ; SSE2
    feat_ht     db "HT   ", 0       ; Hyper-Threading
    
    ; Feature flags สำหรับ CPUID.1:ECX
    feat_sse3   db "SSE3 ", 0       ; SSE3
    feat_pclmul db "PCLMUL", 0      ; PCLMULQDQ (carry-less multiply)
    feat_ssse3  db "SSSE3", 0       ; Supplemental SSE3
    feat_sse41  db "SSE4.1", 0      ; SSE 4.1
    feat_sse42  db "SSE4.2", 0      ; SSE 4.2
    feat_aes    db "AES  ", 0       ; AES Instructions
    feat_avx    db "AVX  ", 0       ; Advanced Vector Extensions
    feat_f16c   db "F16C ", 0       ; Half-precision convert
    feat_rdrand db "RDRAND", 0      ; Hardware Random Number Generator
    
    sep_line db "========================", 10, 0
    header   db "=== CPU Feature Check ===", 10, 0

section .bss
    vendor_str  resb 13
    brand_str   resb 49
    edx_feat    resd 1      ; บันทึก EDX features
    ecx_feat    resd 1      ; บันทึก ECX features

section .text
global _start

; Macro สำหรับ print string
%macro PRINT 2
    mov rax, 1
    mov rdi, 1
    mov rsi, %1
    mov rdx, %2
    syscall
%endmacro

; ฟังก์ชัน: พิมพ์ feature ถ้า bit set
; Input: RCX = pointer to feature name string
;        RBX = feature register value
;        RSI = bit number to check
print_if_set:
    push rax
    push rdi
    push rdx
    
    ; ตรวจสอบ bit
    mov rax, 1
    shl rax, cl                 ; RAX = 1 << bit_number
    test rbx, rax
    jz .not_set
    
    ; Print feature name
    mov rsi, rcx               ; RSI = feature name
    ; หา length
    xor rdx, rdx
.find_len:
    cmp byte [rsi+rdx], 0
    je .print
    inc rdx
    jmp .find_len
.print:
    mov rax, 1
    mov rdi, 1
    syscall
    
.not_set:
    pop rdx
    pop rdi
    pop rax
    ret

_start:
    ; Header
    PRINT header, 25
    PRINT sep_line, 25
    
    ; ===== Vendor String =====
    xor eax, eax
    cpuid
    mov [vendor_str],    ebx
    mov [vendor_str+4],  edx
    mov [vendor_str+8],  ecx
    mov byte [vendor_str+12], 10    ; newline
    
    mov rax, 1
    mov rdi, 1
    mov rsi, vendor_label
    mov rdx, 8
    syscall
    
    mov rax, 1
    mov rdi, 1
    mov rsi, vendor_str
    mov rdx, 13
    syscall
    
    ; ===== Feature Flags =====
    mov eax, 1
    cpuid
    mov [edx_feat], edx     ; บันทึก EDX features
    mov [ecx_feat], ecx     ; บันทึก ECX features
    
    PRINT feat_header, 17
    
    ; ตรวจ EDX features
    mov ebx, [edx_feat]
    
    ; FPU: bit 0
    test ebx, (1 << 0)
    jz .no_fpu
    PRINT feat_fpu, 5
.no_fpu:
    ; TSC: bit 4
    test ebx, (1 << 4)
    jz .no_tsc
    PRINT feat_tsc, 5
.no_tsc:
    ; MSR: bit 5
    test ebx, (1 << 5)
    jz .no_msr
    PRINT feat_msr, 5
.no_msr:
    ; CMOV: bit 15
    test ebx, (1 << 15)
    jz .no_cmov
    PRINT feat_cmov, 5
.no_cmov:
    ; MMX: bit 23
    test ebx, (1 << 23)
    jz .no_mmx
    PRINT feat_mmx, 5
.no_mmx:
    ; SSE: bit 25
    test ebx, (1 << 25)
    jz .no_sse
    PRINT feat_sse, 5
.no_sse:
    ; SSE2: bit 26
    test ebx, (1 << 26)
    jz .no_sse2
    PRINT feat_sse2, 5
.no_sse2:
    ; HTT: bit 28
    test ebx, (1 << 28)
    jz .no_ht
    PRINT feat_ht, 5
.no_ht:
    
    PRINT newline_str, 1
    
    ; ตรวจ ECX features
    mov ebx, [ecx_feat]
    
    ; SSE3: bit 0
    test ebx, (1 << 0)
    jz .no_sse3
    PRINT feat_sse3, 5
.no_sse3:
    ; SSSE3: bit 9
    test ebx, (1 << 9)
    jz .no_ssse3
    PRINT feat_ssse3, 5
.no_ssse3:
    ; SSE4.1: bit 19
    test ebx, (1 << 19)
    jz .no_sse41
    PRINT feat_sse41, 6
.no_sse41:
    ; SSE4.2: bit 20
    test ebx, (1 << 20)
    jz .no_sse42
    PRINT feat_sse42, 6
.no_sse42:
    ; AES: bit 25
    test ebx, (1 << 25)
    jz .no_aes
    PRINT feat_aes, 5
.no_aes:
    ; AVX: bit 28
    test ebx, (1 << 28)
    jz .no_avx
    PRINT feat_avx, 5
.no_avx:
    ; RDRAND: bit 30
    test ebx, (1 << 30)
    jz .no_rdrand
    PRINT feat_rdrand, 6
.no_rdrand:
    
    PRINT newline_str, 1
    PRINT sep_line, 25
    
    mov rax, 60
    xor rdi, rdi
    syscall

section .data
    vendor_label db "Vendor: ", 0
    feat_header  db "Features: ", 10, 0
    newline_str  db 10
```

---

## 11. Memory Model ใน Real Mode vs Protected Mode

### 11.1 Real Mode Memory Segmentation

```
Real Mode Addressing:
Physical Address = (Segment Register × 16) + Offset

ตัวอย่าง:
CS = 0xF000, IP = 0xFFF0
Physical = 0xF000 × 16 + 0xFFF0
         = 0xF0000 + 0xFFF0
         = 0xFFFF0  ← reset vector!

Segments สามารถ overlap กันได้:
Segment 0x1000 covers: 0x10000 - 0x1FFFF
Segment 0x1001 covers: 0x10010 - 0x2000F
→ byte ที่ 0x10010 อยู่ใน segment ทั้งสองพร้อมกัน!
```

### 11.2 Protected Mode: Descriptor Tables

```
Protected Mode ใช้ Descriptor Tables แทน segment * 16:

Global Descriptor Table (GDT):
- ตาราง descriptor สำหรับ segments ทั้งหมดในระบบ
- แต่ละ entry = 8 bytes (Segment Descriptor)
- Register GDTR ชี้ไปที่ GDT

Local Descriptor Table (LDT):
- ตารางแยกสำหรับแต่ละ process/task
- Register LDTR ชี้ไปที่ LDT

Segment Descriptor (8 bytes):
Bits 63-56: Base[31:24]
Bit  55:    G - Granularity (0=byte, 1=4KB)
Bit  54:    D/B - Default operand size (0=16bit, 1=32bit)
Bit  53:    L - Long mode segment (1=64-bit)
Bit  52:    AVL - Available for OS use
Bits 51-48: Limit[19:16]
Bit  47:    P - Present
Bits 46-45: DPL - Descriptor Privilege Level (0-3)
Bit  44:    S - Descriptor type (0=system, 1=code/data)
Bits 43-40: Type - Segment type
Bits 39-16: Base[23:0]
Bits 15-0:  Limit[15:0]

Segment Selector (ค่าที่ใส่ใน CS, DS, etc.):
Bits 15-3: Index ใน GDT หรือ LDT (8192 entries สูงสุด)
Bit  2:    TI - Table Indicator (0=GDT, 1=LDT)
Bits 1-0:  RPL - Requested Privilege Level
```

### 11.3 Flat Memory Model (64-bit)

```
ใน 64-bit Long Mode, Linux ใช้ flat memory model:
- CS, DS, ES, SS: Base=0, Limit=max
- ทุก virtual address เข้าถึงได้ผ่าน registers ปกติ
- ไม่ต้องคิดเรื่อง segments สำหรับ application code

Virtual Address Space (48-bit):
0x0000000000000000 - 0x00007FFFFFFFFFFF  : User space (128TB)
0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF  : Kernel space (128TB)

(Virtual addresses 0x0000800000000000 - 0xFFFF7FFFFFFFFFFF
 เป็น "canonical hole" - ใช้ไม่ได้)
```

---

## 12. ข้อผิดพลาดที่พบบ่อย (Common Mistakes)

### 12.1 ลืม Zero Extension ของ 32-bit Writes

```nasm
; ผิด: คิดว่า upper 32 bits ยังคงค่าเดิม
mov rax, 0x1234567890ABCDEF
mov eax, 0                  ; RAX = 0x0000000000000000 ไม่ใช่ 0x1234567800000000!

; ถูก: ถ้าต้องการเก็บ upper bits
mov rax, 0x1234567890ABCDEF
and rax, 0xFFFFFFFF00000000 ; เก็บ upper 32 bits
or  rax, 0x00000000         ; set lower 32 bits to 0
; หรือ
mov eax, 0                  ; แล้ว upper bits หายไปอัตโนมัติ - ถ้าต้องการ clear ทั้งหมด นี่คือถูกต้อง
```

### 12.2 ลืม CPUID ก่อนใช้ Features

```nasm
; ผิด: ใช้ SSE/AVX โดยไม่ตรวจสอบก่อน
; movaps xmm0, [data]       ; จะ crash ถ้า CPU ไม่รองรับ!

; ถูก: ตรวจสอบก่อนเสมอ
check_sse_support:
    mov eax, 1
    cpuid
    test edx, (1 << 25)     ; SSE bit
    jz .no_sse
    ; ใช้ SSE ได้
    movaps xmm0, [data]
    jmp .done
.no_sse:
    ; fallback code
.done:
    ret
```

### 12.3 การอ่าน FLAGS ที่ไม่ถูกต้อง

```nasm
; ผิด: FLAGS อาจเปลี่ยนระหว่าง instruction
cmp rax, rbx    ; set ZF
push rax        ; PUSH ไม่เปลี่ยน ZF... แต่...
call some_func  ; CALL บันทึก RIP ลง stack, function อาจเปลี่ยน FLAGS!
jz somewhere    ; ZF อาจไม่ใช่ผลจาก CMP แล้ว!

; ถูก: ใช้ conditional jump ทันทีหลัง comparison
cmp rax, rbx
jz somewhere    ; jump ทันทีหลัง CMP
; หรือบันทึก condition ก่อน
cmp rax, rbx
sete al         ; บันทึก ZF ลงใน AL (0 หรือ 1)
call some_func
test al, al     ; ตรวจสอบ condition ที่บันทึกไว้
jnz somewhere
```

### 12.4 การใช้ Segment Registers ผิดใน 64-bit

```nasm
; ผิด: พยายาม set DS ใน 64-bit mode
mov ax, 0x10
mov ds, ax      ; ใน 64-bit, DS ไม่ส่งผลต่อ addressing!
mov rax, [rbx]  ; จะ work เหมือน DS=0 เสมอ

; ถูก: ใน 64-bit mode ไม่ต้องใช้ segment prefixes
; เว้นแต่จะใช้ FS/GS สำหรับ thread-local storage
mov rax, [rbx]  ; OK ไม่ต้องมี segment prefix
mov rax, [fs:0] ; ใช้ FS สำหรับ TLS (OS-specific)
```

### 12.5 ใช้ Control Registers จาก User Mode

```nasm
; ผิด: User mode code พยายามอ่าน Control Registers
mov rax, cr0    ; General Protection Fault (#GP)! ต้องเป็น Ring 0

; ถูก: CR0-CR4 เข้าถึงได้เฉพาะ Ring 0 (kernel mode)
; ถ้าอยาก query CPU info → ใช้ CPUID แทน
; CPUID ทำงานได้ทุก privilege level
mov eax, 1
cpuid           ; ได้ feature flags โดยไม่ต้องเข้า kernel
```

---

## 13. แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Register Arithmetic

เขียนโปรแกรมที่:
1. ใส่ค่า 0x123456789ABCDEF0 ใน RAX
2. อ่านค่า EAX, AX, AH, AL แยกกัน
3. พิมพ์ขนาดของแต่ละส่วน

**โค้ดฝึกหัด:**
```nasm
; ไฟล์: exercise1.asm
; TODO: เติมโค้ดในส่วนที่ขาด

section .data
    msg_rax db "RAX = 0x123456789ABCDEF0", 10, 0
    msg_eax db "EAX should be 0x9ABCDEF0 (lower 32 bits)", 10, 0
    msg_ax  db "AX  should be 0xDEF0 (lower 16 bits)", 10, 0
    msg_al  db "AL  should be 0xF0 (lowest 8 bits)", 10, 0
    msg_ah  db "AH  should be 0xDE (bits 8-15)", 10, 0

section .text
global _start

_start:
    ; TODO: ใส่ค่า 0x123456789ABCDEF0 ใน RAX
    ; TODO: ทดลองอ่านค่าจาก EAX, AX, AH, AL
    ; TODO: ตรวจสอบว่าค่าตรงกับที่คาดหวังหรือเปล่า
    
    ; Hint: ใช้ movzx เพื่อ zero-extend เมื่อ copy ไปยัง register ใหญ่กว่า
    ; movzx eax, al    ; zero-extend AL → EAX
    ; movzx eax, ax    ; zero-extend AX → EAX
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### แบบฝึกหัดที่ 2: FLAGS Analysis

เขียนโปรแกรมที่วิเคราะห์ FLAGS หลังจาก arithmetic operations:

```nasm
; ไฟล์: exercise2.asm
; วิเคราะห์ FLAGS หลัง operations ต่างๆ

section .text
global _start

_start:
    ; Test 1: 0 - 0 → ZF = ?, CF = ?, SF = ?, OF = ?
    xor rax, rax
    sub rax, 0
    ; TODO: บันทึก FLAGS และวิเคราะห์
    
    ; Test 2: 0 - 1 (unsigned: borrow, signed: negative)
    xor rax, rax
    sub rax, 1
    ; TODO: บันทึก FLAGS - CF = 1 (borrow), SF = 1 (negative), ZF = 0
    
    ; Test 3: 0x7FFFFFFF + 1 (signed overflow ใน 32-bit)
    mov eax, 0x7FFFFFFF
    add eax, 1
    ; TODO: OF = 1, SF = 1, ZF = 0
    
    ; Test 4: 0xFFFFFFFF + 1 (unsigned overflow ใน 32-bit)
    mov eax, 0xFFFFFFFF
    add eax, 1
    ; TODO: CF = 1, ZF = 1, OF = 0
    
    ; Hint: ใช้ pushf / pop rax เพื่ออ่าน FLAGS
    ; แล้วตรวจสอบแต่ละ bit ด้วย TEST
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### แบบฝึกหัดที่ 3: CPUID Feature Detection

เขียน function ที่รับ feature flag เป็น parameter และ return 1 ถ้า CPU รองรับ, 0 ถ้าไม่รองรับ:

```nasm
; ไฟล์: exercise3.asm
; สร้าง function: check_feature(leaf, register, bit) → 0 or 1

; Function prototype:
; Input:  RDI = CPUID leaf number (เช่น 1)
;         RSI = which register (0=EAX, 1=EBX, 2=ECX, 3=EDX)
;         RDX = bit number to check (0-31)
; Output: RAX = 1 ถ้า feature supported, 0 ถ้าไม่

check_feature:
    ; TODO: implement this function
    ; Hint:
    ;   1. ใส่ leaf ใน EAX แล้ว CPUID
    ;   2. เลือก register ตาม RSI
    ;   3. ตรวจสอบ bit ตาม RDX
    ;   4. return 1 หรือ 0 ใน RAX
    ret

section .text
global _start

_start:
    ; Test: ตรวจสอบ SSE2 (leaf=1, register=EDX=3, bit=26)
    mov rdi, 1
    mov rsi, 3      ; EDX
    mov rdx, 26     ; SSE2 bit
    call check_feature
    ; RAX = 1 ถ้า SSE2 supported
    
    mov rax, 60
    mov rdi, 0
    syscall
```

### แบบฝึกหัดที่ 4: Debug Register Simulation

เขียนโปรแกรมที่จำลองการทำงานของ hardware breakpoint ด้วย software:

```nasm
; ไฟล์: exercise4.asm
; จำลอง watchpoint: แจ้งเตือนเมื่อค่าใน memory address เปลี่ยน

section .data
    watched_var dq 0x42    ; variable ที่ต้องการ watch

section .text
global _start

; ฟังก์ชัน watch_variable:
; เก็บค่าปัจจุบัน และ compare กับค่าก่อนหน้า
; ถ้าเปลี่ยน → print message

_start:
    ; Read current value
    mov rax, [watched_var]
    push rax               ; บันทึก original value
    
    ; "Change" the variable (จำลอง)
    mov qword [watched_var], 0x99
    
    ; Check if changed
    pop rbx                ; original value
    mov rax, [watched_var] ; current value
    cmp rax, rbx
    je .no_change
    
    ; Variable changed! (ใส่ message แจ้งเตือนที่นี่)
.no_change:
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### แบบฝึกหัดที่ 5: CPUID Leaf Explorer

เขียนโปรแกรมที่แสดง CPUID results สำหรับ leaves หลายๆ ตัว:

```nasm
; ไฟล์: exercise5.asm
; แสดงผล CPUID สำหรับ leaves 0-7 และ 0x80000000-0x80000004

section .text
global _start

; Helper: print hex value ใน RAX (64-bit)
print_hex:
    ; TODO: implement
    ret

; Helper: CPUID and print all 4 registers
cpuid_print_leaf:
    ; Input: EAX = leaf number
    ; TODO:
    ;   1. CPUID
    ;   2. Print "EAX: " + hex(eax)
    ;   3. Print "EBX: " + hex(ebx)
    ;   4. Print "ECX: " + hex(ecx)
    ;   5. Print "EDX: " + hex(edx)
    ret

_start:
    ; Loop through leaves 0-7
    xor r12d, r12d  ; counter = 0
.loop:
    cmp r12d, 7
    jg .done
    
    mov eax, r12d
    call cpuid_print_leaf
    
    inc r12d
    jmp .loop
    
.done:
    ; Extended leaves
    mov eax, 0x80000000
    call cpuid_print_leaf
    ; (เพิ่มเติม: print extended leaves)
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 14. สรุปและ Key Takeaways

### สิ่งที่เรียนรู้ใน Part 006:

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| x86 History | วิวัฒนาการจาก 8086 (16-bit) → 386 (32-bit) → x86-64 (64-bit) |
| GPRs | RAX-RSP (legacy) + R8-R15 (new), AX/EAX/RAX hierarchy |
| Zero Extension | 32-bit write → zero-extends upper 32 bits ของ 64-bit register |
| Segment Regs | Real Mode: segmentation; 64-bit: flat model (FS/GS สำหรับ TLS) |
| FLAGS | CF, ZF, SF, OF, DF คือ flags ที่ใช้บ่อยที่สุด |
| RIP | Instruction pointer, RIP-relative addressing ใน 64-bit |
| Control Regs | CR0 (PE, PG), CR2 (page fault addr), CR3 (page table), CR4 |
| Debug Regs | DR0-DR3 (breakpoint addresses), DR7 (control) |
| Operating Modes | Real Mode → Protected Mode → Long Mode |
| REX Prefix | เพิ่ม register encoding สำหรับ R8-R15 และ 64-bit operands |
| CPUID | Query CPU capabilities: vendor, features, cache info |

### กฎสำคัญที่ต้องจำ:

```
1. เมื่อ write ไปที่ 32-bit register ใน 64-bit mode
   → upper 32 bits ถูก zero-extend อัตโนมัติ!

2. FLAGS สามารถเปลี่ยนได้โดย instruction หลายๆ ตัว
   → ใช้ conditional jump ทันทีหลัง comparison

3. ใน 64-bit mode, CS/DS/ES/SS ไม่ส่งผลต่อ addressing
   → เฉพาะ FS/GS ที่ยังใช้งานได้

4. ก่อนใช้ advanced features (SSE, AVX, etc.)
   → ตรวจสอบด้วย CPUID เสมอ

5. Control Registers (CR0-CR4) เข้าถึงได้เฉพาะ Ring 0
   → ใช้ CPUID สำหรับ feature detection จาก user mode
```

---

## 15. แหล่งอ้างอิง (Resources)

### Official Documentation
- **Intel Software Developer's Manual (SDM)**: https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
  - Volume 1: Basic Architecture (Registers, Memory Model)
  - Volume 2: Instruction Set Reference (CPUID, RDTSC, etc.)
  - Volume 3: System Programming Guide (Control Registers, Paging)

- **AMD Architecture Programmer's Manual**: https://developer.amd.com/resources/developer-guides-manuals/
  - Volume 2: System Programming
  - Volume 3: General Purpose and System Instructions

### หนังสือแนะนำ
- "Introduction to 64 Bit Intel Assembly Language Programming" by Ray Seyfarth
- "Programming from the Ground Up" by Jonathan Bartlett (Free: https://savannah.nongnu.org/projects/pgubook/)
- "Modern X86 Assembly Language Programming" by Daniel Kusswurm

### Tools ที่ใช้ใน Part นี้
```bash
# NASM assembler
sudo apt install nasm

# GNU Debugger (GDB) สำหรับดู registers
sudo apt install gdb

# QEMU สำหรับทดสอบ bootloader
sudo apt install qemu-system-x86

# cpuid utility (Linux)
sudo apt install cpuid
cpuid          # แสดง CPUID information ทั้งหมด
cpuid -1       # Leaf 1 only
```

### คำสั่ง Useful สำหรับตรวจสอบ CPU
```bash
# แสดงข้อมูล CPU บน Linux
cat /proc/cpuinfo

# แสดง CPU flags (features)
grep -m1 flags /proc/cpuinfo

# ตรวจสอบ CPU model
lscpu

# ใช้ CPUID utility
cpuid -r        # raw output

# GDB: ดู registers ขณะ debug
# (gdb) info registers
# (gdb) info registers rflags
# (gdb) x/32xb $rsp      ดู stack

# Disassemble: ดู machine code
# objdump -d -M intel program   # แสดง disassembly ในรูปแบบ Intel syntax
# ndisasm -b64 program.bin      # NASM disassembler
```

---

## 16. Preview: Part 007

ใน Part ถัดไปเราจะเรียน:
- **x86 Instruction Encoding เชิงลึก**: ModRM, SIB, Displacement
- **Addressing Modes ทั้งหมด**: Register, Immediate, Memory แบบต่างๆ
- **Instruction Prefixes**: REX, VEX, EVEX, LOCK, REP
- **ตัวอย่าง**: decode machine code ด้วยมือ และ encode instruction เอง
- **Practical**: เขียน disassembler อย่างง่าย

---

*Part 006 - จบ*  
*เวลาเรียนรวม: ~5-7 ชั่วโมง*  
*ต่อด้วย: Part 007 - x86 Instruction Encoding*
