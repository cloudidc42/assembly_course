# Part 039: ARM64 Calling Convention (AAPCS64)
## หลักการเรียกใช้ฟังก์ชันใน AArch64

---

## บทนำ (Introduction)

**AAPCS64** ย่อมาจาก **Procedure Call Standard for the Arm 64-bit Architecture**
คือมาตรฐานที่กำหนดวิธีการส่งผ่านพารามิเตอร์ เก็บค่า Return และบริหาร Register
ในโปรแกรม AArch64 (ARM64)

การเข้าใจ Calling Convention เป็นพื้นฐานสำคัญสำหรับ:
- การเขียน Assembly ที่ทำงานร่วมกับ C/C++ ได้
- การ Debug โปรแกรมระดับต่ำ
- การเขียน Compiler และ Runtime
- การทำ Reverse Engineering

---

## 1. Register Overview ใน AArch64

```
╔══════════════════════════════════════════════════════════════╗
║              AArch64 General Purpose Registers               ║
╠══════════════╦══════════════╦══════════════════════════════╣
║  64-bit Name ║  32-bit Name ║  หน้าที่                      ║
╠══════════════╬══════════════╬══════════════════════════════╣
║    X0        ║     W0       ║  Arg 1 / Return value         ║
║    X1        ║     W1       ║  Arg 2 / Return value (high)  ║
║    X2        ║     W2       ║  Arg 3                        ║
║    X3        ║     W3       ║  Arg 4                        ║
║    X4        ║     W4       ║  Arg 5                        ║
║    X5        ║     W5       ║  Arg 6                        ║
║    X6        ║     W6       ║  Arg 7                        ║
║    X7        ║     W7       ║  Arg 8                        ║
╠══════════════╬══════════════╬══════════════════════════════╣
║    X8        ║     W8       ║  Indirect result location /   ║
║              ║              ║  Syscall number               ║
╠══════════════╬══════════════╬══════════════════════════════╣
║  X9 - X15   ║  W9 - W15   ║  Caller-saved temporaries     ║
╠══════════════╬══════════════╬══════════════════════════════╣
║  X16 (IP0)  ║    W16       ║  Intra-procedure-call scratch ║
║  X17 (IP1)  ║    W17       ║  Intra-procedure-call scratch ║
╠══════════════╬══════════════╬══════════════════════════════╣
║    X18       ║    W18       ║  Platform register (reserved) ║
╠══════════════╬══════════════╬══════════════════════════════╣
║  X19 - X28  ║  W19 - W28  ║  Callee-saved registers       ║
╠══════════════╬══════════════╬══════════════════════════════╣
║  X29 (FP)   ║    W29       ║  Frame Pointer                ║
║  X30 (LR)   ║    W30       ║  Link Register                ║
╠══════════════╬══════════════╬══════════════════════════════╣
║    XZR       ║    WZR       ║  Zero Register (read = 0)     ║
║    SP        ║    WSP       ║  Stack Pointer                ║
║    PC        ║              ║  Program Counter              ║
╚══════════════╩══════════════╩══════════════════════════════╝
```

### 1.1 Floating-Point / SIMD Registers

```
╔══════════════════════════════════════════════════════════════╗
║              SIMD & Floating-Point Registers                 ║
╠══════════════╦═══════════════════════════════════════════════╣
║   Register   ║  หน้าที่                                       ║
╠══════════════╬═══════════════════════════════════════════════╣
║  V0  - V7   ║  FP/SIMD Arg 1-8 / Return value              ║
║  V8  - V15  ║  Callee-saved (lower 64-bit only)            ║
║  V16 - V31  ║  Caller-saved temporaries                    ║
╠══════════════╬═══════════════════════════════════════════════╣
║  ชื่อย่อของ V registers:                                      ║
║  Qn = 128-bit (quad)                                        ║
║  Dn = 64-bit  (double)                                      ║
║  Sn = 32-bit  (single)                                      ║
║  Hn = 16-bit  (half)                                        ║
║  Bn = 8-bit   (byte)                                        ║
╚══════════════╩═══════════════════════════════════════════════╝
```

---

## 2. Integer Arguments: X0-X7

กฎการส่ง Integer Arguments:
- Arguments 1-8 ส่งผ่าน **X0-X7**
- Arguments ที่เกิน 8 ตัว ส่งผ่าน **Stack**
- ค่าที่เล็กกว่า 64-bit จะถูก **Zero-extend** หรือ **Sign-extend**

```asm
// ===== ตัวอย่างที่ 1: การส่ง Integer Arguments =====
// ไฟล์: int_args.s

    .section .text
    .global main
    .global add_three        // ฟังก์ชัน: a + b + c
    .global add_eight        // ฟังก์ชัน: ผลรวม 8 จำนวน
    .global add_nine         // ฟังก์ชัน: ผลรวม 9 จำนวน (มี Stack arg)

// -------------------------------------------
// int add_three(int a, int b, int c)
// X0 = a, X1 = b, X2 = c
// Return: X0 = a + b + c
// -------------------------------------------
add_three:
    // ไม่ต้อง save registers เพราะใช้แค่ caller-saved
    ADD     W0, W0, W1          // W0 = a + b
    ADD     W0, W0, W2          // W0 = (a + b) + c
    RET                          // Return ค่าใน X0

// -------------------------------------------
// long add_eight(long a, long b, long c, long d,
//               long e, long f, long g, long h)
// X0=a X1=b X2=c X3=d X4=e X5=f X6=g X7=h
// Return: X0 = sum ของทั้งหมด
// -------------------------------------------
add_eight:
    // ใช้ X0 สะสมผลรวม
    ADD     X0, X0, X1          // X0 = a + b
    ADD     X0, X0, X2          // X0 = + c
    ADD     X0, X0, X3          // X0 = + d
    ADD     X0, X0, X4          // X0 = + e
    ADD     X0, X0, X5          // X0 = + f
    ADD     X0, X0, X6          // X0 = + g
    ADD     X0, X0, X7          // X0 = + h
    RET

// -------------------------------------------
// long add_nine(long a, long b, long c, long d,
//              long e, long f, long g, long h,
//              long i)
// X0-X7 = a-h, [SP] = i (บน Stack)
// Return: X0 = sum
// -------------------------------------------
add_nine:
    LDR     X9, [SP]            // โหลด argument ที่ 9 จาก Stack
    ADD     X0, X0, X1
    ADD     X0, X0, X2
    ADD     X0, X0, X3
    ADD     X0, X0, X4
    ADD     X0, X0, X5
    ADD     X0, X0, X6
    ADD     X0, X0, X7
    ADD     X0, X0, X9          // รวม argument ที่ 9
    RET

// -------------------------------------------
// main function
// -------------------------------------------
main:
    STP     X29, X30, [SP, #-16]!   // Save FP และ LR
    MOV     X29, SP                   // ตั้ง Frame Pointer

    // เรียก add_three(10, 20, 30)
    MOV     X0, #10
    MOV     X1, #20
    MOV     X2, #30
    BL      add_three               // X0 = 60

    // เรียก add_eight(1, 2, 3, 4, 5, 6, 7, 8)
    MOV     X0, #1
    MOV     X1, #2
    MOV     X2, #3
    MOV     X3, #4
    MOV     X4, #5
    MOV     X5, #6
    MOV     X6, #7
    MOV     X7, #8
    BL      add_eight               // X0 = 36

    // เรียก add_nine(1, 2, 3, 4, 5, 6, 7, 8, 9)
    // ต้อง push argument ที่ 9 ลง Stack ก่อน
    MOV     X0, #1
    MOV     X1, #2
    MOV     X2, #3
    MOV     X3, #4
    MOV     X4, #5
    MOV     X5, #6
    MOV     X6, #7
    MOV     X7, #8
    MOV     X9, #9
    STR     X9, [SP, #-16]!         // Push argument ที่ 9 (align 16)
    BL      add_nine                // X0 = 45
    ADD     SP, SP, #16             // คืน Stack

    MOV     X0, #0                  // Return 0
    LDP     X29, X30, [SP], #16    // Restore FP และ LR
    RET
```

---

## 3. Floating-Point Arguments: V0-V7

