# Part 002: สถาปัตยกรรมคอมพิวเตอร์พื้นฐาน
## (Computer Architecture Fundamentals)

**ระดับ:** พื้นฐาน (Beginner)  
**เวลาที่ใช้:** 4-6 ชั่วโมง  
**ก่อนหน้า:** [Part 001 - Introduction to Assembly Language](part-001-introduction.md)  
**ถัดไป:** [Part 003 - x86 Registers and Data Types](part-003-registers-data-types.md)

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:

- อธิบาย Von Neumann Architecture และส่วนประกอบหลักได้
- เข้าใจการทำงานของ CPU (ALU, Control Unit, Registers, Cache)
- อธิบาย Memory Hierarchy และผลกระทบต่อ performance
- เข้าใจ Fetch-Decode-Execute cycle อย่างละเอียด
- เปรียบเทียบ CISC vs RISC architecture ได้
- เข้าใจ Endianness และผลกระทบในการเขียน Assembly
- เข้าใจพื้นฐาน Pipeline และ Clock Cycles
- เขียน code ที่แสดงให้เห็นถึงแนวคิดเหล่านี้ได้

---

## 1. Von Neumann Architecture

### 1.1 ประวัติและแนวคิด

Von Neumann Architecture คือสถาปัตยกรรมคอมพิวเตอร์ที่ถูกเสนอโดย John von Neumann ในปี 1945 ซึ่งเป็นพื้นฐานของคอมพิวเตอร์สมัยใหม่เกือบทั้งหมด

**แนวคิดหลัก (Stored-Program Concept):**
โปรแกรมและข้อมูลถูกเก็บไว้ในหน่วยความจำเดียวกัน (Shared Memory) ซึ่งทำให้คอมพิวเตอร์สามารถแก้ไขโปรแกรมของตัวเองได้

```
┌─────────────────────────────────────────────────────────────┐
│                Von Neumann Architecture                     │
│                                                             │
│  ┌──────────────┐         ┌──────────────────────────────┐  │
│  │     CPU      │         │         Memory               │  │
│  │  ┌────────┐  │◄───────►│  ┌──────────────────────┐   │  │
│  │  │  ALU   │  │         │  │  Instructions (Code) │   │  │
│  │  └────────┘  │  Bus    │  ├──────────────────────┤   │  │
│  │  ┌────────┐  │         │  │       Data           │   │  │
│  │  │  CU    │  │         │  ├──────────────────────┤   │  │
│  │  └────────┘  │         │  │       Stack          │   │  │
│  │  ┌────────┐  │         │  ├──────────────────────┤   │  │
│  │  │  Regs  │  │         │  │       Heap           │   │  │
│  │  └────────┘  │         │  └──────────────────────┘   │  │
│  └──────────────┘         └──────────────────────────────┘  │
│          ▲                                                   │
│          │                                                   │
│  ┌───────┴──────┐                                           │
│  │  I/O Devices │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 ส่วนประกอบหลักของ Von Neumann Architecture

| ส่วนประกอบ | หน้าที่ |
|-----------|---------|
| CPU (Central Processing Unit) | ประมวลผลคำสั่ง |
| Memory Unit | เก็บโปรแกรมและข้อมูล |
| Input Unit | รับข้อมูลจากภายนอก |
| Output Unit | ส่งข้อมูลออกไปภายนอก |
| Bus System | เชื่อมต่อส่วนประกอบต่างๆ |

### 1.3 ข้อดีและข้อเสียของ Von Neumann

**ข้อดี:**
- ง่ายในการออกแบบและผลิต
- ใช้หน่วยความจำร่วมกันได้อย่างยืดหยุ่น
- โปรแกรมสามารถแก้ไขตัวเองได้ (Self-modifying code)

**ข้อเสีย (Von Neumann Bottleneck):**
- CPU ต้องรอ memory access ทำให้ช้าลง
- Instruction และ Data ใช้ bus เดียวกัน เกิดการแย่ง bus

---

## 2. Harvard Architecture

Harvard Architecture แยก Memory สำหรับ Instruction และ Data ออกจากกัน ทำให้สามารถ fetch instruction และ read data พร้อมกันได้

```
┌─────────────────────────────────────────────────────────────┐
│                Harvard Architecture                         │
│                                                             │
│  ┌──────────────┐  Instruction Bus  ┌────────────────────┐  │
│  │              │◄─────────────────►│  Instruction Memory│  │
│  │     CPU      │                   └────────────────────┘  │
│  │              │                                           │
│  │  ┌─────────┐ │    Data Bus       ┌────────────────────┐  │
│  │  │  ALU    │ │◄─────────────────►│    Data Memory     │  │
│  │  │  CU     │ │                   └────────────────────┘  │
│  │  │  Regs   │ │                                           │
│  │  └─────────┘ │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

**ตัวอย่างการใช้งาน Harvard Architecture:**
- Microcontrollers (AVR, PIC)
- Digital Signal Processors (DSP)
- ARM Cortex-M series (Modified Harvard)

**Modified Harvard Architecture:**
Modern CPU ส่วนใหญ่ใช้ Modified Harvard ที่มี separate cache สำหรับ instruction และ data แต่ shared main memory

---

## 3. ส่วนประกอบของ CPU

### 3.1 ALU (Arithmetic Logic Unit)

ALU คือหัวใจของ CPU ทำหน้าที่คำนวณและเปรียบเทียบข้อมูล

**การทำงานที่ ALU รองรับ:**

```
Arithmetic Operations:    Logic Operations:
  ADD  - บวก               AND  - ตรรกะ AND
  SUB  - ลบ                OR   - ตรรกะ OR
  MUL  - คูณ               XOR  - ตรรกะ XOR
  DIV  - หาร               NOT  - ตรรกะ NOT
  MOD  - หารเอาเศษ         SHL  - Shift Left
  NEG  - เปลี่ยนเครื่องหมาย SHR  - Shift Right

Comparison Operations:
  CMP  - เปรียบเทียบ (ผลลัพธ์ไปที่ Flags Register)
  TEST - ทดสอบบิต
```

**ตัวอย่าง code แสดงการทำงาน ALU:**

