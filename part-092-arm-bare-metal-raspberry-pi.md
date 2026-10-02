# Part 092: ARM Bare Metal Programming บน Raspberry Pi 3/4

## บทนำ

Bare metal programming หมายถึงการเขียนโปรแกรมที่ทำงานโดยตรงบน hardware โดยไม่มี operating system คั่นกลาง เป็นทักษะที่จำเป็นสำหรับ embedded systems developer, OS developer และผู้ที่ต้องการเข้าใจ hardware อย่างลึกซึ้ง

Raspberry Pi เป็น platform ที่ดีมากสำหรับการเรียนรู้ bare metal เพราะ:
- มี hardware documentation ที่ดี (BCM2837/BCM2711 datasheet)
- ใช้ ARM architecture ที่แพร่หลาย
- ราคาถูก และหาซื้อได้ง่าย
- Community ขนาดใหญ่

---

## 1. Raspberry Pi Boot Process

### 1.1 ขั้นตอนการ Boot

Raspberry Pi มี boot sequence ที่แตกต่างจาก PC ทั่วไป เนื่องจาก GPU (VideoCore IV/VI) เป็น primary processor ที่ boot ก่อน CPU

```
[Power On]
    │
    ▼
[GPU ROM (First Stage Bootloader)]
    │  - Built-in to BCM2837/2711 chip
    │  - อ่าน SD card ใน FAT32 partition แรก
    │  - โหลด bootcode.bin
    ▼
[bootcode.bin (Second Stage Bootloader)]
    │  - อยู่บน SD card
    │  - เปิดใช้งาน SDRAM
    │  - โหลด start.elf
    ▼
[start.elf (GPU Firmware)]
    │  - อ่าน config.txt
    │  - โหลด kernel image ตามที่กำหนดใน config.txt
    │  - ส่ง control ไปยัง ARM CPU
    ▼
[kernel8.img (Our Code!)]
    │  - เข้าไปที่ address 0x80000
    │  - เราควบคุมทุกอย่างจากจุดนี้
    ▼
[Our Bare Metal Program]
```

### 1.2 Files ที่จำเป็นบน SD Card

```
SD Card (FAT32):
├── bootcode.bin      # Second stage bootloader (RPi3 only, RPi4 built-in)
├── start.elf         # GPU firmware
├── fixup.dat         # Linker file for GPU firmware
├── config.txt        # Configuration file
└── kernel8.img       # Our compiled kernel (AArch64)
```

### 1.3 config.txt Options สำหรับ AArch64

```ini
# config.txt

# เปิดใช้งาน 64-bit mode
arm_64bit=1

# ระบุชื่อ kernel image
kernel=kernel8.img

# ปิด Bluetooth (optional, ให้ UART0 ทำงานที่ GPIO 14/15)
dtoverlay=disable-bt

# กำหนด core clock (optional)
core_freq=250

# Disable rainbow splash screen
disable_splash=1

# Wait for SD card
boot_delay=0
```

**RPi3 vs RPi4 Config:**
```ini
# RPi3 (BCM2837) - UART clock = 3MHz
# RPi4 (BCM2711) - UART clock = 48MHz
# ต้องคำนวณ baud rate divisor ต่างกัน
```

---

## 2. BCM2837/BCM2711 Memory Map

### 2.1 Physical Memory Layout (RPi3 - BCM2837)

```
Physical Address Space:
0x00000000 - 0x3EFFFFFF  : ARM Memory (RAM) ~1GB
0x3F000000 - 0x3FFFFFFF  : BCM2837 Peripherals
    0x3F003000            : System Timer
    0x3F00B000            : Interrupt Controller
    0x3F00B880            : VideoCore Mailbox
    0x3F200000            : GPIO
    0x3F201000            : UART0 (PL011)
    0x3F215000            : UART1 (Mini UART)
    0x3F300000            : External Mass Media Controller (EMMC)
    0x3F980000            : USB
0x40000000 - 0x7FFFFFFF  : VideoCore Memory
```

### 2.2 Physical Memory Layout (RPi4 - BCM2711)

```
Physical Address Space:
0x00000000 - 0xFBFFFFFF  : ARM Memory (RAM up to 4/8GB)
0xFC000000 - 0xFFFFFFFF  : BCM2711 Peripherals
    0xFC003000            : System Timer
    0xFC00B000            : Interrupt Controller
    0xFC00B880            : VideoCore Mailbox
    0xFE200000            : GPIO          ← ต่างจาก RPi3!
    0xFE201000            : UART0 (PL011) ← ต่างจาก RPi3!
    0xFE215000            : UART1 (Mini UART)
```

### 2.3 Defines สำหรับ Peripheral Base

```c
// mmio.h
#ifndef MMIO_H
#define MMIO_H

#include <stdint.h>

// ตรวจสอบ board type จาก mailbox หรือ compile-time define
#ifdef RASPBERRY_PI_4
    #define MMIO_BASE   0xFE000000UL
#else
    #define MMIO_BASE   0x3F000000UL  // RPi3 default
#endif

// GPIO Base
#define GPIO_BASE       (MMIO_BASE + 0x200000)

// UART0 (PL011) Base
#define UART0_BASE      (MMIO_BASE + 0x201000)

// Mailbox Base
#define MBOX_BASE       (MMIO_BASE + 0x00B880)

// System Timer Base
#define SYSTIMER_BASE   (MMIO_BASE + 0x003000)

// Memory-mapped I/O read/write
static inline void mmio_write(uint64_t reg, uint32_t val) {
    *(volatile uint32_t*)reg = val;
}

static inline uint32_t mmio_read(uint64_t reg) {
    return *(volatile uint32_t*)reg;
}

#endif
```

---

## 3. Startup Code (AArch64 Assembly)

### 3.1 Entry Point และการตั้งค่าเบื้องต้น

เมื่อ Raspberry Pi boot kernel8.img จะถูกโหลดที่ physical address **0x80000** และ CPU ทุกตัว (4 cores) จะเริ่มทำงานพร้อมกัน เราต้องให้ cores 1-3 หยุดทำงาน และให้ core 0 ทำหน้าที่หลัก

```asm
// start.S - Entry point สำหรับ AArch64 bare metal

.section ".text.boot"   // Section นี้จะ link ไปที่ address 0x80000
.global _start

_start:
    // อ่าน CPU ID จาก MPIDR_EL1 register
    // MPIDR_EL1[1:0] = CPU number (0-3)
    mrs     x1, mpidr_el1
    and     x1, x1, #3         // เอาแค่ 2 bits ล่าง
    
    // ถ้า CPU ID != 0 ให้ loop ตลอดกาล (park)
    cbnz    x1, cpu_idle
    
    // === CPU 0 เท่านั้นที่ผ่านมาถึงจุดนี้ ===
    
    // ตั้งค่า Stack Pointer
    // Stack เริ่มที่ 0x80000 และ grows downward
    // kernel ถูกโหลดที่ 0x80000 ดังนั้น stack อยู่ก่อนหน้า
    ldr     x1, =_start
    mov     sp, x1              // SP = 0x80000

    // Clear BSS section
    // BSS คือ section ของ global/static variables ที่ไม่มีการกำหนดค่า
    ldr     x1, =__bss_start
    ldr     x2, =__bss_end
    
bss_clear_loop:
    cmp     x1, x2
    b.ge    bss_clear_done
    str     xzr, [x1], #8      // เก็บ 0 และเลื่อน pointer 8 bytes
    b       bss_clear_loop
    
bss_clear_done:
    // กระโดดไปยัง main() ใน C
    bl      main
    
    // ถ้า main() return (ไม่ควรเกิด) ให้ halt
    b       cpu_idle

cpu_idle:
    // ให้ CPU เข้า low-power wait state
    wfe                         // Wait For Event
    b       cpu_idle            // ถ้า wake up ให้ sleep ต่อ

.section ".text"
```

### 3.2 Linker Script

```ld
/* link.ld - Linker script สำหรับ RPi3/4 bare metal */

ENTRY(_start)

SECTIONS
{
    /* เริ่มต้นที่ 0x80000 - ที่ GPU โหลด kernel */
    . = 0x80000;
    
    /* Code section - boot code ต้องมาก่อน */
    .text.boot : {
        *(.text.boot)
    }
    
    /* Code section ทั่วไป */
    .text : {
        *(.text)
        *(.text.*)
    }
    
    /* Read-only data (constants, string literals) */
    .rodata : {
        *(.rodata)
        *(.rodata.*)
    }
    
    /* Initialized data */
    .data : {
        *(.data)
        *(.data.*)
    }
    
    /* BSS section - uninitialized data */
    . = ALIGN(8);
    __bss_start = .;
    .bss : {
        *(.bss)
        *(.bss.*)
        *(COMMON)
    }
    . = ALIGN(8);
    __bss_end = .;
    
    /DISCARD/ : {
        *(.comment)
        *(.gnu*)
        *(.note*)
        *(.eh_frame*)
    }
}
```

### 3.3 Makefile

```makefile
# Makefile สำหรับ RPi3/4 bare metal

# Toolchain
CROSS_COMPILE = aarch64-linux-gnu-
CC      = $(CROSS_COMPILE)gcc
AS      = $(CROSS_COMPILE)as
LD      = $(CROSS_COMPILE)ld
OBJCOPY = $(CROSS_COMPILE)objcopy

# Flags
CFLAGS  = -Wall -O2 -ffreestanding -nostdinc -nostdlib -nostartfiles
CFLAGS += -mcpu=cortex-a53 -mstrict-align
ASFLAGS = -mcpu=cortex-a53
LDFLAGS = -nostdlib -nostartfiles

# Source files
SRCS_C  = $(wildcard src/*.c)
SRCS_S  = $(wildcard src/*.S)
OBJS    = $(SRCS_C:.c=.o) $(SRCS_S:.S=.o)

# Targets
all: kernel8.img

%.o: %.S
	$(CC) $(ASFLAGS) -c $< -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

kernel8.elf: $(OBJS)
	$(LD) $(LDFLAGS) -T link.ld -o $@ $(OBJS)

kernel8.img: kernel8.elf
	$(OBJCOPY) -O binary $< $@

clean:
	rm -f src/*.o kernel8.elf kernel8.img

# สำหรับ QEMU testing
qemu:
	qemu-system-aarch64 \
		-machine raspi3b \
		-cpu cortex-a53 \
		-m 1024 \
		-nographic \
		-serial null \
		-serial mon:stdio \
		-kernel kernel8.img

.PHONY: all clean qemu
```

