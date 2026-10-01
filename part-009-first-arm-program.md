# Part 009: โปรแกรม Assembly แรก (ARM)

**ระดับ:** พื้นฐาน → กลาง  
**เวลาที่ใช้:** ประมาณ 4-6 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 001-008 (x86 Assembly พื้นฐาน)

---

## วัตถุประสงค์การเรียนรู้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- เข้าใจโครงสร้าง syntax ของ ARM Assembly ด้วย GAS assembler
- เขียนโปรแกรม Hello World บน ARM Linux ทั้ง 32-bit และ 64-bit (AArch64)
- Compile และรัน ARM Assembly ด้วย cross-compiler และ QEMU emulator
- เข้าใจ ARM calling convention เบื้องต้น
- เขียน bare metal Hello World สำหรับ Raspberry Pi และ STM32 (Cortex-M)
- ใช้ ARM debugger และ objdump วิเคราะห์โปรแกรม
- เขียน inline assembly ใน C สำหรับ ARM architecture
- เปรียบเทียบความแตกต่างระหว่าง ARM และ x86 assembly

---

## 1. ทำความรู้จัก ARM Architecture

### 1.1 ARM คืออะไร?

**ARM** (Advanced RISC Machine) คือสถาปัตยกรรม processor แบบ **RISC** (Reduced Instruction Set Computing) ซึ่งแตกต่างจาก x86 ที่เป็น **CISC** (Complex Instruction Set Computing)

```
┌─────────────────────────────────────────────────┐
│           ARM vs x86 เปรียบเทียบเบื้องต้น          │
├─────────────────┬───────────────────────────────┤
│ คุณสมบัติ        │ ARM                  │ x86     │
├─────────────────┼──────────────────────┼─────────┤
│ ประเภท          │ RISC                 │ CISC    │
│ จำนวน registers │ 16-31 (general)      │ 8-16    │
│ instruction size│ Fixed (4 bytes)      │ Variable│
│ การใช้ไฟ        │ ต่ำมาก               │ สูงกว่า │
│ ใช้งานหลัก      │ Mobile, Embedded     │ Desktop │
│ ตัวอย่าง        │ Cortex-A/M/R series  │ Intel   │
└─────────────────┴──────────────────────┴─────────┘
```

### 1.2 ARM Versions ที่ควรรู้จัก

| Version | ชื่อเรียก | Instruction Set | ใช้กับ |
|---------|-----------|-----------------|--------|
| ARMv6 | ARM11 | ARM/Thumb | Raspberry Pi 1 |
| ARMv7-A | Cortex-A | ARM/Thumb-2 | Android, Pi 2/3 |
| ARMv7-M | Cortex-M | Thumb-2 only | STM32, Arduino Due |
| ARMv8-A | Cortex-A53/72 | AArch32/AArch64 | Pi 3/4, modern phones |
| ARMv9-A | Cortex-X | AArch64 | Snapdragon 8 Gen |

### 1.3 ARM Registers (32-bit ARMv7)

ARM มี registers ทั้งหมด 16 ตัวใน mode ปกติ:

```
┌──────┬──────────┬─────────────────────────────────────────┐
│ Register │ ชื่อ alias │ หน้าที่                             │
├──────────┼──────────┼─────────────────────────────────────┤
│ r0       │ a1       │ Argument 1 / Return value           │
│ r1       │ a2       │ Argument 2                          │
│ r2       │ a3       │ Argument 3                          │
│ r3       │ a4       │ Argument 4                          │
│ r4       │ v1       │ Variable (callee-saved)             │
│ r5       │ v2       │ Variable (callee-saved)             │
│ r6       │ v3       │ Variable (callee-saved)             │
│ r7       │ v4       │ Syscall number (Linux)              │
│ r8       │ v5       │ Variable (callee-saved)             │
│ r9       │ v6/SB    │ Variable / Static Base              │
│ r10      │ v7/SL    │ Variable / Stack Limit              │
│ r11      │ v8/FP    │ Frame Pointer                       │
│ r12      │ ip       │ Intra-procedure scratch             │
│ r13      │ sp       │ Stack Pointer                       │
│ r14      │ lr       │ Link Register (return address)      │
│ r15      │ pc       │ Program Counter                     │
└──────────┴──────────┴─────────────────────────────────────┘
```

### 1.4 ARM Registers (64-bit AArch64)

AArch64 มี registers เพิ่มขึ้นเป็น 31 general-purpose registers:

```
┌──────────┬──────────────────────────────────────────────┐
│ Register │ หน้าที่                                       │
├──────────┼──────────────────────────────────────────────┤
│ x0-x7    │ Arguments / Return values                    │
│ x8       │ Indirect result / syscall number             │
│ x9-x15   │ Temporary (caller-saved)                     │
│ x16-x17  │ Intra-procedure scratch                      │
│ x18      │ Platform register                            │
│ x19-x28  │ Callee-saved variables                       │
│ x29      │ Frame Pointer (FP)                           │
│ x30      │ Link Register (LR)                           │
│ xzr/sp   │ Zero register / Stack Pointer                │
└──────────┴──────────────────────────────────────────────┘

หมายเหตุ: w0-w30 คือ lower 32-bit ของ x0-x30
```

---

## 2. ARM Assembly Syntax ด้วย GAS

### 2.1 GAS (GNU Assembler) สำหรับ ARM

ต่างจาก x86 ที่เราใช้ NASM, สำหรับ ARM จะใช้ **GAS** (GNU Assembler) เป็นหลัก เพราะ:
- เป็น assembler มาตรฐานใน toolchain ของ ARM Linux
- รวมอยู่ใน cross-compiler `arm-linux-gnueabi-gcc`
- รองรับทุก ARM variant

### 2.2 โครงสร้างไฟล์ ARM Assembly (GAS syntax)

```asm
@ ไฟล์: example.s (GAS ARM syntax)
@ สัญลักษณ์ @ ใช้สำหรับ comment ใน ARM GAS
@ /* */ และ // ก็ใช้ได้เช่นกัน

    .section .data              @ ส่วนข้อมูล
message:
    .ascii "Hello ARM!\n"       @ กำหนด string (ไม่มี null terminator)
message_len = . - message       @ คำนวณความยาว string

    .section .text              @ ส่วน code
    .global _start              @ ประกาศ entry point เป็น global symbol

_start:                         @ entry point ของโปรแกรม
    @ ... code goes here ...
    
    mov r7, #1                  @ syscall number สำหรับ exit
    mov r0, #0                  @ exit code = 0
    swi 0                       @ Software Interrupt (system call)
```

### 2.3 ความแตกต่างของ Syntax GAS ARM vs NASM x86

```
┌────────────────────┬──────────────────────┬─────────────────────┐
│ Feature            │ GAS ARM              │ NASM x86            │
├────────────────────┼──────────────────────┼─────────────────────┤
│ Comment            │ @ หรือ /* */         │ ; หรือ # (NASM)     │
│ Directives         │ .section, .global    │ section, global     │
│ String             │ .ascii, .asciz       │ db, times           │
│ Integer data       │ .byte, .word, .long  │ db, dw, dd          │
│ Entry point        │ _start               │ _start              │
│ Instruction format │ op dst, src1, src2   │ op dst, src         │
│ Immediate values   │ #value               │ value (no prefix)   │
│ Hex values         │ #0x1F                │ 0x1F หรือ 1Fh       │
│ Memory access      │ [reg, offset]        │ [reg + offset]      │
│ Syscall            │ swi 0 (32-bit)       │ int 0x80            │
│                    │ svc #0 (AArch64)     │ syscall (x86-64)    │
└────────────────────┴──────────────────────┴─────────────────────┘
```

---

## 3. Hello World บน ARM Linux (32-bit)

### 3.1 โปรแกรม Hello World พื้นฐาน

```asm
@ ============================================================
@ ไฟล์: hello_arm32.s
@ คำอธิบาย: Hello World สำหรับ ARM Linux 32-bit
@ Compile: arm-linux-gnueabi-as -o hello_arm32.o hello_arm32.s
@          arm-linux-gnueabi-ld -o hello_arm32 hello_arm32.o
@ Run:     qemu-arm -L /usr/arm-linux-gnueabi ./hello_arm32
@ Output:  Hello, ARM World!
@ ============================================================

    .section .data                  @ เริ่ม data section
    
message:
    .ascii "Hello, ARM World!\n"    @ ข้อความที่จะแสดง (18 ตัวอักษร)
message_len = . - message           @ คำนวณความยาว: current pos - start of message

    .section .text                  @ เริ่ม code section
    .global _start                  @ ประกาศ _start เป็น global (entry point)

_start:
    @ ============ write syscall ============
    @ syscall number สำหรับ ARM Linux 32-bit write = 4
    @ prototype: ssize_t write(int fd, const void *buf, size_t count)
    
    mov r7, #4                      @ r7 = syscall number (4 = sys_write)
    mov r0, #1                      @ r0 = fd (1 = stdout)
    ldr r1, =message                @ r1 = pointer ไปยัง message
    mov r2, #message_len            @ r2 = จำนวน bytes ที่จะเขียน
    swi 0                           @ Software Interrupt: เรียก kernel
    
    @ ============ exit syscall ============
    @ syscall number สำหรับ ARM Linux 32-bit exit = 1
    
    mov r7, #1                      @ r7 = syscall number (1 = sys_exit)
    mov r0, #0                      @ r0 = exit code (0 = success)
    swi 0                           @ เรียก kernel เพื่อจบโปรแกรม
```

