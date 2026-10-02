# Part 095: Embedded Systems — ARM Cortex-M Architecture

## บทนำ (Introduction)

ARM Cortex-M เป็น processor family ที่ออกแบบมาเฉพาะสำหรับ microcontroller และ embedded systems โดยมีจุดเด่นคือ:
- **Low power consumption** — เหมาะสำหรับ battery-powered devices
- **Deterministic interrupt latency** — critical สำหรับ real-time systems
- **Thumb-2 instruction set** — code density สูง + performance ดี
- **Built-in debug support** — CoreSight debug architecture

ในบทนี้เราจะศึกษาตั้งแต่ architecture ระดับ silicon ไปจนถึงการเขียน assembly สำหรับ context switching ใน RTOS

---

## 1. Cortex-M Family Overview

### 1.1 สายผลิตภัณฑ์และ ISA

| Core     | ISA        | Pipeline  | FPU | MPU | DSP Ext | Use Case                    |
|----------|------------|-----------|-----|-----|---------|-----------------------------|
| Cortex-M0 | ARMv6-M   | 2-stage   | No  | No  | No      | Ultra-low power, simple MCU |
| Cortex-M0+| ARMv6-M   | 2-stage   | No  | Opt | No      | M0 + lower power, MTB       |
| Cortex-M1 | ARMv6-M   | FPGA      | No  | No  | No      | FPGA soft-core              |
| Cortex-M3 | ARMv7-M   | 3-stage   | No  | Opt | No      | General purpose MCU         |
| Cortex-M4 | ARMv7E-M  | 3-stage   | Opt | Opt | Yes     | DSP + control applications  |
| Cortex-M7 | ARMv7E-M  | 6-stage   | Yes | Opt | Yes     | High performance MCU        |
| Cortex-M23| ARMv8-M   | 2-stage   | No  | Opt | No      | TrustZone security          |
| Cortex-M33| ARMv8-M   | 3-stage   | Opt | Opt | Yes     | Secure IoT applications     |
| Cortex-M55| ARMv8.1-M | 4-stage   | Yes | Yes | Yes     | ML/AI at the edge           |

### 1.2 Thumb-2 Instruction Set — ไม่มี ARM32 State

Cortex-M **ทำงานได้เฉพาะใน Thumb state** เท่านั้น ไม่สามารถ switch ไป ARM32 (32-bit ARM instructions) ได้

เหตุผล:
- ลด silicon area (ไม่ต้องมี ARM32 decoder)
- Thumb-2 รองรับทั้ง 16-bit และ 32-bit instructions
- Code density ดีกว่า ARM32 ประมาณ 26%

```
Thumb-2 Instruction Format:
- 16-bit instructions: ใช้ registers R0-R7 (low registers) เป็นหลัก
- 32-bit instructions: ใช้ registers R0-R15 ได้ทั้งหมด

ตัวอย่าง 16-bit vs 32-bit:
  ADDS R0, R1, R2      ; 16-bit (3 registers, low regs only)
  ADD.W R0, R1, #12345 ; 32-bit (immediate ใหญ่เกิน 3-bit)
```

### 1.3 Thumb Interworking และ BLX/BX

แม้ Cortex-M จะไม่มี ARM state แต่ยังคงใช้ **bit[0] ของ branch address** เป็น Thumb indicator:

```asm
; ถ้า bit[0] = 1 → Thumb mode (ถูกต้องสำหรับ Cortex-M)
; ถ้า bit[0] = 0 → ARM mode  (จะเกิด UsageFault บน Cortex-M!)

; Vector table entries ต้องมี bit[0] = 1 เสมอ:
.word reset_handler + 1    ; LSB set = Thumb mode
.word nmi_handler + 1
```

---

## 2. Cortex-M Register File

### 2.1 General Purpose Registers

```
Register Map:
┌─────────────────────────────────────────────┐
│  R0  │ General purpose / argument 1 / return │
│  R1  │ General purpose / argument 2          │
│  R2  │ General purpose / argument 3          │
│  R3  │ General purpose / argument 4          │
│  R4  │ General purpose (callee-saved)        │
│  R5  │ General purpose (callee-saved)        │
│  R6  │ General purpose (callee-saved)        │
│  R7  │ General purpose (callee-saved)        │
│  R8  │ General purpose (callee-saved)        │
│  R9  │ General purpose / platform register   │
│  R10 │ General purpose (callee-saved)        │
│  R11 │ General purpose / frame pointer       │
│  R12 │ Intra-procedure-call scratch register │
│  R13 │ Stack Pointer (SP) — banked           │
│  R14 │ Link Register (LR)                    │
│  R15 │ Program Counter (PC)                  │
└─────────────────────────────────────────────┘
```

### 2.2 AAPCS (ARM Procedure Call Standard)

```
Caller-saved (scratch) registers: R0-R3, R12
Callee-saved registers:           R4-R11
Stack Pointer:                    R13 (SP)
Link Register:                    R14 (LR)
Program Counter:                  R15 (PC)

Function call convention:
- Arguments 1-4 → R0, R1, R2, R3
- Arguments 5+  → stack
- Return value  → R0 (32-bit) หรือ R0:R1 (64-bit)
- SP ต้อง 8-byte aligned ก่อน AAPCS call
```

### 2.3 xPSR — Program Status Register

xPSR เป็น 32-bit register ที่รวม 3 registers ย่อยเข้าด้วยกัน:

```
xPSR[31:0]:
┌─────┬─────┬─────┬─────┬────────┬────────────┬──────────────┐
│ N   │ Z   │ C   │ V   │ Q      │ ICI/IT     │ ISR_NUMBER   │
│[31] │[30] │[29] │[28] │[27]    │[26:25,15:10]│[8:0]        │
└─────┴─────┴─────┴─────┴────────┴────────────┴──────────────┘

APSR (Application PSR) — bits[31:27]:
  N = Negative flag
  Z = Zero flag
  C = Carry flag
  V = Overflow flag
  Q = Saturation flag (ARMv7E-M only)

IPSR (Interrupt PSR) — bits[8:0]:
  ISR_NUMBER = 0          → Thread mode
  ISR_NUMBER = 1          → Reset
  ISR_NUMBER = 2          → NMI
  ISR_NUMBER = 3          → HardFault
  ISR_NUMBER = 4          → MemManage
  ISR_NUMBER = 5          → BusFault
  ISR_NUMBER = 6          → UsageFault
  ISR_NUMBER = 11         → SVCall
  ISR_NUMBER = 14         → PendSV
  ISR_NUMBER = 15         → SysTick
  ISR_NUMBER = 16-255     → IRQ0-IRQ239

EPSR (Execution PSR) — bits[26:24,15:10]:
  T = Thumb state bit [24] → ต้องเป็น 1 เสมอบน Cortex-M
  ICI/IT = interrupt-continuable instruction / IF-THEN state
```

### 2.4 PRIMASK, FAULTMASK, BASEPRI

```asm
; Interrupt masking registers

; PRIMASK — bit[0]: ถ้า 1 จะ disable interrupts ทุกตัวยกเว้น NMI/HardFault
CPSID I          ; Set PRIMASK = 1 (disable configurable interrupts)
CPSIE I          ; Clear PRIMASK = 0 (enable interrupts)

MRS R0, PRIMASK  ; Read PRIMASK → R0
MSR PRIMASK, R0  ; Write R0 → PRIMASK

; FAULTMASK — bit[0]: disable แม้แต่ HardFault (ยกเว้น NMI)
CPSID F          ; Set FAULTMASK = 1
CPSIE F          ; Clear FAULTMASK = 0

; BASEPRI — กำหนด priority threshold (ARMv7-M เท่านั้น)
; interrupt ที่มี priority <= BASEPRI จะถูก mask
MOV R0, #0x40    ; Priority threshold = 64
MSR BASEPRI, R0
MOV R0, #0
MSR BASEPRI, R0  ; Disable masking (BASEPRI = 0)
```

---

## 3. Stack Pointers — MSP และ PSP

### 3.1 Banked Stack Pointers

Cortex-M มี stack pointer 2 ตัว:

```
MSP (Main Stack Pointer):
- ใช้ใน Handler mode (interrupt handlers)
- ใช้ใน Thread mode ถ้า CONTROL[1] = 0
- Reset value = ค่าจากตำแหน่ง 0x00000000 ใน vector table

PSP (Process Stack Pointer):
- ใช้ใน Thread mode ถ้า CONTROL[1] = 1
- RTOS kernel จะแยก stack ของ task ออกจาก interrupt stack
- ทำให้สามารถ detect stack overflow ของ task ได้ง่าย
```

### 3.2 CONTROL Register

```
CONTROL[2]: FPCA (Floating-Point Context Active) — ARMv7E-M
  0 = FP not used in current context
  1 = FP used, need to save FP registers on exception

CONTROL[1]: SPSEL (Stack Pointer Select)
  0 = MSP is used as current SP
  1 = PSP is used as current SP (Thread mode only)

CONTROL[0]: nPRIV (Thread mode privilege)
  0 = Thread mode is privileged
  1 = Thread mode is unprivileged

การใช้งาน:
```

```asm
; อ่าน CONTROL register
MRS R0, CONTROL

; สลับไปใช้ PSP ใน Thread mode
MRS R0, CONTROL
ORR R0, R0, #0x02    ; Set SPSEL bit
MSR CONTROL, R0
ISB                   ; Instruction Synchronization Barrier (จำเป็น!)

; ตั้ง PSP ก่อน switch
LDR R0, =task_stack_top
MSR PSP, R0
```

### 3.3 Stack Frame บน Exception Entry

เมื่อ Cortex-M เข้า exception handler มันจะ **auto-save** (push) registers ต่อไปนี้ลง stack อัตโนมัติ:

```
Exception Stack Frame (without FPU):
┌──────────────────┐ ← SP ก่อน exception (aligned ที่ 8 bytes)
│  xPSR            │ +28  (bit[9] = 1 ถ้า SP ต้องการ align)
│  PC (return addr)│ +24
│  LR (R14)        │ +20
│  R12             │ +16
│  R3              │ +12
│  R2              │ +8
│  R1              │ +4
│  R0              │ +0   ← SP หลัง exception (new SP)
└──────────────────┘

Extended Stack Frame (with FPU, CONTROL[2]=1):
┌──────────────────┐
│  FPSCR           │ +104
│  S15             │ +100
│  ...             │
│  S0              │ +68
│  xPSR            │ +64
│  PC              │ +60
│  LR              │ +56
│  R12             │ +52
│  R3              │ +48
│  R2              │ +44
│  R1              │ +40
│  R0              │ +36
│  Reserved        │ +32 (alignment)
└──────────────────┘
```

### 3.4 EXC_RETURN — Magic LR Values

เมื่อ Cortex-M เข้า exception, LR จะถูกตั้งเป็นค่า EXC_RETURN พิเศษ:

```
EXC_RETURN values:
0xFFFFFFF1 = Return to Handler mode, MSP, no FPU
0xFFFFFFF9 = Return to Thread mode, MSP, no FPU
0xFFFFFFFD = Return to Thread mode, PSP, no FPU
0xFFFFFFE1 = Return to Handler mode, MSP, FPU (M4/M7)
0xFFFFFFE9 = Return to Thread mode, MSP, FPU (M4/M7)
0xFFFFFFED = Return to Thread mode, PSP, FPU (M4/M7)

การใช้งาน:
```

```asm
my_exception_handler:
    ; ... do work ...
    BX LR           ; LR = EXC_RETURN → hardware จัดการ return
    ; หรือ
    POP {PC}        ; Pop saved PC → return from exception
```