```nasm
; ไฟล์: alu_demo.asm
; แสดงการทำงานของ ALU operations ต่างๆ
; คอมไพล์: nasm -f elf64 alu_demo.asm -o alu_demo.o
;           ld alu_demo.o -o alu_demo

section .data
    ; ข้อความแสดงผล
    msg_add     db "ADD result: ", 0
    msg_sub     db "SUB result: ", 0
    msg_and     db "AND result: ", 0
    msg_or      db "OR result:  ", 0
    msg_xor     db "XOR result: ", 0
    newline     db 10                   ; newline character

section .bss
    result      resb 8                  ; จองพื้นที่ 8 bytes สำหรับเก็บผล

section .text
    global _start

_start:
    ; === การทำงานของ ALU: ADD ===
    mov rax, 100        ; ใส่ค่า 100 ลง rax
    mov rbx, 50         ; ใส่ค่า 50 ลง rbx
    add rax, rbx        ; rax = rax + rbx = 150  (ALU ทำงาน)
    ; ผล: rax = 150
    
    ; === การทำงานของ ALU: SUB ===
    mov rax, 200        ; ใส่ค่า 200
    mov rbx, 75         ; ใส่ค่า 75
    sub rax, rbx        ; rax = rax - rbx = 125  (ALU ทำงาน)
    ; ผล: rax = 125
    
    ; === การทำงานของ ALU: AND ===
    ; AND ใช้ทำ bit masking เพื่อเลือกบิตที่ต้องการ
    mov rax, 0xFF       ; 11111111 ในไบนารี
    mov rbx, 0x0F       ; 00001111 ในไบนารี
    and rax, rbx        ; rax = 0xFF AND 0x0F = 0x0F = 00001111
    ; ผล: rax = 0x0F = 15
    
    ; === การทำงานของ ALU: OR ===
    ; OR ใช้ตั้งบิต (Set bits)
    mov rax, 0x00       ; 00000000
    mov rbx, 0x05       ; 00000101
    or  rax, rbx        ; rax = 0x00 OR 0x05 = 0x05
    ; ผล: rax = 0x05 = 5
    
    ; === การทำงานของ ALU: XOR ===
    ; XOR ใช้ toggle บิต หรือ clear register (xor reg, reg)
    mov rax, 0xFF       ; 11111111
    mov rbx, 0x0F       ; 00001111
    xor rax, rbx        ; rax = 11111111 XOR 00001111 = 11110000
    ; ผล: rax = 0xF0 = 240
    
    ; เทคนิค: xor register กับตัวเอง = clear register (เร็วกว่า mov reg, 0)
    xor rax, rax        ; rax = 0  ← วิธีที่ programmers มือโปรใช้
    
    ; === Shift Operations ===
    mov rax, 1          ; rax = 1 = 00000001
    shl rax, 3          ; shift left 3 bits = 00001000 = 8
    ; ผล: rax = 8  (คูณด้วย 2^3 = 8)
    
    mov rax, 16         ; rax = 16 = 00010000
    shr rax, 2          ; shift right 2 bits = 00000100 = 4
    ; ผล: rax = 4  (หารด้วย 2^2 = 4)
    
    ; === EXIT ===
    mov rax, 60         ; system call: exit
    xor rdi, rdi        ; exit code = 0
    syscall             ; เรียก kernel

; หมายเหตุ: ผลลัพธ์ไม่แสดงบนหน้าจอ (ต้องใช้ printf หรือ write syscall)
; Part นี้แสดงการทำงานภายใน ALU เป็นหลัก
```

### 3.2 Control Unit (CU)

Control Unit ควบคุมการทำงานของ CPU โดยการ decode คำสั่งและส่งสัญญาณควบคุมไปยังส่วนต่างๆ

**หน้าที่ของ Control Unit:**
1. **Instruction Fetch** - ดึง instruction จาก memory มาที่ IR (Instruction Register)
2. **Instruction Decode** - แปล opcode เป็น control signals
3. **Execution Control** - ควบคุมการทำงานของ ALU และ registers
4. **Memory Access Control** - ควบคุมการอ่าน/เขียน memory
5. **Pipeline Control** - จัดการ pipeline stages

### 3.3 Registers

Registers คือหน่วยความจำที่เร็วที่สุดใน computer system อยู่ภายใน CPU โดยตรง

**ประเภทของ Registers:**

```
┌─────────────────────────────────────────────────────────────┐
│                x86-64 Register Overview                     │
├──────────────────┬──────────────────────────────────────────┤
│ General Purpose  │ RAX, RBX, RCX, RDX, RSI, RDI, RSP, RBP │
│                  │ R8-R15                                   │
├──────────────────┼──────────────────────────────────────────┤
│ Segment          │ CS, DS, ES, FS, GS, SS                  │
├──────────────────┼──────────────────────────────────────────┤
│ Instruction Ptr  │ RIP (Next instruction address)          │
├──────────────────┼──────────────────────────────────────────┤
│ Flags            │ RFLAGS (Status bits)                     │
├──────────────────┼──────────────────────────────────────────┤
│ SIMD/Float       │ XMM0-XMM15, YMM0-YMM15, ZMM0-ZMM31     │
└──────────────────┴──────────────────────────────────────────┘
```

**ขนาดของ registers ใน x86-64:**

```
64-bit: RAX  [────────────────────────────────]
32-bit: EAX  [────────────────]
16-bit: AX   [────────]
 8-bit: AH   [────]            (High byte ของ AX)
 8-bit: AL        [────]       (Low byte ของ AX)

ตัวอย่าง:
RAX = 0x0123456789ABCDEF
EAX =         0x89ABCDEF   (32 bits ล่าง)
AX  =             0xCDEF   (16 bits ล่าง)
AH  =               0xCD   (High byte ของ AX)
AL  =                 0xEF  (Low byte ของ AX)
```

**ตัวอย่าง code แสดง Register Access:**

```nasm
; ไฟล์: register_demo.asm
; แสดงการเข้าถึง register ในขนาดต่างๆ
; คอมไพล์: nasm -f elf64 register_demo.asm -o register_demo.o
;           ld register_demo.o -o register_demo

section .text
    global _start

_start:
    ; ใส่ค่าใน RAX (64-bit)
    mov rax, 0x0123456789ABCDEF
    
    ; เข้าถึงส่วนต่างๆ ของ RAX
    ; EAX อ่านเฉพาะ 32 bits ล่าง
    ; แต่เมื่อเขียน EAX จะ zero-extend ไปยัง RAX ด้วย!
    
    ; ทดสอบการเขียนผ่าน EAX
    mov eax, 0xDEADBEEF         ; RAX = 0x00000000DEADBEEF (zero-extended!)
    
    ; ทดสอบ AX (16-bit)
    mov ax, 0x1234              ; RAX = 0x00000000DEAD1234
    
    ; ทดสอบ AH (high byte)
    mov ah, 0xFF                ; RAX = 0x00000000DEADFF34
    
    ; ทดสอบ AL (low byte)
    mov al, 0x00                ; RAX = 0x00000000DEADFF00
    
    ; === ตัวอย่างการใช้งานจริง ===
    
    ; วิธีที่ 1: ดึง byte สูงสุดของ 32-bit value
    mov eax, 0x12345678
    shr eax, 24                 ; shift right 24 bits
    ; eax = 0x12 = 18
    
    ; วิธีที่ 2: ดึง byte ต่ำสุด
    mov eax, 0x12345678
    and al, 0xFF                ; al = 0x78 (mask ด้วย 0xFF)
    ; หรือแค่อ่าน AL โดยตรง
    ; al = 0x78 = 120
    
    ; วิธีที่ 3: ดึง bits 8-15
    mov eax, 0x12345678
    shr eax, 8                  ; shift right 8
    and al, 0xFF                ; mask
    ; al = 0x56
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 3.4 Cache Memory

Cache เป็นหน่วยความจำความเร็วสูงที่อยู่ใกล้ CPU มากกว่า RAM

```
Cache Hierarchy:
┌───────────────────────────────────────────────────────────┐
│                    CPU Core                               │
│  ┌─────────────┐                                         │
│  │  Registers  │  ← เร็วที่สุด, ~0.5ns, 64 bytes       │
│  └──────┬──────┘                                         │
│         │                                                 │
│  ┌──────┴──────┐                                         │
│  │  L1 Cache   │  ← ~1ns, 32-64 KB (แยก I-Cache/D-Cache)│
│  └──────┬──────┘                                         │
│         │                                                 │
│  ┌──────┴──────┐                                         │
│  │  L2 Cache   │  ← ~3-10ns, 256KB - 1MB                │
│  └──────┬──────┘                                         │
└─────────┼─────────────────────────────────────────────────┘
          │
  ┌───────┴──────┐
  │  L3 Cache   │  ← ~30-40ns, 8-64MB (shared between cores)
  └───────┬──────┘
          │
  ┌───────┴──────┐
  │   Main RAM   │  ← ~100ns, GBs
  └───────┬──────┘
          │
  ┌───────┴──────┐
  │  SSD/HDD     │  ← ~100μs - 10ms, TBs
  └──────────────┘
