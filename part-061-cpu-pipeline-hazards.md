# Part 061: CPU Pipeline และ Hazards

## บทนำ (Introduction)

ในบทนี้เราจะเรียนรู้เกี่ยวกับการทำงานภายในของ CPU สมัยใหม่ โดยเฉพาะ **pipeline architecture** และปัญหา **hazards** ต่าง ๆ ที่เกิดขึ้น การเข้าใจเรื่องเหล่านี้จะช่วยให้เราเขียนโค้ด assembly ที่มีประสิทธิภาพสูงได้

---

## 1. Modern Superscalar Pipeline Overview

### 1.1 แนวคิดพื้นฐาน

CPU สมัยใหม่ใช้สถาปัตยกรรม **superscalar out-of-order execution** ซึ่งหมายความว่า:

- **Superscalar**: สามารถ execute หลาย instruction พร้อมกันได้ในแต่ละ cycle
- **Out-of-Order (OoO)**: สามารถเรียงลำดับการ execute ใหม่ให้มีประสิทธิภาพมากขึ้น
- **In-Order Retire**: แม้จะ execute ไม่เป็นลำดับ แต่ผลลัพธ์จะถูก commit ตามลำดับโปรแกรมเสมอ

```
โมเดลการทำงานแบบง่าย:

Program Order:    [I1] → [I2] → [I3] → [I4] → [I5]
                   ↓      ↓      ↓      ↓      ↓
Fetch (In-Order): [I1]  [I2]  [I3]  [I4]  [I5]
                   ↓      ↓      ↓      ↓      ↓
Decode:           [I1]  [I2]  [I3]  [I4]  [I5]
                   ↓      ↓      ↓      ↓      ↓
Dispatch:          → Reservation Stations (OoO starts here)
                   ↓
Execute (OoO):    [I1]  [I3]  [I2]  [I5]  [I4]  ← อาจเรียงใหม่
                   ↓
Retire (In-Order): [I1] → [I2] → [I3] → [I4] → [I5]
```

### 1.2 ทำไมต้อง Out-of-Order?

ลองดูตัวอย่าง:

```nasm
; ตัวอย่าง: dependency chain ที่ทำให้เกิด stall
mov rax, [memory_address]   ; L1 cache miss: ใช้เวลา ~100 cycles!
add rbx, rax                 ; ต้องรอ rax → stall
add rcx, rbx                 ; ต้องรอ rbx → stall อีก

; แต่ถ้ามี instruction อื่นที่ไม่ depend กัน:
mov rdx, r8                  ; OoO processor จะ execute ตรงนี้ขณะรอ memory
imul r9, r10                 ; และตรงนี้ด้วย
add r11, r12                 ; และตรงนี้ด้วย
```

Out-of-Order execution ช่วยให้ CPU ทำงาน instruction ที่ "พร้อม" แทนที่จะรอ instruction ที่ยังไม่พร้อม

---

## 2. Pipeline Stages

### 2.1 IF - Instruction Fetch

```
┌─────────────────────────────────────────────┐
│  Instruction Fetch (IF)                      │
│                                             │
│  PC → L1 I-Cache → Instruction Buffer       │
│       ↓                                     │
│  Branch Predictor → Predicted PC             │
│                                             │
│  Intel Skylake: Fetch up to 16 bytes/cycle  │
│  → 4-6 instructions per cycle               │
└─────────────────────────────────────────────┘
```

**รายละเอียด:**
- อ่าน instruction จาก L1 Instruction Cache (32KB บน Skylake)
- Branch Predictor ทำนาย branch target ล่วงหน้า
- Loop Stream Detector (LSD) สำหรับ loop ขนาดเล็ก

### 2.2 ID - Instruction Decode

```
┌─────────────────────────────────────────────┐
│  Instruction Decode (ID)                     │
│                                             │
│  x86 Instructions → Micro-ops (μops)        │
│                                             │
│  Simple instructions: 1 μop                 │
│  Complex instructions: 2-4 μops             │
│  Very complex (e.g., CPUID): many μops      │
│                                             │
│  Skylake: decode up to 4 instructions/cycle │
└─────────────────────────────────────────────┘
```

**ตัวอย่าง Micro-op Fusion:**
```nasm
; Macro-fusion: decoder รวม 2 instructions เป็น 1 μop
cmp rax, rbx
je  .target         ; cmp+je → fused เป็น 1 μop

; Micro-fusion: รวม memory op กับ ALU op
add [rbp-8], rax    ; load + add + store → fused เป็น 1 μop (บางกรณี)
```

### 2.3 IS - Instruction Scheduling / Dispatch

```
┌─────────────────────────────────────────────┐
│  Dispatch / Issue                           │
│                                             │
│  μops → Reservation Stations               │
│       → Reorder Buffer (ROB)               │
│       → Register Alias Table (RAT)         │
│                                             │
│  Skylake ROB: 224 entries                   │
│  Scheduler (RS): 97 entries                 │
└─────────────────────────────────────────────┘
```

### 2.4 EX - Execute

```
┌─────────────────────────────────────────────┐
│  Execute (EX)                               │
│                                             │
│  Execution Units:                           │
│  ├── ALU (Arithmetic Logic Unit)           │
│  ├── AGU (Address Generation Unit)         │
│  ├── FPU (Floating Point Unit)             │
│  ├── Load Unit                              │
│  └── Store Unit                             │
│                                             │
│  Skylake: 8 execution ports                 │
└─────────────────────────────────────────────┘
```

### 2.5 WB - Write Back / Retire

```
┌─────────────────────────────────────────────┐
│  Write Back / Retire                        │
│                                             │
│  ROB → Commit results in program order      │
│      → Update architectural state          │
│      → Free physical registers             │
│      → Handle exceptions/interrupts        │
│                                             │
│  Skylake: retire up to 4 μops/cycle         │
└─────────────────────────────────────────────┘
```

---

## 3. Execution Units

### 3.1 ALU - Arithmetic Logic Unit

ALU ทำการคำนวณ integer operations:

```nasm
; ALU operations - latency 1 cycle, throughput 0.25 cycles
add  rax, rbx       ; Addition
sub  rax, rbx       ; Subtraction
and  rax, rbx       ; Bitwise AND
or   rax, rbx       ; Bitwise OR
xor  rax, rbx       ; Bitwise XOR
not  rax            ; Bitwise NOT
neg  rax            ; Negate

; Shift operations - latency 1, throughput 0.5
shl  rax, 2         ; Shift left (multiply by 4)
shr  rax, 2         ; Shift right unsigned
sar  rax, 2         ; Shift right signed

; Multiply - latency 3, throughput 1
imul rax, rbx       ; Signed multiply
imul rax, rbx, 5    ; rax = rbx * 5

; Divide - very slow! latency ~20-90 cycles
idiv rbx            ; Signed divide: rdx:rax / rbx
div  rbx            ; Unsigned divide
```

**Intel Skylake ALU Ports:**
- Port 0: ALU, multiply, rotate, shift, branch
- Port 1: ALU, multiply, LEA
- Port 5: ALU, vector shuffle, branch
- Port 6: ALU, shift, branch

### 3.2 AGU - Address Generation Unit

```nasm
; AGU คำนวณ effective address สำหรับ memory operations
mov  rax, [rbx + rcx*8 + 16]   ; AGU คำนวณ: rbx + rcx*8 + 16

; LEA ใช้ AGU สำหรับการคำนวณ (ไม่ access memory)
lea  rax, [rbx + rcx*4]         ; rax = rbx + rcx*4 (เร็วกว่า imul)
lea  rax, [rbx + rbx*2]         ; rax = rbx * 3
lea  rax, [rbx*4 + rcx]         ; rax = rbx*4 + rcx
```

**Ports สำหรับ Memory:**
- Port 2, 3: Load (Skylake มี 2 load units)
- Port 4: Store data
- Port 7: Store address

### 3.3 FPU - Floating Point Unit

```nasm
; Scalar FP operations (SSE)
addss  xmm0, xmm1    ; Add single float, latency 4, throughput 0.5
addsd  xmm0, xmm1    ; Add double float, latency 4, throughput 0.5
mulss  xmm0, xmm1    ; Multiply float, latency 4, throughput 0.5
divss  xmm0, xmm1    ; Divide float, latency ~11, throughput ~3

; Packed FP operations (SSE2/AVX)
addps  xmm0, xmm1    ; Add 4 floats packed
addpd  xmm0, xmm1    ; Add 2 doubles packed
vaddps ymm0, ymm1, ymm2  ; AVX: add 8 floats packed

; FMA (Fused Multiply-Add) - AVX2/FMA3
vfmadd213ps ymm0, ymm1, ymm2  ; ymm0 = ymm1*ymm0 + ymm2
```

---

## 4. Reservation Stations (RS)

### 4.1 แนวคิด

Reservation Stations เป็นบัฟเฟอร์ที่เก็บ μops ที่รอ execute:

```
┌──────────────────────────────────────────────────────────┐
│  Reservation Station (Scheduler)                         │
│                                                          │
│  Entry: [μop | src1 | src2 | dest | ready bits]         │
│                                                          │
│  ┌────┬──────────┬──────────┬──────┬──────────┐         │
│  │ op │  src1    │  src2    │ dest │  ready?  │         │
│  ├────┼──────────┼──────────┼──────┼──────────┤         │
│  │ADD │ p23(rdy) │ p45(rdy) │ p67  │  YES     │ → Issue │
│  │MUL │ p12(rdy) │ p34(NO!) │ p56  │  NO      │ → Wait  │
│  │SHL │ p56(rdy) │ p78(rdy) │ p90  │  YES     │ → Issue │
│  └────┴──────────┴──────────┴──────┴──────────┘         │
│                                                          │
│  Skylake RS: 97 entries                                  │
└──────────────────────────────────────────────────────────┘
```