---

## 4. NVIC — Nested Vectored Interrupt Controller

### 4.1 NVIC Overview

```
NVIC Features:
- รองรับ interrupt สูงสุด 240 external IRQs (IRQ0-IRQ239)
- Priority levels: 8-256 levels ขึ้นอยู่กับ implementation
  (STM32F4: 16 levels = 4-bit priority field)
- Nested interrupts: interrupt priority สูงกว่าสามารถ preempt ได้
- Tail-chaining: ลด overhead เมื่อ interrupts เกิดต่อเนื่องกัน
- Late-arrival: รับ interrupt priority สูงกว่าระหว่าง stacking
```

### 4.2 NVIC Registers

```
NVIC Base Address: 0xE000E100

NVIC_ISER[0-7]  (0xE000E100): Interrupt Set Enable Register
NVIC_ICER[0-7]  (0xE000E180): Interrupt Clear Enable Register
NVIC_ISPR[0-7]  (0xE000E200): Interrupt Set Pending Register
NVIC_ICPR[0-7]  (0xE000E280): Interrupt Clear Pending Register
NVIC_IABR[0-7]  (0xE000E300): Interrupt Active Bit Register
NVIC_IPR[0-59]  (0xE000E400): Interrupt Priority Registers (8-bit per IRQ)

SCB Base Address: 0xE000ED00
SCB_VTOR        (0xE000ED08): Vector Table Offset Register
SCB_AIRCR       (0xE000ED0C): Application Interrupt/Reset Control
SCB_SHPR[0-2]   (0xE000ED18): System Handler Priority Registers
```

### 4.3 Priority Configuration

```c
/* STM32F4: 4-bit priority (16 levels), stored in bits[7:4] ของ NVIC_IPR */

/* Priority grouping (AIRCR[10:8]):
   PRIGROUP=0: 7 preempt bits, 1 subpriority bit
   PRIGROUP=3: 4 preempt bits, 4 subpriority bits (default HAL)
   PRIGROUP=4: 3 preempt bits, 5 subpriority bits
   ...
   PRIGROUP=7: 0 preempt bits, 8 subpriority bits (no preemption)
*/

/* Enable IRQ35 (TIM2_IRQn on STM32F4) with priority 5 */
#define NVIC_ISER1  (*(volatile uint32_t*)0xE000E104)
#define NVIC_IPR8   (*(volatile uint32_t*)0xE000E420)

/* Enable TIM2 interrupt (bit 3 of ISER1, since 35-32=3) */
NVIC_ISER1 = (1UL << (35 - 32));

/* Set priority: IPR register byte = IRQ / 4, byte offset = IRQ % 4 */
/* Priority value shifted to bits[7:4] */
uint8_t *ipr = (uint8_t*)0xE000E400;
ipr[35] = (5 << 4);  /* priority 5, stored in upper 4 bits */
```

### 4.4 Assembly NVIC Enable/Disable

```asm
@ Enable IRQ0 (bit 0 of NVIC_ISER0)
LDR R0, =0xE000E100     @ NVIC_ISER0
MOV R1, #1
STR R1, [R0]

@ Enable IRQ32 (bit 0 of NVIC_ISER1)
LDR R0, =0xE000E104     @ NVIC_ISER1
MOV R1, #1
STR R1, [R0]

@ Disable IRQ5 (bit 5 of NVIC_ICER0)
LDR R0, =0xE000E180     @ NVIC_ICER0
MOV R1, #(1 << 5)
STR R1, [R0]

@ Trigger software interrupt on IRQ0 (set pending)
LDR R0, =0xE000E200     @ NVIC_ISPR0
MOV R1, #1
STR R1, [R0]
```

---

## 5. Vector Table

### 5.1 Vector Table Layout

Vector table เริ่มต้นที่ address 0x00000000 (หรือ relocated ผ่าน VTOR):

```
Offset  Exception Number  Exception Type
0x0000  -                 Initial SP value (MSP)
0x0004  1                 Reset Handler
0x0008  2                 NMI Handler
0x000C  3                 HardFault Handler
0x0010  4                 MemManage Handler (MPU Fault)
0x0014  5                 BusFault Handler
0x0018  6                 UsageFault Handler
0x001C  7-10              Reserved
0x002C  11                SVCall Handler (SVC)
0x0030  12                Debug Monitor Handler
0x0034  13                Reserved
0x0038  14                PendSV Handler
0x003C  15                SysTick Handler
0x0040  16 (IRQ0)         External Interrupt 0
0x0044  17 (IRQ1)         External Interrupt 1
...
0x043C  255 (IRQ239)      External Interrupt 239
```

### 5.2 Vector Table ใน Assembly (GNU Assembler)

```asm
/* startup.s — Cortex-M4 startup file */
    .syntax unified
    .cpu cortex-m4
    .fpu softvfp
    .thumb

    .section .isr_vector, "a", %progbits
    .type g_pfnVectors, %object
    .size g_pfnVectors, .-g_pfnVectors

g_pfnVectors:
    /* Initial Stack Pointer — defined in linker script */
    .word _estack

    /* Cortex-M exception vectors */
    .word Reset_Handler          /* Reset                    */
    .word NMI_Handler            /* NMI                      */
    .word HardFault_Handler      /* Hard Fault               */
    .word MemManage_Handler      /* MPU Fault                */
    .word BusFault_Handler       /* Bus Fault                */
    .word UsageFault_Handler     /* Usage Fault              */
    .word 0                      /* Reserved                 */
    .word 0                      /* Reserved                 */
    .word 0                      /* Reserved                 */
    .word 0                      /* Reserved                 */
    .word SVC_Handler            /* SVCall                   */
    .word DebugMon_Handler       /* Debug Monitor            */
    .word 0                      /* Reserved                 */
    .word PendSV_Handler         /* PendSV                   */
    .word SysTick_Handler        /* SysTick                  */

    /* STM32F4 external interrupts (IRQ0-IRQ90+) */
    .word WWDG_IRQHandler
    .word PVD_IRQHandler
    .word TAMP_STAMP_IRQHandler
    .word RTC_WKUP_IRQHandler
    .word FLASH_IRQHandler
    .word RCC_IRQHandler
    .word EXTI0_IRQHandler
    .word EXTI1_IRQHandler
    .word EXTI2_IRQHandler
    .word EXTI3_IRQHandler
    .word EXTI4_IRQHandler
    .word DMA1_Stream0_IRQHandler
    /* ... ต่อไปจนครบ ... */
    .word TIM2_IRQHandler
    .word TIM3_IRQHandler
    .word TIM4_IRQHandler
    .word USART1_IRQHandler
    .word USART2_IRQHandler
    .word USART3_IRQHandler
```

### 5.3 Weak Default Handlers

```asm
/* Default handlers — weak aliases ให้ user override */
    .weak NMI_Handler
    .thumb_set NMI_Handler, Default_Handler

    .weak HardFault_Handler
    .thumb_set HardFault_Handler, Default_Handler

    .weak MemManage_Handler
    .thumb_set MemManage_Handler, Default_Handler

    .weak BusFault_Handler
    .thumb_set BusFault_Handler, Default_Handler

    .weak UsageFault_Handler
    .thumb_set UsageFault_Handler, Default_Handler

    .weak SVC_Handler
    .thumb_set SVC_Handler, Default_Handler

    .weak PendSV_Handler
    .thumb_set PendSV_Handler, Default_Handler

    .weak SysTick_Handler
    .thumb_set SysTick_Handler, Default_Handler

/* Default_Handler — infinite loop สำหรับ unhandled exceptions */
    .section .text.Default_Handler,"ax",%progbits
Default_Handler:
Infinite_Loop:
    B Infinite_Loop
    .size Default_Handler, .-Default_Handler
```

### 5.4 Reset Handler

```asm
    .section .text.Reset_Handler
    .weak Reset_Handler
    .type Reset_Handler, %function
Reset_Handler:
    /* ถ้ามี FPU, enable มันก่อน */
    LDR   R0, =0xE000ED88        /* CPACR address */
    LDR   R1, [R0]
    ORR   R1, R1, #(0xF << 20)  /* Enable CP10 and CP11 (full access) */
    STR   R1, [R0]

    /* Copy .data section จาก FLASH → RAM */
    LDR   R0, =_sdata            /* Destination (RAM start) */
    LDR   R1, =_edata            /* Destination end */
    LDR   R2, =_sidata           /* Source (FLASH load address) */
    MOVS  R3, #0
    B     LoopCopyDataInit

CopyDataInit:
    LDR   R4, [R2, R3]
    STR   R4, [R0, R3]
    ADDS  R3, R3, #4

LoopCopyDataInit:
    ADDS  R4, R0, R3
    CMP   R4, R1
    BCC   CopyDataInit

    /* Zero-fill .bss section */
    LDR   R2, =_sbss
    LDR   R4, =_ebss
    MOVS  R3, #0
    B     LoopFillZerobss

FillZerobss:
    STR   R3, [R2]
    ADDS  R2, R2, #4

LoopFillZerobss:
    CMP   R2, R4
    BCC   FillZerobss

    /* เรียก SystemInit (clock setup) */
    BL    SystemInit

    /* เรียก __libc_init_array (C++ constructors) */
    BL    __libc_init_array

    /* กระโดดไป main */
    BL    main

    /* ถ้า main return (ไม่ควรเกิด) */
LoopForever:
    B     LoopForever

    .size Reset_Handler, .-Reset_Handler
```

---

## 6. SysTick Timer

### 6.1 SysTick Architecture

SysTick เป็น 24-bit countdown timer ที่ built-in ใน Cortex-M core:

```
SysTick Registers (base: 0xE000E010):
┌──────────────────────────────────────────────────────────┐
│  SYST_CSR  (0xE000E010) — Control and Status Register   │
│  SYST_RVR  (0xE000E014) — Reload Value Register         │
│  SYST_CVR  (0xE000E018) — Current Value Register        │
│  SYST_CALIB(0xE000E01C) — Calibration Register          │
└──────────────────────────────────────────────────────────┘

SYST_CSR bits:
[16] COUNTFLAG — เป็น 1 เมื่อ counter ถึง 0 (cleared on read)
[2]  CLKSOURCE — 0=external ref clock, 1=processor clock
[1]  TICKINT   — 0=no interrupt, 1=trigger SysTick exception
[0]  ENABLE    — 0=disabled, 1=enabled
```

### 6.2 SysTick Configuration ใน Assembly

```asm
/* Setup SysTick สำหรับ 1ms tick ที่ 168MHz (STM32F4) */
/* Reload value = (168,000,000 / 1000) - 1 = 167,999 */

SysTick_Config:
    LDR  R0, =0xE000E010         @ SysTick base address

    @ Disable SysTick ก่อน configure
    MOV  R1, #0
    STR  R1, [R0, #0]            @ SYST_CSR = 0

    @ Set reload value
    LDR  R1, =167999             @ 168MHz / 1000Hz - 1
    STR  R1, [R0, #4]            @ SYST_RVR

    @ Clear current value
    MOV  R1, #0
    STR  R1, [R0, #8]            @ SYST_CVR = 0

    @ Enable: processor clock + interrupt + enable
    MOV  R1, #0x07               @ CLKSOURCE=1, TICKINT=1, ENABLE=1
    STR  R1, [R0, #0]            @ SYST_CSR

    BX   LR
```

### 6.3 SysTick Handler

