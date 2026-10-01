# Part 038: AArch64 Architecture และ Instructions

## บทนำ (Introduction)

AArch64 คือ execution state 64-bit ของ ARM architecture ที่เปิดตัวพร้อมกับ ARMv8-A
เป็นการออกแบบใหม่ทั้งหมด ไม่ใช่แค่การขยาย ARM32 ให้กว้างขึ้น โดยมีการเปลี่ยนแปลง
สำคัญหลายอย่างที่ทำให้มีประสิทธิภาพสูงขึ้นและง่ายต่อการ compile

ปัจจุบัน AArch64 ใช้ใน:
- Apple Silicon (M1, M2, M3, M4)
- Android smartphones (Snapdragon, Exynos, Dimensity)
- AWS Graviton servers
- Raspberry Pi 4/5 (64-bit mode)
- Windows on ARM

---

## 1. AArch64 vs ARM32 - ความแตกต่างหลัก

### 1.1 ตารางเปรียบเทียบ (Comparison Table)

| Feature | ARM32 (AArch32) | AArch64 |
|---------|-----------------|---------|
| Register width | 32-bit | 64-bit |
| General registers | 16 (R0-R15) | 31 (X0-X30) |
| PC register | R15 (accessible) | Not directly accessible |
| SP register | R13 | SP (dedicated) |
| LR register | R14 | X30 |
| Condition codes | Most instructions | Separate compare instructions |
| SIMD registers | 16 x 128-bit (VFP/NEON) | 32 x 128-bit (NEON/SVE) |
| Instruction size | Variable (16/32-bit Thumb) | Fixed 32-bit |
| Addressing modes | Complex | Simplified |
| IT block | Yes | No |

### 1.2 ข้อดีของ AArch64 (Advantages)

```
1. Registers มากขึ้น: 31 vs 16 ลด memory access
2. Instructions ขนาดคงที่: 32-bit เสมอ (ง่ายต่อ decode)
3. No condition codes บน arithmetic: ลด dependencies
4. Improved calling convention: arguments ส่งผ่าน X0-X7
5. Better security: PAC (Pointer Authentication), BTI (Branch Target Identification)
6. SVE/SVE2: Scalable Vector Extension
```

---

## 2. Register File ใน AArch64

### 2.1 General-Purpose Registers (GPR)

AArch64 มี 31 general-purpose registers แต่ละ register สามารถใช้ได้ 2 ขนาด:

```
X0  - X30 : 64-bit registers (ใช้ prefix X)
W0  - W30 : 32-bit registers (ใช้ prefix W, lower 32 bits ของ X registers)
```

**หมายเหตุสำคัญ**: เมื่อเขียนค่าลง W register, upper 32 bits ของ X register
ที่สอดคล้องกันจะถูก zero-extend อัตโนมัติ

### 2.2 Special Registers

```asm
// XZR / WZR - Zero Register
// เมื่ออ่าน: ได้ค่า 0 เสมอ
// เมื่อเขียน: ค่าถูกทิ้ง (discard)
MOV X0, XZR        // X0 = 0
ADD X0, X1, XZR    // X0 = X1 + 0 = X1

// SP - Stack Pointer (dedicated, ไม่ใช่ general register)
// ต้องอยู่ใน 16-byte alignment เมื่อ call function
SUB SP, SP, #16    // allocate 16 bytes บน stack

// PC - Program Counter
// ไม่สามารถ read/write โดยตรง
// ใช้ ADR/ADRP/B/BL เพื่อ manipulate
```

### 2.3 System Registers

```asm
// NZCV - Condition Flags Register
// N = Negative flag
// Z = Zero flag
// C = Carry flag
// V = Overflow flag

// อ่าน/เขียน condition flags
MRS X0, NZCV       // Read NZCV -> X0
MSR NZCV, X0       // Write X0 -> NZCV

// SP_EL0, SP_EL1, SP_EL2, SP_EL3 - Stack pointers at each exception level
// ELR_EL1, ELR_EL2, ELR_EL3 - Exception Link Registers
// SPSR_EL1, SPSR_EL2, SPSR_EL3 - Saved Program Status Registers
```

### 2.4 Procedure Call Standard (AAPCS64)

```
Calling Convention สำหรับ AArch64:

Arguments:    X0-X7  (W0-W7 สำหรับ 32-bit args)
Return value: X0-X1  (X0 สำหรับ primary return value)
Scratch:      X9-X15 (Caller-saved, ไม่ต้อง preserve)
Preserved:    X19-X28 (Callee-saved, ต้อง save/restore)
Frame pointer: X29 (FP)
Link register: X30 (LR) - return address
Stack pointer: SP
```

---

## 3. NZCV Condition Flags

### 3.1 ความหมายของ Flags แต่ละตัว

```asm
// N - Negative Flag
// Set เมื่อผลลัพธ์เป็น negative (bit 63 = 1)
SUBS X0, X1, X2   // ถ้า X1 < X2 (signed), N = 1

// Z - Zero Flag
// Set เมื่อผลลัพธ์เป็น 0
SUBS X0, X1, X1   // X1 - X1 = 0, Z = 1

// C - Carry Flag
// Set เมื่อ unsigned overflow (carry out จาก bit 63)
ADDS X0, X1, X2   // ถ้า X1 + X2 > 2^64 - 1, C = 1

// V - Overflow Flag  
// Set เมื่อ signed overflow
ADDS X0, X1, X2   // ถ้า signed result ไม่พอดี 64-bit, V = 1
```

### 3.2 Condition Codes ที่ใช้กับ B.cond

```asm
// Condition codes หลัก
B.EQ label    // Branch if Equal (Z = 1)
B.NE label    // Branch if Not Equal (Z = 0)
B.CS label    // Branch if Carry Set / unsigned >=  (C = 1)
B.CC label    // Branch if Carry Clear / unsigned < (C = 0)
B.MI label    // Branch if Minus/Negative (N = 1)
B.PL label    // Branch if Plus/Non-negative (N = 0)
B.VS label    // Branch if Overflow Set (V = 1)
B.VC label    // Branch if Overflow Clear (V = 0)
B.HI label    // Branch if unsigned Higher (C = 1 && Z = 0)
B.LS label    // Branch if unsigned Lower or Same (C = 0 || Z = 1)
B.GE label    // Branch if signed >= (N = V)
B.LT label    // Branch if signed < (N != V)
B.GT label    // Branch if signed > (Z = 0 && N = V)
B.LE label    // Branch if signed <= (Z = 1 || N != V)
B.AL label    // Branch Always (unconditional)
```

### 3.3 Instructions ที่ Set Flags

```asm
// S suffix = set flags
ADDS X0, X1, X2   // Add and Set flags
SUBS X0, X1, X2   // Subtract and Set flags
ANDS X0, X1, X2   // AND and Set flags (Z, N; C=V=0)
ORRS X0, X1, X2   // ORR and Set flags

// Compare instructions (ผลลัพธ์ทิ้ง แต่ set flags)
CMP  X1, X2       // = SUBS XZR, X1, X2
CMN  X1, X2       // = ADDS XZR, X1, X2 (Compare Negative)
TST  X1, X2       // = ANDS XZR, X1, X2 (Test bits)
```

---

## 4. Arithmetic Instructions

### 4.1 ADD / ADDS / SUB / SUBS

```asm
// ==========================================
// ADD - Add
// ==========================================
ADD X0, X1, X2         // X0 = X1 + X2
ADD X0, X1, #100       // X0 = X1 + 100 (immediate, 12-bit)
ADD X0, X1, X2, LSL #3 // X0 = X1 + (X2 << 3)
ADD X0, X1, X2, LSR #2 // X0 = X1 + (X2 >> 2) unsigned
ADD X0, X1, X2, ASR #1 // X0 = X1 + (X2 >> 1) signed

// ADD extended register (สำหรับ address arithmetic)
ADD X0, X1, W2, UXTB   // X0 = X1 + zero-extend(W2[7:0])
ADD X0, X1, W2, SXTB   // X0 = X1 + sign-extend(W2[7:0])
ADD X0, X1, W2, UXTW   // X0 = X1 + zero-extend(W2)
ADD X0, X1, W2, SXTW   // X0 = X1 + sign-extend(W2)
ADD X0, X1, X2, UXTX   // X0 = X1 + X2 (no extension)

// ADDS - Add and Set Flags
ADDS X0, X1, X2        // X0 = X1 + X2, set NZCV

// SUB - Subtract
SUB  X0, X1, X2        // X0 = X1 - X2
SUB  X0, X1, #50       // X0 = X1 - 50
SUBS X0, X1, X2        // X0 = X1 - X2, set NZCV

// NEG - Negate (= SUB Xd, XZR, Xn)
NEG  X0, X1            // X0 = -X1
NEGS X0, X1            // X0 = -X1, set flags
```