---

## 4. UART0 (PL011) - Serial Communication

### 4.1 PL011 Register Map

PL011 เป็น UART controller ที่ ARM ออกแบบ ใช้ใน Raspberry Pi สำหรับ UART0

```
UART0 Registers (offset จาก UART0_BASE):
┌─────────┬────────┬─────────────────────────────────────────┐
│ Offset  │ Name   │ Description                             │
├─────────┼────────┼─────────────────────────────────────────┤
│ 0x000   │ DR     │ Data Register (read=RX, write=TX)        │
│ 0x004   │ RSRECR │ Receive Status / Error Clear Register    │
│ 0x018   │ FR     │ Flag Register (busy, full, empty flags)  │
│ 0x020   │ ILPR   │ IrDA Low-Power Counter Register         │
│ 0x024   │ IBRD   │ Integer Baud Rate Divisor               │
│ 0x028   │ FBRD   │ Fractional Baud Rate Divisor            │
│ 0x02C   │ LCRH   │ Line Control Register                   │
│ 0x030   │ CR     │ Control Register                        │
│ 0x034   │ IFLS   │ Interrupt FIFO Level Select             │
│ 0x038   │ IMSC   │ Interrupt Mask Set/Clear                │
│ 0x03C   │ RIS    │ Raw Interrupt Status                    │
│ 0x040   │ MIS    │ Masked Interrupt Status                 │
│ 0x044   │ ICR    │ Interrupt Clear Register                │
│ 0x048   │ DMACR  │ DMA Control Register                   │
└─────────┴────────┴─────────────────────────────────────────┘
```

### 4.2 Flag Register (FR) Bits

```
FR Register (0x018):
Bit 7: TXFE - Transmit FIFO empty
Bit 6: RXFF - Receive FIFO full
Bit 5: TXFF - Transmit FIFO full   ← ต้องรอถ้า set
Bit 4: RXFE - Receive FIFO empty   ← ถ้า set แสดงว่าไม่มีข้อมูล
Bit 3: BUSY - UART busy transmitting
Bit 2: DCD  - Data Carrier Detect
Bit 1: DSR  - Data Set Ready
Bit 0: CTS  - Clear To Send
```

### 4.3 Line Control Register (LCRH) Bits

```
LCRH Register (0x02C):
Bit 7:   SPS  - Stick Parity Select
Bit 6:5: WLEN - Word Length: 00=5bit, 01=6bit, 10=7bit, 11=8bit
Bit 4:   FEN  - Enable FIFOs
Bit 3:   STP2 - Two Stop Bits
Bit 2:   EPS  - Even Parity
Bit 1:   PEN  - Parity Enable
Bit 0:   BRK  - Send Break
```

### 4.4 Control Register (CR) Bits

```
CR Register (0x030):
Bit 15:  CTSEN  - CTS hardware flow control enable
Bit 14:  RTSEN  - RTS hardware flow control enable
Bit 11:  RXE    - Receive enable
Bit 10:  TXE    - Transmit enable
Bit 9:   LBE    - Loopback enable
Bit 0:   UARTEN - UART enable
```

### 4.5 UART Initialization Code

```c
// uart.h
#ifndef UART_H
#define UART_H

#include <stdint.h>

void uart_init(void);
void uart_putc(char c);
char uart_getc(void);
void uart_puts(const char *str);
void uart_hex(uint64_t val);
void uart_dec(int64_t val);

#endif
```

```c
// uart.c
#include "uart.h"
#include "mmio.h"
#include "gpio.h"

// UART0 Register Addresses
#define UART0_DR     (UART0_BASE + 0x00)
#define UART0_RSRECR (UART0_BASE + 0x04)
#define UART0_FR     (UART0_BASE + 0x18)
#define UART0_ILPR   (UART0_BASE + 0x20)
#define UART0_IBRD   (UART0_BASE + 0x24)
#define UART0_FBRD   (UART0_BASE + 0x28)
#define UART0_LCRH   (UART0_BASE + 0x2C)
#define UART0_CR     (UART0_BASE + 0x30)
#define UART0_IFLS   (UART0_BASE + 0x34)
#define UART0_IMSC   (UART0_BASE + 0x38)
#define UART0_RIS    (UART0_BASE + 0x3C)
#define UART0_MIS    (UART0_BASE + 0x40)
#define UART0_ICR    (UART0_BASE + 0x44)
#define UART0_DMACR  (UART0_BASE + 0x48)

// UART FR bits
#define UART_FR_RXFE (1 << 4)   // Receive FIFO empty
#define UART_FR_TXFF (1 << 5)   // Transmit FIFO full
#define UART_FR_BUSY (1 << 3)   // UART busy

void uart_init(void) {
    // Step 1: ปิด UART ก่อน
    mmio_write(UART0_CR, 0x00000000);
    
    // Step 2: ตั้งค่า GPIO pins 14 และ 15 สำหรับ UART
    // GPIO 14 = TXD0, GPIO 15 = RXD0
    // ต้องตั้ง function เป็น ALT0 (Alternative Function 0)
    gpio_set_function(14, GPIO_FUNC_ALT0);
    gpio_set_function(15, GPIO_FUNC_ALT0);
    
    // Step 3: ปิด pull-up/pull-down สำหรับ GPIO 14 และ 15
    gpio_set_pull(14, GPIO_PULL_NONE);
    gpio_set_pull(15, GPIO_PULL_NONE);
    
    // Step 4: Clear pending interrupts
    mmio_write(UART0_ICR, 0x7FF);
    
    // Step 5: คำนวณ Baud Rate Divisor สำหรับ 115200 baud
    //
    // RPi3 (BCM2837): UART clock = 3 MHz (default)
    //   Baud rate divisor = UART_CLK / (16 * baud_rate)
    //                     = 3,000,000 / (16 * 115200)
    //                     = 1.627
    //   IBRD = 1 (integer part)
    //   FBRD = round(0.627 * 64) = round(40.1) = 40
    //
    // RPi4 (BCM2711): UART clock = 48 MHz
    //   Baud rate divisor = 48,000,000 / (16 * 115200)
    //                     = 26.041...
    //   IBRD = 26
    //   FBRD = round(0.041... * 64) = round(2.67) = 3
    //
    // Note: config.txt กำหนด init_uart_clock=3000000 สำหรับ RPi3

#ifdef RASPBERRY_PI_4
    // RPi4: 48MHz UART clock
    mmio_write(UART0_IBRD, 26);
    mmio_write(UART0_FBRD, 3);
#else
    // RPi3: 3MHz UART clock (หรือตามที่กำหนดใน config.txt)
    mmio_write(UART0_IBRD, 1);
    mmio_write(UART0_FBRD, 40);
#endif

    // Step 6: ตั้งค่า 8N1 (8 data bits, No parity, 1 stop bit) + FIFO enable
    // LCRH: WLEN=11 (8-bit), FEN=1 (FIFO enable)
    mmio_write(UART0_LCRH, (0x3 << 5) | (1 << 4));
    //                       ^^^^^^^^   ^^^^^^^^^
    //                       WLEN=8bit  FEN=enable FIFO
    
    // Step 7: Mask all interrupts (เราใช้ polling mode)
    mmio_write(UART0_IMSC, 0x7F2);
    
    // Step 8: Enable UART, TX, RX
    // CR: UARTEN=1, TXE=1, RXE=1
    mmio_write(UART0_CR, (1 << 0) | (1 << 8) | (1 << 9));
    //                    ^^^^^^^^   ^^^^^^^^   ^^^^^^^^
    //                    UARTEN     TXE        RXE
}

void uart_putc(char c) {
    // รอจนกว่า Transmit FIFO จะไม่เต็ม
    while (mmio_read(UART0_FR) & UART_FR_TXFF) {
        // spin wait
        asm volatile("nop");
    }
    
    // ส่ง character
    mmio_write(UART0_DR, (uint32_t)c);
}

char uart_getc(void) {
    // รอจนกว่าจะมีข้อมูลใน Receive FIFO
    while (mmio_read(UART0_FR) & UART_FR_RXFE) {
        // spin wait
        asm volatile("nop");
    }
    
    // อ่าน character (เฉพาะ 8 bits ล่าง)
    return (char)(mmio_read(UART0_DR) & 0xFF);
}

void uart_puts(const char *str) {
    while (*str) {
        // แปลง '\n' เป็น '\r\n' สำหรับ terminal emulator
        if (*str == '\n') {
            uart_putc('\r');
        }
        uart_putc(*str++);
    }
}

// พิมพ์ hexadecimal 64-bit value
void uart_hex(uint64_t val) {
    uart_puts("0x");
    for (int i = 60; i >= 0; i -= 4) {
        uint64_t nibble = (val >> i) & 0xF;
        uart_putc(nibble < 10 ? '0' + nibble : 'A' + nibble - 10);
    }
}

// พิมพ์ decimal signed integer
void uart_dec(int64_t val) {
    if (val < 0) {
        uart_putc('-');
        val = -val;
    }
    
    if (val == 0) {
        uart_putc('0');
        return;
    }
    
    char buf[21];  // max 20 digits for int64 + null
    int i = 0;
    
    while (val > 0) {
        buf[i++] = '0' + (val % 10);
        val /= 10;
    }
    
    // reverse
    for (int j = i - 1; j >= 0; j--) {
        uart_putc(buf[j]);
    }
}
```

---

## 5. GPIO (General Purpose Input/Output)

### 5.1 GPIO Register Map