```

**Cache Hit vs Cache Miss:**

```
Cache Hit:  CPU หาข้อมูลเจอใน cache → เร็ว
Cache Miss: CPU ไม่เจอใน cache → ต้องไปดึงจาก level ที่ใหญ่กว่า → ช้า

Cache Hit Rate ที่ดี = 95-99%
```

---

## 4. Memory Hierarchy

### 4.1 ตาราง Memory Hierarchy

| ระดับ | ชื่อ | ความเร็ว | ขนาด | ราคา/byte |
|------|------|---------|------|-----------|
| 0 | Registers | 0.5ns | ~1KB | ที่แพงที่สุด |
| 1 | L1 Cache | 1-2ns | 32-64KB | แพงมาก |
| 2 | L2 Cache | 3-10ns | 256KB-1MB | แพง |
| 3 | L3 Cache | 10-40ns | 8-64MB | ปานกลาง |
| 4 | Main RAM | 60-100ns | 4-64GB | ถูก |
| 5 | SSD | 25-100μs | 256GB-4TB | ถูกมาก |
| 6 | HDD | 5-10ms | 1-20TB | ถูกมากที่สุด |

### 4.2 Spatial and Temporal Locality

```
Temporal Locality: 
    ถ้าใช้ข้อมูลนี้แล้ว มีโอกาสสูงที่จะใช้อีก
    ตัวอย่าง: Loop counter ถูกอ่าน/เขียนซ้ำๆ

Spatial Locality: 
    ถ้าใช้ address นี้ มีโอกาสสูงที่จะใช้ address ใกล้เคียง
    ตัวอย่าง: Array traversal ← ใช้ index เพิ่มขึ้นทีละ 1
```

**ตัวอย่าง code ที่แสดง Cache-Friendly vs Cache-Unfriendly:**

```nasm
; ไฟล์: cache_demo.asm
; แสดงความแตกต่างของ cache-friendly vs cache-unfriendly access
; คอมไพล์: nasm -f elf64 cache_demo.asm -o cache_demo.o
;           ld cache_demo.o -o cache_demo

section .data
    ; Array ขนาด 64 bytes = 8 × 64-bit integers
    array   dq 1, 2, 3, 4, 5, 6, 7, 8

section .text
    global _start

_start:
    ; === Cache-Friendly: Sequential Access ===
    ; เข้าถึง array ตามลำดับ (0, 1, 2, 3, ...)
    ; ข้อมูลอยู่ใน same cache line → Cache Hit สูง
    
    xor rcx, rcx            ; rcx = 0 (loop counter)
    xor rax, rax            ; rax = 0 (sum)
    lea rsi, [array]        ; rsi = address ของ array
    
.sequential_loop:
    cmp rcx, 8              ; ครบ 8 element แล้วหรือยัง?
    jge .sequential_done    ; ถ้าครบ ออกจาก loop
    
    mov rbx, [rsi + rcx*8]  ; โหลด array[rcx]  ← Sequential access
    add rax, rbx             ; sum += array[rcx]
    
    inc rcx                  ; rcx++
    jmp .sequential_loop
    
.sequential_done:
    ; rax = sum of all elements = 36
    
    ; === Cache-Unfriendly: Stride Access (ข้ามทีละมากๆ) ===
    ; ใน 2D array การ traverse ตาม column (row-major language)
    ; จะทำให้ cache miss สูง
    ; ตัวอย่างนี้แสดงแนวคิด ไม่ได้ run จริง
    
    ; GOOD: Row-major traversal
    ; for(i=0; i<N; i++)
    ;   for(j=0; j<M; j++)
    ;     process(matrix[i][j])  ← Cache-friendly
    
    ; BAD: Column-major traversal
    ; for(j=0; j<M; j++)
    ;   for(i=0; i<N; i++)
    ;     process(matrix[i][j])  ← Cache-unfriendly (stride = M)
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 5. Bus Architecture

### 5.1 ประเภทของ Bus

```
┌─────────────────────────────────────────────────────────────┐
│                     Bus Architecture                        │
│                                                             │
│   CPU                Memory            I/O Devices         │
│   ┌────┐             ┌────┐            ┌────┐              │
│   │    │             │    │            │    │              │
│   └────┘             └────┘            └────┘              │
│      │                  │                 │                 │
│ ─────┴──────────────────┴─────────────────┴──── Address Bus │
│      │     (ส่ง address ว่าจะ read/write ที่ไหน)            │
│ ─────┴──────────────────┴─────────────────┴──── Data Bus    │
│      │     (ส่งข้อมูลจริงๆ)                                │
│ ─────┴──────────────────┴─────────────────┴──── Control Bus │
│           (Read/Write signals, Clock, Interrupt, etc.)     │
└─────────────────────────────────────────────────────────────┘
```

**Address Bus:**
- กำหนดจำนวน memory locations ที่ access ได้
- 32-bit Address Bus → 2^32 = 4GB addresses
- 64-bit Address Bus → 2^64 = 16 exabytes addresses (ในทางทฤษฎี)
- x86-64 ในปัจจุบัน ใช้จริง 48 bits → 256 TB

**Data Bus:**
- กำหนดปริมาณข้อมูลที่ transfer ได้ต่อครั้ง
- 64-bit Data Bus → transfer 8 bytes ต่อ clock cycle

**Control Bus:**
- Memory Read (MEMR) - สัญญาณอ่าน memory
- Memory Write (MEMW) - สัญญาณเขียน memory
- I/O Read (IOR) - อ่าน I/O port
- I/O Write (IOW) - เขียน I/O port
- Interrupt Request (IRQ) - ขอ CPU หยุดทำงานชั่วคราว
- Clock - สัญญาณนาฬิกาซิงค์การทำงาน

---

## 6. Fetch-Decode-Execute Cycle

### 6.1 ขั้นตอนการทำงาน

```
┌─────────────────────────────────────────────────────────────┐
│              Fetch-Decode-Execute Cycle                     │
│                                                             │
│  ┌─────────────────┐                                        │
│  │    1. FETCH     │                                        │
│  │  RIP → Memory   │  ← PC (Program Counter/RIP) ชี้ที่      │
│  │  Read Instruction│    instruction ถัดไป                  │
│  │  RIP++          │  ← RIP เพิ่มขึ้นอัตโนมัติ              │
│  └────────┬────────┘                                        │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐                                        │
│  │    2. DECODE    │                                        │
│  │  IR → Control   │  ← Instruction Register (IR) เก็บ       │
│  │  Unit decodes   │    instruction ที่ fetch มา             │
│  │  the opcode     │  ← Control Unit แปล opcode             │
│  └────────┬────────┘                                        │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐                                        │
│  │    3. EXECUTE   │                                        │
│  │  ALU performs   │  ← ALU หรือ Memory unit ทำงาน          │
│  │  the operation  │                                        │
│  │  Store result   │  ← เก็บผลลัพธ์ลงใน register/memory    │
│  └────────┬────────┘                                        │
│           │                                                  │
│           └──────────────────────────────► กลับไป FETCH     │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 ตัวอย่างแบบ Step-by-Step

สมมติเรามี code นี้:

```nasm
; โปรแกรมง่ายๆ
mov rax, 5     ; address 0x1000: opcode B8 05 00 00 00 00 00 00
add rax, 3     ; address 0x100A: opcode 48 83 C0 03
```

**Cycle 1: `mov rax, 5`**
```
FETCH:
    RIP = 0x1000
    Memory[0x1000] → IR = B8 05 00 00 00 00 00 00
    RIP = RIP + 10 = 0x100A  (ขนาดของ instruction นี้)