```asm
// ===== ตัวอย่างที่ 2: Floating-Point Arguments =====
// ไฟล์: fp_args.s

    .section .data
fmt_double: .string "Result: %f\n"

    .section .text
    .global main

// -------------------------------------------
// double fp_sum(double a, double b, double c)
// D0 = a, D1 = b, D2 = c
// Return: D0 = a + b + c
// -------------------------------------------
fp_sum:
    FADD    D0, D0, D1      // D0 = a + b
    FADD    D0, D0, D2      // D0 = (a + b) + c
    RET

// -------------------------------------------
// double mixed_calc(int n, double x, float y)
// X0 = n (integer), D0 = x (double), S1 = y (float)
// Return: D0 = n * x + y
// -------------------------------------------
mixed_calc:
    // แปลง integer n เป็น double
    SCVTF   D2, X0          // D2 = (double)n
    FMUL    D0, D0, D2      // D0 = n * x

    // แปลง float y เป็น double แล้วบวก
    FCVT    D1, S1          // D1 = (double)y
    FADD    D0, D0, D1      // D0 = n * x + y
    RET

// -------------------------------------------
// double dot_product(double a1, double a2, double a3,
//                   double b1, double b2, double b3)
// D0=a1 D1=a2 D2=a3 D3=b1 D4=b2 D5=b3
// Return: D0 = a1*b1 + a2*b2 + a3*b3
// -------------------------------------------
dot_product:
    FMUL    D0, D0, D3      // D0 = a1 * b1
    FMUL    D1, D1, D4      // D1 = a2 * b2
    FMUL    D2, D2, D5      // D2 = a3 * b3
    FADD    D0, D0, D1      // D0 = a1*b1 + a2*b2
    FADD    D0, D0, D2      // D0 = + a3*b3
    RET

main:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    // เรียก dot_product([1.0,2.0,3.0], [4.0,5.0,6.0])
    // Expected: 1*4 + 2*5 + 3*6 = 4 + 10 + 18 = 32
    FMOV    D0, #1.0
    FMOV    D1, #2.0
    FMOV    D2, #3.0
    FMOV    D3, #4.0
    FMOV    D4, #5.0
    FMOV    D5, #6.0
    BL      dot_product             // D0 = 32.0

    // พิมพ์ผลลัพธ์
    ADRP    X0, fmt_double
    ADD     X0, X0, :lo12:fmt_double
    // D0 ยังมีค่า 32.0 อยู่
    BL      printf

    MOV     X0, #0
    LDP     X29, X30, [SP], #16
    RET
```

---

## 4. Return Values

### 4.1 Integer Return Values

```asm
// ===== ตัวอย่างที่ 3: Return Values =====

// -------------------------------------------
// int return_int(void)
// Return ค่า 42 ใน X0
// -------------------------------------------
return_int:
    MOV     W0, #42
    RET

// -------------------------------------------
// long return_long(void)
// Return ค่า 1234567890123 ใน X0
// -------------------------------------------
return_long:
    MOV     X0, #1234567890123
    RET

// -------------------------------------------
// __int128 return_128bit(void)
// Return ค่า 128-bit: X0 = lower 64-bit, X1 = upper 64-bit
// -------------------------------------------
return_128bit:
    MOV     X0, #0xDEADBEEF         // lower 64-bit
    MOV     X1, #0xCAFEBABE         // upper 64-bit
    RET

// -------------------------------------------
// char return_char(void)
// Return ค่า 'A' ใน W0 (Zero-extended เป็น 64-bit)
// -------------------------------------------
return_char:
    MOV     W0, #65                 // 'A' = 65
    RET

// -------------------------------------------
// bool return_bool(void)
// Return 1 (true) ใน W0
// -------------------------------------------
return_bool:
    MOV     W0, #1
    RET
```

### 4.2 Floating-Point Return Values

```asm
// -------------------------------------------
// float return_float(void)
// Return 3.14 ใน S0
// -------------------------------------------
return_float:
    ADRP    X0, pi_float
    ADD     X0, X0, :lo12:pi_float
    LDR     S0, [X0]
    RET

// -------------------------------------------
// double return_double(void)
// Return 3.14159265358979 ใน D0
// -------------------------------------------
return_double:
    ADRP    X0, pi_double
    ADD     X0, X0, :lo12:pi_double
    LDR     D0, [X0]
    RET

    .section .data
pi_float:   .float  3.14
pi_double:  .double 3.14159265358979
```

---

## 5. Caller-Saved vs Callee-Saved Registers

### 5.1 ความแตกต่าง

```
Caller-Saved (Scratch) Registers:
- X0-X18, V0-V7, V16-V31
- Caller ต้อง Save เองก่อนเรียกฟังก์ชัน ถ้าต้องการใช้หลัง CALL
- Callee ไม่จำเป็นต้อง Restore

Callee-Saved (Non-volatile) Registers:
- X19-X28, FP(X29), LR(X30) *, SP
- V8-V15 (lower 64-bit only)
- Callee ต้อง Save และ Restore ถ้าต้องการใช้
- Caller มั่นใจได้ว่าค่าจะไม่เปลี่ยน

* LR(X30): ถ้าฟังก์ชันเรียกฟังก์ชันอื่น ต้อง Save LR
```

```asm
// ===== ตัวอย่างที่ 4: Caller-Saved Registers =====

// -------------------------------------------
// void caller_example(void)
// ตัวอย่าง Caller ที่ต้อง Save registers
// -------------------------------------------
caller_example:
    STP     X29, X30, [SP, #-32]!
    MOV     X29, SP

    // คำนวณบางอย่าง ใส่ใน X0
    MOV     X0, #100

    // ต้องการใช้ค่า X0 หลัง BL แต่ X0 เป็น caller-saved
    // ต้อง Save X0 เอง
    STR     X0, [SP, #16]           // Save X0 ลง Stack

    // เรียกฟังก์ชันที่จะ Clobber X0-X18
    BL      some_function

    // X0 ถูก Clobber แล้ว ต้อง Restore
    LDR     X0, [SP, #16]           // Restore X0

    // ตอนนี้ X0 = 100 เหมือนเดิม

    LDP     X29, X30, [SP], #32
    RET

// -------------------------------------------
// void callee_example(void)
// ตัวอย่าง Callee ที่ต้อง Save Callee-Saved Registers
// -------------------------------------------
callee_example:
    // ต้องการใช้ X19, X20 ซึ่งเป็น Callee-Saved
    // ต้อง Save ก่อนใช้
    STP     X29, X30, [SP, #-48]!
    MOV     X29, SP
    STP     X19, X20, [SP, #16]     // Save X19, X20

    // ใช้ X19, X20 ได้อย่างอิสระ
    MOV     X19, #200
    MOV     X20, #300

    BL      another_function        // X19, X20 ยังคงค่าเดิมหลัง Call

    ADD     X0, X19, X20            // X0 = 500

    // Restore callee-saved registers
    LDP     X19, X20, [SP, #16]
    LDP     X29, X30, [SP], #48
    RET
```

### 5.2 ตัวอย่างการใช้ Callee-Saved Registers ใน Loop

```asm
// ===== ตัวอย่างที่ 5: Using Callee-Saved in Loop =====

// -------------------------------------------
// long sum_array(long *arr, int n)
// X0 = arr pointer, W1 = n (count)
// ใช้ X19=arr, X20=n, X21=sum (callee-saved)
// เพราะ inner loop เรียก printf
// -------------------------------------------
sum_array_verbose:
    STP     X29, X30, [SP, #-64]!
    MOV     X29, SP
    STP     X19, X20, [SP, #16]     // Save X19, X20
    STP     X21, X22, [SP, #32]     // Save X21, X22

    MOV     X19, X0                 // X19 = arr pointer
    MOV     W20, W1                 // W20 = n
    MOV     X21, #0                 // X21 = sum = 0
    MOV     W22, #0                 // W22 = index i = 0

loop_start:
    CMP     W22, W20                // i < n?
    BGE     loop_end

    LDR     X0, [X19, X22, SXTW #3]    // X0 = arr[i]
    ADD     X21, X21, X0               // sum += arr[i]

    // พิมพ์แต่ละค่า (BL จะ Clobber X0-X18 แต่ X19-X28 ยังเหมือนเดิม)
    ADRP    X0, fmt_element
    ADD     X0, X0, :lo12:fmt_element
    MOV     W1, W22                     // index
    LDR     X2, [X19, X22, SXTW #3]    // value
    BL      printf
    // X19, X20, X21, X22 ยังคงค่าเดิม!

    ADD     W22, W22, #1               // i++
    B       loop_start

loop_end:
    MOV     X0, X21                // Return sum

    LDP     X21, X22, [SP, #32]
    LDP     X19, X20, [SP, #16]
    LDP     X29, X30, [SP], #64
    RET

    .section .data
fmt_element: .string "arr[%d] = %ld\n"
```

---

## 6. Frame Pointer (X29) และ Link Register (X30)

### 6.1 การทำงานของ Frame Pointer

```
Stack Layout ของ Function Call:

    High Address
    ┌─────────────────┐
    │  Caller's Frame │
    ├─────────────────┤  <── Caller's SP (before call)
    │  Arguments > 8  │       (ถ้ามี)
    ├─────────────────┤  <── Callee's SP (after STP FP,LR)
    │   Saved LR      │       [SP, #8]
    │   Saved FP      │       [SP, #0]  <── FP ชี้ที่นี่
    ├─────────────────┤
    │  Local Variables│
    │                 │
    ├─────────────────┤  <── Current SP
    │  (Red zone N/A) │
    └─────────────────┘
    Low Address

FP (X29) สร้าง Linked List ของ Stack Frames:
    FP → [Saved FP, Saved LR] → [Saved FP, Saved LR] → ...
    ทำให้ Debugger สามารถ Unwind Stack ได้
```