### 4.2 MUL / MADD / MSUB / MNEG

```asm
// MUL - Multiply (= MADD Xd, Xn, Xm, XZR)
MUL  X0, X1, X2        // X0 = X1 * X2 (lower 64 bits)
MUL  W0, W1, W2        // W0 = W1 * W2 (32-bit)

// MADD - Multiply-Add
MADD X0, X1, X2, X3    // X0 = X1 * X2 + X3

// MSUB - Multiply-Subtract
MSUB X0, X1, X2, X3    // X0 = X3 - X1 * X2

// MNEG - Multiply-Negate (= MSUB Xd, Xn, Xm, XZR)
MNEG X0, X1, X2        // X0 = -(X1 * X2)

// SMULL - Signed Multiply Long (32 x 32 = 64)
SMULL X0, W1, W2       // X0 = (signed)W1 * (signed)W2

// UMULL - Unsigned Multiply Long
UMULL X0, W1, W2       // X0 = (unsigned)W1 * (unsigned)W2

// SMULH - Signed Multiply High (upper 64 bits ของ 128-bit result)
SMULH X0, X1, X2       // X0 = upper64(X1 * X2) signed

// UMULH - Unsigned Multiply High
UMULH X0, X1, X2       // X0 = upper64(X1 * X2) unsigned
```

### 4.3 UDIV / SDIV

```asm
// UDIV - Unsigned Divide
UDIV X0, X1, X2        // X0 = X1 / X2 (unsigned, truncate toward 0)
UDIV W0, W1, W2        // 32-bit unsigned divide

// SDIV - Signed Divide
SDIV X0, X1, X2        // X0 = X1 / X2 (signed, truncate toward 0)
SDIV W0, W1, W2        // 32-bit signed divide

// หมายเหตุ: AArch64 ไม่มี MOD instruction โดยตรง
// คำนวณ modulo ด้วย:
// remainder = dividend - (quotient * divisor)
SDIV X2, X0, X1        // X2 = X0 / X1 (quotient)
MSUB X3, X2, X1, X0   // X3 = X0 - X2 * X1 (remainder)
```

---

## 5. Load/Store Instructions

### 5.1 LDR / STR - Load/Store Register

```asm
// LDR - Load Register
LDR  X0, [X1]          // X0 = Memory[X1] (64-bit)
LDR  W0, [X1]          // W0 = Memory[X1] (32-bit, zero-extend)
LDRH W0, [X1]          // W0 = Memory[X1] (16-bit, zero-extend)
LDRB W0, [X1]          // W0 = Memory[X1] (8-bit, zero-extend)

// Signed load variants
LDRSW X0, [X1]         // X0 = sign-extend(Memory32[X1])
LDRSH X0, [X1]         // X0 = sign-extend(Memory16[X1])
LDRSB X0, [X1]         // X0 = sign-extend(Memory8[X1])

// STR - Store Register
STR  X0, [X1]          // Memory[X1] = X0 (64-bit)
STR  W0, [X1]          // Memory[X1] = W0 (32-bit)
STRH W0, [X1]          // Memory[X1] = W0[15:0] (16-bit)
STRB W0, [X1]          // Memory[X1] = W0[7:0] (8-bit)
```

### 5.2 LDP / STP - Load/Store Pair

```asm
// LDP - Load Pair of Registers
LDP X0, X1, [X2]       // X0 = Mem[X2], X1 = Mem[X2+8]
LDP W0, W1, [X2]       // W0 = Mem[X2], W1 = Mem[X2+4]
LDP X0, X1, [X2, #16]  // X0 = Mem[X2+16], X1 = Mem[X2+24]

// STP - Store Pair of Registers
STP X0, X1, [X2]       // Mem[X2] = X0, Mem[X2+8] = X1
STP W0, W1, [X2]       // Mem[X2] = W0, Mem[X2+4] = W1
STP X29, X30, [SP, #-16]! // Push frame pointer and link register

// สำคัญมาก: LDP/STP ใช้บ่อยมากใน function prologue/epilogue
// Function prologue (save registers):
STP X29, X30, [SP, #-16]!  // SP -= 16, save X29,X30
MOV X29, SP                // set frame pointer

// Function epilogue (restore registers):
LDP X29, X30, [SP], #16    // restore X29,X30, SP += 16
RET                        // return (branch to X30)
```

### 5.3 Addressing Modes ใน AArch64

```asm
// =============================================
// 1. Base Register (Register Offset = 0)
// =============================================
LDR X0, [X1]           // Address = X1
STR X0, [X1]

// =============================================
// 2. Base + Immediate Offset (Unsigned)
// =============================================
LDR X0, [X1, #8]       // Address = X1 + 8
LDR X0, [X1, #-8]      // Address = X1 - 8 (signed offset)
STR X0, [X1, #16]      // Address = X1 + 16

// =============================================
// 3. Base + Register Offset
// =============================================
LDR X0, [X1, X2]       // Address = X1 + X2
LDR X0, [X1, X2, LSL #3]  // Address = X1 + (X2 << 3) = X1 + X2*8
LDR X0, [X1, W2, UXTW #3] // Address = X1 + zero-extend(W2) << 3

// =============================================
// 4. Pre-Index (update address BEFORE access)
// =============================================
LDR X0, [X1, #8]!      // X1 = X1 + 8 FIRST, then X0 = Mem[X1]
STR X0, [X1, #-16]!    // X1 = X1 - 16 FIRST, then Mem[X1] = X0

// =============================================
// 5. Post-Index (update address AFTER access)
// =============================================
LDR X0, [X1], #8       // X0 = Mem[X1] FIRST, then X1 = X1 + 8
STR X0, [X1], #16      // Mem[X1] = X0 FIRST, then X1 = X1 + 16

// =============================================
// 6. PC-Relative (Literal)
// =============================================
LDR X0, =constant      // Pseudo-instruction, assembler จัดการ
LDR X0, label          // X0 = Mem[PC + offset] (offset คำนวณจาก label)
```

---

## 6. ADR และ ADRP

### 6.1 ADR - Address (PC-relative, ±1MB)

```asm
// ADR - Load Address (PC + signed 21-bit offset)
// ใช้สำหรับ addresses ภายใน ±1MB จาก instruction นี้
ADR X0, my_label       // X0 = address ของ my_label

// ตัวอย่างการใช้งาน
.section .data
message: .asciz "Hello, AArch64!\n"

.section .text
.global _start
_start:
    ADR X0, message     // X0 = address ของ string
    // ...
```

### 6.2 ADRP - Address of Page (PC-relative, ±4GB)

```asm
// ADRP - Load 4KB page Address
// ใช้ 12-bit immediate + shift 12 (เทียบกับ page ที่ PC อยู่)
// ใช้คู่กับ ADD หรือ LDR เพื่อ access 64-bit address space

// Pattern มาตรฐาน (position-independent code):
ADRP X0, my_var         // X0 = page address ของ my_var
ADD  X0, X0, :lo12:my_var  // X0 += page offset ของ my_var

// หรือสำหรับ load:
ADRP X0, my_var
LDR  X1, [X0, :lo12:my_var]  // X1 = value ของ my_var

// หรือสำหรับ store:
ADRP X0, my_var
STR  X1, [X0, :lo12:my_var]  // my_var = X1

// ตัวอย่างจริง: load global variable
.section .data
.align 3
counter: .quad 0         // 64-bit counter

.section .text
increment_counter:
    ADRP X0, counter
    LDR  X1, [X0, :lo12:counter]   // โหลดค่าปัจจุบัน
    ADD  X1, X1, #1                 // เพิ่มค่า
    STR  X1, [X0, :lo12:counter]   // เก็บค่าใหม่
    RET
```

