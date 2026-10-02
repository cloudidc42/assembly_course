# Part 063: Branch Prediction Optimization

## การทำนายสาขา (Branch Prediction) และการเพิ่มประสิทธิภาพโค้ด

---

## บทนำ (Introduction)

Branch prediction เป็นหนึ่งในเทคนิคสำคัญที่สุดใน CPU สมัยใหม่ เพื่อซ่อน latency ของ pipeline
เมื่อ CPU พบ conditional branch instruction มันจะต้อง "เดา" ว่า branch จะ taken หรือ not taken
ก่อนที่จะรู้ผลลัพธ์ที่แท้จริง

การเดาผิด (misprediction) มีต้นทุนสูงมาก — ประมาณ **15-20 clock cycles** บน Sandy Bridge
ซึ่งหมายความว่า ถ้า branch misprediction rate สูง ประสิทธิภาพจะลดลงอย่างมีนัยสำคัญ

```
Pipeline Stages (Sandy Bridge):
[IF] [ID] [IS] [EX] [WB]
  |    |    |    |    |
  ↓    ↓    ↓    ↓    ↓
Fetch Decode Issue Execute Write-back

Branch misprediction ทำให้ต้อง flush pipeline ทั้งหมด
และเริ่มใหม่จาก instruction ที่ถูกต้อง
```

---

## 1. Branch Predictor Internals (โครงสร้างภายในของ Branch Predictor)

### 1.1 BTB — Branch Target Buffer

BTB เป็น cache ขนาดเล็กที่เก็บ **ที่อยู่ปลายทาง** ของ branch instructions
เมื่อ CPU fetch instruction ที่เคยเป็น branch มาก่อน จะ lookup BTB เพื่อรู้ว่า
ควร fetch instruction ถัดไปจากที่ไหน

```
BTB Structure:
+------------------+------------------+------------------+
|   PC tag (bits)  |  Target address  |  Valid bit       |
+------------------+------------------+------------------+
|  0x401234 [tag]  |  0x401A00        |  1               |
|  0x401278 [tag]  |  0x401B40        |  1               |
|  0x4012AC [tag]  |  0x40108C        |  1               |
|  ...             |  ...             |  ...             |
+------------------+------------------+------------------+

BTB ขนาดทั่วไป: 512 - 4096 entries
```

**การทำงาน:**
1. CPU fetch instruction ที่ address X
2. ตรวจสอบ BTB ด้วย address X
3. ถ้า hit → ใช้ target address จาก BTB เพื่อ fetch ต่อ
4. ถ้า miss → ไม่รู้ target → ต้อง wait หรือ guess fall-through

### 1.2 BHT — Branch History Table

BHT เก็บ **ประวัติการ taken/not-taken** ของ branch แต่ละ branch
ใช้ saturating counter เพื่อ track ว่า branch มักจะ taken หรือไม่

```
2-bit Saturating Counter (Smith Counter):

States:
  00 = Strongly Not Taken (SNT)
  01 = Weakly Not Taken   (WNT)
  10 = Weakly Taken       (WT)
  11 = Strongly Taken     (ST)

Transitions:
  SNT --[taken]-→ WNT --[taken]-→ WT --[taken]-→ ST
  ST --[not taken]-→ WT --[not taken]-→ WNT --[not taken]-→ SNT

ข้อดี: ทนต่อ "noise" ครั้งเดียว ไม่เปลี่ยน prediction ทันที
```

### 1.3 PHT — Pattern History Table

PHT ใช้ร่วมกับ **Branch History Register (BHR)** เพื่อจำ pattern ของ branches
BHR เก็บ N bits ของ taken/not-taken history ล่าสุด

```
PHT with 2-bit BHR (4-entry PHT):

BHR (Branch History Register): [T][NT][T][T] = 1011b = 11

PHT (Pattern History Table):
  Entry 00: WT  (เมื่อ pattern NT-NT ก่อนหน้า → Weakly Taken)
  Entry 01: ST  (เมื่อ pattern NT-T ก่อนหน้า  → Strongly Taken)
  Entry 10: SNT (เมื่อ pattern T-NT ก่อนหน้า  → Strongly Not Taken)
  Entry 11: ST  (เมื่อ pattern T-T ก่อนหน้า   → Strongly Taken)

การ predict: ดู BHR bits → index PHT → อ่าน 2-bit counter
```

---

## 2. Two-Level Adaptive Prediction

สร้างขึ้นโดย Tse-Yu Yeh และ Yale Patt (1991) เป็นรากฐานของ predictor สมัยใหม่

### 2.1 Local History Predictor

แต่ละ branch มี BHR ของตัวเอง

```
Local Predictor:
+-------------------+         +------------------+
|  Branch PC        |         |  Local BHR Table  |
|  0x401234         |→ index→ |  entry[k]: 10110b |
|  0x401278         |         |  entry[m]: 01001b |
+-------------------+         +------------------+
                                      ↓
                              +------------------+
                              |  PHT (per-branch) |
                              |  counter[10110]   | → predict
                              +------------------+
```

### 2.2 Global History Predictor

ใช้ BHR เดียวร่วมกัน (global) สำหรับทุก branch

```
Global Predictor:
                   +---------------------+
                   |  Global BHR         |
                   |  [T,NT,T,T,NT,...] |
                   +---------------------+
                            ↓
              XOR กับ PC ของ branch
                            ↓
                   +---------------------+
                   |  Global PHT          |
                   |  2-bit counters      |
                   +---------------------+
                            ↓
                         Prediction
```

### 2.3 Tournament Predictor (AMD Athlon, Alpha 21264)

ใช้ทั้ง local และ global predictor แล้วเลือกว่าจะเชื่อใคร

```
Tournament Predictor:

  Branch → [Local Predictor]  → prediction_A
        → [Global Predictor] → prediction_B
        → [Choice Predictor] → เลือก A หรือ B

Choice Predictor: 2-bit counter ต่อ branch
  - ถ้า local ถูกบ่อยกว่า → เลือก local
  - ถ้า global ถูกบ่อยกว่า → เลือก global
```

---

## 3. TAGE Predictor (TAgged GEometric)

TAGE เป็น predictor รุ่นใหม่ที่ใช้ใน Intel และ AMD processors สมัยใหม่
พัฒนาโดย André Seznec (2006)

### 3.1 โครงสร้าง TAGE

```
TAGE Components:

T0: Base predictor (bimodal, 2-bit counters)
T1: Tagged predictor ใช้ history length h1
T2: Tagged predictor ใช้ history length h2 (h2 > h1)
T3: Tagged predictor ใช้ history length h3 (h3 > h2)
T4: Tagged predictor ใช้ history length h4 (h4 > h3)

History lengths (geometric series): 5, 15, 44, 130, 382, ...
nh = L(1) * α^i

ประโยชน์: จำ correlation pattern ระยะยาวมาก
บาง branch ขึ้นกับ pattern ของ branches อื่นๆ หลายสิบ instructions ก่อนหน้า
```

### 3.2 การ Predict ด้วย TAGE

```
TAGE Prediction Flow:

1. Query T4 ด้วย (PC XOR global_history[0..h4])
   → ถ้า tag match → ใช้ counter จาก T4 เป็น prediction

2. ถ้า T4 miss → Query T3
   → ถ้า tag match → ใช้ counter จาก T3

3. ต่อไปเรื่อยๆ จนถึง T1
   → ถ้าทุก Ti miss → ใช้ T0 (base predictor)

"ใช้ predictor ที่มี history ยาวที่สุดที่ hit"
```

### 3.3 การ Update TAGE

```
Update Rules:
- อัพเดท counter ของ provider (predictor ที่ให้ prediction)
- ถ้า prediction ผิด: พยายาม allocate entry ใน Ti ที่มี history ยาวกว่า
- Replacement policy: ชอบ entries ที่มี "useful" counter = 0
- Periodic reset ของ useful bits เพื่อป้องกัน thrashing
```