```asm
    .section .text.SysTick_Handler
    .global SysTick_Handler
    .type SysTick_Handler, %function
SysTick_Handler:
    PUSH {LR}

    /* Increment tick counter */
    LDR  R0, =g_tick_count
    LDR  R1, [R0]
    ADDS R1, R1, #1
    STR  R1, [R0]

    /* Signal RTOS tick (set PendSV pending for context switch) */
    LDR  R0, =0xE000ED04         @ ICSR (Interrupt Control State Register)
    LDR  R1, =0x10000000         @ PENDSVSET bit[28]
    STR  R1, [R0]

    POP  {PC}
    .size SysTick_Handler, .-SysTick_Handler
```

---

## 7. Fault Handlers

### 7.1 HardFault Handler

```asm
    .section .text.HardFault_Handler
    .global HardFault_Handler
    .type HardFault_Handler, %function
HardFault_Handler:
    /* ตรวจสอบว่า fault เกิดจาก Thread mode (PSP) หรือ Handler mode (MSP) */
    TST   LR, #4                 @ Test bit 2 of EXC_RETURN
    ITE   EQ
    MRSEQ R0, MSP                @ EXC_RETURN bit2=0 → use MSP
    MRSNE R0, PSP                @ EXC_RETURN bit2=1 → use PSP

    /* R0 = pointer to stacked exception frame */
    /* ส่ง fault info ไปยัง C handler */
    MOV   R1, LR                 @ Pass EXC_RETURN as second arg
    B     hard_fault_handler_c   @ Branch to C function

    .size HardFault_Handler, .-HardFault_Handler
```

```c
/* C portion of HardFault handler */
void hard_fault_handler_c(uint32_t *stack_frame, uint32_t exc_return)
{
    /* Stack frame layout (stacked by hardware) */
    uint32_t r0  = stack_frame[0];
    uint32_t r1  = stack_frame[1];
    uint32_t r2  = stack_frame[2];
    uint32_t r3  = stack_frame[3];
    uint32_t r12 = stack_frame[4];
    uint32_t lr  = stack_frame[5];
    uint32_t pc  = stack_frame[6];  /* ค่านี้ = instruction ที่ทำให้ fault */
    uint32_t xpsr = stack_frame[7];

    /* อ่าน fault status registers */
    volatile uint32_t hfsr  = *(volatile uint32_t*)0xE000ED2C;  /* HardFault SR */
    volatile uint32_t cfsr  = *(volatile uint32_t*)0xE000ED28;  /* Configurable Fault SR */
    volatile uint32_t mmfar = *(volatile uint32_t*)0xE000ED34;  /* MemManage Address */
    volatile uint32_t bfar  = *(volatile uint32_t*)0xE000ED38;  /* BusFault Address */

    /* CFSR breakdown:
       bits[7:0]  = MMFSR (MemManage Fault Status)
       bits[15:8] = BFSR  (BusFault Status)
       bits[31:16]= UFSR  (UsageFault Status)
    */

    /* Infinite loop for debugging */
    while(1) { __asm__("NOP"); }
}
```

### 7.2 Fault Status Registers

```
CFSR @ 0xE000ED28:

MMFSR (bits[7:0]) — MemManage Fault:
  [7]   MMARVALID — MMFAR holds valid address
  [5]   MLSPERR   — FP lazy state preservation fault
  [4]   MSTKERR   — Stacking for exception entry fault
  [3]   MUNSTKERR — Unstacking for exception return fault
  [1]   DACCVIOL  — Data access violation
  [0]   IACCVIOL  — Instruction access violation

BFSR (bits[15:8]) — BusFault:
  [15]  BFARVALID — BFAR holds valid address
  [13]  LSPERR    — FP lazy state preservation fault
  [12]  STKERR    — Stacking for exception entry fault
  [11]  UNSTKERR  — Unstacking fault
  [10]  IMPRECISERR— Imprecise data bus error
  [9]   PRECISERR — Precise data bus error (BFAR valid)
  [8]   IBUSERR   — Instruction bus error

UFSR (bits[31:16]) — UsageFault:
  [25]  DIVBYZERO — Divide by zero (ถ้าเปิด DIV_0_TRP)
  [24]  UNALIGNED — Unaligned access (ถ้าเปิด UNALIGN_TRP)
  [19]  NOCP      — No coprocessor
  [18]  INVPC     — Invalid PC on exception return
  [17]  INVSTATE  — Invalid state (e.g., execute Thumb=0)
  [16]  UNDEFINSTR— Undefined instruction
```

---

## 8. STM32F4 Specifics

### 8.1 Memory Map

```
STM32F4 Memory Map:
0x00000000 - 0x001FFFFF  Code region (Flash or ITCM)
0x08000000 - 0x080FFFFF  Flash memory (up to 1MB on STM32F407)
0x1FFF0000 - 0x1FFF77FF  System memory (Bootloader)
0x20000000 - 0x2001FFFF  SRAM1 (112KB on STM32F407)
0x20020000 - 0x2002FFFF  SRAM2 (16KB on STM32F407)
0x40000000 - 0x400233FF  APB1 peripherals
0x40010000 - 0x40013FFF  APB2 peripherals
0x40020000 - 0x400FFFFF  AHB1 peripherals (GPIO, RCC, DMA, etc.)
0x50000000 - 0x50060BFF  AHB2 peripherals (USB OTG FS, etc.)
0xE0000000 - 0xFFFFFFFF  Private Peripheral Bus (PPB)
  0xE0000000 - 0xE003FFFF  ITM
  0xE0040000 - 0xE007FFFF  DWT
  0xE000E000 - 0xE000EFFF  SCS (System Control Space)
    0xE000E010              SysTick
    0xE000E100              NVIC
    0xE000ED00              SCB
```

### 8.2 RCC — Reset and Clock Control

```c
/* RCC Base Address */
#define RCC_BASE    0x40023800UL

/* RCC Registers */
#define RCC_CR      (*(volatile uint32_t*)(RCC_BASE + 0x00))  /* Clock control */
#define RCC_PLLCFGR (*(volatile uint32_t*)(RCC_BASE + 0x04))  /* PLL config */
#define RCC_CFGR    (*(volatile uint32_t*)(RCC_BASE + 0x08))  /* Clock config */
#define RCC_CIR     (*(volatile uint32_t*)(RCC_BASE + 0x0C))  /* Clock interrupt */
#define RCC_AHB1ENR (*(volatile uint32_t*)(RCC_BASE + 0x30))  /* AHB1 enable */
#define RCC_AHB2ENR (*(volatile uint32_t*)(RCC_BASE + 0x34))  /* AHB2 enable */
#define RCC_APB1ENR (*(volatile uint32_t*)(RCC_BASE + 0x40))  /* APB1 enable */
#define RCC_APB2ENR (*(volatile uint32_t*)(RCC_BASE + 0x44))  /* APB2 enable */

/* Enable GPIOA clock */
RCC_AHB1ENR |= (1 << 0);   /* GPIOAEN bit */

/* Enable GPIOB clock */
RCC_AHB1ENR |= (1 << 1);   /* GPIOBEN bit */

/* Enable TIM2 clock (APB1) */
RCC_APB1ENR |= (1 << 0);   /* TIM2EN bit */

/* Enable USART1 clock (APB2) */
RCC_APB2ENR |= (1 << 4);   /* USART1EN bit */

/* Enable DMA1 clock (AHB1) */
RCC_AHB1ENR |= (1 << 21);  /* DMA1EN bit */
```

### 8.3 GPIO Registers

```
GPIO Base Addresses (STM32F4):
GPIOA = 0x40020000
GPIOB = 0x40020400
GPIOC = 0x40020800
GPIOD = 0x40020C00
GPIOE = 0x40021000
GPIOF = 0x40021400
GPIOG = 0x40021800
GPIOH = 0x40021C00
GPIOI = 0x40022000

GPIO Register Offsets:
GPIOx_MODER   (0x00): Mode register (2 bits per pin)
GPIOx_OTYPER  (0x04): Output type register (1 bit per pin)
GPIOx_OSPEEDR (0x08): Output speed register (2 bits per pin)
GPIOx_PUPDR   (0x0C): Pull-up/pull-down register (2 bits per pin)
GPIOx_IDR     (0x10): Input data register (16 bits, read-only)
GPIOx_ODR     (0x14): Output data register (16 bits)
GPIOx_BSRR    (0x18): Bit set/reset register (32 bits)
GPIOx_LCKR    (0x1C): Lock register
GPIOx_AFRL    (0x20): Alternate function low register (pins 0-7)
GPIOx_AFRH    (0x24): Alternate function high register (pins 8-15)

MODER values per pin (2 bits):
  00 = Input mode
  01 = General purpose output mode
  10 = Alternate function mode
  11 = Analog mode

OTYPER values per pin (1 bit):
  0 = Push-pull
  1 = Open-drain

OSPEEDR values per pin (2 bits):
  00 = Low speed (2 MHz)
  01 = Medium speed (25 MHz)
  10 = High speed (50 MHz)
  11 = Very high speed (100 MHz)

PUPDR values per pin (2 bits):
  00 = No pull-up/pull-down
  01 = Pull-up
  10 = Pull-down
  11 = Reserved
```

---

## 9. LED Blink Example — ตัวอย่างสมบูรณ์

### 9.1 LED บน STM32F4 Discovery (PD12 = Green LED)

```asm
/* led_blink.s — blink LED on PD12 using pure assembly */
    .syntax unified
    .cpu cortex-m4
    .thumb

    .section .text
    .global main
    .type main, %function

main:
    /* Step 1: Enable GPIOD clock (RCC_AHB1ENR bit 3) */
    LDR  R0, =0x40023830         @ RCC_AHB1ENR
    LDR  R1, [R0]
    ORR  R1, R1, #(1 << 3)      @ GPIODEN bit
    STR  R1, [R0]

    /* Step 2: Set PD12 as output (MODER[25:24] = 01) */
    LDR  R0, =0x40020C00         @ GPIOD base (MODER offset 0x00)
    LDR  R1, [R0]
    BIC  R1, R1, #(3 << 24)     @ Clear MODER12[1:0]
    ORR  R1, R1, #(1 << 24)     @ Set MODER12 = 01 (output)
    STR  R1, [R0]

    /* Step 3: Set push-pull output (OTYPER bit 12 = 0) — ค่า default */
    /* Step 4: Set low speed (OSPEEDR[25:24] = 00) — ค่า default */
    /* Step 5: No pull-up/pull-down (PUPDR[25:24] = 00) — ค่า default */

blink_loop:
    /* Turn ON: Set PD12 via BSRR (bit 12 of lower halfword) */
    LDR  R0, =0x40020C18         @ GPIOD_BSRR
    MOV  R1, #(1 << 12)          @ BS12 bit → set PD12 high
    STR  R1, [R0]

    /* Delay */
    LDR  R2, =500000
delay_on:
    SUBS R2, R2, #1
    BNE  delay_on

    /* Turn OFF: Reset PD12 via BSRR (bit 12 of upper halfword = bit 28) */
    LDR  R0, =0x40020C18         @ GPIOD_BSRR
    MOV  R1, #(1 << 28)          @ BR12 bit → reset PD12 low
    STR  R1, [R0]

    /* Delay */
    LDR  R2, =500000
delay_off:
    SUBS R2, R2, #1
    BNE  delay_off

    B    blink_loop               @ Loop forever

    .size main, .-main
```

### 9.2 Version ที่ใช้ ODR โดยตรง