---

## 7. Branch Instructions

### 7.1 B, BL, BR, BLR, RET

```asm
// B - Branch (unconditional, PC-relative ±128MB)
B label                 // PC = label

// BL - Branch with Link (call function, save return address to X30)
BL function_name        // X30 = PC+4, PC = function_name

// BR - Branch to Register
BR X0                   // PC = X0

// BLR - Branch with Link to Register
BLR X0                  // X30 = PC+4, PC = X0

// RET - Return (branch to X30, or specified register)
RET                     // PC = X30
RET X29                 // PC = X29 (unusual)

// B.cond - Conditional Branch (PC-relative ±1MB)
B.EQ equal_label        // branch if Z=1
B.NE not_equal_label    // branch if Z=0
B.GT greater_label      // branch if signed >
B.LT less_label         // branch if signed <
```

### 7.2 CBZ / CBNZ - Compare and Branch

```asm
// CBZ - Compare and Branch if Zero
// ไม่ต้อง CMP + B.EQ สองขั้นตอน
CBZ  X0, label          // if X0 == 0, branch to label
CBZ  W0, label          // if W0 == 0, branch to label (32-bit)

// CBNZ - Compare and Branch if Non-Zero
CBNZ X0, label          // if X0 != 0, branch to label
CBNZ W0, label          // if W0 != 0, branch to label

// ตัวอย่าง: ตรวจสอบ null pointer
// C code: if (ptr != NULL) { ... }
CBNZ X0, not_null       // if X0 != 0 (not NULL), skip
B    handle_null        // ptr is NULL
not_null:
    // use ptr...

// ตัวอย่าง: loop countdown
    MOV X0, #10         // counter = 10
loop:
    // ... body ...
    SUBS X0, X0, #1     // counter--
    CBNZ X0, loop       // if counter != 0, continue
```

### 7.3 TBZ / TBNZ - Test Bit and Branch

```asm
// TBZ - Test Bit and Branch if Zero
// ตรวจสอบ specific bit และ branch
TBZ  X0, #0, label      // if bit 0 of X0 == 0, branch
TBZ  X0, #63, label     // if bit 63 of X0 == 0, branch
TBZ  W0, #31, label     // if bit 31 of W0 == 0, branch

// TBNZ - Test Bit and Branch if Non-Zero
TBNZ X0, #0, label      // if bit 0 of X0 != 0, branch
TBNZ X0, #7, label      // if bit 7 of X0 != 0, branch

// ตัวอย่าง: ตรวจสอบ even/odd
TBZ W0, #0, even_label  // if bit 0 = 0, number is even
// odd number code here
B   done
even_label:
// even number code here
done:

// ตัวอย่าง: ตรวจสอบ flag bits
// สมมติ X0 = flags register
TBNZ X0, #5, flag5_set  // if bit 5 set, handle
```

---

## 8. System Instructions

### 8.1 MRS / MSR - Move to/from System Register

```asm
// MRS - Move Register from System register
// MRS Xt, <system_register>
MRS X0, NZCV            // Read condition flags
MRS X0, DAIF            // Read interrupt mask bits (D,A,I,F)
MRS X0, CurrentEL       // Read current exception level
MRS X0, SP_EL0          // Read EL0 stack pointer
MRS X0, ELR_EL1         // Read exception link register EL1
MRS X0, SPSR_EL1        // Read saved program status EL1
MRS X0, TPIDR_EL0       // Read thread ID register (user space)
MRS X0, CNTP_TVAL_EL0   // Read physical timer value
MRS X0, CNTPCT_EL0      // Read physical count
MRS X0, MPIDR_EL1       // Read Multiprocessor ID register (core ID)

// MSR - Move to System register from Register
// MSR <system_register>, Xt/Xn
MSR NZCV, X0            // Write condition flags
MSR DAIF, X0            // Write interrupt mask
MSR SP_EL0, X0          // Write EL0 stack pointer
MSR TPIDR_EL0, X0       // Write thread ID (user space)

// MSR immediate form (สำหรับบาง registers)
MSR DAIFSet, #0xF       // Set D,A,I,F bits (disable all interrupts)
MSR DAIFClr, #0x3       // Clear I,F bits (enable IRQ, FIQ)
```

### 8.2 SVC - Supervisor Call (System Call)

```asm
// SVC - Supervisor Call
// เรียก OS service จาก user space (EL0 -> EL1)
// immediate 16-bit เป็น comment/hint สำหรับ OS
SVC #0                  // Linux system call

// Linux AArch64 System Call Convention:
// X8 = system call number
// X0-X5 = arguments
// X0 = return value (หลัง SVC)

// ตัวอย่าง: write system call
MOV X0, #1              // fd = stdout
ADR X1, message         // buf = message
MOV X2, #14            // len = 14
MOV X8, #64            // syscall = write (NR_write = 64)
SVC #0                  // เรียก kernel

// ตัวอย่าง: exit system call
MOV X0, #0              // exit code = 0
MOV X8, #93            // syscall = exit (NR_exit = 93)
SVC #0

// ตัวอย่าง: read system call
MOV X0, #0              // fd = stdin
ADR X1, buffer          // buf
MOV X2, #256           // count
MOV X8, #63            // syscall = read (NR_read = 63)
SVC #0                  // X0 = bytes read หรือ -errno
```

### 8.3 BRK - Breakpoint

```asm
// BRK - Breakpoint Exception
// Generate debug exception
BRK #0                  // Breakpoint, imm16 = 0
BRK #0xDEAD             // Breakpoint with immediate value

// ใช้ใน debug:
// - GDB จะหยุด execution ที่นี่
// - OS จะส่ง SIGTRAP ถ้าไม่มี debugger

// ใน production code อาจใช้เป็น assertion:
CMP X0, #0
B.NE no_error
BRK #1                  // assertion failed!
no_error:
```

### 8.4 HLT - Halt

```asm
// HLT - Halt instruction
// Generate Halting debug event (ใน debug state)
HLT #0                  // Halt, สำหรับ hardware debugger

// ต่างจาก BRK:
// BRK = software debug exception
// HLT = halt-mode debug event (ต้องการ external debugger)
```

### 8.5 ERET - Exception Return

```asm
// ERET - Exception Return
// ใช้ใน exception handlers เพื่อ return จาก exception
// PC = ELR_ELn, PSTATE = SPSR_ELn

// ตัวอย่าง: simple exception handler ที่ EL1
el1_handler:
    // ... handle exception ...
    
    // Restore return address and state
    // (ELR_EL1 และ SPSR_EL1 ถูก save โดย hardware)
    ERET                // กลับไป exception origin

// ตัวอย่าง: เปลี่ยน exception level
// เพื่อ jump to EL0 (เช่นใน bootloader/hypervisor)
setup_el0_entry:
    ADR  X0, el0_entry_point
    MSR  ELR_EL1, X0   // Return address = el0_entry_point
    
    MOV  X0, #0x0      // PSTATE: EL0, SP_EL0, AArch64
    MSR  SPSR_EL1, X0
    
    ERET               // Jump to EL0
```

---

## 9. Bit Manipulation Instructions

### 9.1 UBFX / SBFX - Bit Field Extract