```asm
// ===== ตัวอย่างที่ 6: Frame Pointer Usage =====

// -------------------------------------------
// void function_with_fp(void)
// แสดงการใช้ Frame Pointer อย่างถูกต้อง
// -------------------------------------------
function_with_fp:
    // Standard prologue
    STP     X29, X30, [SP, #-32]!   // Push FP, LR; SP -= 32
    MOV     X29, SP                  // FP = SP (ชี้ที่ frame record)

    // ตอนนี้:
    // SP = FP → [X29, X30] (8 bytes each)
    // [FP, #0]  = Previous FP
    // [FP, #8]  = Return Address (LR)
    // [FP, #16] และต่อไป = Local variables

    STR     X19, [SP, #16]          // Local variable บน Stack

    // ... ทำงาน ...

    // Standard epilogue
    LDR     X19, [SP, #16]
    LDP     X29, X30, [SP], #32     // Pop FP, LR; SP += 32
    RET

// -------------------------------------------
// ฟังก์ชัน Recursive ที่ใช้ FP
// long factorial(long n)
// -------------------------------------------
factorial:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    // Base case: n <= 1 → return 1
    CMP     X0, #1
    BLE     fact_base

    // Recursive case: n * factorial(n-1)
    STP     X0, XZR, [SP, #-16]!    // Save n
    SUB     X0, X0, #1              // X0 = n - 1
    BL      factorial               // X0 = factorial(n-1)

    LDP     X1, XZR, [SP], #16      // Restore n ใน X1
    MUL     X0, X0, X1              // X0 = n * factorial(n-1)

    LDP     X29, X30, [SP], #16
    RET

fact_base:
    MOV     X0, #1
    LDP     X29, X30, [SP], #16
    RET
```

### 6.2 Stack Backtrace ด้วย FP Chain

```asm
// ===== ตัวอย่างที่ 7: Walking the Frame Pointer Chain =====

// -------------------------------------------
// void print_backtrace(void)
// เดิน FP chain เพื่อแสดง Call Stack
// -------------------------------------------
print_backtrace:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    MOV     X19, X29                // เริ่มจาก Current FP
    MOV     W20, #0                 // Frame counter

walk_loop:
    CBZ     X19, walk_done          // FP = 0 หมาย End of chain
    CMP     W20, #20                // จำกัด 20 frames
    BGE     walk_done

    ADRP    X0, fmt_frame
    ADD     X0, X0, :lo12:fmt_frame
    MOV     W1, W20                 // Frame number
    LDR     X2, [X19, #8]          // Return address (LR ที่ Save ไว้)
    BL      printf

    LDR     X19, [X19]             // ไปยัง Previous FP
    ADD     W20, W20, #1
    B       walk_loop

walk_done:
    LDP     X29, X30, [SP], #16
    RET

    .section .data
fmt_frame: .string "Frame #%d: PC = 0x%016lx\n"
```

---

## 7. Stack Alignment: 16-byte Rule

**กฎสำคัญ**: SP ต้องถูก Align ที่ **16 bytes** ณ ทุก `BL` instruction

```asm
// ===== ตัวอย่างที่ 8: Stack Alignment =====

// -------------------------------------------
// WRONG: Stack ไม่ Aligned ที่ 16 bytes
// -------------------------------------------
bad_function:
    STP     X29, X30, [SP, #-16]!   // SP ลด 16: OK
    MOV     X29, SP

    SUB     SP, SP, #8              // SP ลด 8: ตอนนี้ NOT 16-aligned!
    STR     X19, [SP]               // Store variable

    BL      some_func               // WRONG! SP ต้องเป็น multiple ของ 16

    ADD     SP, SP, #8
    LDP     X29, X30, [SP], #16
    RET

// -------------------------------------------
// CORRECT: Stack Aligned ที่ 16 bytes เสมอ
// -------------------------------------------
good_function:
    STP     X29, X30, [SP, #-32]!   // SP ลด 32 (multiple of 16): OK
    MOV     X29, SP
    STR     X19, [SP, #16]          // Store ใน Slot ที่ Allocated แล้ว

    BL      some_func               // OK! SP = 16-aligned

    LDR     X19, [SP, #16]
    LDP     X29, X30, [SP], #32
    RET

// -------------------------------------------
// ตัวอย่าง: Local variables ที่ต้อง Align
// -------------------------------------------
local_vars_example:
    // ต้องการ: int a (4 bytes), long b (8 bytes), char c (1 byte)
    // รวม = 13 bytes → Round up เป็น 16 bytes

    STP     X29, X30, [SP, #-32]!   // 16 สำหรับ FP+LR + 16 สำหรับ locals
    MOV     X29, SP

    // Layout:
    // [SP, #0]  = FP (saved)
    // [SP, #8]  = LR (saved)
    // [SP, #16] = int a (4 bytes)
    // [SP, #20] = padding (4 bytes)
    // [SP, #24] = long b (wait, only 8 bytes left...)

    // ถ้าต้องการ Align ดีขึ้น จัดตาม size (ใหญ่ก่อน):
    // [SP, #16] = long b (8 bytes)
    // [SP, #24] = int a (4 bytes)
    // [SP, #28] = char c (1 byte) + 3 bytes padding

    STR     X0, [SP, #16]           // b = X0
    STR     W1, [SP, #24]           // a = W1
    STRB    W2, [SP, #28]           // c = W2

    LDP     X29, X30, [SP], #32
    RET
```

---

## 8. AArch64 ไม่มี Red Zone

ต่างจาก x86-64 System V ABI ที่มี **128-byte Red Zone**,
**AArch64 ไม่มี Red Zone** เลย

```asm
// ===== ตัวอย่างที่ 9: No Red Zone in AArch64 =====

// -------------------------------------------
// x86-64 (Linux): สามารถใช้ [RSP - 128] เป็น scratch space
// AArch64: ห้ามใช้ memory ต่ำกว่า SP
// -------------------------------------------

// WRONG in AArch64 (แต่ OK ใน x86-64):
wrong_aarch64:
    STR     X0, [SP, #-8]           // WRONG! ใช้ space ต่ำกว่า SP
    // Signal handler อาจ Overwrite [SP-8]!
    LDR     X0, [SP, #-8]           // ค่าอาจเปลี่ยนไปแล้ว
    RET

// CORRECT in AArch64:
correct_aarch64:
    SUB     SP, SP, #16             // จอง Stack space ก่อน
    STR     X0, [SP]                // OK! SP ได้ถูก Adjust แล้ว
    LDR     X0, [SP]
    ADD     SP, SP, #16             // คืน Stack
    RET

// -------------------------------------------
// Leaf Function ที่ไม่เรียก BL สามารถใช้ Registers ได้
// แต่ยังต้องไม่ใช้ memory ต่ำกว่า SP
// -------------------------------------------
leaf_correct:
    // ใช้ Scratch Registers (X9-X15) แทน Stack
    MOV     X9, X0          // ใช้ X9 เป็น Scratch แทนการ Store ลง Stack
    MOV     X10, X1
    ADD     X0, X9, X10
    RET
```

---

## 9. Variadic Functions (va_list)

```asm
// ===== ตัวอย่างที่ 10: Variadic Functions =====

// -------------------------------------------
// ใน C: int my_printf(const char *fmt, ...)
// AArch64 Variadic Convention:
// - Arguments ปกติผ่าน X0-X7 และ V0-V7
// - ไม่มี Register save area แบบ x86-64 (ขึ้นอยู่กับ ABI version)
// -------------------------------------------

// ตัวอย่าง: เรียก printf ใน Assembly
call_printf_example:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    // printf("Hello %s, you are %d years old\n", name, age)
    ADRP    X0, fmt_str
    ADD     X0, X0, :lo12:fmt_str   // X0 = format string
    ADRP    X1, name_str
    ADD     X1, X1, :lo12:name_str  // X1 = name (arg 1)
    MOV     W2, #25                  // W2 = age (arg 2)
    BL      printf

    LDP     X29, X30, [SP], #16
    RET

    .section .data
fmt_str:  .string "Hello %s, you are %d years old\n"
name_str: .string "Alice"

// -------------------------------------------
// Implementation ของ Variadic Function แบบ Manual
// int sum_ints(int count, ...)
// X0 = count, X1-X7 = first 7 integers
// -------------------------------------------
sum_ints:
    STP     X29, X30, [SP, #-48]!
    MOV     X29, SP

    // Save register arguments ที่เป็น variadic ลง Stack
    // เพื่อให้สามารถ iterate ผ่านได้
    STP     X1, X2, [SP, #16]
    STP     X3, X4, [SP, #24]
    STP     X5, X6, [SP, #32]
    STR     X7, [SP, #40]

    MOV     W9, W0                  // W9 = count
    MOV     X10, #0                 // X10 = sum
    ADD     X11, SP, #16            // X11 = pointer ไปยัง saved args

sum_loop:
    CBZ     W9, sum_done
    LDR     X12, [X11], #8          // Load argument แล้วเลื่อน pointer
    ADD     X10, X10, X12
    SUB     W9, W9, #1
    B       sum_loop

sum_done:
    MOV     X0, X10

    LDP     X29, X30, [SP], #48
    RET
```