### 4.2 วิธีทำงาน

```
1. μop ถูก dispatch เข้า RS พร้อม tag ของ physical register
2. RS ตรวจสอบว่า operands พร้อมหรือยัง (ready bits)
3. เมื่อ operands พร้อม → issue ไปยัง execution unit
4. หลัง execute → broadcast result บน Common Data Bus (CDB)
5. RS entries ที่รอ result นี้จะ update ready bits
```

---

## 5. Reorder Buffer (ROB)

### 5.1 วัตถุประสงค์

ROB ทำให้ CPU สามารถ:
- Execute out-of-order แต่ retire in-order
- Handle exceptions อย่างถูกต้อง (precise exceptions)
- รองรับ speculation (branch prediction)

### 5.2 โครงสร้าง

```
┌───────────────────────────────────────────────────────────┐
│  Reorder Buffer (ROB)                                     │
│                                                           │
│  Head ──────────────────────────────────────── Tail       │
│  (Retire end)                              (Dispatch end) │
│                                                           │
│  ┌──────┬────────────┬──────────┬──────────┬──────────┐  │
│  │  ID  │    μop     │  Result  │  Done?   │ Exception│  │
│  ├──────┼────────────┼──────────┼──────────┼──────────┤  │
│  │  42  │ ADD p5,p6  │   127    │  YES     │   No     │← Retire│
│  │  43  │ MUL p7,p8  │    -     │  NO      │   No     │  │
│  │  44  │ JMP pred   │   ?      │  spec.   │   No     │  │
│  │  45  │ ADD p9,p10 │   55     │  YES     │   No     │  │
│  │  46  │ ...        │   ...    │  ...     │  ...     │  │
│  └──────┴────────────┴──────────┴──────────┴──────────┘  │
│                                                           │
│  Skylake ROB: 224 entries                                 │
│  Can retire: 4 μops/cycle                                 │
└───────────────────────────────────────────────────────────┘
```

### 5.3 ROB ขนาดใหญ่ = Memory Latency Hiding ที่ดีขึ้น

```
ROB ขนาด N entries สามารถ "ซ่อน" latency ได้:

ถ้า L1 miss latency = 100 cycles
และ throughput ที่ต้องการ = 4 IPC

ROB ต้องมีอย่างน้อย: 100 × 4 = 400 entries
เพื่อให้ CPU มี work เพียงพอขณะรอ cache miss

Skylake ROB = 224 → เพียงพอสำหรับ ~56 independent instructions
ระหว่าง cache miss (224/4 = 56 cycles worth of instructions)
```

---

## 6. Register Renaming (RAT)

### 6.1 แนวคิด

Register renaming แก้ปัญหา **false dependencies** (WAR, WAW) โดย:
- Map **architectural registers** (rax, rbx, ...) → **physical registers** (p0, p1, p2, ...)
- Physical registers มีจำนวนมากกว่า architectural registers มาก

```
Architectural registers (x86-64): 16 GP registers
Physical registers (Skylake): ~180 GP registers
```

### 6.2 ตัวอย่าง Register Renaming

```nasm
; โค้ดต้นฉบับ (มี WAR dependency)
mov rax, [addr1]    ; Write rax
add rbx, rax        ; Read rax  ← RAW dependency (true)
mov rax, [addr2]    ; Write rax ← WAR dependency (false!) กับบรรทัด 2
add rcx, rax        ; Read rax  ← RAW dependency กับบรรทัด 3
```

```
หลัง Register Renaming:
    mov p1, [addr1]     ; rax → p1
    add p2, p1          ; rbx → p2, ใช้ p1
    mov p3, [addr2]     ; rax → p3 (renamed! ไม่ conflict กับ p1)
    add p4, p3          ; rcx → p4, ใช้ p3

RAT (Register Alias Table):
    rax → p3  (latest mapping)
    rbx → p2
    rcx → p4
```

### 6.3 การแก้ปัญหา False Dependencies

```nasm
; ปัญหา: False dependency จาก partial register update
; (ปัญหาเฉพาะ x86 เนื่องจาก 8/16-bit sub-registers)

; BAD: dependency chain ที่ไม่จำเป็น
; (เมื่อ CPU อ่าน eax หลัง mov al, [mem])
xor  eax, eax       ; เคลียร์ eax เพื่อหลีกเลี่ยง partial update
mov  al, [rbx]      ; ตอนนี้ eax/rax ไม่มี false dependency

; BAD: สำหรับ xmm registers
; บางครั้งต้องการ zero upper bits
vxorps xmm0, xmm0, xmm0  ; Zero ทั้ง register (recognized เป็น zero idiom)

; GOOD: zero-idiom ที่ CPU รู้จัก (ไม่ต้องรอ input)
xor  eax, eax       ; Zero idiom - CPU รู้ว่า result = 0 ทันที
sub  eax, eax       ; Zero idiom
pxor xmm0, xmm0    ; Zero idiom สำหรับ XMM
vxorps ymm0, ymm0, ymm0  ; Zero idiom สำหรับ YMM
```

---

## 7. Data Hazards

### 7.1 RAW - Read After Write (True Dependency)

RAW เป็น **true dependency** ที่หลีกเลี่ยงไม่ได้:

```nasm
; RAW hazard example
add  rax, rbx    ; (1) Write rax
add  rcx, rax    ; (2) Read rax ← ต้องรอ (1) เสร็จก่อน!
```

```
Timeline:
Cycle:  1    2    3    4    5    6
(1):   [IF] [ID] [EX] [WB]
(2):   [IF] [ID] [-]  [-]  [EX] [WB]
                  ↑
                  Stall 2 cycles (รอ rax จาก (1))
                  (กรณีไม่มี forwarding)
```

**การแก้ปัญหา RAW:**
```nasm
; แทรก independent instructions ระหว่างกัน
add  rax, rbx    ; (1) Write rax
mov  rdx, [mem]  ; Independent - ทำระหว่างรอ
mov  r8,  r9     ; Independent - ทำระหว่างรอ
add  rcx, rax    ; (4) Read rax - ตอนนี้ rax พร้อมแล้ว (บน OoO processor)
```

### 7.2 WAR - Write After Read (Anti Dependency)

WAR เป็น **false dependency** ที่ register renaming แก้ได้:

```nasm
; WAR hazard example
add  rcx, rax    ; (1) Read rax
mov  rax, rbx    ; (2) Write rax ← WAR กับ (1)
                 ; (2) ต้องไม่ write rax ก่อน (1) อ่านเสร็จ!
```

```
In-order pipeline: (2) ต้องรอ (1) อ่าน rax
OoO + Register Renaming:
    (1) add rcx, p1      ; p1 = rax
    (2) mov p2, rbx      ; p2 = new rax (renamed)
    → ทำ parallel ได้เลย! ไม่มี conflict
```

### 7.3 WAW - Write After Write (Output Dependency)

WAW เป็น **false dependency** ที่ register renaming แก้ได้เช่นกัน:

```nasm
; WAW hazard example
mov  rax, rbx    ; (1) Write rax
; ... some instructions ...
mov  rax, rcx    ; (2) Write rax ← WAW กับ (1)
                 ; (2) ต้องไม่ commit ก่อน (1)
```

```
ด้วย Register Renaming:
    (1) mov p1, rbx   ; p1 = rax (เวอร์ชัน 1)
    (2) mov p2, rcx   ; p2 = rax (เวอร์ชัน 2, renamed)
    → Execute parallel ได้
    → Retire in-order: (1) ก่อน (2)
    → โค้ดหลัง (2) จะเห็น rax = p2 (correct)
```

### 7.4 สรุป Data Hazards

```
┌─────────────────────────────────────────────────────────┐
│  Hazard Type   │ Dependency │ False? │  Solution        │
├─────────────────────────────────────────────────────────┤
│  RAW           │ True       │  No    │  Forwarding,     │
│  (Read-After-  │            │        │  Instruction     │
│   Write)       │            │        │  Scheduling      │
├─────────────────────────────────────────────────────────┤
│  WAR           │ Anti       │  Yes   │  Register        │
│  (Write-After- │            │        │  Renaming        │
│   Read)        │            │        │                  │
├─────────────────────────────────────────────────────────┤
│  WAW           │ Output     │  Yes   │  Register        │
│  (Write-After- │            │        │  Renaming        │
│   Write)       │            │        │                  │
└─────────────────────────────────────────────────────────┘
```

---

## 8. Structural Hazards

### 8.1 Execution Unit Conflicts

เกิดเมื่อ instructions หลายตัวต้องการใช้ execution unit เดียวกันพร้อมกัน:

```nasm
; Skylake มีแค่ 1 divider unit
; 3 division operations นี้ต้องรันทีละตัว:
div  rbx     ; latency ~40 cycles, throughput ~40 cycles
div  rcx     ; ต้องรอตัวแรกเสร็จ
div  rdx     ; ต้องรอตัวที่สองเสร็จ
; รวม: ~120 cycles!
```

**ตัวอย่าง Execution Ports บน Intel Skylake:**

```
Port 0: ALU, MUL, DIV, shifts, branches, FP-ALU, FP-MUL, FP-DIV
Port 1: ALU, MUL, LEA, FP-ALU, FP-MUL
Port 2: Load + Store AGU
Port 3: Load + Store AGU
Port 4: Store data
Port 5: ALU, vector shuffle, branches
Port 6: ALU, shift, branches, JMP
Port 7: Store AGU

Total throughput: สูงสุด 4 μops/cycle (หรือมากกว่าด้วย fusion)
```

### 8.2 ตัวอย่าง Port Pressure

