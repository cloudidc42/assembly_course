# Part 089: Device Drivers ใน Assembly

## บทนำ

Device Drivers คือโค้ดที่ทำหน้าที่เป็นสะพานระหว่าง Operating System กับ Hardware จริงๆ ใน Assembly เราสามารถเขียน Driver ได้โดยตรงโดยใช้คำสั่ง I/O port และ interrupt handling ซึ่งให้ประสิทธิภาพสูงสุดและควบคุม Hardware ได้อย่างละเอียด

บทนี้ครอบคลุม:
- I/O Port Access: IN/OUT instructions
- PIC (8259A) Driver
- Keyboard Driver (PS/2)
- PIT (8253/8254) Timer Driver
- VGA Text Mode Driver
- Serial UART (16550) Driver
- ATA PIO Mode Driver
- Device Abstraction Layer

---

## 1. I/O Port Access: IN/OUT Instructions

### 1.1 พื้นฐาน I/O Port

ใน x86 Architecture มี Address Space แยกสำหรับ I/O ports ขนาด 64KB (ports 0x0000 - 0xFFFF)

```nasm
; ============================================================
; io_basics.asm - พื้นฐาน I/O Port Access
; ============================================================
; compile: nasm -f elf32 io_basics.asm -o io_basics.o
; link:    ld -m elf_i386 io_basics.o -o io_basics

section .text

; -----------------------------------------------------------
; IN instruction - อ่านข้อมูลจาก I/O port
; -----------------------------------------------------------

; อ่าน 1 byte จาก port
; Input:  DX = port number
; Output: AL = data
inb:
    in al, dx
    ret

; อ่าน 2 bytes (word) จาก port
; Input:  DX = port number
; Output: AX = data
inw:
    in ax, dx
    ret

; อ่าน 4 bytes (dword) จาก port
; Input:  DX = port number
; Output: EAX = data
ind:
    in eax, dx
    ret

; -----------------------------------------------------------
; OUT instruction - เขียนข้อมูลไปยัง I/O port
; -----------------------------------------------------------

; เขียน 1 byte ไปยัง port
; Input:  DX = port number, AL = data
outb:
    out dx, al
    ret

; เขียน 2 bytes ไปยัง port
; Input:  DX = port number, AX = data
outw:
    out dx, ax
    ret

; เขียน 4 bytes ไปยัง port
; Input:  DX = port number, EAX = data
outd:
    out dx, eax
    ret
```

### 1.2 String I/O Instructions

```nasm
; ============================================================
; string_io.asm - String I/O Instructions
; ============================================================

section .data
    buffer times 512 db 0       ; Buffer รับข้อมูล 512 bytes
    src_buffer times 512 db 0   ; Buffer ส่งข้อมูล

section .text

; -----------------------------------------------------------
; INSB - Input String Byte
; อ่านข้อมูลจาก port ไปยัง memory buffer
; Input:  DX = port, ES:EDI = destination buffer, ECX = count
; -----------------------------------------------------------
read_block_insb:
    push edi
    push ecx
    push edx
    
    mov dx, 0x1F0       ; ATA data port (ตัวอย่าง)
    mov edi, buffer
    mov ecx, 512        ; อ่าน 512 bytes
    cld                 ; Direction flag = forward
    rep insb            ; อ่านซ้ำ ECX ครั้ง
    
    pop edx
    pop ecx
    pop edi
    ret

; -----------------------------------------------------------
; INSW - Input String Word (อ่านทีละ 2 bytes)
; ใช้กับ ATA ซึ่งส่งข้อมูลทีละ word
; -----------------------------------------------------------
read_block_insw:
    push edi
    push ecx
    push edx
    
    mov dx, 0x1F0       ; ATA data port
    mov edi, buffer
    mov ecx, 256        ; 256 words = 512 bytes
    cld
    rep insw            ; อ่าน word ซ้ำ ECX ครั้ง
    
    pop edx
    pop ecx
    pop edi
    ret

; -----------------------------------------------------------
; INSD - Input String Dword (อ่านทีละ 4 bytes)
; -----------------------------------------------------------
read_block_insd:
    push edi
    push ecx
    push edx
    
    mov dx, 0x1F0
    mov edi, buffer
    mov ecx, 128        ; 128 dwords = 512 bytes
    cld
    rep insd
    
    pop edx
    pop ecx
    pop edi
    ret

; -----------------------------------------------------------
; OUTSB/OUTSW/OUTSD - Output String
; ส่งข้อมูลจาก memory ไปยัง port
; -----------------------------------------------------------
write_block_outsw:
    push esi
    push ecx
    push edx
    
    mov dx, 0x1F0       ; ATA data port
    mov esi, src_buffer
    mov ecx, 256        ; 256 words = 512 bytes
    cld
    rep outsw           ; ส่ง word ซ้ำ ECX ครั้ง
    
    pop edx
    pop ecx
    pop esi
    ret

; -----------------------------------------------------------
; I/O Wait - รอให้ Hardware ประมวลผล
; เทคนิคคลาสสิก: เขียนไปยัง port 0x80 (unused)
; -----------------------------------------------------------
io_wait:
    push ax
    xor al, al
    out 0x80, al        ; ส่ง 0 ไปยัง port 0x80 (เสียเวลา ~1μs)
    pop ax
    ret
```

### 1.3 Fixed Port IN/OUT

```nasm
; -----------------------------------------------------------
; Fixed port addressing (port number เป็น immediate value)
; ใช้ได้กับ port 0x00 - 0xFF เท่านั้น
; -----------------------------------------------------------

; อ่าน status ของ keyboard
read_keyboard_status:
    in al, 0x64         ; port 0x64 = keyboard status register
    ret

; เปิด/ปิด speaker
enable_speaker:
    in al, 0x61         ; อ่าน port B ของ PPI
    or al, 0x03         ; set bit 0 (timer gate) และ bit 1 (speaker)
    out 0x61, al
    ret

disable_speaker:
    in al, 0x61
    and al, 0xFC        ; clear bit 0 และ bit 1
    out 0x61, al
    ret
```

---

## 2. PIC Driver: 8259A Programmable Interrupt Controller

### 2.1 โครงสร้าง PIC

ระบบ x86 มี PIC 2 ตัว (Master และ Slave) รองรับ IRQ 0-15:
- Master PIC: IRQ 0-7 (ports 0x20, 0x21)
- Slave PIC: IRQ 8-15 (ports 0xA0, 0xA1)

```nasm
; ============================================================
; pic_driver.asm - 8259A PIC Driver
; ============================================================
; compile: nasm -f elf32 pic_driver.asm -o pic_driver.o

section .text

; -----------------------------------------------------------
; PIC Port Definitions
; -----------------------------------------------------------
%define PIC1_CMD    0x20    ; Master PIC command port
%define PIC1_DATA   0x21    ; Master PIC data port
%define PIC2_CMD    0xA0    ; Slave PIC command port
%define PIC2_DATA   0xA1    ; Slave PIC data port

; ICW1 flags
%define ICW1_ICW4   0x01    ; ICW4 ต้องการ
%define ICW1_SINGLE 0x02    ; Single (ไม่ใช่ cascade) mode
%define ICW1_INTERVAL4 0x04 ; Call address interval 4
%define ICW1_LEVEL  0x08    ; Level triggered (ไม่ใช่ edge)
%define ICW1_INIT   0x10    ; Initialization bit (ต้องมีเสมอ)

; ICW4 flags
%define ICW4_8086   0x01    ; 8086/88 mode (ไม่ใช่ MCS-80/85)
%define ICW4_AUTO   0x02    ; Auto EOI
%define ICW4_BUF_SLAVE  0x08  ; Buffered mode slave
%define ICW4_BUF_MASTER 0x0C  ; Buffered mode master
%define ICW4_SFNM   0x10    ; Special fully nested mode

; EOI command
%define PIC_EOI     0x20    ; End of Interrupt

; -----------------------------------------------------------
; pic_init - Initialize PIC with new interrupt vectors
; Input:  AL = offset1 (Master PIC base interrupt, ปกติ 0x20)
;         AH = offset2 (Slave PIC base interrupt, ปกติ 0x28)
; -----------------------------------------------------------
global pic_init
pic_init:
    push ebx
    push edx
    
    movzx ebx, al       ; เก็บ offset1
    movzx edx, ah       ; เก็บ offset2
    
    ; บันทึก interrupt masks ปัจจุบัน
    in al, PIC1_DATA
    push eax            ; บันทึก mask1
    in al, PIC2_DATA
    push eax            ; บันทึก mask2
    
    ; ------- ICW1: Start initialization sequence -------
    ; ส่ง ICW1 ไปยังทั้ง Master และ Slave
    mov al, ICW1_INIT | ICW1_ICW4   ; 0x11
    out PIC1_CMD, al    ; เริ่ม init Master PIC
    call io_wait
    out PIC2_CMD, al    ; เริ่ม init Slave PIC
    call io_wait
    
    ; ------- ICW2: Set interrupt vector offsets -------
    ; กำหนด base interrupt number
    mov al, bl          ; offset1 = 0x20 (IRQ0 -> INT 0x20)
    out PIC1_DATA, al
    call io_wait
    
    mov al, dl          ; offset2 = 0x28 (IRQ8 -> INT 0x28)
    out PIC2_DATA, al
    call io_wait
    
    ; ------- ICW3: Configure cascade -------
    ; Master: บอกว่า IRQ2 เชื่อมกับ Slave
    mov al, 4           ; IRQ2 = bit 2 = 0b00000100
    out PIC1_DATA, al
    call io_wait
    
    ; Slave: บอก cascade identity (IRQ2 = 2)
    mov al, 2           ; Slave อยู่ที่ IRQ2
    out PIC2_DATA, al
    call io_wait
    
    ; ------- ICW4: Set operating mode -------
    mov al, ICW4_8086   ; 0x01: 8086 mode
    out PIC1_DATA, al
    call io_wait
    out PIC2_DATA, al
    call io_wait
    
    ; กู้คืน interrupt masks
    pop eax
    out PIC2_DATA, al   ; กู้คืน mask Slave
    call io_wait
    pop eax
    out PIC1_DATA, al   ; กู้คืน mask Master
    call io_wait
    
    pop edx
    pop ebx
    ret

; -----------------------------------------------------------
; pic_enable_irq - Enable (unmask) specific IRQ
; Input: AL = IRQ number (0-15)
; -----------------------------------------------------------
global pic_enable_irq
pic_enable_irq:
    push ecx
    push edx
    
    movzx ecx, al
    
    cmp cl, 8
    jge .slave_irq
    
    ; Master PIC (IRQ 0-7)
    in al, PIC1_DATA        ; อ่าน mask ปัจจุบัน
    mov edx, 1
    shl edx, cl             ; สร้าง bitmask
    not edx
    and al, dl              ; Clear bit ที่ต้องการ (enable IRQ)
    out PIC1_DATA, al
    jmp .done
    
.slave_irq:
    ; Slave PIC (IRQ 8-15)
    sub cl, 8               ; ปรับ offset สำหรับ Slave
    in al, PIC2_DATA
    mov edx, 1
    shl edx, cl
    not edx
    and al, dl
    out PIC2_DATA, al
    
    ; ต้องเปิด IRQ2 บน Master ด้วย (cascade line)
    in al, PIC1_DATA
    and al, ~(1 << 2)
    out PIC1_DATA, al
    
.done:
    pop edx
    pop ecx
    ret

; -----------------------------------------------------------
; pic_disable_irq - Disable (mask) specific IRQ
; Input: AL = IRQ number (0-15)
; -----------------------------------------------------------
global pic_disable_irq
pic_disable_irq:
    push ecx
    push edx
    
    movzx ecx, al
    
    cmp cl, 8
    jge .slave_irq
    
    ; Master PIC (IRQ 0-7)
    in al, PIC1_DATA
    mov edx, 1
    shl edx, cl
    or al, dl               ; Set bit = disable (mask) IRQ
    out PIC1_DATA, al
    jmp .done
    
.slave_irq:
    sub cl, 8
    in al, PIC2_DATA
    mov edx, 1
    shl edx, cl
    or al, dl
    out PIC2_DATA, al
    
.done:
    pop edx
    pop ecx
    ret

; -----------------------------------------------------------
; pic_send_eoi - Send End of Interrupt signal
; Input: AL = IRQ number (0-15)
; -----------------------------------------------------------
global pic_send_eoi
pic_send_eoi:
    cmp al, 8
    jge .slave_eoi
    
    ; IRQ 0-7: ส่ง EOI ไปยัง Master เท่านั้น
    mov al, PIC_EOI
    out PIC1_CMD, al
    ret
    
.slave_eoi:
    ; IRQ 8-15: ส่ง EOI ไปยัง Slave ก่อน แล้วค่อย Master
    mov al, PIC_EOI
    out PIC2_CMD, al    ; EOI ไปยัง Slave
    out PIC1_CMD, al    ; EOI ไปยัง Master (cascade)
    ret

; -----------------------------------------------------------
; pic_get_irr - Get Interrupt Request Register
; Output: AX = IRR value (Master=AL, Slave=AH)
; -----------------------------------------------------------
global pic_get_irr
pic_get_irr:
    mov al, 0x0A        ; OCW3: Read IRR
    out PIC1_CMD, al
    out PIC2_CMD, al
    in al, PIC1_CMD     ; อ่าน IRR Master
    mov ah, al
    in al, PIC2_CMD     ; อ่าน IRR Slave
    xchg al, ah         ; Master=AL, Slave=AH
    ret

; -----------------------------------------------------------
; pic_get_isr - Get In-Service Register
; Output: AX = ISR value (Master=AL, Slave=AH)
; -----------------------------------------------------------
global pic_get_isr
pic_get_isr:
    mov al, 0x0B        ; OCW3: Read ISR
    out PIC1_CMD, al
    out PIC2_CMD, al
    in al, PIC1_CMD
    mov ah, al
    in al, PIC2_CMD
    xchg al, ah
    ret

; -----------------------------------------------------------
; io_wait - ส่ง dummy write เพื่อรอ I/O
; -----------------------------------------------------------
io_wait:
    push eax
    xor al, al
    out 0x80, al
    pop eax
    ret
```

