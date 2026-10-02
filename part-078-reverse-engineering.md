# Part 078: Reverse Engineering Fundamentals

## บทนำ (Introduction)

Reverse Engineering ในบริบทของ Binary Analysis คือการศึกษาและทำความเข้าใจโปรแกรมที่ compile แล้ว โดยไม่มี source code ต้นฉบับ ทักษะนี้สำคัญมากสำหรับ:

- **CTF (Capture The Flag)** competitions
- **Malware Analysis** - วิเคราะห์ malware
- **Vulnerability Research** - ค้นหาช่องโหว่
- **Interoperability** - ทำให้ระบบต่างๆ ทำงานร่วมกันได้
- **Software Verification** - ตรวจสอบว่า binary ทำงานตามที่ claim ไว้

---

## 1. Static Analysis Workflow

### 1.1 ขั้นตอนการวิเคราะห์ Static Analysis

```
Binary File
    │
    ▼
[file command] ──── identify binary type, architecture, OS
    │
    ▼
[strings command] ── extract printable strings, hints
    │
    ▼
[nm / readelf] ───── list symbols, sections, headers
    │
    ▼
[objdump / ndisasm] ─ disassemble to assembly
    │
    ▼
[IDA Pro / Ghidra] ── advanced analysis, decompile to C
    │
    ▼
[Understanding Logic] ─ understand algorithm, find vulnerability
```

### 1.2 ทำไมต้องทำ Static Analysis ก่อน?

Static Analysis ไม่ต้อง execute binary จึง:
- **ปลอดภัย** สำหรับ malware analysis
- **ไม่ถูก detect** โดย anti-debugging tricks
- **ได้ภาพรวม** ของโปรแกรมทั้งหมดก่อน

---

## 2. `file` Command: Identify Binary Type

### 2.1 การใช้งานพื้นฐาน

```bash
$ file unknown_binary
unknown_binary: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), 
                dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, 
                BuildID[sha1]=abc123..., for GNU/Linux 3.2.0, not stripped

$ file crackme.exe
crackme.exe: PE32+ executable (console) x86-64, for MS Windows

$ file libtest.so
libtest.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), 
            dynamically linked, BuildID[sha1]=..., not stripped

$ file program.o
program.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

### 2.2 สิ่งที่ต้องสังเกตจาก `file` output

| Field | ความหมาย | ตัวอย่าง |
|-------|-----------|----------|
| ELF/PE/Mach-O | Format ของ binary | ELF = Linux, PE = Windows |
| 32-bit / 64-bit | Architecture width | 32-bit = x86, 64-bit = x86-64 |
| LSB / MSB | Endianness | LSB = little-endian, MSB = big-endian |
| dynamically linked | ใช้ shared libraries | มี .so/.dll dependencies |
| statically linked | รวม library ไว้ใน binary | ไฟล์ใหญ่กว่า แต่ portable |
| stripped / not stripped | มี/ไม่มี symbol table | stripped ยากกว่า reverse |

### 2.3 Binary Format Overview

#### ELF (Executable and Linkable Format) - Linux/Unix
```
ELF Header
├── Magic: 7f 45 4c 46 (0x7f ELF)
├── Class: 32/64-bit
├── Data: LSB/MSB
├── Type: ET_EXEC, ET_DYN, ET_REL, ET_CORE
└── Machine: x86, x86-64, ARM, MIPS, etc.

Section Headers (ถ้า not stripped)
├── .text    - code
├── .data    - initialized data
├── .bss     - uninitialized data
├── .rodata  - read-only data (strings)
├── .plt     - Procedure Linkage Table
├── .got     - Global Offset Table
└── .symtab  - symbol table

Program Headers (Segments)
├── LOAD    - loadable segments
├── DYNAMIC - dynamic linking info
├── INTERP  - path to dynamic linker
└── GNU_STACK - stack permissions
```

#### PE (Portable Executable) - Windows
```
DOS Header (MZ header)
├── Magic: 4d 5a (MZ)
└── e_lfanew: offset to PE header

PE Header
├── Signature: 50 45 00 00 (PE\0\0)
├── Machine: x86, x86-64, ARM
├── NumberOfSections
└── Characteristics

Optional Header
├── AddressOfEntryPoint
├── ImageBase (0x400000 typical)
└── DataDirectory

Section Table
├── .text   - code
├── .data   - data
├── .rdata  - read-only data
├── .idata  - import table
└── .edata  - export table
```

#### Mach-O - macOS/iOS
```
Magic: CE FA ED FE (32-bit LE) หรือ CF FA ED FE (64-bit LE)
Load Commands
├── LC_SEGMENT_64 - segments
├── LC_DYLD_INFO   - dynamic linking
├── LC_SYMTAB      - symbol table
└── LC_MAIN        - entry point

Sections
├── __TEXT.__text  - code
├── __DATA.__data  - data
└── __DATA.__bss   - BSS
```

---

## 3. `strings` Command: Extract Printable Strings

### 3.1 การใช้งาน strings

```bash
# พื้นฐาน - ดึง strings ที่ยาว >= 4 ตัวอักษร
$ strings crackme

# กำหนด minimum length
$ strings -n 8 crackme

# แสดง offset ของแต่ละ string
$ strings -t x crackme    # hex offset
$ strings -t d crackme    # decimal offset
$ strings -t o crackme    # octal offset

# ดูใน specific section
$ strings -a crackme      # ดูทั้ง binary (ไม่แค่ data sections)

# Unicode strings (Windows programs)
$ strings -e l crackme    # little-endian 16-bit
$ strings -e b crackme    # big-endian 16-bit
```

### 3.2 สิ่งที่น่าสนใจใน strings output

```bash
$ strings ./crackme | grep -iE "password|pass|key|flag|secret|correct|wrong|congratulation"

# ตัวอย่าง output ที่น่าสนใจ:
Enter password: 
Correct! Here is your flag.
Wrong password!
CTF{th1s_1s_4_fl4g}
/lib64/ld-linux-x86-64.so.2
printf
scanf
strcmp
exit
__libc_start_main
```

### 3.3 วิเคราะห์ strings output

**Interesting strings to look for:**

1. **Flag format patterns**: `CTF{...}`, `flag{...}`, `DUCTF{...}`
2. **Error/success messages**: "Correct", "Wrong", "Congratulations"
3. **Hardcoded credentials**: passwords, keys
4. **File paths**: `/tmp/`, `/etc/passwd`, `C:\Windows\`
5. **URLs**: `http://`, `https://`
6. **Library functions**: `printf`, `strcmp`, `system`
7. **Build info**: compiler version, debug paths
8. **Obfuscated hints**: encoded strings ที่ยังพอ recognize ได้

```bash
# ตัวอย่างการ filter strings อย่างมีประสิทธิภาพ
$ strings crackme | sort | uniq    # remove duplicates
$ strings crackme | grep -v "^.\{1,3\}$"  # >=4 chars
$ strings crackme | grep -E "^[A-Za-z0-9+/]{20,}={0,2}$"  # potential base64
```