```asm
// UBFX - Unsigned Bit Field eXtract
// UBFX Xd, Xn, #lsb, #width
// Xd = zero-extend(Xn[lsb + width - 1 : lsb])
UBFX X0, X1, #8, #8    // X0 = bits [15:8] ของ X1 (zero-extended)
UBFX X0, X1, #0, #4    // X0 = bits [3:0] ของ X1 (lower nibble)
UBFX W0, W1, #16, #8   // W0 = bits [23:16] ของ W1

// SBFX - Signed Bit Field eXtract
// Xd = sign-extend(Xn[lsb + width - 1 : lsb])
SBFX X0, X1, #8, #8    // X0 = sign-extend(bits [15:8] of X1)
SBFX X0, X1, #0, #16   // X0 = sign-extend(lower 16 bits of X1)

// ตัวอย่างใช้งาน: แยก fields จาก packed structure
// สมมติ X0 = packed_value = [flags:4][size:12][type:8][version:8]
UBFX X1, X0, #0, #8    // X1 = version field (bits 7:0)
UBFX X2, X0, #8, #8    // X2 = type field (bits 15:8)
UBFX X3, X0, #16, #12  // X3 = size field (bits 27:16)
UBFX X4, X0, #28, #4   // X4 = flags field (bits 31:28)

// ตัวอย่าง: ดึง red channel จาก RGB888
// X0 = 0x00RRGGBB
UBFX X1, X0, #16, #8   // X1 = red channel
UBFX X2, X0, #8, #8    // X2 = green channel
UBFX X3, X0, #0, #8    // X3 = blue channel
```

### 9.2 UBFIZ / SBFIZ - Bit Field Insert into Zero

```asm
// UBFIZ - Unsigned Bit Field Insert in Zero
// Xd = zero; Xd[lsb + width - 1 : lsb] = Xn[width-1:0]
UBFIZ X0, X1, #8, #8   // X0 = X1[7:0] ใส่ที่ bit position 8

// BFI - Bit Field Insert
// Xd[lsb + width - 1 : lsb] = Xn[width-1:0] (ส่วนอื่นไม่เปลี่ยน)
BFI X0, X1, #8, #8     // ใส่ 8 bits จาก X1 ที่ X0[15:8]

// BFC - Bit Field Clear
// Xd[lsb + width - 1 : lsb] = 0
BFC X0, #8, #8          // X0[15:8] = 0
```

### 9.3 REV / REV16 / REV32 - Byte Reverse

```asm
// REV - Reverse byte order (byte swap ทั้ง register)
REV  X0, X1             // Reverse bytes ใน 64-bit register
                        // 0x0102030405060708 -> 0x0807060504030201
REV  W0, W1             // Reverse bytes ใน 32-bit register
                        // 0x01020304 -> 0x04030201

// REV16 - Reverse bytes within each 16-bit halfword
REV16 X0, X1            // Swap bytes ใน แต่ละ 16-bit chunk
                        // [B0B1|B2B3|B4B5|B6B7] -> [B1B0|B3B2|B5B4|B7B6]
REV16 W0, W1            // 32-bit version

// REV32 - Reverse bytes within each 32-bit word
REV32 X0, X1            // Swap bytes ใน แต่ละ 32-bit chunk (64-bit only)
                        // [B0B1B2B3|B4B5B6B7] -> [B3B2B1B0|B7B6B5B4]

// ตัวอย่างใช้งาน: Network byte order (big-endian) conversion
// Little-endian CPU อ่านค่า 32-bit จาก network
LDR  W0, [X1]           // โหลด 4 bytes
REV  W0, W0             // แปลง big-endian -> little-endian
// ตอนนี้ W0 เป็น host byte order

// แปลงกลับก่อนส่ง:
REV  W0, W0             // little-endian -> big-endian
STR  W0, [X1]           // เก็บลง network buffer
```

### 9.4 CLS / CLZ / RBIT

```asm
// CLZ - Count Leading Zeros
CLZ  X0, X1             // X0 = number of leading zeros ใน X1
CLZ  W0, W1             // 32-bit version

// ตัวอย่าง:
// X1 = 0x0000000000000001 -> CLZ = 63
// X1 = 0x8000000000000000 -> CLZ = 0
// X1 = 0x0000000000000000 -> CLZ = 64

// ใช้คำนวณ floor(log2(x)):
// log2(x) = 63 - CLZ(x) สำหรับ x > 0
CLZ  X1, X0             // X1 = leading zeros
MOV  X2, #63
SUB  X2, X2, X1         // X2 = 63 - CLZ = floor(log2(X0))

// CLS - Count Leading Sign bits
// นับ bits หลังจาก leading bit ที่เหมือนกับ sign bit
CLS  X0, X1             // X0 = number of redundant sign bits
CLS  W0, W1

// ตัวอย่าง:
// X1 = 0x000000000000001F -> CLS = 58 (positive, leading 58 zeros after sign)
// X1 = 0xFFFFFFFFFFFFFF80 -> CLS = 56 (negative, leading 56 ones after sign)

// ใช้สำหรับ: normalization ใน floating-point emulation

// RBIT - Reverse Bits
RBIT X0, X1             // X0 = X1 with all bits reversed
RBIT W0, W1

// ตัวอย่าง:
// X1 = 0x0000000000000001 -> RBIT -> 0x8000000000000000
// X1 = 0x0102030405060708 -> RBIT -> bit-reversed version
```

---

## 10. Logical Instructions

### 10.1 AND / ORR / EOR / BIC / EON / ORN

```asm
// AND - Bitwise AND
AND  X0, X1, X2         // X0 = X1 & X2
AND  X0, X1, #0xFF      // X0 = X1 & 0xFF (mask lower 8 bits)
ANDS X0, X1, X2         // AND and set flags

// ORR - Bitwise OR (inclusive OR)
ORR  X0, X1, X2         // X0 = X1 | X2
ORR  X0, X1, #0x100     // X0 = X1 | 0x100 (set bit 8)

// MOV เป็น alias ของ ORR:
MOV  X0, X1             // = ORR X0, XZR, X1
MOV  X0, #100           // = ORR X0, XZR, #100

// EOR - Exclusive OR
EOR  X0, X1, X2         // X0 = X1 ^ X2
EOR  X0, X1, #0xFF      // XOR lower 8 bits

// BIC - Bit Clear (AND NOT)
BIC  X0, X1, X2         // X0 = X1 & (~X2)
BIC  X0, X1, #0xFF      // Clear lower 8 bits ของ X1

// ORN - OR NOT
ORN  X0, X1, X2         // X0 = X1 | (~X2)
MVN  X0, X1             // = ORN X0, XZR, X1 (bitwise NOT)

// EON - Exclusive OR NOT
EON  X0, X1, X2         // X0 = X1 ^ (~X2)
```

---

## 11. Shift และ Rotate Instructions

### 11.1 LSL / LSR / ASR / ROR

```asm
// ใน AArch64, shift เป็น modifier ของ instructions อื่น
// แต่สามารถใช้เป็น standalone ได้ (เป็น alias)

// LSL - Logical Shift Left
LSL  X0, X1, #3         // X0 = X1 << 3 (multiply by 8)
LSL  X0, X1, X2         // X0 = X1 << X2[5:0]

// LSR - Logical Shift Right (unsigned)
LSR  X0, X1, #3         // X0 = X1 >> 3 (unsigned divide by 8)
LSR  X0, X1, X2         // X0 = X1 >> X2[5:0]

// ASR - Arithmetic Shift Right (signed)
ASR  X0, X1, #3         // X0 = X1 >> 3 (signed divide by 8)
ASR  X0, X1, X2         // X0 = X1 >> X2[5:0] (sign-extended)

// ROR - Rotate Right
ROR  X0, X1, #8         // Rotate right by 8 bits
ROR  X0, X1, X2         // Rotate right by X2[5:0] bits

// ตัวอย่างการใช้งาน:
// คำนวณ X1 * 40 = X1 * 32 + X1 * 8
LSL  X2, X1, #5         // X2 = X1 * 32
ADD  X0, X2, X1, LSL #3 // X0 = X2 + X1 * 8 = X1 * 40
```

---

## 12. Move Instructions

### 12.1 MOV / MOVZ / MOVK / MOVN