```asm
/* ใช้ XOR toggle บน ODR */
blink_loop_v2:
    /* Toggle PD12 ด้วย EOR */
    LDR  R0, =0x40020C14         @ GPIOD_ODR
    LDR  R1, [R0]
    EOR  R1, R1, #(1 << 12)     @ Toggle bit 12
    STR  R1, [R0]

    /* Software delay */
    LDR  R2, =2000000
delay_v2:
    SUBS R2, R2, #1
    BNE  delay_v2

    B    blink_loop_v2
```

### 9.3 C Version สำหรับเปรียบเทียบ

```c
#include <stdint.h>

/* Register definitions */
#define RCC_AHB1ENR   (*(volatile uint32_t*)0x40023830)
#define GPIOD_MODER   (*(volatile uint32_t*)0x40020C00)
#define GPIOD_OTYPER  (*(volatile uint32_t*)0x40020C04)
#define GPIOD_OSPEEDR (*(volatile uint32_t*)0x40020C08)
#define GPIOD_PUPDR   (*(volatile uint32_t*)0x40020C0C)
#define GPIOD_IDR     (*(volatile uint32_t*)0x40020C10)
#define GPIOD_ODR     (*(volatile uint32_t*)0x40020C14)
#define GPIOD_BSRR    (*(volatile uint32_t*)0x40020C18)

static void delay_ms(uint32_t ms)
{
    /* ใช้ SysTick หรือ software delay */
    volatile uint32_t i;
    for(i = 0; i < ms * 3360; i++);
}

int main(void)
{
    /* Enable GPIOD clock */
    RCC_AHB1ENR |= (1U << 3);

    /* Configure PD12 as output push-pull, low speed */
    GPIOD_MODER  &= ~(3U << 24);
    GPIOD_MODER  |=  (1U << 24);   /* Output mode */
    GPIOD_OTYPER &= ~(1U << 12);   /* Push-pull */
    GPIOD_OSPEEDR&= ~(3U << 24);   /* Low speed */
    GPIOD_PUPDR  &= ~(3U << 24);   /* No pull */

    while(1)
    {
        GPIOD_BSRR = (1U << 12);   /* Set PD12 */
        delay_ms(500);
        GPIOD_BSRR = (1U << 28);   /* Reset PD12 */
        delay_ms(500);
    }
}
```

---

## 10. UART / USART บน STM32F4

### 10.1 USART Registers

```
USART1 Base: 0x40011000
USART2 Base: 0x40004400
USART3 Base: 0x40004800

Register Offsets:
USART_SR   (0x00): Status Register
USART_DR   (0x04): Data Register (8 หรือ 9 bit)
USART_BRR  (0x08): Baud Rate Register
USART_CR1  (0x0C): Control Register 1
USART_CR2  (0x10): Control Register 2
USART_CR3  (0x14): Control Register 3
USART_GTPR (0x18): Guard time and prescaler

USART_SR bits:
  [7] TXE  — Transmit data register empty (ready for new data)
  [6] TC   — Transmission complete
  [5] RXNE — Read data register not empty (data available)
  [4] IDLE — Idle line detected
  [3] ORE  — Overrun error
  [2] NE   — Noise detected
  [1] FE   — Framing error
  [0] PE   — Parity error

USART_CR1 bits:
  [13] UE    — USART enable
  [12] M     — Word length (0=8bit, 1=9bit)
  [10] PCE   — Parity control enable
  [9]  PS    — Parity selection (0=even, 1=odd)
  [8]  PEIE  — PE interrupt enable
  [7]  TXEIE — TXE interrupt enable
  [6]  TCIE  — TC interrupt enable
  [5]  RXNEIE— RXNE interrupt enable
  [3]  TE    — Transmitter enable
  [2]  RE    — Receiver enable
```

### 10.2 Baud Rate Calculation

```
Baud Rate = fCK / (8 × (2 - OVER8) × USARTDIV)

สำหรับ 8x oversampling (OVER8=0):
USARTDIV = fCK / (16 × BaudRate)

ตัวอย่าง: fCK = 84MHz (USART1 on APB2), BaudRate = 115200
USARTDIV = 84,000,000 / (16 × 115,200) = 45.5729...

Integer part   = 45   → stored in USART_BRR[15:4]
Fractional part = 0.5729 × 16 = 9.16 ≈ 9 → stored in USART_BRR[3:0]

USART_BRR = (45 << 4) | 9 = 0x02D9

ตรวจสอบ:
Actual baud = 84,000,000 / (16 × 45.5625) = 115,207 bps (~0.006% error)
```

### 10.3 USART Initialization ใน Assembly

```asm
/* USART1 init: PA9=TX, PA10=RX, 115200 8N1 @ 84MHz */
    .section .text
    .global USART1_Init
    .type USART1_Init, %function
USART1_Init:
    PUSH {R4, LR}

    /* 1. Enable clocks for GPIOA and USART1 */
    LDR  R4, =0x40023800          @ RCC base
    LDR  R0, [R4, #0x30]          @ RCC_AHB1ENR
    ORR  R0, R0, #(1 << 0)        @ GPIOAEN
    STR  R0, [R4, #0x30]

    LDR  R0, [R4, #0x44]          @ RCC_APB2ENR
    ORR  R0, R0, #(1 << 4)        @ USART1EN
    STR  R0, [R4, #0x44]

    /* 2. Configure PA9 (TX) and PA10 (RX) as AF7 (USART1) */
    LDR  R4, =0x40020000          @ GPIOA base

    /* PA9, PA10: Alternate function mode (MODER = 10) */
    LDR  R0, [R4, #0x00]          @ GPIOA_MODER
    BIC  R0, R0, #(0xF << 18)     @ Clear MODER9[1:0] and MODER10[1:0]
    ORR  R0, R0, #(0xA << 18)     @ Set AF mode for both
    STR  R0, [R4, #0x00]

    /* Set pull-up on RX (PA10) */
    LDR  R0, [R4, #0x0C]          @ GPIOA_PUPDR
    ORR  R0, R0, #(1 << 20)       @ Pull-up on PA10
    STR  R0, [R4, #0x0C]

    /* Set AF7 in AFRH (pins 8-15, using AFRH offset 0x24) */
    /* PA9 → AFRH[7:4], PA10 → AFRH[11:8] */
    LDR  R0, [R4, #0x24]          @ GPIOA_AFRH
    BIC  R0, R0, #(0xFF << 4)     @ Clear AF for PA9 and PA10
    ORR  R0, R0, #(0x77 << 4)     @ AF7 for PA9(bits7:4) and PA10(bits11:8)
    STR  R0, [R4, #0x24]

    /* 3. Configure USART1 */
    LDR  R4, =0x40011000          @ USART1 base

    /* BRR = 0x02D9 for 115200 @ 84MHz */
    LDR  R0, =0x02D9
    STR  R0, [R4, #0x08]          @ USART_BRR

    /* CR1: Enable TE + RE + UE (bits 3, 2, 13) */
    LDR  R0, =((1<<13)|(1<<3)|(1<<2))
    STR  R0, [R4, #0x0C]          @ USART_CR1

    POP  {R4, PC}
    .size USART1_Init, .-USART1_Init


    .global USART1_SendChar
    .type USART1_SendChar, %function
USART1_SendChar:
    /* R0 = character to send */
    LDR  R1, =0x40011000          @ USART1 base
wait_txe:
    LDR  R2, [R1, #0x00]          @ USART_SR
    TST  R2, #(1 << 7)            @ TXE bit
    BEQ  wait_txe                  @ Wait if TXE=0
    STR  R0, [R1, #0x04]          @ Write to USART_DR
    BX   LR
    .size USART1_SendChar, .-USART1_SendChar


    .global USART1_SendString
    .type USART1_SendString, %function
USART1_SendString:
    /* R0 = pointer to null-terminated string */
    PUSH {R4, LR}
    MOV  R4, R0
send_next:
    LDRB R0, [R4], #1             @ Load byte, post-increment
    CBZ  R0, send_done             @ If null, done
    BL   USART1_SendChar
    B    send_next
send_done:
    POP  {R4, PC}
    .size USART1_SendString, .-USART1_SendString
```

---

## 11. DMA Controller

### 11.1 DMA Architecture บน STM32F4

```
STM32F4 มี DMA 2 ตัว:
- DMA1: 8 streams, แต่ละ stream มี 8 channels (0-7)
- DMA2: 8 streams, แต่ละ stream มี 8 channels (0-7)

DMA1 Base: 0x40026000
DMA2 Base: 0x40026400

Register Map (per stream, offset from DMA base):
Stream 0: +0x010
Stream 1: +0x028
Stream 2: +0x040
Stream 3: +0x058
Stream 4: +0x070
Stream 5: +0x088
Stream 6: +0x0A0
Stream 7: +0x0B8

Per-stream registers (offset from stream base):
+0x00: DMA_SxCR   — Configuration Register
+0x04: DMA_SxNDTR — Number of Data items Register
+0x08: DMA_SxPAR  — Peripheral Address Register
+0x0C: DMA_SxM0AR — Memory 0 Address Register
+0x10: DMA_SxM1AR — Memory 1 Address Register (double buffer)
+0x14: DMA_SxFCR  — FIFO Control Register
```

### 11.2 DMA Transfer — Memory to Peripheral (UART TX)

```c
/* DMA2 Stream7 Ch4 → USART1_TX */
#define DMA2_BASE       0x40026400UL
#define DMA2_S7_BASE    (DMA2_BASE + 0x0B8)

#define DMA_SxCR        (*(volatile uint32_t*)(DMA2_S7_BASE + 0x00))
#define DMA_SxNDTR      (*(volatile uint32_t*)(DMA2_S7_BASE + 0x04))
#define DMA_SxPAR       (*(volatile uint32_t*)(DMA2_S7_BASE + 0x08))
#define DMA_SxM0AR      (*(volatile uint32_t*)(DMA2_S7_BASE + 0x0C))
#define DMA_SxFCR       (*(volatile uint32_t*)(DMA2_S7_BASE + 0x14))

#define USART1_DR_ADDR  0x40011004UL

void DMA_UART_TX_Setup(uint8_t *buffer, uint16_t length)
{
    /* Disable stream */
    DMA_SxCR &= ~(1U << 0);
    while(DMA_SxCR & 1);  /* Wait until disabled */

    /* Peripheral address = USART1_DR */
    DMA_SxPAR = USART1_DR_ADDR;

    /* Memory address = our buffer */
    DMA_SxM0AR = (uint32_t)buffer;

    /* Number of data items */
    DMA_SxNDTR = length;

    /* CR configuration:
       Channel 4 (bits[27:25] = 100)
       Memory data size 8-bit (bits[14:13] = 00)
       Peripheral data size 8-bit (bits[12:11] = 00)
       Memory increment mode (bit[10] = 1)
       Peripheral fixed (bit[9] = 0)
       Direction: memory-to-peripheral (bits[7:6] = 01)
       Transfer complete interrupt enable (bit[4] = 1)
       Enable stream (bit[0] = 1)
    */
    DMA_SxCR = (4U << 25) |  /* Channel 4 */
               (1U << 10) |  /* MINC */
               (1U << 6)  |  /* DIR: Mem→Periph */
               (1U << 4)  |  /* TCIE */
               (1U << 0);    /* EN */

    /* Enable USART DMA transmit */
    *(volatile uint32_t*)0x40011014 |= (1U << 7);  /* USART_CR3 DMAT */
}
```

### 11.3 DMA Interrupt Handler

```c
void DMA2_Stream7_IRQHandler(void)
{
    /* Check Transfer Complete flag (HISR bit 27 for stream 7) */
    if(*(volatile uint32_t*)(DMA2_BASE + 0x04) & (1U << 27))
    {
        /* Clear flag (write to HIFCR) */
        *(volatile uint32_t*)(DMA2_BASE + 0x0C) = (1U << 27);

        /* Mark transfer as done */
        dma_transfer_complete = 1;
    }
}
```