---

## 10. Struct Passing Rules

### 10.1 Small Structs (ไม่เกิน 16 bytes)

```asm
// ===== ตัวอย่างที่ 11: Small Struct Passing =====

// C Struct:
// struct Point { long x; long y; };  // 16 bytes
//
// void process_point(struct Point p)
// → X0 = p.x, X1 = p.y

process_point:
    // X0 = x, X1 = y
    MUL     X0, X0, X1      // ไม่ต้อง Dereference
    RET

// C Struct:
// struct Small { int a; int b; };  // 8 bytes
//
// void process_small(struct Small s)
// → X0 = packed {b:32|a:32} (ขึ้นอยู่กับ endian)

// =============================================
// Return Struct ขนาดเล็ก:
// struct Point make_point(long x, long y)
// → X0 = x, X1 = y
// =============================================
make_point:
    // X0 = x, X1 = y จาก Parameters
    // Return: X0 = p.x, X1 = p.y (ไม่ต้องทำอะไร)
    RET

// =============================================
// struct ThreeLong { long a, b, c; }; // 24 bytes > 16!
// void big_struct(struct ThreeLong t)
// → ส่งผ่าน Pointer (X0 = &t)
// =============================================
process_big_struct:
    LDR     X1, [X0]        // X1 = t.a
    LDR     X2, [X0, #8]    // X2 = t.b
    LDR     X3, [X0, #16]   // X3 = t.c
    ADD     X0, X1, X2
    ADD     X0, X0, X3      // Return a + b + c
    RET
```

### 10.2 Large Structs by Reference

```asm
// ===== ตัวอย่างที่ 12: Large Struct by Reference =====

// -------------------------------------------
// struct Matrix4x4 { float m[4][4]; }; // 64 bytes
//
// struct Matrix4x4 multiply(struct Matrix4x4 a, struct Matrix4x4 b)
// C Compiler จะแปลงเป็น:
// void multiply(struct Matrix4x4 *result,   ← X8 (hidden pointer)
//               struct Matrix4x4 *a,         ← X0
//               struct Matrix4x4 *b)          ← X1
// -------------------------------------------
matrix_multiply:
    STP     X29, X30, [SP, #-80]!
    MOV     X29, SP
    STP     X19, X20, [SP, #16]
    STP     X21, X22, [SP, #32]

    MOV     X19, X8                 // X19 = result pointer (จาก X8!)
    MOV     X20, X0                 // X20 = a pointer
    MOV     X21, X1                 // X21 = b pointer

    // ... ทำ Matrix Multiplication ...
    // เขียนผลลัพธ์ผ่าน X19

    LDP     X21, X22, [SP, #32]
    LDP     X19, X20, [SP, #16]
    LDP     X29, X30, [SP], #80
    RET

// -------------------------------------------
// Caller สำหรับ Large Struct Return:
// -------------------------------------------
caller_large_struct:
    STP     X29, X30, [SP, #-80]!
    MOV     X29, SP

    // จอง Space สำหรับ Result struct (64 bytes)
    SUB     SP, SP, #64
    MOV     X8, SP              // X8 = pointer ไปยัง Return buffer

    // โหลด a และ b pointers
    ADRP    X0, matrix_a
    ADD     X0, X0, :lo12:matrix_a
    ADRP    X1, matrix_b
    ADD     X1, X1, :lo12:matrix_b

    BL      matrix_multiply     // ผลลัพธ์อยู่ที่ [SP]

    ADD     SP, SP, #64         // คืน Stack
    LDP     X29, X30, [SP], #80
    RET

    .section .data
matrix_a: .space 64
matrix_b: .space 64
```

---

## 11. HFA (Homogeneous Floating-point Aggregate)

**HFA** คือ Struct ที่มี Floating-point members ชนิดเดียวกัน ไม่เกิน 4 ตัว

```asm
// ===== ตัวอย่างที่ 13: HFA Rules =====

// C Definitions:
// struct Vec2 { float x, y; };         // HFA: 2 floats
// struct Vec3 { float x, y, z; };      // HFA: 3 floats
// struct Vec4 { float x, y, z, w; };   // HFA: 4 floats ← Maximum
// struct Vec5 { float x, y, z, w, v;}; // NOT HFA: 5 elements > 4
//
// struct DVec2 { double x, y; };       // HFA: 2 doubles
// struct DVec4 { double x, y, z, w; }; // HFA: 4 doubles ← Maximum

// -------------------------------------------
// float dot2(struct Vec2 a, struct Vec2 b)
//
// HFA Passing:
// a.x → S0, a.y → S1
// b.x → S2, b.y → S3
// Return: S0
// -------------------------------------------
dot2:
    FMUL    S0, S0, S2          // S0 = a.x * b.x
    FMUL    S1, S1, S3          // S1 = a.y * b.y
    FADD    S0, S0, S1          // S0 = a.x*b.x + a.y*b.y
    RET

// -------------------------------------------
// float dot3(struct Vec3 a, struct Vec3 b)
//
// a.x → S0, a.y → S1, a.z → S2
// b.x → S3, b.y → S4, b.z → S5
// Return: S0
// -------------------------------------------
dot3:
    FMUL    S0, S0, S3          // S0 = a.x * b.x
    FMUL    S1, S1, S4          // S1 = a.y * b.y
    FMUL    S2, S2, S5          // S2 = a.z * b.z
    FADD    S0, S0, S1
    FADD    S0, S0, S2
    RET

// -------------------------------------------
// struct Vec3 normalize(struct Vec3 v)
//
// Input: S0=x, S1=y, S2=z
// Return: S0=nx, S1=ny, S2=nz
// -------------------------------------------
normalize3:
    // คำนวณ length squared: x^2 + y^2 + z^2
    FMUL    S3, S0, S0          // S3 = x^2
    FMUL    S4, S1, S1          // S4 = y^2
    FMUL    S5, S2, S2          // S5 = z^2
    FADD    S3, S3, S4          // S3 = x^2 + y^2
    FADD    S3, S3, S5          // S3 = x^2 + y^2 + z^2

    // คำนวณ length = sqrt(length_sq)
    FSQRT   S3, S3              // S3 = length

    // normalize: divide each component by length
    FDIV    S0, S0, S3          // nx = x / length
    FDIV    S1, S1, S3          // ny = y / length
    FDIV    S2, S2, S3          // nz = z / length
    RET

// -------------------------------------------
// Double HFA Example:
// double magnitude(struct DVec4 v)
// D0=x D1=y D2=z D3=w
// -------------------------------------------
magnitude4:
    FMUL    D0, D0, D0
    FMUL    D1, D1, D1
    FMUL    D2, D2, D2
    FMUL    D3, D3, D3
    FADD    D0, D0, D1
    FADD    D0, D0, D2
    FADD    D0, D0, D3
    FSQRT   D0, D0
    RET
```

---

## 12. AArch64 vs x86-64 ABI Comparison

```
╔══════════════════════════════╦═════════════════════╦═══════════════════════╗
║ Feature                      ║ AArch64 (AAPCS64)   ║ x86-64 (System V)    ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Integer arg registers        ║ X0-X7 (8 regs)      ║ RDI,RSI,RDX,RCX,     ║
║                              ║                     ║ R8,R9 (6 regs)       ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ FP/SIMD arg registers        ║ V0-V7 (8 regs)      ║ XMM0-XMM7 (8 regs)  ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Return (integer)             ║ X0 (X0+X1 for 128)  ║ RAX (RAX+RDX for 128)║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Return (floating-point)      ║ V0                  ║ XMM0                 ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Large struct hidden pointer  ║ X8                  ║ RDI (shifts others)  ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Caller-saved                 ║ X0-X18, V0-V7,      ║ RAX,RCX,RDX,RSI,    ║
║                              ║ V16-V31             ║ RDI,R8-R11,XMM0-XMM15║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Callee-saved                 ║ X19-X28, FP, LR*    ║ RBX,RBP,R12-R15     ║
║                              ║ V8-V15 (lower 64bit)║                      ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Stack alignment at call      ║ 16 bytes            ║ 16 bytes             ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Red zone                     ║ NO                  ║ YES (128 bytes)      ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Call instruction             ║ BL dest             ║ CALL dest            ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Return instruction           ║ RET (= BR X30)      ║ RET                  ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Link register                ║ X30 (LR)            ║ [RSP] (on stack)     ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Small struct (<= 16 bytes)   ║ X0+X1 registers     ║ RAX+RDX registers    ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ HFA                          ║ YES (up to 4 elem)  ║ YES (via XMM regs)   ║
╠══════════════════════════════╬═════════════════════╬═══════════════════════╣
║ Varargs register save area   ║ Not mandatory        ║ Yes (XMM0-XMM7)     ║
╚══════════════════════════════╩═════════════════════╩═══════════════════════╝
```