DECODE:
    Opcode B8 = MOV RAX, immediate
    Operand = 0x0000000000000005

EXECUTE:
    RAX ← 5
    (ไม่มี memory access)
```

**Cycle 2: `add rax, 3`**
```
FETCH:
    RIP = 0x100A
    Memory[0x100A] → IR = 48 83 C0 03
    RIP = RIP + 4 = 0x100E

DECODE:
    Opcode 48 83 C0 = ADD RAX, immediate (sign-extended 8-bit)
    Operand = 3

EXECUTE:
    ALU: RAX ← RAX + 3 = 5 + 3 = 8
    FLAGS ← updated (ZF=0, CF=0, etc.)
```

---

## 7. Clock Cycles และ CPI

### 7.1 Clock Frequency

```
Clock = สัญญาณนาฬิกาที่ sync การทำงานของ CPU

3 GHz CPU:
    Clock period = 1/3,000,000,000 = ~0.333 nanoseconds
    ทุก 0.333ns จะมี 1 clock cycle

1 clock cycle ≈ 0.333 ns (3 GHz)
1 clock cycle ≈ 0.25 ns  (4 GHz)
1 clock cycle ≈ 0.2 ns   (5 GHz)
```

### 7.2 CPI (Cycles Per Instruction)

```
CPI = จำนวน clock cycles ที่ต้องใช้ per instruction

ตัวอย่าง CPI (approximate, Intel x86-64):
  MOV reg, reg    → 1 cycle
  ADD reg, reg    → 1 cycle
  MUL reg, reg    → 3-5 cycles
  DIV reg, reg    → 20-40 cycles
  MOV reg, [mem]  → 4-5 cycles (cache hit) / 100+ (cache miss)
  JMP/CALL        → 1-3 cycles (branch prediction hit)

CPU Time = Instructions × CPI × Clock Period
         = Instructions × CPI / Clock Frequency
```

### 7.3 ตัวอย่าง Performance Calculation

```
โจทย์: โปรแกรมมี 1,000,000 instructions
       CPU speed = 3 GHz
       Average CPI = 2

CPU Time = 1,000,000 × 2 / 3,000,000,000
         = 2,000,000 / 3,000,000,000
         = 0.000667 seconds
         ≈ 0.667 milliseconds
```

---

## 8. Pipeline Architecture

### 8.1 แนวคิด Pipeline

Pipeline เปรียบเหมือนสายการผลิต: แทนที่จะรอ instruction หนึ่งเสร็จก่อนค่อยทำอีก instruction ให้ทำหลาย instructions พร้อมกันในแต่ละ stage

```
Without Pipeline (Sequential):
    Inst 1: [Fetch][Decode][Execute]
    Inst 2:                         [Fetch][Decode][Execute]
    Inst 3:                                                 [Fetch][Decode][Execute]
    
    เวลา: 3 × 3 = 9 cycles สำหรับ 3 instructions

With 3-Stage Pipeline:
    Cycle:   1       2       3       4       5
    Inst 1: [Fetch][Decode][Execute]
    Inst 2:        [Fetch][Decode][Execute]
    Inst 3:               [Fetch][Decode][Execute]
    
    เวลา: 5 cycles สำหรับ 3 instructions (เร็วขึ้น 44%)
```

### 8.2 Modern CPU Pipeline (Intel Sandy Bridge 14 stages)

```
Stage 1:  Instruction Fetch (IF1)
Stage 2:  Instruction Fetch (IF2)
Stage 3:  Instruction Queue
Stage 4:  Instruction Decode (ID1) - Decode prefix
Stage 5:  Instruction Decode (ID2) - Decode opcode
Stage 6:  Micro-op Queue
Stage 7:  Rename/Allocate
Stage 8:  Micro-op Dispatch
Stage 9:  Execute (EX1)
Stage 10: Execute (EX2)
Stage 11: Execute (EX3)
Stage 12: Write-back (WB1)
Stage 13: Write-back (WB2)
Stage 14: Retire
```

### 8.3 Pipeline Hazards

```
1. Structural Hazards:
   Hardware ไม่สามารถทำ 2 operations พร้อมกันได้
   แก้: Stall (bubble) หรือเพิ่ม hardware unit

2. Data Hazards (RAW - Read After Write):
   Inst A: ADD RAX, 5    ← เขียน RAX
   Inst B: MOV RBX, RAX  ← อ่าน RAX  ← ต้องรอ Inst A เสร็จก่อน!
   แก้: Pipeline stall, Forwarding/Bypassing

3. Control Hazards (Branch Hazards):
   JNZ somewhere        ← ยังไม่รู้ว่าจะ jump ไปไหน
                           ต้องรอ calculate branch condition ก่อน
   แก้: Branch Prediction
```

**ตัวอย่าง code ที่แสดง Data Hazard:**

```nasm
; ไฟล์: pipeline_hazard.asm
; แสดง Data Hazard และวิธีหลีกเลี่ยง
; คอมไพล์: nasm -f elf64 pipeline_hazard.asm -o pipeline_hazard.o
;           ld pipeline_hazard.o -o pipeline_hazard

section .data
    x   dq 10
    y   dq 20
    z   dq 0

section .text
    global _start

_start:
    ; === วิธีที่แย่: Data Hazard ===
    ; CPU ต้องรอ (stall) เพราะ rax ยังไม่พร้อม
    mov rax, [x]        ; Load x → rax (4-5 cycles)
    add rax, 5          ; ← อาจต้อง stall รอ rax จาก instruction บน
    mov [z], rax        ; store ผลลัพธ์
    
    ; === วิธีที่ดีกว่า: แทรก independent instruction ===
    mov rax, [x]        ; Load x
    mov rbx, [y]        ; Load y (independent - CPU ทำพร้อมกับ rax ได้)
    add rax, rbx        ; ตอนนี้ rax พร้อมแล้ว (ใช้เวลาใน load rbx)
    mov [z], rax        ; store
    
    ; ผล: เร็วกว่าเพราะลด pipeline stall
    
    ; === Branch Prediction Example ===
    mov rcx, 100        ; loop 100 ครั้ง
.loop:
    ; CPU จะ predict ว่า loop จะ taken (เพราะเกิดบ่อย)
    ; ถ้า predict ถูก → ไม่เสีย cycles
    ; ถ้า predict ผิด (loop end) → เสีย ~15 cycles
    dec rcx
    jnz .loop           ; branch taken 99 ครั้ง, not taken 1 ครั้ง
                        ; Branch predictor จะ predict taken เกือบตลอด
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 9. CISC vs RISC

### 9.1 CISC (Complex Instruction Set Computer)

**ตัวอย่าง: x86/x86-64 (Intel, AMD)**

```
ลักษณะ:
- Instructions หลายร้อยแบบ
- Instructions มีความซับซ้อนสูง
- ทำงานกับ memory ได้โดยตรง
- ขนาด instruction ไม่เท่ากัน (1-15 bytes ใน x86-64)
- มี microcode: complex instruction → หลาย micro-operations

ตัวอย่าง CISC instruction:
  MOVSB   ; copy byte from [RSI] to [RDI], increment both
           ; ทำ 3 operations ในคำสั่งเดียว:
           ; 1. Read [RSI]
           ; 2. Write to [RDI]
           ; 3. Increment RSI and RDI
  
  REP MOVSD ; copy ECX DWORDs จาก RSI ไป RDI
             ; ทำ loop ใน 1 instruction!
  
  MUL mem   ; คูณ RAX กับค่าใน memory โดยตรง
             ; ไม่ต้อง load ก่อน
```