---

## 12. Context Switch — Assembly Implementation

### 12.1 RTOS Context Switch Concept

Context switch คือการ save state ของ task ปัจจุบัน แล้ว restore state ของ task ถัดไป:

```
Hardware auto-saves (on exception entry):  R0-R3, R12, LR, PC, xPSR
Software must save:                        R4-R11 (callee-saved + not auto-saved)
For FPU context (if used):                 S0-S15 (auto), S16-S31 (manual)
```

### 12.2 Task Control Block (TCB)

```c
/* Task Control Block structure */
typedef struct {
    volatile uint32_t *stack_ptr;   /* Offset 0: must be first! */
    void              *stack_base;
    uint32_t           stack_size;
    uint8_t            priority;
    uint8_t            state;
    const char        *name;
} TCB_t;

TCB_t *current_tcb;   /* Pointer to currently running task's TCB */
TCB_t *next_tcb;      /* Pointer to next task's TCB */
```

### 12.3 PendSV Handler — Context Switch Assembly

PendSV มี priority ต่ำที่สุด (set ไว้ที่ 0xFF) เพื่อให้ context switch เกิดหลัง interrupts อื่นทั้งหมด:

```asm
/* pendsv_handler.s — FreeRTOS-style context switch for Cortex-M4 */
    .syntax unified
    .cpu cortex-m4
    .thumb

    .extern current_tcb
    .extern next_tcb

    .section .text.PendSV_Handler
    .global PendSV_Handler
    .type PendSV_Handler, %function
    .align 4

PendSV_Handler:
    /* ===== SAVE CURRENT TASK CONTEXT ===== */

    /* Disable interrupts during context switch */
    CPSID   I

    /* Get current PSP (task's stack pointer) */
    MRS     R0, PSP
    ISB

    /* ถ้ามี FPU และ task ใช้ FP, ต้อง save S16-S31 ด้วย */
    /* ตรวจสอบ CONTROL[2] (FPCA) */
#if defined(__FPU_USED) && (__FPU_USED == 1U)
    TST     LR, #0x10              @ Test bit 4 of EXC_RETURN
    IT      EQ
    VSTMDBEQ R0!, {S16-S31}        @ Save S16-S31 (only if FPU active)
#endif

    /* Save R4-R11 บน PSP stack */
    STMDB   R0!, {R4-R11, LR}     @ Push R4-R11 + EXC_RETURN (LR)

    /* Save new PSP ลง TCB ของ current task */
    LDR     R3, =current_tcb
    LDR     R1, [R3]               @ R1 = pointer to current TCB
    STR     R0, [R1]               @ TCB->stack_ptr = PSP

    /* ===== SWITCH TO NEXT TASK ===== */

    /* Load next_tcb → current_tcb */
    LDR     R2, =next_tcb
    LDR     R2, [R2]               @ R2 = pointer to next TCB
    STR     R2, [R3]               @ current_tcb = next_tcb

    /* Load PSP ของ next task จาก TCB */
    LDR     R0, [R2]               @ R0 = next TCB's stack_ptr

    /* ===== RESTORE NEXT TASK CONTEXT ===== */

    /* Restore R4-R11 + LR (EXC_RETURN) จาก task stack */
    LDMIA   R0!, {R4-R11, LR}     @ Pop R4-R11 + EXC_RETURN

#if defined(__FPU_USED) && (__FPU_USED == 1U)
    TST     LR, #0x10              @ Test bit 4 of EXC_RETURN
    IT      EQ
    VLDMIAEQ R0!, {S16-S31}        @ Restore S16-S31 (if FPU active)
#endif

    /* Set PSP to restored value */
    MSR     PSP, R0
    ISB

    /* Enable interrupts */
    CPSIE   I

    /* Return from exception:
       hardware will pop R0-R3, R12, LR, PC, xPSR from PSP */
    BX      LR

    .size PendSV_Handler, .-PendSV_Handler
```

### 12.4 Task Stack Initialization

ก่อนจะ start task ครั้งแรก ต้องตั้ง stack ให้ดูเหมือนว่า task เพิ่ง suspend ออกมาจาก PendSV:

```c
/* Initialize task stack for first launch */
uint32_t *task_stack_init(uint32_t *stack_top, TaskFunction_t task_func, void *param)
{
    /* Stack grows downward — ลดจาก stack_top */

    /* Fake exception frame (auto-saved by hardware on real exception) */
    *(--stack_top) = 0x01000000UL;          /* xPSR: Thumb bit set */
    *(--stack_top) = (uint32_t)task_func;   /* PC: task entry point */
    *(--stack_top) = (uint32_t)0xFFFFFFFDUL;/* LR: EXC_RETURN (PSP, no FPU) */
    *(--stack_top) = 0x00000000UL;          /* R12 */
    *(--stack_top) = 0x00000000UL;          /* R3 */
    *(--stack_top) = 0x00000000UL;          /* R2 */
    *(--stack_top) = 0x00000000UL;          /* R1 */
    *(--stack_top) = (uint32_t)param;       /* R0: first argument */

    /* Fake software-saved frame (saved by PendSV_Handler) */
    *(--stack_top) = 0xFFFFFFFDUL;          /* EXC_RETURN (stored as LR in STMDB) */
    *(--stack_top) = 0x00000000UL;          /* R11 */
    *(--stack_top) = 0x00000000UL;          /* R10 */
    *(--stack_top) = 0x00000000UL;          /* R9 */
    *(--stack_top) = 0x00000000UL;          /* R8 */
    *(--stack_top) = 0x00000000UL;          /* R7 */
    *(--stack_top) = 0x00000000UL;          /* R6 */
    *(--stack_top) = 0x00000000UL;          /* R5 */
    *(--stack_top) = 0x00000000UL;          /* R4 */

    return stack_top;  /* This is the initial PSP for this task */
}

/* Stack frame layout after initialization:
   (high address = stack_top passed in)
   xPSR     ← original stack_top - 1
   PC       ← original stack_top - 2
   LR_exc   ← original stack_top - 3
   R12      ← original stack_top - 4
   R3       ← original stack_top - 5
   R2       ← original stack_top - 6
   R1       ← original stack_top - 7
   R0/param ← original stack_top - 8
   EXC_RET  ← original stack_top - 9  (fake LR from STMDB)
   R11      ← original stack_top - 10
   ...
   R4       ← original stack_top - 17 ← returned as initial PSP
*/
```

### 12.5 SVC Handler สำหรับ Start First Task

```asm
/* SVC 0 = start scheduler (first context switch to first task) */
    .section .text.SVC_Handler
    .global SVC_Handler
    .type SVC_Handler, %function
SVC_Handler:
    /* Check SVC number */
    TST     LR, #4
    ITE     EQ
    MRSEQ   R0, MSP
    MRSNE   R0, PSP

    /* Get SVC number from instruction opcode */
    LDR     R0, [R0, #24]          @ PC from stack frame
    LDRB    R0, [R0, #-2]          @ SVC number (byte before PC)

    CMP     R0, #0
    BEQ     svc_start_first_task

    /* Other SVC numbers... */
    BX      LR

svc_start_first_task:
    /* Set PendSV to lowest priority */
    LDR     R0, =0xE000ED22        @ SCB_SHPR3 + offset for PendSV
    LDR     R1, [R0]
    ORR     R1, R1, #(0xFF << 16)  @ PendSV priority = 0xFF (lowest)
    STR     R1, [R0]

    /* Set SysTick to second lowest */
    LDR     R0, =0xE000ED23
    MOV     R1, #0xFE
    STRB    R1, [R0]

    /* Load first task's PSP */
    LDR     R2, =current_tcb
    LDR     R2, [R2]               @ current TCB pointer
    LDR     R0, [R2]               @ stack_ptr

    /* Restore R4-R11 + EXC_RETURN */
    LDMIA   R0!, {R4-R11, LR}

    /* Set PSP */
    MSR     PSP, R0
    ISB

    /* Switch to unprivileged Thread mode using PSP */
    MOV     R0, #0x03              @ SPSEL=1, nPRIV=1
    MSR     CONTROL, R0
    ISB

    /* Enable interrupts */
    CPSIE   I
    CPSIE   F

    /* Return to task — hardware pops R0-R3, R12, LR, PC, xPSR */
    BX      LR

    .size SVC_Handler, .-SVC_Handler
```

---

## 13. FreeRTOS Assembly Port (Cortex-M4)

### 13.1 port.c Assembly Functions

FreeRTOS port สำหรับ Cortex-M4 มี assembly functions หลัก:

```asm
/* FreeRTOS portasm.s — ARM Cortex-M4 port */
    .syntax unified
    .cpu cortex-m4
    .thumb

/* Offsets used in TCB (from FreeRTOS port header) */
#define portTASK_FUNCTION_PROTO 0

    .extern pxCurrentTCB
    .extern vTaskSwitchContext

/*
 * vPortStartFirstTask — เรียกครั้งแรกเพื่อ start scheduler
 * ใช้ SVC 0
 */
    .section .text.vPortStartFirstTask
    .global vPortStartFirstTask
    .type vPortStartFirstTask, %function
vPortStartFirstTask:
    /* VTOR register มี address ของ vector table */
    LDR  R0, =0xE000ED08           @ SCB_VTOR
    LDR  R0, [R0]
    LDR  R0, [R0]                  @ First entry = initial MSP
    MSR  MSP, R0                   @ Reset MSP to initial value

    /* Enable interrupts */
    CPSIE I
    CPSIE F
    DSB
    ISB

    SVC  0                         @ SVC #0 → start first task
    .size vPortStartFirstTask, .-vPortStartFirstTask


/*
 * xPortPendSVHandler — actual context switch
 */
    .section .text.xPortPendSVHandler
    .global xPortPendSVHandler
    .global PendSV_Handler
    .type xPortPendSVHandler, %function
xPortPendSVHandler:
PendSV_Handler:
    MRS     R0, PSP
    ISB

    /* ถ้า configUSE_TASK_FPU_SUPPORT = 1 */
    LDR     R3, =pxCurrentTCB
    LDR     R2, [R3]

#if ( configUSE_TASK_FPU_SUPPORT == 1 )
    TST     LR, #0x10
    IT      EQ
    VSTMDBEQ R0!, {S16-S31}

    /* Save EXC_RETURN in TCB */
    STR     LR, [R2, #4]           @ pxCurrentTCB->pxTopOfStack + 4 = LR
#endif

    STMDB   R0!, {R4-R11}
    STR     R0, [R2]               @ pxCurrentTCB->pxTopOfStack = PSP

    STMDB   SP!, {R0, R3}
    MOV     R0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
    MSR     BASEPRI, R0
    DSB
    ISB
    BL      vTaskSwitchContext
    MOV     R0, #0
    MSR     BASEPRI, R0
    LDMIA   SP!, {R0, R3}

    LDR     R1, [R3]               @ R1 = pxCurrentTCB (after switch)
    LDR     R0, [R1]               @ R0 = new PSP (pxTopOfStack)

    LDMIA   R0!, {R4-R11}

#if ( configUSE_TASK_FPU_SUPPORT == 1 )
    LDR     LR, [R1, #4]           @ Restore EXC_RETURN
    TST     LR, #0x10
    IT      EQ
    VLDMIAEQ R0!, {S16-S31}
#endif

    MSR     PSP, R0
    ISB
    BX      LR
    .size xPortPendSVHandler, .-xPortPendSVHandler


/*
 * vPortSVCHandler — handles SVC calls from FreeRTOS
 */
    .section .text.vPortSVCHandler
    .global vPortSVCHandler
    .global SVC_Handler
    .type vPortSVCHandler, %function
vPortSVCHandler:
SVC_Handler:
    LDR     R3, =pxCurrentTCB
    LDR     R1, [R3]
    LDR     R0, [R1]               @ pxCurrentTCB->pxTopOfStack

    LDMIA   R0!, {R4-R11, R14}

#if ( configUSE_TASK_FPU_SUPPORT == 1 )
    TST     R14, #0x10
    IT      EQ
    VLDMIAEQ R0!, {S16-S31}
#endif

    MSR     PSP, R0
    ISB

    MOV     R0, #0
    MSR     BASEPRI, R0

    ORR     R14, #0x0D             @ EXC_RETURN: Thread, PSP, no FPU
    BX      R14
    .size vPortSVCHandler, .-vPortSVCHandler
```