---

## 3. Keyboard Driver (PS/2)

### 3.1 PS/2 Keyboard Interface

```nasm
; ============================================================
; keyboard_driver.asm - PS/2 Keyboard Driver
; ============================================================
; IRQ1 -> INT 0x21 (หลังจาก remap PIC ด้วย offset 0x20)

section .data

; -----------------------------------------------------------
; Scan Code Set 1 -> ASCII Conversion Table
; index = scan code, value = ASCII character
; -----------------------------------------------------------
scancode_table_lower:
    db 0,    27,  '1', '2', '3', '4', '5', '6'  ; 0x00 - 0x07
    db '7',  '8', '9', '0', '-', '=', 8,   9    ; 0x08 - 0x0F (8=BS, 9=TAB)
    db 'q',  'w', 'e', 'r', 't', 'y', 'u', 'i'  ; 0x10 - 0x17
    db 'o',  'p', '[', ']', 13,  0,   'a', 's'  ; 0x18 - 0x1F (13=CR)
    db 'd',  'f', 'g', 'h', 'j', 'k', 'l', ';'  ; 0x20 - 0x27
    db "'",  '`', 0,   '\', 'z', 'x', 'c', 'v'  ; 0x28 - 0x2F
    db 'b',  'n', 'm', ',', '.', '/', 0,   '*'  ; 0x30 - 0x37
    db 0,    ' ', 0,   0,   0,   0,   0,   0    ; 0x38 - 0x3F
    db 0,    0,   0,   0,   0,   0,   0,   '7'  ; 0x40 - 0x47
    db '8',  '9', '-', '4', '5', '6', '+', '1'  ; 0x48 - 0x4F
    db '2',  '3', '0', '.', 0,   0,   0,   0    ; 0x50 - 0x57

; Shifted version ของ scancode table
scancode_table_upper:
    db 0,    27,  '!', '@', '#', '$', '%', '^'  ; 0x00 - 0x07
    db '&',  '*', '(', ')', '_', '+', 8,   9    ; 0x08 - 0x0F
    db 'Q',  'W', 'E', 'R', 'T', 'Y', 'U', 'I'  ; 0x10 - 0x17
    db 'O',  'P', '{', '}', 13,  0,   'A', 'S'  ; 0x18 - 0x1F
    db 'D',  'F', 'G', 'H', 'J', 'K', 'L', ':'  ; 0x20 - 0x27
    db '"',  '~', 0,   '|', 'Z', 'X', 'C', 'V'  ; 0x28 - 0x2F
    db 'B',  'N', 'M', '<', '>', '?', 0,   '*'  ; 0x30 - 0x37
    db 0,    ' ', 0,   0,   0,   0,   0,   0    ; 0x38 - 0x3F

; -----------------------------------------------------------
; Keyboard State Variables
; -----------------------------------------------------------
shift_pressed:  db 0    ; 1 = shift กดอยู่
ctrl_pressed:   db 0    ; 1 = ctrl กดอยู่
alt_pressed:    db 0    ; 1 = alt กดอยู่
caps_lock:      db 0    ; 1 = caps lock เปิด
num_lock:       db 0    ; 1 = num lock เปิด

; Keyboard buffer (circular buffer)
kb_buffer:      times 256 db 0
kb_head:        dw 0    ; ตำแหน่งที่จะเขียน
kb_tail:        dw 0    ; ตำแหน่งที่จะอ่าน

section .text

; -----------------------------------------------------------
; Scan code สำหรับ modifier keys
; -----------------------------------------------------------
%define SCAN_LSHIFT     0x2A
%define SCAN_RSHIFT     0x36
%define SCAN_CTRL       0x1D
%define SCAN_ALT        0x38
%define SCAN_CAPS_LOCK  0x3A
%define SCAN_NUM_LOCK   0x45
%define SCAN_BREAK_MASK 0x80    ; bit 7 = key release (break code)

; PS/2 Ports
%define KBD_DATA_PORT   0x60    ; Data register
%define KBD_STATUS_PORT 0x64    ; Status register (read)
%define KBD_CMD_PORT    0x64    ; Command register (write)

; Status register bits
%define KBD_STATUS_OBF  0x01    ; Output Buffer Full (data ready to read)
%define KBD_STATUS_IBF  0x02    ; Input Buffer Full (busy)

; -----------------------------------------------------------
; keyboard_handler - IRQ1 Interrupt Handler
; -----------------------------------------------------------
global keyboard_handler
keyboard_handler:
    push eax
    push ebx
    push ecx
    
    ; อ่าน scan code จาก port 0x60
    in al, KBD_DATA_PORT
    
    ; เก็บ scan code ไว้ใน BL
    mov bl, al
    
    ; ตรวจสอบว่า break code หรือ make code
    test al, SCAN_BREAK_MASK
    jnz .key_release
    
    ; ============ KEY PRESS (Make Code) ============
    
    ; ตรวจสอบ modifier keys
    cmp al, SCAN_LSHIFT
    je .left_shift_down
    cmp al, SCAN_RSHIFT
    je .right_shift_down
    cmp al, SCAN_CTRL
    je .ctrl_down
    cmp al, SCAN_ALT
    je .alt_down
    cmp al, SCAN_CAPS_LOCK
    je .caps_lock_toggle
    cmp al, SCAN_NUM_LOCK
    je .num_lock_toggle
    
    ; แปลง scan code เป็น ASCII
    call scancode_to_ascii
    test al, al             ; ตรวจสอบว่ามี character
    jz .end_handler
    
    ; ใส่ character เข้า keyboard buffer
    call kb_buffer_put
    jmp .end_handler
    
.left_shift_down:
.right_shift_down:
    mov byte [shift_pressed], 1
    jmp .end_handler
    
.ctrl_down:
    mov byte [ctrl_pressed], 1
    jmp .end_handler
    
.alt_down:
    mov byte [alt_pressed], 1
    jmp .end_handler
    
.caps_lock_toggle:
    xor byte [caps_lock], 1     ; Toggle caps lock
    jmp .end_handler
    
.num_lock_toggle:
    xor byte [num_lock], 1      ; Toggle num lock
    jmp .end_handler
    
    ; ============ KEY RELEASE (Break Code) ============
.key_release:
    and al, ~SCAN_BREAK_MASK    ; ลบ break bit ออก
    
    cmp al, SCAN_LSHIFT
    je .left_shift_up
    cmp al, SCAN_RSHIFT
    je .right_shift_up
    cmp al, SCAN_CTRL
    je .ctrl_up
    cmp al, SCAN_ALT
    je .alt_up
    jmp .end_handler
    
.left_shift_up:
.right_shift_up:
    mov byte [shift_pressed], 0
    jmp .end_handler
    
.ctrl_up:
    mov byte [ctrl_pressed], 0
    jmp .end_handler
    
.alt_up:
    mov byte [alt_pressed], 0
    jmp .end_handler
    
.end_handler:
    ; ส่ง EOI ไปยัง PIC
    mov al, 0x20
    out 0x20, al
    
    pop ecx
    pop ebx
    pop eax
    iret

; -----------------------------------------------------------
; scancode_to_ascii - แปลง scan code เป็น ASCII
; Input:  AL = scan code (make code, ไม่มี break bit)
; Output: AL = ASCII character (0 = ไม่มี character)
; -----------------------------------------------------------
scancode_to_ascii:
    push ebx
    push ecx
    
    ; ตรวจสอบว่า scan code อยู่ในช่วงที่รองรับ
    cmp al, 0x57        ; ขนาดของ table
    jae .no_char
    
    movzx ebx, al       ; ใช้ scan code เป็น index
    
    ; ตรวจสอบว่าต้องใช้ uppercase หรือ lowercase
    mov cl, [shift_pressed]
    
    ; caps_lock กลับ shift state สำหรับตัวอักษร
    ; (เฉพาะตัวอักษร a-z, A-Z)
    test cl, cl
    jz .check_caps
    
    ; Shift กดอยู่ -> ใช้ upper table
    mov al, [scancode_table_upper + ebx]
    jmp .done
    