```nasm
; ปัญหา: too many operations on Port 0
; (MUL, DIV, SHIFT ล้วนใช้ Port 0)
imul rax, rbx    ; Port 0 or 1
imul rcx, rdx    ; Port 0 or 1
imul r8,  r9     ; Port 0 or 1
shl  r10, 3      ; Port 0 or 6
; → Port 0 เป็น bottleneck

; ทางแก้: ใช้ LEA แทน shift สำหรับ *2, *4, *8
lea  r10, [r10*8] ; ใช้ Port 1 หรือ 5 (AGU)
```

---

## 9. Control Hazards

### 9.1 Branch Misprediction

```nasm
; ปัญหา: branch prediction miss
cmp  rax, 0
je   .target         ; Predicted: not taken
; --- CPU เริ่ม fetch/decode/execute instructions ต่อไปนี้ ---
add  rbx, 1          ; ← ถ้า branch taken จริง instructions เหล่านี้
add  rcx, 2          ;   ต้อง flush ออก (pipeline squash!)
add  rdx, 3
.target:
; ค่าใช้จ่าย: ~15-20 cycles สำหรับ misprediction บน Skylake
```

```
Branch Prediction Hit:
Cycle:  1    2    3    4    5    6    7
Fetch: [br] [i2] [i3] [i4] ...
(branch resolves at cycle 4, prediction was correct)
→ ไม่มี penalty

Branch Prediction Miss:
Cycle:  1    2    3    4    5    6    7    8    ...  20
Fetch: [br] [i2] [i3] [i4] → misprediction detected
                             → flush i2,i3,i4 (wasted work!)
                             → fetch correct path [c1] [c2] [c3] ...
→ ~15-20 cycle penalty
```

### 9.2 Branch Prediction Techniques

```nasm
; 1. Static prediction: forward branches = not taken, backward = taken

; 2. ใช้ cmov เพื่อหลีกเลี่ยง branch
; BAD: branch version
cmp  rax, rbx
jg   .greater
mov  rax, rbx        ; rax = min(rax, rbx)
.greater:

; GOOD: branchless version (ถ้า branch unpredictable)
cmp  rax, rbx
cmovg rax, rbx       ; rax = min(rax, rbx) - no branch!

; 3. ใช้ conditional move สำหรับ selection
; BAD:
test rax, rax
jz   .zero
mov  rbx, 1
jmp  .done
.zero:
mov  rbx, 0
.done:

; GOOD:
test rax, rax
mov  rbx, 1
cmovz rbx, [zero_val]  ; หรือ xor/cmov pattern
```

### 9.3 วัดต้นทุน Branch Misprediction

```nasm
; ใช้ RDTSC เพื่อวัด
rdtsc                    ; Read Time-Stamp Counter
mov  [start_low], eax
mov  [start_high], edx

; --- code to measure ---
.loop:
    cmp  rax, threshold
    jg   .branch_taken   ; Unpredictable branch (random data)
    add  rbx, 1
    jmp  .continue
.branch_taken:
    add  rcx, 1
.continue:
    dec  rdx
    jnz  .loop

rdtsc
; คำนวณ cycles ที่ใช้
```

---

## 10. Pipeline Stalls

### 10.1 ประเภทของ Stalls

```
1. Data Hazard Stall (Load-Use Stall)
   ─────────────────────────────────
   mov  rax, [memory]    ; latency = L1: 4c, L2: 12c, L3: 40c, DRAM: 200c
   add  rbx, rax          ; ต้องรอ! → stall

2. Structural Hazard Stall
   ────────────────────────
   div  rax, rbx   ; latency 20-90 cycles
   div  rcx, rdx   ; ต้องรอ divider ว่าง

3. Control Hazard Stall (Branch Penalty)
   ───────────────────────────────────────
   je   .somewhere  ; misprediction → ~15-20 cycle flush

4. ROB/RS Full Stall
   ──────────────────
   (เมื่อ ROB หรือ Reservation Station เต็ม → frontend stall)
```

### 10.2 การนับ Stall Cycles

```nasm
; ใช้ Performance Counters (Linux perf):
; perf stat -e cycles,instructions,stalled-cycles-frontend,stalled-cycles-backend ./program

; Metrics:
; IPC = instructions / cycles           (target: >3 สำหรับ Skylake)
; CPI = cycles / instructions           (target: <0.33)
; Frontend stalls: instruction fetch stalls
; Backend stalls: execution unit stalls

; ตัวอย่าง perf output:
;  Performance counter stats for './my_program':
;         1,234,567 cycles
;         4,567,890 instructions     #    3.70 insn per cycle
;           123,456 stalled-cycles-frontend  #   10.00% frontend cycles idle
;            45,678 stalled-cycles-backend   #    3.70% backend cycles idle
```

### 10.3 วิเคราะห์ Stalls ด้วย Perf

```bash
# วัด CPU events
perf stat -e \
  cycles,\
  instructions,\
  cache-references,\
  cache-misses,\
  branch-misses,\
  cpu/event=0x0e,umask=0x01,name=uops_issued_any/,\
  cpu/event=0xa2,umask=0x01,name=resource_stalls_any/ \
  ./your_program

# Profile ด้วย sampling
perf record -e cycles:u ./your_program
perf report

# Annotate source
perf annotate
```

---

## 11. Forwarding / Bypassing

### 11.1 แนวคิด Forwarding

แทนที่จะรอ result เขียนลง register file ก่อน ค่าสามารถ "forward" โดยตรงจาก output ของ execution stage ไปยัง input ของ instruction ถัดไป:

```
Without Forwarding:
Cycle:  1    2    3    4    5    6    7
ADD:   [IF] [ID] [EX] [WB]
SUB:   [IF] [ID] [--] [--] [EX] [WB]
              ↑ stall 2 cycles รอ WB

With Forwarding:
Cycle:  1    2    3    4    5
ADD:   [IF] [ID] [EX] [WB]
SUB:   [IF] [ID] [EX] [WB]
              ↑ result forwarded from EX→EX
              (ไม่ต้องรอ WB!)
```

### 11.2 Forwarding Paths บน Intel

```
┌────────────────────────────────────────────────┐
│  Forwarding Network                            │
│                                                │
│  EX stage output → RS input (same cycle+1)    │
│  Load result → ALU input (1 cycle later)       │
│  (Load-use hazard: still 1 cycle penalty)      │
│                                                │
│  Fast path (0 extra cycles):                  │
│    ADD/SUB/AND/OR/XOR result → next ALU        │
│    CMP → CMOV (same cycle)                    │
│                                                │
│  Slower path (1 cycle penalty):               │
│    Load → ALU (load-use hazard)               │
└────────────────────────────────────────────────┘
```

### 11.3 Load-Use Hazard

```nasm
; Load-use hazard: 1 cycle penalty (แม้มี forwarding)
mov  rax, [rbx]      ; Load: latency = 4 cycles (L1 hit)
add  rcx, rax        ; ใช้ rax ทันที: 1 cycle stall (ไม่ได้รอ 4 cycles เพราะ forwarding)

; หลีกเลี่ยงโดย:
mov  rax, [rbx]      ; Load
add  rdx, r8         ; Independent instruction (ทำระหว่างรอ)
add  rcx, rax        ; ตอนนี้ rax พร้อมแล้ว (ไม่มี stall)
```

---

## 12. Instruction Scheduling

### 12.1 หลักการ

Instruction scheduling คือการเรียงลำดับ instructions ใหม่เพื่อ:
1. ลด stalls จาก data hazards
2. ใช้ execution units ให้มีประสิทธิภาพสูงสุด
3. ลด port pressure

### 12.2 ตัวอย่าง Scheduling

```nasm
; BAD scheduling - dependency chains ทำให้ช้า
mov  rax, [array]        ; Load 1 (latency 4)
add  rax, 10             ; ต้องรอ load 1 (stall 3 cycles)
mov  [result1], rax      ; Store

mov  rbx, [array+8]      ; Load 2 (latency 4)
add  rbx, 20             ; ต้องรอ load 2 (stall 3 cycles)
mov  [result2], rbx      ; Store

; GOOD scheduling - interleave loads
mov  rax, [array]        ; Load 1
mov  rbx, [array+8]      ; Load 2 (ทำพร้อมกัน!)
add  rax, 10             ; ตอนนี้ rax พร้อมแล้ว (หรือเกือบพร้อม)
add  rbx, 20             ; และ rbx ด้วย
mov  [result1], rax      ; Store 1
mov  [result2], rbx      ; Store 2
```

### 12.3 ตัวอย่างซับซ้อน: Loop Scheduling

```nasm
; Original loop (unoptimized)
; for (int i = 0; i < n; i++) result[i] = a[i] + b[i];

.loop_bad:
    mov  eax, [rsi + rcx*4]    ; Load a[i]
    add  eax, [rdi + rcx*4]    ; Load b[i] + add (load-use hazard!)
    mov  [rdx + rcx*4], eax    ; Store result[i]
    inc  rcx
    cmp  rcx, r8
    jl   .loop_bad

; Optimized loop (unrolled x2 with better scheduling)
.loop_good:
    mov  eax, [rsi + rcx*4]       ; Load a[i]
    mov  ebx, [rsi + rcx*4 + 4]   ; Load a[i+1]
    add  eax, [rdi + rcx*4]       ; Load b[i] + add (rax ready by now)
    add  ebx, [rdi + rcx*4 + 4]   ; Load b[i+1] + add
    mov  [rdx + rcx*4], eax       ; Store result[i]
    mov  [rdx + rcx*4 + 4], ebx   ; Store result[i+1]
    add  rcx, 2
    cmp  rcx, r8
    jl   .loop_good
```

---

## 13. Latency vs Throughput

### 13.1 ความแตกต่าง

```
Latency:    เวลาที่ใช้ตั้งแต่เริ่ม instruction จนได้ผลลัพธ์
            (วัดเป็น cycles)
            
Throughput: จำนวน instruction ที่สามารถเริ่มต่อ cycle
            (มักแสดงเป็น reciprocal throughput = cycles/instruction)
```