### 13.2 FreeRTOS Tick Interrupt

```asm
/*
 * xPortSysTickHandler — FreeRTOS SysTick handler
 */
    .section .text.xPortSysTickHandler
    .global xPortSysTickHandler
    .global SysTick_Handler
    .type xPortSysTickHandler, %function
xPortSysTickHandler:
SysTick_Handler:
    /* Mask interrupts up to configMAX_SYSCALL_INTERRUPT_PRIORITY */
    MOV     R0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
    MSR     BASEPRI, R0
    DSB
    ISB

    /* Call FreeRTOS tick handler */
    BL      xTaskIncrementTick

    /* If context switch required (return != 0), trigger PendSV */
    CBZ     R0, no_switch
    LDR     R1, =0xE000ED04       @ ICSR
    LDR     R0, =0x10000000       @ PENDSVSET
    STR     R0, [R1]

no_switch:
    /* Unmask interrupts */
    MOV     R0, #0
    MSR     BASEPRI, R0

    BX      LR
    .size xPortSysTickHandler, .-xPortSysTickHandler
```

---

## 14. Linker Script สำหรับ Cortex-M4

### 14.1 STM32F407 Linker Script

```ld
/* STM32F407VGTx.ld — Linker script for STM32F407 */

/* Entry point */
ENTRY(Reset_Handler)

/* Memory regions */
MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
  RAM   (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
  CCMRAM(xrw) : ORIGIN = 0x10000000, LENGTH = 64K
}

/* Stack and heap sizes */
_Min_Heap_Size  = 0x200;
_Min_Stack_Size = 0x400;

SECTIONS
{
  /* Vector table and code in FLASH */
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))
    . = ALIGN(4);
  } >FLASH

  .text :
  {
    . = ALIGN(4);
    *(.text)
    *(.text*)
    *(.glue_7)
    *(.glue_7t)
    *(.eh_frame)
    KEEP(*(.init))
    KEEP(*(.fini))
    . = ALIGN(4);
    _etext = .;
  } >FLASH

  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)
    *(.rodata*)
    . = ALIGN(4);
  } >FLASH

  /* ARM exception handling tables */
  .ARM.extab :
  {
    *(.ARM.extab* .gnu.linkonce.armextab.*)
  } >FLASH

  .ARM :
  {
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
  } >FLASH

  /* Initialized data — load from FLASH, run in RAM */
  _sidata = LOADADDR(.data);

  .data :
  {
    . = ALIGN(4);
    _sdata = .;
    *(.data)
    *(.data*)
    . = ALIGN(4);
    _edata = .;
  } >RAM AT> FLASH

  /* CCM RAM — fast access, cannot run code with DMA */
  .ccmram :
  {
    . = ALIGN(4);
    _sccmram = .;
    *(.ccmram)
    *(.ccmram*)
    . = ALIGN(4);
    _eccmram = .;
  } >CCMRAM AT> FLASH

  /* BSS — zero-initialized data in RAM */
  .bss :
  {
    . = ALIGN(4);
    _sbss = .;
    __bss_start__ = _sbss;
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
    __bss_end__ = _ebss;
  } >RAM

  /* User heap */
  ._user_heap_stack :
  {
    . = ALIGN(8);
    PROVIDE(end = .);
    PROVIDE(_end = .);
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >RAM

  /* Stack top = end of RAM */
  _estack = ORIGIN(RAM) + LENGTH(RAM);
}
```

---

## 15. Cortex-M4 DSP Instructions

### 15.1 SIMD Instructions (ARMv7E-M)

```asm
/* SIMD Packed operations — 2x16-bit or 4x8-bit operations in one instruction */

/* SADD16: Signed parallel Add 16 */
/* R0 = {R1[31:16]+R2[31:16], R1[15:0]+R2[15:0]} */
SADD16  R0, R1, R2

/* SSUB16: Signed parallel Subtract 16 */
SSUB16  R0, R1, R2

/* SADD8: Signed parallel Add 8 (4 byte operations) */
SADD8   R0, R1, R2

/* QADD16: Saturating parallel Add */
QADD16  R0, R1, R2

/* SMAD BB/BT/TB/TT: Signed Multiply-Accumulate (16-bit × 16-bit + 32-bit) */
/* B = bottom half (bits[15:0]), T = top half (bits[31:16]) */
SMLABB  R0, R1, R2, R3   @ R0 = R1[15:0] × R2[15:0] + R3
SMLABT  R0, R1, R2, R3   @ R0 = R1[15:0] × R2[31:16] + R3
SMLATB  R0, R1, R2, R3   @ R0 = R1[31:16] × R2[15:0] + R3
SMLATT  R0, R1, R2, R3   @ R0 = R1[31:16] × R2[31:16] + R3

/* SMLAD: Dual 16-bit Multiply-Accumulate */
/* R0 = R1[15:0]×R2[15:0] + R1[31:16]×R2[31:16] + R3 */
SMLAD   R0, R1, R2, R3

/* Useful for FIR filter: */
/* output += coeff[i] × sample[i] */
```

### 15.2 FPU Instructions (VFP4)

```asm
/* FPU single-precision operations */
VADD.F32  S0, S1, S2    @ S0 = S1 + S2
VSUB.F32  S0, S1, S2    @ S0 = S1 - S2
VMUL.F32  S0, S1, S2    @ S0 = S1 × S2
VDIV.F32  S0, S1, S2    @ S0 = S1 / S2
VSQRT.F32 S0, S1        @ S0 = sqrt(S1)
VMLA.F32  S0, S1, S2    @ S0 += S1 × S2 (multiply-accumulate)
VMLS.F32  S0, S1, S2    @ S0 -= S1 × S2 (multiply-subtract)
VNEG.F32  S0, S1        @ S0 = -S1
VABS.F32  S0, S1        @ S0 = |S1|

/* Load/Store */
VLDR    S0, [R0]        @ S0 = *(float*)R0
VSTR    S0, [R0]        @ *(float*)R0 = S0
VLDM    R0, {S0-S7}     @ Load 8 floats from R0
VSTM    R0, {S0-S7}     @ Store 8 floats to R0

/* Move between ARM and FP registers */
VMOV    S0, R0          @ S0 = R0 (integer bits copied)
VMOV    R0, S0          @ R0 = S0
VCVT.F32.S32 S0, S0    @ Convert signed int → float
VCVT.S32.F32 S0, S0    @ Convert float → signed int (truncate)
VCVTR.S32.F32 S0, S0   @ Convert float → signed int (round)
```

---

## 16. MPU — Memory Protection Unit

### 16.1 MPU Overview

```
MPU ให้ความสามารถ:
- แบ่งหน่วยความจำออกเป็น regions (สูงสุด 8 หรือ 16 regions)
- กำหนด access permissions (Read-only, Read-Write, No-access)
- กำหนด memory attributes (Cacheable, Bufferable, Shareable)
- ป้องกัน task จาก access memory ของ task อื่น (ใน RTOS)
- ป้องกัน task จาก corrupt kernel data
```

### 16.2 MPU Registers

```
MPU Base: 0xE000ED90

MPU_TYPE    (0xE000ED90): Type Register (read-only)
MPU_CTRL    (0xE000ED94): Control Register
MPU_RNR     (0xE000ED98): Region Number Register
MPU_RBAR    (0xE000ED9C): Region Base Address Register
MPU_RASR    (0xE000EDA0): Region Attribute and Size Register
MPU_RBAR_A1 (0xE000EDA4): Alias 1 of RBAR
MPU_RASR_A1 (0xE000EDA8): Alias 1 of RASR
/* Aliases A2, A3 at 0xEDAC-0xEDB8 */
```

### 16.3 MPU Configuration

```c
/* Configure MPU region for Flash (read-only, executable) */
void MPU_Config(void)
{
    /* Disable MPU */
    MPU->CTRL = 0;

    /* Region 0: Flash — read-only, executable */
    MPU->RNR = 0;
    MPU->RBAR = 0x08000000;             /* Base address */
    MPU->RASR =
        (0x1F << 1)    |   /* SIZE[5:1] = 0x1F → 2^(31+1) = 4GB? No. 0x12 = 512KB */
        /* SIZE = 0x12 (18) → 2^(18+1) = 512KB */
        (0 << 28)      |   /* XN = 0 (executable) */
        (0x06 << 24)   |   /* AP = 110 (privileged R/W, unprivileged R-only) */
        (0 << 19)      |   /* TEX = 000 */
        (1 << 17)      |   /* C = 1 (cacheable) */
        (1 << 16)      |   /* B = 1 (bufferable) */
        (1 << 0);          /* ENABLE */

    /* Region 1: SRAM — read-write, not executable */
    MPU->RNR = 1;
    MPU->RBAR = 0x20000000;
    MPU->RASR =
        (0x11 << 1)    |   /* SIZE = 0x11 (17) → 2^18 = 256KB */
        (1 << 28)      |   /* XN = 1 (never execute) */
        (0x03 << 24)   |   /* AP = 011 (full access) */
        (1 << 0);

    /* Enable MPU with default memory map for privileged software */
    MPU->CTRL = (1 << 2) | /* PRIVDEFENA: privileged default map */
                (1 << 0);  /* ENABLE */

    /* Ensure MPU is enabled before next instruction */
    __DSB();
    __ISB();
}
```

---

## 17. Bit-Band Operations

### 17.1 Bit-Band เฉพาะ Cortex-M3/M4

```
Bit-Band Regions:
SRAM Bit-Band:     0x20000000 - 0x200FFFFF (1MB)
SRAM Alias:        0x22000000 - 0x23FFFFFF (32MB)

Peripheral Bit-Band: 0x40000000 - 0x400FFFFF (1MB)
Peripheral Alias:    0x42000000 - 0x43FFFFFF (32MB)

Formula:
bit_word_addr = bit_band_base + (byte_offset × 32) + (bit_number × 4)

ตัวอย่าง: bit 3 ของ byte ที่ 0x20000200
bit_word_addr = 0x22000000 + (0x200 × 32) + (3 × 4)
             = 0x22000000 + 0x4000 + 0x0C
             = 0x2200400C
```

### 17.2 Bit-Band Access ใน C