```
GPIO Registers (offset จาก GPIO_BASE = 0x3F200000 หรือ 0xFE200000):

Function Select Registers (GPFSEL):
  GPFSEL0 (0x000): GPIO 0-9   (3 bits per pin = 30 bits used)
  GPFSEL1 (0x004): GPIO 10-19
  GPFSEL2 (0x008): GPIO 20-29
  GPFSEL3 (0x00C): GPIO 30-39
  GPFSEL4 (0x010): GPIO 40-49
  GPFSEL5 (0x014): GPIO 50-57

  ค่าสำหรับ function select (3 bits):
  000 = Input
  001 = Output
  100 = Alt function 0
  101 = Alt function 1
  110 = Alt function 2
  111 = Alt function 3
  011 = Alt function 4
  010 = Alt function 5

Output Set Registers (GPSET):
  GPSET0  (0x01C): GPIO 0-31
  GPSET1  (0x020): GPIO 32-53
  (write 1 to set, write 0 has no effect)

Output Clear Registers (GPCLR):
  GPCLR0  (0x028): GPIO 0-31
  GPCLR1  (0x02C): GPIO 32-53
  (write 1 to clear, write 0 has no effect)

Pin Level Registers (GPLEV):
  GPLEV0  (0x034): GPIO 0-31
  GPLEV1  (0x038): GPIO 32-53
  (read current pin state)

Pull-Up/Down Registers (RPi3 - BCM2837):
  GPPUD   (0x094): Pull up/down control
    00 = Disable pull up/down
    01 = Enable Pull Down
    10 = Enable Pull Up
  GPPUDCLK0 (0x098): GPIO 0-31
  GPPUDCLK1 (0x09C): GPIO 32-53
  (clock the control signal into selected GPIO)

Pull-Up/Down Registers (RPi4 - BCM2711 - ต่างกัน!):
  GPIO_PUP_PDN_CNTRL_REG0 (0x0E4): GPIO 0-15
  GPIO_PUP_PDN_CNTRL_REG1 (0x0E8): GPIO 16-31
  GPIO_PUP_PDN_CNTRL_REG2 (0x0EC): GPIO 32-47
  GPIO_PUP_PDN_CNTRL_REG3 (0x0F0): GPIO 48-57
  (2 bits per pin: 00=None, 01=Pull Up, 10=Pull Down)
```

### 5.2 GPIO Code

```c
// gpio.h
#ifndef GPIO_H
#define GPIO_H

#include <stdint.h>

// GPIO Function Select values
#define GPIO_FUNC_INPUT  0
#define GPIO_FUNC_OUTPUT 1
#define GPIO_FUNC_ALT0   4
#define GPIO_FUNC_ALT1   5
#define GPIO_FUNC_ALT2   6
#define GPIO_FUNC_ALT3   7
#define GPIO_FUNC_ALT4   3
#define GPIO_FUNC_ALT5   2

// GPIO Pull Up/Down
#define GPIO_PULL_NONE   0
#define GPIO_PULL_UP     1
#define GPIO_PULL_DOWN   2

void gpio_set_function(uint32_t pin, uint32_t func);
void gpio_set_pull(uint32_t pin, uint32_t pull);
void gpio_set(uint32_t pin);
void gpio_clear(uint32_t pin);
int  gpio_get(uint32_t pin);

#endif
```

```c
// gpio.c
#include "gpio.h"
#include "mmio.h"

// GPIO Register Addresses
#define GPFSEL0  (GPIO_BASE + 0x00)
#define GPFSEL1  (GPIO_BASE + 0x04)
#define GPFSEL2  (GPIO_BASE + 0x08)
#define GPFSEL3  (GPIO_BASE + 0x0C)
#define GPFSEL4  (GPIO_BASE + 0x10)
#define GPFSEL5  (GPIO_BASE + 0x14)

#define GPSET0   (GPIO_BASE + 0x1C)
#define GPSET1   (GPIO_BASE + 0x20)

#define GPCLR0   (GPIO_BASE + 0x28)
#define GPCLR1   (GPIO_BASE + 0x2C)

#define GPLEV0   (GPIO_BASE + 0x34)
#define GPLEV1   (GPIO_BASE + 0x38)

// RPi3 Pull up/down
#define GPPUD      (GPIO_BASE + 0x94)
#define GPPUDCLK0  (GPIO_BASE + 0x98)
#define GPPUDCLK1  (GPIO_BASE + 0x9C)

// RPi4 Pull up/down (ต่างกัน!)
#define GPIO_PUP_PDN_CNTRL_REG0 (GPIO_BASE + 0xE4)
#define GPIO_PUP_PDN_CNTRL_REG1 (GPIO_BASE + 0xE8)

void gpio_set_function(uint32_t pin, uint32_t func) {
    // คำนวณ register และ bit position
    // แต่ละ register ควบคุม 10 pins (3 bits per pin)
    uint32_t reg_offset = (pin / 10) * 4;  // 4 bytes per register
    uint32_t bit_offset = (pin % 10) * 3;  // 3 bits per pin
    
    uint64_t reg_addr = GPFSEL0 + reg_offset;
    
    // อ่านค่าปัจจุบัน
    uint32_t val = mmio_read(reg_addr);
    
    // Clear 3 bits สำหรับ pin นี้
    val &= ~(0x7 << bit_offset);
    
    // Set function bits
    val |= (func & 0x7) << bit_offset;
    
    // เขียนกลับ
    mmio_write(reg_addr, val);
}

// Simple delay function (ใช้ loop)
static void delay_cycles(uint32_t cycles) {
    for (uint32_t i = 0; i < cycles; i++) {
        asm volatile("nop");
    }
}

void gpio_set_pull(uint32_t pin, uint32_t pull) {
#ifdef RASPBERRY_PI_4
    // RPi4: ใช้ GPIO_PUP_PDN_CNTRL registers (ต่างกันจาก RPi3!)
    uint32_t reg_idx = pin / 16;
    uint32_t bit_offset = (pin % 16) * 2;
    
    uint64_t reg_addr = GPIO_PUP_PDN_CNTRL_REG0 + (reg_idx * 4);
    
    uint32_t val = mmio_read(reg_addr);
    val &= ~(0x3 << bit_offset);
    
    // RPi4 encoding: 00=None, 01=Pull Up, 10=Pull Down
    if (pull == GPIO_PULL_UP) val |= (1 << bit_offset);
    else if (pull == GPIO_PULL_DOWN) val |= (2 << bit_offset);
    
    mmio_write(reg_addr, val);
#else
    // RPi3: ใช้ GPPUD + GPPUDCLK sequence
    // ขั้นตอน (จาก BCM2837 datasheet section 6.1):
    
    // 1. เขียน control signal ลงใน GPPUD
    mmio_write(GPPUD, pull & 0x3);
    
    // 2. รอ 150 cycles เพื่อให้ control signal stable
    delay_cycles(150);
    
    // 3. Clock control signal เข้าไปยัง GPIO ที่ต้องการ
    if (pin < 32) {
        mmio_write(GPPUDCLK0, 1 << pin);
    } else {
        mmio_write(GPPUDCLK1, 1 << (pin - 32));
    }
    
    // 4. รออีก 150 cycles
    delay_cycles(150);
    
    // 5. Clear GPPUD
    mmio_write(GPPUD, 0);
    
    // 6. Clear clock register
    if (pin < 32) {
        mmio_write(GPPUDCLK0, 0);
    } else {
        mmio_write(GPPUDCLK1, 0);
    }
#endif
}

void gpio_set(uint32_t pin) {
    if (pin < 32) {
        mmio_write(GPSET0, 1 << pin);
    } else {
        mmio_write(GPSET1, 1 << (pin - 32));
    }
}

void gpio_clear(uint32_t pin) {
    if (pin < 32) {
        mmio_write(GPCLR0, 1 << pin);
    } else {
        mmio_write(GPCLR1, 1 << (pin - 32));
    }
}

int gpio_get(uint32_t pin) {
    uint32_t val;
    if (pin < 32) {
        val = mmio_read(GPLEV0);
        return (val >> pin) & 1;
    } else {
        val = mmio_read(GPLEV1);
        return (val >> (pin - 32)) & 1;
    }
}
```

---

## 6. LED Blink

### 6.1 LED บน Raspberry Pi

Raspberry Pi 3 มี activity LED อยู่ที่ GPIO **29** (ไม่ใช่ 47 เหมือน RPi2)
Raspberry Pi 4 มี activity LED อยู่ที่ GPIO **42**

แต่เราสามารถต่อ LED ภายนอกได้ที่ GPIO pin ใดก็ได้

```
GPIO 16 (Pin 36 บน header) - External LED
                   ┌───────────────┐
GPIO 16 ────────── │ 330Ω resistor │──── LED ────── GND
                   └───────────────┘
```

### 6.2 LED Blink Code

```c
// led.h
#ifndef LED_H
#define LED_H

#define LED_GPIO 16   // ใช้ GPIO 16 สำหรับ external LED

void led_init(void);
void led_on(void);
void led_off(void);
void led_blink(uint32_t times, uint32_t delay_ms);

#endif
```

```c
// led.c
#include "led.h"
#include "gpio.h"
#include "timer.h"

void led_init(void) {
    // ตั้งค่า GPIO 16 เป็น output
    gpio_set_function(LED_GPIO, GPIO_FUNC_OUTPUT);
    gpio_clear(LED_GPIO);  // เริ่มต้นดับ LED
}

void led_on(void) {
    gpio_set(LED_GPIO);
}

void led_off(void) {
    gpio_clear(LED_GPIO);
}

void led_blink(uint32_t times, uint32_t delay_ms) {
    for (uint32_t i = 0; i < times; i++) {
        led_on();
        timer_wait_ms(delay_ms);
        led_off();
        timer_wait_ms(delay_ms);
    }
}
```

### 6.3 System Timer สำหรับ Delay

```c
// timer.h
#ifndef TIMER_H
#define TIMER_H

#include <stdint.h>

void timer_wait_us(uint32_t us);
void timer_wait_ms(uint32_t ms);
uint64_t timer_get_us(void);

#endif
```