### 13.2 ตารางสำคัญ (Intel Skylake)

```
┌──────────────────────────────────────────────────────────────┐
│  Instruction    │ Latency │ Recip. TP │ Ports               │
├──────────────────────────────────────────────────────────────┤
│  ADD/SUB/AND    │   1     │   0.25    │ p0156               │
│  OR/XOR/NOT     │   1     │   0.25    │ p0156               │
│  MOV r,r        │   0*    │   0.25    │ p0156               │
│  IMUL r,r       │   3     │   1       │ p1                  │
│  IMUL r,r,imm   │   3     │   1       │ p1                  │
│  LEA (simple)   │   1     │   0.5     │ p15                 │
│  LEA (complex)  │   3     │   1       │ p1                  │
├──────────────────────────────────────────────────────────────┤
│  SHL/SHR (imm)  │   1     │   0.5     │ p06                 │
│  SHL/SHR (CL)   │   3     │   1       │ p06                 │
├──────────────────────────────────────────────────────────────┤
│  DIV r64        │  35-88  │  21-74    │ p0                  │
│  IDIV r64       │  35-88  │  21-74    │ p0                  │
├──────────────────────────────────────────────────────────────┤
│  LOAD (L1 hit)  │   4     │   0.5     │ p23                 │
│  LOAD (L2 hit)  │  12     │   -       │ -                   │
│  LOAD (L3 hit)  │  42     │   -       │ -                   │
│  LOAD (DRAM)    │~200     │   -       │ -                   │
│  STORE          │   -     │   1       │ p4+p237             │
├──────────────────────────────────────────────────────────────┤
│  JMP (direct)   │   0*    │   1       │ p06                 │
│  Jcc (taken)    │   0*    │   1       │ p06                 │
│  CALL/RET       │   0*    │   3/3     │ p0156+stack         │
└──────────────────────────────────────────────────────────────┘
* Zero latency = result available immediately (MOV elimination, etc.)
```

### 13.3 ตัวอย่างการใช้ Throughput vs Latency

```nasm
; Case 1: Serial chain - bottleneck คือ LATENCY
; ทุก instruction ต้อง depend กับตัวก่อน
imul rax, rax       ; latency 3
imul rax, rax       ; latency 3 (ต้องรอ)
imul rax, rax       ; latency 3 (ต้องรอ)
imul rax, rax       ; latency 3 (ต้องรอ)
; รวม: 4 × 3 = 12 cycles (ไม่ว่า throughput จะเป็นเท่าไร)

; Case 2: Independent chain - bottleneck คือ THROUGHPUT
; IMUL throughput = 1 instruction/cycle
imul rax, rbx       ; starts cycle 1
imul rcx, rdx       ; starts cycle 2 (independent!)
imul r8,  r9        ; starts cycle 3 (independent!)
imul r10, r11       ; starts cycle 4 (independent!)
; รวม: 4 cycles (ทำแบบ pipeline!)
; (แต่ต้องรอ ~3 cycles ก่อนใช้ผลลัพธ์แต่ละตัว)
```

---

## 14. Dependency Chain Analysis

### 14.1 Serial Dependency Chain

```nasm
; Serial chain - performance จำกัดโดย latency
; ตัวอย่าง: หา sum ของ array
xor  rax, rax           ; sum = 0
mov  rcx, n             ; counter = n
mov  rsi, array         ; pointer = array
.loop:
    add  rax, [rsi]     ; sum += *ptr (RAW dependency กับ add ก่อนหน้า!)
    add  rsi, 8         ; ptr++ (independent)
    dec  rcx
    jnz  .loop

; แต่ละ iteration ต้องรอ rax จาก iteration ก่อน
; Throughput จำกัด: 1 iteration / 4 cycles (latency ของ load+add)
; แม้ loop body จะมี instructions น้อย
```

### 14.2 Parallel Dependency Chain (Unrolling)

```nasm
; Parallel chains - ดีกว่ามาก
xor  rax, rax           ; sum0 = 0
xor  rbx, rbx           ; sum1 = 0
xor  rcx, rcx           ; sum2 = 0
xor  rdx, rdx           ; sum3 = 0
mov  r8,  n             ; counter
mov  rsi, array         ; pointer
.loop:
    add  rax, [rsi]        ; sum0 += ptr[0]
    add  rbx, [rsi + 8]    ; sum1 += ptr[1] (independent chain!)
    add  rcx, [rsi + 16]   ; sum2 += ptr[2] (independent chain!)
    add  rdx, [rsi + 24]   ; sum3 += ptr[3] (independent chain!)
    add  rsi, 32           ; ptr += 4
    sub  r8,  4
    jnz  .loop

; รวม chains หลัง loop
add  rax, rbx
add  rax, rcx
add  rax, rdx

; 4 chains ทำงาน parallel → throughput สูงขึ้น ~4x
; Bottleneck: load throughput (2 loads/cycle) → 2 cycles/iteration
;             (ประสิทธิภาพ ~2x ดีกว่า serial)
```

### 14.3 Vectorized Sum (ดีที่สุด)

```nasm
; ใช้ AVX2 SIMD - 8 integers ต่อ instruction
vpxor    ymm0, ymm0, ymm0    ; sum_vec = 0
vpxor    ymm1, ymm1, ymm1    ; sum_vec2 = 0
mov      rsi, array
mov      rcx, n
shr      rcx, 4              ; n/16 iterations

.loop:
    vpaddd   ymm0, ymm0, [rsi]       ; sum_vec += ptr[0..7]
    vpaddd   ymm1, ymm1, [rsi + 32]  ; sum_vec2 += ptr[8..15]
    add      rsi, 64
    dec      rcx
    jnz      .loop

; Horizontal sum
vpaddd   ymm0, ymm0, ymm1            ; รวม 2 vectors
vextracti128 xmm1, ymm0, 1           ; extract upper 128 bits
vpaddd   xmm0, xmm0, xmm1           ; horizontal add 1
vpshufd  xmm1, xmm0, 0b00001110     ; shuffle
vpaddd   xmm0, xmm0, xmm1           ; horizontal add 2
vpshufd  xmm1, xmm0, 0b00000001     ; shuffle
vpaddd   xmm0, xmm0, xmm1           ; horizontal add 3
vmovd    eax, xmm0                   ; extract result
```

---

## 15. Intel Microarchitecture Details

### 15.1 Skylake Microarchitecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Intel Skylake Frontend                        │
│                                                                  │
│  L1 I-Cache (32KB, 8-way)                                       │
│     ↓ (16 bytes/cycle)                                          │
│  Instruction Queue                                               │
│     ↓                                                           │
│  Pre-Decode (6 instructions/cycle)                               │
│     ↓                                                           │
│  Instruction Queue                                               │
│     ↓                                                           │
│  Decode (4 μops/cycle)    ←→  μop Cache (DSB, 1536 μops)       │
│     ↓                                                           │
│  Loop Stream Detector (LSD, up to 25μops)                       │
│     ↓                                                           │
│  μop Queue (28 μops, IDQ)                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Intel Skylake Backend                         │
│                                                                  │
│  Rename/Allocate (4 μops/cycle)                                 │
│     ↓ → Register File (180 int + 168 FP physical registers)     │
│     ↓ → ROB (224 entries)                                       │
│     ↓ → Reservation Station (97 entries)                        │
│     ↓                                                           │
│  Issue (8 ports × 1-2 μops/cycle)                               │
│  ┌────────────────────────────────────────┐                     │
│  │ Port 0: ALU+MUL+DIV+SHIFT+VEC+FP      │                     │
│  │ Port 1: ALU+MUL+LEA+VEC+FP            │                     │
│  │ Port 2: Load+StoreAGU                 │                     │
│  │ Port 3: Load+StoreAGU                 │                     │
│  │ Port 4: Store Data                    │                     │
│  │ Port 5: ALU+VEC+Shuffle+Branch        │                     │
│  │ Port 6: ALU+SHIFT+Branch              │                     │
│  │ Port 7: StoreAGU                      │                     │
│  └────────────────────────────────────────┘                     │
│     ↓                                                           │
│  Retire (4 μops/cycle)                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 15.2 Alder Lake (12th Gen) Differences

```
Alder Lake ใช้ hybrid architecture:
┌──────────────────────────────────────────────────────────┐
│  P-Core (Performance Core) - Golden Cove                 │
│    ROB: 512 entries (มากกว่า Skylake 2x!)               │
│    RS: 168 entries                                       │
│    Width: 6-wide decode, 12-wide execution              │
│    Execution ports: 12 ports                             │
│                                                          │
│  E-Core (Efficiency Core) - Gracemont                   │
│    Simpler pipeline, 4 wide                             │
│    ใช้พลังงานน้อยกว่า                                   │
│    Intel Thread Director จัดการว่า task ไหนไปที่ไหน     │
└──────────────────────────────────────────────────────────┘
```

---

## 16. Agner Fog Tables

### 16.1 แหล่งข้อมูล

Agner Fog เป็นผู้รวบรวมข้อมูล microarchitecture ที่ละเอียดที่สุด:
- https://www.agner.org/optimize/
- ไฟล์: instruction_tables.pdf, microarchitecture.pdf, optimizing_assembly.pdf

### 16.2 วิธีอ่านตาราง

```
Format ในตาราง Agner Fog:
Instruction | Operands | Latency | Recip.Throughput | μops | Execution unit

ตัวอย่าง (Skylake):
ADD         | r,r       |    1    |      0.25        |  1   | p0156
IMUL        | r64,r64   |    3    |      1           |  1   | p1
DIV         | r64       |  35-88  |     21-74        | 35-88| p0
MOVAPS      | x,x       |    0    |      0.33        |  1   | p0156
ADDPS       | x,x       |    4    |      0.5         |  1   | p01
```