---

## 4. `nm`: List Symbols

### 4.1 การใช้งาน nm

```bash
# แสดง symbol ทั้งหมด
$ nm binary

# ใช้กับ stripped binary (บางครั้งยังมี dynamic symbols)
$ nm -D binary        # dynamic symbols เท่านั้น

# แสดงขนาดของ symbols
$ nm -S binary

# sort by address
$ nm -n binary

# demangle C++ names
$ nm -C binary

# แสดงทุก symbols รวม debugging symbols
$ nm -a binary
```

### 4.2 Symbol Types

```
Symbol Table Output Format:
[address] [type] [name]

Types:
T/t - .text section (code)    T = global, t = local
D/d - .data section (initialized data)
B/b - .bss section (uninitialized)
R/r - .rodata (read-only)
U   - undefined (จาก external library)
W/w - weak symbol
I   - indirect reference
```

```bash
$ nm crackme
                 U exit@GLIBC_2.2.5
                 U printf@GLIBC_2.2.5
                 U scanf@GLIBC_2.2.5
                 U strcmp@GLIBC_2.2.5
0000000000401000 T _start
0000000000401050 T main
0000000000401200 T check_password
0000000000402000 D secret_key
0000000000402020 B user_buffer
```

### 4.3 Stripped vs Unstripped Binary

**Unstripped binary** (ง่ายต่อการ reverse):
```bash
$ nm unstripped_binary
0000000000401050 T main
0000000000401100 T calculate_checksum
0000000000401200 T verify_license
0000000000401300 T decrypt_string
# เห็น function names ทั้งหมด!
```

**Stripped binary** (ยากกว่า):
```bash
$ nm stripped_binary
nm: stripped_binary: no symbols

# แต่ยังใช้ dynamic symbols ได้
$ nm -D stripped_binary
                 U printf@GLIBC_2.2.5
                 U strcmp@GLIBC_2.2.5
# เห็นแค่ library calls
```

**วิธีตรวจสอบว่า stripped หรือไม่:**
```bash
$ file binary
binary: ELF 64-bit LSB executable, ..., stripped
# หรือ
binary: ELF 64-bit LSB executable, ..., not stripped
```

---

## 5. `objdump`: Disassemble และวิเคราะห์

### 5.1 การใช้งาน objdump

```bash
# Disassemble code sections
$ objdump -d binary

# Disassemble ทุก section ที่มี code
$ objdump -D binary

# แสดง symbol table
$ objdump -t binary

# แสดง headers
$ objdump -x binary

# แสดง section headers
$ objdump -h binary

# แสดง dynamic symbols
$ objdump -T binary

# Intel syntax (อ่านง่ายกว่า AT&T)
$ objdump -d -M intel binary

# Disassemble + show source (ถ้ามี debug info)
$ objdump -d -S binary

# แสดง relocation entries
$ objdump -r binary

# disassemble เฉพาะ function
$ objdump -d binary | grep -A 50 "<main>:"
```

### 5.2 ตัวอย่าง objdump Output

```bash
$ objdump -d -M intel crackme

crackme:     file format elf64-little

Disassembly of section .text:

0000000000401050 <main>:
  401050:	55                   	push   rbp
  401051:	48 89 e5             	mov    rbp,rsp
  401054:	48 83 ec 30          	sub    rsp,0x30
  401058:	bf 00 20 40 00       	mov    edi,0x402000
  40105d:	e8 ce ff ff ff       	call   401030 <printf@plt>
  401062:	48 8d 45 d0          	lea    rax,[rbp-0x30]
  401066:	48 89 c6             	mov    rsi,rax
  401069:	bf 1a 20 40 00       	mov    edi,0x40201a
  40106e:	b8 00 00 00 00       	mov    eax,0x0
  401073:	e8 c8 ff ff ff       	call   401040 <scanf@plt>
  401078:	48 8d 45 d0          	lea    rax,[rbp-0x30]
  40107c:	be 20 20 40 00       	mov    esi,0x402020
  401081:	48 89 c7             	mov    rdi,rax
  401084:	e8 b7 ff ff ff       	call   401040 <strcmp@plt>
  401089:	85 c0                	test   eax,eax
  40108b:	75 0e                	jne    40109b <main+0x4b>
  40108d:	bf 30 20 40 00       	mov    edi,0x402030
  401092:	e8 99 ff ff ff       	call   401030 <printf@plt>
  401097:	eb 0c                	jmp    4010a5 <main+0x55>
  40109b:	bf 48 20 40 00       	mov    edi,0x402048
  4010a0:	e8 8b ff ff ff       	call   401030 <printf@plt>
  4010a5:	b8 00 00 00 00       	mov    eax,0x0
  4010aa:	c9                   	leave  
  4010ab:	c3                   	ret    
```

### 5.3 วิเคราะห์ objdump Output

จาก output ข้างบน เราสามารถ reverse engineer logic ได้:
1. `printf` พิมพ์ prompt (ที่ address 0x402000)
2. `scanf` อ่าน input จาก user เก็บใน `rbp-0x30`
3. `strcmp` เปรียบเทียบ input กับ hardcoded string (ที่ 0x402020)
4. ถ้า `strcmp` return 0 (equal) → print success (0x402030)
5. ถ้าไม่เท่ากัน → jump ไป print failure (0x402048)

```bash
# ดู strings ที่ address เหล่านั้น
$ objdump -s -j .rodata crackme

Contents of section .rodata:
 402000 456e7465 72207061 7373776f 72643a20  Enter password: 
 402010 00000000 00000000 25730a00 00000000  ........%s......
 402020 73656372 65743132 33000000 00000000  secret123.......
 402030 436f7272 65637421 0a000000 00000000  Correct!........
 402048 57726f6e 6721 0a00                   Wrong!..
# Password คือ "secret123"!
```

---

## 6. `readelf`: All ELF Information

### 6.1 การใช้งาน readelf

```bash
# แสดงทุกอย่าง
$ readelf -a binary

# แสดงเฉพาะ ELF header
$ readelf -h binary

# แสดง program headers (segments)
$ readelf -l binary

# แสดง section headers
$ readelf -S binary

# แสดง symbol table
$ readelf -s binary

# แสดง dynamic section
$ readelf -d binary

# แสดง relocation tables
$ readelf -r binary

# แสดง notes
$ readelf -n binary

# แสดง string table
$ readelf -p .rodata binary

# แสดง hex dump
$ readelf -x .rodata binary
```

### 6.2 ตัวอย่าง readelf Output

```bash
$ readelf -h crackme
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x401000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          8192 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         6
  Size of section headers:           64 (bytes)
  Number of section headers:         18
  Section header string table index: 17

$ readelf -S crackme
There are 18 section headers, starting at offset 0x2000:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000401000  00001000
       00000000000000ac  0000000000000000  AX       0     0     16
  [ 2] .rodata           PROGBITS         0000000000402000  00002000
       0000000000000050  0000000000000000   A       0     0     8
  ...
```