```c
// timer.c
#include "timer.h"
#include "mmio.h"

// System Timer Registers (BCM2837/2711)
// System Timer ทำงานที่ 1MHz (1 microsecond per tick)
#define SYSTIMER_CS  (SYSTIMER_BASE + 0x00)  // Control/Status
#define SYSTIMER_CLO (SYSTIMER_BASE + 0x04)  // Counter Lower 32 bits
#define SYSTIMER_CHI (SYSTIMER_BASE + 0x08)  // Counter Higher 32 bits
#define SYSTIMER_C0  (SYSTIMER_BASE + 0x0C)  // Compare 0
#define SYSTIMER_C1  (SYSTIMER_BASE + 0x10)  // Compare 1
#define SYSTIMER_C2  (SYSTIMER_BASE + 0x14)  // Compare 2
#define SYSTIMER_C3  (SYSTIMER_BASE + 0x18)  // Compare 3

uint64_t timer_get_us(void) {
    // อ่าน 64-bit counter
    // ต้องอ่าน CHI ก่อนและหลัง CLO เพื่อ handle overflow
    uint32_t hi, lo;
    
    do {
        hi = mmio_read(SYSTIMER_CHI);
        lo = mmio_read(SYSTIMER_CLO);
    } while (hi != mmio_read(SYSTIMER_CHI));
    // ถ้า hi เปลี่ยนระหว่างอ่าน lo แสดงว่า overflow → อ่านใหม่
    
    return ((uint64_t)hi << 32) | lo;
}

void timer_wait_us(uint32_t us) {
    uint64_t start = timer_get_us();
    while (timer_get_us() - start < us) {
        asm volatile("nop");
    }
}

void timer_wait_ms(uint32_t ms) {
    timer_wait_us(ms * 1000);
}
```

---

## 7. UART printf Implementation

### 7.1 Simple printf

```c
// printf.c
#include "uart.h"
#include <stdarg.h>
#include <stdint.h>

// ขนาด buffer สำหรับ number conversion
#define PRINT_BUF_SIZE 64

static void print_unsigned(uint64_t val, int base, int uppercase, int width, char pad) {
    static const char digits_lower[] = "0123456789abcdef";
    static const char digits_upper[] = "0123456789ABCDEF";
    const char *digits = uppercase ? digits_upper : digits_lower;
    
    char buf[PRINT_BUF_SIZE];
    int i = 0;
    
    if (val == 0) {
        buf[i++] = '0';
    } else {
        while (val > 0) {
            buf[i++] = digits[val % base];
            val /= base;
        }
    }
    
    // Padding
    while (i < width) {
        buf[i++] = pad;
    }
    
    // Print in reverse
    while (i > 0) {
        uart_putc(buf[--i]);
    }
}

void mini_printf(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    
    while (*fmt) {
        if (*fmt != '%') {
            uart_putc(*fmt++);
            continue;
        }
        
        fmt++;  // skip '%'
        
        // Parse width and padding
        char pad = ' ';
        int width = 0;
        
        if (*fmt == '0') {
            pad = '0';
            fmt++;
        }
        
        while (*fmt >= '0' && *fmt <= '9') {
            width = width * 10 + (*fmt - '0');
            fmt++;
        }
        
        // Parse long modifier
        int is_long = 0;
        if (*fmt == 'l') {
            is_long = 1;
            fmt++;
            if (*fmt == 'l') {
                fmt++;  // 'll' modifier
            }
        }
        
        switch (*fmt) {
            case 'd':
            case 'i': {
                int64_t val = is_long ? va_arg(args, int64_t) : va_arg(args, int32_t);
                if (val < 0) {
                    uart_putc('-');
                    val = -val;
                    if (width > 0) width--;
                }
                print_unsigned((uint64_t)val, 10, 0, width, pad);
                break;
            }
            case 'u': {
                uint64_t val = is_long ? va_arg(args, uint64_t) : va_arg(args, uint32_t);
                print_unsigned(val, 10, 0, width, pad);
                break;
            }
            case 'x': {
                uint64_t val = is_long ? va_arg(args, uint64_t) : va_arg(args, uint32_t);
                print_unsigned(val, 16, 0, width, pad);
                break;
            }
            case 'X': {
                uint64_t val = is_long ? va_arg(args, uint64_t) : va_arg(args, uint32_t);
                print_unsigned(val, 16, 1, width, pad);
                break;
            }
            case 'p': {
                uint64_t val = (uint64_t)va_arg(args, void*);
                uart_puts("0x");
                print_unsigned(val, 16, 0, 16, '0');
                break;
            }
            case 'c': {
                char c = (char)va_arg(args, int);
                uart_putc(c);
                break;
            }
            case 's': {
                const char *s = va_arg(args, const char*);
                if (!s) s = "(null)";
                uart_puts(s);
                break;
            }
            case '%':
                uart_putc('%');
                break;
            default:
                uart_putc('%');
                uart_putc(*fmt);
                break;
        }
        
        fmt++;
    }
    
    va_end(args);
}
```

---

## 8. VideoCore Mailbox Interface

### 8.1 Mailbox Overview

Mailbox เป็น interface สำหรับ communication ระหว่าง ARM CPU และ VideoCore GPU ใช้สำหรับ:
- ขอข้อมูล board (serial number, revision, memory)
- ตั้งค่า framebuffer สำหรับการแสดงผล
- ควบคุม power management
- ดึง clock frequencies

### 8.2 Mailbox Registers

```
Mailbox Registers (offset จาก MBOX_BASE = 0x3F00B880):
┌──────────┬──────────┬─────────────────────────────────┐
│ Offset   │ Name     │ Description                     │
├──────────┼──────────┼─────────────────────────────────┤
│ 0x00     │ READ     │ Mailbox read (channel 0-3)       │
│ 0x10     │ POLL     │ Read without consuming           │
│ 0x14     │ SENDER   │ Sender ID                       │
│ 0x18     │ STATUS   │ Status flags                    │
│ 0x1C     │ CONFIG   │ Configuration                   │
│ 0x20     │ WRITE    │ Mailbox write                   │
└──────────┴──────────┴─────────────────────────────────┘

Mailbox STATUS bits:
  Bit 31: FULL  - mailbox is full (cannot write)
  Bit 30: EMPTY - mailbox is empty (nothing to read)

Mailbox Message Format (lower 4 bits = channel):
  [31:4] = address >> 4 (message buffer address, 16-byte aligned)
  [3:0]  = channel number

Channel 8: Property tags interface (ที่ใช้บ่อยที่สุด)
```

### 8.3 Property Tags Interface

```
Property Tags Message Buffer Layout:
┌─────────────────────────────────────────────┐
│ Offset 0:  Buffer Size (bytes, total)        │
│ Offset 4:  Request/Response Code             │
│             0x00000000 = Request             │
│             0x80000000 = Response success    │
│             0x80000001 = Response error      │
│                                             │
│ [Tags...]                                   │
│                                             │
│ Tag Format:                                 │
│   Offset 0: Tag ID                          │
│   Offset 4: Value Buffer Size (bytes)       │
│   Offset 8: Request/Response Indicator      │
│              Bit 31: 0=Request, 1=Response  │
│              Bits 30:0 = value length       │
│   Offset 12+: Value bytes                  │
│                                             │
│ End Tag: 0x00000000                        │
└─────────────────────────────────────────────┘
```

### 8.4 Mailbox Code

```c
// mailbox.h
#ifndef MAILBOX_H
#define MAILBOX_H

#include <stdint.h>

// Property tag IDs
#define MBOX_TAG_GETSERIAL          0x00010004
#define MBOX_TAG_GETBOARDREVISION   0x00010002
#define MBOX_TAG_GETARMMEMORY       0x00010005
#define MBOX_TAG_GETVCMEMORY        0x00010006
#define MBOX_TAG_SETCLKRATE         0x00038002
#define MBOX_TAG_GETCLKRATE         0x00030002
#define MBOX_TAG_SETPOWER           0x00028001
#define MBOX_TAG_ALLOCBUFFER        0x00040001
#define MBOX_TAG_GETPHYSDISPLAY     0x00040003
#define MBOX_TAG_SETVIRTDISPLAY     0x00048004
#define MBOX_TAG_SETDEPTH           0x00048005
#define MBOX_TAG_SETPIXELORDER      0x00048006
#define MBOX_TAG_GETVIRTOFFSET      0x00040009
#define MBOX_TAG_SETVIRTOFFSET      0x00048009
#define MBOX_TAG_LAST               0x00000000

#define MBOX_CHANNEL_PROP           8

// Message buffer ต้อง 16-byte aligned
// ใช้ __attribute__((aligned(16)))
extern volatile uint32_t mbox_buf[36] __attribute__((aligned(16)));

int  mbox_call(uint8_t channel);
int  mbox_get_board_serial(uint64_t *serial);
int  mbox_get_arm_memory(uint32_t *base, uint32_t *size);
int  mbox_get_board_revision(uint32_t *rev);

#endif
```