---

## 4. Indirect Branch Prediction: IBT

Indirect branches คือ branches ที่ target address ไม่คงที่
เช่น `jmp [rax]`, `call [rbx]`, virtual function calls, function pointers

### 4.1 ปัญหาของ Indirect Branches

```asm
; Direct branch — BTB รู้ target
jne .label_fixed      ; target คงที่ = 0x401234

; Indirect branch — target ไม่คงที่
jmp rax              ; target ขึ้นกับ runtime value ของ rax
call [rbx + 8]       ; virtual function call
```

```
IBT (Indirect Branch Target):
- Intel เพิ่ม IBT ใน Control-flow Enforcement Technology (CET)
- IBT instruction: ENDBR32, ENDBR64
- CPU ต้องเห็น ENDBR ที่ target ของ indirect jump
- ป้องกัน ROP/JOP attacks

; ตัวอย่าง ENDBR64:
section .text
func_ptr_target:
    endbr64          ; IBT: บอก CPU ว่า function นี้เป็น valid indirect target
    push rbp
    mov rbp, rsp
    ; ...
    pop rbp
    ret
```

### 4.2 Indirect Branch Predictor (Intel)

```
Intel Indirect Branch Predictor:
- ใช้ target history เพิ่มเติม
- path-based prediction
- เก็บ history ของ N targets ที่ผ่านมา

เช่น virtual dispatch loop:
for (auto* obj : objects) {
    obj->method();  // indirect call ผ่าน vtable
}

ถ้า objects[] มีหลาย types สลับกัน → predictor จำ pattern ได้
ถ้า random types → misprediction สูง
```

---

## 5. Return Address Stack (RAS) และ RSB

### 5.1 Return Address Stack (RAS)

RAS เป็น hardware stack ที่เก็บ return address สำหรับ `call` instructions

```
RAS Operation:

CALL instruction:
  push_RAS(next_PC)   ; เก็บ return address ใน RAS
  jump_to(target)

RET instruction:
  predicted_return = pop_RAS()
  แทนที่จะรอ memory read จาก stack
  CPU ทำนาย return address จาก RAS ทันที

ความแม่นยำ: สูงมาก (~97-99%) สำหรับ well-structured code
ขนาด RAS: 16-64 entries ทั่วไป
```

```
RAS Overflow (deep recursion):
ถ้า call depth > RAS size → entries เก่าจะถูก overwrite
return จาก recursive calls ลึกๆ จะ mispredicit

; ตัวอย่างปัญหา:
recursive_func:
    ; ถ้า call depth = 100 และ RAS = 16 entries
    ; entries ที่ 17-100 จะ mispredicit เมื่อ return
    call recursive_func
    ret
```

### 5.2 RSB — Return Stack Buffer

RSB เป็นชื่อที่ Intel ใช้สำหรับ RAS (เป็นสิ่งเดียวกัน)
แต่มีความสำคัญพิเศษหลัง Spectre vulnerability

```
RSB และ Spectre:

ปัญหา: Spectre variant 2 (CVE-2017-5715)
- Attacker สามารถ poison BTB ของ indirect branches
- ทำให้ speculative execution ไปยัง gadget ที่เลือกได้

RSB stuffing (mitigation):
- หลัง context switch หรือ privilege change
- OS จะ "stuff" RSB ด้วย addresses ที่ไม่เป็น secret
- ป้องกัน RSB underflow ที่อาจนำไปสู่ speculation ผ่าน BTB

; Linux kernel RSB stuffing (conceptual):
; FILL_RETURN_BUFFER macro ใน arch/x86/include/asm/nospec-branch.h
```

---

## 6. Spectre Vulnerability และ Branch Prediction

### 6.1 Spectre Variant 1 (Bounds Check Bypass)

```c
// Vulnerable C code:
if (index < array1_size) {          // Branch ที่ถูก exploit
    value = array1[index];           // Speculative load
    x = array2[value * 512];        // Cache timing side channel
}

// Attack:
// 1. Train predictor ให้คิดว่า branch จะ taken (index valid)
// 2. ใส่ index ที่ out-of-bounds
// 3. CPU speculatively ทำ: load secret, index array2
// 4. แม้ branch ถูก squash แต่ cache state ยังอยู่
// 5. Attacker วัด timing ของ array2 access → รู้ค่า secret
```

### 6.2 Mitigation และผลกระทบต่อ Performance

```asm
; lfence (Load Fence) — serializing instruction
; หยุด speculative execution จนกว่า branch ก่อนหน้าจะ resolve

; Mitigated code (conceptual assembly):
cmp rax, [array1_size]
jae .out_of_bounds
lfence                    ; ป้องกัน speculative execution ต่อ
movzx rdx, byte [array1 + rax]
; ...

; ข้อเสีย: lfence มีต้นทุน ~15-30 cycles
; กระทบ performance ของ loop ที่มี bounds check

; Retpoline — mitigation สำหรับ indirect branches:
; แทน indirect jump ด้วย pattern พิเศษที่ไม่ leak ผ่าน BTB
retpoline_thunk:
    call .inner
.inner:
    ; manipulate RSB
    lea rsp, [rsp + 8]       ; ลบ return address จาก stack
    lfence
    ret                       ; return ไปยัง RSB → infinite loop speculation
```

---

## 7. Branch Misprediction Penalty

### 7.1 Penalty บน Sandy Bridge (15-20 cycles)

```
Sandy Bridge Pipeline (14 stages):
Stage 1:  IF1  — Instruction Fetch 1
Stage 2:  IF2  — Instruction Fetch 2
Stage 3:  ID   — Instruction Decode
Stage 4:  RAT  — Register Alias Table
Stage 5:  ROB  — Reorder Buffer allocation
Stage 6:  RS   — Reservation Station
Stage 7:  EX1  — Execute 1
Stage 8:  EX2  — Execute 2
Stage 9:  EX3  — Execute 3 (branch resolve)
Stage 10: WB   — Write Back
...

Branch resolution เกิดที่ stage 9
Pipeline flush = 14 - 9 = ~5 stages ของ work ที่ต้อง redo
รวม fetch latency = ~15-20 cycles total penalty

สมการโดยประมาณ:
IPC_effective = IPC_peak × (1 - misprediction_rate × penalty / IPC_peak)
```

### 7.2 Measurement ด้วย perf stat

```bash
# วัด branch mispredictions:
perf stat -e branches,branch-misses ./your_program

# ตัวอย่าง output:
#  Performance counter stats for './sort_benchmark':
#
#     1,234,567,890   branches                  #  987.654 M/sec
#        12,345,678   branch-misses             #    1.00% of all branches
#
#        1.250042785 seconds time elapsed

# วัดแบบละเอียดกว่า:
perf stat -e \
  branches,\
  branch-misses,\
  branch-load-misses,\
  mispredicted-branch,\
  cycles,\
  instructions \
  ./your_program

# Annotate source ด้วย branch stats:
perf record -e branch-misses:pp ./your_program
perf annotate
```

### 7.3 ผลกระทบต่อ Performance จริง

```
ตัวอย่าง: Bubble Sort vs Sorted Input

// Random data (high misprediction ~50%):
for (int i = 0; i < n-1; i++)
    if (a[i] > a[i+1])        // ~50% branch misprediction
        swap(a[i], a[i+1]);

// Sorted data (low misprediction ~0%):
// branch almost always not taken → predictor trains perfectly

เวลา (n=100000):
  Random:  ~2.5 seconds  (มาก mispredictions)
  Sorted:  ~0.3 seconds  (น้อย mispredictions)

ต่างกัน ~8x เฉพาะจาก branch prediction!
```

---

## 8. Branchless Programming Techniques

การเขียนโค้ดโดยหลีกเลี่ยง conditional branches เพื่อลด misprediction penalty

### 8.1 CMOV — Conditional Move

`CMOV` เป็น instruction ที่ทำ conditional data movement โดยไม่มี branch