```nasm
; ตัวอย่าง CISC code (x86-64)
; คัดลอก string 10 bytes
section .data
    src  db "Hello Asm!", 0
section .bss
    dst  resb 11

section .text
    global _start
_start:
    ; CISC way: ใช้ REP MOVSB
    lea rsi, [src]      ; source address
    lea rdi, [dst]      ; destination address
    mov rcx, 10         ; จำนวน bytes
    rep movsb           ; copy RCX bytes จาก [RSI] ไป [RDI]
                        ; instruction เดียวทำได้ทั้งหมด!
    
    mov rax, 60
    xor rdi, rdi
    syscall
```

### 9.2 RISC (Reduced Instruction Set Computer)

**ตัวอย่าง: ARM (iPhone, Android, Apple Silicon, Raspberry Pi)**

```
ลักษณะ:
- Instructions น้อยกว่า (แต่ทรงพลัง)
- ทุก instruction ขนาดเท่ากัน (4 bytes ใน ARM64)
- Load/Store architecture: ทำงานกับ memory เฉพาะ LOAD/STORE
- ทำ operations กับ registers เท่านั้น
- Pipeline-friendly มากกว่า
- Compiler สร้าง efficient code ได้ง่ายกว่า
```

```asm
// ตัวอย่าง RISC code (ARM64/AArch64)
// คัดลอก string - ต้องทำทีละ step

// ARM64 way: load/store แยกกัน
// Syntax: GAS (GNU Assembler)
// คอมไพล์: as -o arm_demo.o arm_demo.s
//           ld arm_demo.o -o arm_demo

.text
.global _start

_start:
    // Load address of src
    adr x0, src          // x0 = address of src
    adr x1, dst          // x1 = address of dst
    mov x2, #10          // x2 = 10 (counter)

copy_loop:
    ldrb w3, [x0], #1   // Load byte from [x0], post-increment x0
    strb w3, [x1], #1   // Store byte to [x1], post-increment x1
    subs x2, x2, #1     // x2-- and set flags
    b.ne copy_loop       // if not zero, continue

    // Exit
    mov x8, #93          // syscall: exit
    mov x0, #0           // exit code 0
    svc #0               // system call

.data
src: .ascii "Hello Asm!"
.bss
dst: .space 11
```

### 9.3 เปรียบเทียบ CISC vs RISC

| ด้าน | CISC (x86) | RISC (ARM) |
|------|-----------|-----------|
| Instruction count | มาก (100s) | น้อย (ค่อนข้าง) |
| Instruction size | ไม่เท่ากัน | เท่ากัน (4 bytes) |
| Memory access | ทุก instruction | Load/Store only |
| Pipeline | ซับซ้อนกว่า | Pipeline-friendly |
| Power consumption | สูงกว่า | ต่ำกว่า |
| Code density | สูง (compact) | ต่ำกว่า |
| Use case | Desktop, Server | Mobile, Embedded |

---

## 10. Endianness

### 10.1 ความหมาย

Endianness คือลำดับการเก็บ bytes ในหน่วยความจำ

```
ค่า 0x12345678 (32-bit integer):
  Byte 0 = 0x12  (Most Significant Byte - MSB)
  Byte 1 = 0x34
  Byte 2 = 0x56
  Byte 3 = 0x78  (Least Significant Byte - LSB)

Little-Endian (x86, x86-64, ARM ใน LE mode):
  Address: 0x1000  0x1001  0x1002  0x1003
  Value:   0x78    0x56    0x34    0x12
           ^LSB                    ^MSB
  → เก็บ byte ที่สำคัญน้อยที่สุดก่อน (ที่ address ต่ำสุด)

Big-Endian (PowerPC, SPARC, Network protocols):
  Address: 0x1000  0x1001  0x1002  0x1003
  Value:   0x12    0x34    0x56    0x78
           ^MSB                    ^LSB
  → เก็บ byte ที่สำคัญที่สุดก่อน (ที่ address ต่ำสุด)
```

### 10.2 ตัวอย่าง Endianness ใน Assembly

```nasm
; ไฟล์: endianness.asm
; แสดง endianness ของ x86-64 (Little-Endian)
; คอมไพล์: nasm -f elf64 endianness.asm -o endianness.o
;           ld endianness.o -o endianness

section .data
    ; เก็บ 32-bit value 0x12345678
    value32  dd 0x12345678      ; dd = Define Doubleword (32-bit)
    
    ; เก็บ 64-bit value
    value64  dq 0x0102030405060708

section .bss
    buffer   resb 16

section .text
    global _start

_start:
    ; === อ่าน 32-bit value ===
    mov eax, [value32]      ; eax = 0x12345678
    ; ใน memory: 78 56 34 12  (Little-Endian!)
    ; แต่ใน register: 0x12345678 (ถูกต้อง - CPU จัดการให้)
    
    ; === อ่านทีละ byte เพื่อดู byte order ===
    movzx eax, byte [value32]       ; อ่าน byte แรก = 0x78 (LSB!)
    movzx ebx, byte [value32 + 1]   ; อ่าน byte ที่ 2 = 0x56
    movzx ecx, byte [value32 + 2]   ; อ่าน byte ที่ 3 = 0x34
    movzx edx, byte [value32 + 3]   ; อ่าน byte ที่ 4 = 0x12 (MSB!)
    
    ; eax = 0x78, ebx = 0x56, ecx = 0x34, edx = 0x12
    ; ยืนยัน: x86 เป็น Little-Endian
    
    ; === BSWAP: แปลง Little-Endian ↔ Big-Endian ===
    mov eax, 0x12345678
    bswap eax               ; swap bytes: 0x78563412
    ; ใช้เมื่อรับ/ส่งข้อมูลผ่าน network (Network byte order = Big-Endian)
    
    ; === สำหรับ Network Programming ===
    ; htons (host to network short) - แปลง 16-bit
    mov ax, 0x1234          ; host byte order (Little-Endian)
    xchg ah, al             ; swap bytes → 0x3412 (network byte order)
    
    ; htonl (host to network long) - แปลง 32-bit
    mov eax, 0x12345678
    bswap eax               ; → 0x78563412
    
    mov rax, 60
    xor rdi, rdi
    syscall

; หมายเหตุการทดสอบ Endianness:
; สามารถ disassemble ดูได้:
; objdump -d endianness | less
; หรือใช้ gdb:
; gdb endianness
; (gdb) x/4xb &value32   ← แสดง 4 bytes ใน hex
; Output: 0x... 0x78  0x56  0x34  0x12  ← Little-Endian!
```

---

## 11. Virtual Memory Overview

### 11.1 ความหมาย

Virtual Memory คือระบบที่ทำให้แต่ละ process เห็นว่าตัวเองมี memory ทั้งหมดสำหรับตัวเอง

```
Physical Memory (RAM): 16 GB จริงๆ
Virtual Address Space (each process): 256 TB (48-bit)

Process A: เห็น address 0x0000 ถึง 0x7FFF_FFFF_FFFF
Process B: เห็น address 0x0000 ถึง 0x7FFF_FFFF_FFFF

ทั้งสองใช้ virtual address เดียวกัน แต่ map ไปยัง physical address ต่างกัน!
```

### 11.2 Virtual Memory Layout (Linux x86-64)