### 16.3 ตัวอย่างการใช้ตาราง

```nasm
; เราต้องการ: a * b + c * d (สำหรับ scalar int)

; Version 1: Sequential
imul rax, rbx       ; 3 cycles latency
imul rcx, rdx       ; 3 cycles latency (ต้องรอ?)
add  rax, rcx       ; 1 cycle

; ตรวจสอบ dependencies:
; - imul1 กับ imul2: ไม่มี dependency → รันพร้อมกัน!
; - add ต้องรอ imul1 และ imul2

; Critical path: max(3, 3) + 1 = 4 cycles
; (ทั้งสอง imul รันพร้อมกัน, จากนั้น add)

; Version 2: FMA (ถ้า operands เป็น float)
; vfmadd213ss xmm0, xmm1, xmm2  ; xmm0 = xmm1*xmm0 + xmm2
; Latency: 4, Throughput: 0.5
; = 1 instruction แทน 2!
```

---

## 17. IACA - Intel Architecture Code Analyzer

### 17.1 ภาพรวม

IACA (Intel Architecture Code Analyzer) วิเคราะห์ static throughput ของ code region:

```nasm
; ใช้ markers ใน code:
%include "iacaMarks.h"  ; หรือใส่ magic bytes โดยตรง

IACA_START
; --- hot loop to analyze ---
.loop:
    movdqu  xmm0, [rsi]
    movdqu  xmm1, [rdi]
    paddd   xmm0, xmm1
    movdqu  [rdx], xmm0
    add     rsi, 16
    add     rdi, 16
    add     rdx, 16
    dec     rcx
    jnz     .loop
IACA_END

; Run: iaca -arch SKL myprogram
```

### 17.2 ตัวอย่าง IACA Output

```
Throughput Analysis Report
--------------------------
Block Throughput: 2.00 Cycles/Iteration    Throughput Bottleneck: Ports {2 3}

Port Binding In Cycles Per Iteration:
-------------------------------------------------
|  Port  |  0   -  DV  |  1   |  2   |  3   |  4   |  5   |  6   |  7   |
-------------------------------------------------
| Cycles | 0.0    0.0   | 1.0  | 2.0  | 2.0  | 2.0  | 0.0  | 1.0  | 0.0  |
-------------------------------------------------

N - port number or number of cycles resource used
DV - Divider pipe (n)
D - Data fetch pipe (load, i-cache)
F - Macro Fusion with the previous instruction occurred
* - instruction micro-ops not bound to a port
^ - Micro Fusion occurred
# - ESP Tracking push/pop not supported, estimated key register use
@ - SSE instruction followed an AVX256/AVX512 instruction, followed by SSE
! - instruction not supported, was not accounted in Analysis
```

---

## 18. LLVM-MCA

### 18.1 ภาพรวม

llvm-mca (LLVM Machine Code Analyzer) เป็น alternative ที่ free และ open source:

```bash
# ติดตั้ง
apt-get install llvm  # Ubuntu/Debian

# วิเคราะห์ assembly file
llvm-mca -mcpu=skylake -iterations=100 mycode.s

# หรือผ่าน stdin
echo "
.loop:
  imulq %rbx, %rax
  imulq %rcx, %rdx
  addq  %rax, %rdx
  decq  %rsi
  jnz .loop
" | llvm-mca -mcpu=skylake
```

### 18.2 ตัวอย่าง llvm-mca Output

```
Iterations:        100
Instructions:      500
Total Cycles:      303
Total uOps:        500

Dispatch Width:    6
uOps Per Cycle:    1.65
IPC:               1.65
Block RThroughput: 3.0

Instruction Info:
[1]: #uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 1      3     1.00                        imulq   %rbx, %rax
 1      3     1.00                        imulq   %rcx, %rdx
 1      1     0.25                        addq    %rax, %rdx
 1      1     0.25                        decq    %rsi
 1      1     1.00                        jnz     .loop


Resources:
[0]   - SKLDivider
[1]   - SKLFPDivider
[2]   - SKLPort0
[3]   - SKLPort1
[4]   - SKLPort2
[5]   - SKLPort3
[6]   - SKLPort4
[7]   - SKLPort5
[8]   - SKLPort6
[9]   - SKLPort7

Resource pressure per iteration:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]
 -      -     0.99   2.01   -      -      -      -      1.00   -

Resource pressure by instruction:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]    Instructions:
 -      -      -     1.00   -      -      -      -      -      -     imulq   %rbx, %rax
 -      -      -     1.00   -      -      -      -      -      -     imulq   %rcx, %rdx
 -      -     0.33   0.01   -      -      -      0.33   0.33   -     addq    %rax, %rdx
 -      -     0.33   0.01   -      -      -      0.33   0.33   -     decq    %rsi
 -      -     0.33    -     -      -      -      -      0.34   -     jnz     .loop
```

### 18.3 ใช้ llvm-mca กับ C Code

```bash
# Compile C to assembly แล้ว analyze
gcc -O2 -march=skylake -S mycode.c -o mycode.s
llvm-mca -mcpu=skylake mycode.s

# ใส่ markers ใน C code
void myfunction() {
    __asm volatile("# LLVM-MCA-BEGIN myloop");
    // hot loop here
    for (int i = 0; i < n; i++) {
        result[i] = a[i] + b[i];
    }
    __asm volatile("# LLVM-MCA-END myloop");
}
```

---

## 19. Code Examples: Bad vs Good Ordering

### 19.1 Example 1: Memory Dependency Chain

```nasm
; ============================================================
; BAD: Load-use hazard chain
; ============================================================
section .text
bad_memory_chain:
    ; Each load feeds directly into the next computation
    mov  rax, [rdi]          ; Load a[0] - 4 cycle latency
    add  rax, [rdi + 8]      ; Load a[1], add to rax (stall!)
    add  rax, [rdi + 16]     ; Load a[2], add (stall!)
    add  rax, [rdi + 24]     ; Load a[3], add (stall!)
    ; Total: ~16+ cycles due to serial loads
    ret

; ============================================================
; GOOD: Overlapping loads
; ============================================================
good_memory_chain:
    ; Issue all loads first, then add
    mov  rax, [rdi]          ; Load a[0]
    mov  rbx, [rdi + 8]      ; Load a[1] (parallel with a[0]!)
    mov  rcx, [rdi + 16]     ; Load a[2] (parallel!)
    mov  rdx, [rdi + 24]     ; Load a[3] (parallel!)
    add  rax, rbx            ; Now compute (all loads hit L1)
    add  rax, rcx
    add  rax, rdx
    ; Total: ~4-5 cycles (all loads done before first add)
    ret
```

### 19.2 Example 2: Division Avoidance

```nasm
; ============================================================
; BAD: Using DIV (very slow)
; ============================================================
section .text
bad_division:
    ; Check if x is divisible by powers of 2
    mov  rax, rdi            ; x
    mov  rbx, 2
    xor  rdx, rdx
    div  rbx                 ; x / 2, latency ~35-88 cycles!
    test rdx, rdx
    jz   .divisible_by_2
    
    mov  rax, rdi
    mov  rbx, 4
    xor  rdx, rdx
    div  rbx                 ; x / 4, another ~35-88 cycles!
    ; ...
    ret

; ============================================================
; GOOD: Using bit operations (much faster)
; ============================================================
good_division:
    ; Check divisibility by powers of 2 using AND
    test rdi, 1              ; x & 1 == 0 → divisible by 2
    jz   .divisible_by_2
    
    test rdi, 3              ; x & 3 == 0 → divisible by 4
    jz   .divisible_by_4
    
    test rdi, 7              ; x & 7 == 0 → divisible by 8
    jz   .divisible_by_8
    
    ; AND latency: 1 cycle vs DIV: 35-88 cycles!
    ret

; ============================================================
; GOOD: Replace division by constant with multiplication
; ============================================================
; x / 7  →  x * (1/7 as fixed point)
; Compiler does this automatically, but useful to know

divide_by_7:
    ; Method: multiply by magic number, then shift
    ; This is what compilers generate for x / 7:
    mov  rax, rdi
    mov  rcx, 2454267027     ; magic = ceil(2^33 / 7)
    imul rax, rcx            ; 3 cycle latency
    sar  rax, 33             ; arithmetic shift right 33
    ; Result in rax = rdi / 7
    ; Total: ~4 cycles vs ~35-88 for DIV
    ret
```

### 19.3 Example 3: Branch vs. Branchless

```nasm
; ============================================================
; BAD: Unpredictable branch (random comparison)
; ============================================================
section .data
random_array: times 1000 dd 0   ; filled with random values

section .text
bad_branch_version:
    ; Count elements greater than threshold
    xor  ecx, ecx            ; count = 0
    xor  eax, eax            ; i = 0
    mov  r8d, threshold
.loop_bad:
    cmp  eax, 1000
    jge  .done_bad
    mov  edx, [rdi + rax*4]
    cmp  edx, r8d
    jle  .skip_bad           ; Unpredictable! ~50% misprediction
    inc  ecx                 ; count++
.skip_bad:
    inc  eax
    jmp  .loop_bad
.done_bad:
    mov  eax, ecx
    ret

; ============================================================
; GOOD: Branchless (for unpredictable data)
; ============================================================
good_branchless_version:
    xor  ecx, ecx            ; count = 0
    xor  eax, eax
    mov  r8d, threshold
.loop_good:
    cmp  eax, 1000
    jge  .done_good
    mov  edx, [rdi + rax*4]
    cmp  edx, r8d
    setg dl                  ; dl = 1 if greater, 0 if not
    movzx edx, dl
    add  ecx, edx            ; count += (x > threshold) ? 1 : 0
    inc  eax
    jmp  .loop_good          ; This branch IS predictable (always taken)
.done_good:
    mov  eax, ecx
    ret

; ============================================================
; BEST: Vectorized branchless
; ============================================================
best_vectorized_version:
    xor       ecx, ecx               ; count = 0
    vpbroadcastd ymm1, [threshold_ptr]  ; broadcast threshold to all 8 lanes
    vpxor     ymm2, ymm2, ymm2       ; zero accumulator
.loop_best:
    vmovdqu   ymm0, [rdi + rax*4]   ; load 8 ints
    vpcmpgtd  ymm3, ymm0, ymm1      ; compare: ff if greater, 00 if not
    vpsubd    ymm2, ymm2, ymm3      ; subtract (add -1 or 0 per lane)
    add       rax, 8
    cmp       rax, 1000
    jl        .loop_best
    
    ; Horizontal sum of ymm2
    vextracti128 xmm3, ymm2, 1
    vpaddd    xmm2, xmm2, xmm3
    vpshufd   xmm3, xmm2, 0b00001110
    vpaddd    xmm2, xmm2, xmm3
    vpshufd   xmm3, xmm2, 0b00000001
    vpaddd    xmm2, xmm2, xmm3
    vmovd     eax, xmm2
    ret
```