---

## 7. Compiler Patterns Recognition

### 7.1 Function Prologue และ Epilogue

**x86-64 Function Prologue (standard):**
```asm
push rbp           ; บันทึก caller's base pointer
mov  rbp, rsp      ; ตั้ง base pointer ของ function นี้
sub  rsp, 0x??     ; จอง stack space สำหรับ local variables
```

**Function Epilogue:**
```asm
; วิธีที่ 1 (ชัดเจน)
mov  rsp, rbp      ; restore stack pointer
pop  rbp           ; restore base pointer
ret                ; return

; วิธีที่ 2 (ย่อ)
leave              ; = mov rsp, rbp + pop rbp
ret
```

**x86 (32-bit) Function Prologue:**
```asm
push ebp
mov  ebp, esp
sub  esp, 0x??
```

**Function with no local variables:**
```asm
; Leaf function (ไม่เรียก function อื่น)
; บางครั้งไม่มี prologue/epilogue เลย
; หรือ compiler optimize ออก
```

### 7.2 Local Variable Access

**x86-64 Local Variables (relative to RBP):**
```asm
; Local variables อยู่ที่ [rbp - offset]
mov  QWORD PTR [rbp-0x8], rax    ; int64 variable ที่ rbp-8
mov  DWORD PTR [rbp-0xc], 0x0    ; int32 ที่ rbp-12
mov  BYTE PTR  [rbp-0xd], 0x41   ; char ที่ rbp-13

; Function arguments (first 6 in registers: rdi, rsi, rdx, rcx, r8, r9)
; ถ้า function มี many arguments:
mov  QWORD PTR [rbp-0x8], rdi    ; save arg1
mov  QWORD PTR [rbp-0x10], rsi   ; save arg2
```

**Stack Layout ตัวอย่าง:**
```
Higher addresses
┌─────────────┐
│  ...        │
│  saved ret  │  [rbp+8]  = return address
│  saved rbp  │  [rbp+0]  = caller's rbp  ← RBP points here
│  local1     │  [rbp-8]
│  local2     │  [rbp-16]
│  local3     │  [rbp-24]
│  ...        │
└─────────────┘
Lower addresses (RSP points here)
```

### 7.3 Array Indexing Patterns

**1D Array Access:**
```c
// C code
int arr[10];
arr[i] = 5;
```

```asm
; assembly equivalent (arr ที่ rbp-0x30, i ที่ rbp-0x4)
mov  eax, DWORD PTR [rbp-0x4]    ; eax = i
cdqe                              ; sign-extend eax to rax
mov  DWORD PTR [rbp+rax*4-0x30], 5  ; arr[i] = 5
; rax*4 เพราะ int = 4 bytes
```

**2D Array Access:**
```c
int matrix[3][4];
matrix[row][col] = val;
```

```asm
; address = base + (row * 4 + col) * sizeof(int)
; = base + (row * 4 + col) * 4
mov  eax, DWORD PTR [rbp-0x4]    ; row
imul eax, eax, 4                  ; row * 4 (number of cols)
add  eax, DWORD PTR [rbp-0x8]    ; + col
cdqe
lea  rdx, [rax*4]                 ; * sizeof(int)
lea  rax, [rbp-0x50]              ; base of matrix
mov  DWORD PTR [rdx+rax], 0xa    ; matrix[row][col] = val
```

### 7.4 Switch-Case Patterns (Jump Table)

**Switch with dense cases → Jump Table:**
```c
switch (x) {
    case 0: do_a(); break;
    case 1: do_b(); break;
    case 2: do_c(); break;
    case 3: do_d(); break;
    default: do_default(); break;
}
```

```asm
mov  eax, DWORD PTR [rbp-0x4]   ; eax = x
cmp  eax, 3                      ; compare with max case
ja   .default                    ; if x > 3, goto default
; Jump table lookup:
mov  eax, eax
lea  rdx, [rip+0x2000]           ; address of jump table
movsxd rax, DWORD PTR [rdx+rax*4]  ; load table entry
add  rax, rdx                    ; compute target address
jmp  rax                         ; jump to case

.jump_table:
    .long case_0 - .jump_table
    .long case_1 - .jump_table
    .long case_2 - .jump_table
    .long case_3 - .jump_table
```

**Switch with sparse cases → if/else chain:**
```c
switch (x) {
    case 1:   ...
    case 100: ...
    case 200: ...
}
```

```asm
; Compiled as if-else เพราะ sparse
cmp eax, 1
je  .case_1
cmp eax, 100
je  .case_100
cmp eax, 200
je  .case_200
jmp .default
```

### 7.5 If/Else Chains

**Simple if-else:**
```c
if (a > b) {
    x = 1;
} else {
    x = 2;
}
```

```asm
mov  eax, DWORD PTR [rbp-0x4]   ; a
cmp  eax, DWORD PTR [rbp-0x8]   ; compare with b
jle  .else                        ; if a <= b, goto else
mov  DWORD PTR [rbp-0xc], 1     ; x = 1
jmp  .end
.else:
mov  DWORD PTR [rbp-0xc], 2     ; x = 2
.end:
```

**Compound conditions:**
```c
if (a > 0 && b < 10) { ... }
```

```asm
; Short-circuit evaluation
cmp  eax, 0
jle  .skip          ; if a <= 0, skip (short-circuit)
cmp  ecx, 10
jge  .skip          ; if b >= 10, skip
; ... body ...
.skip:
```

### 7.6 Loop Patterns

**do-while loop:**
```c
do {
    body;
} while (condition);
```

```asm
.loop_start:
    ; body
    ; ...
    cmp eax, 0       ; test condition
    jne .loop_start  ; loop if condition true
    ; หลัง loop
```
ข้อสังเกต: body อยู่ก่อน condition check

**for/while loop (condition ตรวจก่อน):**
```c
while (condition) {
    body;
}
```

```asm
    jmp .check        ; jump to condition first
.loop_start:
    ; body
    ; ...
.check:
    cmp eax, 0        ; test condition
    jne .loop_start   ; loop if condition true
    ; หลัง loop
```

หรือ (อีกแบบ):
```asm
.check:
    cmp eax, 0
    je  .end_loop    ; exit if condition false
    ; body
    jmp .check
.end_loop:
```

**for loop:**
```c
for (i = 0; i < n; i++) {
    arr[i] = i * 2;
}
```

```asm
    mov DWORD PTR [rbp-0x4], 0    ; i = 0
    jmp .for_cond
.for_body:
    mov eax, [rbp-0x4]            ; eax = i
    ; ... body ...
    add DWORD PTR [rbp-0x4], 1    ; i++
.for_cond:
    mov eax, [rbp-0x4]            ; eax = i
    cmp eax, [rbp-0x8]            ; compare with n
    jl  .for_body                  ; if i < n, loop
```

---