.check_caps:
    ; ตรวจสอบ caps lock
    mov cl, [caps_lock]
    test cl, cl
    jz .use_lower
    
    ; Caps lock เปิด -> ใช้ upper table สำหรับตัวอักษร
    mov al, [scancode_table_upper + ebx]
    ; ถ้าไม่ใช่ตัวอักษร ให้ใช้ lower
    cmp al, 'A'
    jb .use_lower
    cmp al, 'Z'
    ja .use_lower
    jmp .done
    
.use_lower:
    mov al, [scancode_table_lower + ebx]
    jmp .done
    
.no_char:
    xor al, al
    
.done:
    ; ถ้า Ctrl กดอยู่ ปรับ ASCII เป็น control character
    push eax
    mov cl, [ctrl_pressed]
    test cl, cl
    pop eax
    jz .return
    
    ; Ctrl+A = 0x01, Ctrl+B = 0x02, ..., Ctrl+Z = 0x1A
    cmp al, 'a'
    jb .return
    cmp al, 'z'
    ja .return
    sub al, 'a' - 1     ; แปลงเป็น control character
    
.return:
    pop ecx
    pop ebx
    ret

; -----------------------------------------------------------
; kb_buffer_put - ใส่ character เข้า keyboard buffer
; Input: AL = character
; -----------------------------------------------------------
kb_buffer_put:
    push ebx
    push ecx
    
    movzx ebx, word [kb_head]
    mov [kb_buffer + ebx], al
    
    ; เลื่อน head pointer (circular)
    inc bx
    and bx, 255             ; % 256
    
    ; ตรวจสอบว่า buffer เต็มหรือยัง
    cmp bx, [kb_tail]
    je .buffer_full
    
    mov [kb_head], bx       ; อัปเดต head
    jmp .done
    
.buffer_full:
    ; Buffer เต็ม: ไม่ใส่ข้อมูล (drop character)
    
.done:
    pop ecx
    pop ebx
    ret

; -----------------------------------------------------------
; kb_buffer_get - ดึง character จาก keyboard buffer
; Output: AL = character (0 = buffer ว่าง)
;         ZF = 1 ถ้า buffer ว่าง
; -----------------------------------------------------------
global kb_buffer_get
kb_buffer_get:
    push ebx
    
    movzx ebx, word [kb_tail]
    cmp bx, [kb_head]
    je .empty               ; tail == head = ว่าง
    
    mov al, [kb_buffer + ebx]
    
    inc bx
    and bx, 255
    mov [kb_tail], bx
    
    test al, al             ; set flags
    pop ebx
    ret
    
.empty:
    xor al, al              ; ZF = 1
    pop ebx
    ret

; -----------------------------------------------------------
; kb_wait_for_key - รอจนกว่าจะมี key press
; Output: AL = ASCII character
; -----------------------------------------------------------
global kb_wait_for_key
kb_wait_for_key:
.wait:
    call kb_buffer_get
    jz .wait                ; ถ้า ZF=1 (buffer ว่าง) รอต่อ
    ret

; -----------------------------------------------------------
; keyboard_init - Initialize keyboard controller
; -----------------------------------------------------------
global keyboard_init
keyboard_init:
    ; รอให้ keyboard controller ว่าง
    call kbd_wait_input
    
    ; ส่งคำสั่ง enable keyboard (0xAE)
    mov al, 0xAE
    out KBD_CMD_PORT, al
    
    ; เปิดใช้ IRQ1
    mov al, 1               ; IRQ1
    call pic_enable_irq
    ret

; รอให้ input buffer ว่าง
kbd_wait_input:
    push eax
.wait:
    in al, KBD_STATUS_PORT
    test al, KBD_STATUS_IBF
    jnz .wait               ; รอถ้า buffer เต็ม
    pop eax
    ret

; รอให้ output buffer มีข้อมูล
kbd_wait_output:
    push eax
.wait:
    in al, KBD_STATUS_PORT
    test al, KBD_STATUS_OBF
    jz .wait                ; รอถ้า buffer ว่าง
    pop eax
    ret
```

---

## 4. PIT (8253/8254) Driver

### 4.1 Programmable Interval Timer

```nasm
; ============================================================
; pit_driver.asm - PIT 8253/8254 Driver
; ============================================================
; PIT มี 3 channels:
; Channel 0: IRQ0 (system timer)
; Channel 1: DRAM refresh (obsolete)
; Channel 2: PC Speaker

section .data

; -----------------------------------------------------------
; Timer callback table (ลงทะเบียน callback สำหรับ timer)
; -----------------------------------------------------------
%define MAX_TIMER_CALLBACKS 8

timer_callbacks:
    times MAX_TIMER_CALLBACKS dd 0  ; function pointers
timer_intervals:
    times MAX_TIMER_CALLBACKS dd 0  ; interval ใน ticks
timer_counters:
    times MAX_TIMER_CALLBACKS dd 0  ; counter ปัจจุบัน

tick_count: dd 0        ; จำนวน timer ticks ทั้งหมด

section .text

; PIT Ports
%define PIT_CH0     0x40    ; Channel 0 data port
%define PIT_CH1     0x41    ; Channel 1 data port
%define PIT_CH2     0x42    ; Channel 2 data port
%define PIT_CMD     0x43    ; Mode/Command register

; PIT Oscillator frequency
%define PIT_BASE_FREQ   1193182     ; ~1.193182 MHz

; -----------------------------------------------------------
; pit_set_frequency - ตั้งความถี่ Channel 0
; Input: EAX = ความถี่ที่ต้องการ (Hz)
; -----------------------------------------------------------
global pit_set_frequency
pit_set_frequency:
    push edx
    push ebx
    
    ; คำนวณ divisor: divisor = PIT_BASE_FREQ / frequency
    mov ebx, eax            ; เก็บ frequency
    mov eax, PIT_BASE_FREQ
    xor edx, edx
    div ebx                 ; EAX = divisor
    
    ; ส่งคำสั่งตั้งค่า Channel 0
    ; Mode/Command byte:
    ; Bit 7-6: 00 = Channel 0
    ; Bit 5-4: 11 = Access lobyte/hibyte
    ; Bit 3-1: 011 = Mode 3 (Square wave)
    ; Bit 0:   0 = Binary counting
    mov bl, al              ; เก็บ divisor low byte
    mov bh, ah              ; เก็บ divisor high byte
    
    mov al, 0x36            ; 00 110 110: Ch0, lo/hi, mode3, binary
    out PIT_CMD, al
    
    mov al, bl              ; ส่ง low byte ก่อน
    out PIT_CH0, al
    
    mov al, bh              ; ส่ง high byte
    out PIT_CH0, al
    
    pop ebx
    pop edx
    ret

; -----------------------------------------------------------
; pit_init - Initialize PIT ที่ 100 Hz (10ms per tick)
; -----------------------------------------------------------
global pit_init
pit_init:
    mov eax, 100            ; 100 Hz
    call pit_set_frequency
    
    ; เปิด IRQ0
    mov al, 0
    call pic_enable_irq
    ret

; -----------------------------------------------------------
; pit_read_count - อ่าน current count ของ Channel 0
; Output: AX = current count value
; -----------------------------------------------------------
global pit_read_count
pit_read_count:
    ; Latch command: อ่านค่าปัจจุบัน
    mov al, 0x00            ; 00 000000: Channel 0, latch count
    out PIT_CMD, al
    
    in al, PIT_CH0          ; อ่าน low byte
    mov ah, al
    in al, PIT_CH0          ; อ่าน high byte
    xchg al, ah             ; AX = count (high:low)
    ret

; -----------------------------------------------------------
; timer_handler - IRQ0 Timer Interrupt Handler
; -----------------------------------------------------------
global timer_handler
timer_handler:
    push eax
    push ebx
    push ecx
    push edx
    
    ; เพิ่ม tick counter
    inc dword [tick_count]
    
    ; ตรวจสอบและเรียก registered callbacks
    xor ecx, ecx
.check_callbacks:
    cmp ecx, MAX_TIMER_CALLBACKS
    jge .done_callbacks
    
    ; ตรวจสอบว่ามี callback
    mov eax, [timer_callbacks + ecx*4]
    test eax, eax
    jz .next_callback
    
    ; ลด counter
    dec dword [timer_counters + ecx*4]
    jnz .next_callback
    
    ; Reset counter
    mov ebx, [timer_intervals + ecx*4]
    mov [timer_counters + ecx*4], ebx
    
    ; เรียก callback
    push ecx
    call eax                ; เรียก callback function
    pop ecx
    
.next_callback:
    inc ecx
    jmp .check_callbacks
    
.done_callbacks:
    ; ส่ง EOI
    mov al, 0x20
    out 0x20, al
    
    pop edx
    pop ecx
    pop ebx
    pop eax
    iret

; -----------------------------------------------------------
; timer_register_callback - ลงทะเบียน timer callback
; Input:  EAX = function pointer
;         EBX = interval in ticks
; Output: EAX = callback ID (-1 ถ้าเต็ม)
; -----------------------------------------------------------
global timer_register_callback
timer_register_callback:
    push ecx
    
    xor ecx, ecx
.find_slot:
    cmp ecx, MAX_TIMER_CALLBACKS
    jge .no_slot
    
    cmp dword [timer_callbacks + ecx*4], 0
    je .found_slot
    
    inc ecx
    jmp .find_slot
    
.found_slot:
    mov [timer_callbacks + ecx*4], eax
    mov [timer_intervals + ecx*4], ebx
    mov [timer_counters + ecx*4], ebx
    mov eax, ecx            ; return ID
    pop ecx
    ret
    
.no_slot:
    mov eax, -1
    pop ecx
    ret

; -----------------------------------------------------------
; timer_unregister_callback - ยกเลิก callback
; Input: EAX = callback ID
; -----------------------------------------------------------
global timer_unregister_callback
timer_unregister_callback:
    cmp eax, MAX_TIMER_CALLBACKS
    jge .done
    
    mov dword [timer_callbacks + eax*4], 0
    mov dword [timer_intervals + eax*4], 0
    mov dword [timer_counters + eax*4], 0
    
.done:
    ret

; -----------------------------------------------------------
; pit_sleep - รอ n milliseconds
; Input: EAX = milliseconds
; -----------------------------------------------------------
global pit_sleep
pit_sleep:
    push ebx
    
    ; แปลงเป็น ticks (100 Hz = 10ms per tick)
    ; ticks = ms / 10
    mov ebx, 10
    xor edx, edx
    div ebx                 ; EAX = ticks
    
    ; เพิ่ม target tick
    add eax, [tick_count]
    
.wait:
    cmp [tick_count], eax
    jl .wait
    
    pop ebx
    ret