```c
// mailbox.c
#include "mailbox.h"
#include "mmio.h"

// Mailbox Registers
#define MBOX_READ   (MBOX_BASE + 0x00)
#define MBOX_POLL   (MBOX_BASE + 0x10)
#define MBOX_SENDER (MBOX_BASE + 0x14)
#define MBOX_STATUS (MBOX_BASE + 0x18)
#define MBOX_CONFIG (MBOX_BASE + 0x1C)
#define MBOX_WRITE  (MBOX_BASE + 0x20)

#define MBOX_STATUS_FULL  (1U << 31)
#define MBOX_STATUS_EMPTY (1U << 30)

// Message buffer (16-byte aligned!)
volatile uint32_t mbox_buf[36] __attribute__((aligned(16)));

int mbox_call(uint8_t channel) {
    // รวม address กับ channel (ต้องลบ 4 bits เพราะ buffer aligned 16)
    uint32_t msg = ((uint32_t)(uint64_t)mbox_buf & ~0xF) | (channel & 0xF);
    
    // รอจนกว่า mailbox จะว่าง (ไม่เต็ม)
    while (mmio_read(MBOX_STATUS) & MBOX_STATUS_FULL) {
        asm volatile("nop");
    }
    
    // Data memory barrier ก่อนเขียน
    asm volatile("dmb sy" ::: "memory");
    
    // ส่ง message
    mmio_write(MBOX_WRITE, msg);
    
    // รอ response
    while (1) {
        // รอจนกว่าจะมีข้อมูลใน mailbox
        while (mmio_read(MBOX_STATUS) & MBOX_STATUS_EMPTY) {
            asm volatile("nop");
        }
        
        // Data memory barrier
        asm volatile("dmb sy" ::: "memory");
        
        // อ่าน response
        uint32_t resp = mmio_read(MBOX_READ);
        
        // ตรวจสอบว่า response สำหรับ channel ของเรา
        if ((resp & 0xF) == channel) {
            // ตรวจสอบ success flag (bit 31 ของ buffer[1])
            return mbox_buf[1] == 0x80000000;
        }
    }
}

int mbox_get_board_serial(uint64_t *serial) {
    mbox_buf[0] = 8 * 4;        // buffer size (8 words * 4 bytes)
    mbox_buf[1] = 0;            // request code
    mbox_buf[2] = MBOX_TAG_GETSERIAL;  // tag
    mbox_buf[3] = 8;            // value buffer size (8 bytes = 2 words)
    mbox_buf[4] = 0;            // request indicator
    mbox_buf[5] = 0;            // value[0] (will be filled)
    mbox_buf[6] = 0;            // value[1] (will be filled)
    mbox_buf[7] = MBOX_TAG_LAST;    // end tag
    
    if (!mbox_call(MBOX_CHANNEL_PROP)) return 0;
    
    *serial = ((uint64_t)mbox_buf[6] << 32) | mbox_buf[5];
    return 1;
}

int mbox_get_arm_memory(uint32_t *base, uint32_t *size) {
    mbox_buf[0] = 8 * 4;
    mbox_buf[1] = 0;
    mbox_buf[2] = MBOX_TAG_GETARMMEMORY;
    mbox_buf[3] = 8;
    mbox_buf[4] = 0;
    mbox_buf[5] = 0;    // base address
    mbox_buf[6] = 0;    // size
    mbox_buf[7] = MBOX_TAG_LAST;
    
    if (!mbox_call(MBOX_CHANNEL_PROP)) return 0;
    
    *base = mbox_buf[5];
    *size = mbox_buf[6];
    return 1;
}

int mbox_get_board_revision(uint32_t *rev) {
    mbox_buf[0] = 7 * 4;
    mbox_buf[1] = 0;
    mbox_buf[2] = MBOX_TAG_GETBOARDREVISION;
    mbox_buf[3] = 4;
    mbox_buf[4] = 0;
    mbox_buf[5] = 0;    // revision
    mbox_buf[6] = MBOX_TAG_LAST;
    
    if (!mbox_call(MBOX_CHANNEL_PROP)) return 0;
    
    *rev = mbox_buf[5];
    return 1;
}
```

---

## 9. Framebuffer และ Drawing Pixels

### 9.1 Framebuffer Setup ผ่าน Mailbox

```c
// framebuffer.h
#ifndef FRAMEBUFFER_H
#define FRAMEBUFFER_H

#include <stdint.h>

typedef struct {
    uint32_t *ptr;      // pointer ไปยัง framebuffer
    uint32_t width;     // ความกว้าง (pixels)
    uint32_t height;    // ความสูง (pixels)
    uint32_t pitch;     // bytes per row
    uint32_t bpp;       // bits per pixel
} Framebuffer;

extern Framebuffer fb;

int  fb_init(uint32_t width, uint32_t height);
void fb_draw_pixel(uint32_t x, uint32_t y, uint32_t color);
void fb_clear(uint32_t color);
void fb_draw_rect(uint32_t x, uint32_t y, uint32_t w, uint32_t h, uint32_t color);
void fb_draw_char(uint32_t x, uint32_t y, char c, uint32_t fg, uint32_t bg);
void fb_draw_string(uint32_t x, uint32_t y, const char *str, uint32_t fg, uint32_t bg);

// Color macros (RGB32)
#define RGB(r,g,b) (((r)<<16)|((g)<<8)|(b))
#define COLOR_BLACK   0x000000
#define COLOR_WHITE   0xFFFFFF
#define COLOR_RED     0xFF0000
#define COLOR_GREEN   0x00FF00
#define COLOR_BLUE    0x0000FF
#define COLOR_YELLOW  0xFFFF00

#endif
```

```c
// framebuffer.c
#include "framebuffer.h"
#include "mailbox.h"

Framebuffer fb = {0};

int fb_init(uint32_t width, uint32_t height) {
    // Setup framebuffer ผ่าน mailbox property tags
    // ต้องส่ง tags หลายตัวพร้อมกัน
    
    mbox_buf[0]  = 35 * 4;       // total size
    mbox_buf[1]  = 0;            // request
    
    // Tag: Set Physical Display Size
    mbox_buf[2]  = 0x00048003;   // SETPHYSDISPLAY
    mbox_buf[3]  = 8;            // value size
    mbox_buf[4]  = 8;            // request indicator + size
    mbox_buf[5]  = width;
    mbox_buf[6]  = height;
    
    // Tag: Set Virtual Display Size
    mbox_buf[7]  = 0x00048004;   // SETVIRTDISPLAY
    mbox_buf[8]  = 8;
    mbox_buf[9]  = 8;
    mbox_buf[10] = width;
    mbox_buf[11] = height;
    
    // Tag: Set Bit Depth (32 bpp)
    mbox_buf[12] = 0x00048005;   // SETDEPTH
    mbox_buf[13] = 4;
    mbox_buf[14] = 4;
    mbox_buf[15] = 32;           // 32 bits per pixel
    
    // Tag: Set Pixel Order (RGB)
    mbox_buf[16] = 0x00048006;
    mbox_buf[17] = 4;
    mbox_buf[18] = 4;
    mbox_buf[19] = 1;            // RGB
    
    // Tag: Allocate Buffer (alignment = 4096)
    mbox_buf[20] = 0x00040001;   // ALLOCBUFFER
    mbox_buf[21] = 8;
    mbox_buf[22] = 8;
    mbox_buf[23] = 4096;         // alignment
    mbox_buf[24] = 0;            // size (response)
    
    // Tag: Get Pitch
    mbox_buf[25] = 0x00040008;   // GETPITCH
    mbox_buf[26] = 4;
    mbox_buf[27] = 4;
    mbox_buf[28] = 0;
    
    mbox_buf[29] = MBOX_TAG_LAST;
    
    if (!mbox_call(MBOX_CHANNEL_PROP)) {
        return 0;
    }
    
    if (mbox_buf[23] == 0) {
        return 0;   // buffer allocation failed
    }
    
    // GPU address to ARM address conversion
    // GPU มี bus address ที่ต่างจาก ARM address
    // ต้องลบ 0xC0000000 (VideoCore alias)
    fb.ptr    = (uint32_t*)((uint64_t)(mbox_buf[23] & 0x3FFFFFFF));
    fb.width  = mbox_buf[5];
    fb.height = mbox_buf[6];
    fb.pitch  = mbox_buf[28];
    fb.bpp    = 32;
    
    return 1;
}

void fb_draw_pixel(uint32_t x, uint32_t y, uint32_t color) {
    if (x >= fb.width || y >= fb.height) return;
    
    // คำนวณ address ของ pixel
    uint32_t *pixel = (uint32_t*)((uint8_t*)fb.ptr + y * fb.pitch + x * 4);
    *pixel = color;
}

void fb_clear(uint32_t color) {
    for (uint32_t y = 0; y < fb.height; y++) {
        for (uint32_t x = 0; x < fb.width; x++) {
            fb_draw_pixel(x, y, color);
        }
    }
}

void fb_draw_rect(uint32_t x, uint32_t y, uint32_t w, uint32_t h, uint32_t color) {
    for (uint32_t dy = 0; dy < h; dy++) {
        for (uint32_t dx = 0; dx < w; dx++) {
            fb_draw_pixel(x + dx, y + dy, color);
        }
    }
}

// 8x8 font (simplified - เฉพาะ ASCII 32-127)
// แต่ละ character ใช้ 8 bytes (8 rows x 8 pixels)
static const uint8_t font8x8_basic[128][8] = {
    // ตัวอย่าง A (0x41)
    [0x41] = {
        0b00011000,
        0b00100100,
        0b01000010,
        0b01111110,
        0b01000010,
        0b01000010,
        0b01000010,
        0b00000000
    },
    // ตัวอย่าง B (0x42)
    [0x42] = {
        0b01111100,
        0b01000010,
        0b01000010,
        0b01111100,
        0b01000010,
        0b01000010,
        0b01111100,
        0b00000000
    },
    // ... (ใน code จริงจะมี character ครบทุกตัว)
};

void fb_draw_char(uint32_t x, uint32_t y, char c, uint32_t fg, uint32_t bg) {
    if (c < 32 || c > 127) c = '?';
    
    const uint8_t *glyph = font8x8_basic[(uint8_t)c];
    
    for (int row = 0; row < 8; row++) {
        for (int col = 0; col < 8; col++) {
            uint32_t color = (glyph[row] & (1 << (7-col))) ? fg : bg;
            fb_draw_pixel(x + col, y + row, color);
        }
    }
}

void fb_draw_string(uint32_t x, uint32_t y, const char *str, uint32_t fg, uint32_t bg) {
    uint32_t cx = x;
    while (*str) {
        if (*str == '\n') {
            cx = x;
            y += 8;
        } else {
            fb_draw_char(cx, y, *str, fg, bg);
            cx += 8;
        }
        str++;
    }
}
```

---

## 10. AArch64 MMU Setup (Simple)

### 10.1 MMU ทำงานอย่างไร