## 8. Virtual Functions (vtable pointer)

### 8.1 C++ Virtual Function Mechanism

```cpp
class Animal {
public:
    virtual void speak() { printf("..."); }
    virtual void move()  { printf("moves"); }
    int age;
};

class Dog : public Animal {
public:
    virtual void speak() override { printf("Woof!"); }
    // move() inherited
    char name[16];
};
```

**Memory Layout of Dog object:**
```
Dog object:
┌──────────────┐ ← object pointer
│ vtable ptr   │  8 bytes (pointer to vtable)
├──────────────┤
│ age (int)    │  4 bytes
├──────────────┤
│ padding      │  4 bytes
├──────────────┤
│ name[16]     │ 16 bytes
└──────────────┘

Dog vtable:
┌──────────────┐ ← vtable pointer value
│ Dog::speak   │  pointer to Dog::speak
├──────────────┤
│ Animal::move │  pointer to Animal::move (inherited)
└──────────────┘
```

**Assembly สำหรับ virtual call:**
```asm
; dog->speak()
mov  rax, QWORD PTR [rbp-0x8]   ; rax = dog pointer
mov  rax, QWORD PTR [rax]        ; rax = vtable pointer
mov  rdx, QWORD PTR [rax]        ; rdx = vtable[0] = speak()
mov  rax, QWORD PTR [rbp-0x8]   ; rax = dog (this pointer)
mov  rdi, rax                    ; rdi = this
call rdx                          ; call Dog::speak

; dog->move() (second virtual function)
mov  rax, QWORD PTR [rbp-0x8]   ; rax = dog pointer
mov  rax, QWORD PTR [rax]        ; rax = vtable pointer
mov  rdx, QWORD PTR [rax+8]     ; rdx = vtable[1] = move()
mov  rax, QWORD PTR [rbp-0x8]
mov  rdi, rax
call rdx
```

### 8.2 Multiple Inheritance และ vtable

```cpp
class A { virtual void foo(); };
class B { virtual void bar(); };
class C : public A, public B { 
    virtual void foo() override;
    virtual void bar() override;
};
```

```
C object:
┌──────────────┐
│ vtable_A ptr │  → C's vtable for A interface
├──────────────┤
│ A data       │
├──────────────┤
│ vtable_B ptr │  → C's vtable for B interface
├──────────────┤
│ B data       │
└──────────────┘
```

---

## 9. C++ Object Layout

### 9.1 Simple Class Layout

```cpp
class Point {
    double x;   // 8 bytes
    double y;   // 8 bytes
    int    id;  // 4 bytes
    // 4 bytes padding
};
// Total: 24 bytes
```

**Accessing members in assembly:**
```asm
; Point* p in rbp-0x8
mov  rax, [rbp-0x8]          ; rax = p
movsd xmm0, [rax]            ; xmm0 = p->x  (offset 0)
movsd xmm1, [rax+8]          ; xmm1 = p->y  (offset 8)
mov  ecx, [rax+16]           ; ecx = p->id  (offset 16)
```

### 9.2 Inheritance Layout

```cpp
class Base {
    int base_data;   // offset 0
};

class Derived : public Base {
    int derived_data; // offset 4 (after Base)
};
```

### 9.3 การ Recognize Object Access Pattern

```asm
; Pattern ที่เห็นบ่อยสำหรับ object access
; obj->field1 = rax + 0
; obj->field2 = rax + 8
; obj->field3 = rax + 16

; ถ้าเห็น pattern [rax], [rax+8], [rax+16]
; คาดว่ากำลัง access fields ของ struct/class
```

---

## 10. String Obfuscation Patterns

### 10.1 XOR Obfuscation

วิธีที่ใช้บ่อยที่สุดใน malware:

```c
// Original string: "Hello"
// Key: 0x42
char obfuscated[] = {0x0A, 0x27, 0x2E, 0x2E, 0x2D};  // H^0x42, e^0x42, ...

void decode() {
    for (int i = 0; i < 5; i++) {
        obfuscated[i] ^= 0x42;
    }
    printf("%s\n", obfuscated);  // "Hello"
}
```

**Assembly pattern:**
```asm
.decode_loop:
    movzx eax, BYTE PTR [rbp+rcx-0x20]   ; load byte
    xor   eax, 0x42                        ; XOR with key
    mov   BYTE PTR [rbp+rcx-0x20], al     ; store back
    add   rcx, 1
    cmp   rcx, 5
    jl    .decode_loop
```

**ค้นหา XOR obfuscation ใน disassembly:**
```bash
# ค้นหา xor instruction ที่ไม่ใช่ xor reg,reg (zero idiom)
$ objdump -d binary | grep "xor" | grep -v "eax,eax\|rax,rax\|ecx,ecx"
```

### 10.2 ROT13/ROT Variants

```c
// ROT13: เลื่อน character ไป 13 ตำแหน่ง
char rot13(char c) {
    if (c >= 'A' && c <= 'Z')
        return ((c - 'A' + 13) % 26) + 'A';
    if (c >= 'a' && c <= 'z')
        return ((c - 'a' + 13) % 26) + 'a';
    return c;
}
```

**Pattern ใน assembly:**
```asm
; Pattern ที่บ่งชี้ ROT:
; มี comparison กับ 'A' (0x41) และ 'Z' (0x5A)
; มี add/sub ด้วยค่า constant (13 สำหรับ ROT13)
; มี modulo operation (div 26)
cmp  al, 0x41    ; 'A'
jl   .not_upper
cmp  al, 0x5a    ; 'Z'
jg   .not_upper
sub  al, 0x41    ; -= 'A'
add  al, 13      ; + rotation amount
mov  ah, 0
mov  cl, 26
div  cl          ; % 26
mov  al, ah      ; remainder
add  al, 0x41    ; += 'A'
```

### 10.3 Base64 Obfuscation

```c
// Base64 encoded string
const char encoded[] = "SGVsbG8gV29ybGQ=";

void decode_and_use() {
    char decoded[20];
    base64_decode(encoded, decoded);
    // ใช้ decoded string
}
```

**ค้นหา base64:**
```bash
# Pattern: ยาว, มีแค่ A-Z a-z 0-9 + / =
$ strings binary | grep -E "^[A-Za-z0-9+/]{20,}={0,2}$"

# หรือดู table lookup
$ objdump -d binary | grep "lea.*0x" | head -20
# ถ้าเห็น access ไปยัง table ที่ใหญ่ (64 bytes) อาจเป็น base64 table
```

### 10.4 Stack String Construction

```c
// เพื่อหลีกเลี่ยง strings ที่ตรวจจับได้ง่าย
void stack_string() {
    char s[8];
    s[0] = 'p';
    s[1] = 'a';
    s[2] = 's';
    s[3] = 's';
    s[4] = '\0';
    // ใช้ s
}
```