```

---

## 5. VGA Text Mode Driver

### 5.1 VGA Text Mode 80x25

```nasm
; ============================================================
; vga_driver.asm - VGA Text Mode Driver
; ============================================================
; VGA Text Mode: 80 columns x 25 rows
; Video memory: 0xB8000
; Each cell: 2 bytes (character byte + attribute byte)
; Attribute: bits 7-4 = background color, bits 3-0 = foreground color

section .data

; -----------------------------------------------------------
; VGA Color Constants (4-bit color)
; -----------------------------------------------------------
%define VGA_BLACK           0x0
%define VGA_BLUE            0x1
%define VGA_GREEN           0x2
%define VGA_CYAN            0x3
%define VGA_RED             0x4
%define VGA_MAGENTA         0x5
%define VGA_BROWN           0x6
%define VGA_LIGHT_GREY      0x7
%define VGA_DARK_GREY       0x8
%define VGA_LIGHT_BLUE      0x9
%define VGA_LIGHT_GREEN     0xA
%define VGA_LIGHT_CYAN      0xB
%define VGA_LIGHT_RED       0xC
%define VGA_LIGHT_MAGENTA   0xD
%define VGA_YELLOW          0xE
%define VGA_WHITE           0xF

; VGA เมมโมรีและขนาด
%define VGA_MEMORY      0xB8000
%define VGA_WIDTH       80
%define VGA_HEIGHT      25
%define VGA_SIZE        (VGA_WIDTH * VGA_HEIGHT * 2)

; VGA CRT Controller ports (hardware cursor)
%define VGA_CTRL_REG    0x3D4   ; Index register
%define VGA_DATA_REG    0x3D5   ; Data register

; Current cursor position
vga_cursor_x:   db 0
vga_cursor_y:   db 0

; Current text color
vga_color:      db 0x0F     ; White on Black (default)

; -----------------------------------------------------------
; Format string buffer สำหรับ printf
; -----------------------------------------------------------
printf_buffer:  times 256 db 0

section .text

; -----------------------------------------------------------
; vga_make_color - สร้าง color attribute byte
; Input:  AL = foreground color (0-15)
;         AH = background color (0-15)
; Output: AL = color attribute byte
; -----------------------------------------------------------
global vga_make_color
vga_make_color:
    ; attribute = (background << 4) | foreground
    shl ah, 4
    or al, ah
    ret

; -----------------------------------------------------------
; vga_putchar - พิมพ์ character หนึ่งตัว
; Input: AL = character
;        BL = color attribute
;        CL = column (x)
;        CH = row (y)
; -----------------------------------------------------------
global vga_putchar_at
vga_putchar_at:
    push edx
    push edi
    
    ; คำนวณ offset ใน video memory
    ; offset = (row * VGA_WIDTH + col) * 2
    movzx edi, ch           ; row
    imul edi, VGA_WIDTH
    movzx edx, cl           ; col
    add edi, edx
    shl edi, 1              ; * 2 (2 bytes per cell)
    add edi, VGA_MEMORY     ; base address
    
    mov [edi], al           ; เขียน character
    mov [edi+1], bl         ; เขียน attribute
    
    pop edi
    pop edx
    ret

; -----------------------------------------------------------
; vga_putchar - พิมพ์ character ที่ cursor ปัจจุบัน
; Input: AL = character
; -----------------------------------------------------------
global vga_putchar
vga_putchar:
    push ebx
    push ecx
    
    mov cl, [vga_cursor_x]
    mov ch, [vga_cursor_y]
    mov bl, [vga_color]
    
    ; จัดการ special characters
    cmp al, 10              ; newline '\n'
    je .newline
    cmp al, 13              ; carriage return '\r'
    je .carriage_return
    cmp al, 8               ; backspace
    je .backspace
    cmp al, 9               ; tab
    je .tab
    
    ; พิมพ์ character ปกติ
    call vga_putchar_at
    
    ; เลื่อน cursor ไปขวา
    inc cl
    cmp cl, VGA_WIDTH
    jl .update_cursor
    
    ; ถึงขอบขวา: ขึ้นบรรทัดใหม่
    xor cl, cl
    inc ch
    jmp .check_scroll
    
.newline:
    xor cl, cl
    inc ch
    jmp .check_scroll
    
.carriage_return:
    xor cl, cl
    jmp .update_cursor
    
.backspace:
    test cl, cl
    jz .update_cursor
    dec cl
    ; ลบ character
    push eax
    mov al, ' '
    call vga_putchar_at
    pop eax
    jmp .update_cursor
    
.tab:
    ; Tab = 4 spaces (หรือ align to next 4-boundary)
    mov al, cl
    and al, 0xFC            ; round down to 4
    add al, 4               ; next tab stop
    mov cl, al
    cmp cl, VGA_WIDTH
    jl .update_cursor
    xor cl, cl
    inc ch
    jmp .check_scroll
    
.check_scroll:
    cmp ch, VGA_HEIGHT
    jl .update_cursor
    ; Scroll screen
    call vga_scroll
    dec ch
    
.update_cursor:
    mov [vga_cursor_x], cl
    mov [vga_cursor_y], ch
    call vga_update_cursor
    
    pop ecx
    pop ebx
    ret

; -----------------------------------------------------------
; vga_puts - พิมพ์ null-terminated string
; Input: ESI = pointer to string
; -----------------------------------------------------------
global vga_puts
vga_puts:
    push eax
    
.loop:
    mov al, [esi]
    test al, al
    jz .done
    call vga_putchar
    inc esi
    jmp .loop
    
.done:
    pop eax
    ret

; -----------------------------------------------------------
; vga_printf - printf แบบง่าย
; Input: ESI = format string
;        Stack = arguments
; รองรับ: %c, %s, %d, %x, %u
; -----------------------------------------------------------
global vga_printf
vga_printf:
    push ebp
    mov ebp, esp
    push esi
    push edi
    push eax
    push ebx
    push ecx
    push edx
    
    ; อ้างอิง argument แรกหลังจาก format string
    ; ESP หลังจาก push = [ebp-24]
    ; format string = ESI (ส่งผ่าน register ตามที่กำหนด)
    ; arguments อยู่ที่ [ebp+8], [ebp+12], ...
    lea edi, [ebp+8]        ; pointer ไปยัง arguments
    
.format_loop:
    mov al, [esi]
    test al, al
    jz .done
    inc esi
    
    cmp al, '%'
    jne .normal_char
    
    ; ประมวลผล format specifier
    mov al, [esi]
    inc esi
    
    cmp al, 'c'
    je .fmt_char
    cmp al, 's'
    je .fmt_string
    cmp al, 'd'
    je .fmt_decimal
    cmp al, 'x'
    je .fmt_hex
    cmp al, 'u'
    je .fmt_unsigned
    cmp al, '%'
    je .fmt_percent
    jmp .normal_char        ; ไม่รู้จัก: พิมพ์ตามปกติ
    
.fmt_char:
    mov eax, [edi]
    add edi, 4
    call vga_putchar
    jmp .format_loop
    
.fmt_string:
    push esi
    mov esi, [edi]
    add edi, 4
    call vga_puts
    pop esi
    jmp .format_loop
    
.fmt_decimal:
    mov eax, [edi]
    add edi, 4
    call vga_print_int
    jmp .format_loop
    
.fmt_hex:
    mov eax, [edi]
    add edi, 4
    call vga_print_hex
    jmp .format_loop
    
.fmt_unsigned:
    mov eax, [edi]
    add edi, 4
    call vga_print_uint
    jmp .format_loop
    
.fmt_percent:
    mov al, '%'
    
.normal_char:
    call vga_putchar
    jmp .format_loop
    
.done:
    pop edx
    pop ecx
    pop ebx
    pop eax
    pop edi
    pop esi
    pop ebp
    ret

; -----------------------------------------------------------
; vga_print_int - พิมพ์จำนวนเต็มมีเครื่องหมาย (signed)
; Input: EAX = integer
; -----------------------------------------------------------
global vga_print_int
vga_print_int:
    push eax
    push esi
    
    ; ตรวจสอบว่าเป็นลบ
    test eax, eax
    jns .positive
    
    ; พิมพ์ '-'
    push eax
    mov al, '-'
    call vga_putchar
    pop eax
    neg eax                 ; แปลงเป็นบวก
    
.positive:
    call vga_print_uint
    
    pop esi
    pop eax
    ret

; -----------------------------------------------------------
; vga_print_uint - พิมพ์จำนวนเต็มไม่มีเครื่องหมาย
; Input: EAX = unsigned integer
; -----------------------------------------------------------
global vga_print_uint
vga_print_uint:
    push eax
    push ebx
    push ecx
    push edx
    
    ; แปลง integer เป็น decimal string (reverse order)
    mov ecx, 0              ; digit count
    mov ebx, 10             ; divisor
    
    ; กรณีพิเศษ: EAX = 0
    test eax, eax
    jnz .convert_loop
    push 0x30               ; '0'
    inc ecx
    jmp .print_digits
    
.convert_loop:
    test eax, eax
    jz .print_digits
    
    xor edx, edx
    div ebx                 ; EAX = quotient, EDX = remainder
    add edx, 0x30           ; แปลงเป็น ASCII
    push edx                ; เก็บบน stack (reverse order)
    inc ecx
    jmp .convert_loop
    
.print_digits:
    test ecx, ecx
    jz .done
    pop eax
    call vga_putchar
    dec ecx
    jmp .print_digits
    
.done:
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

; -----------------------------------------------------------
; vga_print_hex - พิมพ์ hexadecimal
; Input: EAX = value
; -----------------------------------------------------------
global vga_print_hex
vga_print_hex:
    push eax
    push ebx
    push ecx
    
    ; พิมพ์ "0x" prefix
    push eax
    mov al, '0'
    call vga_putchar
    mov al, 'x'
    call vga_putchar
    pop eax
    
    ; พิมพ์ 8 hex digits
    mov ecx, 8
    rol eax, 4              ; เริ่มจาก nibble สูงสุด
    
.loop:
    push eax
    and al, 0x0F
    add al, '0'
    cmp al, '9'
    jle .print_digit
    add al, 'A' - '0' - 10  ; แปลงเป็น A-F
    
.print_digit:
    call vga_putchar
    pop eax
    rol eax, 4
    dec ecx
    jnz .loop
    
    pop ecx
    pop ebx
    pop eax
    ret

; -----------------------------------------------------------
; vga_clear_screen - ล้างหน้าจอ
; -----------------------------------------------------------
global vga_clear_screen
vga_clear_screen:
    push eax
    push ecx
    push edi
    
    mov edi, VGA_MEMORY
    movzx eax, byte [vga_color]
    shl eax, 8
    or ax, ' '              ; attribute:space
    mov ecx, VGA_WIDTH * VGA_HEIGHT
    
    ; ใช้ REP STOSW เพื่อล้างหน้าจออย่างรวดเร็ว
    rep stosw
    
    ; Reset cursor
    mov byte [vga_cursor_x], 0
    mov byte [vga_cursor_y], 0
    call vga_update_cursor
    
    pop edi
    pop ecx
    pop eax
    ret