```asm
// MOV - Move (general alias)
MOV  X0, X1             // X0 = X1 (register to register)
MOV  X0, #100           // X0 = 100 (small immediate)

// MOVZ - Move with Zero (load 16-bit immediate, zero other bits)
MOVZ X0, #0x1234        // X0 = 0x0000000000001234
MOVZ X0, #0x1234, LSL #16  // X0 = 0x0000000012340000
MOVZ X0, #0x1234, LSL #32  // X0 = 0x0000123400000000
MOVZ X0, #0x1234, LSL #48  // X0 = 0x1234000000000000

// MOVK - Move with Keep (load 16-bit immediate, keep other bits)
// ใช้สร้าง large constants ด้วยหลาย instructions
MOVZ X0, #0x5678        // X0 = 0x0000000000005678
MOVK X0, #0x1234, LSL #16  // X0 = 0x0000000012345678
MOVK X0, #0xABCD, LSL #32  // X0 = 0x0000ABCD12345678
MOVK X0, #0xDEAD, LSL #48  // X0 = 0xDEADABCD12345678

// MOVN - Move with NOT (load 16-bit immediate, invert)
MOVN X0, #0             // X0 = 0xFFFFFFFFFFFFFFFF = -1
MOVN X0, #1             // X0 = 0xFFFFFFFFFFFFFFFE = -2
```

---

## 13. ตัวอย่างโปรแกรมจริง (Complete Programs)

### 13.1 Hello World - AArch64 Linux

```asm
// hello_aarch64.s
// รัน: as -o hello.o hello_aarch64.s && ld -o hello hello.o
// หรือ: aarch64-linux-gnu-as -o hello.o hello_aarch64.s
//        aarch64-linux-gnu-ld -o hello hello.o

.section .data
message:
    .ascii "สวัสดี AArch64!\n"  // Thai greeting
msg_len = . - message            // คำนวณ length

.section .text
.global _start

_start:
    // write(stdout, message, msg_len)
    MOV X0, #1              // fd = 1 (stdout)
    ADR X1, message         // buf = &message
    MOV X2, #msg_len        // len = msg_len
    MOV X8, #64             // syscall number: write
    SVC #0                  // เรียก kernel

    // exit(0)
    MOV X0, #0              // exit code = 0
    MOV X8, #93             // syscall number: exit
    SVC #0
```

### 13.2 Function Call และ Stack Frame

```asm
// stack_frame.s
// ตัวอย่าง: function ที่มี stack frame ถูกต้อง

.section .text
.global add_three_numbers

// int64_t add_three_numbers(int64_t a, int64_t b, int64_t c)
// Arguments: X0=a, X1=b, X2=c
// Return: X0 = a + b + c
add_three_numbers:
    // Prologue: สร้าง stack frame
    STP X29, X30, [SP, #-16]!  // save frame pointer and link register
    MOV X29, SP                 // set frame pointer

    // Body: คำนวณ
    ADD X0, X0, X1              // X0 = a + b
    ADD X0, X0, X2              // X0 = (a + b) + c

    // Epilogue: ทำลาย stack frame
    LDP X29, X30, [SP], #16    // restore frame pointer and link register
    RET                         // กลับไป caller

// complex_function: function ที่ใช้ callee-saved registers
// int64_t complex_calc(int64_t x)
complex_calc:
    STP X29, X30, [SP, #-48]!  // save FP, LR
    MOV X29, SP
    STP X19, X20, [SP, #16]    // save X19, X20 (callee-saved)
    STP X21, X22, [SP, #32]    // save X21, X22

    MOV X19, X0                 // เก็บ argument ใน preserved register
    
    // เรียก function อื่น (X30 ถูก overwrite)
    MOV X0, #5
    BL  some_other_function     // X0 = some_other_function(5)
    
    // X19 ยังมีค่าเดิม (ไม่ถูก overwrite)
    MUL X20, X19, X0            // X20 = x * result
    
    MOV X0, X20                 // set return value

    LDP X21, X22, [SP, #32]    // restore X21, X22
    LDP X19, X20, [SP, #16]    // restore X19, X20
    LDP X29, X30, [SP], #48    // restore FP, LR
    RET
```

### 13.3 Array Operations

```asm
// array_ops.s
// ตัวอย่าง: operations บน array

.section .data
.align 3
numbers: .quad 5, 3, 8, 1, 9, 2, 7, 4, 6, 0  // 10 elements
count:   .quad 10

.section .text
.global array_sum

// int64_t array_sum(int64_t *arr, int64_t n)
// X0 = pointer to array, X1 = count
// Returns: X0 = sum of all elements
array_sum:
    CBZ  X1, sum_return_zero  // if n == 0, return 0
    
    MOV  X2, #0              // sum = 0
    MOV  X3, #0              // index = 0

sum_loop:
    LDR  X4, [X0, X3, LSL #3]  // X4 = arr[index] (8 bytes per element)
    ADD  X2, X2, X4             // sum += arr[index]
    ADD  X3, X3, #1             // index++
    CMP  X3, X1                 // compare index with count
    B.LT sum_loop               // if index < count, continue

    MOV  X0, X2                 // return sum
    RET

sum_return_zero:
    MOV  X0, #0
    RET

// void array_reverse(int64_t *arr, int64_t n)
// X0 = pointer, X1 = count
array_reverse:
    STP  X29, X30, [SP, #-32]!
    MOV  X29, SP
    STP  X19, X20, [SP, #16]

    MOV  X19, X0                // save arr pointer
    SUB  X20, X1, #1            // right = n - 1
    MOV  X2, #0                 // left = 0

reverse_loop:
    CMP  X2, X20
    B.GE reverse_done           // if left >= right, done

    // swap arr[left] and arr[right]
    LDR  X3, [X19, X2, LSL #3]  // X3 = arr[left]
    LDR  X4, [X19, X20, LSL #3] // X4 = arr[right]
    STR  X4, [X19, X2, LSL #3]  // arr[left] = X4
    STR  X3, [X19, X20, LSL #3] // arr[right] = X3

    ADD  X2, X2, #1             // left++
    SUB  X20, X20, #1           // right--
    B    reverse_loop

reverse_done:
    LDP  X19, X20, [SP, #16]
    LDP  X29, X30, [SP], #32
    RET
```

### 13.4 String Operations

```asm
// string_ops.s
// ตัวอย่าง: string manipulation

.section .text

// int64_t strlen_asm(const char *s)
// X0 = pointer to null-terminated string
// Returns: X0 = length (not including null terminator)
strlen_asm:
    MOV  X1, X0              // save start pointer
    
strlen_loop:
    LDRB W2, [X0], #1        // load byte, post-increment pointer
    CBNZ W2, strlen_loop     // if not null, continue
    
    SUB  X0, X0, X1          // length = end - start
    SUB  X0, X0, #1          // subtract null byte
    RET

// void strcpy_asm(char *dst, const char *src)
// X0 = destination, X1 = source
strcpy_asm:
strcpy_loop:
    LDRB W2, [X1], #1        // load byte from src, advance src
    STRB W2, [X0], #1        // store byte to dst, advance dst
    CBNZ W2, strcpy_loop     // if not null, continue
    RET

// int strcmp_asm(const char *s1, const char *s2)
// X0 = s1, X1 = s2
// Returns: 0 if equal, <0 if s1 < s2, >0 if s1 > s2
strcmp_asm:
strcmp_loop:
    LDRB W2, [X0], #1        // load char from s1
    LDRB W3, [X1], #1        // load char from s2
    CBZ  W2, strcmp_check    // if s1[i] == 0, check if equal
    CMP  W2, W3
    B.EQ strcmp_loop         // if equal, next char
    
strcmp_check:
    SUB  W0, W2, W3          // return s1[i] - s2[i]
    RET
```

### 13.5 NZCV Flags Demo