```asm
; Pattern: push individual bytes onto stack
; หรือ mov BYTE PTR ทีละตัว
mov  BYTE PTR [rbp-0x8], 0x70    ; 'p'
mov  BYTE PTR [rbp-0x7], 0x61    ; 'a'
mov  BYTE PTR [rbp-0x6], 0x73    ; 's'
mov  BYTE PTR [rbp-0x5], 0x73    ; 's'
mov  BYTE PTR [rbp-0x4], 0x00    ; '\0'
```

---

## 11. Anti-Disassembly Tricks

### 11.1 Jump to Middle of Instruction

**Concept:** สร้าง jump ที่ jump ไปยัง byte กลางของ instruction อื่น ทำให้ linear disassembler ถอดรหัสผิด

```
Memory layout:
addr:    EB 01      ; JMP $+3 (jump to addr+3)
addr+2:  [trash byte 0xE9]  ; ← disassembler คิดว่าเริ่ม instruction ที่นี่
addr+3:  [real instruction]  ; ← แต่จริงๆ execution มาที่นี่
```

**ตัวอย่าง:**
```
Bytes: EB 01 E8 90 90 90 90
                
Linear disassembly (ผิด):
  addr+0: EB 01       JMP addr+3
  addr+2: E8 90 90 90 CALL addr+0x909095  (ผิด! E8 เป็น byte ก่อน real instruction)
  
True execution:
  addr+0: EB 01       JMP addr+3
  addr+3: 90          NOP   (จริงๆ เริ่มที่นี่)
  addr+4: 90          NOP
  addr+5: 90          NOP
  addr+6: 90          NOP
```

### 11.2 Overlapping Instructions

```
Bytes: FF E0 [หลัง]

Disassembly from start:
  FF E0    JMP RAX

ถ้า disassemble จาก offset 1:
  E0 [xx]   LOOPNE [xx]  (different instruction!)
```

**ตัวอย่างที่ซับซ้อน:**
```
; Attacker เขียน code ที่อ่านได้ 2 วิธี
; วิธีที่ 1 (linear disassembler เห็น)
74 06        JZ +6    (taken)
. . .
; หลัง JZ
EB 05        JMP +5   (real code path)

; วิธีที่ 2 (actual execution - JZ taken)
; jump ไปที่ byte กลาง EB instruction
```

### 11.3 Bogus Conditional Jumps

```asm
; Always-true condition ที่ดูเหมือน conditional
xor  eax, eax    ; eax = 0
jz   .real_code  ; always jumps (เพราะ eax = 0 เสมอ)
; dead code ที่ confuse disassembler
db  0xE8         ; garbage byte

.real_code:
; actual code
```

### 11.4 Self-Modifying Code (SMC)

```c
void modify_and_run() {
    // decrypt code at runtime
    unsigned char code[] = {0x48, 0x31, 0xC0, 0xC3};  // decrypted XOR opcodes
    for (int i = 0; i < 4; i++) {
        code[i] ^= 0x55;  // encrypt at compile time, decrypt at runtime
    }
    // execute code
    ((void(*)())code)();
}
```

### 11.5 วิธีรับมือกับ Anti-Disassembly

1. **ใช้ IDA Pro/Ghidra** ที่มี recursive descent disassembly
2. **Manual patching**: NOP out bogus bytes
3. **Dynamic analysis**: run กับ debugger แล้วดู actual execution
4. **Binary patching**: แก้ bytes ที่ confuse disassembler

```bash
# ใน IDA: กด 'D' เพื่อ undefine, แล้ว 'C' เพื่อ redefine as code
# ใน Ghidra: right-click → Clear Code Bytes, แล้ว Disassemble
```

---

## 12. IDA Pro Basics

### 12.1 Navigation

**Keyboard Shortcuts ที่สำคัญ:**

| Shortcut | Action |
|----------|--------|
| `G` | Go to address |
| `Ctrl+F` | Find text |
| `Alt+T` | Find text in code |
| `Space` | Toggle graph/text view |
| `Esc` | Go back |
| `Enter` | Follow reference |
| `X` | Cross-references |
| `N` | Rename |
| `Y` | Change type |
| `D` | Undefine |
| `C` | Define as code |
| `A` | Define as string |
| `;` | Add comment |
| `/` | Add function comment |
| `Ctrl+P` | Jump to function |
| `F5` | Decompile (Hex-Rays) |

### 12.2 Functions View

```
Functions Window (Alt+F):
- แสดงทุก function ที่ IDA identify ได้
- สำหรับ stripped binary: sub_401000, sub_401050, ...
- ตั้งชื่อ function ด้วย N key
- ดู Function size (บอกถึงความซับซ้อน)
```

### 12.3 Graph View

```
Graph View (Space):
- แสดง control flow ของ function
- สี:
  - สีฟ้า: basic block ปกติ
  - สีแดง: conditional branch (false/fall-through)
  - สีเขียว: conditional branch (true/taken)
  
- ใช้ zoom in/out ด้วย scroll
- ใช้ Ctrl+scroll สำหรับ navigate
```

### 12.4 Strings View

```
View → Open Subviews → Strings (Shift+F12):
- แสดง strings ทั้งหมดใน binary
- Double-click เพื่อ navigate ไปที่ string
- จาก string: กด X เพื่อดู cross-references
  - ดูว่า string ถูกใช้ที่ function ไหน
```

### 12.5 Cross-References (Xrefs)

```
กด X ที่ address/name ใดๆ เพื่อดู:
- ที่ไหนบ้างที่ reference address นี้
- call xrefs: ใครเรียก function นี้
- data xrefs: ใครอ่าน/เขียน data นี้

ใช้ประโยชน์:
1. ค้นหา string "correct" → X → ดูว่า check_password() ใช้มัน
2. จาก check_password() ดูว่าใครเรียก → main()
3. follow chain กลับไปได้
```

### 12.6 Renaming และ Type Information

```python
# IDA Python Script สำหรับ rename functions
import idc
import idaapi

# Rename function ที่ address
idc.set_name(0x401050, "check_password", idc.SN_CHECK)

# ตั้ง type ของ function
idc.SetType(0x401050, "int __cdecl check_password(char *input)")

# Add comment
idc.set_cmt(0x401089, "compare user input with hardcoded password", 0)
```

---

## 13. Ghidra Basics

### 13.1 Open Project

```
1. เปิด Ghidra
2. File → New Project
3. Non-Shared Project → Next
4. เลือก directory, ใส่ชื่อ → Finish
5. ลาก binary เข้า project หรือ File → Import File
6. Double-click binary → เปิด CodeBrowser
7. ตอบ "Yes" เมื่อถาม Auto Analyze → Analyze
```

### 13.2 Decompiler Output

**Decompiler Window** (ด้านขวา หรือ Window → Decompiler):

```c
/* Ghidra decompiler output ตัวอย่าง */
undefined8 main(void)
{
  int iVar1;
  char local_38 [40];
  
  printf("Enter password: ");
  __isoc99_scanf("%s", local_38);
  iVar1 = strcmp(local_38, "secret123");
  if (iVar1 == 0) {
    printf("Correct!\n");
  } else {
    printf("Wrong!\n");
  }
  return 0;
}
```