```
Virtual Address Space (User Space):

High Address (0xFFFF_FFFF_FFFF_FFFF)
┌──────────────────────────────────────┐
│         Kernel Space                 │  ← ห้าม user access
│  (0xFFFF_8000_0000_0000 and above)  │
├──────────────────────────────────────┤
│         Stack                        │  ← เติบโตลงมา ↓
│         (grows downward)             │
├──────────────────────────────────────┤
│         (unmapped gap)               │
├──────────────────────────────────────┤
│         Shared Libraries             │
│         (.so / .dll files)           │
├──────────────────────────────────────┤
│         Heap                         │  ← เติบโตขึ้น ↑
│         (grows upward)               │
├──────────────────────────────────────┤
│         BSS Segment                  │  ← uninitialized data
├──────────────────────────────────────┤
│         Data Segment                 │  ← initialized data
├──────────────────────────────────────┤
│         Text Segment                 │  ← executable code
│         (0x0000_0040_0000 typical)  │
└──────────────────────────────────────┘
Low Address (0x0000_0000_0000_0000)
```

### 11.3 ตัวอย่าง Memory Layout ใน Assembly

```nasm
; ไฟล์: memory_layout.asm
; แสดง memory segments ต่างๆ
; คอมไพล์: nasm -f elf64 memory_layout.asm -o memory_layout.o
;           ld memory_layout.o -o memory_layout
; รัน: ./memory_layout
; ดู map: cat /proc/$(pgrep memory_layout)/maps

section .data
    ; Data Segment: initialized data
    ; อยู่ที่ address ต่ำกว่า heap
    initialized_var   dq 42        ; 64-bit integer = 42
    hello_str         db "Hello World", 0
    pi_value          dq 3         ; approximation

section .bss
    ; BSS Segment: uninitialized data (ถูก zero-initialized โดย OS)
    ; ไม่กิน space ใน executable file!
    large_buffer      resb 4096    ; 4KB buffer
    counter           resq 1       ; 1 quadword (64-bit)
    temp_vars         resq 10      ; 10 × 64-bit variables

section .text
    global _start

_start:
    ; โปรแกรมอยู่ใน Text Segment
    ; ค่าตัวแปรอยู่ใน Data/BSS Segment
    ; Stack อยู่ที่ top ของ virtual memory (เติบโตลง)
    
    ; Stack pointer (RSP) ชี้ที่ top ของ stack
    ; เมื่อ push ข้อมูล RSP ลดลง (stack grows downward)
    
    ; === แสดงการใช้ stack ===
    push rax            ; RSP -= 8, [RSP] = rax
    push rbx            ; RSP -= 8, [RSP] = rbx
    push rcx            ; RSP -= 8, [RSP] = rcx
    
    ; RSP ตอนนี้ต่ำกว่าตอนเริ่ม 24 bytes
    
    pop rcx             ; rcx = [RSP], RSP += 8
    pop rbx             ; rbx = [RSP], RSP += 8
    pop rax             ; rax = [RSP], RSP += 8
    
    ; RSP กลับมาที่เดิม
    
    ; === ทดสอบ BSS ถูก zero-initialized ===
    mov rax, [counter]  ; ควรเป็น 0 (zero-initialized)
    ; rax = 0
    
    ; === Write ค่าลง memory ===
    mov qword [counter], 100    ; counter = 100
    mov rax, [counter]          ; rax = 100
    
    ; จบโปรแกรม
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 12. ตัวอย่างรวม: Demo โชว์ Architecture Concepts

```nasm
; ไฟล์: arch_demo.asm
; โปรแกรมรวมแสดงแนวคิด architecture ต่างๆ
; คอมไพล์: nasm -f elf64 arch_demo.asm -o arch_demo.o
;           ld arch_demo.o -o arch_demo
; รัน: ./arch_demo && echo "Exit: $?"

section .data
    ; Test data
    numbers dq 10, 20, 30, 40, 50     ; array of 5 qwords
    count   equ 5                      ; array size (constant)

section .bss
    sum     resq 1      ; เก็บผลรวม
    avg     resq 1      ; เก็บค่าเฉลี่ย

section .text
    global _start

_start:
    ; ============================================
    ; DEMO 1: Register Usage
    ; แสดงการใช้ registers ต่างๆ
    ; ============================================
    xor rax, rax        ; rax = 0 (sum accumulator)
    xor rbx, rbx        ; rbx = 0 (loop counter)
    lea rsi, [numbers]  ; rsi = pointer to array
    
    ; ============================================
    ; DEMO 2: Loop with Pipeline-Friendly Code
    ; Sequential memory access = Cache Friendly
    ; ============================================
.sum_loop:
    cmp rbx, count          ; ครบ 5 elements?
    jge .sum_done           ; ถ้าครบ ออกจาก loop
    
    mov rcx, [rsi + rbx*8]  ; โหลด numbers[rbx]
    ; โหลด address ถัดไปด้วยเพื่อ prefetch
    add rax, rcx            ; sum += numbers[rbx]
    
    inc rbx                 ; rbx++
    jmp .sum_loop
    
.sum_done:
    ; rax = 10 + 20 + 30 + 40 + 50 = 150
    mov [sum], rax          ; เก็บผลรวม
    
    ; ============================================
    ; DEMO 3: Division (ALU operation)
    ; ============================================
    mov rax, [sum]          ; rax = 150 (dividend)
    xor rdx, rdx            ; rdx = 0 (rdx:rax = dividend pair)
    mov rbx, count          ; rbx = 5 (divisor)
    div rbx                 ; rax = 150/5 = 30, rdx = 150%5 = 0
    ; rax = 30 (quotient)
    ; rdx = 0  (remainder)
    
    mov [avg], rax          ; เก็บค่าเฉลี่ย = 30
    
    ; ============================================
    ; DEMO 4: Bit manipulation (ALU operations)
    ; ============================================
    mov rax, [avg]          ; rax = 30 = 0b00011110
    
    ; ตรวจสอบว่าเป็นเลขคู่หรือไม่
    test rax, 1             ; test bit 0
    jz .is_even             ; ถ้า bit 0 = 0, เป็นเลขคู่
    ; ถ้าถึงตรงนี้ = เลขคี่
    jmp .done
    
.is_even:
    ; 30 เป็นเลขคู่
    ; ทำการ multiply by 2 ด้วย shift
    shl rax, 1              ; rax = rax * 2 = 60
    ; (shift left 1 = คูณ 2, เร็วกว่า mul!)
    
.done:
    ; ============================================
    ; DEMO 5: Stack operations
    ; ============================================
    ; บันทึก registers ที่จะใช้
    push rax                ; save rax
    push rbx                ; save rbx
    
    ; ทำงานบางอย่าง
    mov rax, 0xFF
    mov rbx, 0x0F
    and rax, rbx            ; rax = 0x0F
    
    ; restore registers
    pop rbx                 ; rbx กลับมาเหมือนเดิม
    pop rax                 ; rax กลับมาเหมือนเดิม
    ; (rax = 60 จากการคำนวณก่อนหน้า)
    
    ; ============================================
    ; EXIT: return sum/10 as exit code
    ; ============================================
    mov rax, 60             ; syscall: exit
    mov rdi, [avg]          ; exit code = avg = 30
    syscall
    ; ทดสอบ: ./arch_demo; echo $?  → ควรแสดง 30
```

**วิธีทดสอบ:**
```bash
# Compile
nasm -f elf64 arch_demo.asm -o arch_demo.o
ld arch_demo.o -o arch_demo

# Run
./arch_demo
echo "Exit code (avg): $?"   # ควรได้ 30

# Debug ด้วย GDB
gdb arch_demo
(gdb) break _start
(gdb) run
(gdb) info registers     # ดู registers ทั้งหมด
(gdb) si                 # step instruction
(gdb) print $rax         # ดูค่า rax
```

---

## 13. ข้อผิดพลาดที่พบบ่อย (Common Mistakes)

### 13.1 ความเข้าใจผิดเรื่อง Endianness

```nasm
; ❌ ผิด: คิดว่า byte แรกใน memory คือ MSB
section .data
    value dd 0x12345678