---

## 13. Linux Syscall Convention บน AArch64

```asm
// ===== ตัวอย่างที่ 14: Linux Syscall Convention =====

// -------------------------------------------
// AArch64 Linux Syscall Convention:
// X8  = Syscall number
// X0  = Argument 1
// X1  = Argument 2
// X2  = Argument 3
// X3  = Argument 4
// X4  = Argument 5
// X5  = Argument 6
// SVC #0 → เรียก Kernel
// Return: X0 = Return value (หรือ -errno ถ้า error)
// -------------------------------------------

    .section .data
msg:        .string "Hello, AArch64 Linux!\n"
msg_len     = . - msg

    .section .text
    .global _start

// -------------------------------------------
// _start: Entry point (ไม่ใช้ C library)
// -------------------------------------------
_start:
    // syscall: write(1, msg, msg_len)
    MOV     X8, #64             // SYS_write = 64
    MOV     X0, #1              // fd = STDOUT
    ADRP    X1, msg
    ADD     X1, X1, :lo12:msg   // X1 = &msg
    MOV     X2, #msg_len        // X2 = length
    SVC     #0                  // เรียก Kernel

    // syscall: exit(0)
    MOV     X8, #93             // SYS_exit = 93
    MOV     X0, #0              // exit code = 0
    SVC     #0

// -------------------------------------------
// Syscall Numbers ที่ใช้บ่อยใน AArch64 Linux:
// -------------------------------------------
// SYS_read     = 63
// SYS_write    = 64
// SYS_openat   = 56
// SYS_close    = 57
// SYS_lseek    = 62
// SYS_mmap     = 222
// SYS_munmap   = 215
// SYS_brk      = 214
// SYS_exit     = 93
// SYS_exit_group = 94
// SYS_fork     = 1079 (แตกต่างจาก x86!)
// SYS_clone    = 220
// SYS_execve   = 221
// SYS_getpid   = 172
// SYS_kill     = 129

// -------------------------------------------
// ตัวอย่าง: Custom read() wrapper
// ssize_t sys_read(int fd, void *buf, size_t count)
// -------------------------------------------
sys_read:
    MOV     X8, #63             // SYS_read = 63
    SVC     #0
    RET                         // Return value ใน X0

// -------------------------------------------
// ตัวอย่าง: Custom write() wrapper
// ssize_t sys_write(int fd, const void *buf, size_t count)
// -------------------------------------------
sys_write:
    MOV     X8, #64             // SYS_write = 64
    SVC     #0
    RET

// -------------------------------------------
// ตัวอย่าง: sys_mmap()
// void* sys_mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset)
// X0=addr X1=len X2=prot X3=flags X4=fd X5=offset
// -------------------------------------------
sys_mmap:
    MOV     X8, #222            // SYS_mmap = 222
    SVC     #0
    RET

// -------------------------------------------
// ตัวอย่าง: ตรวจสอบ Error จาก Syscall
// -------------------------------------------
check_syscall_error:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    // เรียก write()
    MOV     X8, #64
    MOV     X0, #1
    ADRP    X1, msg
    ADD     X1, X1, :lo12:msg
    MOV     X2, #msg_len
    SVC     #0

    // ตรวจ error: ถ้า X0 < 0 แสดงว่า error
    TBNZ    X0, #63, syscall_error    // Test Bit N (sign bit) ≠ 0 → error
    // หรือ:
    // CMN X0, #4096   (ถ้า X0 >= -4096 แสดงว่า error)
    // BLS syscall_error

    B       syscall_ok

syscall_error:
    NEG     X0, X0              // X0 = errno (positive)
    // ... handle error ...

syscall_ok:
    LDP     X29, X30, [SP], #16
    RET
```

---

## 14. Complete Program: Calculator with Calling Convention

```asm
// ===== ตัวอย่างที่ 15: Complete Calculator Program =====
// ไฟล์: calc.s
// คอมไพล์: aarch64-linux-gnu-gcc -o calc calc.s
// รัน:     qemu-aarch64 ./calc

    .section .data

welcome:    .string "=== AArch64 Calculator ===\n"
fmt_add:    .string "%ld + %ld = %ld\n"
fmt_sub:    .string "%ld - %ld = %ld\n"
fmt_mul:    .string "%ld * %ld = %ld\n"
fmt_div:    .string "%ld / %ld = %ld\n"
fmt_mod:    .string "%ld %% %ld = %ld\n"
fmt_fp:     .string "%.2f ^ 2 = %.2f\n"
err_div0:   .string "Error: Division by zero!\n"

    .section .text
    .global main

// -------------------------------------------
// long add(long a, long b)
// -------------------------------------------
add:
    ADD     X0, X0, X1
    RET

// -------------------------------------------
// long sub(long a, long b)
// -------------------------------------------
sub:
    SUB     X0, X0, X1
    RET

// -------------------------------------------
// long mul(long a, long b)
// -------------------------------------------
mul:
    MUL     X0, X0, X1
    RET

// -------------------------------------------
// long divmod(long a, long b, long *remainder)
// X0=a X1=b X2=&remainder
// Return: X0 = quotient, *X2 = remainder
// -------------------------------------------
divmod:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    CBZ     X1, div_by_zero     // ตรวจสอบ Division by zero

    // SDIV: Signed Division
    SDIV    X3, X0, X1          // X3 = a / b (quotient)

    // คำนวณ remainder: remainder = a - (quotient * b)
    MSUB    X4, X3, X1, X0      // X4 = a - (X3 * b) = remainder

    STR     X4, [X2]            // *remainder = X4
    MOV     X0, X3              // Return quotient

    LDP     X29, X30, [SP], #16
    RET

div_by_zero:
    ADRP    X0, err_div0
    ADD     X0, X0, :lo12:err_div0
    BL      printf
    MOV     X0, #-1             // Return error code
    LDP     X29, X30, [SP], #16
    RET

// -------------------------------------------
// double square(double x)
// -------------------------------------------
square:
    FMUL    D0, D0, D0
    RET

// -------------------------------------------
// void print_results(long a, long b)
// แสดงผลลัพธ์การคำนวณทั้งหมด
// -------------------------------------------
print_results:
    STP     X29, X30, [SP, #-48]!
    MOV     X29, SP
    STP     X19, X20, [SP, #16]
    STP     X21, X22, [SP, #32]

    MOV     X19, X0             // X19 = a
    MOV     X20, X1             // X20 = b

    // Addition
    MOV     X0, X19
    MOV     X1, X20
    BL      add
    MOV     X21, X0             // X21 = result

    ADRP    X0, fmt_add
    ADD     X0, X0, :lo12:fmt_add
    MOV     X1, X19
    MOV     X2, X20
    MOV     X3, X21
    BL      printf

    // Subtraction
    MOV     X0, X19
    MOV     X1, X20
    BL      sub
    MOV     X21, X0

    ADRP    X0, fmt_sub
    ADD     X0, X0, :lo12:fmt_sub
    MOV     X1, X19
    MOV     X2, X20
    MOV     X3, X21
    BL      printf

    // Multiplication
    MOV     X0, X19
    MOV     X1, X20
    BL      mul
    MOV     X21, X0

    ADRP    X0, fmt_mul
    ADD     X0, X0, :lo12:fmt_mul
    MOV     X1, X19
    MOV     X2, X20
    MOV     X3, X21
    BL      printf

    // Division with remainder
    ADD     X22, SP, #-8        // Address สำหรับเก็บ remainder
    // แต่เดี๋ยว SP ยังไม่ adjust... ใช้ stack frame แทน
    SUB     SP, SP, #16         // จองพื้นที่
    MOV     X0, X19
    MOV     X1, X20
    MOV     X2, SP              // X2 = &remainder
    BL      divmod
    MOV     X21, X0             // quotient
    LDR     X22, [SP]           // remainder
    ADD     SP, SP, #16

    CMP     X21, #-1
    BEQ     skip_div

    ADRP    X0, fmt_div
    ADD     X0, X0, :lo12:fmt_div
    MOV     X1, X19
    MOV     X2, X20
    MOV     X3, X21
    BL      printf

    ADRP    X0, fmt_mod
    ADD     X0, X0, :lo12:fmt_mod
    MOV     X1, X19
    MOV     X2, X20
    MOV     X3, X22
    BL      printf

skip_div:
    LDP     X21, X22, [SP, #32]
    LDP     X19, X20, [SP, #16]
    LDP     X29, X30, [SP], #48
    RET

// -------------------------------------------
// main
// -------------------------------------------
main:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    ADRP    X0, welcome
    ADD     X0, X0, :lo12:welcome
    BL      printf

    // ทดสอบ: 100 และ 7
    MOV     X0, #100
    MOV     X1, #7
    BL      print_results

    // ทดสอบ FP: square(4.5)
    FMOV    D0, #4.5
    BL      square              // D0 = 20.25

    ADRP    X0, fmt_fp
    ADD     X0, X0, :lo12:fmt_fp
    FMOV    D1, #4.5
    // D0 ยังมี 20.25
    // แต่ printf ต้องการ D1=input, D2=output
    FMOV    D2, D0              // D2 = 20.25
    FMOV    D1, #4.5            // D1 = 4.5
    BL      printf

    MOV     X0, #0
    LDP     X29, X30, [SP], #16
    RET
```