### 3.2 การ Compile และ Run

```bash
# ติดตั้ง cross-compiler และ QEMU (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y \
    gcc-arm-linux-gnueabi \
    binutils-arm-linux-gnueabi \
    qemu-user \
    qemu-user-static

# Assemble: แปลง .s ไปเป็น object file .o
arm-linux-gnueabi-as -o hello_arm32.o hello_arm32.s

# Link: แปลง object file ไปเป็น executable
arm-linux-gnueabi-ld -o hello_arm32 hello_arm32.o

# ตรวจสอบ file type
file hello_arm32
# Output: hello_arm32: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV)...

# Run ด้วย QEMU (user mode)
qemu-arm -L /usr/arm-linux-gnueabi ./hello_arm32
# Output: Hello, ARM World!

# หรือใช้ qemu-arm-static (สำหรับ static binary)
qemu-arm-static ./hello_arm32
```

### 3.3 การวิเคราะห์ด้วย objdump

```bash
# ดู disassembly ของ executable
arm-linux-gnueabi-objdump -d hello_arm32

# ผลลัพธ์จะแสดงประมาณนี้:
# hello_arm32:     file format elf32-littlearm
# 
# Disassembly of section .text:
# 
# 00010054 <_start>:
#    10054:       e3a07004        mov     r7, #4
#    10058:       e3a00001        mov     r0, #1
#    1005c:       e59f1010        ldr     r1, [pc, #16]
#    10060:       e3a02012        mov     r2, #18
#    10064:       ef000000        svc     0x00000000
#    10068:       e3a07001        mov     r7, #1
#    1006c:       e3a00000        mov     r0, #0
#    10070:       ef000000        svc     0x00000000

# ดู symbols ทั้งหมด
arm-linux-gnueabi-nm hello_arm32

# ดู sections
arm-linux-gnueabi-objdump -h hello_arm32

# ดู raw hex + disassembly
arm-linux-gnueabi-objdump -D -M force-thumb hello_arm32
```

---

## 4. Hello World บน AArch64 Linux (64-bit)

### 4.1 โปรแกรม Hello World AArch64

```asm
// ============================================================
// ไฟล์: hello_aarch64.s
// คำอธิบาย: Hello World สำหรับ AArch64 (ARM 64-bit) Linux
// Compile: aarch64-linux-gnu-as -o hello_aarch64.o hello_aarch64.s
//          aarch64-linux-gnu-ld -o hello_aarch64 hello_aarch64.o
// Run:     qemu-aarch64 -L /usr/aarch64-linux-gnu ./hello_aarch64
// Output:  Hello, AArch64 World!
// ============================================================

    .section .data                      // เริ่ม data section
    
message:
    .ascii "Hello, AArch64 World!\n"    // ข้อความ (22 ตัวอักษร)
message_len = . - message               // ความยาว string

    .section .text                      // เริ่ม code section
    .global _start                      // ประกาศ entry point

_start:
    // ============ write syscall (AArch64) ============
    // AArch64 syscall numbers แตกต่างจาก ARM 32-bit!
    // write = 64 (0x40)
    // Arguments: x0=fd, x1=buf, x2=count
    // Syscall number ใส่ใน x8
    
    mov x8, #64                         // x8 = syscall number (64 = sys_write)
    mov x0, #1                          // x0 = fd (1 = stdout)
    adr x1, message                     // x1 = address ของ message (PC-relative)
    mov x2, #message_len                // x2 = จำนวน bytes
    svc #0                              // Supervisor Call: เรียก kernel
    
    // ============ exit syscall (AArch64) ============
    // exit = 93 (0x5D) ใน AArch64
    
    mov x8, #93                         // x8 = syscall number (93 = sys_exit)
    mov x0, #0                          // x0 = exit code
    svc #0                              // เรียก kernel

// ============================================================
// สรุป syscall numbers ที่ต่างกัน:
// 
// Syscall   │ ARM 32-bit │ AArch64
// ──────────┼────────────┼────────
// write     │ 4          │ 64
// exit      │ 1          │ 93
// read      │ 3          │ 63
// open      │ 5          │ 56
// close     │ 6          │ 57
// ============================================================
```

### 4.2 การ Compile AArch64

```bash
# ติดตั้ง AArch64 cross-compiler
sudo apt-get install -y \
    gcc-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    qemu-user

# Assemble
aarch64-linux-gnu-as -o hello_aarch64.o hello_aarch64.s

# Link
aarch64-linux-gnu-ld -o hello_aarch64 hello_aarch64.o

# ตรวจสอบ
file hello_aarch64
# Output: hello_aarch64: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV)...

# Run
qemu-aarch64 -L /usr/aarch64-linux-gnu ./hello_aarch64
# Output: Hello, AArch64 World!

# Disassembly
aarch64-linux-gnu-objdump -d hello_aarch64
```

---

## 5. ARM Calling Convention (AAPCS)

### 5.1 AAPCS (ARM Architecture Procedure Call Standard)

กฎสำหรับการเรียกฟังก์ชันใน ARM:

```
┌────────────────────────────────────────────────────────────┐
│          ARM 32-bit Calling Convention (AAPCS)             │
├────────────┬───────────────────────────────────────────────┤
│ Registers  │ หน้าที่                                        │
├────────────┼───────────────────────────────────────────────┤
│ r0-r3      │ Arguments 1-4 / Return value (r0, r0+r1)      │
│ r4-r11     │ Callee-saved (ฟังก์ชันต้อง save/restore)       │
│ r12 (ip)   │ Scratch register (ทำลายได้)                    │
│ r13 (sp)   │ Stack Pointer (ต้อง 8-byte aligned)           │
│ r14 (lr)   │ Link Register (return address)                │
│ r15 (pc)   │ Program Counter                               │
└────────────┴───────────────────────────────────────────────┘
```

### 5.2 ตัวอย่าง Function Call

```asm
@ ============================================================
@ ไฟล์: function_call_arm.s
@ คำอธิบาย: ตัวอย่างการเรียกฟังก์ชันใน ARM
@ ============================================================

    .section .data
result_msg:
    .ascii "Sum = "
result_num:
    .ascii "XX\n"                   @ placeholder สำหรับตัวเลข

    .section .text
    .global _start

@ ฟังก์ชัน: int add(int a, int b)
@ Input:  r0 = a, r1 = b  
@ Output: r0 = a + b
add_numbers:
    @ ไม่ต้อง save registers เพราะใช้แค่ r0, r1
    add r0, r0, r1                  @ r0 = r0 + r1 (ARM: add dst, src1, src2)
    bx lr                           @ return (branch to Link Register)

@ ฟังก์ชัน: void print_char(char c)
@ Input: r0 = character to print
print_char:
    push {r1-r7}                    @ save registers ที่จะใช้
    
    mov r2, r0                      @ เก็บ char ไว้ใน r2 ชั่วคราว
    
    @ สร้าง string บน stack
    sub sp, sp, #4                  @ จองพื้นที่ 4 bytes บน stack
    strb r2, [sp]                   @ เก็บ char ลง stack
    
    mov r7, #4                      @ syscall = write
    mov r0, #1                      @ fd = stdout
    mov r1, sp                      @ pointer ไปยัง char บน stack
    mov r2, #1                      @ 1 byte
    swi 0
    
    add sp, sp, #4                  @ คืน stack
    pop {r1-r7}                     @ restore registers
    bx lr                           @ return

_start:
    @ เรียก add_numbers(15, 27)
    mov r0, #15                     @ argument 1 = 15
    mov r1, #27                     @ argument 2 = 27
    bl add_numbers                  @ Branch with Link (เรียกฟังก์ชัน, lr = return addr)
    @ ตอนนี้ r0 = 42 (15 + 27)
    
    @ แสดงผลลัพธ์ (convert เป็น ASCII แล้วแสดง)
    add r0, r0, #'0'                @ ถ้าผลลัพธ์ < 10: convert เป็น ASCII digit
    bl print_char                   @ แสดง character
    
    @ แสดง newline
    mov r0, #'\n'
    bl print_char
    
    @ exit
    mov r7, #1
    mov r0, #0
    swi 0
```

### 5.3 Stack Frame ใน ARM

```asm
@ ============================================================
@ ตัวอย่าง: proper stack frame setup
@ ============================================================

my_function:
    @ Function prologue - setup stack frame
    push {r4-r11, lr}               @ save callee-saved registers + lr
    sub sp, sp, #16                 @ จองพื้นที่ local variables (16 bytes)
    
    @ ทำงานของฟังก์ชัน...
    str r0, [sp, #0]                @ เก็บ argument ลง local var
    str r1, [sp, #4]                @ เก็บ argument ลง local var
    
    @ ทำการคำนวณ
    ldr r4, [sp, #0]                @ โหลด local var
    ldr r5, [sp, #4]
    mul r0, r4, r5                  @ r0 = r4 * r5 (return value)
    
    @ Function epilogue - restore stack frame
    add sp, sp, #16                 @ คืน local variables space
    pop {r4-r11, pc}                @ restore registers + return (pc = lr)
    @ หมายเหตุ: pop {pc} จะ load lr ไปยัง pc ซึ่งทำการ return
```

---

## 6. Raspberry Pi Hello World (Bare Metal)