```asm
; Traditional code with branch:
; if (rax > rbx) rax = rbx

section .text
global branch_min
branch_min:
    cmp rax, rbx
    jle .done           ; branch — อาจ mispredicit
    mov rax, rbx
.done:
    ret

; Branchless version with CMOV:
global branchless_min
branchless_min:
    cmp rax, rbx
    cmovg rax, rbx      ; conditional move if greater — ไม่มี branch!
    ret

; CMOV variants:
; cmova  — if above (unsigned)
; cmovae — if above or equal
; cmovb  — if below (unsigned)
; cmovbe — if below or equal
; cmove  — if equal
; cmovne — if not equal
; cmovg  — if greater (signed)
; cmovge — if greater or equal
; cmovl  — if less (signed)
; cmovle — if less or equal
; cmovs  — if sign (negative)
; cmovns — if not sign (positive)
```

### 8.2 Branchless Min และ Max

```asm
; ============================================================
; Branchless Min (NASM x86-64)
; ============================================================
; รับ a ใน rdi, b ใน rsi
; return min ใน rax

section .text
global branchless_min_signed
branchless_min_signed:
    ; วิธี CMOV (แนะนำ):
    mov rax, rdi
    cmp rax, rsi
    cmovg rax, rsi      ; ถ้า rax > rsi, rax = rsi
    ret

global branchless_max_signed
branchless_max_signed:
    mov rax, rdi
    cmp rax, rsi
    cmovl rax, rsi      ; ถ้า rax < rsi, rax = rsi
    ret

; วิธี arithmetic (ไม่ใช้ CMOV — สำหรับ reference):
; min(a,b) ด้วย bit tricks:
; t = a - b
; min = b + (t & (t >> 63))   ; 63 = bit width - 1
global arithmetic_min
arithmetic_min:
    ; rdi = a, rsi = b
    mov rax, rdi        ; rax = a
    sub rax, rsi        ; rax = a - b (t)
    mov rcx, rax
    sar rcx, 63         ; rcx = sign_extend(t >> 63) = 0 หรือ -1
    and rax, rcx        ; rax = t & mask (0 ถ้า a>=b, t ถ้า a<b)
    add rax, rsi        ; rax = b + (t & mask) = min(a,b)
    ret
```

### 8.3 Branchless Absolute Value

```asm
; ============================================================
; Branchless Absolute Value
; ============================================================
; |x| = (x + mask) XOR mask
; ที่ mask = x >> 31 (sign extension)
;
; ถ้า x >= 0: mask = 0,  |x| = (x + 0) XOR 0 = x
; ถ้า x < 0:  mask = -1, |x| = (x - 1) XOR (-1) = ~(x-1) = -x

section .text
global branchless_abs32
branchless_abs32:
    ; rdi = x (32-bit value ใน edi)
    mov eax, edi
    mov ecx, eax
    sar ecx, 31         ; ecx = mask (0 หรือ 0xFFFFFFFF)
    add eax, ecx        ; eax = x + mask
    xor eax, ecx        ; eax = (x + mask) XOR mask = |x|
    ret

global branchless_abs64
branchless_abs64:
    ; rdi = x (64-bit)
    mov rax, rdi
    mov rcx, rax
    sar rcx, 63         ; rcx = mask (0 หรือ 0xFFFFFFFFFFFFFFFF)
    add rax, rcx
    xor rax, rcx
    ret

; ตรวจสอบความถูกต้อง:
; x = -5 = 0xFFFFFFFFFFFFFFFB
; mask = -1 = 0xFFFFFFFFFFFFFFFF
; x + mask = -6 = 0xFFFFFFFFFFFFFFFA
; (-6) XOR (-1) = 0xFFFFFFFFFFFFFFFA XOR 0xFFFFFFFFFFFFFFFF = 0x0000000000000005 = 5 ✓
```

### 8.4 SETcc — Set Byte on Condition

```asm
; SETcc: set a byte register based on condition flags
; ใช้ทำ boolean operations โดยไม่มี branch

section .text

; ตัวอย่าง: count elements > threshold
global count_greater
count_greater:
    ; rdi = array pointer
    ; rsi = length
    ; rdx = threshold
    ; return count ใน rax

    xor rax, rax        ; count = 0
    xor rcx, rcx        ; index = 0

.loop:
    cmp rcx, rsi
    jge .done

    cmp qword [rdi + rcx*8], rdx
    setg cl_byte        ; set cl = 1 ถ้า > threshold, 0 ถ้าไม่
    ; แต่ cl ใช้ไม่ได้ตรงๆ เพราะ clobbered
    ; ใช้ r8b แทน:
    setg r8b            ; r8b = 1 หรือ 0
    movzx r8, r8b       ; zero-extend เป็น 64-bit
    add rax, r8         ; count += (value > threshold)

    inc rcx
    jmp .loop

.done:
    ret

; เวอร์ชันที่ดีกว่า ด้วย SIMD (AVX2):
; ดูใน part สำหรับ SIMD optimization
```

### 8.5 Branchless Selection (ternary operator)

```asm
; ============================================================
; Branchless Selection: result = cond ? a : b
; เทียบเท่า: result = -(cond) AND a OR ~-(cond) AND b
; ============================================================
; โดยที่ -(cond) = 0 ถ้า cond=0, -1 (0xFFFF...) ถ้า cond=1

section .text
global branchless_select
branchless_select:
    ; rdi = cond (0 หรือ non-zero)
    ; rsi = a
    ; rdx = b
    ; return: rax = cond ? a : b

    ; normalize cond ให้เป็น 0 หรือ 1
    test rdi, rdi
    setnz al            ; al = (cond != 0) ? 1 : 0
    movzx rax, al

    ; สร้าง mask: -(cond) = 0 หรือ 0xFFFF...
    neg rax             ; rax = 0 หรือ 0xFFFFFFFFFFFFFFFF

    ; branchless selection
    ; rax    = mask (0 หรือ -1)
    ; a = rsi, b = rdx
    mov rcx, rax
    not rcx             ; rcx = ~mask

    and rsi, rax        ; rsi = a & mask   (= a ถ้า cond, = 0 ถ้าไม่)
    and rdx, rcx        ; rdx = b & ~mask  (= 0 ถ้า cond, = b ถ้าไม่)
    or rax, rsi         ; รวมกัน
    ; oops rax ถูก mask แล้ว ต้อง or อีกครั้ง:
    ; ใหม่:

    xor rax, rax
    test rdi, rdi
    setnz al
    movzx rax, al       ; rax = 0 หรือ 1
    neg rax             ; mask = 0 หรือ -1

    and rsi, rax        ; rsi = a & mask
    andn rdx, rax, rdx  ; rdx = b & ~mask (ใช้ BMI1 andn)
    ; หรือ:
    ; not rax → rcx
    ; and rdx, rcx

    or rax, rsi         ; ต้องเริ่มใหม่ rax ไม่ใช่ mask แล้ว

; วิธีที่สะอาดกว่า:
global branchless_select_clean
branchless_select_clean:
    ; rdi = cond, rsi = a, rdx = b
    mov rax, rdi
    test rax, rax
    setnz al
    movzx rax, al       ; rax = 0 หรือ 1
    neg rax             ; mask: 0 → 0, 1 → -1 (all-ones)

    ; result = (a & mask) | (b & ~mask)
    mov rcx, rax
    not rcx             ; rcx = ~mask

    and rsi, rax        ; a & mask
    and rdx, rcx        ; b & ~mask
    or rsi, rdx         ; combined

    mov rax, rsi
    ret

; อีกวิธีที่ง่าย — ใช้ CMOV:
global cmov_select
cmov_select:
    ; rdi = cond, rsi = a, rdx = b
    mov rax, rdx        ; default = b
    test rdi, rdi
    cmovnz rax, rsi     ; ถ้า cond != 0, rax = a
    ret
```

---

## 9. Branchless Clamp