### 19.4 Example 4: Port Pressure Optimization

```nasm
; ============================================================
; BAD: Too many Port 1 operations (IMUL bottleneck)
; ============================================================
section .text
bad_port_pressure:
    ; 4 IMULs all compete for Port 1
    imul rax, rbx            ; Port 1, latency 3
    imul rax, rcx            ; Port 1, latency 3 (serial dependency!)
    imul rax, rdx            ; Port 1, latency 3 (serial dependency!)
    imul rax, r8             ; Port 1, latency 3 (serial dependency!)
    ; Total: 4 × 3 = 12 cycles (latency-bound)
    ret

; ============================================================
; GOOD: Break dependency chain (if mathematically equivalent)
; ============================================================
good_port_pressure:
    ; Compute (a*b) * (c*d) instead of ((a*b)*c)*d
    ; ถ้า overflow ไม่ใช่ปัญหา:
    imul rax, rbx            ; p1: a*b → rax (chain 1)
    imul rcx, rdx            ; p1: c*d → rcx (chain 2, independent!)
    imul rax, rcx            ; p1: (a*b) * (c*d) → rax
    ; Total: max(3,3) + 3 = 6 cycles (ดีกว่า 2x!)
    ret

; ============================================================
; BAD: Mixing shifts and multiply without balancing
; ============================================================
bad_mixed_ops:
    ; All these use Port 0 or 6
    shl  rax, 3              ; Port 0/6
    sar  rbx, 2              ; Port 0/6  
    shl  rcx, 4              ; Port 0/6
    sar  rdx, 1              ; Port 0/6
    shl  r8,  5              ; Port 0/6
    ; → Port 0/6 saturated at 2 ops/cycle
    ; But Port 1,5 idle!

; ============================================================
; GOOD: Use LEA to balance ports
; ============================================================
good_mixed_ops:
    ; LEA for multiply-by-constant uses Port 1 or 5
    lea  rax, [rax*8]        ; Port 1/5 (multiply by 8)
    sar  rbx, 2              ; Port 0/6 (shift)
    lea  rcx, [rcx*16]       ; Port 1/5 (multiply by 16)
    sar  rdx, 1              ; Port 0/6 (shift)
    lea  r8,  [r8*32]        ; Port 1/5 (multiply by 32)
    ; → Load balanced across ports!
```

---

## 20. Practical Analysis Workflow

### 20.1 ขั้นตอนการ Optimize Assembly

```
1. Profile ก่อน optimize
   ──────────────────────
   perf record ./program
   perf report
   → ระบุ hot functions/loops

2. วิเคราะห์ด้วย llvm-mca
   ─────────────────────────
   llvm-mca -mcpu=<your_cpu> hotloop.s
   → ดู bottleneck: ports, latency, throughput

3. ตรวจสอบ Agner Fog tables
   ─────────────────────────
   → หา latency/throughput ของแต่ละ instruction
   → ระบุ critical path

4. แก้ไข
   ──────
   - ถ้า latency bound: unroll, software pipelining
   - ถ้า throughput bound: reorder, use different instructions
   - ถ้า memory bound: prefetch, improve cache locality

5. Verify ด้วย perf stat
   ──────────────────────
   perf stat -e cycles,instructions,cache-misses ./program
```

### 20.2 ตัวอย่าง Full Analysis

```nasm
; ============================================================
; Hot loop to optimize: dot product of two float arrays
; ============================================================

; Version 1: Naive scalar
dot_product_v1:
    xorps   xmm0, xmm0      ; sum = 0
    xor     eax,  eax       ; i = 0
.loop:
    movss   xmm1, [rdi + rax*4]    ; load a[i]
    mulss   xmm1, [rsi + rax*4]    ; a[i] * b[i] (load+mul)
    addss   xmm0, xmm1              ; sum += product
    inc     eax
    cmp     eax, edx
    jl      .loop
    ret
    
; llvm-mca analysis:
; Bottleneck: addss latency (4 cycles) in serial chain
; Throughput: ~4 cycles/iteration (latency bound)

; ============================================================
; Version 2: Multiple accumulators (break latency chain)
; ============================================================
dot_product_v2:
    vxorps  ymm0, ymm0, ymm0    ; sum0 = 0
    vxorps  ymm1, ymm1, ymm1    ; sum1 = 0
    xor     eax,  eax
    sub     edx, 15             ; loop unroll count
    jle     .cleanup
.loop:
    ; Process 16 floats per iteration (2 × 8-float AVX)
    vmovups ymm2, [rdi + rax*4]       ; load a[0..7]
    vmovups ymm3, [rdi + rax*4 + 32]  ; load a[8..15]
    vfmadd231ps ymm0, ymm2, [rsi + rax*4]       ; sum0 += a[0..7] * b[0..7]
    vfmadd231ps ymm1, ymm3, [rsi + rax*4 + 32]  ; sum1 += a[8..15] * b[8..15]
    add     eax, 16
    sub     edx, 16
    jg      .loop
    
    ; Combine accumulators
    vaddps  ymm0, ymm0, ymm1
    
    ; Horizontal sum
    vextractf128 xmm1, ymm0, 1
    vaddps  xmm0, xmm0, xmm1
    vhaddps xmm0, xmm0, xmm0
    vhaddps xmm0, xmm0, xmm0
    vmovss  [result], xmm0
    ; ... cleanup for remaining elements
    ret
    
; llvm-mca analysis:
; Bottleneck: Load ports (2 loads/cycle for 2 ymm loads/iter = 1 iter/cycle)
; Throughput: ~1 cycle/16 floats = much better!
```

---

## 21. Memory Access Patterns และ Pipeline

### 21.1 Cache-Friendly Access

```nasm
; ============================================================
; BAD: Column-major access of row-major matrix (cache unfriendly)
; ============================================================
; Matrix stored row-major: M[row][col] = M[row * COLS + col]
bad_column_access:
    ; Access column 0: M[0][0], M[1][0], M[2][0], ...
    ; Each access jumps COLS * 4 bytes → cache line misses!
    xor  eax, eax           ; row = 0
.col_loop:
    mov  edx, [rdi + rax*4] ; M[row][0] = [base + row*COLS*4]
    ; ... process edx ...
    add  eax, COLS          ; next row
    cmp  eax, ROWS*COLS
    jl   .col_loop

; ============================================================
; GOOD: Row-major access (cache friendly)
; ============================================================
good_row_access:
    ; Access row 0: M[0][0], M[0][1], M[0][2], ...
    ; Sequential access → excellent cache behavior
    xor  eax, eax           ; index = 0
.row_loop:
    mov  edx, [rdi + rax*4] ; M[row][col] = sequential!
    ; ... process edx ...
    inc  eax
    cmp  eax, ROWS*COLS
    jl   .row_loop
```

### 21.2 Software Prefetching

```nasm
; ============================================================
; Prefetching to hide memory latency
; ============================================================
; prefetcht0: prefetch to L1 and higher
; prefetcht1: prefetch to L2 and higher
; prefetcht2: prefetch to L3 and higher
; prefetchnta: non-temporal (bypass cache)

prefetch_example:
    mov  rax, rdi           ; pointer to array
    mov  rcx, n             ; count
    
    ; Prefetch distance: typically 64-256 bytes ahead
    ; (depends on memory bandwidth and loop latency)
    PREFETCH_DISTANCE equ 256
    
.loop:
    prefetcht0 [rax + PREFETCH_DISTANCE]    ; Prefetch ahead
    
    ; Process current element
    movdqu  xmm0, [rax]
    ; ... computation ...
    movdqu  [rax], xmm0
    
    add  rax, 16
    sub  rcx, 4
    jnz  .loop
    ret

; Note: prefetch ไม่ใช่ guarantee - เป็น hint เท่านั้น
; ต้อง tune ระยะ prefetch ตาม target platform
```

---

## 22. Advanced Topics

### 22.1 Store Forwarding

```nasm
; Store forwarding: เมื่อ load อ่าน address ที่ store เพิ่งเขียน
; CPU สามารถ forward data จาก store buffer ได้โดยตรง

; GOOD: exact size match → store forwarding เร็ว (4-5 cycles)
mov  [rsp - 8], rax     ; store 64-bit
mov  rbx, [rsp - 8]     ; load 64-bit ← forwarding!

; BAD: size mismatch → store forwarding violation (~13 cycles penalty!)
mov  [rsp - 8], rax     ; store 64-bit
mov  bl,  [rsp - 8]     ; load 8-bit ← violation! processor stalls

; BAD: partial overlap → forwarding violation
mov  [rsp - 8], rax     ; store at offset 0
mov  rbx, [rsp - 4]     ; load at offset 4 → partial overlap = violation
```