### 6.1 Bare Metal คืออะไร?

Bare metal หมายถึงการเขียน code ที่รันโดยตรงบน hardware โดยไม่มี OS รองรับ ต้องจัดการทุกอย่างเอง:
- กำหนด entry point
- ตั้งค่า stack
- เปิดใช้งาน peripherals (UART, GPIO, etc.)

### 6.2 Memory Map ของ Raspberry Pi 2/3

```
┌─────────────────────────────────────────────────────────────┐
│            Raspberry Pi 2/3 (BCM2836/2837) Memory Map        │
├──────────────────────┬──────────────────────────────────────┤
│ Address Range        │ Description                          │
├──────────────────────┼──────────────────────────────────────┤
│ 0x00000000-0x3EFFFFFF│ RAM (1GB)                            │
│ 0x3F000000-0x3FFFFFFF│ Peripherals (BCM2836 peripherals)    │
│ 0x3F201000           │ UART0 (PL011)                        │
│ 0x3F215000           │ AUX (Mini UART)                      │
│ 0x3F200000           │ GPIO                                 │
│ 0x40000000+          │ Local peripherals (ARM Timer, etc.)  │
└──────────────────────┴──────────────────────────────────────┘

Raspberry Pi 1 (BCM2835): peripherals start at 0x20000000
```

### 6.3 Bare Metal Hello World ผ่าน UART

```asm
@ ============================================================
@ ไฟล์: rpi_baremetal.s
@ คำอธิบาย: Bare Metal Hello World สำหรับ Raspberry Pi 2/3
@ Compile: arm-none-eabi-as -o rpi_baremetal.o rpi_baremetal.s
@          arm-none-eabi-ld -T linker.ld -o kernel.elf rpi_baremetal.o
@          arm-none-eabi-objcopy -O binary kernel.elf kernel.img
@ ============================================================

@ Peripheral base address
.equ PERIPHERAL_BASE,   0x3F000000      @ BCM2836/2837 (Pi 2/3)
.equ GPIO_BASE,         PERIPHERAL_BASE + 0x200000
.equ UART0_BASE,        PERIPHERAL_BASE + 0x201000

@ UART0 Register offsets
.equ UART0_DR,          0x00            @ Data Register
.equ UART0_FR,          0x18            @ Flag Register
.equ UART0_IBRD,        0x24            @ Integer Baud Rate Divisor
.equ UART0_FBRD,        0x28            @ Fractional Baud Rate Divisor
.equ UART0_LCRH,        0x2C            @ Line Control Register
.equ UART0_CR,          0x30            @ Control Register
.equ UART0_IMSC,        0x38            @ Interrupt Mask
.equ UART0_ICR,         0x44            @ Interrupt Clear Register

@ Flag Register bits
.equ UART0_FR_BUSY,     (1 << 3)        @ UART busy
.equ UART0_FR_TXFF,     (1 << 5)        @ Transmit FIFO full

    .section .text
    .global _start

_start:
    @ ตั้งค่า stack pointer
    ldr sp, =0x8000                 @ stack เริ่มต้นที่ 0x8000 (ลดลง)
    
    @ เรียกฟังก์ชัน uart_init
    bl uart_init
    
    @ แสดงข้อความ
    ldr r0, =hello_msg              @ r0 = pointer ไปยัง string
    bl uart_puts                    @ เรียก uart_puts
    
    @ วนลูปไม่สิ้นสุด (Bare metal ต้องวนลูป)
halt:
    b halt                          @ บ้านๆ infinite loop

@ ============================================================
@ ฟังก์ชัน: uart_init - ตั้งค่า UART0
@ ============================================================
uart_init:
    push {r4, r5, lr}
    
    ldr r4, =UART0_BASE             @ r4 = base address ของ UART0
    
    @ ปิด UART ก่อนตั้งค่า
    mov r5, #0
    str r5, [r4, #UART0_CR]        @ UART0_CR = 0 (disable)
    
    @ ล้าง Interrupts ทั้งหมด
    mov r5, #0x7FF
    str r5, [r4, #UART0_ICR]       @ clear all interrupts
    
    @ ตั้งค่า Baud Rate: 115200
    @ Clock = 3MHz, 115200 baud
    @ Divisor = 3000000 / (16 * 115200) = 1.627...
    @ IBRD = 1, FBRD = 40 (0.627 * 64 + 0.5 = 40.6 -> 40)
    mov r5, #1
    str r5, [r4, #UART0_IBRD]      @ Integer baud rate = 1
    mov r5, #40
    str r5, [r4, #UART0_FBRD]      @ Fractional baud rate = 40
    
    @ ตั้งค่า Line Control: 8 bits, no parity, 1 stop bit, enable FIFO
    mov r5, #0x70                   @ 0b01110000: WLEN=11(8bit), FEN=1(FIFO enable)
    str r5, [r4, #UART0_LCRH]
    
    @ เปิดใช้งาน UART: enable TX, RX, UART
    mov r5, #0x301                  @ RXEN | TXEN | UARTEN
    str r5, [r4, #UART0_CR]
    
    pop {r4, r5, pc}

@ ============================================================
@ ฟังก์ชัน: uart_putc - ส่ง 1 character ผ่าน UART
@ Input: r0 = character
@ ============================================================
uart_putc:
    push {r1, r2, lr}
    
    ldr r1, =UART0_BASE
    
uart_putc_wait:
    ldr r2, [r1, #UART0_FR]        @ อ่าน Flag Register
    tst r2, #UART0_FR_TXFF          @ ตรวจสอบ TX FIFO full
    bne uart_putc_wait              @ ถ้า FIFO เต็ม รอก่อน
    
    str r0, [r1, #UART0_DR]        @ ส่ง character ไปยัง Data Register
    
    pop {r1, r2, pc}

@ ============================================================
@ ฟังก์ชัน: uart_puts - ส่ง string ผ่าน UART
@ Input: r0 = pointer ไปยัง null-terminated string
@ ============================================================
uart_puts:
    push {r4, r5, lr}
    
    mov r4, r0                      @ r4 = string pointer
    
uart_puts_loop:
    ldrb r0, [r4], #1              @ โหลด byte จาก string แล้วเลื่อน pointer
    cmp r0, #0                      @ ตรวจสอบ null terminator
    beq uart_puts_done              @ ถ้า null จบลูป
    
    bl uart_putc                    @ ส่ง character
    b uart_puts_loop                @ วนซ้ำ

uart_puts_done:
    pop {r4, r5, pc}

    .section .data
hello_msg:
    .asciz "Hello, Raspberry Pi!\r\n"    @ null-terminated string
```

### 6.4 Linker Script สำหรับ Raspberry Pi

```ld
/* ไฟล์: linker.ld */
/* Linker script สำหรับ Raspberry Pi bare metal */

ENTRY(_start)               /* entry point */

SECTIONS {
    . = 0x8000;             /* โค้ดเริ่มที่ 0x8000 (ตำแหน่งที่ bootloader โหลดมา) */
    
    .text : {
        *(.text)            /* รวม text sections ทั้งหมด */
    }
    
    .data : {
        *(.data)            /* รวม data sections ทั้งหมด */
    }
    
    .bss : {
        *(.bss)             /* รวม bss sections ทั้งหมด (uninitialized data) */
    }
}
```

```bash
# Build commands สำหรับ Raspberry Pi bare metal
# ติดตั้ง toolchain
sudo apt-get install -y gcc-arm-none-eabi binutils-arm-none-eabi

# Assemble
arm-none-eabi-as -mcpu=cortex-a7 -o rpi_baremetal.o rpi_baremetal.s

# Link
arm-none-eabi-ld -T linker.ld -o kernel.elf rpi_baremetal.o

# สร้าง binary image
arm-none-eabi-objcopy -O binary kernel.elf kernel.img

# ตรวจสอบ size
arm-none-eabi-size kernel.elf

# รัน บน QEMU (Pi 2 emulation)
qemu-system-arm \
    -machine raspi2 \
    -kernel kernel.img \
    -serial stdio \
    -no-reboot
```

---

## 7. STM32 Hello World (Cortex-M UART)

### 7.1 STM32 Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│              STM32F4 (Cortex-M4) Memory Map                 │
├──────────────────────┬─────────────────────────────────────┤
│ 0x00000000           │ Vector Table (flash mirror)         │
│ 0x08000000           │ Flash Memory (512KB - 2MB)          │
│ 0x20000000           │ SRAM (128KB - 384KB)                │
│ 0x40000000           │ APB1 Peripherals                    │
│ 0x40010000           │ APB2 Peripherals                    │
│ 0x40020000           │ AHB1 Peripherals (GPIO, RCC, DMA)  │
│ 0xE0000000           │ ARM Cortex-M4 Internal Peripherals  │
└──────────────────────┴─────────────────────────────────────┘
```

### 7.2 STM32 Hello World via USART2

```asm
@ ============================================================
@ ไฟล์: stm32_hello.s
@ คำอธิบาย: Hello World บน STM32F4 ผ่าน USART2
@ Target: STM32F407 (Cortex-M4)
@ Compile: arm-none-eabi-as -mcpu=cortex-m4 -mthumb -o stm32_hello.o stm32_hello.s
@ ============================================================

@ STM32F4xx Base Addresses
.equ RCC_BASE,          0x40023800      @ Reset and Clock Control
.equ GPIOA_BASE,        0x40020000      @ GPIO Port A
.equ USART2_BASE,       0x40004400      @ USART2