### 13.3 Renaming ใน Ghidra

```
1. Double-click variable name → rename ในที่เดียว → update ทุกที่
2. Right-click → Rename Variable
3. Right-click → Rename Function
4. L (keyboard) = Rename ที่ cursor

ตัวอย่าง:
- เปลี่ยน local_38 → user_input
- เปลี่ยน FUN_00401050 → check_password
- เปลี่ยน DAT_00402020 → correct_password_str
```

### 13.4 Useful Ghidra Features

**Symbol Table** (Window → Symbol Table):
- แสดงทุก symbol
- Filter ด้วย search box

**Defined Strings** (Window → Defined Strings):
- แสดง strings ทั้งหมด
- Double-click → navigate

**Function Call Graph** (Graph → Show Function Call Graph):
- แสดง call relationships

**Script Manager** (Window → Script Manager):
- รัน scripts Python/Java

```python
# Ghidra Python Script ตัวอย่าง
# ค้นหา functions ที่เรียก strcmp
from ghidra.program.model.symbol import RefType

func_manager = currentProgram.getFunctionManager()
for func in func_manager.getFunctions(True):
    entry = func.getEntryPoint()
    refs = getReferencesTo(entry)
    for ref in refs:
        if "strcmp" in str(ref):
            print("strcmp called in:", func.getName())
```

---

## 14. Binary Ninja Workflow

### 14.1 Basic Navigation

```
Binary Ninja interface:
- Left panel: Functions list
- Center: Disassembly / MLIL / HLIL view
- Right panel: Cross-references, Symbols

Views (เลือกได้จาก dropdown):
- Disassembly: raw assembly
- Lifted IL: เป็น intermediate representation
- MLIL (Medium Level IL): higher-level
- HLIL (High Level IL): เหมือน decompiled C
```

### 14.2 HLIL (High Level IL)

```c
// Binary Ninja HLIL output ตัวอย่าง
int32_t main()
{
    char buf[0x30];
    printf("Enter password: ");
    scanf("%s", &buf);
    if (strcmp(&buf, "secret123") == 0)
        printf("Correct!\n");
    else
        printf("Wrong!\n");
    return 0;
}
```

### 14.3 Python API

```python
# Binary Ninja Python API
import binaryninja as bn

# Open binary
bv = bn.open_view("/path/to/binary")

# List functions
for func in bv.functions:
    print(func.name, hex(func.start))

# Get function by name
main = bv.get_function_at(bv.get_symbols_by_name("main")[0].address)

# Get HLIL
hlil = main.high_level_il
for instr in hlil:
    print(instr)

# Add comment
bv.set_comment_at(0x401050, "This is check_password()")
```

---

## 15. CTF Crackme Analysis Walkthrough

### 15.1 ตัวอย่าง: Simple Crackme

```bash
$ file crackme_easy
crackme_easy: ELF 64-bit LSB executable, x86-64, not stripped

$ strings crackme_easy
...
Enter the key: 
Correct! Flag is: CTF{%s}
Wrong key!
...
```

**สังเกต**: String "CTF{%s}" บอกว่า key ที่ใส่จะเป็น flag

```bash
$ nm crackme_easy
0000000000401050 T main
0000000000401100 T check_key
0000000000402000 D key_data

$ objdump -d -M intel crackme_easy
```

```asm
0000000000401100 <check_key>:
  401100: push   rbp
  401101: mov    rbp,rsp
  401104: sub    rsp,0x10
  401108: mov    QWORD PTR [rbp-0x8],rdi    ; save key pointer
  40110c: mov    rax,QWORD PTR [rbp-0x8]
  401110: mov    rdi,rax
  401113: call   401040 <strlen@plt>
  401118: cmp    eax,0x10                    ; key length must be 16
  40111b: jne    401150 <check_key+0x50>    ; length check fail
  40111d: mov    ecx,0x0                    ; counter = 0
  401122: jmp    401140
  
  401124: mov    rax,QWORD PTR [rbp-0x8]
  401128: add    rax,rcx                    ; key[i]
  40112b: movzx  eax,BYTE PTR [rax]
  40112e: xor    eax,0x55                   ; key[i] XOR 0x55
  401131: lea    rdx,[rip+0x10ee]           ; rdx = encrypted_key
  401138: movzx  edx,BYTE PTR [rdx+rcx*1]
  40113c: cmp    al,dl                      ; compare
  40113e: jne    401150                     ; mismatch
  401140: add    rcx,0x1                    ; i++
  401144: cmp    rcx,0x10                   ; i < 16?
  401148: jl     401124
  40114a: mov    eax,0x1                    ; return 1 (correct)
  40114f: jmp    401155
  401150: mov    eax,0x0                    ; return 0 (wrong)
  401155: leave
  401156: ret
```

**Analysis:**
1. Key ต้องยาว 16 ตัวอักษร
2. แต่ละ byte ของ key XOR ด้วย 0x55
3. เปรียบเทียบกับ encrypted_key array ที่ rip+0x10ee

```bash
# ดู encrypted_key
$ objdump -s crackme_easy | grep -A 5 "402220:"
# สมมติว่าได้: 36 30 27 27 3c 24 27 30 30 3c 30 36 3d 3d 36 30
```

**Solving:**
```python
# Solve script
encrypted = [0x36, 0x30, 0x27, 0x27, 0x3c, 0x24, 0x27, 0x30,
             0x30, 0x3c, 0x30, 0x36, 0x3d, 0x3d, 0x36, 0x30]
key_byte = 0x55

result = ""
for b in encrypted:
    result += chr(b ^ key_byte)
    
print("Key:", result)
# Key: ch3ck_y0ur_cr4ck
```

### 15.2 ตัวอย่าง: Stripped Binary Crackme

```bash
$ file crackme_stripped
crackme_stripped: ELF 64-bit LSB executable, x86-64, stripped

$ strings crackme_stripped
Enter key: 
Access granted!
Access denied.
/lib64/ld-linux-x86-64.so.2
strcmp
printf
scanf
```

**Strategy สำหรับ stripped binary:**

1. หา entry point จาก ELF header
2. ดู call chain จาก _start → main
3. หา string references
4. ตาม logic

```bash
$ readelf -h crackme_stripped | grep "Entry"
  Entry point address: 0x401000

$ objdump -d -M intel crackme_stripped | grep -A 30 "401000:"
```

```asm
0000000000401000 <_start>:
  401000: ...
  401015: call 0x401020    ; call __libc_start_main setup
  ; ดู argument ที่ส่งใน rdi = address ของ main function

; ค้นหา main โดยดูจาก __libc_start_main argument
; rdi register ก่อน call __libc_start_main
```

### 15.3 ตัวอย่าง: Multi-Stage Crackme