---

## 15. ARM32 vs AArch64 Calling Convention

```asm
// ===== ตัวอย่างที่ 16: ARM32 Calling Convention =====
// (AAPCS - Procedure Call Standard for ARM Architecture)

// -------------------------------------------
// ARM32 Calling Convention:
// R0-R3  = Arguments 1-4 (Integer)
// R0-R1  = Return value (R0+R1 สำหรับ 64-bit)
// R0-R3, R12 = Caller-saved
// R4-R11, LR = Callee-saved  (แต่ LR ต้อง Save ถ้าเรียก BL)
// SP (R13) = Stack Pointer (4-byte aligned, 8-byte at call)
// LR (R14) = Link Register
// PC (R15) = Program Counter
//
// Arguments 5+ → Stack
// Floating-point: VFP D0-D7 (ถ้า VFP enabled)
// -------------------------------------------

// ARM32 Assembly (EABI):
// int add_arm32(int a, int b)
// R0 = a, R1 = b
// Return: R0

// .syntax unified
// .arch armv7-a
// .thumb

// thumb_add:
//     ADD R0, R0, R1
//     BX  LR

// arm32_add:
//     ADD R0, R0, R1
//     MOV PC, LR    // หรือ BX LR (สำหรับ interworking)

// -------------------------------------------
// AArch64 Equivalent:
// int add_aarch64(int a, int b)
// X0 = a, X1 = b
// Return: X0
// -------------------------------------------
add_aarch64:
    ADD     W0, W0, W1
    RET

// -------------------------------------------
// ข้อแตกต่างสำคัญ ARM32 vs AArch64:
//
// 1. Register names:
//    ARM32: R0-R15 (32-bit เท่านั้น)
//    AArch64: X0-X30 (64-bit), W0-W30 (32-bit view)
//
// 2. Arguments:
//    ARM32: R0-R3 (4 registers)
//    AArch64: X0-X7 (8 registers)
//
// 3. Return:
//    ARM32: R0 (R0+R1 สำหรับ 64-bit)
//    AArch64: X0 (X0+X1 สำหรับ 128-bit)
//
// 4. FP Registers:
//    ARM32: S0-S31 / D0-D15 (VFP) - ถ้า enable
//    AArch64: V0-V31 (always available)
//
// 5. Stack:
//    ARM32: 4-byte aligned, 8-byte at call
//    AArch64: 16-byte aligned always
//
// 6. Red Zone:
//    ARM32: No red zone
//    AArch64: No red zone
// -------------------------------------------
```

---

## 16. Practical Examples: Interop with C

```asm
// ===== ตัวอย่างที่ 17: C Interoperability =====
// ไฟล์: interop.s
// ใช้งานร่วมกับ interop.c

    .section .text

// -------------------------------------------
// C Prototype: int asm_strlen(const char *s)
// X0 = s (string pointer)
// Return: X0 = length
// -------------------------------------------
    .global asm_strlen
asm_strlen:
    MOV     X1, X0          // X1 = ตัวชี้ปัจจุบัน
strlen_loop:
    LDRB    W2, [X1], #1    // โหลด byte แล้วเลื่อน X1
    CBNZ    W2, strlen_loop  // ถ้าไม่ใช่ null byte ทำต่อ
    SUB     X0, X1, X0      // X0 = X1 - start
    SUB     X0, X0, #1      // -1 เพราะนับ null byte ด้วย
    RET

// -------------------------------------------
// C Prototype: void asm_memcpy(void *dst, const void *src, size_t n)
// X0=dst X1=src X2=n
// -------------------------------------------
    .global asm_memcpy
asm_memcpy:
    CBZ     X2, memcpy_done
memcpy_loop:
    LDRB    W3, [X1], #1    // โหลดจาก src
    STRB    W3, [X0], #1    // เก็บที่ dst
    SUBS    X2, X2, #1
    B.NE    memcpy_loop
memcpy_done:
    RET

// -------------------------------------------
// C Prototype: int asm_strcmp(const char *s1, const char *s2)
// X0=s1 X1=s2
// Return: X0 < 0 if s1 < s2, 0 if equal, > 0 if s1 > s2
// -------------------------------------------
    .global asm_strcmp
asm_strcmp:
strcmp_loop:
    LDRB    W2, [X0], #1    // W2 = *s1++
    LDRB    W3, [X1], #1    // W3 = *s2++
    SUBS    W4, W2, W3      // W4 = *s1 - *s2
    B.NE    strcmp_diff      // ถ้าต่างกัน → return
    CBNZ    W2, strcmp_loop  // ถ้ายังไม่ถึง null → ทำต่อ
    MOV     W0, #0          // Equal
    RET
strcmp_diff:
    SXTB    W0, W4          // Sign-extend result
    RET

// -------------------------------------------
// C Prototype: void *asm_memset(void *s, int c, size_t n)
// X0=s X1=c X2=n
// Return: X0 = s (original pointer)
// -------------------------------------------
    .global asm_memset
asm_memset:
    MOV     X3, X0          // Save original pointer
    CBZ     X2, memset_done
memset_loop:
    STRB    W1, [X3], #1
    SUBS    X2, X2, #1
    B.NE    memset_loop
memset_done:
    // X0 ยังชี้ที่ s (ไม่ได้เปลี่ยน)
    RET
```

```c
// ไฟล์: interop.c
// คอมไพล์: aarch64-linux-gnu-gcc -o interop interop.c interop.s

#include <stdio.h>
#include <assert.h>

// Declarations
extern int   asm_strlen(const char *s);
extern void  asm_memcpy(void *dst, const void *src, size_t n);
extern int   asm_strcmp(const char *s1, const char *s2);
extern void *asm_memset(void *s, int c, size_t n);

int main(void) {
    // ทดสอบ asm_strlen
    const char *hello = "Hello, World!";
    int len = asm_strlen(hello);
    printf("strlen(\"%s\") = %d\n", hello, len);
    assert(len == 13);

    // ทดสอบ asm_memcpy
    char src[] = "Assembly is fun!";
    char dst[32] = {0};
    asm_memcpy(dst, src, sizeof(src));
    printf("memcpy result: \"%s\"\n", dst);

    // ทดสอบ asm_strcmp
    printf("strcmp(\"abc\", \"abc\") = %d\n", asm_strcmp("abc", "abc"));
    printf("strcmp(\"abc\", \"abd\") = %d\n", asm_strcmp("abc", "abd"));
    printf("strcmp(\"abd\", \"abc\") = %d\n", asm_strcmp("abd", "abc"));

    // ทดสอบ asm_memset
    char buf[10];
    asm_memset(buf, 'A', 9);
    buf[9] = '\0';
    printf("memset result: \"%s\"\n", buf);

    printf("All tests passed!\n");
    return 0;
}
```

---

## 17. การ Compile และ Run ด้วย QEMU