@ RCC Register offsets
.equ RCC_AHB1ENR,       0x30            @ AHB1 peripheral clock enable
.equ RCC_APB1ENR,       0x40            @ APB1 peripheral clock enable

@ GPIO Register offsets  
.equ GPIO_MODER,        0x00            @ Mode register
.equ GPIO_AFRL,         0x20            @ Alternate function low register

@ USART Register offsets
.equ USART_SR,          0x00            @ Status register
.equ USART_DR,          0x04            @ Data register
.equ USART_BRR,         0x08            @ Baud rate register
.equ USART_CR1,         0x0C            @ Control register 1

@ Bit definitions
.equ RCC_AHB1ENR_GPIOAEN,  (1 << 0)    @ GPIOA clock enable
.equ RCC_APB1ENR_USART2EN, (1 << 17)   @ USART2 clock enable
.equ USART_SR_TXE,          (1 << 7)    @ TXE: Transmit data register empty
.equ USART_CR1_UE,          (1 << 13)   @ UE: USART enable
.equ USART_CR1_TE,          (1 << 3)    @ TE: Transmitter enable

    .syntax unified                     @ ใช้ Unified Assembly Language (UAL)
    .cpu cortex-m4                      @ target CPU
    .thumb                              @ Cortex-M ใช้ Thumb mode เสมอ

@ Vector Table - ต้องอยู่ที่ offset 0 ของ flash
    .section .isr_vector, "a"
    .word   0x20020000                  @ Initial Stack Pointer (top of 128KB SRAM)
    .word   reset_handler               @ Reset Handler
    .word   default_handler             @ NMI Handler
    .word   default_handler             @ HardFault Handler
    @ ... (ISR vectors อื่นๆ)

    .section .text
    .thumb_func
    .global reset_handler

reset_handler:
    @ ตั้งค่า stack pointer (เพิ่มเติมจาก vector table)
    ldr sp, =0x20020000                 @ Stack top
    
    @ เรียก main
    bl main
    
    @ ถ้า main return ให้วนลูปไม่สิ้นสุด
    b .

    .thumb_func
main:
    push {lr}
    
    bl setup_clocks                     @ ตั้งค่า clock
    bl setup_gpio                       @ ตั้งค่า GPIO สำหรับ UART
    bl setup_usart                      @ ตั้งค่า USART
    
    ldr r0, =hello_msg
    bl usart_puts                       @ แสดงข้อความ
    
    @ วนลูปไม่สิ้นสุด
loop:
    b loop
    
    pop {pc}