```c
// Source code (เราไม่มี แต่ reverse ออกมาได้)
int stage1(char *input) {
    // Check length
    if (strlen(input) != 20) return 0;
    return 1;
}

int stage2(char *input) {
    // Check checksum
    int sum = 0;
    for (int i = 0; i < 20; i++) sum += input[i];
    return (sum == 0x7d3) ? 1 : 0;
}

int stage3(char *input) {
    // Check specific positions
    return (input[0] == 'C' && 
            input[5] == '{' && 
            input[19] == '}') ? 1 : 0;
}
```

**Approach:**
1. Identify stages ผ่าน call chain
2. Understand ทุก constraint
3. Solve ด้วย constraint solver (z3) หรือ manual

```python
# z3 solver
from z3 import *

# Create symbolic variables
key = [BitVec(f'k{i}', 8) for i in range(20)]
s = Solver()

# Stage 1: length is 20 (already set by array size)

# Stage 2: checksum
s.add(Sum(key) == 0x7d3)

# Stage 3: specific positions
s.add(key[0] == ord('C'))
s.add(key[5] == ord('{'))
s.add(key[19] == ord('}'))

# All printable ASCII
for k in key:
    s.add(k >= 0x20, k <= 0x7e)

if s.check() == sat:
    m = s.model()
    result = ''.join(chr(m[k].as_long()) for k in key)
    print("Key:", result)
```

---

## 16. Common CTF Reverse Engineering Techniques

### 16.1 Patching Binary

```bash
# ใช้ hex editor เพื่อ patch condition
# ตัวอย่าง: เปลี่ยน JNE เป็น JE
# JNE = 0x75, JE = 0x74

$ python3 -c "
data = open('crackme', 'rb').read()
# แก้ address ที่ต้องการ
addr = 0x40108b
patched = data[:addr] + b'\\x74' + data[addr+1:]  # change 0x75 (JNE) to 0x74 (JE)
open('crackme_patched', 'wb').write(patched)
"
$ chmod +x crackme_patched
$ ./crackme_patched
# ใส่อะไรก็ได้ก็ผ่าน
```

### 16.2 Using ltrace/strace

```bash
# ltrace: trace library calls
$ ltrace ./crackme
printf("Enter password: ") = 16
scanf("%s", 0x7ffd1234)
strcmp("wrongpassword", "secret123") = -18  # ← เห็น arguments!
printf("Wrong!\n") = 7
+++ exited (status 0) +++

# strace: trace system calls  
$ strace ./crackme
execve("./crackme", ["./crackme"], ...)
...
write(1, "Enter password: ", 16)
read(0, "test\n", 4096)
write(1, "Wrong!\n", 7)
```

### 16.3 Dynamic Analysis ด้วย GDB

```bash
$ gdb -q ./crackme
(gdb) break strcmp
(gdb) run
# ใส่ password อะไรก็ได้
(gdb) info registers
# ดู rdi = first argument, rsi = second argument
(gdb) x/s $rdi     # ดู argument 1
(gdb) x/s $rsi     # ดู argument 2 (hardcoded password!)

# หรือ break ก่อน conditional jump
(gdb) disas main
(gdb) break *0x40108b    ; break ที่ test instruction
(gdb) continue
(gdb) info registers rax  ; ถ้า rax = 0, strcmp เท่ากัน
```

### 16.4 Angr (Symbolic Execution)

```python
import angr

# Load binary
project = angr.Project('./crackme', auto_load_libs=False)

# Find path ที่ print "Correct"
# และ avoid path ที่ print "Wrong"

# หา addresses
correct_addr = 0x40108d  # address ของ printf("Correct!")
wrong_addr   = 0x40109b  # address ของ printf("Wrong!")

# Create simulation manager
simgr = project.factory.simulation_manager(project.factory.entry_state())

# Explore
simgr.explore(find=correct_addr, avoid=wrong_addr)

if simgr.found:
    state = simgr.found[0]
    # ดู stdin ที่นำไปสู่ correct path
    print("Password:", state.posix.dumps(0))
```

---

## 17. Common Patterns สรุป

### 17.1 Pattern Recognition Quick Reference

```
Pattern                  | C Code equivalent
─────────────────────────┼────────────────────────────
push rbp; mov rbp,rsp   | function entry
leave; ret              | function exit
sub rsp, N              | local variable allocation
[rbp - N]               | local variable access
[rbp + N]               | function argument (stack-passed)
mov rdi,X; call printf  | printf(X)
mov rdi,X; call strlen  | strlen(X)
mov rdi,X; call strcmp  | compare strings
test eax,eax; jz        | if (result == 0)
test eax,eax; jnz       | if (result != 0)
cmp eax,N; jl           | if (eax < N)
cmp eax,N; jge          | if (eax >= N)
xor eax,eax             | eax = 0 (zero idiom)
lea rax,[rbp-N]         | rax = &local_variable
[rax]                   | *pointer dereference
[rax + rcx*4]           | array[index] (int array)
[rax + rcx*8]           | array[index] (long/ptr array)
mov rax,[rax]           | vtable pointer load
call [rax+N]            | virtual function call
```

### 17.2 Calling Conventions

**System V AMD64 (Linux):**
```
Integer/Pointer arguments:   rdi, rsi, rdx, rcx, r8, r9
Float arguments:             xmm0-xmm7
Return value:                rax (integer), xmm0 (float)
Caller-saved:                rax, rcx, rdx, rdi, rsi, r8, r9, r10, r11
Callee-saved:                rbx, rbp, r12, r13, r14, r15
```

**Microsoft x64 (Windows):**
```
Integer/Pointer arguments:   rcx, rdx, r8, r9
Float arguments:             xmm0-xmm3
Return value:                rax
Shadow space:                32 bytes บน stack ก่อน call
Callee-saved:                rbx, rbp, rdi, rsi, r12-r15
```

---

## 18. Tools Setup และ Tips

### 18.1 ติดตั้ง Tools

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install binutils  # nm, objdump, readelf, strings
sudo apt install gdb
sudo apt install ltrace strace
sudo apt install radare2

# Python tools
pip3 install pwntools
pip3 install angr
pip3 install capstone    # disassembly library
pip3 install keystone-engine  # assembler library

# Ghidra (ต้อง Java 11+)
sudo apt install openjdk-11-jdk
wget https://ghidra-sre.org/ghidra_10.x_PUBLIC.zip
unzip ghidra_*.zip
./ghidra_*/ghidraRun

# Binary Ninja (commercial, มี free version)
# ดาวน์โหลดจาก https://binary.ninja/
```

### 18.2 One-Liner Tips

```bash
# ดูทุก function call ใน binary
$ objdump -d binary | grep "call" | grep -oP "call.*" | sort | uniq -c | sort -rn

# Extract all hex strings
$ xxd binary | grep -oP "[0-9a-f]{8}" | sort | uniq

# ดู imported functions
$ objdump -T binary | grep "GLIBC"

# ดู exported functions
$ nm -D --defined-only binary