### 22.2 Micro-op Cache (DSB - Decoded Stream Buffer)

```nasm
; Intel Skylake DSB: 1536 μops, organized as 6-way cache
; 32 sets × 6 ways × 8 μops per way = 1536 μops

; Advantages:
; - DSB feeds 6 μops/cycle (vs 4 from legacy decoder)
; - Lower power consumption
; - Better for dense instruction mixes

; Instructions that don't fit in DSB well:
; - Instructions > 7 bytes (can span cache line boundaries)
; - Very complex instructions (many μops)

; To maximize DSB usage:
; - Keep loops small (< 1536 μops)
; - Align critical loops to 32-byte boundaries
section .text
align 32            ; Align to 32-byte boundary for DSB efficiency
.hot_loop:
    ; ... loop body ...
    dec  rcx
    jnz  .hot_loop
```

### 22.3 LSD - Loop Stream Detector

```nasm
; LSD (Loop Stream Detector) จำ μops ของ loop ขนาดเล็ก
; Skylake LSD: สูงสุด 25 μops

; GOOD: small loop fits in LSD
.small_loop:                    ; 3 μops per iteration
    add  rax, [rsi + rcx*8]    ; μop 1
    dec  rcx                    ; μop 2 (fused with jnz → 1 μop)
    jnz  .small_loop            ; fused
    ; Total: 3 μops → fits in LSD perfectly

; Note: LSD disabled in some CPU revisions (Skylake errata SKZ24)
; ต้อง verify บน target CPU จริง
```

---

## 23. Benchmark Template

### 23.1 NASM Benchmark Framework

```nasm
; ============================================================
; Micro-benchmark template (NASM, Linux)
; ============================================================
section .data
    align   16
    cycles_start: dq 0
    cycles_end:   dq 0
    iterations:   equ 1000000

section .text
    global  main
    extern  printf

main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 64
    push    rbx
    push    r12
    push    r13
    push    r14
    push    r15
    
    ; Warm up CPU (avoid frequency scaling effects)
    mov     rcx, 10000
.warmup:
    dec     rcx
    jnz     .warmup
    
    ; Serialize before measurement
    xor     eax, eax
    cpuid                       ; Serializing instruction
    rdtsc                       ; Read timestamp
    mov     [cycles_start],   eax
    mov     [cycles_start+4], edx
    
    ; ==========================================
    ; === HOT CODE TO BENCHMARK HERE ===
    mov     rcx, iterations
.bench_loop:
    ; --- YOUR CODE HERE ---
    nop                         ; Replace with actual code
    ; ----------------------
    dec     rcx
    jnz     .bench_loop
    ; ==========================================
    
    ; Serialize after measurement
    xor     eax, eax
    cpuid
    rdtsc
    mov     [cycles_end],   eax
    mov     [cycles_end+4], edx
    
    ; Calculate cycles
    mov     rax, [cycles_end]
    sub     rax, [cycles_start]
    xor     rdx, rdx
    mov     rcx, iterations
    div     rcx                 ; cycles per iteration
    
    ; Print result
    mov     rdi, fmt_string
    mov     rsi, rax
    xor     eax, eax
    call    printf
    
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbx
    xor     eax, eax
    leave
    ret

section .data
    fmt_string: db "Cycles per iteration: %lu", 10, 0
```

### 23.2 GAS Syntax Benchmark (AT&T Syntax)

```gas
# ============================================================
# Micro-benchmark template (GAS/AT&T syntax)
# ============================================================
    .section .data
    .align 16
cycles_start:   .quad 0
cycles_end:     .quad 0
fmt_string:     .string "Cycles per iteration: %lu\n"

    .text
    .globl main
main:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $64, %rsp
    
    # Warm up
    movl    $10000, %ecx
.warmup:
    decl    %ecx
    jnz     .warmup
    
    # Start measurement
    xorl    %eax, %eax
    cpuid
    rdtsc
    movl    %eax, cycles_start(%rip)
    movl    %edx, cycles_start+4(%rip)
    
    movl    $1000000, %ecx
.bench_loop:
    # --- YOUR CODE HERE (AT&T syntax) ---
    nop
    # ------------------------------------
    decl    %ecx
    jnz     .bench_loop
    
    # End measurement
    xorl    %eax, %eax
    cpuid
    rdtsc
    movl    %eax, cycles_end(%rip)
    movl    %edx, cycles_end+4(%rip)
    
    # Calculate and print
    movq    cycles_end(%rip), %rax
    subq    cycles_start(%rip), %rax
    xorl    %edx, %edx
    movl    $1000000, %ecx
    divq    %rcx
    
    movq    $fmt_string, %rdi
    movq    %rax, %rsi
    xorl    %eax, %eax
    call    printf
    
    xorl    %eax, %eax
    leave
    ret
```

---

## 24. สรุปหลักการ Optimization

### 24.1 Critical Rules

```
Rule 1: วัดก่อน optimize
────────────────────────
ใช้ perf, llvm-mca, IACA เพื่อระบุ bottleneck จริง
อย่าเดา!

Rule 2: ระบุ bottleneck
────────────────────────
- Latency bound: ต้อง break dependency chains
- Throughput bound: ต้องสมดุล port usage
- Memory bound: ต้องปรับปรุง cache locality / prefetch
- Branch bound: ต้องใช้ branchless หรือแก้ prediction

Rule 3: การ Unroll
───────────────────
- Unroll 2-8x เพื่อ expose instruction-level parallelism
- ระวัง: unroll มากเกินไปอาจ evict μop cache (DSB)
- ใช้ หลาย accumulators เพื่อ break latency chains

Rule 4: SIMD/Vectorization
────────────────────────────
- SSE2: 2 doubles หรือ 4 floats ต่อ register
- AVX2: 4 doubles หรือ 8 floats ต่อ register
- AVX-512: 8 doubles หรือ 16 floats ต่อ register
- Vectorization = หลาย× speedup

Rule 5: Memory Hierarchy
─────────────────────────
L1 cache: 4 cycles, 32KB, พยายามทำ working set ให้ fit
L2 cache: 12 cycles, 256KB
L3 cache: 40 cycles, หลาย MB
DRAM: ~200 cycles
→ Cache miss = disaster สำหรับ performance
```

### 24.2 Instruction Selection Quick Reference

```
แทน:              ใช้:
────────────────────────────────────────────────
x / 2^n           x >> n  (sar/shr)
x * 2^n           x << n  (shl) หรือ lea [x*2^n]
x * 3             lea [x + x*2]
x * 5             lea [x + x*4]
x * 9             lea [x + x*8]
x * 10            lea [x + x*4]; add; shl 1 หรือ imul
x / 7 (const)     compiler magic: imul + sar
if-else (random)  cmov, setcc + and/or
branch + inc      setcc + add

ใช้ XMM/YMM:      แทน scalar FP operations
FMA:              แทน separate mul+add
SIMD compare:     แทน scalar branch loops
```

---

## 25. Real-World Example: Optimized Memcpy

```nasm
; ============================================================
; Optimized memcpy using SIMD and pipeline techniques
; ============================================================
section .text
global fast_memcpy
; fast_memcpy(void *dst, void *src, size_t n)
; rdi = dst, rsi = src, rdx = n
fast_memcpy:
    push    rbp
    mov     rbp, rsp
    
    ; Handle small copies
    cmp     rdx, 32
    jl      .small_copy
    
    ; Handle alignment
    ; (simplified - production code would align first)
    
    cmp     rdx, 256
    jl      .medium_copy
    
    ; Large copy: use non-temporal stores if > L3 cache
    ; (to avoid cache pollution)
    cmp     rdx, 8*1024*1024    ; 8MB (approximate L3 size)
    jg      .nt_copy
    
.avx_copy:
    ; AVX copy: 32 bytes per store
    ; Two loads per iteration for better throughput
    sub     rdx, 63             ; Adjust for 2x32 byte copies
    js      .avx_tail
    
.avx_loop:
    vmovups ymm0, [rsi]         ; Load 32 bytes
    vmovups ymm1, [rsi + 32]    ; Load next 32 bytes
    vmovups [rdi], ymm0         ; Store 32 bytes
    vmovups [rdi + 32], ymm1    ; Store next 32 bytes
    add     rsi, 64
    add     rdi, 64
    sub     rdx, 64
    jns     .avx_loop
    
    add     rdx, 64             ; Restore remainder
    ; Fall through to tail handling
    jmp     .tail
    
.nt_copy:
    ; Non-temporal stores bypass cache
    sub     rdx, 63
    js      .tail
.nt_loop:
    vmovups ymm0, [rsi]
    vmovups ymm1, [rsi + 32]
    vmovntps [rdi], ymm0        ; Non-temporal store
    vmovntps [rdi + 32], ymm1
    add     rsi, 64
    add     rdi, 64
    sub     rdx, 64
    jns     .nt_loop
    sfence                      ; Fence after non-temporal stores
    add     rdx, 64
    jmp     .tail
    
.medium_copy:
    ; SSE copy for medium sizes
    sub     rdx, 31
    js      .tail
.sse_loop:
    movdqu  xmm0, [rsi]
    movdqu  xmm1, [rsi + 16]
    movdqu  [rdi], xmm0
    movdqu  [rdi + 16], xmm1
    add     rsi, 32
    add     rdi, 32
    sub     rdx, 32
    jns     .sse_loop
    add     rdx, 32
    
.tail:
    ; Handle remaining bytes (0-63)
    test    rdx, 32
    jz      .tail_16
    movdqu  xmm0, [rsi]
    movdqu  xmm1, [rsi + 16]
    movdqu  [rdi], xmm0
    movdqu  [rdi + 16], xmm1
    add     rsi, 32
    add     rdi, 32
    
.tail_16:
    test    rdx, 16
    jz      .tail_8
    movdqu  xmm0, [rsi]
    movdqu  [rdi], xmm0
    add     rsi, 16
    add     rdi, 16

.tail_8:
    test    rdx, 8
    jz      .tail_4
    mov     rax, [rsi]
    mov     [rdi], rax
    add     rsi, 8
    add     rdi, 8
    
.tail_4:
    test    rdx, 4
    jz      .tail_2
    mov     eax, [rsi]
    mov     [rdi], eax
    add     rsi, 4
    add     rdi, 4
    
.tail_2:
    test    rdx, 2
    jz      .tail_1
    movzx   eax, word [rsi]
    mov     [rdi], ax
    add     rsi, 2
    add     rdi, 2
    
.tail_1:
    test    rdx, 1
    jz      .done
    movzx   eax, byte [rsi]
    mov     [rdi], al
    
.done:
    vzeroupper                  ; Avoid SSE/AVX transition penalty
    leave
    ret
    
.small_copy:
    ; rep movsb for small sizes (fast on modern CPUs with ERMSB)
    mov     rcx, rdx
    rep movsb
    leave
    ret
```