; -----------------------------------------------------------
; vga_scroll - เลื่อนหน้าจอขึ้น 1 บรรทัด
; -----------------------------------------------------------
global vga_scroll
vga_scroll:
    push eax
    push ecx
    push edi
    push esi
    
    ; ย้าย rows 1-24 ไปยัง rows 0-23
    mov edi, VGA_MEMORY
    mov esi, VGA_MEMORY + VGA_WIDTH * 2     ; row 1
    mov ecx, (VGA_HEIGHT - 1) * VGA_WIDTH   ; จำนวน words
    rep movsw
    
    ; ล้าง row ล่างสุด
    mov edi, VGA_MEMORY + (VGA_HEIGHT - 1) * VGA_WIDTH * 2
    movzx eax, byte [vga_color]
    shl eax, 8
    or ax, ' '
    mov ecx, VGA_WIDTH
    rep stosw
    
    pop esi
    pop edi
    pop ecx
    pop eax
    ret

; -----------------------------------------------------------
; vga_update_cursor - อัปเดต hardware cursor position
; -----------------------------------------------------------
global vga_update_cursor
vga_update_cursor:
    push eax
    push edx
    
    ; คำนวณ linear position
    movzx eax, byte [vga_cursor_y]
    imul eax, VGA_WIDTH
    movzx edx, byte [vga_cursor_x]
    add eax, edx            ; position = y * width + x
    
    ; ส่งค่า cursor low byte
    mov dx, VGA_CTRL_REG
    mov al, 0x0F            ; Cursor Location Low register
    out dx, al
    
    mov dx, VGA_DATA_REG
    movzx eax, byte [vga_cursor_x]
    movzx edx, byte [vga_cursor_y]
    imul edx, VGA_WIDTH
    add eax, edx
    out 0x3D5, al           ; low byte
    
    ; ส่งค่า cursor high byte
    mov dx, VGA_CTRL_REG
    mov al, 0x0E            ; Cursor Location High register
    out dx, al
    
    mov dx, VGA_DATA_REG
    movzx eax, byte [vga_cursor_x]
    movzx edx, byte [vga_cursor_y]
    imul edx, VGA_WIDTH
    add eax, edx
    shr eax, 8
    out 0x3D5, al           ; high byte
    
    pop edx
    pop eax
    ret

; -----------------------------------------------------------
; vga_set_color - กำหนด text color
; Input: AL = color attribute byte
; -----------------------------------------------------------
global vga_set_color
vga_set_color:
    mov [vga_color], al
    ret

; -----------------------------------------------------------
; vga_set_cursor - กำหนดตำแหน่ง cursor
; Input: CL = x, CH = y
; -----------------------------------------------------------
global vga_set_cursor
vga_set_cursor:
    mov [vga_cursor_x], cl
    mov [vga_cursor_y], ch
    call vga_update_cursor
    ret
```

---

## 6. Serial UART (16550) Driver

### 6.1 Universal Asynchronous Receiver/Transmitter

```nasm
; ============================================================
; uart_driver.asm - Serial UART 16550 Driver
; ============================================================
; COM1: I/O base = 0x3F8
; COM2: I/O base = 0x2F8
; COM3: I/O base = 0x3E8
; COM4: I/O base = 0x2E8

section .data

; UART Register offsets (relative to base port)
%define UART_RX         0   ; Receive buffer (read, DLAB=0)
%define UART_TX         0   ; Transmit buffer (write, DLAB=0)
%define UART_DLL        0   ; Divisor Latch Low (DLAB=1)
%define UART_IER        1   ; Interrupt Enable Register (DLAB=0)
%define UART_DLH        1   ; Divisor Latch High (DLAB=1)
%define UART_IIR        2   ; Interrupt Identification Register (read)
%define UART_FCR        2   ; FIFO Control Register (write)
%define UART_LCR        3   ; Line Control Register
%define UART_MCR        4   ; Modem Control Register
%define UART_LSR        5   ; Line Status Register
%define UART_MSR        6   ; Modem Status Register
%define UART_SR         7   ; Scratch Register

; Line Control Register bits
%define UART_LCR_DLAB   0x80    ; Divisor Latch Access Bit
%define UART_LCR_8N1    0x03    ; 8 data bits, no parity, 1 stop bit

; Line Status Register bits
%define UART_LSR_DR     0x01    ; Data Ready
%define UART_LSR_THRE   0x20    ; Transmit Holding Register Empty

; Interrupt Enable Register bits
%define UART_IER_RX     0x01    ; Enable Received Data Available Interrupt
%define UART_IER_TX     0x02    ; Enable Transmit Holding Register Empty Interrupt

; FIFO Control Register bits
%define UART_FCR_ENABLE 0x01    ; Enable FIFO
%define UART_FCR_CLEAR_RX 0x02  ; Clear Receive FIFO
%define UART_FCR_CLEAR_TX 0x04  ; Clear Transmit FIFO
%define UART_FCR_14     0xC0    ; Interrupt trigger level 14 bytes

; COM1 base port
%define COM1_BASE   0x3F8

; UART receive buffer
uart_rx_buffer: times 256 db 0
uart_rx_head:   dw 0
uart_rx_tail:   dw 0

section .text

; -----------------------------------------------------------
; uart_init - Initialize UART
; Input:  DX = base port (e.g. 0x3F8 for COM1)
;         EAX = baud rate (e.g. 9600, 115200)
; -----------------------------------------------------------
global uart_init
uart_init:
    push ebx
    push ecx
    
    mov ecx, edx            ; เก็บ base port
    
    ; คำนวณ baud rate divisor
    ; divisor = 115200 / baud_rate
    mov ebx, eax
    mov eax, 115200
    xor edx, edx
    div ebx                 ; EAX = divisor
    push eax                ; เก็บ divisor
    
    ; ปิด interrupts ก่อน
    mov dx, cx
    add dx, UART_IER
    xor al, al
    out dx, al
    
    ; ตั้งค่า DLAB bit เพื่อตั้ง baud rate
    mov dx, cx
    add dx, UART_LCR
    mov al, UART_LCR_DLAB
    out dx, al
    
    ; ส่ง divisor low byte
    pop eax                 ; ดึง divisor
    push eax
    mov dx, cx
    add dx, UART_DLL
    out dx, al              ; low byte
    
    ; ส่ง divisor high byte
    pop eax
    shr eax, 8
    mov dx, cx
    add dx, UART_DLH
    out dx, al              ; high byte
    
    ; ตั้ง line format: 8N1 (8 data bits, no parity, 1 stop bit)
    mov dx, cx
    add dx, UART_LCR
    mov al, UART_LCR_8N1    ; 0x03
    out dx, al
    
    ; เปิดใช้งาน FIFO, clear FIFOs, trigger level 14 bytes
    mov dx, cx
    add dx, UART_FCR
    mov al, UART_FCR_ENABLE | UART_FCR_CLEAR_RX | UART_FCR_CLEAR_TX | UART_FCR_14
    out dx, al
    
    ; เปิด DTR, RTS, OUT2 (OUT2 ต้องเปิดเพื่อให้ IRQ ทำงาน)
    mov dx, cx
    add dx, UART_MCR
    mov al, 0x0B            ; DTR | RTS | OUT2
    out dx, al
    
    ; เปิด Receive interrupt
    mov dx, cx
    add dx, UART_IER
    mov al, UART_IER_RX     ; 0x01
    out dx, al
    
    pop ecx
    pop ebx
    ret

; -----------------------------------------------------------
; uart_send_byte - ส่ง 1 byte ผ่าน UART
; Input:  DX = base port
;         AL = byte to send
; -----------------------------------------------------------
global uart_send_byte
uart_send_byte:
    push eax
    push ebx
    push edx
    
    mov bl, al              ; เก็บ data
    
    ; รอให้ Transmit Holding Register ว่าง
.wait_tx:
    add dx, UART_LSR        ; LSR port
    in al, dx
    test al, UART_LSR_THRE  ; ตรวจสอบ THRE bit
    sub dx, UART_LSR        ; กลับไปยัง base port
    jz .wait_tx
    
    ; ส่ง byte
    mov al, bl
    out dx, al              ; TX register = base + 0
    
    pop edx
    pop ebx
    pop eax
    ret

; -----------------------------------------------------------
; uart_send_string - ส่ง null-terminated string
; Input:  DX = base port
;         ESI = string pointer
; -----------------------------------------------------------
global uart_send_string
uart_send_string:
    push eax
    
.loop:
    mov al, [esi]
    test al, al
    jz .done
    call uart_send_byte
    inc esi
    jmp .loop
    
.done:
    pop eax
    ret

; -----------------------------------------------------------
; uart_recv_byte - รับ 1 byte จาก UART (non-blocking)
; Input:  DX = base port
; Output: AL = received byte
;         ZF = 1 ถ้าไม่มีข้อมูล
; -----------------------------------------------------------
global uart_recv_byte
uart_recv_byte:
    push edx
    
    add dx, UART_LSR
    in al, dx
    sub dx, UART_LSR
    
    test al, UART_LSR_DR    ; ตรวจสอบ Data Ready bit
    jz .no_data
    
    in al, dx               ; อ่าน byte จาก RX buffer
    test al, al             ; set flags
    pop edx
    ret
    
.no_data:
    xor al, al              ; ZF = 1
    pop edx
    ret

; -----------------------------------------------------------
; uart_recv_byte_blocking - รับ byte (blocking)
; Input:  DX = base port
; Output: AL = received byte
; -----------------------------------------------------------
global uart_recv_byte_blocking
uart_recv_byte_blocking:
.wait:
    call uart_recv_byte
    jz .wait
    ret

; -----------------------------------------------------------
; uart_handler - Serial IRQ Handler (IRQ3 หรือ IRQ4)
; -----------------------------------------------------------
global uart_handler
uart_handler:
    push eax
    push edx
    
    mov dx, COM1_BASE
    
.read_loop:
    ; อ่านจนกว่า FIFO จะว่าง
    call uart_recv_byte
    jz .done_reading
    
    ; ใส่เข้า receive buffer
    push eax
    mov eax, [uart_rx_head]
    push eax
    pop eax                 ; dummy (เพื่อให้โค้ดกระชับ)
    ; ใส่เข้า circular buffer
    push edx
    movzx edx, word [uart_rx_head]
    mov [uart_rx_buffer + edx], al
    inc dx
    and dx, 255
    mov [uart_rx_head], dx
    pop edx
    pop eax
    
    jmp .read_loop
    
.done_reading:
    ; ส่ง EOI (IRQ4 = COM1)
    mov al, 0x20
    out 0x20, al
    
    pop edx
    pop eax
    iret
```

---

## 7. ATA PIO Mode Driver

### 7.1 ATA/IDE Hard Drive Interface

```nasm
; ============================================================
; ata_driver.asm - ATA PIO Mode Driver
; ============================================================
; Primary ATA: ports 0x1F0-0x1F7, IRQ14
; Secondary ATA: ports 0x170-0x177, IRQ15
; เป็น Polling mode (ไม่ใช้ interrupt)

section .data