```c
/* Macro สำหรับ bit-band access */
#define BITBAND_SRAM(addr, bit)  \
    ((volatile uint32_t*)(0x22000000 + ((uint32_t)(addr) - 0x20000000)*32 + (bit)*4))

#define BITBAND_PERIPH(addr, bit) \
    ((volatile uint32_t*)(0x42000000 + ((uint32_t)(addr) - 0x40000000)*32 + (bit)*4))

/* ตัวอย่าง: Toggle GPIO bit แบบ atomic */
/* GPIOD_ODR[12] ผ่าน bit-band */
volatile uint32_t *pd12_bit = BITBAND_PERIPH(0x40020C14, 12);

/* Atomic read-modify-write — no interrupt can occur between */
*pd12_bit = 1;    /* Set PD12 */
*pd12_bit = 0;    /* Clear PD12 */
*pd12_bit ^= 1;   /* Toggle PD12 (แต่ XOR ไม่ atomic ใน bit-band) */
```

---

## 18. Clock System (STM32F4)

### 18.1 Clock Tree

```
STM32F4 Clock Sources:
1. HSI (High Speed Internal) — 16MHz RC oscillator
2. HSE (High Speed External) — 4-26MHz crystal/oscillator
3. PLL — Phase-Locked Loop (หลัก)
4. LSI (Low Speed Internal) — 32kHz (สำหรับ IWDG, RTC)
5. LSE (Low Speed External) — 32.768kHz (สำหรับ RTC)

PLL Configuration สำหรับ 168MHz (max STM32F407):
HSE = 8MHz
PLLM = 8 (HSE/PLLM = 1MHz → VCO input)
PLLN = 336 (VCO = 1 × 336 = 336MHz)
PLLP = 2 (SYSCLK = 336/2 = 168MHz)
PLLQ = 7 (USB = 336/7 = 48MHz)

Bus dividers:
AHB  (HCLK)  = SYSCLK / 1 = 168MHz
APB1 (PCLK1) = HCLK / 4   = 42MHz  (TIMx × 2 = 84MHz)
APB2 (PCLK2) = HCLK / 2   = 84MHz  (TIMx × 2 = 168MHz)
```

### 18.2 SystemInit Assembly

```asm
    .section .text.SystemInit
    .global SystemInit
    .type SystemInit, %function
SystemInit:
    PUSH {LR}

    /* Step 1: Reset RCC configuration */
    LDR  R0, =0x40023800            @ RCC base

    /* Enable HSI (bit 0 of CR) */
    LDR  R1, [R0, #0x00]            @ RCC_CR
    ORR  R1, R1, #1                 @ HSION
    STR  R1, [R0, #0x00]

wait_hsi_rdy:
    LDR  R1, [R0, #0x00]            @ RCC_CR
    TST  R1, #2                     @ HSIRDY bit
    BEQ  wait_hsi_rdy

    /* Select HSI as system clock */
    LDR  R1, [R0, #0x08]            @ RCC_CFGR
    BIC  R1, R1, #3                 @ Clear SW[1:0]
    STR  R1, [R0, #0x08]            @ SW = 00 (HSI)

wait_hsi_sw:
    LDR  R1, [R0, #0x08]
    AND  R1, R1, #0x0C              @ SWS[1:0] in bits[3:2]
    CMP  R1, #0
    BNE  wait_hsi_sw

    /* Step 2: Configure Flash latency for 168MHz (5 wait states) */
    LDR  R1, =0x40023C00            @ FLASH_ACR
    LDR  R2, [R1]
    BIC  R2, R2, #0x07              @ Clear LATENCY
    ORR  R2, R2, #0x05              @ 5 wait states
    ORR  R2, R2, #(1 << 8)         @ PRFTEN (prefetch enable)
    ORR  R2, R2, #(1 << 9)         @ ICEN (instruction cache)
    ORR  R2, R2, #(1 << 10)        @ DCEN (data cache)
    STR  R2, [R1]

    /* Step 3: Enable HSE */
    LDR  R1, [R0, #0x00]            @ RCC_CR
    ORR  R1, R1, #(1 << 16)        @ HSEON
    STR  R1, [R0, #0x00]

wait_hse_rdy:
    LDR  R1, [R0, #0x00]
    TST  R1, #(1 << 17)            @ HSERDY
    BEQ  wait_hse_rdy

    /* Step 4: Configure PLL */
    /* PLLCFGR: PLLM=8, PLLN=336, PLLP=2, PLLSRC=HSE, PLLQ=7 */
    LDR  R1, =((8 << 0) | (336 << 6) | (0 << 16) | (1 << 22) | (7 << 24))
    STR  R1, [R0, #0x04]            @ RCC_PLLCFGR

    /* Enable PLL */
    LDR  R1, [R0, #0x00]
    ORR  R1, R1, #(1 << 24)        @ PLLON
    STR  R1, [R0, #0x00]

wait_pll_rdy:
    LDR  R1, [R0, #0x00]
    TST  R1, #(1 << 25)            @ PLLRDY
    BEQ  wait_pll_rdy

    /* Step 5: Configure bus dividers */
    /* HPRE=0 (AHB=SYSCLK), PPRE1=5 (APB1=HCLK/4), PPRE2=4 (APB2=HCLK/2) */
    LDR  R1, [R0, #0x08]
    BIC  R1, R1, #(0xFF << 4)      @ Clear HPRE, PPRE1, PPRE2
    ORR  R1, R1, #(0x05 << 10)    @ PPRE1 = 101 (÷4)
    ORR  R1, R1, #(0x04 << 13)    @ PPRE2 = 100 (÷2)
    STR  R1, [R0, #0x08]

    /* Step 6: Switch to PLL */
    LDR  R1, [R0, #0x08]
    ORR  R1, R1, #2                 @ SW = 10 (PLL)
    STR  R1, [R0, #0x08]

wait_pll_sw:
    LDR  R1, [R0, #0x08]
    AND  R1, R1, #0x0C
    CMP  R1, #8                     @ SWS = 10 (PLL selected)
    BNE  wait_pll_sw

    POP  {PC}
    .size SystemInit, .-SystemInit
```

---

## 19. Interrupt Tail-Chaining และ Late-Arrival

### 19.1 Tail-Chaining

Cortex-M optimization: ถ้ามี interrupt รอระหว่างที่กำลัง return จาก exception handler หนึ่ง จะไม่ทำ unstacking + stacking ใหม่ แต่ jump ตรงไปยัง handler ถัดไปเลย

```
ปกติ (ไม่มี tail-chain):
  Task → [Stacking: 12 cycles] → IRQ1 handler
  → [Unstacking: 10 cycles] → Task
  → [Stacking: 12 cycles] → IRQ2 handler
  → [Unstacking: 10 cycles] → Task

Tail-chaining:
  Task → [Stacking: 12 cycles] → IRQ1 handler
  → [Tail-chain: 6 cycles] → IRQ2 handler (ไม่ unstack+stack)
  → [Unstacking: 10 cycles] → Task

ประหยัด: 10 + 12 - 6 = 16 cycles ต่อ tail-chain
```

### 19.2 Late-Arrival

```
Late-arrival: ถ้า interrupt priority สูงกว่า arrive ระหว่าง stacking:
  Task → [Stacking starts for IRQ1]
  → [IRQ2 (higher priority) arrives during stacking]
  → [Complete stacking] → IRQ2 handler  (ไม่ใช่ IRQ1)
  → [Tail-chain] → IRQ1 handler
  → [Unstacking] → Task

ผล: ทั้ง IRQ1 และ IRQ2 จัดการด้วย stacking เพียงครั้งเดียว
```

---

## 20. Debugging — ITM และ SWD

### 20.1 ITM (Instrumentation Trace Macrocell)

```c
/* ITM printf สำหรับ debug output ผ่าน SWD */
#define ITM_PORT0_U32   (*(volatile uint32_t*)0xE0000000)
#define ITM_TER         (*(volatile uint32_t*)0xE0000E00)  /* Trace Enable */
#define ITM_TCR         (*(volatile uint32_t*)0xE0000E80)  /* Trace Control */

void ITM_SendChar(char c)
{
    /* Check if ITM is enabled and channel 0 enabled */
    if((ITM_TCR & 1) && (ITM_TER & 1))
    {
        /* Wait until port ready */
        while(ITM_PORT0_U32 == 0);
        /* Write character */
        *((volatile uint8_t*)0xE0000000) = (uint8_t)c;
    }
}

int _write(int fd, char *ptr, int len)
{
    for(int i = 0; i < len; i++) {
        ITM_SendChar(*ptr++);
    }
    return len;
}
```

### 20.2 DWT (Data Watchpoint and Trace) — Cycle Counter

```c
/* ใช้ DWT_CYCCNT สำหรับ cycle-accurate timing */
#define DWT_CTRL   (*(volatile uint32_t*)0xE0001000)
#define DWT_CYCCNT (*(volatile uint32_t*)0xE0001004)
#define CoreDebug_DEMCR (*(volatile uint32_t*)0xE000EDFC)

void DWT_Init(void)
{
    CoreDebug_DEMCR |= (1 << 24);   /* TRCENA: enable DWT */
    DWT_CYCCNT = 0;
    DWT_CTRL |= 1;                   /* CYCCNTENA */
}

uint32_t DWT_GetCycles(void)
{
    return DWT_CYCCNT;
}

/* ตัวอย่างการวัด execution time */
uint32_t t_start = DWT_GetCycles();
/* ... code to measure ... */
uint32_t t_end = DWT_GetCycles();
uint32_t elapsed_cycles = t_end - t_start;
/* elapsed_time_us = elapsed_cycles / (CPU_MHz) */
```

---

## 21. Advanced: Semihosting

### 21.1 Semihosting สำหรับ Debug I/O

```asm
/* Semihosting ใช้ BKPT instruction เพื่อ communicate กับ debugger */
/* ARM semihosting: BKPT 0xAB */

    .section .text.semihosting_write0
    .global semihosting_write0
semihosting_write0:
    /* R0 = operation code (0x04 = WRITE0, print null-terminated string) */
    /* R1 = pointer to string */
    MOV R0, #0x04              @ WRITE0 operation
    /* R1 already contains string pointer */
    BKPT 0xAB                  @ Semihosting call
    BX   LR
```

---

## 22. ตัวอย่างสมบูรณ์: RTOS Minimal

### 22.1 Two-Task Round-Robin Scheduler

```c
/* minimal_rtos.c — สาธิต context switch ใน 2 tasks */
#include <stdint.h>

#define STACK_SIZE 256  /* words */

/* Task stacks */
static uint32_t task1_stack[STACK_SIZE];
static uint32_t task2_stack[STACK_SIZE];

/* Task TCBs */
typedef struct { volatile uint32_t *sp; } TCB;
static TCB tcb1, tcb2;
TCB *current_tcb = &tcb1;
TCB *next_tcb    = &tcb2;

/* Task functions */
void task1_func(void *arg)
{
    volatile uint32_t cnt = 0;
    while(1) {
        cnt++;
        /* Toggle LED on PA5 */
        *(volatile uint32_t*)0x40020014 ^= (1 << 5);
        /* Yield — trigger PendSV */
        *(volatile uint32_t*)0xE000ED04 = (1 << 28);  /* PENDSVSET */
    }
}

void task2_func(void *arg)
{
    volatile uint32_t cnt = 0;
    while(1) {
        cnt++;
        /* Toggle LED on PA6 */
        *(volatile uint32_t*)0x40020014 ^= (1 << 6);
        /* Yield */
        *(volatile uint32_t*)0xE000ED04 = (1 << 28);
    }
}

/* Initialize task stack (same as section 12.4) */
static uint32_t *init_stack(uint32_t *top, void (*func)(void*), void *arg)
{
    *(--top) = 0x01000000;          /* xPSR */
    *(--top) = (uint32_t)func;      /* PC */
    *(--top) = 0xFFFFFFFD;          /* LR (EXC_RETURN) */
    *(--top) = 0;                   /* R12 */
    *(--top) = 0;                   /* R3 */
    *(--top) = 0;                   /* R2 */
    *(--top) = 0;                   /* R1 */
    *(--top) = (uint32_t)arg;       /* R0 */
    /* Software-saved by PendSV */
    *(--top) = 0xFFFFFFFD;          /* EXC_RETURN (as LR) */
    *(--top) = 0; *(--top) = 0;     /* R11, R10 */
    *(--top) = 0; *(--top) = 0;     /* R9, R8 */
    *(--top) = 0; *(--top) = 0;     /* R7, R6 */
    *(--top) = 0; *(--top) = 0;     /* R5, R4 */
    return top;
}

int main(void)
{
    /* Initialize task stacks */
    tcb1.sp = init_stack(task1_stack + STACK_SIZE, task1_func, 0);
    tcb2.sp = init_stack(task2_stack + STACK_SIZE, task2_func, 0);

    /* Set PendSV to lowest priority */
    *(volatile uint32_t*)0xE000ED20 |= (0xFF << 16);  /* SHPR3 PendSV = 255 */

    /* Switch to PSP and start first task via SVC */
    __asm volatile (
        "SVC #0\n"
    );

    while(1);  /* Never reached */
}
```