---

## 26. Checklist สำหรับ Optimization

### 26.1 Pipeline Analysis Checklist

```
□ วัด baseline performance ด้วย perf stat
□ ระบุ hot loop ด้วย perf record + perf report
□ วิเคราะห์ด้วย llvm-mca เพื่อดู theoretical throughput
□ ตรวจสอบ Agner Fog tables สำหรับ exact latency/throughput
□ ดู CPU-specific port mapping (Intel Skylake, Alder Lake, etc.)

Data Hazard Check:
□ มี load-use hazard หรือไม่? (load ตามด้วย immediate use)
□ มี serial dependency chain ยาวหรือไม่?
□ สามารถ unroll เพื่อ expose parallelism ได้หรือไม่?
□ สามารถใช้ multiple accumulators ได้หรือไม่?

Structural Hazard Check:
□ มี port pressure bottleneck หรือไม่?
□ สามารถ balance ports ด้วย alternative instructions ได้หรือไม่?
□ มี division/sqrt ที่สามารถแทนด้วย reciprocal multiply ได้หรือไม่?

Control Hazard Check:
□ มี unpredictable branches หรือไม่?
□ สามารถใช้ cmov/setcc แทน branches ได้หรือไม่?
□ มี branch ในloop ที่ดูเหมือน predictable หรือไม่?

Memory Check:
□ Working set fit in L1/L2/L3?
□ Access pattern เป็น sequential หรือ random?
□ ควร prefetch หรือไม่?
□ ใช้ non-temporal stores สำหรับ large writes?
□ มี store-forwarding violations หรือไม่?
```

---

## 27. Tools Summary

### 27.1 เครื่องมือที่ใช้บ่อย

```bash
# 1. perf - Linux Performance Counters
perf stat ./program
perf record ./program && perf report
perf annotate --symbol=myfunction

# 2. llvm-mca - Static Analysis
llvm-mca -mcpu=skylake -iterations=100 code.s

# 3. Agner Fog tools (Windows)
# objconv - object file converter + disassembler
# ดาวน์โหลดจาก: https://www.agner.org/optimize/

# 4. IACA (Intel Architecture Code Analyzer) - deprecated
# ใช้ SDE (Intel Software Development Emulator) แทน

# 5. uiCA - Accurate throughput estimator
# https://uica.uops.info/
# uica mycode.asm

# 6. OSACA - Open Source Architecture Code Analyzer
# pip install osaca
osaca --arch SKX mycode.s

# 7. Intel VTune Profiler (ฟรีสำหรับ personal use)
# vtune -collect hotspots ./program
# vtune -report hotspots

# 8. AMD uProf (สำหรับ AMD CPUs)
# AMDuProfCLI collect --config tbp ./program
```

---

## 28. ตัวอย่างโจทย์ปฏิบัติ

### โจทย์ 1: วิเคราะห์ Dependency Chain

```nasm
; วิเคราะห์ว่า loop นี้ใช้กี่ cycles ต่อ iteration บน Skylake?
; (ดู Agner Fog table สำหรับ latency)
.loop:
    imul rax, rbx       ; Latency: ?
    add  rax, rcx       ; Latency: ?
    imul rax, rdx       ; Latency: ?
    dec  r8             ; Latency: ?
    jnz  .loop

; เฉลย: critical path = imul(3) + add(1) + imul(3) = 7 cycles/iter
; (dec+jnz เป็น fused แต่ไม่ใช่ critical path)
```

### โจทย์ 2: แก้ Port Pressure

```nasm
; ปรับปรุง code ต่อไปนี้ให้ใช้ทุก port ได้สมดุลขึ้น:
.loop:
    imul r8,  [rdi + rax*8]      ; Port 1
    imul r9,  [rdi + rax*8 + 8]  ; Port 1
    imul r10, [rdi + rax*8 + 16] ; Port 1
    imul r11, [rdi + rax*8 + 24] ; Port 1
    add  rax, 4
    cmp  rax, rcx
    jl   .loop

; 힌트: IMUL ใช้ Port 1 ทั้งหมด → throughput 1/cycle
;      แต่มี 4 IMUL ต่อ iteration → bottleneck!
;      พิจารณา: สามารถใช้ shift + add แทน IMUL ได้หรือไม่?
;      (ถ้าตัวคูณเป็น constant power of 2)
```

### โจทย์ 3: แก้ Branch Penalty

```nasm
; แปลง code นี้ให้เป็น branchless:
; (สมมติว่า a[i] มี random values)
count_positives:
    xor  ecx, ecx       ; count = 0
    xor  eax, eax       ; i = 0
.loop:
    mov  edx, [rdi + rax*4]
    test edx, edx
    jle  .skip          ; ← unpredictable!
    inc  ecx
.skip:
    inc  eax
    cmp  eax, esi
    jl   .loop
    mov  eax, ecx
    ret

; เฉลย:
count_positives_branchless:
    xor  ecx, ecx
    xor  eax, eax
.loop:
    mov  edx, [rdi + rax*4]
    test edx, edx
    setg dl             ; dl = 1 if positive, 0 if not
    movzx edx, dl
    add  ecx, edx       ; count += (a[i] > 0)
    inc  eax
    cmp  eax, esi
    jl   .loop
    mov  eax, ecx
    ret
```

---

## 29. อ้างอิงและแหล่งเรียนรู้เพิ่มเติม

### 29.1 เอกสารและหนังสือ

```
1. Agner Fog's Optimization Guides (FREE)
   https://www.agner.org/optimize/
   - Software optimization resources
   - Instruction tables
   - Microarchitecture manual
   - Calling conventions

2. Intel 64 and IA-32 Architectures Optimization Reference Manual
   https://software.intel.com/en-us/articles/intel-sdm
   - แหล่งข้อมูลทางการจาก Intel

3. "Computer Organization and Design" - Patterson & Hennessy
   - ทฤษฎี pipeline ที่สมบูรณ์

4. "Computer Architecture: A Quantitative Approach" - Hennessy & Patterson
   - Deep dive ใน OoO execution, superscalar

5. "Hacker's Delight" - Henry S. Warren
   - Tricks สำหรับ bit manipulation, multiply tricks

6. Intel Architecture Code Analyzer (IACA) Documentation
   https://software.intel.com/en-us/articles/intel-architecture-code-analyzer
```

### 29.2 เว็บไซต์ที่มีประโยชน์

```
1. https://uops.info/
   - Database ของ μops สำหรับ Intel/AMD CPUs ทุกรุ่น
   - ละเอียดกว่า Agner Fog สำหรับบาง CPUs

2. https://godbolt.org/ (Compiler Explorer)
   - ดู assembly output จาก C/C++
   - ทดสอบ instruction different CPUs

3. https://www.felixcloutier.com/x86/
   - x86 Instruction reference (readable version of Intel manual)

4. https://sandpile.org/
   - x86 CPU microarchitecture details

5. https://llvm.org/docs/CommandGuide/llvm-mca.html
   - llvm-mca documentation
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **Modern Superscalar Pipeline**: in-order fetch → OoO execute → in-order retire
2. **Pipeline Stages**: IF, ID, IS, EX, WB และหน้าที่ของแต่ละ stage
3. **Execution Units**: ALU, AGU, FPU, Load/Store และ port mapping บน Intel Skylake
4. **Reservation Stations**: buffer สำหรับ μops ที่รอ execute
5. **Reorder Buffer**: ทำให้ OoO execution มี precise exceptions
6. **Register Renaming**: แก้ false dependencies (WAR, WAW)
7. **Data Hazards**: RAW (true), WAR (false), WAW (false) และวิธีแก้
8. **Structural Hazards**: execution unit conflicts และการ balance ports
9. **Control Hazards**: branch misprediction และ branchless techniques
10. **Pipeline Stalls**: วิธีนับและหลีกเลี่ยง
11. **Forwarding/Bypassing**: ลด stall cycles
12. **Instruction Scheduling**: เรียง instructions เพื่อ throughput สูงสุด
13. **Latency vs Throughput**: ความแตกต่างและ implications
14. **Dependency Chain Analysis**: serial vs parallel chains
15. **Intel Microarchitecture**: Skylake, Alder Lake details
16. **Tools**: Agner Fog tables, IACA, llvm-mca สำหรับการวิเคราะห์

ความรู้เหล่านี้เป็นพื้นฐานสำคัญสำหรับการเขียน assembly ที่มีประสิทธิภาพสูง และการทำความเข้าใจว่าทำไมโค้ดบางอย่างถึงเร็วหรือช้ากว่าที่คาด

---

*Part 061 จบ — ต่อไป: Part 062: SIMD Optimization Techniques*