@ ตั้งค่า Clock: เปิด GPIOA และ USART2
setup_clocks:
    ldr r0, =RCC_BASE
    
    @ เปิด GPIOA clock
    ldr r1, [r0, #RCC_AHB1ENR]
    orr r1, r1, #RCC_AHB1ENR_GPIOAEN
    str r1, [r0, #RCC_AHB1ENR]
    
    @ เปิด USART2 clock
    ldr r1, [r0, #RCC_APB1ENR]
    orr r1, r1, #RCC_APB1ENR_USART2EN
    str r1, [r0, #RCC_APB1ENR]
    
    bx lr

@ ตั้งค่า GPIO: PA2 = USART2_TX (Alternate Function 7)
setup_gpio:
    ldr r0, =GPIOA_BASE
    
    @ PA2 = Alternate function mode (MODER bits [5:4] = 10)
    ldr r1, [r0, #GPIO_MODER]
    bic r1, r1, #(3 << 4)              @ ล้าง bits 5:4
    orr r1, r1, #(2 << 4)              @ set AF mode
    str r1, [r0, #GPIO_MODER]
    
    @ PA2 = AF7 (USART2) ใน AFRL bits [11:8]
    ldr r1, [r0, #GPIO_AFRL]
    bic r1, r1, #(0xF << 8)            @ ล้าง bits 11:8
    orr r1, r1, #(7 << 8)              @ set AF7
    str r1, [r0, #GPIO_AFRL]
    
    bx lr

@ ตั้งค่า USART2: 115200 baud, 8N1
setup_usart:
    ldr r0, =USART2_BASE
    
    @ ตั้งค่า Baud Rate: 115200 บน APB1 clock 16MHz
    @ USARTDIV = 16MHz / (16 * 115200) = 8.68...
    @ Mantissa = 8, Fraction = 0.68 * 16 = 10.9 -> 11
    @ BRR = (8 << 4) | 11 = 0x008B
    mov r1, #0x008B
    str r1, [r0, #USART_BRR]
    
    @ เปิด USART: UE + TE
    mov r1, #(USART_CR1_UE | USART_CR1_TE)
    str r1, [r0, #USART_CR1]
    
    bx lr

@ ส่ง 1 byte ผ่าน USART
@ r0 = byte to send
usart_putc:
    ldr r1, =USART2_BASE
    
usart_putc_wait:
    ldr r2, [r1, #USART_SR]            @ อ่าน Status Register
    tst r2, #USART_SR_TXE               @ ตรวจ TXE bit
    beq usart_putc_wait                 @ รอถ้า TX buffer ยังไม่ว่าง
    
    str r0, [r1, #USART_DR]            @ ส่ง byte
    bx lr

@ ส่ง null-terminated string
@ r0 = pointer to string
usart_puts:
    push {r4, lr}
    mov r4, r0
    
usart_puts_loop:
    ldrb r0, [r4], #1
    cbz r0, usart_puts_done             @ CBZ = Compare and Branch if Zero (Thumb-2)
    bl usart_putc
    b usart_puts_loop
    
usart_puts_done:
    pop {r4, pc}

default_handler:
    b .                                 @ Infinite loop for unhandled interrupts

    .section .rodata
hello_msg:
    .asciz "Hello, STM32!\r\n"
```

```bash
# Build สำหรับ STM32
arm-none-eabi-as \
    -mcpu=cortex-m4 \
    -mthumb \
    -mfpu=fpv4-sp-d16 \
    -mfloat-abi=hard \
    -o stm32_hello.o \
    stm32_hello.s

arm-none-eabi-ld \
    -T stm32f4.ld \
    -o stm32_hello.elf \
    stm32_hello.o

# แปลงเป็น binary สำหรับ flash
arm-none-eabi-objcopy -O binary stm32_hello.elf stm32_hello.bin

# Flash ด้วย st-flash (ถ้ามี ST-Link)
st-flash write stm32_hello.bin 0x08000000

# หรือรันบน QEMU (ถ้ามี QEMU + STM32 support)
qemu-system-arm \
    -machine netduino2 \
    -kernel stm32_hello.elf \
    -serial stdio
```

---

## 8. ARM Debugger

### 8.1 การใช้ arm-linux-gnueabi-gdb

```bash
# Compile with debug info (-g flag)
arm-linux-gnueabi-as -g -o hello_arm32.o hello_arm32.s
arm-linux-gnueabi-ld -o hello_arm32 hello_arm32.o

# Start GDB debugging
qemu-arm -g 1234 -L /usr/arm-linux-gnueabi ./hello_arm32 &
arm-linux-gnueabi-gdb hello_arm32

# คำสั่งใน GDB:
(gdb) target remote :1234    # เชื่อมต่อกับ QEMU
(gdb) break _start           # set breakpoint ที่ _start
(gdb) continue               # รันจนถึง breakpoint
(gdb) info registers         # แสดง ARM registers ทั้งหมด
(gdb) x/10i $pc              # ดู 10 instructions จาก PC
(gdb) x/s $r1                # ดู string ที่ r1 ชี้ไปยัง
(gdb) stepi                  # Execute 1 instruction
(gdb) nexti                  # Execute 1 instruction (ไม่ step into call)
(gdb) print $r0              # แสดงค่าใน r0
(gdb) disassemble            # disassemble ฟังก์ชันปัจจุบัน
```

### 8.2 GDB Script สำหรับ ARM

```
# ไฟล์: arm_debug.gdb
# GDB script สำหรับ debug ARM programs

# เชื่อมต่อ remote target
target remote :1234

# ตั้งค่า architecture
set architecture arm

# set output format
set disassembly-flavor att

# Define custom command เพื่อแสดง ARM registers
define arm_regs
    printf "r0 = 0x%08x  r1 = 0x%08x  r2 = 0x%08x  r3 = 0x%08x\n", $r0, $r1, $r2, $r3
    printf "r4 = 0x%08x  r5 = 0x%08x  r6 = 0x%08x  r7 = 0x%08x\n", $r4, $r5, $r6, $r7
    printf "sp = 0x%08x  lr = 0x%08x  pc = 0x%08x\n", $sp, $lr, $pc
end

# Breakpoint ที่ _start
break _start
continue

# แสดง registers
arm_regs
```

```bash
# Run GDB ด้วย script
arm-linux-gnueabi-gdb -x arm_debug.gdb hello_arm32
```

---

## 9. Inline Assembly ใน C สำหรับ ARM

### 9.1 GCC Inline Assembly Syntax

```c
// ============================================================
// ไฟล์: inline_arm.c
// คำอธิบาย: Inline Assembly ใน C สำหรับ ARM
// Compile: arm-linux-gnueabi-gcc -o inline_arm inline_arm.c
// ============================================================

#include <stdio.h>
#include <stdint.h>

// Basic inline assembly
void basic_inline_example() {
    int result;
    
    // GCC inline asm syntax:
    // asm("instruction" : outputs : inputs : clobbers);
    
    // ตัวอย่าง: r0 = 5 + 3 ด้วย ARM assembly
    asm("mov r0, #5\n\t"           // r0 = 5
        "add r0, r0, #3\n\t"       // r0 = r0 + 3
        "mov %0, r0"                // output: result = r0
        : "=r" (result)             // output operand: result ใน register
        :                           // ไม่มี inputs
        : "r0"                      // clobbers: บอก GCC ว่าเราใช้ r0
        );
    
    printf("Result: %d\n", result); // Output: Result: 8
}

// Inline assembly ที่ดีกว่า: ใช้ constraints
int add_asm(int a, int b) {
    int result;
    
    asm("add %0, %1, %2"           // result = a + b
        : "=r" (result)             // output: result ใน register (%0)
        : "r" (a), "r" (b)          // inputs: a=%1, b=%2 ใน registers
        :                           // ไม่มี clobbers
        );
    
    return result;
}

// ตัวอย่าง: การอ่าน CPSR (Current Program Status Register)
uint32_t get_cpsr() {
    uint32_t cpsr;
    
    asm("mrs %0, cpsr"             // MRS = Move from System Register to Register
        : "=r" (cpsr)
        );
    
    return cpsr;
}

// ตัวอย่าง: Memory barrier
void memory_barrier() {
    asm volatile("dmb"             // DMB = Data Memory Barrier
                 ::: "memory");    // clobbers memory
}

// ตัวอย่าง: CLZ (Count Leading Zeros)
int count_leading_zeros(uint32_t x) {
    int count;
    
    asm("clz %0, %1"               // CLZ instruction
        : "=r" (count)
        : "r" (x)
        );
    
    return count;
}

// ตัวอย่าง: SIMD/NEON inline (ARM-specific)
// ใช้สำหรับ vector operations
void neon_add_example() {
    int32_t a[4] = {1, 2, 3, 4};
    int32_t b[4] = {10, 20, 30, 40};
    int32_t c[4];
    
    asm("vld1.32 {d0, d1}, [%1]\n\t"   // โหลด 4x32-bit integers จาก a
        "vld1.32 {d2, d3}, [%2]\n\t"   // โหลด 4x32-bit integers จาก b
        "vadd.i32 q0, q0, q1\n\t"      // NEON: q0 = q0 + q1 (4 integers parallel)
        "vst1.32 {d0, d1}, [%0]"        // เก็บผลลัพธ์ลัง c
        :
        : "r" (c), "r" (a), "r" (b)
        : "q0", "q1", "memory"          // clobbers: q0, q1 registers, memory
        );
    
    printf("NEON result: %d %d %d %d\n", c[0], c[1], c[2], c[3]);
    // Output: NEON result: 11 22 33 44
}

// ตัวอย่าง AArch64 inline assembly
#if defined(__aarch64__)
uint64_t get_system_counter() {
    uint64_t count;
    
    asm("mrs %0, cntvct_el0"       // อ่าน virtual timer counter
        : "=r" (count)
        );
    
    return count;
}

int64_t neon64_dot_product(int32_t* a, int32_t* b, int n) {
    int64_t result = 0;
    
    // NEON AArch64: คำนวณ dot product
    asm("movi v0.4s, #0\n\t"       // v0 = 0
        "1:\n\t"
        "cbz %2, 2f\n\t"
        "ldr q1, [%0], #16\n\t"    // โหลด 4 int32 จาก a
        "ldr q2, [%1], #16\n\t"    // โหลด 4 int32 จาก b
        "mul v3.4s, v1.4s, v2.4s\n\t" // multiply
        "addv s4, v3.4s\n\t"       // ลดผลลัพธ์
        "saddw x3, xzr, s4\n\t"
        "add %3, %3, x3\n\t"
        "sub %2, %2, #4\n\t"
        "b 1b\n\t"
        "2:"
        : "+r" (a), "+r" (b), "+r" (n), "+r" (result)
        :
        : "v0", "v1", "v2", "v3", "v4", "x3"
        );
    
    return result;
}
#endif

int main() {
    printf("=== ARM Inline Assembly Examples ===\n");
    
    basic_inline_example();
    
    int sum = add_asm(15, 27);
    printf("add_asm(15, 27) = %d\n", sum);
    
    uint32_t cpsr = get_cpsr();
    printf("CPSR = 0x%08X\n", cpsr);
    
    int clz = count_leading_zeros(0x00FF0000);
    printf("CLZ(0x00FF0000) = %d\n", clz);  // Output: 8
    
    neon_add_example();
    
    return 0;
}
```

```bash
# Compile สำหรับ ARM 32-bit
arm-linux-gnueabi-gcc \
    -march=armv7-a \
    -mfpu=neon \
    -mfloat-abi=softfp \
    -o inline_arm \
    inline_arm.c

# Compile สำหรับ AArch64
aarch64-linux-gnu-gcc \
    -march=armv8-a \
    -o inline_aarch64 \
    inline_arm.c

# Run
qemu-arm -L /usr/arm-linux-gnueabi ./inline_arm
```

---

## 10. เปรียบเทียบ ARM vs x86 Hello World

### 10.1 x86 Hello World (NASM, ทบทวน)

```asm
; ============================================================
; ไฟล์: hello_x86.asm (NASM syntax, x86 32-bit)
; ============================================================

section .data
    message db "Hello, x86 World!", 10     ; string + newline
    message_len equ $ - message             ; ความยาว

section .text
    global _start

_start:
    ; write syscall (x86 Linux)
    mov eax, 4                  ; syscall number 4 = sys_write
    mov ebx, 1                  ; fd = 1 (stdout)
    mov ecx, message            ; pointer ไปยัง message
    mov edx, message_len        ; ความยาว
    int 0x80                    ; เรียก kernel
    
    ; exit syscall
    mov eax, 1                  ; syscall number 1 = sys_exit
    xor ebx, ebx                ; exit code = 0
    int 0x80
```

### 10.2 x86-64 Hello World

```asm
; ============================================================
; ไฟล์: hello_x64.asm (NASM syntax, x86-64)
; ============================================================

section .data
    message db "Hello, x64 World!", 10
    message_len equ $ - message

section .text
    global _start

_start:
    ; write syscall (x86-64 Linux)
    mov rax, 1                  ; syscall number 1 = sys_write (ต่างจาก 32-bit!)
    mov rdi, 1                  ; fd = 1 (stdout)
    mov rsi, message            ; pointer ไปยัง message
    mov rdx, message_len        ; ความยาว
    syscall                     ; เรียก kernel (ต่างจาก int 0x80)
    
    ; exit syscall
    mov rax, 60                 ; syscall number 60 = sys_exit
    xor rdi, rdi                ; exit code = 0
    syscall
```

### 10.3 ตารางเปรียบเทียบครบถ้วน

```
┌────────────────────┬────────────┬───────────┬────────────┬────────────┐
│ Feature            │ x86 32-bit │ x86-64    │ ARM 32-bit │ AArch64    │
├────────────────────┼────────────┼───────────┼────────────┼────────────┤
│ Assembler          │ NASM       │ NASM      │ GAS        │ GAS        │
│ File extension     │ .asm/.s    │ .asm/.s   │ .s         │ .s         │
│ Comment char       │ ;          │ ;         │ @          │ //         │
│ Data section       │ section .data │ section .data │ .section .data │ .section .data │
│ Code section       │ section .text │ section .text │ .section .text │ .section .text │
│ String             │ db "text"  │ db "text" │ .ascii "text" │ .ascii "text" │
│ write syscall #    │ 4          │ 1         │ 4          │ 64         │
│ exit syscall #     │ 1          │ 60        │ 1          │ 93         │
│ Syscall register   │ eax        │ rax       │ r7         │ x8         │
│ Arg1 register      │ ebx        │ rdi       │ r0         │ x0         │
│ Arg2 register      │ ecx        │ rsi       │ r1         │ x1         │
│ Arg3 register      │ edx        │ rdx       │ r2         │ x2         │
│ Syscall instruction│ int 0x80   │ syscall   │ swi 0      │ svc #0     │
│ Return address     │ stack      │ stack     │ r14 (lr)   │ x30 (lr)   │
│ Call instruction   │ call       │ call      │ bl         │ bl         │
│ Return instruction │ ret        │ ret       │ bx lr      │ ret / br x30│
│ Load immediate     │ mov eax, 5 │ mov rax, 5│ mov r0, #5 │ mov x0, #5 │
│ Add registers      │ add eax, ebx │ add rax, rbx │ add r0, r0, r1 │ add x0, x0, x1 │
│ Memory access      │ [base+off] │ [base+off]│ [base, #off]│ [base, #off] │
│ Condition codes    │ EFLAGS     │ EFLAGS    │ CPSR flags │ PSTATE flags│
│ Conditional exec   │ cmov*      │ cmov*     │ any instr  │ csel/cset  │
└────────────────────┴────────────┴───────────┴────────────┴────────────┘
```

### 10.4 ARM-Specific Features ที่ไม่มีใน x86

```asm
@ ============================================================
@ ARM Features ที่น่าสนใจ
@ ============================================================

@ 1. Conditional Execution (ARM 32-bit unique feature)
@ ทุก instruction สามารถเพิ่ม suffix เพื่อทำแบบ conditional
    cmp r0, #10                 @ เปรียบเทียบ r0 กับ 10
    movgt r1, #1                @ ถ้า r0 > 10: r1 = 1 (Greater Than)
    movle r1, #0                @ ถ้า r0 <= 10: r1 = 0 (Less or Equal)

@ 2. Flexible Second Operand (barrel shifter)
    mov r0, r1, lsl #4          @ r0 = r1 << 4 (shift ใน instruction เดียว!)
    add r0, r1, r2, asr #2      @ r0 = r1 + (r2 >> 2) arithmetic shift

@ 3. Multiple Register Load/Store
    ldmia r0!, {r1-r4}          @ Load Multiple Increment After: โหลด 4 registers
    stmdb sp!, {r4-r11, lr}     @ Store Multiple Decrement Before: PUSH หลาย registers

@ 4. LDR/STR ที่มีหลาย addressing modes
    ldr r0, [r1]                @ load จาก address ใน r1
    ldr r0, [r1, #4]            @ load จาก r1+4
    ldr r0, [r1, #4]!           @ load จาก r1+4, แล้ว r1 = r1+4 (pre-index)
    ldr r0, [r1], #4            @ load จาก r1, แล้ว r1 = r1+4 (post-index)
    ldr r0, [r1, r2]            @ load จาก r1+r2
    ldr r0, [r1, r2, lsl #2]    @ load จาก r1 + (r2<<2)

@ 5. PC-relative addressing ง่ายกว่า
    ldr r0, =my_var             @ โหลด absolute address (GAS จัดการ literal pool)
    adr r0, my_label            @ PC-relative address (range จำกัด)
    adrl r0, far_label          @ PC-relative สำหรับ range ไกลกว่า
```

---

## 11. ARM Advanced Topics

### 11.1 Thumb และ Thumb-2 Instruction Set

```asm
@ ============================================================
@ Thumb mode: 16-bit instructions สำหรับ code density
@ ============================================================

    .thumb                          @ เปลี่ยนเป็น Thumb mode

    @ Thumb instructions เป็น 16-bit ทำให้ code เล็กลง
    @ แต่มีข้อจำกัดมากกว่า ARM mode

    @ Thumb 16-bit:
    movs r0, #5                     @ ตั้งค่า r0 = 5 (และ set flags)
    adds r0, r0, r1                 @ r0 += r1 (set flags)
    
    @ Thumb-2 เพิ่ม 32-bit instructions:
    .syntax unified                  @ ใช้ UAL syntax
    mov.w r0, #0x12345678           @ 32-bit immediate
    
    @ Interworking: เปลี่ยนระหว่าง ARM/Thumb mode
    bx lr                           @ ถ้า LSB ของ lr = 1 จะเปลี่ยนเป็น Thumb

@ สลับ mode ด้วย BLX
my_thumb_func:
    .thumb_func                     @ บอก assembler ว่านี่คือ Thumb function
    push {r4, lr}
    @ ... Thumb code ...
    pop {r4, pc}                    @ return
```

### 11.2 NEON SIMD สำหรับ Performance

```asm
@ ============================================================
@ NEON: ARM SIMD extensions
@ ============================================================

    .fpu neon                       @ เปิดใช้ NEON

@ NEON registers:
@ d0-d31: 64-bit double-word registers
@ q0-q15: 128-bit quad-word registers (q0 = d0+d1, q1 = d2+d3, ...)

@ ตัวอย่าง: คูณ vector 4 floats พร้อมกัน
vector_multiply:
    @ สมมุติ r0 = pointer ไปยัง float array a
    @ r1 = pointer ไปยัง float array b
    @ r2 = pointer สำหรับเก็บผลลัพธ์
    
    vld1.32 {q0}, [r0]!             @ โหลด 4 floats จาก a (และเลื่อน r0)
    vld1.32 {q1}, [r1]!             @ โหลด 4 floats จาก b (และเลื่อน r1)
    vmul.f32 q2, q0, q1             @ คูณทีละ 4 floats: q2 = q0 * q1
    vst1.32 {q2}, [r2]!             @ เก็บผลลัพธ์ 4 floats
    bx lr
    
@ ตัวอย่าง: NEON RGB to Grayscale
rgb_to_gray_neon:
    @ r0 = src RGB data, r1 = dst gray data, r2 = pixel count
    
    @ Grayscale formula: Y = 0.299*R + 0.587*G + 0.114*B
    @ ใช้ integer approximation: Y = (77*R + 150*G + 29*B) >> 8
    
    vmov.i16 d6, #77                @ d6 = {77, 77, 77, 77} (R weight)
    vmov.i16 d7, #150               @ d7 = {150, 150, 150, 150} (G weight)
    vmov.i16 d8, #29                @ d8 = {29, 29, 29, 29} (B weight)

1:
    vld3.8 {d0, d1, d2}, [r0]!      @ interleaved load: d0=R, d1=G, d2=B (8 pixels)
    
    vmovl.u8 q5, d0                  @ expand R to 16-bit
    vmovl.u8 q6, d1                  @ expand G to 16-bit
    vmovl.u8 q7, d2                  @ expand B to 16-bit
    
    vmul.i16 q5, q5, q3              @ R * 77
    vmla.i16 q5, q6, q4              @ += G * 150
    vmla.i16 q5, q7, q5              @ += B * 29
    
    vshrn.i16 d0, q5, #8             @ >> 8 (แล้วลดขนาดกลับเป็น 8-bit)
    vst1.8 {d0}, [r1]!               @ เก็บผลลัพธ์ 8 pixels
    
    subs r2, r2, #8                  @ ลด counter
    bgt 1b                           @ วนซ้ำถ้า > 0
    
    bx lr
```

---

## 12. Cross-compile บน x86 Linux

### 12.1 การติดตั้ง Toolchain

```bash
# สำหรับ Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y \
    # ARM 32-bit Linux (eabi = Embedded ABI)
    gcc-arm-linux-gnueabi \
    g++-arm-linux-gnueabi \
    binutils-arm-linux-gnueabi \
    \
    # ARM 32-bit Linux (eabihf = hardware float)
    gcc-arm-linux-gnueabihf \
    g++-arm-linux-gnueabihf \
    binutils-arm-linux-gnueabihf \
    \
    # AArch64 Linux
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    \
    # ARM bare metal (Cortex-M, etc.)
    gcc-arm-none-eabi \
    binutils-arm-none-eabi \
    \
    # QEMU user-mode emulation
    qemu-user \
    qemu-user-static \
    \
    # QEMU system emulation (full machine)
    qemu-system-arm \
    qemu-system-aarch64

# ตรวจสอบ installation
arm-linux-gnueabi-gcc --version
aarch64-linux-gnu-gcc --version
qemu-arm --version
```

### 12.2 Cross-compile สรุป Commands

```bash
# ============================================================
# ARM 32-bit (soft float)
# ============================================================
arm-linux-gnueabi-as -o prog.o prog.s        # assemble
arm-linux-gnueabi-ld -o prog prog.o           # link
arm-linux-gnueabi-gcc -o prog prog.c          # C via GCC
qemu-arm -L /usr/arm-linux-gnueabi ./prog     # run

# ============================================================
# ARM 32-bit (hard float - สำหรับ Pi 2/3 32-bit)
# ============================================================
arm-linux-gnueabihf-as -o prog.o prog.s
arm-linux-gnueabihf-ld -o prog prog.o
arm-linux-gnueabihf-gcc -o prog prog.c
qemu-arm -L /usr/arm-linux-gnueabihf ./prog

# ============================================================
# AArch64 (64-bit)
# ============================================================
aarch64-linux-gnu-as -o prog.o prog.s
aarch64-linux-gnu-ld -o prog prog.o
aarch64-linux-gnu-gcc -o prog prog.c
qemu-aarch64 -L /usr/aarch64-linux-gnu ./prog

# ============================================================
# ARM None EABI (bare metal)
# ============================================================
arm-none-eabi-as -mcpu=cortex-m4 -mthumb -o prog.o prog.s
arm-none-eabi-ld -T linker.ld -o prog.elf prog.o
arm-none-eabi-objcopy -O binary prog.elf prog.bin

# ============================================================
# objdump flags สำหรับ ARM
# ============================================================
arm-linux-gnueabi-objdump -d prog          # disassembly (ARM mode)
arm-linux-gnueabi-objdump -M force-thumb -d prog  # disassembly (Thumb mode)
arm-linux-gnueabi-objdump -D prog          # disassembly all sections
arm-linux-gnueabi-objdump -s prog          # full hex dump
arm-linux-gnueabi-readelf -a prog          # full ELF info

# ============================================================
# nm (symbol table)
# ============================================================
arm-linux-gnueabi-nm prog                  # list symbols
arm-linux-gnueabi-nm -C prog               # demangled C++ names

# ============================================================
# size (section sizes)
# ============================================================
arm-linux-gnueabi-size prog                # text/data/bss sizes
```

### 12.3 Makefile สำหรับ ARM Projects

```makefile
# ไฟล์: Makefile
# Makefile สำหรับ ARM Assembly projects

# ARM 32-bit tools
ARM_PREFIX  = arm-linux-gnueabi
ARM_AS      = $(ARM_PREFIX)-as
ARM_LD      = $(ARM_PREFIX)-ld
ARM_OBJDUMP = $(ARM_PREFIX)-objdump
ARM_GDB     = $(ARM_PREFIX)-gdb
QEMU_ARM    = qemu-arm
QEMU_FLAGS  = -L /usr/arm-linux-gnueabi

# AArch64 tools
A64_PREFIX  = aarch64-linux-gnu
A64_AS      = $(A64_PREFIX)-as
A64_LD      = $(A64_PREFIX)-ld
QEMU_A64    = qemu-aarch64
QEMU_A64_FLAGS = -L /usr/aarch64-linux-gnu

# Source files
ARM_SRCS    = $(wildcard *_arm32.s)
A64_SRCS    = $(wildcard *_aarch64.s)
ARM_BINS    = $(ARM_SRCS:.s=)
A64_BINS    = $(A64_SRCS:.s=)

.PHONY: all arm64 clean run_arm run_64

all: arm 64

arm: $(ARM_BINS)
64: $(A64_BINS)

# Pattern rules
%_arm32: %_arm32.o
	$(ARM_LD) -o $@ $<

%_arm32.o: %_arm32.s
	$(ARM_AS) -g -o $@ $<

%_aarch64: %_aarch64.o
	$(A64_LD) -o $@ $<

%_aarch64.o: %_aarch64.s
	$(A64_AS) -g -o $@ $<

run_arm: $(ARM_BINS)
	@for bin in $(ARM_BINS); do \
		echo "Running $$bin:"; \
		$(QEMU_ARM) $(QEMU_FLAGS) ./$$bin; \
	done

run_64: $(A64_BINS)
	@for bin in $(A64_BINS); do \
		echo "Running $$bin:"; \
		$(QEMU_A64) $(QEMU_A64_FLAGS) ./$$bin; \
	done

disasm: $(ARM_BINS)
	@for bin in $(ARM_BINS); do \
		echo "=== Disassembly of $$bin ==="; \
		$(ARM_OBJDUMP) -d $$bin; \
	done

clean:
	rm -f *.o $(ARM_BINS) $(A64_BINS)
```

---

## 13. ข้อผิดพลาดที่พบบ่อยและวิธีแก้

### 13.1 Common Mistakes

```
┌────────────────────────────────────────────────────────────────┐
│                  ข้อผิดพลาดที่พบบ่อยใน ARM Assembly            │
├────────────────────┬───────────────────────┬───────────────────┤
│ ข้อผิดพลาด         │ สาเหตุ                │ วิธีแก้           │
├────────────────────┼───────────────────────┼───────────────────┤
│ ลืม # ใน immediate │ mov r0, 5 (ผิด)       │ mov r0, #5        │
│ ใช้ swi บน 64-bit  │ swi ไม่ work ใน AArch64│ ใช้ svc #0        │
│ syscall number ผิด │ ใช้ x86 numbers       │ ดู ARM syscall table│
│ Stack misalign     │ ไม่ align 8-byte      │ sub sp, sp, #8    │
│ ลืม bx lr         │ ฟังก์ชันไม่ return     │ bx lr ทุก function│
│ Thumb/ARM mode     │ interworking ผิด      │ ใช้ .thumb_func    │
│ Literal pool       │ LDR ไม่ได้ค่า         │ ใช้ =label syntax  │
│ Byte order         │ little vs big endian  │ ตรวจสอบ target     │
└────────────────────┴───────────────────────┴───────────────────┘
```

### 13.2 ตัวอย่าง Error และ Fix

```asm
@ ============================================================
@ ตัวอย่างข้อผิดพลาดที่พบบ่อย
@ ============================================================

@ ❌ ผิด: ลืม # สำหรับ immediate
    mov r0, 5               @ ERROR: 5 ถูกตีความเป็น register address

@ ✅ ถูก: ใช้ # สำหรับ immediate  
    mov r0, #5              @ r0 = 5

@ -------------------------

@ ❌ ผิด: immediate ใหญ่เกินไป
    mov r0, #0x12345678     @ ERROR: ARM immediate จำกัดขนาด
    
@ ✅ ถูก: ใช้ LDR pseudo-instruction
    ldr r0, =0x12345678     @ assembler จะสร้าง literal pool

@ -------------------------

@ ❌ ผิด: ลืม save/restore LR ใน non-leaf function
bad_function:
    bl sub_function         @ bl จะ overwrite lr!
    bx lr                   @ lr ถูก sub_function ทำลายแล้ว

@ ✅ ถูก: save LR ก่อน call
good_function:
    push {lr}               @ save lr ก่อน
    bl sub_function
    pop {pc}                @ restore และ return

@ -------------------------

@ ❌ ผิด: Stack ไม่ aligned
    sub sp, sp, #4          @ allocate 4 bytes (alignment issue)
    
@ ✅ ถูก: Stack ต้อง 8-byte aligned เสมอ
    sub sp, sp, #8          @ allocate 8 bytes (4 bytes ใช้จริง, 4 bytes padding)

@ -------------------------

@ ❌ ผิด: ใช้ swi บน AArch64
    mov x8, #64             
    swi 0                   @ ERROR: ใช้ SWI บน AArch64 ไม่ได้

@ ✅ ถูก: ใช้ svc บน AArch64
    mov x8, #64
    svc #0                  @ AArch64 ใช้ svc
```

---

## 14. แบบฝึกหัด (Exercises)

### Exercise 1: Hello World หลายภาษา

```
เป้าหมาย: เขียนโปรแกรมแสดงข้อความ "สวัสดี ARM!" ในภาษา UTF-8

ขั้นตอน:
1. แปลงข้อความ "สวัสดี ARM!\n" เป็น UTF-8 hex bytes
   ตอบ: Thai characters ใช้ 3 bytes ต่อ character ใน UTF-8
   
2. สร้างไฟล์ hello_thai.s
3. ประกาศ message ด้วย .byte directive สำหรับ UTF-8
4. Compile และ run ด้วย QEMU
5. ตรวจสอบว่าแสดงผลถูกต้อง

Template:
    .section .data
message:
    .byte 0xE0, 0xB8, 0xAA  @ ส
    .byte 0xE0, 0xB8, 0xA7  @ ว
    .byte 0xE0, 0xB8, 0xB2  @ า
    .byte 0xE0, 0xB8, 0xAA  @ ส
    .byte 0xE0, 0xB8, 0x94  @ ด
    .byte 0xE0, 0xB8, 0xB5  @ ี
    .byte ' ', 'A', 'R', 'M', '!', '\n'
message_len = . - message
```

### Exercise 2: โปรแกรม Calculator

```asm
@ Exercise 2: เขียน ARM Calculator ที่รับ input และคำนวณ

@ เขียนฟังก์ชันต่อไปนี้:
@ 1. read_number(fd) -> r0: อ่านตัวเลขจาก stdin
@ 2. print_number(r0): แสดงตัวเลข
@ 3. main: รับ 2 ตัวเลข แสดง sum, difference, product

@ Hint: ใช้ sys_read (syscall #3) อ่าน 1 byte แล้ว convert ASCII to int
@ สำหรับ '0'-'9': int_val = ascii_char - '0'

@ Template:
    .section .text
    .global _start

@ อ่านตัวเลข 1 หลักจาก stdin
@ Output: r0 = number (0-9)
read_digit:
    @ TODO: implement
    @ 1. sys_read 1 byte ไปยัง buffer
    @ 2. แปลง ASCII เป็น integer
    bx lr

@ แสดงตัวเลข (0-999)
@ Input: r0 = number
print_number:
    @ TODO: implement
    @ 1. แปลง integer เป็น ASCII digits
    @ 2. sys_write ทีละ digit
    bx lr

_start:
    @ TODO: ใช้ read_digit และ print_number
    @ แสดง: "A + B = C"
```

### Exercise 3: AArch64 String Length

```asm
// Exercise 3: เขียนฟังก์ชัน strlen สำหรับ AArch64

// เขียนฟังก์ชัน:
// size_t my_strlen(const char* str)
// Input: x0 = pointer ไปยัง null-terminated string
// Output: x0 = ความยาว string (ไม่รวม null)

    .section .text
    .global my_strlen

my_strlen:
    // TODO: implement
    // Hint: 
    // 1. save pointer เริ่มต้น ไว้ใน x1
    // 2. วนลูป: โหลด byte ด้วย ldrb, check null
    // 3. return: x0 = current_ptr - start_ptr
    
    mov x1, x0          // save start pointer
loop:
    ldrb w2, [x0], #1   // load byte, increment pointer
    cbnz w2, loop        // ถ้าไม่ใช่ null วนต่อ
    sub x0, x0, x1       // คำนวณความยาว
    sub x0, x0, #1       // ลบ 1 (เพราะเกิน 1 ไปแล้ว)
    ret

// Test program:
    .global _start
_start:
    adr x0, test_str
    bl my_strlen
    // x0 should = 11 (length of "Hello World")
    
    mov x8, #93
    svc #0

    .section .data
test_str: .asciz "Hello World"
```

### Exercise 4: Raspberry Pi LED Blink (Bare Metal)

```asm
@ Exercise 4: เขียนโปรแกรม Blink LED บน Raspberry Pi (bare metal)

@ Raspberry Pi 2/3 - GPIO 47 (onboard LED)
@ การ blink LED:
@ 1. ตั้งค่า GPIO 47 เป็น output (FSEL register)
@ 2. Set output HIGH (GPSET register) -> LED켜
@ 3. Delay
@ 4. Set output LOW (GPCLR register) -> LED ดับ
@ 5. Delay
@ 6. วนซ้ำ

@ GPIO registers:
.equ GPIO_BASE,   0x3F200000
.equ GPFSEL4,     0x10        @ Function Select 4 (GPIO 40-49)
.equ GPSET1,      0x20        @ Pin Output Set 1 (GPIO 32-63)
.equ GPCLR1,      0x2C        @ Pin Output Clear 1 (GPIO 32-63)

@ GPIO 47 อยู่ใน FSEL4 bits [23:21]
@ GPIO 47 - 40 = 7, offset = 7*3 = 21 bits

@ TODO: เขียนโปรแกรมนี้ให้สมบูรณ์

    .section .text
    .global _start

_start:
    ldr r4, =GPIO_BASE
    
    @ ตั้งค่า GPIO47 เป็น output: set bits [23:21] = 001
    @ TODO: implement
    
blink_loop:
    @ LED ON: set GPIO47 (bit 15 ใน GPSET1, เพราะ 47-32=15)
    @ TODO: implement
    
    @ Delay
    bl delay
    
    @ LED OFF: clear GPIO47
    @ TODO: implement
    
    @ Delay
    bl delay
    
    b blink_loop

delay:
    @ Simple delay loop
    mov r0, #0x800000       @ ประมาณ 0.5 วินาที บน Pi 2
1:  subs r0, r0, #1
    bne 1b
    bx lr
```

### Exercise 5: ARM Inline Assembly Benchmark

```c
// Exercise 5: เปรียบเทียบ performance ระหว่าง C และ ARM inline assembly

// เขียนและ benchmark ฟังก์ชัน 2 versions:
// 1. C implementation: memcpy ปกติ
// 2. ARM NEON implementation: memcpy ด้วย NEON (16 bytes ต่อครั้ง)

// วัด performance โดย:
// 1. copy 100MB ด้วย C memcpy
// 2. copy 100MB ด้วย NEON memcpy
// 3. แสดงเวลาและ throughput

#include <stdio.h>
#include <string.h>
#include <time.h>
#include <stdlib.h>

// C version
void memcpy_c(void* dst, const void* src, size_t n) {
    memcpy(dst, src, n);
}

// ARM NEON version
void memcpy_neon(void* dst, const void* src, size_t n) {
    // TODO: implement ด้วย NEON instructions
    // Hint: ใช้ vld1.8 / vst1.8 เพื่อ copy 16 bytes ต่อครั้ง
    // asm volatile(
    //     "1:\n\t"
    //     "vld1.8 {q0}, [%1]!\n\t"   // โหลด 16 bytes
    //     "vst1.8 {q0}, [%0]!\n\t"   // เก็บ 16 bytes
    //     "subs %2, %2, #16\n\t"     // ลด counter
    //     "bgt 1b"                    // วนซ้ำ
    //     : "+r"(dst), "+r"(src), "+r"(n)
    //     :
    //     : "q0", "memory"
    // );
}

double benchmark(void (*func)(void*, const void*, size_t), 
                 void* dst, const void* src, size_t n, int iterations) {
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    for (int i = 0; i < iterations; i++) {
        func(dst, src, n);
    }
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) + 
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    return elapsed;
}

int main() {
    size_t size = 1024 * 1024;  // 1MB
    int iter = 100;
    
    char* src = malloc(size);
    char* dst = malloc(size);
    
    // เติมข้อมูล
    for (size_t i = 0; i < size; i++) src[i] = i & 0xFF;
    
    double t_c    = benchmark(memcpy_c,    dst, src, size, iter);
    double t_neon = benchmark(memcpy_neon, dst, src, size, iter);
    
    double gb_c    = (size * iter) / t_c    / 1e9;
    double gb_neon = (size * iter) / t_neon / 1e9;
    
    printf("C memcpy:    %.3f GB/s\n", gb_c);
    printf("NEON memcpy: %.3f GB/s\n", gb_neon);
    printf("Speedup:     %.2fx\n", gb_neon / gb_c);
    
    free(src);
    free(dst);
    return 0;
}
```

---

## 15. สรุปและ Key Takeaways

### 15.1 สิ่งที่เรียนรู้ใน Part นี้

```
✅ ARM Assembly ใช้ GAS (GNU Assembler) เป็น assembler มาตรฐาน
✅ ARM 32-bit (ARMv7): ใช้ r0-r15, syscall ผ่าน swi 0, r7 = syscall number
✅ AArch64 (ARMv8): ใช้ x0-x30, syscall ผ่าน svc #0, x8 = syscall number
✅ Syscall numbers แตกต่างกัน: ARM32 write=4, AArch64 write=64
✅ ARM มี Features พิเศษ: conditional execution, barrel shifter, LDM/STM
✅ Cross-compile บน x86 ด้วย arm-linux-gnueabi-* tools
✅ Test ด้วย QEMU: qemu-arm, qemu-aarch64
✅ Bare metal ต้องจัดการ hardware โดยตรง (UART, GPIO)
✅ NEON SIMD ช่วยเพิ่ม performance ด้วยการทำงาน parallel
✅ Inline Assembly ใน C ช่วยใช้ ARM instructions พิเศษ
```

### 15.2 Syscall Reference Table สำคัญ

```
ARM Linux 32-bit Syscall Numbers:
┌─────────┬─────┬─────────────────────────────────────────────┐
│ Syscall │ Num │ Prototype                                   │
├─────────┼─────┼─────────────────────────────────────────────┤
│ exit    │ 1   │ void exit(int status)                       │
│ fork    │ 2   │ pid_t fork(void)                            │
│ read    │ 3   │ ssize_t read(int fd, void* buf, size_t n)   │
│ write   │ 4   │ ssize_t write(int fd, void* buf, size_t n)  │
│ open    │ 5   │ int open(const char* path, int flags, ...)  │
│ close   │ 6   │ int close(int fd)                           │
│ getpid  │ 20  │ pid_t getpid(void)                          │
│ brk     │ 45  │ int brk(void* addr)                         │
└─────────┴─────┴─────────────────────────────────────────────┘

AArch64 Linux Syscall Numbers:
┌─────────┬─────┬─────────────────────────────────────────────┐
│ Syscall │ Num │ Prototype                                   │
├─────────┼─────┼─────────────────────────────────────────────┤
│ read    │ 63  │ ssize_t read(int fd, void* buf, size_t n)   │
│ write   │ 64  │ ssize_t write(int fd, void* buf, size_t n)  │
│ open    │ 56  │ int open(const char* path, int flags, ...)  │
│ close   │ 57  │ int close(int fd)                           │
│ exit    │ 93  │ void exit(int status)                       │
│ getpid  │ 172 │ pid_t getpid(void)                          │
│ brk     │ 214 │ int brk(void* addr)                         │
└─────────┴─────┴─────────────────────────────────────────────┘
```

---

## 16. References และ Resources

### 16.1 ARM Official Documentation

- **ARM Architecture Reference Manual (ARM ARM)**
  - ARMv7-A/R: https://developer.arm.com/documentation/ddi0406/
  - ARMv8-A (AArch64): https://developer.arm.com/documentation/ddi0487/
  
- **ARM Cortex-M Programming Guide**
  - https://developer.arm.com/documentation/dui0553/

- **AAPCS (ARM ABI)**
  - https://github.com/ARM-software/abi-aa/blob/main/aapcs32/aapcs32.rst

### 16.2 Tools and References

- **GAS (GNU Assembler) Manual**
  - https://sourceware.org/binutils/docs/as/
  
- **ARM Linux Syscall Table**
  - ARM 32-bit: https://syscalls.mebeim.net/?table=arm/32
  - AArch64: https://syscalls.mebeim.net/?table=arm64/64
  
- **QEMU Documentation**
  - https://www.qemu.org/docs/master/

### 16.3 หนังสือแนะนำ

- "ARM Assembly Language: Fundamentals and Techniques" - William Hohl
- "The Definitive Guide to ARM Cortex-M3 and Cortex-M4 Processors" - Joseph Yiu
- "Computer Organization and Architecture" - Stallings
- "Programming with 64-Bit ARM Assembly Language" - Stephen Smith

### 16.4 Online Resources

- **ARM Developer** (arm.com/developer) - Official ARM resources
- **Compiler Explorer** (godbolt.org) - ดู assembly output จาก C code
- **ARM Architecture blog** - https://community.arm.com/
- **Raspberry Pi Documentation** - https://www.raspberrypi.com/documentation/

---

## ในส่วนถัดไป (Part 010)

ใน Part 010 เราจะเรียนรู้:
- **ARM Data Operations**: การคำนวณและ bit manipulation
- **ARM Memory Operations**: การจัดการ memory อย่างละเอียด
- **ARM Branch Instructions**: conditional branches และ loops
- **ARM Barrel Shifter**: การใช้ shifter operations อย่างมีประสิทธิภาพ
- **ARMv8 Advanced Features**: หน้าที่พิเศษของ AArch64

---

**Part 009 สิ้นสุดแล้ว**  
*ยินดีด้วย! คุณเขียน ARM Assembly ได้แล้ว สำหรับทั้ง 32-bit และ 64-bit!*

```
สถิติ Part 009:
- Topics covered: 16
- Code examples: 15+
- Lines of ARM assembly: ~500
- Lines of C with inline asm: ~150
- Exercises: 5
- Total content: ~900+ lines
```