```asm
; ============================================================
; Branchless Clamp: result = max(lo, min(x, hi))
; ============================================================

section .text
global branchless_clamp
branchless_clamp:
    ; rdi = x
    ; rsi = lo
    ; rdx = hi
    ; return: rax = clamp(x, lo, hi)

    mov rax, rdi

    ; clamp to hi: min(x, hi)
    cmp rax, rdx
    cmovg rax, rdx      ; ถ้า x > hi, rax = hi

    ; clamp to lo: max(result, lo)
    cmp rax, rsi
    cmovl rax, rsi      ; ถ้า result < lo, rax = lo

    ret

; ตรวจสอบ:
; clamp(5, 0, 10)  → 5  ✓
; clamp(-3, 0, 10) → 0  ✓
; clamp(15, 0, 10) → 10 ✓

; SIMD version (AVX2 สำหรับ array):
global clamp_array_avx2
clamp_array_avx2:
    ; rdi = array
    ; rsi = length (ต้องหาร 8 ลงตัว)
    ; rdx = lo
    ; rcx = hi

    vmovd xmm2, edx
    vpbroadcastd ymm2, xmm2     ; ymm2 = [lo, lo, lo, lo, lo, lo, lo, lo]
    vmovd xmm3, ecx
    vpbroadcastd ymm3, xmm3     ; ymm3 = [hi, ...]

    xor rax, rax
.loop:
    vmovdqu ymm0, [rdi + rax*4]
    vpminsd ymm0, ymm0, ymm3    ; min(x, hi)
    vpmaxsd ymm0, ymm0, ymm2    ; max(result, lo)
    vmovdqu [rdi + rax*4], ymm0

    add rax, 8
    cmp rax, rsi
    jl .loop

    vzeroupper
    ret
```

---

## 10. Sorting Hot-Path Branches

ในการ sort เช่น comparator function จะถูก call บ่อยมาก
เรียงลำดับ cases ตาม probability ที่เกิดขึ้นบ่อย

### 10.1 Comparison Function Optimization

```asm
; ============================================================
; Sort comparator — branchless version
; ============================================================
; int compare(int a, int b)
; return: negative ถ้า a < b, 0 ถ้า a = b, positive ถ้า a > b

section .text
global compare_branchy
compare_branchy:
    ; rdi = a, rsi = b (32-bit values ใน edi, esi)
    mov eax, 0
    cmp edi, esi
    jl .less            ; branch #1
    jg .greater         ; branch #2
    ret
.less:
    mov eax, -1
    ret
.greater:
    mov eax, 1
    ret

; Branchless version:
global compare_branchless
compare_branchless:
    ; rdi = a, rsi = b
    xor eax, eax
    cmp edi, esi
    setg al             ; al = 1 ถ้า a > b
    setnl cl            ; cl = 1 ถ้า a >= b (เดิม setge)
    ; ต้องการ: -1, 0, 1
    ; วิธีที่ง่าย:
    mov eax, edi
    sub eax, esi        ; a - b
    ; แต่อาจ overflow! ใช้วิธีที่ปลอดภัยกว่า:
    xor eax, eax
    cmp edi, esi
    setg al             ; al = (a > b) ? 1 : 0
    setl cl             ; cl = (a < b) ? 1 : 0
    sub eax, ecx        ; eax = (a>b) - (a<b) = {-1, 0, 1}
    ret

; ทดสอบ:
; compare(5, 3) → 1-0 = 1   ✓
; compare(3, 5) → 0-1 = -1  ✓
; compare(5, 5) → 0-0 = 0   ✓
```

### 10.2 Most-Likely-First Branch Ordering

```asm
; ============================================================
; เรียง branches ตาม probability (most likely first)
; ============================================================

; Bad: ถ้า input มักจะเป็น positive
validate_value:
    cmp rdi, 0
    jl .negative        ; น้อยครั้ง → branch taken น้อย แต่อยู่ก่อน
    cmp rdi, 1000
    jg .too_large       ; น้อยครั้ง
    ; common case: valid value
    mov rax, rdi
    ret
.negative:
    mov rax, -1
    ret
.too_large:
    mov rax, -2
    ret

; Good: common case ไม่ต้องมี branch
validate_value_optimized:
    ; สมมติ: ส่วนใหญ่ input อยู่ใน [0, 1000]
    sub rdi, 0          ; rdi -= lo (lo = 0, ไม่เปลี่ยน)
    cmp rdi, 1000
    ja .out_of_range    ; unsigned compare: ครอบคลุมทั้ง < 0 และ > 1000
    ; common case: valid
    mov rax, rdi
    ret
.out_of_range:
    ; จัดการ error
    mov rax, -1
    ret

; เทคนิค: แปลง "range check" จาก 2 branches เป็น 1 branch
; ด้วย unsigned arithmetic
; ถ้า x < lo หรือ x > hi ก็คือ (unsigned)(x - lo) > (hi - lo)
```

---

## 11. __builtin_expect (GCC Branch Hint)

### 11.1 การใช้งานใน C/C++

```c
// บอก compiler ว่า branch นี้มักจะ taken/not taken

// ไม่มี hint:
if (error_occurred) {
    handle_error();
}

// มี hint — บอกว่า error_occurred มักจะเป็น false:
if (__builtin_expect(error_occurred, 0)) {
    handle_error();
}

// หรือใช้ macro:
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

if (unlikely(error_occurred)) {
    handle_error();
}
```

### 11.2 ผลลัพธ์ Assembly

```c
// ตัวอย่างที่แสดงผลของ __builtin_expect:

int process(int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        if (__builtin_expect(data[i] < 0, 0)) {  // unlikely negative
            return -1;
        }
        sum += data[i];
    }
    return sum;
}
```

```asm
; Assembly ที่ compiler สร้าง (GCC -O2):
; โดยไม่มี __builtin_expect:
;   cmp [data+i], 0
;   jl error_case    ; branch อยู่ตรงกลาง code
;   add sum, [data+i]
;
; ด้วย __builtin_expect(x, 0):
;   cmp [data+i], 0
;   jl error_case    ; error path อยู่ห่างออกไป (cold path)
;   add sum, [data+i]
;   ; hot loop ต่อเนื่องกัน → better I-cache usage

; Compiler วาง "hot path" ไว้ตรงๆ (fall-through)
; และ "cold path" ไว้ไกลออกไป
; ทำให้ I-cache ใช้ได้มีประสิทธิภาพมากขึ้น
```

### 11.3 GNU Assembler Hints

```asm
; GAS (GNU Assembler) ไม่มี branch hint โดยตรง
; แต่ branch prediction hints มีอยู่ใน prefix (deprecated):
; 0x2E = branch not taken hint (CS segment override อดีต)
; 0x3E = branch taken hint (DS segment override อดีต)
; Intel ไม่รองรับแล้วใน P4+ แต่ AMD ยังรองรับบางรุ่น

; สมัยใหม่: วาง branch ให้ถูก (likely path = fall-through)
; เป็น "manual __builtin_expect"

section .text
global hot_function
hot_function:
    test rdi, rdi
    jz .cold_case       ; เมื่อ rdi=0 (unlikely) → jump away

    ; HOT PATH ตรงนี้ (likely case)
    add rax, rdi
    ; ...
    ret

.cold_case:
    ; cold code อยู่ห่างออกไป → ไม่กวน I-cache ของ hot path
    mov rax, 0
    ret
```

---

## 12. Removing Branches with Bitwise Arithmetic

### 12.1 Boolean to Mask Conversion