; Primary ATA Controller ports
%define ATA_DATA        0x1F0   ; Data register (16-bit)
%define ATA_ERROR       0x1F1   ; Error register (read) / Features (write)
%define ATA_SECTOR_COUNT 0x1F2  ; Sector count
%define ATA_LBA_LOW     0x1F3   ; LBA bits 0-7
%define ATA_LBA_MID     0x1F4   ; LBA bits 8-15
%define ATA_LBA_HIGH    0x1F5   ; LBA bits 16-23
%define ATA_DRIVE_HEAD  0x1F6   ; Drive/Head register
%define ATA_STATUS      0x1F7   ; Status register (read)
%define ATA_COMMAND     0x1F7   ; Command register (write)

; Alt Status / Device Control
%define ATA_ALT_STATUS  0x3F6   ; Alternate Status (read)
%define ATA_DEV_CTRL    0x3F6   ; Device Control (write)

; ATA Status Register bits
%define ATA_SR_BSY      0x80    ; Busy
%define ATA_SR_DRDY     0x40    ; Drive Ready
%define ATA_SR_DF       0x20    ; Drive Fault
%define ATA_SR_DSC      0x10    ; Drive Seek Complete
%define ATA_SR_DRQ      0x08    ; Data Request Ready
%define ATA_SR_CORR     0x04    ; Corrected Data
%define ATA_SR_IDX      0x02    ; Index
%define ATA_SR_ERR      0x01    ; Error

; ATA Error Register bits
%define ATA_ER_BBK      0x80    ; Bad Block
%define ATA_ER_UNC      0x40    ; Uncorrectable Data
%define ATA_ER_MC       0x20    ; Media Changed
%define ATA_ER_IDNF     0x10    ; ID mark Not Found
%define ATA_ER_MCR      0x08    ; Media Change Request
%define ATA_ER_ABRT     0x04    ; Command Aborted
%define ATA_ER_TK0NF    0x02    ; Track 0 Not Found
%define ATA_ER_AMNF     0x01    ; Address Mark Not Found

; ATA Commands
%define ATA_CMD_READ_PIO    0x20    ; Read Sectors (with retry)
%define ATA_CMD_WRITE_PIO   0x30    ; Write Sectors (with retry)
%define ATA_CMD_IDENTIFY    0xEC    ; Identify Drive

; Drive/Head register bits
%define ATA_HEAD_LBA    0x40    ; LBA mode
%define ATA_HEAD_MASTER 0xA0    ; Select Master drive (0xA0 | 0x40 for LBA)
%define ATA_HEAD_SLAVE  0xB0    ; Select Slave drive

; ข้อมูล identify drive
ata_identify_data: times 512 db 0

section .text

; -----------------------------------------------------------
; ata_wait_bsy - รอให้ BSY bit clear
; -----------------------------------------------------------
ata_wait_bsy:
    push eax
.wait:
    in al, ATA_STATUS
    test al, ATA_SR_BSY
    jnz .wait
    pop eax
    ret

; -----------------------------------------------------------
; ata_wait_drq - รอให้ DRQ bit set (ข้อมูลพร้อม)
; Output: ZF = 0 ถ้าสำเร็จ, ZF = 1 ถ้า error
; -----------------------------------------------------------
ata_wait_drq:
    push ecx
    
    mov ecx, 100000         ; timeout counter
.wait:
    in al, ATA_STATUS
    
    test al, ATA_SR_ERR     ; ตรวจสอบ error
    jnz .error
    test al, ATA_SR_DRQ     ; ตรวจสอบ data ready
    jnz .ready
    
    dec ecx
    jnz .wait
    
.error:
    xor al, al              ; ZF = 1 (error)
    pop ecx
    ret
    
.ready:
    or al, 1                ; ZF = 0 (success)
    pop ecx
    ret

; -----------------------------------------------------------
; ata_identify - รับข้อมูล drive
; Input:  AL = drive (0 = Master, 1 = Slave)
; Output: EAX = 0 ถ้าสำเร็จ, -1 ถ้า error
; -----------------------------------------------------------
global ata_identify
ata_identify:
    push ebx
    push ecx
    push edx
    push edi
    
    ; เลือก drive
    test al, al
    jz .select_master
    mov al, ATA_HEAD_SLAVE  ; 0xB0
    jmp .send_select
.select_master:
    mov al, ATA_HEAD_MASTER  ; 0xA0
.send_select:
    out ATA_DRIVE_HEAD, al
    
    ; ส่ง 0 ไปยัง sector count, LBA registers
    xor al, al
    out ATA_SECTOR_COUNT, al
    out ATA_LBA_LOW, al
    out ATA_LBA_MID, al
    out ATA_LBA_HIGH, al
    
    ; ส่งคำสั่ง IDENTIFY
    mov al, ATA_CMD_IDENTIFY
    out ATA_COMMAND, al
    
    ; รอ
    in al, ATA_STATUS
    test al, al
    jz .no_drive            ; ถ้า status = 0 ไม่มี drive
    
    call ata_wait_bsy
    
    ; ตรวจสอบว่า LBA registers เปลี่ยนค่าหรือไม่
    ; (อาจเป็น ATAPI device)
    in al, ATA_LBA_MID
    cmp al, 0
    jne .atapi_device
    in al, ATA_LBA_HIGH
    cmp al, 0
    jne .atapi_device
    
    call ata_wait_drq
    jz .error
    
    ; อ่าน 256 words (512 bytes) ของ identify data
    mov edi, ata_identify_data
    mov ecx, 256
    mov dx, ATA_DATA
    rep insw
    
    xor eax, eax            ; success
    jmp .done
    
.no_drive:
.atapi_device:
.error:
    mov eax, -1
    
.done:
    pop edi
    pop edx
    pop ecx
    pop ebx
    ret

; -----------------------------------------------------------
; ata_read_sector - อ่าน 1 sector จาก ATA drive (LBA28)
; Input:  EAX = LBA address
;         EDI = destination buffer (512 bytes)
;         BL  = drive (0 = Master, 1 = Slave)
; Output: EAX = 0 ถ้าสำเร็จ, -1 ถ้า error
; -----------------------------------------------------------
global ata_read_sector
ata_read_sector:
    push ecx
    push edx
    push esi
    push edi
    push ebx
    
    mov esi, eax            ; เก็บ LBA
    
    ; รอให้ drive ว่าง
    call ata_wait_bsy
    
    ; ส่ง Drive/Head register (LBA mode)
    ; byte = 0xE0 | (drive << 4) | ((LBA >> 24) & 0x0F)
    mov al, 0xE0
    test bl, 1
    jz .master
    or al, 0x10             ; Slave bit
.master:
    mov ecx, esi
    shr ecx, 24
    and cl, 0x0F
    or al, cl
    out ATA_DRIVE_HEAD, al
    
    ; ส่ง sector count = 1
    mov al, 1
    out ATA_SECTOR_COUNT, al
    
    ; ส่ง LBA address (24 bits)
    mov al, sil             ; LBA bits 0-7
    out ATA_LBA_LOW, al
    
    mov eax, esi
    shr eax, 8
    out ATA_LBA_MID, al     ; LBA bits 8-15
    
    mov eax, esi
    shr eax, 16
    out ATA_LBA_HIGH, al    ; LBA bits 16-23
    
    ; ส่งคำสั่ง READ
    mov al, ATA_CMD_READ_PIO
    out ATA_COMMAND, al
    
    ; รอ DRQ
    call ata_wait_bsy
    call ata_wait_drq
    jz .error
    
    ; อ่าน 256 words = 512 bytes
    mov ecx, 256
    mov dx, ATA_DATA
    rep insw
    
    xor eax, eax            ; success
    jmp .done
    
.error:
    mov eax, -1
    
.done:
    pop ebx
    pop edi
    pop esi
    pop edx
    pop ecx
    ret

; -----------------------------------------------------------
; ata_write_sector - เขียน 1 sector ไปยัง ATA drive (LBA28)
; Input:  EAX = LBA address
;         ESI = source buffer (512 bytes)
;         BL  = drive (0 = Master, 1 = Slave)
; Output: EAX = 0 ถ้าสำเร็จ, -1 ถ้า error
; -----------------------------------------------------------
global ata_write_sector
ata_write_sector:
    push ecx
    push edx
    push edi
    push ebx
    
    mov edi, eax            ; เก็บ LBA
    
    ; รอให้ drive ว่าง
    call ata_wait_bsy
    
    ; ส่ง Drive/Head register
    mov al, 0xE0
    test bl, 1
    jz .master
    or al, 0x10
.master:
    mov ecx, edi
    shr ecx, 24
    and cl, 0x0F
    or al, cl
    out ATA_DRIVE_HEAD, al
    
    ; ส่ง parameters
    mov al, 1
    out ATA_SECTOR_COUNT, al
    
    mov al, dil
    out ATA_LBA_LOW, al
    
    mov eax, edi
    shr eax, 8
    out ATA_LBA_MID, al
    
    mov eax, edi
    shr eax, 16
    out ATA_LBA_HIGH, al
    
    ; ส่งคำสั่ง WRITE
    mov al, ATA_CMD_WRITE_PIO
    out ATA_COMMAND, al
    
    ; รอ DRQ
    call ata_wait_bsy
    call ata_wait_drq
    jz .error
    
    ; เขียน 256 words = 512 bytes
    mov ecx, 256
    mov dx, ATA_DATA
    rep outsw
    
    ; Flush cache (optional)
    mov al, 0xE7            ; FLUSH CACHE command
    out ATA_COMMAND, al
    call ata_wait_bsy
    
    xor eax, eax            ; success
    jmp .done
    
.error:
    mov eax, -1
    
.done:
    pop ebx
    pop edi
    pop edx
    pop ecx
    ret
```

---

## 8. Device Abstraction Layer

### 8.1 Driver Interface ด้วย Function Pointers

```nasm
; ============================================================
; device_abstraction.asm - Device Abstraction Layer
; ============================================================
; ใช้ function pointers เพื่อสร้าง uniform driver interface

section .data

; -----------------------------------------------------------
; Device Descriptor Structure
; -----------------------------------------------------------
struc device_t
    .name:          resb 32     ; device name string
    .type:          resd 1      ; device type (0=char, 1=block)
    .flags:         resd 1      ; device flags
    .init:          resd 1      ; pointer: int init(device_t*)
    .read:          resd 1      ; pointer: int read(device_t*, buf, size)
    .write:         resd 1      ; pointer: int write(device_t*, buf, size)
    .ioctl:         resd 1      ; pointer: int ioctl(device_t*, cmd, arg)
    .close:         resd 1      ; pointer: int close(device_t*)
    .private_data:  resd 1      ; driver-specific data pointer
    .size           equ $-.name ; total struct size
endstruc

; Device type constants
%define DEV_TYPE_CHAR   0       ; Character device
%define DEV_TYPE_BLOCK  1       ; Block device

; Device flags
%define DEV_FLAG_OPEN   0x01    ; Device is open
%define DEV_FLAG_RDONLY 0x02    ; Read-only
%define DEV_FLAG_BUSY   0x04    ; Device is busy