```asm
// flags_demo.s
// แสดงการทำงานของ NZCV flags

.section .text
.global _start

_start:
    // =====================
    // Test Z flag (zero)
    // =====================
    MOV  X0, #5
    SUBS X1, X0, X0         // 5 - 5 = 0, Z=1
    B.EQ zero_case
    B    not_zero_case

zero_case:
    // Z flag is set
    B    continue1
not_zero_case:
    // Z flag is clear
continue1:

    // =====================
    // Test N flag (negative)
    // =====================
    MOV  X0, #3
    MOV  X1, #10
    SUBS X2, X0, X1         // 3 - 10 = -7, N=1
    B.MI negative_case
    B    positive_case

negative_case:
    // N flag is set (result is negative)
    B    continue2
positive_case:
    // N flag is clear
continue2:

    // =====================
    // Test C flag (carry)
    // =====================
    MOV  X0, #0xFFFFFFFFFFFFFFFF  // max uint64
    ADDS X1, X0, #1          // overflow! C=1, Z=1
    B.CS carry_case
    B    no_carry_case

carry_case:
    // C flag is set (unsigned overflow)
    B    continue3
no_carry_case:
    // C flag is clear
continue3:

    // =====================
    // Test V flag (overflow)
    // =====================
    MOV  X0, #0x7FFFFFFFFFFFFFFF  // max int64
    ADDS X1, X0, #1          // signed overflow! V=1
    B.VS overflow_case
    B    no_overflow_case

overflow_case:
    // V flag is set (signed overflow)
    B    continue4
no_overflow_case:
    // V flag is clear
continue4:

    // exit
    MOV X0, #0
    MOV X8, #93
    SVC #0
```

### 13.6 Bit Manipulation Example

```asm
// bitmanip.s
// ตัวอย่าง: bit manipulation ด้วย UBFX, REV, CLZ, RBIT

.section .text
.global _start

_start:
    // =====================
    // UBFX example
    // =====================
    // สมมติ X0 เป็น packed pixel: [A:8][R:8][G:8][B:8]
    MOV  X0, #0xFFAA8844     // A=FF, R=AA, G=88, B=44

    UBFX X1, X0, #0,  #8    // X1 = B = 0x44
    UBFX X2, X0, #8,  #8    // X2 = G = 0x88
    UBFX X3, X0, #16, #8    // X3 = R = 0xAA
    UBFX X4, X0, #24, #8    // X4 = A = 0xFF

    // =====================
    // REV example: network to host byte order
    // =====================
    MOV  W0, #0x01020304     // network byte order
    REV  W0, W0              // host byte order: 0x04030201

    // =====================
    // CLZ example: find highest set bit
    // =====================
    MOV  X0, #0x00000100     // bit 8 is highest
    CLZ  X1, X0              // X1 = 55 (64 - 8 - 1)
    MOV  X2, #63
    SUB  X2, X2, X1          // X2 = 8 = floor(log2(256))

    // =====================
    // RBIT example
    // =====================
    MOV  X0, #0x0F0F0F0F     // pattern
    RBIT X1, X0              // bit-reversed: 0xF0F0F0F000000000

    // =====================
    // CLS example
    // =====================
    MOV  X0, #0x3FFFFFFFFFFFFFFF  // positive, 2 leading zeros after sign
    CLS  X1, X0              // X1 = 1 (1 redundant sign bit)

    MOV  X0, #0
    MOV  X8, #93
    SVC  #0
```

---

## 14. Compilation และ Execution ด้วย QEMU

### 14.1 ติดตั้ง Tools

```bash
# Ubuntu/Debian
sudo apt install gcc-aarch64-linux-gnu
sudo apt install binutils-aarch64-linux-gnu
sudo apt install qemu-user
sudo apt install qemu-user-static

# Fedora/RHEL
sudo dnf install gcc-aarch64-linux-gnu
sudo dnf install qemu-user

# macOS (Apple Silicon - native)
# ใช้ as และ ld โดยตรง

# macOS (Intel - cross compile)
brew install aarch64-elf-binutils
brew install qemu
```

### 14.2 Compile และ Run

```bash
# วิธีที่ 1: GNU Assembler + Linker (standalone)
aarch64-linux-gnu-as -o hello.o hello_aarch64.s
aarch64-linux-gnu-ld -o hello hello.o
qemu-aarch64 ./hello

# วิธีที่ 2: GCC (ง่ายกว่า)
aarch64-linux-gnu-gcc -o hello hello_aarch64.s
qemu-aarch64 ./hello

# วิธีที่ 3: GCC พร้อม C runtime
aarch64-linux-gnu-gcc -static -o program main.s helper.s
qemu-aarch64 ./program

# วิธีที่ 4: Native บน AArch64 machine (Raspberry Pi, ARM server)
as -o hello.o hello_aarch64.s
ld -o hello hello.o
./hello

# Debug ด้วย GDB
qemu-aarch64 -g 1234 ./hello &
aarch64-linux-gnu-gdb hello
(gdb) target remote :1234
(gdb) break _start
(gdb) continue
(gdb) info registers
(gdb) stepi
```

### 14.3 Makefile สำหรับ AArch64

```makefile
# Makefile สำหรับ AArch64 Assembly

# Cross-compilation tools
AS = aarch64-linux-gnu-as
LD = aarch64-linux-gnu-ld
GCC = aarch64-linux-gnu-gcc
OBJDUMP = aarch64-linux-gnu-objdump
QEMU = qemu-aarch64

# Flags
ASFLAGS = -g
LDFLAGS =
QEMU_FLAGS = -L /usr/aarch64-linux-gnu

# Sources
SRCS = $(wildcard *.s)
OBJS = $(SRCS:.s=.o)
TARGETS = $(SRCS:.s=)

.PHONY: all clean run disasm

all: $(TARGETS)

%.o: %.s
	$(AS) $(ASFLAGS) -o $@ $<

%: %.o
	$(LD) $(LDFLAGS) -o $@ $<

run: all
	$(QEMU) $(QEMU_FLAGS) ./$(word 1, $(TARGETS))

disasm: all
	$(OBJDUMP) -d $(word 1, $(TARGETS))

clean:
	rm -f $(OBJS) $(TARGETS)
```

### 14.4 การใช้ Static Linking กับ C Library

```asm
// c_interop.s
// ตัวอย่าง: เรียกใช้ printf จาก C library

.section .data
fmt_str: .asciz "Result: %ld\n"

.section .text
.global main
.extern printf

main:
    STP X29, X30, [SP, #-16]!
    MOV X29, SP

    // printf("Result: %ld\n", 42)
    ADRP X0, fmt_str
    ADD  X0, X0, :lo12:fmt_str  // format string
    MOV  X1, #42                // argument
    BL   printf                 // call printf

    MOV  X0, #0                 // return 0
    LDP  X29, X30, [SP], #16
    RET
```

```bash
# Compile ด้วย GCC (จะ link C library โดยอัตโนมัติ)
aarch64-linux-gnu-gcc -o c_interop c_interop.s
qemu-aarch64 -L /usr/aarch64-linux-gnu ./c_interop

# หรือ static linking
aarch64-linux-gnu-gcc -static -o c_interop c_interop.s
qemu-aarch64 ./c_interop
```

---

## 15. Conditional Instructions

### 15.1 CSEL / CSINC / CSINV / CSNEG

```asm
// CSEL - Conditional SELect
// Xd = (condition) ? Xn : Xm
CMP  X0, #0
CSEL X1, X2, X3, EQ    // X1 = (X0 == 0) ? X2 : X3
CSEL X1, X2, X3, GT    // X1 = (signed >) ? X2 : X3
CSEL X1, X2, X3, CS    // X1 = (carry set) ? X2 : X3

// CSINC - Conditional Select INCremented
// Xd = (condition) ? Xn : Xm+1
CSINC X0, X1, XZR, EQ  // X0 = (Z=1) ? X1 : 1

// CINC - Conditional INCrement (alias)
// Xd = (condition) ? Xn : Xn+1
CMP  X0, #5
CINC X1, X2, GT         // X1 = (X0 > 5) ? X2 : X2+1

// CSET - Conditional SET (common alias)
// Xd = (condition) ? 1 : 0
CMP  X0, #0
CSET X1, EQ             // X1 = 1 if equal, 0 otherwise
CSET X1, GT             // X1 = 1 if greater, 0 otherwise

// CSETM - Conditional SET Mask
// Xd = (condition) ? -1 : 0
CMP  X0, #0
CSETM X1, EQ            // X1 = -1 (all 1s) if equal, 0 otherwise

// CSINV - Conditional Select INVerted
// Xd = (condition) ? Xn : ~Xm
CSINV X0, X1, X2, EQ   // X0 = (Z=1) ? X1 : ~X2

// CSNEG - Conditional Select NEGated
// Xd = (condition) ? Xn : -Xm
CSNEG X0, X1, X2, GT   // X0 = (signed >) ? X1 : -X2

// CNEG - Conditional NEGate (alias)
CMP  X0, #0
CNEG X1, X2, LT         // X1 = (X0 < 0) ? -X2 : X2
```