```asm
; ============================================================
; แปลง boolean เป็น bitmask โดยใช้ neg
; ============================================================
; neg 0 = 0
; neg 1 = -1 = 0xFFFFFFFF...

section .text

; ตัวอย่าง: คำนวณ value ที่แตกต่างกันตาม flag
; ถ้า flag=1: result = a
; ถ้า flag=0: result = b

global select_with_neg
select_with_neg:
    ; rdi = flag (0 หรือ 1)
    ; rsi = a
    ; rdx = b

    mov rax, rdi        ; rax = flag (0 หรือ 1)
    neg rax             ; rax = 0 หรือ -1 (mask)

    mov rcx, rax
    not rcx             ; rcx = ~mask

    and rsi, rax        ; a & mask
    and rdx, rcx        ; b & ~mask
    or rax, rsi         ; (ต้องระวัง rax ถูก clobber แล้ว)

    ; ทำใหม่อย่างถูกต้อง:
    mov rax, rdi
    neg rax             ; mask

    and rsi, rax        ; rsi = a ถ้า flag=1, 0 ถ้า flag=0
    not rax             ; ~mask
    and rdx, rax        ; rdx = 0 ถ้า flag=1, b ถ้า flag=0
    or rax, rsi         ; อีกครั้ง rax ถูก clobber...

; วิธีที่ถูกต้อง:
global select_bitwise
select_bitwise:
    ; rdi = flag (0 หรือ 1), rsi = a, rdx = b
    ; result = (a & mask) | (b & ~mask)

    mov rax, rdi
    neg rax             ; rax = mask (0 หรือ -1)

    mov r8, rax
    not r8              ; r8 = ~mask

    and rsi, rax        ; rsi = a & mask
    and rdx, r8         ; rdx = b & ~mask
    or rsi, rdx         ; combine

    mov rax, rsi
    ret
```

### 12.2 Branchless Fibonacci Check (is even)

```asm
; ============================================================
; Is Even — ไม่มี branch
; ============================================================

global is_even
is_even:
    ; rdi = number
    ; return 1 ถ้า even, 0 ถ้า odd
    xor rax, rax
    test rdi, 1         ; ตรวจ bit 0
    setz al             ; al = 1 ถ้า zero (even)
    ret

; Is Odd:
global is_odd
is_odd:
    mov rax, rdi
    and rax, 1          ; rax = bit 0
    ret
```

### 12.3 Branchless Sign Function

```asm
; ============================================================
; Sign function: sign(x) = -1, 0, หรือ 1
; ============================================================

global sign_function
sign_function:
    ; rdi = x
    ; return: -1 ถ้า x<0, 0 ถ้า x=0, 1 ถ้า x>0

    mov rax, rdi
    sar rax, 63         ; arithmetic shift: 0 หรือ -1

    neg rdi             ; -x
    sar rdi, 63         ; 0 หรือ -1
    neg rdi             ; 0 หรือ 1  (positive sign contribution)

    or rax, rdi         ; combine: -1 | 0 = -1, 0 | 1 = 1, 0 | 0 = 0
    ret

; ตรวจสอบ:
; x = 5:   sar → 0,  neg(5)=-5, sar(-5)→-1, neg(-1)=1,  0|1 = 1  ✓
; x = -3:  sar → -1, neg(-3)=3, sar(3) →0,  neg(0)=0,  -1|0 = -1 ✓
; x = 0:   sar → 0,  neg(0)=0,  sar(0) →0,  neg(0)=0,  0|0 = 0   ✓
```

---

## 13. Lookup Tables vs Branches

### 13.1 เมื่อ Lookup Table ดีกว่า Branches

```asm
; ============================================================
; ตัวอย่าง: แปลง digit เป็น character
; ============================================================

; วิธี branch:
digit_to_char_branch:
    ; rdi = digit (0-9 หรือ 10-15 สำหรับ hex)
    cmp rdi, 9
    jle .decimal
    ; a-f:
    add rdi, 'a' - 10
    mov rax, rdi
    ret
.decimal:
    add rdi, '0'
    mov rax, rdi
    ret

; วิธี lookup table:
section .data
hex_chars: db "0123456789abcdef"

section .text
digit_to_char_lut:
    lea rax, [rel hex_chars]
    movzx rax, byte [rax + rdi]
    ret

; Lookup table เหมาะเมื่อ:
; 1. Input range เล็ก (fit in cache)
; 2. Computation ซับซ้อน (หลาย branches)
; 3. Branches มี high misprediction rate

; ไม่เหมาะเมื่อ:
; 1. Table ใหญ่เกิน L1 cache → cache miss แพงกว่า branch
; 2. Pattern ง่าย predictable ได้
```

### 13.2 Jump Table (Switch Statement)

```asm
; ============================================================
; Jump Table — อีกรูปแบบของ lookup table
; ============================================================

section .data
align 8
jump_table:
    dq case_0
    dq case_1
    dq case_2
    dq case_3
    dq case_default

section .text
global switch_example
switch_example:
    ; rdi = selector (0-3)

    ; bounds check
    cmp rdi, 4
    jae .default

    ; indirect jump via table
    lea rax, [rel jump_table]
    jmp [rax + rdi*8]

case_0:
    mov rax, 100
    ret
case_1:
    mov rax, 200
    ret
case_2:
    mov rax, 300
    ret
case_3:
    mov rax, 400
    ret
.default:
case_default:
    mov rax, -1
    ret

; Jump table trade-offs:
; Pro: O(1) dispatch แทน O(n) if-else chain
; Pro: branch predictor สามารถ learn pattern
; Con: indirect branch → BTB miss ครั้งแรก
; Con: Spectre mitigation อาจทำให้ช้าลง (retpoline)
```

---

## 14. Predictable vs Unpredictable Branches

### 14.1 Predictable Branches

```asm
; ============================================================
; Branch ที่ predictable — ดี
; ============================================================

; Loop counter branch — predictable:
; taken N-1 ครั้ง, not-taken ครั้งสุดท้าย
; predictor train ได้ง่าย

global sum_array
sum_array:
    ; rdi = array, rsi = n
    xor rax, rax
    xor rcx, rcx
.loop:
    cmp rcx, rsi
    jge .done           ; น้อยครั้งมาก = predictable
    add rax, [rdi + rcx*8]
    inc rcx
    jmp .loop
.done:
    ret

; Error check branch — predictable:
; เกือบทุกครั้ง error = 0 → predictor เรียน not-taken
if_error_check:
    test rax, rax       ; error flag
    jnz .handle_error   ; taken น้อยมาก → strongly predicted not-taken
    ; normal processing...
    ret
.handle_error:
    ; ...
    ret
```

### 14.2 Unpredictable Branches (Performance Cliff)

```c
// ตัวอย่างที่สร้าง unpredictable branches:

// Random data comparison — worst case:
void sort_random_data(int* arr, int n) {
    for (int i = 0; i < n-1; i++) {
        for (int j = i+1; j < n; j++) {
            if (arr[i] > arr[j]) {   // ~50% taken → very bad!
                swap(arr[i], arr[j]);
            }
        }
    }
}

// Data-dependent branches — problematic:
int process_stream(char* data, int n) {
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (data[i] == 'A') {  // frequency unknown
            count++;
        }
    }
    return count;
}
```

```asm
; ============================================================
; Performance cliff demonstration
; ============================================================
; ทดสอบด้วย data ที่ sorted vs random

; Sorted data (branch always predictable):
; input: 1,2,3,4,5,...,n
; cmp จะ taken สม่ำเสมอ → predictor happy → fast

; Random data (branch ~50% misprediction):
; input: random → cmp mispredicted บ่อย → slow

; ตัวอย่างผล (n=10M):
; Sorted:  ~30ms  (0.1% misprediction)
; Random: ~250ms  (49.9% misprediction)

; Performance cliff: ~8x slowdown จาก misprediction!
```

### 14.3 Sorting-Based Optimization

```c
// เทคนิค: sort data ก่อนถ้าเป็นไปได้
// หรือ separate data ตาม branch condition

// ก่อน sort:
for (int i = 0; i < n; i++) {
    if (data[i] >= 128) {  // unpredictable
        sum += data[i];
    }
}

// หลัง sort:
std::sort(data, data + n);
for (int i = 0; i < n; i++) {
    if (data[i] >= 128) {  // predictable! — false region then true region
        sum += data[i];
    }
}
// ทำนายผิดครั้งเดียวตรง boundary → เร็วมาก!
```