; -----------------------------------------------------------
; Device Registry (รองรับ 16 devices)
; -----------------------------------------------------------
%define MAX_DEVICES 16

device_registry:
    times MAX_DEVICES * device_t.size db 0

device_count: dd 0

; -----------------------------------------------------------
; ตัวอย่าง Keyboard Device Descriptor
; -----------------------------------------------------------
keyboard_device:
    istruc device_t
        at device_t.name,         db "keyboard", 0
        times (32-9) db 0
        at device_t.type,         dd DEV_TYPE_CHAR
        at device_t.flags,        dd 0
        at device_t.init,         dd keyboard_dev_init
        at device_t.read,         dd keyboard_dev_read
        at device_t.write,        dd keyboard_dev_write
        at device_t.ioctl,        dd keyboard_dev_ioctl
        at device_t.close,        dd keyboard_dev_close
        at device_t.private_data, dd 0
    iend

; VGA Device Descriptor
vga_device:
    istruc device_t
        at device_t.name,         db "vga_text", 0
        times (32-8) db 0
        at device_t.type,         dd DEV_TYPE_CHAR
        at device_t.flags,        dd 0
        at device_t.init,         dd vga_dev_init
        at device_t.read,         dd vga_dev_null    ; VGA ไม่มี read
        at device_t.write,        dd vga_dev_write
        at device_t.ioctl,        dd vga_dev_ioctl
        at device_t.close,        dd vga_dev_null
        at device_t.private_data, dd 0
    iend

; Serial Device Descriptor
serial_device:
    istruc device_t
        at device_t.name,         db "serial0", 0
        times (32-7) db 0
        at device_t.type,         dd DEV_TYPE_CHAR
        at device_t.flags,        dd 0
        at device_t.init,         dd serial_dev_init
        at device_t.read,         dd serial_dev_read
        at device_t.write,        dd serial_dev_write
        at device_t.ioctl,        dd serial_dev_ioctl
        at device_t.close,        dd serial_dev_close
        at device_t.private_data, dd COM1_BASE
    iend

section .text

; -----------------------------------------------------------
; device_register - ลงทะเบียน device driver
; Input:  ESI = pointer to device_t structure
; Output: EAX = device ID (-1 ถ้าเต็ม)
; -----------------------------------------------------------
global device_register
device_register:
    push ecx
    push edi
    
    ; ตรวจสอบว่า registry เต็มหรือยัง
    mov eax, [device_count]
    cmp eax, MAX_DEVICES
    jge .full
    
    ; คำนวณตำแหน่งใน registry
    mov edi, device_registry
    imul eax, device_t.size
    add edi, eax
    
    ; คัดลอก device descriptor
    mov ecx, device_t.size / 4
    rep movsd
    
    ; เรียก init function
    mov edi, device_registry
    mov eax, [device_count]
    imul eax, device_t.size
    add edi, eax
    
    mov eax, [edi + device_t.init]
    test eax, eax
    jz .no_init
    
    push edi                ; ส่ง device pointer
    call eax
    add esp, 4
    
.no_init:
    ; เพิ่ม device count
    mov eax, [device_count]
    mov ecx, eax            ; return ID
    inc dword [device_count]
    
    mov eax, ecx
    pop edi
    pop ecx
    ret
    
.full:
    mov eax, -1
    pop edi
    pop ecx
    ret

; -----------------------------------------------------------
; device_read - อ่านข้อมูลจาก device
; Input:  EAX = device ID
;         EDI = buffer pointer
;         ECX = size
; Output: EAX = bytes read (-1 ถ้า error)
; -----------------------------------------------------------
global device_read
device_read:
    push ebx
    push esi
    
    ; ตรวจสอบ device ID
    cmp eax, [device_count]
    jge .error
    
    ; หา device descriptor
    mov esi, device_registry
    imul eax, device_t.size
    add esi, eax
    
    ; เรียก read function
    mov eax, [esi + device_t.read]
    test eax, eax
    jz .error
    
    push ecx                ; size
    push edi                ; buffer
    push esi                ; device pointer
    call eax
    add esp, 12
    
    pop esi
    pop ebx
    ret
    
.error:
    mov eax, -1
    pop esi
    pop ebx
    ret

; -----------------------------------------------------------
; device_write - เขียนข้อมูลไปยัง device
; Input:  EAX = device ID
;         ESI = buffer pointer
;         ECX = size
; Output: EAX = bytes written (-1 ถ้า error)
; -----------------------------------------------------------
global device_write
device_write:
    push ebx
    push edi
    
    cmp eax, [device_count]
    jge .error
    
    mov edi, device_registry
    imul eax, device_t.size
    add edi, eax
    
    mov eax, [edi + device_t.write]
    test eax, eax
    jz .error
    
    push ecx                ; size
    push esi                ; buffer
    push edi                ; device pointer
    call eax
    add esp, 12
    
    pop edi
    pop ebx
    ret
    
.error:
    mov eax, -1
    pop edi
    pop ebx
    ret

; -----------------------------------------------------------
; ตัวอย่าง Keyboard device functions
; -----------------------------------------------------------

keyboard_dev_init:
    call keyboard_init
    xor eax, eax
    ret

keyboard_dev_read:
    ; args: [esp+4]=device, [esp+8]=buffer, [esp+12]=size
    push ebp
    mov ebp, esp
    push edi
    push ecx
    
    mov edi, [ebp+12]       ; buffer
    mov ecx, [ebp+16]       ; size
    push ecx
    xor ecx, ecx            ; bytes read counter
    
.read_loop:
    pop eax
    push eax
    cmp ecx, eax
    jge .done
    
    call kb_buffer_get
    jz .done                ; buffer ว่าง
    
    mov [edi + ecx], al
    inc ecx
    jmp .read_loop
    
.done:
    pop eax
    mov eax, ecx            ; return bytes read
    pop ecx
    pop edi
    pop ebp
    ret

keyboard_dev_write:
    xor eax, eax            ; keyboard ไม่รองรับ write
    ret

keyboard_dev_ioctl:
    xor eax, eax
    ret

keyboard_dev_close:
    xor eax, eax
    ret

; -----------------------------------------------------------
; ตัวอย่าง VGA device functions
; -----------------------------------------------------------

vga_dev_init:
    call vga_clear_screen
    xor eax, eax
    ret

vga_dev_write:
    push ebp
    mov ebp, esp
    push esi
    push ecx
    
    mov esi, [ebp+12]       ; buffer
    mov ecx, [ebp+16]       ; size
    
.write_loop:
    test ecx, ecx
    jz .done
    
    mov al, [esi]
    call vga_putchar
    inc esi
    dec ecx
    jmp .write_loop
    
.done:
    mov eax, [ebp+16]       ; return bytes written
    pop ecx
    pop esi
    pop ebp
    ret

vga_dev_null:
    xor eax, eax
    ret

vga_dev_ioctl:
    ; ตัวอย่าง ioctl commands
    ; [esp+8] = cmd, [esp+12] = arg
    push ebp
    mov ebp, esp
    
    mov eax, [ebp+12]       ; cmd
    cmp eax, 0x01           ; IOCTL_SET_COLOR
    jne .unknown
    
    mov eax, [ebp+16]       ; arg = color
    call vga_set_color
    xor eax, eax
    pop ebp
    ret
    
.unknown:
    mov eax, -1
    pop ebp
    ret

; -----------------------------------------------------------
; ตัวอย่าง Serial device functions
; -----------------------------------------------------------

serial_dev_init:
    push ebp
    mov ebp, esp
    
    ; อ่าน base port จาก private_data
    mov eax, [ebp+8]        ; device pointer
    mov edx, [eax + device_t.private_data]
    mov eax, 115200         ; baud rate
    call uart_init
    
    xor eax, eax
    pop ebp
    ret

serial_dev_read:
    push ebp
    mov ebp, esp
    push edi
    push ecx
    
    mov eax, [ebp+8]
    mov edx, [eax + device_t.private_data]
    mov edi, [ebp+12]
    mov ecx, [ebp+16]
    push ecx
    xor ecx, ecx
    
.read_loop:
    pop eax
    push eax
    cmp ecx, eax
    jge .done
    
    call uart_recv_byte
    jz .done
    
    mov [edi + ecx], al
    inc ecx
    jmp .read_loop
    
.done:
    pop eax
    mov eax, ecx
    pop ecx
    pop edi
    pop ebp
    ret

serial_dev_write:
    push ebp
    mov ebp, esp
    push esi
    push ecx
    
    mov eax, [ebp+8]
    mov edx, [eax + device_t.private_data]
    mov esi, [ebp+12]
    mov ecx, [ebp+16]
    
.write_loop:
    test ecx, ecx
    jz .done
    
    mov al, [esi]
    call uart_send_byte
    inc esi
    dec ecx
    jmp .write_loop
    
.done:
    mov eax, [ebp+16]
    pop ecx
    pop esi
    pop ebp
    ret

serial_dev_ioctl:
    xor eax, eax
    ret

serial_dev_close:
    xor eax, eax
    ret
```

---

## 9. Kernel Bootstrap และ Interrupt Setup

### 9.1 IDT Setup สำหรับ Drivers

```nasm
; ============================================================
; kernel_init.asm - Kernel initialization สำหรับ drivers
; ============================================================

section .data

; IDT (Interrupt Descriptor Table) - 256 entries
idt:
    times 256 * 8 db 0      ; 256 entries x 8 bytes each

; IDT Descriptor สำหรับ LIDT instruction
idt_descriptor:
    dw 256 * 8 - 1          ; limit
    dd idt                   ; base address

; GDT (ใช้ flat model 32-bit)
gdt:
    ; Null descriptor
    dd 0, 0
    
    ; Code segment: base=0, limit=4GB, ring 0, 32-bit
    dw 0xFFFF               ; limit low
    dw 0x0000               ; base low
    db 0x00                 ; base mid
    db 0x9A                 ; access: present, ring0, code, execute/read
    db 0xCF                 ; flags: 4KB granularity, 32-bit + limit high
    db 0x00                 ; base high

    ; Data segment: base=0, limit=4GB, ring 0
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0x92                 ; access: present, ring0, data, read/write
    db 0xCF
    db 0x00

gdt_descriptor:
    dw 3 * 8 - 1            ; limit (3 entries)
    dd gdt                   ; base

%define GDT_CODE_SEG    0x08
%define GDT_DATA_SEG    0x10

section .text
global _start

; -----------------------------------------------------------
; _start - Entry point
; -----------------------------------------------------------
_start:
    ; ตั้งค่า stack
    mov esp, 0x90000
    
    ; โหลด GDT
    lgdt [gdt_descriptor]
    
    ; เข้า protected mode
    mov eax, cr0
    or eax, 1
    mov cr0, eax
    
    ; Far jump เพื่อ flush pipeline
    jmp GDT_CODE_SEG:.protected_mode
    
.protected_mode:
    ; ตั้งค่า segments
    mov ax, GDT_DATA_SEG
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; เรียก kernel main
    call kernel_main
    
    ; Halt
    cli
    hlt