# Check binary security properties
$ checksec --file=binary  # (จาก pwntools)
# NX: No-eXecute stack
# PIE: Position Independent Executable
# RELRO: RELocation Read-Only
# Canary: Stack canary protection
# ASLR: Address Space Layout Randomization

# Decompress packed binary (UPX)
$ file packed_binary    # จะเห็น "UPX compressed"
$ upx -d packed_binary  # decompress
```

### 18.3 radare2 Quick Reference

```bash
$ r2 ./crackme

[0x00401000]> aaa        # analyze all
[0x00401000]> afl        # list functions
[0x00401000]> pdf @ main # print disassembly of main
[0x00401000]> iz          # strings in data sections
[0x00401000]> iI          # binary info
[0x00401000]> s main      # seek to main
[0x00401000]> VV          # visual graph mode
[0x00401000]> px 32 @ 0x402000  # hex dump 32 bytes
[0x00401000]> pdc @ main  # decompile main (with r2dec plugin)
```

---

## 19. Practice Exercises

### Exercise 1: Basic Static Analysis

```
Task: วิเคราะห์ binary ต่อไปนี้ด้วย static analysis เท่านั้น

1. รัน file command และบอก:
   - Binary format
   - Architecture
   - Stripped หรือไม่
   
2. รัน strings และค้นหา:
   - Possible passwords/keys
   - Success/failure messages
   
3. รัน nm และบอก:
   - Functions ที่น่าสนใจ
   
4. รัน objdump -d และ:
   - หา main function
   - Trace logic ของ password check
   - หา hardcoded password
```

### Exercise 2: Recognize Patterns

```
ดู assembly snippet ต่อไปนี้และบอกว่า C code เป็นอะไร:

Snippet A:
  push rbp
  mov rbp, rsp
  sub rsp, 0x20
  mov [rbp-0x14], edi      ; arg1
  mov [rbp-0x4], 0         ; var = 0
  jmp .L2
.L3:
  mov eax, [rbp-0x14]      ; eax = arg1
  add [rbp-0x4], eax       ; var += arg1
  sub [rbp-0x14], 1        ; arg1--
.L2:
  cmp [rbp-0x14], 0
  jg .L3                    ; while arg1 > 0
  mov eax, [rbp-0x4]       ; return var
  leave
  ret

Snippet B:
  mov rax, [rbp-0x8]
  mov rax, [rax]           ; vtable ptr
  mov rdx, [rax]           ; first vtable entry
  mov rax, [rbp-0x8]
  mov rdi, rax
  call rdx
```

**Answers:**
```
Snippet A: 
int sum_n(int n) {
    int var = 0;
    while (n > 0) {
        var += n;
        n--;
    }
    return var;  // n + (n-1) + ... + 1 = n*(n+1)/2
}

Snippet B:
object->virtual_method();  // virtual function call
```

### Exercise 3: Solve a Crackme

```
Binary logic (reverse engineered):
1. Input length ต้องเป็น 12
2. input[0] == 'A'
3. Sum ของทุก character ต้องเท่ากับ 0x3E8 (1000)
4. input[11] == '!'
5. input[6] == '-'

เขียน Python script เพื่อหา valid input
```

```python
# Solution
from z3 import *

key = [BitVec(f'c{i}', 8) for i in range(12)]
s = Solver()

# Printable ASCII
for c in key:
    s.add(c >= 0x20, c <= 0x7e)

# Constraints
s.add(Sum(key) == 0x3E8)
s.add(key[0] == ord('A'))
s.add(key[11] == ord('!'))
s.add(key[6] == ord('-'))

if s.check() == sat:
    m = s.model()
    result = ''.join(chr(m[c].as_long()) for c in key)
    print("Key:", result)
    # Verify
    assert len(result) == 12
    assert result[0] == 'A'
    assert result[11] == '!'
    assert result[6] == '-'
    assert sum(ord(c) for c in result) == 0x3E8
    print("Verified!")
```

---

## 20. Summary และ Checklist

### 20.1 Reverse Engineering Checklist

```
[ ] Step 1: file command
    [ ] Binary format (ELF/PE/Mach-O)
    [ ] Architecture (x86/x64/ARM)
    [ ] Stripped/Not stripped
    [ ] Linked type (dynamic/static)

[ ] Step 2: strings analysis
    [ ] Hardcoded strings
    [ ] Error/success messages
    [ ] Suspicious encoded strings
    [ ] Imported library functions

[ ] Step 3: Symbol analysis (nm)
    [ ] Interesting function names
    [ ] Global variables
    [ ] External calls

[ ] Step 4: Structure analysis (readelf/objdump -x)
    [ ] Section list
    [ ] Unusual sections
    [ ] Entry point

[ ] Step 5: Disassembly (objdump -d)
    [ ] Find main/entry
    [ ] Follow call chain
    [ ] Identify key functions
    [ ] Note addresses of interest

[ ] Step 6: Advanced analysis (IDA/Ghidra)
    [ ] Full decompilation
    [ ] Cross-references
    [ ] Rename variables/functions
    [ ] Understand algorithm

[ ] Step 7: Dynamic analysis (if needed)
    [ ] ltrace/strace
    [ ] GDB breakpoints
    [ ] Angr symbolic execution

[ ] Step 8: Solve/Exploit
    [ ] Patch binary
    [ ] Write keygen
    [ ] Use solver
```

### 20.2 Quick Command Reference

```bash
# Full analysis pipeline
file binary
strings -n 6 -t x binary
nm -D binary
readelf -a binary 2>&1 | less
objdump -d -M intel binary | less
# แล้วเปิดใน Ghidra/IDA สำหรับ analysis ลึก
```

---

## บทสรุป

Reverse Engineering เป็นทักษะที่ต้องฝึกฝนอย่างสม่ำเสมอ ขั้นตอนหลักคือ:

1. **Reconnaissance**: `file`, `strings`, `nm` - ได้ภาพรวม
2. **Structure Analysis**: `readelf`, `objdump -x` - เข้าใจ binary layout
3. **Disassembly**: `objdump -d`, IDA, Ghidra - อ่าน assembly
4. **Pattern Recognition**: จำ compiler patterns พื้นฐาน
5. **Dynamic Confirmation**: GDB, ltrace เพื่อ verify hypothesis
6. **Solving**: patch, keygen, z3 solver

ใน CTF context ส่วนใหญ่จะเจอ:
- Simple comparison (strcmp, memcmp)
- XOR/ROT decryption
- Multi-stage checks
- Anti-debugging (แก้ด้วย patch หรือ scripting)

การฝึกฝนกับ CTF platforms เช่น:
- **crackmes.one** - collection ของ crackmes
- **picoCTF** - beginner-friendly CTF
- **pwn.college** - comprehensive RE/pwn challenges
- **HackTheBox** - real-world style challenges

จะช่วยพัฒนาทักษะได้เร็วที่สุด

---

*Part 078 จบ - ต่อไป Part 079: Dynamic Analysis and Debugging*