---

## 15. Full Code Examples

### 15.1 Branchless Comparator สำหรับ Sorting

```asm
; ============================================================
; NASM x86-64: Branchless Comparison Network
; Sorting Networks สำหรับ small fixed-size arrays
; ============================================================

section .text
global sort3_branchless
; sort 3 elements ใน rdi[0], rdi[1], rdi[2]
sort3_branchless:
    ; โหลด elements
    mov rax, [rdi]
    mov rbx, [rdi + 8]
    mov rcx, [rdi + 16]

    ; Sorting network สำหรับ 3 elements:
    ; Compare-and-swap pairs: (0,1), (0,2), (1,2)

    ; Step 1: swap(0, 1) if a[0] > a[1]
    mov r8, rax
    mov r9, rbx
    cmp r8, r9
    cmovg r8, rbx       ; if a > b: r8 = b
    cmovg r9, rax       ; if a > b: r9 = a
    mov rax, r8
    mov rbx, r9

    ; Step 2: swap(0, 2) if a[0] > a[2]
    mov r8, rax
    mov r9, rcx
    cmp r8, r9
    cmovg r8, rcx
    cmovg r9, rax
    mov rax, r8
    mov rcx, r9

    ; Step 3: swap(1, 2) if a[1] > a[2]
    mov r8, rbx
    mov r9, rcx
    cmp r8, r9
    cmovg r8, rcx
    cmovg r9, rbx
    mov rbx, r8
    mov rcx, r9

    ; เก็บผลลัพธ์
    mov [rdi], rax
    mov [rdi + 8], rbx
    mov [rdi + 16], rcx

    ret

; ============================================================
global sort4_branchless
; sort 4 elements — sorting network optimal = 5 comparisons
sort4_branchless:
    mov rax, [rdi]
    mov rbx, [rdi + 8]
    mov rcx, [rdi + 16]
    mov rdx, [rdi + 24]

    ; Network: (0,1),(2,3),(0,2),(1,3),(1,2)

%macro CMP_SWAP 2
    mov r8, %1
    mov r9, %2
    cmp r8, r9
    cmovg r8, %2
    cmovg r9, %1
    mov %1, r8
    mov %2, r9
%endmacro

    ; (0,1)
    mov r8, rax
    mov r9, rbx
    cmp r8, r9
    cmovg r8, rbx
    cmovg r9, rax
    mov rax, r8
    mov rbx, r9

    ; (2,3)
    mov r8, rcx
    mov r9, rdx
    cmp r8, r9
    cmovg r8, rdx
    cmovg r9, rcx
    mov rcx, r8
    mov rdx, r9

    ; (0,2)
    mov r8, rax
    mov r9, rcx
    cmp r8, r9
    cmovg r8, rcx
    cmovg r9, rax
    mov rax, r8
    mov rcx, r9

    ; (1,3)
    mov r8, rbx
    mov r9, rdx
    cmp r8, r9
    cmovg r8, rdx
    cmovg r9, rbx
    mov rbx, r8
    mov rdx, r9

    ; (1,2)
    mov r8, rbx
    mov r9, rcx
    cmp r8, r9
    cmovg r8, rcx
    cmovg r9, rbx
    mov rbx, r8
    mov rcx, r9

    mov [rdi], rax
    mov [rdi + 8], rbx
    mov [rdi + 16], rcx
    mov [rdi + 24], rdx
    ret
```

### 15.2 Branchless Binary Search

```asm
; ============================================================
; Branchless Binary Search
; ใช้ CMOV แทน conditional jump
; ============================================================

section .text
global binary_search_branchless
; rdi = array (sorted int64[])
; rsi = length
; rdx = target
; return: rax = index หรือ -1

binary_search_branchless:
    push rbx

    ; base = 0, len = rsi
    xor rax, rax        ; base = 0
    mov rbx, rsi        ; len

.loop:
    test rbx, rbx
    jz .not_found

    mov rcx, rbx
    shr rcx, 1          ; mid_offset = len / 2

    lea r8, [rdi + rax*8]   ; pointer to base
    mov r9, [r8 + rcx*8]    ; arr[base + mid_offset]

    cmp r9, rdx
    setl r10b           ; r10 = (arr[mid] < target)
    movzx r10, r10b

    ; ถ้า arr[mid] < target: base += mid_offset + 1, len -= mid_offset + 1
    ; ถ้า arr[mid] >= target: base unchanged, len = mid_offset

    ; ต้อง update โดยไม่มี branch:
    ; new_base = base + (arr[mid] < target) * (mid_offset + 1)
    ; new_len  = len - mid_offset - (arr[mid] < target)

    lea r11, [rcx + 1]  ; mid_offset + 1
    imul r11, r10       ; * (arr[mid] < target)

    add rax, r11        ; new_base
    sub rbx, rcx        ; len -= mid_offset
    sub rbx, r10        ; len -= (arr[mid] < target)

    jmp .loop

.not_found:
    ; ตรวจว่า arr[base] == target
    lea r8, [rdi + rax*8]
    cmp [r8], rdx
    je .found
    mov rax, -1
    pop rbx
    ret
.found:
    pop rbx
    ret

; Note: Binary search แบบ branchless มี branch เดียวตอน verify
; ทำให้จำนวน mispredictions ลดลงจาก O(log n) เป็น O(1)
```

### 15.3 Sort Comparison สำหรับ Benchmark

```c
// ============================================================
// C code สำหรับ benchmark: branch vs branchless sort
// ============================================================

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

// Branch-based comparison
static inline int cmp_branch(const void* a, const void* b) {
    int x = *(const int*)a;
    int y = *(const int*)b;
    if (x < y) return -1;
    if (x > y) return  1;
    return 0;
}

// Branchless comparison
static inline int cmp_branchless(const void* a, const void* b) {
    int x = *(const int*)a;
    int y = *(const int*)b;
    return (x > y) - (x < y);  // branchless!
}

#define N 10000000

int main() {
    int* data1 = malloc(N * sizeof(int));
    int* data2 = malloc(N * sizeof(int));

    // fill random data
    srand(42);
    for (int i = 0; i < N; i++) {
        data1[i] = rand();
    }
    memcpy(data2, data1, N * sizeof(int));

    // benchmark branch-based:
    clock_t t1 = clock();
    qsort(data1, N, sizeof(int), cmp_branch);
    clock_t t2 = clock();

    // benchmark branchless:
    clock_t t3 = clock();
    qsort(data2, N, sizeof(int), cmp_branchless);
    clock_t t4 = clock();

    printf("Branch-based:    %.3f ms\n",
           1000.0 * (t2 - t1) / CLOCKS_PER_SEC);
    printf("Branchless:      %.3f ms\n",
           1000.0 * (t4 - t3) / CLOCKS_PER_SEC);

    free(data1);
    free(data2);
    return 0;
}

// คาดหวัง: branchless เร็วกว่า 10-30% บน random data
// เพราะ comparator ถูกเรียกหลายล้านครั้ง
```

---

## 16. Advanced Techniques

### 16.1 Profile-Guided Optimization (PGO)

```bash
# Step 1: Compile with instrumentation
gcc -O2 -fprofile-generate program.c -o program_instrumented

# Step 2: Run with representative input
./program_instrumented < typical_input.txt

# Step 3: Compile ด้วย profile data
gcc -O2 -fprofile-use program.c -o program_optimized

# ผล: GCC จะ:
# - reorder branches ตาม actual frequency
# - วาง hot code ใน same cache lines
# - inline functions ที่ถูกเรียกบ่อย
# - unroll loops ที่เหมาะสม
```

### 16.2 Intel IACA (Intel Architecture Code Analyzer)