```
Virtual Address (64-bit) → MMU → Physical Address

AArch64 Translation Table:
Level 0 (PGD): 512 entries, each covers 512GB
Level 1 (PUD): 512 entries, each covers 1GB
Level 2 (PMD): 512 entries, each covers 2MB
Level 3 (PTE): 512 entries, each covers 4KB

ค่า bit ต่าง ๆ ใน Translation Table Entry:
Bit 0:    Valid bit
Bit 1:    Table/Block descriptor
Bits 11:2: Lower block attributes
  - Bits 4:2: AttrIdx (index into MAIR_EL1)
  - Bit 5: NS (non-secure)
  - Bit 7:6: AP (access permission)
  - Bit 8: SH (shareable)
  - Bit 9: AF (access flag)
  - Bit 10: nG (not global)
Bits 47:12: Output address
Bits 63:52: Upper block attributes
```

### 10.2 Simple Identity Mapping

```asm
// mmu.S - Simple MMU setup สำหรับ identity mapping
// Map virtual address = physical address (1:1)

.section ".text"
.global mmu_init

// Translation tables (16KB aligned สำหรับ 4-level paging)
.section ".bss"
.align 14   // 2^14 = 16384 = 16KB alignment
ttbr0_l1:
    .space 4096   // Level 1 table (512 entries x 8 bytes = 4096 bytes)

.section ".text"
mmu_init:
    // 1. สร้าง Level 1 translation table
    // Map แรก 4GB เป็น 1GB blocks (device memory สำหรับ peripheral area)
    
    ldr x0, =ttbr0_l1
    
    // Entry 0: 0x0000000000 - 0x003FFFFFFF (1GB) = Normal memory
    // Block descriptor: address | AF | SH=Inner | AP=RW | AttrIdx=0 | Valid
    mov x1, #0x0000000000
    orr x1, x1, #(1 << 10)   // AF = Access Flag
    orr x1, x1, #(3 << 8)    // SH = Inner Shareable
    orr x1, x1, #(1 << 0)    // Valid
    // AttrIdx=0 = Normal memory (ต้องตั้งใน MAIR_EL1)
    str x1, [x0, #0]          // Entry 0
    
    // Entry 1: 0x0040000000 - 0x007FFFFFFF (1GB) = Device memory (peripherals)
    mov x1, #0x0040000000
    orr x1, x1, #(1 << 10)   // AF
    orr x1, x1, #(2 << 8)    // SH = Outer Shareable
    orr x1, x1, #(1 << 2)    // AttrIdx=1 = Device nGnRE
    orr x1, x1, #(1 << 0)    // Valid
    str x1, [x0, #8]          // Entry 1
    
    // 2. ตั้งค่า MAIR_EL1 (Memory Attribute Indirection Register)
    // Index 0: Normal memory (WB, RA, WA)
    // Index 1: Device nGnRnE memory
    mov x0, #0xFF             // Attr0: Normal memory (0xFF = WB/RA/WA outer, inner)
    orr x0, x0, #(0x04 << 8) // Attr1: Device nGnRE (0x04)
    msr mair_el1, x0
    
    // 3. ตั้งค่า TCR_EL1 (Translation Control Register)
    // T0SZ=25: ช่วง address ขนาด 2^(64-25) = 2^39 bytes
    // TG0=00: 4KB granule
    // SH0=11: Inner shareable
    // ORGN0=01: Normal memory, WB cacheable
    // IRGN0=01: Normal memory, WB cacheable
    ldr x0, =0x00000000B5593519
    msr tcr_el1, x0
    
    // 4. ตั้งค่า TTBR0_EL1 (Translation Table Base Register)
    ldr x0, =ttbr0_l1
    msr ttbr0_el1, x0
    
    // 5. Enable MMU และ Caches ใน SCTLR_EL1
    mrs x0, sctlr_el1
    orr x0, x0, #(1 << 0)    // M = MMU enable
    orr x0, x0, #(1 << 2)    // C = Data cache enable
    orr x0, x0, #(1 << 12)   // I = Instruction cache enable
    
    // ISB ก่อน enable MMU
    isb
    
    msr sctlr_el1, x0
    
    // ISB หลัง enable MMU
    isb
    
    ret
```

---

## 11. Complete Example - LED Blink + UART Hello World

### 11.1 Project Structure

```
bare-metal-rpi/
├── Makefile
├── link.ld
├── config.txt
├── src/
│   ├── start.S       // Entry point
│   ├── main.c        // Main program
│   ├── uart.c        // UART driver
│   ├── uart.h
│   ├── gpio.c        // GPIO driver
│   ├── gpio.h
│   ├── timer.c       // Timer/delay
│   ├── timer.h
│   ├── led.c         // LED control
│   ├── led.h
│   ├── mailbox.c     // VideoCore mailbox
│   ├── mailbox.h
│   ├── printf.c      // Simple printf
│   ├── mmio.h
│   └── framebuffer.c // Framebuffer (optional)
└── README.md
```

### 11.2 main.c - Complete Program

```c
// main.c
#include "uart.h"
#include "gpio.h"
#include "timer.h"
#include "led.h"
#include "mailbox.h"

// ข้อมูล board revision RPi3 Model B
// https://www.raspberrypi.com/documentation/computers/raspberry-pi.html
static const char* get_model_name(uint32_t rev) {
    // Revision codes (new format since 2012)
    // Bits 23:16 = Model: 8=RPi3B, 9=RPi3B+, 11=RPi4B
    uint32_t model = (rev >> 4) & 0xFF;
    switch(model) {
        case 0x08: return "Raspberry Pi 3 Model B";
        case 0x0D: return "Raspberry Pi 3 Model B+";
        case 0x11: return "Raspberry Pi 4 Model B";
        case 0x13: return "Raspberry Pi 4 Model B (8GB)";
        default: return "Unknown Model";
    }
}

void main(void) {
    // === 1. Initialize UART ===
    uart_init();
    
    // === 2. Print Banner ===
    uart_puts("\r\n");
    uart_puts("==========================================\r\n");
    uart_puts("  ARM Bare Metal on Raspberry Pi\r\n");
    uart_puts("  AArch64 Assembly Course - Part 092\r\n");
    uart_puts("==========================================\r\n");
    uart_puts("\r\n");
    
    // === 3. Query Board Information via Mailbox ===
    uart_puts("[*] Querying board information...\r\n");
    
    uint64_t serial = 0;
    if (mbox_get_board_serial(&serial)) {
        uart_puts("    Serial: 0x");
        uart_hex(serial);
        uart_puts("\r\n");
    } else {
        uart_puts("    Serial: (failed to get)\r\n");
    }
    
    uint32_t revision = 0;
    if (mbox_get_board_revision(&revision)) {
        uart_puts("    Revision: 0x");
        uart_hex(revision);
        uart_puts(" (");
        uart_puts(get_model_name(revision));
        uart_puts(")\r\n");
    }
    
    uint32_t arm_base = 0, arm_size = 0;
    if (mbox_get_arm_memory(&arm_base, &arm_size)) {
        uart_puts("    ARM Memory Base: 0x");
        uart_hex(arm_base);
        uart_puts("\r\n    ARM Memory Size: ");
        uart_dec(arm_size / (1024 * 1024));
        uart_puts(" MB\r\n");
    }
    
    // === 4. Initialize LED ===
    uart_puts("\r\n[*] Initializing LED on GPIO ");
    uart_dec(LED_GPIO);
    uart_puts("...\r\n");
    
    led_init();
    
    // === 5. Blink 3 ครั้งเพื่อบอกว่า init สำเร็จ ===
    uart_puts("[*] Blinking 3 times (init success signal)...\r\n");
    led_blink(3, 200);  // 3 ครั้ง, 200ms interval
    
    // === 6. Print System Register Info ===
    uart_puts("\r\n[*] System Register Info:\r\n");
    
    uint64_t midr;
    asm volatile("mrs %0, midr_el1" : "=r"(midr));
    uart_puts("    MIDR_EL1: 0x");
    uart_hex(midr);
    uart_puts("\r\n");
    
    uint64_t mpidr;
    asm volatile("mrs %0, mpidr_el1" : "=r"(mpidr));
    uart_puts("    MPIDR_EL1: 0x");
    uart_hex(mpidr);
    uart_puts(" (CPU ");
    uart_dec(mpidr & 3);
    uart_puts(")\r\n");
    
    // EL level ปัจจุบัน
    uint64_t el;
    asm volatile(
        "mrs %0, currentel\n"
        "lsr %0, %0, #2"
        : "=r"(el)
    );
    uart_puts("    CurrentEL: EL");
    uart_dec(el);
    uart_puts("\r\n");
    
    // === 7. Main Loop - Blink LED และแสดง uptime ===
    uart_puts("\r\n[*] Entering main loop...\r\n");
    uart_puts("    LED blinks every 1 second\r\n\r\n");
    
    uint32_t count = 0;
    
    while (1) {
        led_on();
        timer_wait_ms(500);
        led_off();
        timer_wait_ms(500);
        
        count++;
        
        // แสดง uptime ทุก 10 วินาที
        if (count % 10 == 0) {
            uart_puts("Uptime: ");
            uart_dec(count);
            uart_puts(" seconds\r\n");
        }
    }
    
    // ไม่ควรถึงจุดนี้
    while (1) {
        asm volatile("wfe");
    }
}
```

---

## 12. AArch64 Assembly Examples สำหรับ Bare Metal

### 12.1 Memory-Mapped I/O ใน Assembly

```asm
// asm_utils.S - Assembly utilities สำหรับ bare metal

.section ".text"
.global asm_delay
.global asm_read32
.global asm_write32
.global asm_get_el

// void asm_delay(uint64_t cycles)
// ทำ delay โดยนับ cycles
asm_delay:
    subs    x0, x0, #1
    b.ne    asm_delay
    ret

// uint32_t asm_read32(uint64_t addr)
// อ่าน 32-bit value จาก memory-mapped register
asm_read32:
    ldr     w0, [x0]    // อ่านจาก address ใน x0
    ret

// void asm_write32(uint64_t addr, uint32_t val)
// เขียน 32-bit value ไปยัง memory-mapped register
asm_write32:
    str     w1, [x0]    // เขียน w1 ไปที่ address ใน x0
    ret

// uint64_t asm_get_el(void)
// Return current exception level (0-3)
asm_get_el:
    mrs     x0, CurrentEL
    lsr     x0, x0, #2
    and     x0, x0, #3
    ret
```