```bash
# ===== การ Compile AArch64 Assembly บน x86-64 Host =====

# ติดตั้ง Cross-Compiler และ QEMU (Ubuntu/Debian)
sudo apt-get install gcc-aarch64-linux-gnu qemu-user

# Compile Assembly เป็น Object File
aarch64-linux-gnu-as -o program.o program.s

# Link เป็น Executable
aarch64-linux-gnu-ld -o program program.o

# รันด้วย QEMU User Mode
qemu-aarch64 ./program

# ===== Compile ร่วมกับ C Library =====
aarch64-linux-gnu-gcc -o program program.c program.s

# รันด้วย QEMU (ต้องระบุ sysroot ถ้าใช้ shared library)
qemu-aarch64 -L /usr/aarch64-linux-gnu ./program

# ===== Build Script =====
# ไฟล์: build.sh

#!/bin/bash
CC=aarch64-linux-gnu-gcc
AS=aarch64-linux-gnu-as
LD=aarch64-linux-gnu-ld

# Standalone Assembly (ไม่ใช้ C library)
${AS} -o standalone.o standalone.s
${LD} -o standalone standalone.o
echo "Built standalone"

# Assembly + C Library
${CC} -o with_libc main.c asm_funcs.s
echo "Built with_libc"

# รัน
echo "=== Running standalone ==="
qemu-aarch64 ./standalone

echo "=== Running with_libc ==="
qemu-aarch64 -L /usr/aarch64-linux-gnu ./with_libc

# ===== Debug ด้วย GDB =====
# รัน QEMU ในโหมด Debug (รอ GDB connection ที่ port 1234)
qemu-aarch64 -g 1234 ./program &

# เปิด GDB ใน Terminal ใหม่
aarch64-linux-gnu-gdb ./program
# ใน GDB:
# (gdb) target remote :1234
# (gdb) break main
# (gdb) continue
# (gdb) info registers
# (gdb) x/10x $sp
```

---

## 18. ตัวอย่างขั้นสูง: SIMD ด้วย Calling Convention

```asm
// ===== ตัวอย่างที่ 18: SIMD with Calling Convention =====

// -------------------------------------------
// void vector_add_float(float *dst, const float *a,
//                       const float *b, int n)
// X0=dst X1=a X2=b W3=n
// -------------------------------------------
    .global vector_add_float
vector_add_float:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP

    // Process 4 floats at a time using SIMD
    LSR     W4, W3, #2          // W4 = n / 4 (จำนวน iterations แบบ SIMD)
    CBZ     W4, simd_tail       // ถ้า < 4 elements → ไปที่ scalar tail

simd_loop:
    LD1     {V0.4S}, [X1], #16  // โหลด 4 floats จาก a
    LD1     {V1.4S}, [X2], #16  // โหลด 4 floats จาก b
    FADD    V0.4S, V0.4S, V1.4S // V0 = a + b (4 floats พร้อมกัน)
    ST1     {V0.4S}, [X0], #16  // เก็บ 4 floats ลง dst
    SUBS    W4, W4, #1
    B.NE    simd_loop

simd_tail:
    // จัดการ elements ที่เหลือ (น้อยกว่า 4)
    AND     W3, W3, #3          // W3 = n % 4
    CBZ     W3, simd_done

scalar_loop:
    LDR     S0, [X1], #4
    LDR     S1, [X2], #4
    FADD    S0, S0, S1
    STR     S0, [X0], #4
    SUBS    W3, W3, #1
    B.NE    scalar_loop

simd_done:
    LDP     X29, X30, [SP], #16
    RET

// -------------------------------------------
// V8-V15 Callee-saved (lower 64-bit)
// ตัวอย่างการ Save/Restore SIMD callee-saved
// -------------------------------------------
simd_callee_example:
    STP     X29, X30, [SP, #-80]!
    MOV     X29, SP

    // Save lower 64-bit ของ V8-V11 (callee-saved)
    STP     D8, D9,  [SP, #16]
    STP     D10, D11, [SP, #32]
    // หมายเหตุ: บันทึกเฉพาะ lower 64-bit (D registers)
    // upper 64-bit ไม่ต้อง save!

    // ใช้ V8-V11 ได้อย่างอิสระ
    FMOV    D8, #1.0
    FMOV    D9, #2.0
    FMOV    D10, #3.0
    FMOV    D11, #4.0

    BL      some_function       // V8-V11 ยังเหมือนเดิมหลัง Call

    // Restore
    LDP     D8, D9,  [SP, #16]
    LDP     D10, D11, [SP, #32]
    LDP     X29, X30, [SP], #80
    RET
```

---

## 19. Tail Call Optimization

```asm
// ===== ตัวอย่างที่ 19: Tail Call Optimization =====

// -------------------------------------------
// ปกติ: f() เรียก g() แล้ว Return ค่าของ g()
// -------------------------------------------
normal_call:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP
    BL      target_function     // เรียก และ Return ค่า
    LDP     X29, X30, [SP], #16
    RET                         // Return ค่าที่ได้จาก target_function

// -------------------------------------------
// Tail Call: ถ้า g() เป็น Last Call ใน f()
// สามารถ Optimize เป็น Jump แทน Call ได้
// -------------------------------------------
tail_call_optimized:
    STP     X29, X30, [SP, #-16]!
    MOV     X29, SP
    // ... ทำงานบางอย่าง ...
    LDP     X29, X30, [SP], #16 // Restore ก่อน
    B       target_function     // B แทน BL → Tail Call!
    // ไม่ต้อง RET เพราะ target_function จะ Return ไปยัง caller ของเรา

// -------------------------------------------
// ตัวอย่าง Tail Recursive: sum_to_n_tail
// long sum_tail(long n, long acc)
// X0 = n, X1 = acc
// -------------------------------------------
sum_tail:
    CBZ     X0, sum_tail_base   // n == 0?
    ADD     X1, X1, X0          // acc += n
    SUB     X0, X0, #1          // n--
    B       sum_tail             // Tail call ตัวเอง (ไม่สร้าง Stack frame ใหม่!)

sum_tail_base:
    MOV     X0, X1              // Return acc
    RET

// เรียกใช้: sum_tail(100, 0) → ผลรวม 1+2+...+100 = 5050
// โดยไม่ Stack Overflow แม้ n ใหญ่มาก
```

---

## 20. แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Integer Arguments

```asm
// โจทย์: เขียนฟังก์ชัน max5 ที่รับ 5 integers และ Return ค่าสูงสุด
// C Prototype: long max5(long a, long b, long c, long d, long e)
// X0=a X1=b X2=c X3=d X4=e

// TODO: เขียน implementation ของ max5
max5:
    // 힌트: ใช้ CMP และ CSEL
    // CSEL Xd, Xn, Xm, cond  → Xd = (cond) ? Xn : Xm

    // ขั้นที่ 1: เปรียบเทียบ a และ b
    CMP     X0, X1
    CSEL    X0, X0, X1, GT      // X0 = max(a, b)

    // ขั้นที่ 2: เปรียบเทียบกับ c
    CMP     X0, X2
    CSEL    X0, X0, X2, GT      // X0 = max(max(a,b), c)

    // ขั้นที่ 3: เปรียบเทียบกับ d
    CMP     X0, X3
    CSEL    X0, X0, X3, GT

    // ขั้นที่ 4: เปรียบเทียบกับ e
    CMP     X0, X4
    CSEL    X0, X0, X4, GT

    RET
```

### แบบฝึกหัดที่ 2: Struct Passing

```asm
// โจทย์: เขียนฟังก์ชัน distance ที่คำนวณระยะทางระหว่าง 2 points
// C:
// struct Point { double x; double y; };
// double distance(struct Point a, struct Point b);
//
// HFA Passing: a.x=D0 a.y=D1 b.x=D2 b.y=D3

distance:
    // TODO: คำนวณ sqrt((a.x-b.x)^2 + (a.y-b.y)^2)
    FSUB    D0, D0, D2          // D0 = a.x - b.x
    FSUB    D1, D1, D3          // D1 = a.y - b.y
    FMUL    D0, D0, D0          // D0 = dx^2
    FMUL    D1, D1, D1          // D1 = dy^2
    FADD    D0, D0, D1          // D0 = dx^2 + dy^2
    FSQRT   D0, D0              // D0 = sqrt(...)
    RET
```

### แบบฝึกหัดที่ 3: Syscall Implementation

```asm
// โจทย์: เขียน Program แบบ Standalone (ไม่ใช้ C library)
// ที่อ่าน string จาก stdin แล้วพิมพ์กลับในรูป uppercase

    .section .bss
buffer:     .space 256

    .section .data
newline:    .byte 10

    .section .text
    .global _start

_start:
    // อ่านจาก stdin
    MOV     X8, #63             // SYS_read
    MOV     X0, #0              // fd = stdin
    ADRP    X1, buffer
    ADD     X1, X1, :lo12:buffer
    MOV     X2, #255            // max bytes
    SVC     #0
    MOV     X9, X0              // X9 = bytes read

    // แปลงเป็น uppercase
    ADRP    X1, buffer
    ADD     X1, X1, :lo12:buffer
    MOV     X10, X9             // counter

upper_loop:
    CBZ     X10, upper_done
    LDRB    W2, [X1]
    CMP     W2, #'a'
    B.LT    not_lower
    CMP     W2, #'z'
    B.GT    not_lower
    SUB     W2, W2, #32         // 'a' - 'A' = 32
    STRB    W2, [X1]
not_lower:
    ADD     X1, X1, #1
    SUB     X10, X10, #1
    B       upper_loop

upper_done:
    // พิมพ์ผลลัพธ์
    MOV     X8, #64             // SYS_write
    MOV     X0, #1              // fd = stdout
    ADRP    X1, buffer
    ADD     X1, X1, :lo12:buffer
    MOV     X2, X9
    SVC     #0

    // Exit
    MOV     X8, #93
    MOV     X0, #0
    SVC     #0
```