---

## 23. Power Management

### 23.1 Sleep Modes บน Cortex-M

```asm
/* WFI — Wait For Interrupt (Sleep Until IRQ) */
WFI    @ Processor สลีปจนกว่าจะมี interrupt

/* WFE — Wait For Event */
WFE    @ สลีปจนกว่าจะมี event (interrupt หรือ SEV instruction)

/* SEV — Send Event (broadcast to all processors ใน multicore) */
SEV

/* ใช้ SLEEPDEEP bit ใน SCB_SCR สำหรับ deep sleep */
LDR  R0, =0xE000ED10          @ SCB_SCR
LDR  R1, [R0]
ORR  R1, R1, #(1 << 2)       @ SLEEPDEEP
STR  R1, [R0]
WFI                            @ Enter deep sleep (Stop/Standby mode)
```

### 23.2 Low Power States

```
Cortex-M Sleep Modes (depend on vendor implementation):
1. Sleep mode:        WFI/WFE, CPU clock stopped, peripherals running
2. Deep Sleep mode:   SLEEPDEEP=1, multiple clock domains stopped
3. Vendor-specific:   STM32F4 adds:
   - Stop mode:       VDD domain on, 1.2V domain off
   - Standby mode:    Only backup domain powered (RTC, BKPSRAM)

STM32F4 Stop mode:
  - Regulator in low-power mode
  - All clocks stopped
  - SRAM and register contents preserved
  - Wake up: EXTI line, RTC alarm, IWDG

Wakeup sequence after Stop:
  HSI selected automatically as system clock
  Must reconfigure PLL if needed → call SystemInit()
```

---

## 24. สรุปสิ่งที่ต้องจำ

### 24.1 Key Addresses

```
System Control Space (SCS) Quick Reference:
0xE000E008  ACTLR  — Auxiliary Control Register
0xE000E010  SYST_CSR — SysTick Control
0xE000E014  SYST_RVR — SysTick Reload
0xE000E018  SYST_CVR — SysTick Current
0xE000E100  NVIC_ISER0 — IRQ Enable (31:0)
0xE000E180  NVIC_ICER0 — IRQ Disable
0xE000E200  NVIC_ISPR0 — IRQ Set Pending
0xE000E280  NVIC_ICPR0 — IRQ Clear Pending
0xE000E400  NVIC_IPR0  — IRQ Priority
0xE000ED00  SCB_CPUID  — CPU ID
0xE000ED04  SCB_ICSR   — Interrupt Control State (PENDSVSET bit28)
0xE000ED08  SCB_VTOR   — Vector Table Offset
0xE000ED0C  SCB_AIRCR  — Application Interrupt Reset Control
0xE000ED10  SCB_SCR    — System Control (SLEEPDEEP bit2)
0xE000ED14  SCB_CCR    — Configuration Control
0xE000ED18  SCB_SHPR1  — System Handler Priority 1
0xE000ED1C  SCB_SHPR2  — System Handler Priority 2
0xE000ED20  SCB_SHPR3  — System Handler Priority 3
0xE000ED28  SCB_CFSR   — Configurable Fault Status
0xE000ED2C  SCB_HFSR   — HardFault Status
0xE000ED34  SCB_MMFAR  — MemManage Address
0xE000ED38  SCB_BFAR   — BusFault Address
0xE000ED88  SCB_CPACR  — Coprocessor Access (FPU enable)
```

### 24.2 คำสั่ง Assembly ที่สำคัญ

```asm
/* Special register access */
MRS  Rd, <spec_reg>    @ Move from special register to Rd
MSR  <spec_reg>, Rn    @ Move from Rn to special register

/* Barriers — จำเป็นหลัง MSR CONTROL และ memory-mapped operations */
DSB                    @ Data Synchronization Barrier
DMB                    @ Data Memory Barrier
ISB                    @ Instruction Synchronization Barrier

/* Interrupt enable/disable */
CPSID I                @ Disable IRQ (PRIMASK=1)
CPSIE I                @ Enable IRQ
CPSID F                @ Disable IRQ+FaultMask
CPSIE F                @ Enable FaultMask

/* Load/Store multiple (Context save/restore) */
STMDB R0!, {R4-R11}    @ Push R4-R11, update R0 (used in PendSV)
LDMIA R0!, {R4-R11}    @ Pop R4-R11, update R0

/* Test and branch */
CBZ  Rn, label         @ Compare and branch if Zero
CBNZ Rn, label         @ Compare and branch if Not Zero
TBB  [Rn, Rm]          @ Table Branch Byte (jump table)
TBH  [Rn, Rm, LSL #1]  @ Table Branch Halfword
```

### 24.3 Checklist สำหรับ Bare-Metal Project

```
□ 1. Vector table — ISP (initial SP) + handlers ทั้งหมด ต้องมี Thumb bit
□ 2. Reset handler — copy .data, zero .bss, call SystemInit, call main
□ 3. SystemInit — clock config (HSE+PLL) + Flash latency
□ 4. Peripheral clock enable — RCC_AHBxENR / APBxENR
□ 5. GPIO config — MODER / OTYPER / OSPEEDR / PUPDR / AFR
□ 6. NVIC — priority + enable ก่อน peripheral interrupt enable
□ 7. PendSV priority — ต้องต่ำสุด (0xFF) สำหรับ RTOS
□ 8. Stack alignment — 8-byte aligned ก่อน calling convention
□ 9. ISB after MSR CONTROL — mandatory barrier
□ 10. Volatile for hardware registers — prevent compiler optimization
```

---

## 25. Workshop: GPIO Input + Interrupt

```asm
/* workshop_gpio_irq.s — ปุ่ม PA0 → interrupt → toggle LED PD12 */
    .syntax unified
    .cpu cortex-m4
    .thumb

    .section .text
    .global main

main:
    BL   enable_clocks
    BL   config_gpio
    BL   config_exti
    BL   config_nvic

loop:
    WFI                        @ Sleep until interrupt
    B    loop

enable_clocks:
    /* Enable GPIOA (bit0), GPIOD (bit3), SYSCFG (APB2 bit14) */
    LDR  R0, =0x40023830       @ RCC_AHB1ENR
    LDR  R1, [R0]
    ORR  R1, R1, #((1<<0)|(1<<3))
    STR  R1, [R0]
    LDR  R0, =0x40023844       @ RCC_APB2ENR
    LDR  R1, [R0]
    ORR  R1, R1, #(1<<14)      @ SYSCFGEN
    STR  R1, [R0]
    BX   LR

config_gpio:
    /* PA0: input, pull-up */
    LDR  R0, =0x40020000       @ GPIOA base
    LDR  R1, [R0, #0x0C]       @ PUPDR
    ORR  R1, R1, #(1<<0)       @ PUPDR0 = 01 (pull-up)
    STR  R1, [R0, #0x0C]
    /* PD12: output */
    LDR  R0, =0x40020C00       @ GPIOD base
    LDR  R1, [R0, #0x00]       @ MODER
    BIC  R1, R1, #(3<<24)
    ORR  R1, R1, #(1<<24)      @ MODER12 = output
    STR  R1, [R0, #0x00]
    BX   LR

config_exti:
    /* SYSCFG_EXTICR1: route PA0 to EXTI0 */
    LDR  R0, =0x40013808       @ SYSCFG_EXTICR1
    LDR  R1, [R0]
    BIC  R1, R1, #0x000F       @ Clear EXTI0[3:0]
    /* PA = 0000, so no need to set anything */
    STR  R1, [R0]

    /* EXTI_IMR: unmask EXTI0 */
    LDR  R0, =0x40013C00       @ EXTI_IMR
    LDR  R1, [R0]
    ORR  R1, R1, #(1<<0)
    STR  R1, [R0]

    /* EXTI_FTSR: falling edge trigger on EXTI0 */
    LDR  R0, =0x40013C0C       @ EXTI_FTSR
    LDR  R1, [R0]
    ORR  R1, R1, #(1<<0)
    STR  R1, [R0]
    BX   LR

config_nvic:
    /* EXTI0 = IRQ6, NVIC_ISER0 bit 6 */
    LDR  R0, =0xE000E100       @ NVIC_ISER0
    MOV  R1, #(1<<6)
    STR  R1, [R0]
    /* Priority = 5 */
    LDR  R0, =0xE000E406       @ NVIC_IPR1 byte 2 (IPR for IRQ6)
    MOV  R1, #(5<<4)
    STRB R1, [R0]
    BX   LR

    /* EXTI0 IRQ Handler */
    .global EXTI0_IRQHandler
EXTI0_IRQHandler:
    /* Clear pending bit */
    LDR  R0, =0x40013C14       @ EXTI_PR
    MOV  R1, #(1<<0)
    STR  R1, [R0]

    /* Toggle PD12 */
    LDR  R0, =0x40020C14       @ GPIOD_ODR
    LDR  R1, [R0]
    EOR  R1, R1, #(1<<12)
    STR  R1, [R0]

    BX   LR
```

---

## สรุปท้ายบท

ARM Cortex-M architecture ถูกออกแบบมาเพื่อ embedded systems โดยเฉพาะ:

1. **Thumb-2 only** — ไม่มี ARM32 state ทำให้ silicon area เล็กลงและ code dense
2. **Register file** — R0-R15 + xPSR (APSR/IPSR/EPSR) + special registers
3. **Two stack pointers** — MSP สำหรับ kernel/interrupt, PSP สำหรับ task
4. **NVIC** — nested, vectored interrupt controller รองรับถึง 240 IRQs
5. **Vector table** — เริ่มที่ 0x00000000, แต่ละ entry เป็น handler address + Thumb bit
6. **SysTick** — 24-bit countdown timer สำหรับ RTOS tick
7. **PendSV** — lowest priority exception สำหรับ context switch
8. **Context switch** — hardware auto-saves R0-R3/R12/LR/PC/xPSR, software saves R4-R11

การเขียน assembly สำหรับ Cortex-M ต้องเข้าใจ:
- Exception entry/return mechanism (EXC_RETURN values)
- Stack frame layout
- Barrier instructions (DSB/DMB/ISB)
- AAPCS calling convention
- Memory-mapped peripheral access pattern

บทถัดไป: **Part 096 — RISC-V Architecture** จะศึกษา open-source ISA ที่กำลังเติบโตอย่างรวดเร็ว

---

*จบ Part 095: Embedded Systems (ARM Cortex-M)*