### 12.2 Direct GPIO Manipulation ใน Assembly

```asm
// gpio_asm.S - GPIO operations ใน Assembly (สำหรับ performance critical code)

.equ GPIO_BASE_RPi3, 0x3F200000
.equ GPIO_BASE_RPi4, 0xFE200000

.equ GPFSEL0_OFF,  0x00
.equ GPSET0_OFF,   0x1C
.equ GPCLR0_OFF,   0x28
.equ GPLEV0_OFF,   0x34

.section ".text"
.global gpio_asm_blink

// void gpio_asm_blink(uint32_t gpio_base, uint32_t pin, uint32_t delay)
// x0 = GPIO base address
// w1 = pin number (0-31)
// w2 = delay count
gpio_asm_blink:
    // เก็บ registers
    stp     x19, x20, [sp, #-16]!
    stp     x21, x30, [sp, #-16]!
    
    mov     x19, x0     // GPIO base
    mov     w20, w1     // pin number
    mov     w21, w2     // delay
    
    // คำนวณ bitmask สำหรับ pin
    mov     w22, #1
    lsl     w22, w22, w20  // mask = 1 << pin
    
.Lblink_loop:
    // Set GPIO high
    str     w22, [x19, #0x1C]   // GPSET0
    
    // Delay
    mov     w0, w21
.Ldelay1:
    subs    w0, w0, #1
    b.ne    .Ldelay1
    
    // Set GPIO low
    str     w22, [x19, #0x28]   // GPCLR0
    
    // Delay
    mov     w0, w21
.Ldelay2:
    subs    w0, w0, #1
    b.ne    .Ldelay2
    
    b       .Lblink_loop   // loop forever
    
    // restore (ไม่มีทาง return จาก infinite loop แต่ใส่ไว้เพื่อ completeness)
    ldp     x21, x30, [sp], #16
    ldp     x19, x20, [sp], #16
    ret
```

### 12.3 Exception Vector Table

```asm
// exception.S - Exception vector table สำหรับ AArch64

// Vector table ต้อง align ที่ 0x800 (2048 bytes)
.section ".text.vectors"
.global exception_vector_table

.align 11   // 2^11 = 2048

exception_vector_table:
    // === Current EL with SP0 ===
    
    // Synchronous
    .align 7    // แต่ละ entry ต้อง 128-byte aligned
    b           sync_handler_sp0
    
    // IRQ/vIRQ
    .align 7
    b           irq_handler_sp0
    
    // FIQ/vFIQ
    .align 7
    b           fiq_handler_sp0
    
    // SError/vSError
    .align 7
    b           serror_handler_sp0
    
    // === Current EL with SPx ===
    
    .align 7
    b           sync_handler_spx     // Synchronous
    
    .align 7
    b           irq_handler_spx      // IRQ
    
    .align 7
    b           fiq_handler_spx      // FIQ
    
    .align 7
    b           serror_handler_spx   // SError
    
    // === Lower EL using AArch64 ===
    
    .align 7
    b           sync_handler_lower64
    
    .align 7
    b           irq_handler_lower64
    
    .align 7
    b           fiq_handler_lower64
    
    .align 7
    b           serror_handler_lower64
    
    // === Lower EL using AArch32 ===
    
    .align 7
    b           sync_handler_lower32
    
    .align 7
    b           irq_handler_lower32
    
    .align 7
    b           fiq_handler_lower32
    
    .align 7
    b           serror_handler_lower32


// Macro สำหรับ save/restore registers
.macro SAVE_REGS
    sub     sp, sp, #0x110
    stp     x0,  x1,  [sp, #0x00]
    stp     x2,  x3,  [sp, #0x10]
    stp     x4,  x5,  [sp, #0x20]
    stp     x6,  x7,  [sp, #0x30]
    stp     x8,  x9,  [sp, #0x40]
    stp     x10, x11, [sp, #0x50]
    stp     x12, x13, [sp, #0x60]
    stp     x14, x15, [sp, #0x70]
    stp     x16, x17, [sp, #0x80]
    stp     x18, x19, [sp, #0x90]
    stp     x20, x21, [sp, #0xA0]
    stp     x22, x23, [sp, #0xB0]
    stp     x24, x25, [sp, #0xC0]
    stp     x26, x27, [sp, #0xD0]
    stp     x28, x29, [sp, #0xE0]
    mrs     x0, elr_el1
    mrs     x1, spsr_el1
    stp     x30, x0,  [sp, #0xF0]
    str     x1,       [sp, #0x100]
.endm

.macro RESTORE_REGS
    ldr     x1,       [sp, #0x100]
    ldp     x30, x0,  [sp, #0xF0]
    msr     elr_el1, x0
    msr     spsr_el1, x1
    ldp     x28, x29, [sp, #0xE0]
    ldp     x26, x27, [sp, #0xD0]
    ldp     x24, x25, [sp, #0xC0]
    ldp     x22, x23, [sp, #0xB0]
    ldp     x20, x21, [sp, #0xA0]
    ldp     x18, x19, [sp, #0x90]
    ldp     x16, x17, [sp, #0x80]
    ldp     x14, x15, [sp, #0x70]
    ldp     x12, x13, [sp, #0x60]
    ldp     x10, x11, [sp, #0x50]
    ldp     x8,  x9,  [sp, #0x40]
    ldp     x6,  x7,  [sp, #0x30]
    ldp     x4,  x5,  [sp, #0x20]
    ldp     x2,  x3,  [sp, #0x10]
    ldp     x0,  x1,  [sp, #0x00]
    add     sp, sp, #0x110
.endm

// Exception handlers (ที่ใช้งานจริง - SPx)
sync_handler_spx:
    SAVE_REGS
    
    mrs     x0, esr_el1     // Exception Syndrome Register
    mrs     x1, elr_el1     // Exception Link Register (PC ที่ทำให้เกิด exception)
    mrs     x2, far_el1     // Fault Address Register
    
    bl      handle_sync_exception  // C function
    
    RESTORE_REGS
    eret

irq_handler_spx:
    SAVE_REGS
    bl      handle_irq
    RESTORE_REGS
    eret

// Stub handlers สำหรับ handlers ที่ยังไม่ implement
sync_handler_sp0:
irq_handler_sp0:
fiq_handler_sp0:
serror_handler_sp0:
fiq_handler_spx:
serror_handler_spx:
sync_handler_lower64:
irq_handler_lower64:
fiq_handler_lower64:
serror_handler_lower64:
sync_handler_lower32:
irq_handler_lower32:
fiq_handler_lower32:
serror_handler_lower32:
    // Spin forever
1:  wfe
    b 1b
```

---

## 13. Testing ด้วย QEMU

### 13.1 QEMU Setup

```bash
# ติดตั้ง QEMU
sudo apt-get install qemu-system-aarch64

# ตรวจสอบ QEMU machines ที่ support
qemu-system-aarch64 -machine help | grep raspi

# Compile code
make

# Run ใน QEMU (RPi3)
qemu-system-aarch64 \
    -machine raspi3b \
    -cpu cortex-a53 \
    -m 1024 \
    -nographic \
    -serial null \
    -serial mon:stdio \
    -kernel kernel8.img

# Run ใน QEMU พร้อม debug (GDB)
qemu-system-aarch64 \
    -machine raspi3b \
    -cpu cortex-a53 \
    -m 1024 \
    -nographic \
    -serial null \
    -serial mon:stdio \
    -kernel kernel8.img \
    -S -s          # -S = stop at startup, -s = GDB server port 1234
```

### 13.2 GDB Debugging

```bash
# เปิด terminal ใหม่
aarch64-linux-gnu-gdb kernel8.elf

# ใน GDB:
(gdb) target remote :1234
(gdb) break main
(gdb) continue
(gdb) info registers
(gdb) x/10i $pc         # ดู 10 instructions จาก PC
(gdb) x/4xw 0x3F200000  # ดู GPIO registers
```

### 13.3 Real Hardware Testing

```bash
# Format SD card (Linux)
sudo fdisk /dev/sdX
# สร้าง FAT32 partition แรก

# Mount
sudo mount /dev/sdX1 /mnt/sdcard

# Copy files
sudo cp firmware/bootcode.bin /mnt/sdcard/
sudo cp firmware/start.elf /mnt/sdcard/
sudo cp firmware/fixup.dat /mnt/sdcard/
sudo cp config.txt /mnt/sdcard/
sudo cp kernel8.img /mnt/sdcard/

# Unmount
sudo umount /mnt/sdcard

# เสียบ SD card ใน RPi และเชื่อมต่อ UART adapter
# GPIO 14 (TXD) → RX ของ adapter
# GPIO 15 (RXD) → TX ของ adapter
# GND → GND

# เปิด serial terminal
screen /dev/ttyUSB0 115200
# หรือ
minicom -b 115200 -D /dev/ttyUSB0
```

---

## 14. Advanced Topics

### 14.1 Interrupt Handling

```c
// interrupt.h
#ifndef INTERRUPT_H
#define INTERRUPT_H

#include <stdint.h>

// Interrupt Controller Registers (BCM2837)
#define IRQ_BASE        (MMIO_BASE + 0x00B000)
#define IRQ_BASIC_PEND  (IRQ_BASE + 0x200)
#define IRQ_PEND1       (IRQ_BASE + 0x204)
#define IRQ_PEND2       (IRQ_BASE + 0x208)
#define IRQ_FIQ_CTRL    (IRQ_BASE + 0x20C)
#define IRQ_ENABLE1     (IRQ_BASE + 0x210)
#define IRQ_ENABLE2     (IRQ_BASE + 0x214)
#define IRQ_ENABLE_BASIC (IRQ_BASE + 0x218)
#define IRQ_DISABLE1    (IRQ_BASE + 0x21C)
#define IRQ_DISABLE2    (IRQ_BASE + 0x220)

// IRQ numbers
#define IRQ_SYSTEM_TIMER_1  1   // System Timer Compare 1
#define IRQ_SYSTEM_TIMER_3  3   // System Timer Compare 3
#define IRQ_USB             9
#define IRQ_UART            57  // UART (PL011)

void irq_init(void);
void irq_enable(uint32_t irq_num);
void irq_disable(uint32_t irq_num);
void handle_irq(void);                   // called from assembly handler
void handle_sync_exception(uint64_t esr, uint64_t elr, uint64_t far);

// Timer interrupt callback
typedef void (*irq_handler_t)(void);
void irq_register_handler(uint32_t irq_num, irq_handler_t handler);

#endif
```