### แบบฝึกหัดที่ 4: Callee-Saved Registers

```asm
// โจทย์: เขียน fibonacci(n) แบบ Iterative ที่ใช้ Callee-Saved Registers
// C Prototype: long fibonacci(long n)
// ต้องใช้ X19-X21 และ Restore อย่างถูกต้อง

fibonacci:
    STP     X29, X30, [SP, #-48]!
    MOV     X29, SP
    STP     X19, X20, [SP, #16]
    STR     X21, [SP, #32]

    MOV     X19, X0             // X19 = n
    MOV     X20, #0             // X20 = a = fib(0)
    MOV     X21, #1             // X21 = b = fib(1)

    // Base cases
    CMP     X19, #0
    BEQ     fib_return_a
    CMP     X19, #1
    BEQ     fib_return_b

    SUB     X19, X19, #1        // n -= 1 (เริ่มจาก 1 เพราะเริ่มมี b อยู่แล้ว)

fib_loop:
    CBZ     X19, fib_done
    ADD     X0, X20, X21        // X0 = a + b
    MOV     X20, X21            // a = b
    MOV     X21, X0             // b = a + b
    SUB     X19, X19, #1
    B       fib_loop

fib_done:
    MOV     X0, X21             // Return b = fib(n)
    B       fib_restore

fib_return_a:
    MOV     X0, X20
    B       fib_restore

fib_return_b:
    MOV     X0, X21

fib_restore:
    LDR     X21, [SP, #32]
    LDP     X19, X20, [SP, #16]
    LDP     X29, X30, [SP], #48
    RET
```

---

## 21. Summary Table: AAPCS64 Quick Reference

```
╔══════════════════════════════════════════════════════════════════╗
║                 AAPCS64 Quick Reference Card                     ║
╠══════════════════════════════════════════════════════════════════╣
║  Function Arguments (Integer/Pointer):                           ║
║  1st  → X0    5th  → X4                                         ║
║  2nd  → X1    6th  → X5                                         ║
║  3rd  → X2    7th  → X6                                         ║
║  4th  → X3    8th  → X7                                         ║
║  9th+ → Stack (8-byte aligned, pushed right-to-left equivalent)  ║
╠══════════════════════════════════════════════════════════════════╣
║  Function Arguments (Floating-Point):                            ║
║  1st  → V0/D0/S0    5th  → V4/D4/S4                            ║
║  2nd  → V1/D1/S1    6th  → V5/D5/S5                            ║
║  3rd  → V2/D2/S2    7th  → V6/D6/S6                            ║
║  4th  → V3/D3/S3    8th  → V7/D7/S7                            ║
╠══════════════════════════════════════════════════════════════════╣
║  Return Values:                                                   ║
║  Integer ≤ 64-bit  → X0                                         ║
║  Integer = 128-bit → X0 (low) + X1 (high)                      ║
║  Float (single)    → S0                                          ║
║  Float (double)    → D0                                          ║
║  Small struct ≤16  → X0 (+X1 if needed)                        ║
║  Large struct >16  → *X8 (caller allocates, passes in X8)       ║
║  HFA (≤4 FP)      → V0-V3                                       ║
╠══════════════════════════════════════════════════════════════════╣
║  Caller-Saved (Scratch):                                         ║
║  X0-X18, V0-V7, V16-V31                                        ║
╠══════════════════════════════════════════════════════════════════╣
║  Callee-Saved (Preserved):                                       ║
║  X19-X28, X29(FP), X30(LR), SP                                 ║
║  V8-V15 (lower 64-bit only, upper can be trashed)              ║
╠══════════════════════════════════════════════════════════════════╣
║  Stack:                                                           ║
║  - SP must be 16-byte aligned at every BL/SVC                   ║
║  - No Red Zone (unlike x86-64 Linux)                            ║
║  - Grows downward (SP decreases)                                 ║
╠══════════════════════════════════════════════════════════════════╣
║  Linux Syscall:                                                   ║
║  X8 = syscall number                                             ║
║  X0-X5 = up to 6 arguments                                      ║
║  SVC #0 = invoke kernel                                          ║
║  X0 = return value / -errno on error                            ║
╠══════════════════════════════════════════════════════════════════╣
║  Special Registers:                                               ║
║  X8  = Indirect result (large struct return) / Syscall #        ║
║  X16 = IP0 (Intra-procedure-call scratch)                       ║
║  X17 = IP1 (Intra-procedure-call scratch)                       ║
║  X18 = Platform register (avoid in portable code)               ║
║  X29 = FP (Frame Pointer)                                        ║
║  X30 = LR (Link Register)                                        ║
║  XZR = Zero Register                                             ║
║  SP  = Stack Pointer                                             ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 22. Function Prologue/Epilogue Templates

```asm
// ===== Template 1: Leaf Function (ไม่เรียก BL) =====
leaf_func_template:
    // ไม่ต้อง Save LR (ไม่มี BL)
    // ใช้เฉพาะ Caller-saved registers (X0-X18, V0-V7, V16-V31)
    // ไม่ต้องแตะ SP
    ADD     X0, X0, X1
    RET

// ===== Template 2: Non-Leaf Function (มี BL) =====
non_leaf_template:
    STP     X29, X30, [SP, #-16]!   // Save FP, LR
    MOV     X29, SP                  // Set FP
    // ... body ...
    LDP     X29, X30, [SP], #16     // Restore FP, LR
    RET

// ===== Template 3: Non-Leaf with Local Variables =====
with_locals_template:
    // สมมติต้องการ 32 bytes สำหรับ local vars
    STP     X29, X30, [SP, #-48]!   // 16 (FP+LR) + 32 (locals) → 48 (align 16)
    MOV     X29, SP
    // Local vars ที่ [SP+16], [SP+24], [SP+32], [SP+40]
    // ...
    LDP     X29, X30, [SP], #48
    RET

// ===== Template 4: Non-Leaf with Callee-Saved Registers =====
with_callee_saved_template:
    STP     X29, X30, [SP, #-48]!
    MOV     X29, SP
    STP     X19, X20, [SP, #16]     // Save X19, X20
    STP     X21, X22, [SP, #32]     // Save X21, X22
    // ใช้ X19-X22 ได้อย่างอิสระ
    // ...
    LDP     X21, X22, [SP, #32]     // Restore ย้อนกลับ (LIFO)
    LDP     X19, X20, [SP, #16]
    LDP     X29, X30, [SP], #48
    RET

// ===== Template 5: SIMD Function with Callee-Saved V Registers =====
with_simd_template:
    STP     X29, X30, [SP, #-48]!
    MOV     X29, SP
    STP     D8,  D9,  [SP, #16]     // Save lower 64-bit ของ V8, V9
    STP     D10, D11, [SP, #32]     // Save lower 64-bit ของ V10, V11
    // ใช้ Q8-Q11 (128-bit) ได้ แต่ upper 64-bit ต้อง restore เอง
    // ง่ายกว่า: ใช้ V0-V7, V16-V31 แทนถ้าทำได้
    // ...
    LDP     D10, D11, [SP, #32]
    LDP     D8,  D9,  [SP, #16]
    LDP     X29, X30, [SP], #48
    RET
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **Register Roles**: X0-X7 สำหรับ Integer Args, V0-V7 สำหรับ FP Args
2. **Return Values**: X0 (หรือ X0+X1 สำหรับ 128-bit), V0 สำหรับ FP
3. **Caller vs Callee Saved**: ทำความเข้าใจว่า Register ไหน ใครต้อง Save
4. **Frame Pointer (X29)**: สร้าง Linked List ของ Stack Frames
5. **Link Register (X30)**: เก็บ Return Address
6. **16-byte Stack Alignment**: กฎที่ต้องทำตามเสมอ
7. **No Red Zone**: ต้องจอง Stack ก่อนใช้
8. **Struct Passing**: เล็ก (≤16 bytes) ผ่าน Register, ใหญ่ผ่าน Pointer
9. **HFA**: Struct ที่มี FP members เหมือนกัน ≤4 ตัว ส่งผ่าน V Registers
10. **Linux Syscall**: X8=number, X0-X5=args, SVC #0
11. **QEMU**: ใช้ทดสอบ AArch64 บน x86-64 Host

---

**ต่อไป → Part 040: ARM64 SIMD/NEON Programming Basics**

---
*หลักสูตร Assembly Programming: Part 039/100+*
*ระดับ: Intermediate-Advanced*