; ✅ ถูก: ใน Little-Endian (x86), byte แรกคือ LSB
; memory layout: 78 56 34 12
; ต้องใช้ BSWAP เมื่อต้องการ Big-Endian
mov eax, [value]    ; eax = 0x12345678 (ถูกต้อง - CPU จัดการให้)
bswap eax           ; eax = 0x78563412 (ถ้าต้องการ Big-Endian)
```

### 13.2 ปัญหา Stack Alignment

```nasm
; ❌ ผิด: เรียก function โดย stack ไม่ align 16 bytes
call printf         ; อาจ crash ถ้า stack ไม่ 16-byte aligned!

; ✅ ถูก: ตรวจสอบ alignment ก่อน call
; x86-64 ABI กำหนด: stack ต้อง align 16 bytes ก่อน CALL instruction
; (หลัง CALL, RSP % 16 == 8 เพราะ CALL push return address)
and rsp, -16        ; align stack to 16 bytes
sub rsp, 8          ; adjust สำหรับ return address
call printf
```

### 13.3 การ Write ผ่าน EAX ล้าง upper 32 bits ของ RAX

```nasm
; ❌ อาจทำให้งง:
mov rax, 0xDEADBEEFCAFEBABE
mov eax, 0x12345678         ; ← นี่จะ ZERO รึเปล่า upper 32 bits!
; rax = 0x0000000012345678  ← ไม่ใช่ 0xDEADBEEF12345678!

; ✅ ถ้าต้องการเก็บ upper 32 bits:
mov rax, 0xDEADBEEFCAFEBABE
mov ax, 0x1234               ; เขียนเฉพาะ 16 bits ล่าง
; rax = 0xDEADBEEFCAFE1234  ← OK!
```

### 13.4 ลืม Clear RDX ก่อน DIV

```nasm
; ❌ ผิด: ลืม clear rdx
mov rax, 100
div rbx             ; ← rdx อาจมีค่าเหลือจากการคำนวณก่อนหน้า!
                    ; จะทำให้ได้ผลผิด หรือ divide overflow exception

; ✅ ถูก:
mov rax, 100
xor rdx, rdx        ; ← clear rdx ก่อนเสมอ!
div rbx
```

### 13.5 Confuse ระหว่าง Address และ Value

```nasm
section .data
    x dq 42

section .text
    ; ❌ ผิด: ใช้ label โดยตรง = ค่า address
    mov rax, x          ; rax = address ของ x (ไม่ใช่ค่า 42!)
    
    ; ✅ ถูก: ใช้ [] เพื่อ dereference
    mov rax, [x]        ; rax = 42 (ค่าที่เก็บอยู่ใน x)
    
    ; ✅ ถ้าต้องการ address:
    lea rax, [x]        ; rax = address ของ x (ถูกต้องตามเจตนา)
```

---

## 14. แบบฝึกหัด (Practical Exercises)

### แบบฝึกหัดที่ 1: Memory Viewer (Beginner)

เขียนโปรแกรมที่ตรวจสอบ endianness ของระบบ

```nasm
; ไฟล์: ex01_endian_check.asm
; โจทย์: ตรวจสอบว่าระบบเป็น Little-Endian หรือ Big-Endian
; แสดงผลผ่าน exit code: 0 = little-endian, 1 = big-endian
; คอมไพล์: nasm -f elf64 ex01_endian_check.asm -o ex01.o && ld ex01.o -o ex01
; ทดสอบ: ./ex01 && echo "Little-Endian" || echo "Big-Endian"

; 힌트: เก็บ 0x01 ใน 32-bit memory แล้วอ่านทีละ byte

section .data
    test_val dd 0x00000001      ; เก็บค่า 1 เป็น 32-bit

section .text
    global _start

_start:
    ; TODO: อ่าน byte แรกของ test_val
    ; ถ้า byte แรก = 0x01 → Little-Endian
    ; ถ้า byte แรก = 0x00 → Big-Endian
    
    movzx rax, byte [test_val]  ; อ่าน byte แรก
    
    ; ตรวจสอบผล
    cmp rax, 1                  ; ถ้า byte แรก = 1 → Little-Endian
    je .little_endian
    
    ; Big-Endian
    mov rax, 60
    mov rdi, 1                  ; exit code 1 = Big-Endian
    syscall
    
.little_endian:
    mov rax, 60
    xor rdi, rdi                ; exit code 0 = Little-Endian
    syscall
```

### แบบฝึกหัดที่ 2: Bit Manipulation (Beginner-Intermediate)

```nasm
; ไฟล์: ex02_bit_ops.asm
; โจทย์: เขียนโปรแกรมที่ทำ bit operations ต่อไปนี้
; 1. Set bit 3 ของ register
; 2. Clear bit 5 ของ register
; 3. Toggle bit 7 ของ register
; 4. Check bit 0 (ตรวจว่าเลขคู่/คี่)

section .text
    global _start

_start:
    mov rax, 0b10100000     ; rax = 0xA0 = 160

    ; === SET bit 3 ===
    ; OR กับ mask ที่มี bit 3 = 1
    or rax, (1 << 3)        ; rax |= 0b00001000
    ; rax = 0b10101000 = 0xA8 = 168

    ; === CLEAR bit 5 ===
    ; AND กับ mask ที่มี bit 5 = 0 (ทุก bit อื่น = 1)
    and rax, ~(1 << 5)      ; rax &= ~0b00100000
    ; bit 5 ถูก clear: rax = 0b10001000 = 0x88 = 136

    ; === TOGGLE bit 7 ===
    ; XOR กับ mask ที่มี bit 7 = 1
    xor rax, (1 << 7)       ; rax ^= 0b10000000
    ; toggle bit 7: rax = 0b00001000 = 0x08 = 8

    ; === CHECK bit 0 (even/odd) ===
    test rax, 1             ; test bit 0
    jz .is_even             ; ZF=1 ถ้า bit 0 = 0 (เลขคู่)
    ; เลขคี่
    mov rdi, 1
    jmp .exit
.is_even:
    ; เลขคู่
    xor rdi, rdi

.exit:
    mov rax, 60
    syscall
    ; exit code: 0 = even, 1 = odd
    ; 8 เป็นเลขคู่ → exit code 0
```

### แบบฝึกหัดที่ 3: Array Sum with Cache Optimization (Intermediate)

```nasm
; ไฟล์: ex03_array_sum.asm
; โจทย์: หาผลรวมของ array ด้วย 2 วิธี
; วิธี 1: Sequential access (Cache-friendly)
; วิธี 2: เปรียบเทียบประสิทธิภาพ
; ทดสอบ: ./ex03; echo "Sum exit code: $?"

section .data
    ; Array 10 elements
    arr dq 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
    arr_len equ 10

section .text
    global _start

_start:
    ; หาผลรวม array
    xor rax, rax            ; sum = 0
    xor rcx, rcx            ; i = 0
    lea rsi, [arr]          ; pointer to arr

.loop:
    cmp rcx, arr_len        ; i < arr_len?
    jge .done

    add rax, [rsi + rcx*8]  ; sum += arr[i]
    inc rcx                 ; i++
    jmp .loop

.done:
    ; rax = 1+2+3+...+10 = 55
    ; return sum ผ่าน exit code
    mov rdi, rax            ; exit code = sum = 55
    mov rax, 60
    syscall
    ; ทดสอบ: ./ex03; echo $?  → ควรได้ 55