```c
// interrupt.c
#include "interrupt.h"
#include "mmio.h"
#include "uart.h"

// ตาราง handlers
#define MAX_IRQ 72
static irq_handler_t irq_handlers[MAX_IRQ] = {0};

void irq_init(void) {
    // ปิด interrupts ทั้งหมดก่อน
    mmio_write(IRQ_DISABLE1, 0xFFFFFFFF);
    mmio_write(IRQ_DISABLE2, 0xFFFFFFFF);
    
    // ตั้งค่า exception vector table
    extern uint64_t exception_vector_table;
    asm volatile(
        "msr vbar_el1, %0"
        :: "r"(&exception_vector_table)
    );
    
    // Enable IRQ ใน DAIF register
    asm volatile("msr daifclr, #2");  // Clear I bit = enable IRQ
}

void irq_enable(uint32_t irq_num) {
    if (irq_num < 32) {
        mmio_write(IRQ_ENABLE1, 1 << irq_num);
    } else if (irq_num < 64) {
        mmio_write(IRQ_ENABLE2, 1 << (irq_num - 32));
    } else {
        mmio_write(IRQ_ENABLE_BASIC, 1 << (irq_num - 64));
    }
}

void irq_register_handler(uint32_t irq_num, irq_handler_t handler) {
    if (irq_num < MAX_IRQ) {
        irq_handlers[irq_num] = handler;
    }
}

// เรียกจาก assembly exception handler
void handle_irq(void) {
    uint32_t pend1 = mmio_read(IRQ_PEND1);
    uint32_t pend2 = mmio_read(IRQ_PEND2);
    
    // ตรวจสอบ pending interrupts
    for (int i = 0; i < 32; i++) {
        if ((pend1 >> i) & 1) {
            if (irq_handlers[i]) {
                irq_handlers[i]();
            }
        }
    }
    
    for (int i = 0; i < 32; i++) {
        if ((pend2 >> i) & 1) {
            if (irq_handlers[32 + i]) {
                irq_handlers[32 + i]();
            }
        }
    }
}

void handle_sync_exception(uint64_t esr, uint64_t elr, uint64_t far) {
    uart_puts("\r\n!!! SYNCHRONOUS EXCEPTION !!!\r\n");
    uart_puts("ESR_EL1: 0x"); uart_hex(esr); uart_puts("\r\n");
    uart_puts("ELR_EL1: 0x"); uart_hex(elr); uart_puts("\r\n");
    uart_puts("FAR_EL1: 0x"); uart_hex(far); uart_puts("\r\n");
    
    // Exception class (ESR_EL1[31:26])
    uint64_t ec = (esr >> 26) & 0x3F;
    uart_puts("Exception Class: 0x"); uart_hex(ec); uart_puts(" = ");
    
    switch (ec) {
        case 0x00: uart_puts("Unknown reason\r\n"); break;
        case 0x01: uart_puts("WFI/WFE trapped\r\n"); break;
        case 0x0E: uart_puts("Illegal execution state\r\n"); break;
        case 0x15: uart_puts("SVC in AArch64\r\n"); break;
        case 0x20: uart_puts("Instruction abort (lower EL)\r\n"); break;
        case 0x21: uart_puts("Instruction abort (same EL)\r\n"); break;
        case 0x24: uart_puts("Data abort (lower EL)\r\n"); break;
        case 0x25: uart_puts("Data abort (same EL)\r\n"); break;
        case 0x26: uart_puts("Stack alignment fault\r\n"); break;
        case 0x30: uart_puts("Breakpoint (lower EL)\r\n"); break;
        case 0x31: uart_puts("Breakpoint (same EL)\r\n"); break;
        default: uart_puts("Other\r\n"); break;
    }
    
    // Halt
    while (1) {
        asm volatile("wfe");
    }
}
```

### 14.2 System Timer Interrupt

```c
// systimer.c - System Timer สำหรับ periodic interrupts
#include "interrupt.h"
#include "mmio.h"

#define SYSTIMER_INTERVAL   1000000  // 1 second (1MHz clock)

static volatile uint32_t tick_count = 0;

static void systimer_handler(void) {
    tick_count++;
    
    // อัปเดต compare register สำหรับ next interrupt
    uint32_t curr = mmio_read(SYSTIMER_CLO);
    mmio_write(SYSTIMER_C1, curr + SYSTIMER_INTERVAL);
    
    // Clear interrupt flag
    mmio_write(SYSTIMER_CS, (1 << 1));   // CS bit 1 for Compare 1
}

void systimer_init(void) {
    // ลงทะเบียน handler
    irq_register_handler(IRQ_SYSTEM_TIMER_1, systimer_handler);
    
    // ตั้ง compare value
    uint32_t curr = mmio_read(SYSTIMER_CLO);
    mmio_write(SYSTIMER_C1, curr + SYSTIMER_INTERVAL);
    
    // Enable System Timer IRQ 1
    irq_enable(IRQ_SYSTEM_TIMER_1);
}

uint32_t systimer_get_ticks(void) {
    return tick_count;
}
```

---

## 15. สรุปและ Troubleshooting

### 15.1 ปัญหาที่พบบ่อย

**1. ไม่มี output จาก UART:**
```
ตรวจสอบ:
- config.txt มี arm_64bit=1 และ dtoverlay=disable-bt
- GPIO 14/15 ต่อ function ALT0
- Baud rate divisor ถูกต้องสำหรับ board ที่ใช้
- UART clock ที่กำหนดใน config.txt ตรงกับ code
```

**2. Board hang หลัง boot:**
```
ตรวจสอบ:
- start.S ปิด CPU 1-3 ก่อน (MPIDR check)
- Stack pointer ตั้งค่าถูกต้อง
- BSS clear ก่อน jump to main
- link.ld ถูกต้อง (entry ที่ 0x80000)
```

**3. Mailbox call ไม่สำเร็จ:**
```
ตรวจสอบ:
- Buffer align 16 bytes
- Buffer size ถูกต้อง
- Data memory barrier (dmb sy) ก่อน/หลัง mailbox call
- Channel = 8 สำหรับ property interface
```

**4. GPIO ไม่ทำงาน:**
```
ตรวจสอบ:
- GPFSEL register ค่าถูก (3 bits per pin)
- GPPUD sequence ถูกต้องสำหรับ RPi3 (150 cycle delay)
- RPi4 ใช้ GPIO_PUP_PDN_CNTRL registers แทน GPPUD
- Peripheral base address ถูกต้อง (RPi3 vs RPi4)
```

### 15.2 เครื่องมือที่มีประโยชน์

```bash
# Disassemble kernel image
aarch64-linux-gnu-objdump -d kernel8.elf | head -100

# ดู symbols
aarch64-linux-gnu-nm kernel8.elf | sort

# ดู sections
aarch64-linux-gnu-size kernel8.elf
aarch64-linux-gnu-readelf -S kernel8.elf

# ตรวจสอบ entry point
aarch64-linux-gnu-readelf -h kernel8.elf | grep "Entry point"

# Dump first 256 bytes ของ kernel image
xxd kernel8.img | head -16
```

### 15.3 Resources ที่ควรอ่าน

```
1. BCM2837 ARM Peripherals Manual (RPi3)
   https://www.raspberrypi.com/documentation/computers/processors.html

2. BCM2711 ARM Peripherals Manual (RPi4)
   https://datasheets.raspberrypi.com/bcm2711/bcm2711-peripherals.pdf

3. ARM Cortex-A53 Technical Reference Manual
   https://developer.arm.com/documentation/ddi0500/latest/

4. ARM Architecture Reference Manual (AArch64)
   https://developer.arm.com/documentation/ddi0487/latest

5. Raspberry Pi Firmware Documentation
   https://github.com/raspberrypi/firmware/wiki

6. OSDev Wiki - Raspberry Pi Bare Bones
   https://wiki.osdev.org/Raspberry_Pi_Bare_Bones

7. Learning OS Development using Linux kernel and Raspberry Pi
   https://github.com/s-matyukevich/raspberry-pi-os
```

---

## สรุป Part 092

ในบทนี้เราได้เรียนรู้:

1. **Boot Process**: GPU ROM → bootcode.bin → start.elf → kernel8.img, การกำหนด config.txt
2. **Memory Map**: ตำแหน่ง peripheral registers สำหรับ BCM2837 (RPi3) และ BCM2711 (RPi4)
3. **Startup Code**: AArch64 entry point ที่ 0x80000, การ park CPUs 1-3, stack setup, BSS clear
4. **UART0 (PL011)**: Register map, baud rate calculation, 8N1 configuration, polling I/O
5. **GPIO**: GPFSEL function select, GPSET/GPCLR output, GPLEV input, pull-up/down programming
6. **LED Blink**: GPIO output สำหรับ LED, System Timer สำหรับ precise delay
7. **printf**: Simple formatted output ผ่าน UART
8. **Mailbox Interface**: Property tag interface สำหรับ board information
9. **Framebuffer**: การตั้งค่า display และ draw pixels
10. **MMU Setup**: Simple identity mapping ด้วย Level 1 translation table
11. **Exception Handling**: Vector table, saving/restoring registers, ESR decoding
12. **QEMU Testing**: การ test โดยไม่ต้องมี hardware จริง

Bare metal programming บน Raspberry Pi เป็นทักษะที่ให้ความเข้าใจเกี่ยวกับ hardware อย่างลึกซึ้ง ซึ่งเป็นพื้นฐานสำคัญสำหรับการพัฒนา operating systems, embedded systems และ real-time systems