### 15.2 ตัวอย่าง: max/min functions

```asm
// max_min.s
// ตัวอย่าง: หาค่า maximum และ minimum

// int64_t max(int64_t a, int64_t b)
max_signed:
    CMP  X0, X1
    CSEL X0, X0, X1, GT    // return (a > b) ? a : b
    RET

// int64_t min(int64_t a, int64_t b)
min_signed:
    CMP  X0, X1
    CSEL X0, X0, X1, LT    // return (a < b) ? a : b
    RET

// uint64_t max_unsigned(uint64_t a, uint64_t b)
max_unsigned:
    CMP  X0, X1
    CSEL X0, X0, X1, HI    // return (a > b unsigned) ? a : b
    RET

// int64_t abs_val(int64_t x)
abs_val:
    NEG  X1, X0             // X1 = -x
    CMP  X0, #0
    CSEL X0, X0, X1, GE     // return (x >= 0) ? x : -x
    RET

// int64_t clamp(int64_t x, int64_t lo, int64_t hi)
// X0 = x, X1 = lo, X2 = hi
clamp:
    CMP  X0, X1
    CSEL X0, X1, X0, LT    // X0 = max(x, lo)
    CMP  X0, X2
    CSEL X0, X2, X0, GT    // X0 = min(X0, hi)
    RET
```

---

## 16. Memory Barriers และ Atomic Operations

### 16.1 DMB / DSB / ISB

```asm
// DMB - Data Memory Barrier
DMB ISH     // Inner Shareable domain barrier
DMB ISHST   // Store barrier
DMB ISHLD   // Load barrier
DMB SY      // Full system barrier (expensive)
DMB NSH     // Non-Shareable barrier
DMB OSH     // Outer Shareable barrier

// DSB - Data Synchronization Barrier
DSB ISH     // Wait for all memory accesses to complete
DSB SY      // Full system synchronization (most strict)

// ISB - Instruction Synchronization Barrier
ISB         // Flush pipeline, reload from memory
// ใช้หลัง:
// - เปลี่ยน system registers ที่กระทบ instruction fetch
// - Self-modifying code
// - ย้าย execution level

// ตัวอย่าง: write to device register
STR W0, [X1]            // write to device
DSB SY                  // ensure write completes
```

### 16.2 LDXR / STXR - Load/Store Exclusive

```asm
// LDXR - Load eXclusive Register
// STXR - Store eXclusive Register
// ใช้สร้าง atomic operations (lock-free)

// ตัวอย่าง: atomic increment
atomic_increment:
try_again:
    LDXR  X0, [X1]          // load-exclusive ค่าปัจจุบัน
    ADD   X0, X0, #1        // เพิ่มค่า
    STXR  W2, X0, [X1]      // store-exclusive
    CBNZ  W2, try_again     // ถ้า exclusive monitor cleared, ลองใหม่
    RET

// ตัวอย่าง: compare-and-swap (CAS)
// bool cas(int64_t *ptr, int64_t expected, int64_t desired)
// X0 = ptr, X1 = expected, X2 = desired
// Returns: X0 = 1 if successful, 0 otherwise
cas:
cas_retry:
    LDXR  X3, [X0]          // load-exclusive
    CMP   X3, X1            // compare with expected
    B.NE  cas_fail          // not equal, fail
    STXR  W4, X2, [X0]      // store desired
    CBNZ  W4, cas_retry     // exclusive monitor fail, retry
    MOV   X0, #1            // success
    RET
cas_fail:
    CLREX                   // clear exclusive monitor
    MOV   X0, #0            // fail
    RET
```

---

## 17. โปรแกรม Demo สมบูรณ์

### 17.1 Calculator Program

```asm
// calculator.s
// Simple integer calculator using AArch64

.section .data
prompt:     .asciz "Enter two numbers and operation (n1 op n2): \n"
result_msg: .asciz "Result: "
newline:    .asciz "\n"
invalid:    .asciz "Invalid input\n"
buf:        .space 64              // input buffer

.section .text
.global main
.extern printf
.extern scanf
.extern exit

// Supported operations: + - * /
main:
    STP  X29, X30, [SP, #-48]!
    MOV  X29, SP
    // เก็บ local variables ที่ [SP+16] และ [SP+24]
    
    // Print prompt
    ADRP X0, prompt
    ADD  X0, X0, :lo12:prompt
    BL   printf
    
    // Read input: scanf("%ld %c %ld", &a, &op, &b)
    // (ย่อ implementation)
    
    LDP  X29, X30, [SP], #48
    RET

// int64_t do_calc(int64_t a, char op, int64_t b)
// X0 = a, W1 = op (char), X2 = b
do_calc:
    CMP  W1, #'+'
    B.EQ calc_add
    CMP  W1, #'-'
    B.EQ calc_sub
    CMP  W1, #'*'
    B.EQ calc_mul
    CMP  W1, #'/'
    B.EQ calc_div
    MOV  X0, #0             // invalid op
    RET

calc_add:
    ADD  X0, X0, X2
    RET
calc_sub:
    SUB  X0, X0, X2
    RET
calc_mul:
    MUL  X0, X0, X2
    RET
calc_div:
    CBZ  X2, div_by_zero
    SDIV X0, X0, X2
    RET
div_by_zero:
    MOV  X0, #0
    RET
```

### 17.2 Fibonacci ด้วย Recursion

```asm
// fibonacci.s
// คำนวณ Fibonacci ด้วย recursive function

.section .text
.global fibonacci

// uint64_t fibonacci(uint64_t n)
// X0 = n
// Returns: X0 = fibonacci(n)
fibonacci:
    // Base cases
    CMP  X0, #0
    B.EQ fib_base_0          // fib(0) = 0
    CMP  X0, #1
    B.LE fib_base_1          // fib(1) = 1

    // Recursive case: fib(n) = fib(n-1) + fib(n-2)
    STP  X29, X30, [SP, #-32]!
    MOV  X29, SP
    STR  X19, [SP, #16]      // save callee-saved X19
    
    MOV  X19, X0             // save n
    
    SUB  X0, X19, #1
    BL   fibonacci           // fib(n-1)
    MOV  X20, X0             // save fib(n-1) result... wait, need to save X20 too
    // แก้ไข: ต้อง save X20 ด้วย
    
    SUB  X0, X19, #2
    BL   fibonacci           // fib(n-2)
    ADD  X0, X0, X20         // fib(n) = fib(n-1) + fib(n-2)
    
    LDR  X19, [SP, #16]
    LDP  X29, X30, [SP], #32
    RET

fib_base_0:
    MOV  X0, #0
    RET
fib_base_1:
    // X0 is already 0 or 1, just return it
    RET

// =============================================
// Iterative version (more efficient)
// uint64_t fibonacci_iter(uint64_t n)
fibonacci_iter:
    CBZ  X0, fib_iter_done   // fib(0) = 0

    MOV  X1, #0              // prev = 0
    MOV  X2, #1              // curr = 1
    MOV  X3, #1              // i = 1

fib_iter_loop:
    CMP  X3, X0
    B.GE fib_iter_result

    ADD  X4, X1, X2          // next = prev + curr
    MOV  X1, X2              // prev = curr
    MOV  X2, X4              // curr = next
    ADD  X3, X3, #1          // i++
    B    fib_iter_loop

fib_iter_result:
    MOV  X0, X2              // return curr
    RET

fib_iter_done:
    MOV  X0, #0
    RET
```

---

## 18. Exercises (แบบฝึกหัด)