```

### แบบฝึกหัดที่ 4: BSWAP Network Byte Order (Intermediate)

```nasm
; ไฟล์: ex04_network_bytes.asm
; โจทย์: แปลง IP address จาก host byte order เป็น network byte order
; IP: 192.168.1.1
; Host (Little-Endian): 0x0101A8C0
; Network (Big-Endian):  0xC0A80101

section .data
    ; IP 192.168.1.1 ใน Little-Endian order
    ; 192 = 0xC0, 168 = 0xA8, 1 = 0x01, 1 = 0x01
    ip_host dd 0x0101A8C0   ; Little-Endian representation

section .text
    global _start

_start:
    ; โหลด IP ใน host byte order
    mov eax, [ip_host]      ; eax = 0x0101A8C0
    
    ; แปลงเป็น network byte order (Big-Endian)
    bswap eax               ; eax = 0xC0A80101
    
    ; ตอนนี้ eax = 0xC0A80101
    ; สามารถส่งผ่าน network ได้โดยตรง
    
    ; แยก octets ออกมาตรวจสอบ
    ; eax = C0 A8 01 01
    ;       ^MSB      ^LSB (ใน Big-Endian)
    
    ; ดึง octet แรก (192 = 0xC0)
    mov ebx, eax
    shr ebx, 24             ; ebx = 0xC0 = 192
    
    ; ดึง octet สอง (168 = 0xA8)
    mov ecx, eax
    shr ecx, 16
    and ecx, 0xFF           ; ecx = 0xA8 = 168
    
    mov rax, 60
    mov rdi, rbx            ; exit code = first octet = 192
    syscall
    ; ทดสอบ: ./ex04; echo $?  → ควรได้ 192
```

### แบบฝึกหัดที่ 5: Fibonacci ด้วย Registers (Intermediate-Advanced)

```nasm
; ไฟล์: ex05_fibonacci.asm
; โจทย์: คำนวณ Fibonacci sequence ใช้เฉพาะ registers
; (ไม่ใช้ memory) เพื่อ maximize cache efficiency
; หา F(10) = 55

section .text
    global _start

_start:
    ; F(0) = 0, F(1) = 1
    ; F(n) = F(n-1) + F(n-2)
    
    ; ใช้ registers:
    ; rax = F(n-2)  (previous previous)
    ; rbx = F(n-1)  (previous)
    ; rcx = n (target)
    ; rdx = loop counter
    ; rsi = F(n)   (current)
    
    xor rax, rax        ; F(0) = 0
    mov rbx, 1          ; F(1) = 1
    mov rcx, 10         ; ต้องการ F(10)
    mov rdx, 2          ; เริ่มจาก F(2)
    
.fib_loop:
    cmp rdx, rcx        ; ถึง F(10) แล้วหรือยัง?
    jg .fib_done
    
    mov rsi, rax        ; rsi = F(n-2)
    add rsi, rbx        ; rsi = F(n-2) + F(n-1) = F(n)
    
    mov rax, rbx        ; F(n-2) ← F(n-1)
    mov rbx, rsi        ; F(n-1) ← F(n)
    
    inc rdx             ; n++
    jmp .fib_loop
    
.fib_done:
    ; rbx = F(10) = 55
    mov rax, 60
    mov rdi, rbx        ; exit code = F(10) = 55
    syscall
    ; ทดสอบ: ./ex05; echo $?  → ควรได้ 55
```

---

## 15. สรุปและ Key Takeaways

### สรุปสำคัญ

1. **Von Neumann Architecture** - ใช้ shared memory สำหรับ data และ instructions, มี bottleneck
2. **Harvard Architecture** - แยก memory สำหรับ data และ instructions, เร็วกว่าสำหรับ embedded
3. **CPU Components** - ALU (คำนวณ), CU (ควบคุม), Registers (เก็บค่าชั่วคราว), Cache (เก็บข้อมูลที่ใช้บ่อย)
4. **Memory Hierarchy** - Registers → L1 → L2 → L3 → RAM → Storage (เร็วลดลง, ใหญ่ขึ้น, ถูกลง)
5. **Fetch-Decode-Execute** - วงจรพื้นฐานของ CPU ทุกการทำงาน
6. **Pipeline** - ทำ instruction หลายตัวพร้อมกัน → เร็วขึ้น แต่มี hazards
7. **CISC vs RISC** - x86 (CISC: complex instructions) vs ARM (RISC: simple instructions)
8. **Little-Endian** - x86 เก็บ LSB ก่อน, ต้อง BSWAP เมื่อ network communication

### Checklist ก่อนเขียน Assembly

```
□ เข้าใจ target architecture (x86-64 หรือ ARM64?)
□ รู้จัก registers ที่ใช้ (general purpose, flags, etc.)
□ จำ endianness ของ architecture ได้
□ เข้าใจ memory layout (code, data, stack, heap)
□ รู้วิธี avoid cache misses (sequential access)
□ เข้าใจ pipeline hazards (data dependency)
□ รู้จัก ABI (Application Binary Interface) ของ OS
```

---

## 16. Resources และ References

### เครื่องมือที่แนะนำ

```bash
# ดู machine code ของ assembly:
nasm -f elf64 file.asm -o file.o
objdump -d file.o | less          # disassemble
objdump -d -M intel file.o        # Intel syntax

# Debug:
gdb ./program
(gdb) layout asm                  # แสดง assembly view
(gdb) info registers              # ดู registers ทั้งหมด
(gdb) x/16xb $rsp                 # ดู 16 bytes ที่ stack
(gdb) x/4xg &variable             # ดู 4 qwords ที่ variable

# ดู system calls:
man 2 syscall
cat /usr/include/asm/unistd_64.h  # syscall numbers
ausyscall --dump                   # หรือใช้ ausyscall

# Performance analysis:
perf stat ./program               # CPU statistics
perf record ./program             # profiling
perf report                       # วิเคราะห์ผล
valgrind --tool=cachegrind ./prog # cache analysis
```

### หนังสือแนะนำ

1. **"Computer Organization and Design" - Patterson & Hennessy**
   - คลาสสิคที่สุดสำหรับ computer architecture
   
2. **"Computer Systems: A Programmer's Perspective" - Bryant & O'Hallaron**
   - เน้น x86-64, ดีมากสำหรับ system programmers

3. **"Modern Processor Design: Fundamentals of Superscalar Processors" - Shen & Lipasti**
   - สำหรับคนที่ต้องการลึกลงไปเรื่อง pipeline และ out-of-order execution

### Online Resources

- **Intel 64 and IA-32 Architectures Software Developer Manuals**: https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
- **ARM Architecture Reference Manual**: https://developer.arm.com/documentation/
- **Compiler Explorer (Godbolt)**: https://godbolt.org/ - ดูว่า C code compile เป็น Assembly ยังไง
- **x86 Instruction Reference**: https://www.felixcloutier.com/x86/
- **NASM Documentation**: https://www.nasm.us/doc/

---

## สิ่งที่จะได้เรียนใน Part ถัดไป

**Part 003: x86 Registers and Data Types**
- ทุก register ใน x86-64 อย่างละเอียด
- Data types: BYTE, WORD, DWORD, QWORD
- การใช้ SIMD registers (XMM, YMM)
- Flags register และ bit flags ต่างๆ
- Register conventions ตาม System V AMD64 ABI

---

*Part 002 จบแล้ว! ให้แน่ใจว่าคุณ compile และ run ทุก code example และทำแบบฝึกหัดทั้ง 5 ข้อก่อนไปยัง Part ถัดไป*

*ถ้ามีข้อสงสัย ให้ลอง trace การทำงานทีละขั้นตอนด้วย GDB และสังเกต registers เปลี่ยนแปลง*