```asm
; ทำ annotation สำหรับ IACA analysis
; (ใช้สำหรับ static analysis ของ throughput)

extern IACA_start
extern IACA_end

global hot_loop
hot_loop:
    call IACA_start

.loop:
    ; your hot loop code here
    movdqu xmm0, [rdi]
    paddd xmm0, xmm1
    movdqu [rdi], xmm0
    add rdi, 16
    dec rsi
    jnz .loop

    call IACA_end
    ret
```

### 16.3 Micro-benchmark สำหรับ Branch Prediction

```asm
; ============================================================
; Micro-benchmark: วัด branch misprediction cost
; ============================================================

section .data
align 64
test_array: times 1000000 dd 0    ; array of 1M integers

section .text
global bench_predictable_branch
bench_predictable_branch:
    ; เติม array ด้วย increasing values (0,1,2,...)
    ; branch "if x > 500000" จะ unpredictable ตรงกลาง
    ; แต่ predictable ในส่วน < 500000 และ > 500000

    lea rdi, [rel test_array]
    xor rcx, rcx
    xor rax, rax

.loop:
    cmp rcx, 1000000
    jge .done

    cmp dword [rdi + rcx*4], 500000
    jg .else
    inc rax
    jmp .next
.else:
    dec rax
.next:
    inc rcx
    jmp .loop

.done:
    ret
```

---

## 17. การใช้ perf สำหรับ Branch Analysis

### 17.1 perf stat Commands

```bash
# วัด overall branch statistics:
perf stat -e branches,branch-misses,branch-loads,branch-load-misses \
    ./your_program

# วัด per-function:
perf record -e branch-misses:pp -g ./your_program
perf report --stdio

# วัด cycle-accurate:
perf stat -e cycles,instructions,branches,branch-misses \
    -r 5 ./your_program  # run 5 times และเฉลี่ย

# Top-down analysis:
perf stat --topdown ./your_program

# ตัวอย่าง output ที่ดี:
# Performance counter stats for './my_program':
#
#    2,345,678,901   cycles                    # 2.346 GHz
#    3,456,789,012   instructions              # 1.47  insn per cycle
#      234,567,890   branches                  # 234.6 M/sec
#        1,234,567   branch-misses             # 0.53% of all branches
#
# ผล: 0.53% misprediction rate → ดีมาก!
# ถ้าเกิน 5% → ควรพิจารณา branchless techniques
```

### 17.2 perf annotate

```bash
# annotate source code ด้วย branch miss counts:
perf record -e branch-misses:pp ./your_program
perf annotate --stdio | head -100

# ตัวอย่าง output:
# Percent | Source code & Disassembly
# --------|----------------------------
#   0.01  |    cmp    [rbx+0x8], rax
#  47.32  |    jg     .L42              ← HOT MISPREDICTED BRANCH!
#   0.00  |    mov    rax, [rbx+0x10]
#   0.02  |  .L42:
```

### 17.3 Intel VTune

```bash
# Intel VTune Profiler (ถ้าติดตั้ง):
vtune -collect microarchitecture ./your_program
vtune -report hotspots

# หรือใช้ Intel Advisor สำหรับ SIMD recommendations:
advisor --collect=survey ./your_program
advisor --collect=tripcounts ./your_program
advisor --report=roofline
```

---

## 18. Case Study: Sorting Hot-Path

### 18.1 Optimizing QuickSort Partition

```asm
; ============================================================
; QuickSort Partition — branchless version
; ============================================================

section .text
global partition_branchless
; rdi = array (int64[])
; rsi = left index
; rdx = right index (inclusive)
; return: rax = partition index

partition_branchless:
    push rbx
    push r12
    push r13
    push r14
    push r15

    ; pivot = arr[right]
    mov r15, [rdi + rdx*8]   ; pivot

    mov rbx, rsi             ; i = left - 1
    dec rbx

    mov r12, rsi             ; j = left

.loop:
    cmp r12, rdx             ; j < right?
    jge .done_loop

    ; arr[j] <= pivot?
    mov r13, [rdi + r12*8]
    cmp r13, r15

    ; branchless swap:
    ; ถ้า arr[j] <= pivot: swap(arr[++i], arr[j])
    setle r14b
    movzx r14, r14b          ; r14 = (arr[j] <= pivot) ? 1 : 0

    ; inc i conditionally
    add rbx, r14             ; i += r14 (0 หรือ 1)

    ; swap arr[i] กับ arr[j] ถ้า condition true
    mov rax, [rdi + rbx*8]   ; arr[i]
    mov r8, [rdi + r12*8]    ; arr[j]

    ; สร้าง mask
    mov r9, r14
    neg r9                   ; mask = 0 หรือ -1

    ; เตรียม swapped values
    mov r10, rax
    mov r11, r8
    ; ถ้า r14=1: arr[i]=arr[j], arr[j]=arr[i] (original)
    ; ถ้า r14=0: ไม่เปลี่ยน

    ; branchless swap using XOR trick (ถ้า addresses ต่างกัน):
    ; เพิ่ม mask-based select:
    and r10, r9              ; arr[i] & mask
    and r11, r9              ; arr[j] & mask

    ; ง่ายกว่า: ใช้ cmov
    mov r10, rax
    cmp r14, 1
    cmove r10, r8            ; ถ้า r14=1: r10 = arr[j]
    cmove r8, rax            ; ถ้า r14=1: r8 = arr[i]
    cmove rax, r10

    mov [rdi + rbx*8], r10   ; write ไม่ว่าจะ swap หรือไม่

    ; ถ้า r14=0, เราเพิ่งเขียน arr[j] กลับไปที่ arr[i] (ค่าเดิม)
    ; และ rbx ไม่ได้เพิ่ม

    inc r12                  ; j++
    jmp .loop

.done_loop:
    ; swap arr[i+1] กับ arr[right]
    inc rbx                  ; i+1
    mov rax, [rdi + rbx*8]
    mov r8, [rdi + rdx*8]
    mov [rdi + rbx*8], r8
    mov [rdi + rdx*8], rax

    mov rax, rbx             ; return partition index

    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    ret
```

### 18.2 Benchmark: Branch vs Branchless Sort

```c
// ============================================================
// Complete benchmark program
// ============================================================

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <stdint.h>

#define N (1 << 20)   // 1M elements

// Branchless compare (as implemented in assembly earlier):
static int cmp_branchless(const void* a, const void* b) {
    int64_t x = *(const int64_t*)a;
    int64_t y = *(const int64_t*)b;
    return (x > y) - (x < y);
}

static int cmp_branch(const void* a, const void* b) {
    int64_t x = *(const int64_t*)a;
    int64_t y = *(const int64_t*)b;
    if (x < y) return -1;
    if (x > y) return  1;
    return 0;
}

uint64_t rdtsc() {
    uint32_t lo, hi;
    __asm__ volatile ("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

int main() {
    int64_t* data = malloc(N * sizeof(int64_t));
    int64_t* data_copy = malloc(N * sizeof(int64_t));

    srand(12345);
    for (int i = 0; i < N; i++) {
        data[i] = (int64_t)rand() * rand();
    }

    printf("Sorting %d elements (int64_t)\n\n", N);

    // Warm up cache:
    memcpy(data_copy, data, N * sizeof(int64_t));
    qsort(data_copy, N, sizeof(int64_t), cmp_branch);

    // Test 1: Branch comparator
    memcpy(data_copy, data, N * sizeof(int64_t));
    uint64_t t1 = rdtsc();
    qsort(data_copy, N, sizeof(int64_t), cmp_branch);
    uint64_t t2 = rdtsc();
    printf("Branch comparator:    %llu cycles (%.2f cycles/elem)\n",
           t2-t1, (double)(t2-t1)/N);

    // Test 2: Branchless comparator
    memcpy(data_copy, data, N * sizeof(int64_t));
    uint64_t t3 = rdtsc();
    qsort(data_copy, N, sizeof(int64_t), cmp_branchless);
    uint64_t t4 = rdtsc();
    printf("Branchless comparator: %llu cycles (%.2f cycles/elem)\n",
           t4-t3, (double)(t4-t3)/N);

    printf("\nSpeedup: %.2fx\n", (double)(t2-t1)/(t4-t3));

    free(data);
    free(data_copy);
    return 0;
}
```