; -----------------------------------------------------------
; kernel_main - เริ่มต้น drivers ทั้งหมด
; -----------------------------------------------------------
global kernel_main
kernel_main:
    ; 1. Setup IDT
    call idt_init
    
    ; 2. Initialize PIC (remap IRQs: Master=0x20, Slave=0x28)
    mov al, 0x20
    mov ah, 0x28
    call pic_init
    
    ; 3. Initialize PIT (100 Hz)
    call pit_init
    
    ; 4. Initialize VGA
    call vga_clear_screen
    
    ; 5. Initialize Keyboard
    call keyboard_init
    
    ; 6. เปิด interrupts
    sti
    
    ; 7. ลงทะเบียน devices
    mov esi, keyboard_device
    call device_register
    
    mov esi, vga_device
    call device_register
    
    mov esi, serial_device
    call device_register
    
    ; 8. Test: พิมพ์ข้อความ
    mov esi, msg_welcome
    call vga_puts
    
    ; Main loop
.main_loop:
    hlt                     ; รอ interrupt
    jmp .main_loop

; -----------------------------------------------------------
; idt_init - Initialize IDT
; -----------------------------------------------------------
idt_init:
    push eax
    push ebx
    push ecx
    
    ; ตั้งค่า IRQ handlers ใน IDT
    ; IRQ0 -> INT 0x20: Timer
    mov eax, timer_handler
    mov bx, 0x20
    call idt_set_gate
    
    ; IRQ1 -> INT 0x21: Keyboard
    mov eax, keyboard_handler
    mov bx, 0x21
    call idt_set_gate
    
    ; IRQ4 -> INT 0x24: Serial (COM1)
    mov eax, uart_handler
    mov bx, 0x24
    call idt_set_gate
    
    ; โหลด IDT
    lidt [idt_descriptor]
    
    pop ecx
    pop ebx
    pop eax
    ret

; -----------------------------------------------------------
; idt_set_gate - ตั้งค่า IDT entry
; Input:  EAX = handler address
;         BX  = interrupt number
; -----------------------------------------------------------
idt_set_gate:
    push edx
    push edi
    
    ; คำนวณ address ใน IDT
    movzx edi, bx
    shl edi, 3              ; * 8 (8 bytes per entry)
    add edi, idt
    
    ; IDT Gate Descriptor format:
    ; Bytes 0-1: offset low (bits 0-15)
    ; Bytes 2-3: segment selector
    ; Byte 4:    reserved (0)
    ; Byte 5:    type/attributes (0x8E = interrupt gate, ring 0)
    ; Bytes 6-7: offset high (bits 16-31)
    
    mov [edi], ax           ; offset low
    mov word [edi+2], GDT_CODE_SEG  ; code segment
    mov byte [edi+4], 0     ; reserved
    mov byte [edi+5], 0x8E  ; present, ring0, 32-bit interrupt gate
    shr eax, 16
    mov [edi+6], ax         ; offset high
    
    pop edi
    pop edx
    ret

section .data
msg_welcome:
    db "Assembly Device Drivers Initialized!", 13, 10, 0
```

---

## 10. Makefile และ QEMU Testing

```makefile
# ============================================================
# Makefile สำหรับ Part 089: Device Drivers
# ============================================================

NASM    = nasm
LD      = ld
QEMU    = qemu-system-i386

# NASM flags
NASM_FLAGS = -f elf32 -g -F dwarf

# Linker flags
LD_FLAGS = -m elf_i386 -T linker.ld

# Source files
SOURCES = kernel_init.asm    \
          pic_driver.asm     \
          keyboard_driver.asm \
          pit_driver.asm     \
          vga_driver.asm     \
          uart_driver.asm    \
          ata_driver.asm     \
          device_abstraction.asm

# Object files
OBJECTS = $(SOURCES:.asm=.o)

# Output
KERNEL  = kernel.bin
ISO     = kernel.iso

.PHONY: all clean run debug iso

all: $(KERNEL)

# Compile NASM source files
%.o: %.asm
	$(NASM) $(NASM_FLAGS) $< -o $@

# Link all object files
$(KERNEL): $(OBJECTS)
	$(LD) $(LD_FLAGS) -o $@ $^
	@echo "Built: $@"

# Run in QEMU (bare metal)
run: $(KERNEL)
	$(QEMU) -kernel $(KERNEL) \
	        -serial stdio      \
	        -no-reboot         \
	        -no-shutdown

# Run with GDB debugging
debug: $(KERNEL)
	$(QEMU) -kernel $(KERNEL)  \
	        -serial stdio       \
	        -s -S               \
	        -no-reboot &
	gdb -ex "target remote :1234" \
	    -ex "symbol-file $(KERNEL)"

# สร้าง bootable ISO (ต้องมี GRUB)
iso: $(KERNEL)
	mkdir -p iso/boot/grub
	cp $(KERNEL) iso/boot/
	echo 'set timeout=0'                    > iso/boot/grub/grub.cfg
	echo 'set default=0'                   >> iso/boot/grub/grub.cfg
	echo 'menuentry "Assembly Drivers" {'  >> iso/boot/grub/grub.cfg
	echo '  multiboot /boot/$(KERNEL)'     >> iso/boot/grub/grub.cfg
	echo '}'                               >> iso/boot/grub/grub.cfg
	grub-mkrescue -o $(ISO) iso/
	@echo "ISO created: $(ISO)"

# รัน ISO ใน QEMU
run-iso: $(ISO)
	$(QEMU) -cdrom $(ISO)          \
	        -serial stdio           \
	        -m 32M                  \
	        -no-reboot

# Test serial port output
test-serial: $(KERNEL)
	$(QEMU) -kernel $(KERNEL)      \
	        -serial file:serial.log \
	        -display none           \
	        -no-reboot
	@cat serial.log

# Test keyboard (interactive)
test-keyboard: $(KERNEL)
	$(QEMU) -kernel $(KERNEL) \
	        -serial stdio

# Test ATA (สร้าง disk image ก่อน)
test-ata: $(KERNEL)
	dd if=/dev/zero of=test_disk.img bs=512 count=2048
	$(QEMU) -kernel $(KERNEL)          \
	        -drive file=test_disk.img,  \
	               format=raw,          \
	               if=ide              \
	        -serial stdio

clean:
	rm -f $(OBJECTS) $(KERNEL) $(ISO)
	rm -rf iso/
	rm -f serial.log test_disk.img
```

### Linker Script

```ld
/* linker.ld - Linker script สำหรับ kernel */
OUTPUT_FORMAT("elf32-i386")
ENTRY(_start)

SECTIONS
{
    . = 0x100000;   /* โหลด kernel ที่ 1MB */

    .text :
    {
        *(.text)
    }

    .data :
    {
        *(.data)
        *(.rodata)
    }

    .bss :
    {
        *(.bss)
        *(COMMON)
    }
}
```

---

## 11. สรุปและ Quick Reference

### 11.1 I/O Port Map สำหรับ Common Devices

| Device | Port(s) | Description |
|--------|---------|-------------|
| PIC Master | 0x20, 0x21 | Interrupt Controller |
| PIC Slave | 0xA0, 0xA1 | Interrupt Controller |
| PIT Channel 0 | 0x40 | System Timer |
| PIT Command | 0x43 | Timer Mode/Command |
| PS/2 Keyboard | 0x60, 0x64 | Data / Status/Command |
| VGA CRTC | 0x3D4, 0x3D5 | CRT Controller |
| Serial COM1 | 0x3F8-0x3FF | UART Registers |
| ATA Primary | 0x1F0-0x1F7 | Hard Drive |
| Speaker | 0x61 | PC Speaker |

### 11.2 IRQ Assignments

| IRQ | Interrupt | Device |
|-----|-----------|--------|
| IRQ0 | INT 0x20 | PIT Timer |
| IRQ1 | INT 0x21 | PS/2 Keyboard |
| IRQ2 | INT 0x22 | PIC Cascade |
| IRQ3 | INT 0x23 | COM2 Serial |
| IRQ4 | INT 0x24 | COM1 Serial |
| IRQ6 | INT 0x26 | Floppy Disk |
| IRQ8 | INT 0x28 | Real-Time Clock |
| IRQ14 | INT 0x2E | ATA Primary |
| IRQ15 | INT 0x2F | ATA Secondary |

### 11.3 VGA Color Codes

| Code | Color | Code | Color |
|------|-------|------|-------|
| 0x0 | Black | 0x8 | Dark Grey |
| 0x1 | Blue | 0x9 | Light Blue |
| 0x2 | Green | 0xA | Light Green |
| 0x3 | Cyan | 0xB | Light Cyan |
| 0x4 | Red | 0xC | Light Red |
| 0x5 | Magenta | 0xD | Light Magenta |
| 0x6 | Brown | 0xE | Yellow |
| 0x7 | Light Grey | 0xF | White |

### 11.4 QEMU Test Commands

```bash
# รัน kernel แบบ debug
qemu-system-i386 -kernel kernel.bin -s -S &
gdb kernel.bin -ex "target remote :1234"

# Monitor serial output
qemu-system-i386 -kernel kernel.bin -serial stdio

# Test กับ virtual disk
qemu-system-i386 -kernel kernel.bin \
  -drive file=disk.img,format=raw,if=ide \
  -serial stdio -m 32M

# ใช้ monitor console
qemu-system-i386 -kernel kernel.bin \
  -monitor stdio \
  -serial file:serial.log

# เช็ค I/O port activity (ต้องใช้ QEMU monitor)
# (qemu) info pic        - ดูสถานะ PIC
# (qemu) info irq        - ดู IRQ statistics
# (qemu) outb 0x60 0x1E  - ส่ง scan code จำลอง
```

---

## 12. แบบฝึกหัด

1. **PIC Exercise**: เพิ่ม IRQ handler สำหรับ Real-Time Clock (IRQ8) และแสดงเวลาใน VGA
2. **Keyboard Exercise**: เพิ่มการรองรับ Extended Scan Codes (Escape prefix 0xE0) สำหรับ arrow keys
3. **PIT Exercise**: ใช้ PIT Channel 2 ร่วมกับ Port 0x61 เพื่อสร้างเสียง beep ที่ความถี่ต่างๆ
4. **VGA Exercise**: Implement double buffering ใน VGA text mode
5. **UART Exercise**: Implement circular buffer สำหรับ TX ด้วย interrupt-driven
6. **ATA Exercise**: อ่าน MBR (sector 0) และแสดง partition table
7. **Device Abstraction**: เพิ่ม device_ioctl() ที่รองรับคำสั่ง IOCTL_GET_INFO

---

## 13. References

- Intel 64 and IA-32 Architectures Software Developer Manual
- OSDev Wiki: https://wiki.osdev.org
- 8259A Programmable Interrupt Controller datasheet
- 8253/8254 Programmable Interval Timer datasheet
- NS16550A UART datasheet
- ATA/ATAPI-6 specification
- VGA Hardware Reference (FreeVGA project)
- NASM Manual: https://nasm.us/doc/