### Exercise 1: Register Manipulation
```asm
// TODO: เติมโค้ด
// 1. Load immediate 0xDEADBEEF12345678 ลงใน X0 โดยใช้ MOVZ/MOVK
// 2. Extract bits [31:16] จาก X0 ใส่ X1
// 3. Reverse bytes ใน W0 (32-bit portion)
// 4. Count leading zeros ใน X0

// Answer template:
exercise1:
    // part 1: load 0xDEADBEEF12345678
    MOVZ X0, #0x5678              // bits [15:0]
    MOVK X0, #0x1234, LSL #16    // bits [31:16]
    MOVK X0, #0xBEEF, LSL #32   // bits [47:32]
    MOVK X0, #0xDEAD, LSL #48   // bits [63:48]
    
    // part 2: extract bits [31:16]
    UBFX X1, X0, #16, #16
    
    // part 3: byte reverse W0
    REV W0, W0
    
    // part 4: count leading zeros
    CLZ X2, X0
    RET
```

### Exercise 2: Array Processing
```asm
// TODO: implement these functions
// 1. find_max(int64_t *arr, int64_t n) -> int64_t
// 2. find_min(int64_t *arr, int64_t n) -> int64_t
// 3. array_average(int64_t *arr, int64_t n) -> int64_t

find_max:
    // X0 = array pointer, X1 = count
    LDR  X2, [X0], #8       // max = arr[0], advance pointer
    SUB  X1, X1, #1         // n--
    
find_max_loop:
    CBZ  X1, find_max_done
    LDR  X3, [X0], #8       // load next element
    CMP  X3, X2
    CSEL X2, X3, X2, GT     // max = (arr[i] > max) ? arr[i] : max
    SUB  X1, X1, #1
    B    find_max_loop
    
find_max_done:
    MOV  X0, X2
    RET
```

### Exercise 3: String Functions
```asm
// TODO: implement string functions
// 1. str_toupper(char *s) - แปลง lowercase เป็น uppercase
// 2. str_count_char(const char *s, char c) - นับจำนวน char
// 3. str_reverse(char *s) - reverse string in-place

str_toupper:
    // X0 = pointer to string
str_toupper_loop:
    LDRB W1, [X0]           // load char
    CBZ  W1, str_toupper_done  // null terminator
    
    // check if lowercase: 'a'(97) <= c <= 'z'(122)
    CMP  W1, #'a'
    B.LT str_toupper_next
    CMP  W1, #'z'
    B.GT str_toupper_next
    
    SUB  W1, W1, #32        // convert to uppercase
    STRB W1, [X0]           // store back
    
str_toupper_next:
    ADD  X0, X0, #1
    B    str_toupper_loop
    
str_toupper_done:
    RET
```

### Exercise 4: Bit Operations Challenge
```asm
// TODO: implement bit manipulation functions
// 1. count_bits(uint64_t x) - นับจำนวน 1-bits (popcount)
// 2. is_power_of_2(uint64_t x) -> bool
// 3. next_power_of_2(uint64_t x) -> uint64_t

// Hint for is_power_of_2:
// x is power of 2 if x != 0 AND (x & (x-1)) == 0
is_power_of_2:
    CBZ  X0, not_power        // 0 is not power of 2
    SUB  X1, X0, #1           // x - 1
    ANDS X2, X0, X1           // x & (x-1)
    CSET X0, EQ               // 1 if (x & (x-1)) == 0
    RET
not_power:
    MOV  X0, #0
    RET

// next_power_of_2 using CLZ:
next_power_of_2:
    SUB  X1, X0, #1           // x - 1
    CLZ  X2, X1               // count leading zeros
    MOV  X0, #1
    MOV  X3, #64
    SUB  X3, X3, X2           // 64 - clz
    LSL  X0, X0, X3           // 1 << (64 - clz)
    RET
```

---

## 19. สรุป AArch64 Cheat Sheet

### 19.1 Register Quick Reference

```
Registers:
  X0-X7   : Function arguments / return values (caller-saved)
  X8      : Indirect result location / syscall number
  X9-X15  : Temporary registers (caller-saved)
  X16,X17 : Intra-procedure-call temporaries (IP0, IP1)
  X18     : Platform register (reserved)
  X19-X28 : Callee-saved registers
  X29     : Frame Pointer (FP)
  X30     : Link Register (LR) = return address
  SP      : Stack Pointer (must be 16-byte aligned)
  XZR     : Zero register (reads 0, writes discarded)
  PC      : Program Counter (not directly accessible)

Special:
  W<n>    : 32-bit view of X<n> (writes zero-extend to 64-bit)
  WZR     : 32-bit zero register
```

### 19.2 Common Patterns

```asm
// Function call
BL  function_name           // ส่งค่า return address ไปยัง X30

// Return from function
RET                         // branch to X30

// NULL check
CBZ X0, null_handler        // if X0 == NULL, handle it

// Load/Store with offset
LDR X0, [X1, #8]            // load 64-bit at X1+8
STR X0, [X1, #8]            // store 64-bit to X1+8

// Multiply by power of 2
LSL X0, X0, #3              // x *= 8

// Absolute value (signed)
NEG X1, X0
CMP X0, #0
CSEL X0, X0, X1, GE         // return x >= 0 ? x : -x

// Loop (countdown)
MOV X0, #n
loop:
    // ... body ...
    SUBS X0, X0, #1
    B.NE loop

// if-else
CMP X0, X1
B.EQ then_part
    // else part
    B   done
then_part:
    // then part
done:
```

### 19.3 System Call Quick Reference (Linux AArch64)

```
Syscall convention:
  X8 = syscall number
  X0-X5 = arguments
  X0 = return value / -errno

Common syscalls:
  63  = read(fd, buf, count)
  64  = write(fd, buf, count)
  93  = exit(code)
  94  = exit_group(code)
  172 = getpid()
  178 = gettid()
  220 = clone(flags, stack, ...)
  222 = mmap(addr, len, prot, flags, fd, off)
  215 = munmap(addr, len)
  80  = fstat(fd, buf)
  56  = openat(dirfd, path, flags, mode)
  57  = close(fd)
```

---

## 20. สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **AArch64 vs ARM32**: AArch64 มี 31 registers (vs 16), instructions ขนาดคงที่ 32-bit,
   และ addressing modes ที่เรียบง่ายกว่า

2. **Register File**: X0-X30 (64-bit), W0-W30 (32-bit), XZR (zero), SP (stack pointer),
   PC (program counter ไม่สามารถ access โดยตรง)

3. **NZCV Flags**: Negative, Zero, Carry, Overflow - ใช้กับ conditional branches

4. **Arithmetic**: ADD/SUB/MUL/UDIV/SDIV และ variants ต่างๆ รวมถึง wide multiply

5. **Load/Store**: LDR/STR/LDP/STP พร้อม addressing modes หลากหลาย
   (base, immediate offset, register offset, pre/post-index)

6. **ADR/ADRP**: PC-relative addressing สำหรับ position-independent code

7. **Branches**: B/BL/BR/BLR/RET, B.cond, CBZ/CBNZ, TBZ/TBNZ

8. **System Instructions**: MRS/MSR (system registers), SVC (syscalls),
   BRK (breakpoint), HLT, ERET (exception return)

9. **Bit Manipulation**: UBFX/SBFX (extract), REV/REV16/REV32 (byte swap),
   CLZ/CLS/RBIT (bit counting and reversal)

10. **Conditional Select**: CSEL/CSINC/CSET ช่วยหลีกเลี่ยง branch misprediction

### ขั้นตอนต่อไป (Next Steps)

- Part 039: NEON/SIMD Instructions ใน AArch64
- Part 040: Floating-Point ใน AArch64 (scalar)
- Part 041: SVE/SVE2 - Scalable Vector Extension
- Part 042: AArch64 Linux Kernel Programming
- Part 043: AArch64 Optimization Techniques

---

*หมายเหตุ: ตัวอย่างในบทนี้ใช้ GNU Assembler (GAS) syntax สำหรับ AArch64*
*สามารถ compile บน Ubuntu/Debian ด้วย aarch64-linux-gnu-as หรือ native AArch64 system*