---

## 19. GAS Syntax Examples

### 19.1 Branchless Min ใน GAS

```asm
# ============================================================
# GNU Assembler (GAS) syntax
# ============================================================
# Branchless Min using CMOV

    .text
    .globl branchless_min_gas
    .type  branchless_min_gas, @function

branchless_min_gas:
    # rdi = a, rsi = b
    # return min(a, b) ใน rax

    movq    %rdi, %rax      # rax = a
    cmpq    %rsi, %rax      # a vs b
    cmovgq  %rsi, %rax      # if a > b: rax = b
    ret

    .size branchless_min_gas, .-branchless_min_gas


    .globl branchless_abs_gas
    .type  branchless_abs_gas, @function

branchless_abs_gas:
    # rdi = x
    # return |x| ใน rax

    movq    %rdi, %rax
    movq    %rax, %rcx
    sarq    $63, %rcx       # mask = sign extension
    addq    %rcx, %rax      # x + mask
    xorq    %rcx, %rax      # (x + mask) XOR mask = |x|
    ret

    .size branchless_abs_gas, .-branchless_abs_gas


    .globl select_cmov_gas
    .type  select_cmov_gas, @function

select_cmov_gas:
    # rdi = cond, rsi = a, rdx = b
    # return: cond ? a : b

    movq    %rdx, %rax      # default = b
    testq   %rdi, %rdi
    cmovnzq %rsi, %rax      # if cond != 0: rax = a
    ret

    .size select_cmov_gas, .-select_cmov_gas
```

### 19.2 SIMD Branchless Comparison ใน GAS

```asm
# ============================================================
# AVX2 Branchless Clamp Array (GAS syntax)
# ============================================================

    .text
    .globl clamp_array_avx2_gas
    .type  clamp_array_avx2_gas, @function

clamp_array_avx2_gas:
    # rdi = int32_t* array
    # rsi = count (multiple of 8)
    # edx = lo (int32)
    # ecx = hi (int32)

    vmovd       %edx, %xmm2
    vpbroadcastd %xmm2, %ymm2      # ymm2 = [lo x8]

    vmovd       %ecx, %xmm3
    vpbroadcastd %xmm3, %ymm3      # ymm3 = [hi x8]

    xorl        %eax, %eax

.loop:
    vmovdqu     (%rdi,%rax,4), %ymm0    # load 8 ints
    vpmaxsd     %ymm2, %ymm0, %ymm0    # max(x, lo) — floor
    vpminsd     %ymm3, %ymm0, %ymm0    # min(x, hi) — ceiling
    vmovdqu     %ymm0, (%rdi,%rax,4)   # store 8 ints

    addl        $8, %eax
    cmpl        %esi, %eax
    jl          .loop

    vzeroupper
    ret

    .size clamp_array_avx2_gas, .-clamp_array_avx2_gas
```

---

## 20. Practical Guidelines สรุป

### 20.1 เมื่อใช้ Branchless Techniques

```
ใช้ Branchless เมื่อ:
✓ Branch misprediction rate > 5% (วัดด้วย perf)
✓ Branch อยู่ใน inner loop ที่ run ล้านครั้ง
✓ Data ไม่มี pattern ที่ predictor จะเรียนได้ (random/data-dependent)
✓ CMOV instruction ไม่สร้าง false dependency ที่แย่กว่า

ไม่ใช้ Branchless เมื่อ:
✗ Branch มี strong bias (>90% toward one path)
✗ Branchless code อ่านยากกว่ามาก โดยไม่มี gain
✗ CMOV สร้าง long dependency chain (latency แย่กว่า misprediction)
✗ Loop ถูก vectorized อยู่แล้ว (SIMD จะจัดการให้)
```

### 20.2 Checklist สำหรับ Branch Optimization

```
Branch Optimization Checklist:
□ 1. วัด misprediction rate ด้วย perf stat ก่อน
□ 2. ระบุ hot branches ด้วย perf annotate
□ 3. ตรวจสอบว่า data สามารถ sort ล่วงหน้าได้ไหม
□ 4. พิจารณา branchless version ด้วย CMOV/SETcc
□ 5. ใช้ __builtin_expect สำหรับ known-bias branches
□ 6. จัด branch order ให้ common case เป็น fall-through
□ 7. พิจารณา lookup table ถ้า branch pattern ซับซ้อน
□ 8. วัดผลหลังแก้ไข — อย่าเดา, วัดเสมอ!
```

### 20.3 Quick Reference: CMOV Instructions

```asm
; ============================================================
; Quick Reference: CMOV Instructions
; ============================================================

; Signed comparisons:
cmovl  dst, src   ; if less (SF ≠ OF)
cmovle dst, src   ; if less or equal (SF ≠ OF or ZF)
cmovg  dst, src   ; if greater (ZF=0 and SF=OF)
cmovge dst, src   ; if greater or equal (SF=OF)
cmove  dst, src   ; if equal (ZF=1)
cmovne dst, src   ; if not equal (ZF=0)

; Unsigned comparisons:
cmovb  dst, src   ; if below (CF=1)
cmovbe dst, src   ; if below or equal (CF=1 or ZF=1)
cmova  dst, src   ; if above (CF=0 and ZF=0)
cmovae dst, src   ; if above or equal (CF=0)

; Flag-based:
cmovs  dst, src   ; if sign (SF=1) — negative result
cmovns dst, src   ; if not sign (SF=0) — non-negative
cmovo  dst, src   ; if overflow (OF=1)
cmovno dst, src   ; if not overflow (OF=0)
cmovp  dst, src   ; if parity (PF=1)
cmovnp dst, src   ; if not parity (PF=0)
```

---

## 21. สรุป (Summary)

Branch prediction เป็นหัวใจสำคัญของ CPU performance สมัยใหม่
การเข้าใจและเพิ่มประสิทธิภาพ branches สามารถทำให้โค้ดเร็วขึ้นได้หลายเท่า

**Key Takeaways:**

1. **Branch misprediction penalty**: 15-20 cycles บน Sandy Bridge
   เพียงพอที่จะทำให้ code ช้าลง 5-10x ถ้า misprediction rate สูง

2. **Modern predictors** (TAGE) ฉลาดมาก — สามารถเรียน pattern ยาวได้
   แต่ random data ยังคง unpredictable

3. **CMOV** เป็น weapon หลักของ branchless programming
   ง่าย อ่านได้ ทำงานได้ดีบน hardware ทุกตัว

4. **วัดก่อนแก้** — ใช้ `perf stat branch-misses` เสมอ
   อย่าสมมติว่าโค้ดไหน hot โดยไม่วัด

5. **Spectre vulnerability** เชื่อมโยงกับ branch prediction โดยตรง
   mitigation (retpoline, lfence) มีต้นทุน performance จริง

```
"Branch prediction is the art of knowing the future.
 Branchless code is the art of not needing to."
                                    — Unknown
```

---

## อ้างอิง (References)

- Tse-Yu Yeh, Yale N. Patt. "Two-Level Adaptive Training Branch Prediction." MICRO 1991.
- André Seznec, Pierre Michaud. "A Case for (Partially) TAgged GEometric History Length Predictors." Journal of Instruction Level Parallelism, 2006.
- Intel 64 and IA-32 Architectures Optimization Reference Manual, Chapter 3.
- Agner Fog. "Optimizing software in C++." https://agner.org/optimize/
- Paul Kocher et al. "Spectre Attacks: Exploiting Speculative Execution." IEEE S&P 2019.
- Intel Architecture Code Analyzer (IACA) documentation.
- Linux perf documentation: https://perf.wiki.kernel.org/

---

*จบ Part 063: Branch Prediction Optimization*

*ต่อไป Part 064: Cache Optimization and Memory Access Patterns*
